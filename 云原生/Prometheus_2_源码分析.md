# Prometheus 源码分析

本文基于的prometheus版本是2.36.1，按照prometheus的使用顺序，对源码展开分析。主要目的是通过分析源码来学习了解Prometheus的设计思想和功能，并不会很详细的对各种判断分支、配置选项等去做一一说明。

源码的根目录需要关注的主要目录：
|---cmd：用于生成各种可执行文件
|---config
|---discovery
|---model
|---notifier
|---promql
|---rules
|---scrape
|---storage
|---tsdb
|---util
|---web
|---......

## cmd

### prometheus
用于生成prometheus可执行文件，
使用kingpin来进行启动命令的解析，支持server、agent两种模式，有公用的配置项和模式独有的配置项，如果填写了其他模式的配置项，启动会报错。
```
a := kingpin.New(filepath.Base(os.Args[0]), "The Prometheus monitoring server").UsageWriter(os.Stdout)
...
a.Flag(... // 通用配置项
serverOnlyFlag(... // 作为服务的独有配置项
agentOnlyFlag(...  // 作为agent独有的配置项
...
```

通过设计一些列任务，使用github.com/oklog/run来一口气启动这些任务，github.com/oklog/run通过管理goroutine来实现任务的运行。从中可以看出，agengt模式相比较server模式，少了Query Engine、Rule manager、TSDB，多了WAL storage（预写日志，文件保存少部分日志）。
```
var g run.Group
...
{
    // Scrape discovery manager.
    g.Add(
        func() error {
            err := discoveryManagerScrape.Run()
            level.Info(logger).Log("msg", "Scrape discovery manager stopped")
            return err
        },
        func(err error) {
            level.Info(logger).Log("msg", "Stopping scrape discovery manager...")
            cancelScrape()
        },
    )
}
...
if err := g.Run(); err != nil {
		level.Error(logger).Log("err", err)
		os.Exit(1)
	}
	level.Info(logger).Log("msg", "See you next time!")
```

#### prometheus 模式
server模式，就是平时使用的prometheus模式。
agent模式，把prometheus单纯看成一个监控数据汇总的采集端，通过pull模式拉取监控数据，然后通过remote write的形式，push汇总数据写入到远程存储中。可以在远程那端，实现多prometheus的数据汇总展示。适用于多环境监控、边缘监控、内网监控等场景。并且为了保证稳定高效，agent模式会默认关闭其 UI 查询能力，报警以及本地存储等能力。

### promtool
一个prometheus提供的命令行工具，可以完成：
+ 配置文件的检测
+ prometheus数据查询
+ 对prometheus进行单元测试、测试rule
+ 对prometheus进行基准测试
+ 数据库导入导出
+ ......

## web
采用前后端分离，ui方面使用react来实现的，api方面使用原生的net/http来实现的。
在web/web.go里面定义了web运行逻辑的路由匹配规则，在web/api/api.go里面定义了prometheus能力的开放接口

## config
在config/config.go文件里面定义了配置文件对应格式内容，在cmd/prometheus/main.go的时候会对config配置文件进行初始化，在run任务的时候，添加了，去支持reload配置文件
```

```

## scrape

## discovery


## model


## storage


## tsdb

## promql

## rules

## notifier
