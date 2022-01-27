<img src="https://user-images.githubusercontent.com/9434884/43697219-3cb4ef3a-9975-11e8-9a9c-73f4f537442d.png" alt="Sentinel Logo" width="50%">

# 修改 sentinel-dashboard
1. 调整为 zookeeper 注册中心
配置文件添加配置项：
~~~
#指定 zk 注册地址（apache-zookeeper-3.7.0）
sentinel.datasource.zk.address=127.0.0.1:2181
~~~
2. 菜单栏调整，保留以下支持：
流控规则、熔断规则、授权规则、系统规则、机器列表


其他参见 [alibaba sentinel](https://github.com/alibaba/Sentinel)
