# VictoriaMetrics-读写分析篇

## 数据写入
以Prometheus数据写入为例子研究：
1.数据会从```/api/v1/import/prometheus```这个API进入，在```VM->app->vminsert->prometheusimport->request_handler.go->insertRows```函数中，进行数据处理
2.

### vminsert


### vmstorage


## 数据读取


### vminsert


### vmstorage

