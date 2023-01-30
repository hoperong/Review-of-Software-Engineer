# APScheduler
## 概述
Advanced Python Scheduler（APScheduler）是一个Python库，https://github.com/agronholm/apscheduler，可让您安排Python代码稍后执行，一次或定期执行。您可以根据需要动态添加或删除旧作业。如果将作业存储在数据库中，它们也将在调度程序重新启动并保持其状态的过程中幸免。重新启动调度程序后，它将运行脱机1时应运行的所有作业。

除其他事项外，APScheduler可用作跨平台的，特定于应用程序的替代程序，以替换平台特定的计划程序，例如cron守护程序或Windows任务计划程序。但是请注意，APScheduler本身不是守护程序或服务，也不是任何命令行工具附带的。它主要是要在现有应用程序中运行。也就是说，APScheduler确实为您提供了一些构建块，以构建调度程序服务或运行专用的调度程序进程。

四个基本概念：
schedulers：调度程序，用于管理名下注册的所有job的一个控制中心。
jobstores：job的存储器，用于保存job数据，实质上也就是job。
triggers：触发器，执行job的形式。
executors：执行器，job的运行模式。

简单来说，apscheduler这个库，就是提供任务调度功能，创建一个scheduler管理多个job，使用各种存储形式保存job信息以此构建job对象，利用triggers去描述job执行的条件并且去触发job的执行，通过executors去运行job。
## jobstores
在apscheduler->jobstores->base.py里面定义了BaseJobStore类，作为job的基础类，里面使用@abstractmethod（python的抽象方法）来定义子类规范，然后通过继承BaseJobStore类，去写支持不同存储底层的jobstore子类，最终实现不同存储类型的job。

抽象方法有：
lookup_job：使用job_id获取一个job
get_due_jobs：参数是一个时间now，查询当前时间到now时间之间会被触发的jobs
get_next_run_time：获取离当前最近的job的触发时间
get_all_jobs：获取所有job
add_job：添加一个job
update_job：利用一个job对象去更新job
remove_job：删除一个job，使用job_id
remove_all_jobs：删除所有job

目前支持memory、mongodb、redis、rethinkdb、sqlalchemy、zookeeper。

拿sqlalchemy举例。
```
class SQLAlchemyJobStore(BaseJobStore):

    def __init__(self, url=None, engine=None, tablename='apscheduler_jobs', metadata=None,
                 pickle_protocol=pickle.HIGHEST_PROTOCOL, tableschema=None, engine_options=None):
        super(SQLAlchemyJobStore, self).__init__()
        self.pickle_protocol = pickle_protocol
        metadata = maybe_ref(metadata) or MetaData()

        if engine:
            self.engine = maybe_ref(engine)
        elif url:
            self.engine = create_engine(url, **(engine_options or {}))
        else:
            raise ValueError('Need either "engine" or "url" defined')

        # 191 = max key length in MySQL for InnoDB/utf8mb4 tables,
        # 25 = precision that translates to an 8-byte float
        
        # 使用表来存储job，定义了，id、下一次运行时间、job信息，结构表
        # 从这个表结构，可以先预想出一种可能任务运行模式
        # 1.按照时间从小到大排序order by next_run_time，取第一个limit 1，然后sleep到那个时间点运行任务，运行完后计算出这个任务的下次运行时间，更新数据库
        # 2.重复上一步
        
        self.jobs_t = Table(
            tablename, metadata,
            Column('id', Unicode(191, _warn_on_bytestring=False), primary_key=True),
            Column('next_run_time', Float(25), index=True),
            Column('job_state', LargeBinary, nullable=False),
            schema=tableschema
        )

    def get_all_jobs(self):
        jobs = self._get_jobs()
        self._fix_paused_jobs_sorting(jobs)
        return jobs

    def add_job(self, job):
        insert = self.jobs_t.insert().values(**{
            'id': job.id,
            'next_run_time': datetime_to_utc_timestamp(job.next_run_time),
            'job_state': pickle.dumps(job.__getstate__(), self.pickle_protocol)
        })
        try:
            self.engine.execute(insert)
        except IntegrityError:
            raise ConflictingIdError(job.id)

    def _reconstitute_job(self, job_state):
        # 反序列化出job_state
        job_state = pickle.loads(job_state)
        # 把存储形式信息，保存进job_state里
        job_state['jobstore'] = self
        # 构建job对象
        job = Job.__new__(Job)
        # 把job_state信息使用job的__setstate__方法保存进job里
        job.__setstate__(job_state)
        job._scheduler = self._scheduler
        job._jobstore_alias = self._alias
        return job

    def _get_jobs(self, *conditions):
        jobs = []
        selectable = select([self.jobs_t.c.id, self.jobs_t.c.job_state]).\
            order_by(self.jobs_t.c.next_run_time)
        selectable = selectable.where(and_(*conditions)) if conditions else selectable
        failed_job_ids = set()
        for row in self.engine.execute(selectable):
            try:
                jobs.append(self._reconstitute_job(row.job_state))
            except BaseException:
                self._logger.exception('Unable to restore job "%s" -- removing it', row.id)
                failed_job_ids.add(row.id)

        # Remove all the jobs we failed to restore
        # 会删除掉数据库里面失败的任务，所以数据库里面记录的都是正在运行的数据
        if failed_job_ids:
            delete = self.jobs_t.delete().where(self.jobs_t.c.id.in_(failed_job_ids))
            self.engine.execute(delete)

        return jobs
```

## triggers
trigger也是和jobstore一样，使用父类BaseTrigger定义抽象方法，子类继承，来实现多种trigger。

抽象方法：
get_next_fire_time：计算下次运行时间

trigger在APScheduler中，实质上用于解析翻译触发配置，给schedulers计算job下次调用时间，返回给schedulers，让schedulers知道接下来的各个任务会在什么时候需要调用，去安排后续的job调用规划。

目前分三种trigger：
interval，间隔、周期性任务，每三分钟、每两小时这样子。
date，一次性任务。
cron，Cron样式的调度。

拿interval举例。

```
class IntervalTrigger(BaseTrigger):

    def get_next_fire_time(self, previous_fire_time, now):
        if previous_fire_time:
            # 如果存在上一次时间，加周期时间就是下一次时间
            next_fire_time = previous_fire_time + self.interval
        elif self.start_date > now:
            # 如果开始时间大于当前，表示还没有开始，则下次时间就是开始时间
            next_fire_time = self.start_date
        else:
            # 如果开始时间小于当前，表示已经开始了n轮，通过时间差和周期时间算出当前已经经历了n轮，则下次时间就是开始时间+(n+1)*周期时间
            timediff_seconds = timedelta_seconds(now - self.start_date)
            next_interval_num = int(ceil(timediff_seconds / self.interval_length))
            next_fire_time = self.start_date + self.interval * next_interval_num

        if self.jitter is not None:
            next_fire_time = self._apply_jitter(next_fire_time, self.jitter, now)

        if not self.end_date or next_fire_time <= self.end_date:
            return self.timezone.normalize(next_fire_time)

    def __getstate__(self):
        return {
            'version': 2,
            'timezone': self.timezone,
            'start_date': self.start_date,
            'end_date': self.end_date,
            'interval': self.interval,
            'jitter': self.jitter,
        }
```

## executors
executor也是和jobstore一样，使用父类BaseExecutor定义抽象方法，子类继承，来实现多种executor。

抽象方法：
_do_submit_job：计算下次运行时间

schedulers通过调用executor的submit_job方法，来运行job。父类的submit_job里面调用了抽象方法_do_submit_job。

目前有七种executor：asyncio、debug、gevent、thread_pool、process_pool、tornado、twisted

拿thread_pool举例。

```
class BasePoolExecutor(BaseExecutor):
    @abstractmethod
    def __init__(self, pool):
        super(BasePoolExecutor, self).__init__()
        self._pool = pool

    def _do_submit_job(self, job, run_times):
        def callback(f):
            exc, tb = (f.exception_info() if hasattr(f, 'exception_info') else
                       (f.exception(), getattr(f.exception(), '__traceback__', None)))
            if exc:
                self._run_job_error(job.id, exc, tb)
            else:
                self._run_job_success(job.id, f.result())

        try:
            # 使用池子来执行job
            f = self._pool.submit(run_job, job, job._jobstore_alias, run_times, self._logger.name)
        except BrokenProcessPool:
            self._logger.warning('Process pool is broken; replacing pool with a fresh instance')
            self._pool = self._pool.__class__(self._pool._max_workers)
            f = self._pool.submit(run_job, job, job._jobstore_alias, run_times, self._logger.name)
        # 利用回调来处理失败、成功
        f.add_done_callback(callback)

    def shutdown(self, wait=True):
        self._pool.shutdown(wait)


class ThreadPoolExecutor(BasePoolExecutor):

    def __init__(self, max_workers=10):
        # 使用线程池
        pool = concurrent.futures.ThreadPoolExecutor(int(max_workers))
        super(ThreadPoolExecutor, self).__init__(pool)
```

## job
apscheduler->job.py
```
class Job(object):

    def __getstate__(self):
        # Don't allow this Job to be serialized if the function reference could not be determined
        if not self.func_ref:
            raise ValueError(
                'This Job cannot be serialized since the reference to its callable (%r) could not '
                'be determined. Consider giving a textual reference (module:function name) '
                'instead.' % (self.func,))

        # Instance methods cannot survive serialization as-is, so store the "self" argument
        # explicitly
        func = self.func
        if ismethod(func) and not isclass(func.__self__) and obj_to_ref(func) == self.func_ref:
            args = (func.__self__,) + tuple(self.args)
        else:
            args = self.args

        return {
            'version': 1,
            'id': self.id,
            'func': self.func_ref,
            'trigger': self.trigger,
            'executor': self.executor,
            'args': args,
            'kwargs': self.kwargs,
            'name': self.name,
            'misfire_grace_time': self.misfire_grace_time,
            'coalesce': self.coalesce,
            'max_instances': self.max_instances,
            'next_run_time': self.next_run_time
        }
```

## schedulers
scheduler也是和jobstore一样，使用父类BaseScheduler定义抽象方法，子类继承，来实现多种scheduler。

拿background举例。

```
class BaseScheduler(six.with_metaclass(ABCMeta)):
    
    def start(self, paused=False):
        if self.state != STATE_STOPPED:
            raise SchedulerAlreadyRunningError

        self._check_uwsgi()

        with self._executors_lock:
            # Create a default executor if nothing else is configured
            if 'default' not in self._executors:
                self.add_executor(self._create_default_executor(), 'default')

            # Start all the executors
            for alias, executor in six.iteritems(self._executors):
                executor.start(self, alias)

        with self._jobstores_lock:
            # Create a default job store if nothing else is configured
            if 'default' not in self._jobstores:
                self.add_jobstore(self._create_default_jobstore(), 'default')

            # Start all the job stores
            for alias, store in six.iteritems(self._jobstores):
                store.start(self, alias)

            # Schedule all pending jobs
            for job, jobstore_alias, replace_existing in self._pending_jobs:
                self._real_add_job(job, jobstore_alias, replace_existing)
            del self._pending_jobs[:]

        self.state = STATE_PAUSED if paused else STATE_RUNNING
        self._logger.info('Scheduler started')
        self._dispatch_event(SchedulerEvent(EVENT_SCHEDULER_START))

        if not paused:
            self.wakeup()

    def _process_jobs(self):
        if self.state == STATE_PAUSED:
            self._logger.debug('Scheduler is paused -- not processing jobs')
            return None

        self._logger.debug('Looking for jobs to run')
        now = datetime.now(self.timezone)
        next_wakeup_time = None
        events = []

        with self._jobstores_lock:
            for jobstore_alias, jobstore in six.iteritems(self._jobstores):
                try:
                    due_jobs = jobstore.get_due_jobs(now)
                except Exception as e:
                    # Schedule a wakeup at least in jobstore_retry_interval seconds
                    self._logger.warning('Error getting due jobs from job store %r: %s',
                                         jobstore_alias, e)
                    retry_wakeup_time = now + timedelta(seconds=self.jobstore_retry_interval)
                    if not next_wakeup_time or next_wakeup_time > retry_wakeup_time:
                        next_wakeup_time = retry_wakeup_time

                    continue

                for job in due_jobs:
                    # Look up the job's executor
                    try:
                        executor = self._lookup_executor(job.executor)
                    except BaseException:
                        self._logger.error(
                            'Executor lookup ("%s") failed for job "%s" -- removing it from the '
                            'job store', job.executor, job)
                        self.remove_job(job.id, jobstore_alias)
                        continue

                    run_times = job._get_run_times(now)
                    run_times = run_times[-1:] if run_times and job.coalesce else run_times
                    if run_times:
                        try:
                            executor.submit_job(job, run_times)
                        except MaxInstancesReachedError:
                            self._logger.warning(
                                'Execution of job "%s" skipped: maximum number of running '
                                'instances reached (%d)', job, job.max_instances)
                            event = JobSubmissionEvent(EVENT_JOB_MAX_INSTANCES, job.id,
                                                       jobstore_alias, run_times)
                            events.append(event)
                        except BaseException:
                            self._logger.exception('Error submitting job "%s" to executor "%s"',
                                                   job, job.executor)
                        else:
                            event = JobSubmissionEvent(EVENT_JOB_SUBMITTED, job.id, jobstore_alias,
                                                       run_times)
                            events.append(event)

                        # Update the job if it has a next execution time.
                        # Otherwise remove it from the job store.
                        job_next_run = job.trigger.get_next_fire_time(run_times[-1], now)
                        if job_next_run:
                            job._modify(next_run_time=job_next_run)
                            jobstore.update_job(job)
                        else:
                            self.remove_job(job.id, jobstore_alias)

                # Set a new next wakeup time if there isn't one yet or
                # the jobstore has an even earlier one
                jobstore_next_run_time = jobstore.get_next_run_time()
                if jobstore_next_run_time and (next_wakeup_time is None or
                                               jobstore_next_run_time < next_wakeup_time):
                    next_wakeup_time = jobstore_next_run_time.astimezone(self.timezone)

        # Dispatch collected events
        for event in events:
            self._dispatch_event(event)

        # Determine the delay until this method should be called again
        if self.state == STATE_PAUSED:
            wait_seconds = None
            self._logger.debug('Scheduler is paused; waiting until resume() is called')
        elif next_wakeup_time is None:
            wait_seconds = None
            self._logger.debug('No jobs; waiting until a job is added')
        else:
            wait_seconds = min(max(timedelta_seconds(next_wakeup_time - now), 0), TIMEOUT_MAX)
            self._logger.debug('Next wakeup is due at %s (in %f seconds)', next_wakeup_time,
                               wait_seconds)

        return wait_seconds


class BlockingScheduler(BaseScheduler):

    _event = None

    def start(self, *args, **kwargs):
        if self._event is None or self._event.is_set():
            self._event = Event()

        super(BlockingScheduler, self).start(*args, **kwargs)
        self._main_loop()

    def shutdown(self, wait=True):
        super(BlockingScheduler, self).shutdown(wait)
        self._event.set()

    def _main_loop(self):
        wait_seconds = TIMEOUT_MAX
        while self.state != STATE_STOPPED:
            # from threading import Event 的wait
            self._event.wait(wait_seconds)
            self._event.clear()
            wait_seconds = self._process_jobs()

    def wakeup(self):
        self._event.set()


class BackgroundScheduler(BlockingScheduler):

    _thread = None

    def _configure(self, config):
        self._daemon = asbool(config.pop('daemon', True))
        super(BackgroundScheduler, self)._configure(config)

    def start(self, *args, **kwargs):
        if self._event is None or self._event.is_set():
            self._event = Event()

        BaseScheduler.start(self, *args, **kwargs)
        self._thread = Thread(target=self._main_loop, name='APScheduler')
        self._thread.daemon = self._daemon
        self._thread.start()

    def shutdown(self, *args, **kwargs):
        super(BackgroundScheduler, self).shutdown(*args, **kwargs)
        self._thread.join()
        del self._thread
```