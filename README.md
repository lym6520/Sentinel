<img src="https://user-images.githubusercontent.com/9434884/43697219-3cb4ef3a-9975-11e8-9a9c-73f4f537442d.png" alt="Sentinel Logo" width="50%">

# sentinel-dashboard 数据源改造
## 调整为 zookeeper 注册中心
配置文件添加配置项：
~~~
#指定 zk 注册地址（apache-zookeeper-3.7.0）
sentinel.datasource.zk.address=127.0.0.1:2181
~~~
## 监控数据调整为持久化存储到mysql
配置文件添加如下配置：
~~~
# 系统监控的数据持久化
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/code_artisan?useSSL=false&useUnicode=true&characterEncoding=UTF8&allowMultiQueries=true
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# spring data jpa
spring.jpa.hibernate.ddl-auto=none
spring.jpa.hibernate.use-new-id-generator-mappings=false
spring.jpa.database-platform=org.hibernate.dialect.MySQLDialect
spring.jpa.show-sql=true
~~~
mysql 建表脚本： 
~~~
-- 创建监控数据表
CREATE TABLE `sentinel_metric` (
  `id` INT NOT NULL AUTO_INCREMENT COMMENT 'id，主键',
  `gmt_create` DATETIME COMMENT '创建时间',
  `gmt_modified` DATETIME COMMENT '修改时间',
  `app` VARCHAR(100) COMMENT '应用名称',
  `timestamp` DATETIME COMMENT '统计时间',
  `resource` VARCHAR(250) COMMENT '资源名称',
  `pass_qps` INT COMMENT '通过qps',
  `success_qps` INT COMMENT '成功qps',
  `block_qps` INT COMMENT '限流qps',
  `exception_qps` INT COMMENT '发送异常的次数',
  `rt` DOUBLE COMMENT '所有successQps的rt的和',
  `_count` INT COMMENT '本次聚合的总条数',
  `resource_code` INT COMMENT '资源的hashCode',
  INDEX app_idx(`app`) USING BTREE,
  INDEX resource_idx(`resource`) USING BTREE,
  INDEX timestamp_idx(`timestamp`) USING BTREE,
  PRIMARY KEY (`id`)
) ENGINE=INNODB DEFAULT CHARSET=utf8;
~~~

## 菜单栏调整，保留以下支持：
实时监控、簇点链路、流控规则、熔断规则、系统规则、授权规则、集群流控、机器列表


## 应用端接入说明
> 应用端接入处理参考

1. maven 添加如下依赖：

`<sentinelVersion>1.8.3</sentinelVersion>`
```
        <!--sentinel 限流控制-->
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-core</artifactId>
            <version>${sentinelVersion}</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-transport-simple-http</artifactId>
            <version>${sentinelVersion}</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-datasource-zookeeper</artifactId>
            <version>${sentinelVersion}</version>
        </dependency>

        <!--  引入 集群流控管理  -->
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-cluster-client-default</artifactId>
            <version>${sentinelVersion}</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-cluster-server-default</artifactId>
            <version>${sentinelVersion}</version>
        </dependency>
```
2. 请求入口控制：
```
        // 资源名称 根据介入端自行选择设置，确保描述资源唯一性
        String name = request.getMethod() + ":" + requestUrl.getPath();
        // 授权规则--> 黑白名单控制: 调用方信息设置
        // ogigin 对应黑白名单名称,这里设置为用户名，针对用户名做黑白名单控制
        String origin = UserSession.getSelectedUser().getUsername();
        ContextUtil.enter(name, origin);
        Entry entry = null;
        // 务必保证 finally 会被执行
        try {
            // 资源名可使用任意有业务语义的字符串，注意数目不能太多（超过 1K），超出几千请作为参数传入而不要直接作为资源名
            // EntryType 代表流量类型（inbound/outbound），其中系统规则只对 IN 类型的埋点生效
            entry = SphU.entry(name, EntryType.IN);
            // 被保护的业务逻辑
            // todo do something...
        } catch (BlockException ex) {
            // 资源访问阻止，被限流或被降级
            // 进行相应的处理操作
            LOGGER.warn(String.format("目前请求压力较大，已启动流量限制！-->触发限流控制:%s[%s]",ex.getRuleLimitApp(), ex.getRule()), ex);
        } catch (Exception ex) {
            // 若需要配置降级规则，需要通过这种方式记录业务异常
            Tracer.traceEntry(ex, entry);
            LOGGER.error(String.format("系统繁忙[sentinel]！ --> %s",ex.getMessage()), ex);
           
        } finally {
            // 务必保证 exit，务必保证每个 entry 与 exit 配对
            if (entry != null) {
                entry.exit();
            }
        }
```

3. 应用程序启动时执行相关初始化，参考如下：
```
  //流量规则动态加载
  sentinelConfig.init();
```
SentinelConfig
```
@Component
public class SentinelConfig {
    private static final Logger LOGGER = Logger.getLogger(SentinelConfig.class);

    private final String configDataId = "cluster-client-config";
    private final String clusterMapDataId = "cluster-map";
    private static final String SEPARATOR = "@";


    public void init(){
        // remoteAddress 代表 ZooKeeper 服务端的地址
        String remoteAddress = SystemConfig.getStringPro("sentinel.datasource.zk.address");
        String groupId = SystemConfig.getStringPro("sentinel.datasource.zk.groupId","sentinel_rule_config");
        groupId = groupId + "/" + AppNameUtil.getAppName();//System.getProperty("project.name");

        initDynamicRuleProperty(remoteAddress, groupId);

        // 开启集群流控处理 --- begin
        // 集群流控 init
        initClientConfigProperty(remoteAddress, groupId);
        initServerTransportConfigProperty(remoteAddress, groupId);

        // Register token server related data source.
        // Register dynamic rule data source supplier for token server:
        registerClusterRuleSupplier(remoteAddress, groupId);
        // Token server transport config extracted from assign map:
        initServerTransportConfigProperty(remoteAddress, groupId);

        // Init cluster state property for extracting mode from cluster map data source.
        initStateProperty(remoteAddress, groupId);
        // 开启集群流控处理 --- end
    }
    public void initDynamicRuleProperty( String remoteAddress, String groupId){
        // 流量规则动态加载

        // path 对应 ZK 中的数据路径
        LOGGER.info(String.format("enable sentinel datasource, zk:%s, groupId:%s", remoteAddress, groupId));
        // 流控规则
        String flowDataId = SystemConfig.getStringPro("sentinel.datasource.flowDataId","flow");
        ReadableDataSource<String, List<FlowRule>> flowRuleDataSource = new ZookeeperDataSource<>(remoteAddress, groupId, flowDataId,
                source -> JSON.parseObject(source, new TypeReference<List<FlowRule>>() {}));
        FlowRuleManager.register2Property(flowRuleDataSource.getProperty());

        //熔断规则
        String degradeDataId = SystemConfig.getStringPro("sentinel.datasource.degradeDataId","degrade");
        ReadableDataSource<String, List<DegradeRule>> degradeRuleDataSource = new ZookeeperDataSource<>(remoteAddress, groupId, degradeDataId,
                source -> JSON.parseObject(source, new TypeReference<List<DegradeRule>>() {}));
        DegradeRuleManager.register2Property(degradeRuleDataSource.getProperty());

        // 授权规则
        String authorityDataId = SystemConfig.getStringPro("sentinel.datasource.authorityDataId","authority");
        ReadableDataSource<String, List<AuthorityRule>> authorityRuleDataSource = new ZookeeperDataSource<>(remoteAddress, groupId, authorityDataId,
                source -> JSON.parseObject(source, new TypeReference<List<AuthorityRule>>() {}));
        AuthorityRuleManager.register2Property(authorityRuleDataSource.getProperty());

        // 系统规则
        String systemDataId = SystemConfig.getStringPro("sentinel.datasource.systemDataId","system");
        ReadableDataSource<String, List<SystemRule>> systemRuleDataSource = new ZookeeperDataSource<>(remoteAddress, groupId, systemDataId,
                source -> JSON.parseObject(source, new TypeReference<List<SystemRule>>() {}));
        SystemRuleManager.register2Property(systemRuleDataSource.getProperty());

        EventObserverRegistry.getInstance().addStateChangeObserver("logging",
                (prevState, newState, rule, snapshotValue) -> {
                    if (newState == CircuitBreaker.State.OPEN) {
                        // 变换至 OPEN state 时会携带触发时的值
                        LOGGER.info(String.format("%s -> OPEN at %d, snapshotValue=%.2f", prevState.name(),
                                TimeUtil.currentTimeMillis(), snapshotValue));
                    } else {
                        LOGGER.info(String.format("%s -> %s at %d", prevState.name(), newState.name(),
                                TimeUtil.currentTimeMillis()));
                    }
                });
    }

    private void initClientConfigProperty(String remoteAddress, String groupId) {
        ReadableDataSource<String, ClusterClientConfig> clientConfigDs = new ZookeeperDataSource<>(remoteAddress, groupId,
                configDataId, source -> JSON.parseObject(source, new TypeReference<ClusterClientConfig>() {}));
        ClusterClientConfigManager.registerClientConfigProperty(clientConfigDs.getProperty());
    }

    private void initServerTransportConfigProperty(String remoteAddress, String groupId) {
        ReadableDataSource<String, ServerTransportConfig> serverTransportDs = new ZookeeperDataSource<>(remoteAddress, groupId,
                clusterMapDataId, source -> {
            List<ClusterGroupEntity> groupList = JSON.parseObject(source, new TypeReference<List<ClusterGroupEntity>>() {});
            return Optional.ofNullable(groupList)
                    .flatMap(this::extractServerTransportConfig)
                    .orElse(null);
        });
        ClusterServerConfigManager.registerServerTransportProperty(serverTransportDs.getProperty());
    }

    private Optional<ServerTransportConfig> extractServerTransportConfig(List<ClusterGroupEntity> groupList) {
        return groupList.stream()
                .filter(this::machineEqual)
                .findAny()
                .map(e -> new ServerTransportConfig().setPort(e.getPort()).setIdleSeconds(600));
    }

    private boolean machineEqual(/*@Valid*/ ClusterGroupEntity group) {
        return getCurrentMachineId().equals(group.getMachineId());
    }
    private String getCurrentMachineId() {
        // Note: this may not work well for container-based env.
        return HostNameUtil.getIp() + SEPARATOR + TransportConfig.getRuntimePort();
    }

    private void registerClusterRuleSupplier(String remoteAddress, String groupId) {
        // Register cluster flow rule property supplier which creates data source by namespace.
        // Flow rule dataId 保持一致
        String flowDataId = SystemConfig.getStringPro("sentinel.datasource.flowDataId","flow");
        ClusterFlowRuleManager.setPropertySupplier(namespace -> {
            ReadableDataSource<String, List<FlowRule>> ds = new ZookeeperDataSource<>(remoteAddress, groupId,
                    flowDataId, source -> JSON.parseObject(source, new TypeReference<List<FlowRule>>() {}));
            return ds.getProperty();
        });
        // Register cluster parameter flow rule property supplier which creates data source by namespace.
//        ClusterParamFlowRuleManager.setPropertySupplier(namespace -> {
//            ReadableDataSource<String, List<ParamFlowRule>> ds = new ZookeeperDataSource<>(remoteAddress, groupId,
//                    flowDataId, source -> JSON.parseObject(source, new TypeReference<List<ParamFlowRule>>() {}));
//            return ds.getProperty();
//        });
    }

    private void initStateProperty(String remoteAddress, String groupId) {
        // Cluster map format:
        // [{"clientSet":["112.12.88.66@8729","112.12.88.67@8727"],"ip":"112.12.88.68","machineId":"112.12.88.68@8728","port":11111}]
        // machineId: <ip@commandPort>, commandPort for port exposed to Sentinel dashboard (transport module)
        ReadableDataSource<String, Integer> clusterModeDs = new ZookeeperDataSource<>(remoteAddress, groupId,
                clusterMapDataId, source -> {
            List<ClusterGroupEntity> groupList = JSON.parseObject(source, new TypeReference<List<ClusterGroupEntity>>() {});
            return Optional.ofNullable(groupList)
                    .map(this::extractMode)
                    .orElse(ClusterStateManager.CLUSTER_NOT_STARTED);
        });
        ClusterStateManager.registerProperty(clusterModeDs.getProperty());
    }

    private int extractMode(List<ClusterGroupEntity> groupList) {
        // If any server group machineId matches current, then it's token server.
        if (groupList.stream().anyMatch(this::machineEqual)) {
            return ClusterStateManager.CLUSTER_SERVER;
        }
        // If current machine belongs to any of the token server group, then it's token client.
        // Otherwise it's unassigned, should be set to NOT_STARTED.
        boolean canBeClient = groupList.stream()
                .flatMap(e -> e.getClientSet().stream())
                .filter(Objects::nonNull)
                .anyMatch(e -> e.equals(getCurrentMachineId()));
        return canBeClient ? ClusterStateManager.CLUSTER_CLIENT : ClusterStateManager.CLUSTER_NOT_STARTED;
    }

}

```
ClusterGroupEntity
```
public class ClusterGroupEntity {

    private String machineId;
    private String ip;
    private Integer port;

    private Set<String> clientSet;

    public String getMachineId() {
        return machineId;
    }

    public ClusterGroupEntity setMachineId(String machineId) {
        this.machineId = machineId;
        return this;
    }

    public String getIp() {
        return ip;
    }

    public ClusterGroupEntity setIp(String ip) {
        this.ip = ip;
        return this;
    }

    public Integer getPort() {
        return port;
    }

    public ClusterGroupEntity setPort(Integer port) {
        this.port = port;
        return this;
    }

    public Set<String> getClientSet() {
        return clientSet;
    }

    public ClusterGroupEntity setClientSet(Set<String> clientSet) {
        this.clientSet = clientSet;
        return this;
    }

    @Override
    public String toString() {
        return "ClusterGroupEntity{" +
            "machineId='" + machineId + '\'' +
            ", ip='" + ip + '\'' +
            ", port=" + port +
            ", clientSet=" + clientSet +
            '}';
    }
}

```

其他参见 [alibaba sentinel](https://github.com/alibaba/Sentinel)
