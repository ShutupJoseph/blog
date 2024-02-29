## 官方文档:
https://shardingsphere.apache.org/document/current/cn/quick-start/  
proxy 里缺少很多东西，要看jdbc里

## Example:
```
databaseName: 逻辑库名

dataSources:
  write_ds:
    url: jdbc:mysql://地址:3306/库?useSSL=yes&useUnicode=true&characterEncoding=utf8&allowMultiQueries=true&serverTimezone=Asia/Shanghai
    username: 
    password: 
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 500
    minPoolSize: 1
  read_ds_0:
    url: jdbc:mysql://地址:3306/库?useSSL=yes&useUnicode=true&characterEncoding=utf8&allowMultiQueries=true&serverTimezone=Asia/Shanghai
    username: 
    password: 
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 500
    minPoolSize: 1

rules:
- !READWRITE_SPLITTING
  dataSources:
    readwirte_ds:
      writeDataSourceName: write_ds
      readDataSourceNames:
        - read_ds_0
      transactionalReadQueryStrategy: PRIMARY
      loadBalancerName: random
  loadBalancers:
    random:
      type: RANDOM
- !SINGLE
  tables:
    - "*.*"
  defaultDataSource: write_ds
```
ps 当前版本示例版本为5.4.1,如果不使用!SINGLE 单表功能逻辑库里不会同步表的元数据，如果要做ddl修改需用从逻辑库执行，也可以用```REFRESH TABLE METADATA;```手动同步
