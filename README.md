### 快速开始 nacos-server

#### 1.环境准备

- 64 bit OS，支持 Linux/Unix/Mac/Windows，推荐选用 Linux/Unix/Mac。
- 64 bit JDK 1.8+；[下载](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) & [配置](https://docs.oracle.com/cd/E19182-01/820-7851/inst_cli_jdk_javahome_t/)。
- Maven 3.2.x+；[下载](https://maven.apache.org/download.cgi) & [配置](https://maven.apache.org/settings.html)。

#### 2.下载源码包或安装包

##### 从github上下载源码方式获取

```bash
$ git clone https://github.com/alibaba/nacos.git
$ cd nacos/
$ mvn -Prelease-nacos clean install -U  
$ ls -al distribution/target/
# change the $version to your actual path
$ cd distribution/target/nacos-server-$version/nacos/bin
```

##### 下载编译后压缩包方式

```bash
$ wegt https://github.com/alibaba/nacos/releases/download/1.0.0/nacos-server-1.0.0.tar.gz
$ tar -xvf nacos-server-1.0.0.tar.gz
$ cd nacos/bin
```

#### 3.启动关闭服务

```bash
# 启动命令(standalone代表着单机模式运行，非集群模式)
$ sh startup.sh -m standalone
$ sh shutdown.sh
```



###  nacos-server docker 集群基于mysql集群部署

```bash
$ docker ps
CONTAINER ID        IMAGE                                      COMMAND                  CREATED             STATUS                  PORTS                                            NAMES
c813f6f556b0        store/oracle/mysql-enterprise-server:5.7   "/entrypoint.sh mysq…"   21 hours ago        Up 21 hours (healthy)   0.0.0.0:3306->3306/tcp, 33060/tcp                mysql5.7
# 进入 nacos-server-1.0.0.tar.gz  解压后根目录
$ cd nacos
# 将 nacos-server 建表sql 拷贝到 mysql5.7 容器中
$  docker cp conf/nacos-mysql.sql mysql5.7:/root/mysql/sql

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| nacos_devtest      |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
# 没有 nacos_devtest 库的自行创建一个
mysql> use nacos_devtest;
Database changed
# 执行建表语句
mysql> source /root/mysql/sql/nacos-mysql.sql

# 创建nacos集群配置文件 cluster.conf 用于配置nacos应用ip及端口
$ mkdir conf && touch ./conf/cluster.conf
# 创建 env 目录以放置 nacos-hostname.env 文件用于配置数据库相关信息
$ mkdir env
$ cd env
$ touch nacos-hostname.env
$ cd ..
# 创建 docker-compose.yml
$ touch docker-compose.yml
$ mkdir prometheus && touch ./prometheus/prometheus-cluster.yaml
$ mkdir init.d && touch ./init.d/custom.properties
$ mkdir -p cluster-logs/nacos1 cluster-logs/nacos2 cluster-logs/nacos3
# docker-compose 执行后 docker 会创建一个名为 nacos_default 的 network
# ？network命名方式 当前配置文件所在文件夹_default
$ docker-compose up
# 将 mysql5.7 链接 到 nacos_default
$ docker network connect nacos_default mysql5.7
```

#### 相关配置文件内容
- conf/cluster.conf

  - ```properties
    # msi 宿主机 /etc/hosts 文件中配置，在 docker-compose.yml 中配置好文件映射
    msi:8848
    msi:8849
    msi:8850
    ```

- env/nacos-hostname.env

  - ```properties
    #nacos dev env
    PREFER_HOST_MODE=hostname
    NACOS_SERVERS=nacos1:8848 nacos2:8848 nacos3:8848
    MYSQL_MASTER_SERVICE_HOST=mysql
    MYSQL_MASTER_SERVICE_DB_NAME=nacos_devtest
    MYSQL_MASTER_SERVICE_PORT=3306
    MYSQL_SLAVE_SERVICE_HOST=mysql
    MYSQL_SLAVE_SERVICE_PORT=3306
    MYSQL_MASTER_SERVICE_USER=zhangkai
    MYSQL_MASTER_SERVICE_PASSWORD=password@456
    ```


- docker-compose.yml

  - ```yaml
    version: "3"
    networks:
      nacos_default:
        external: true
    services:
      nacos1:
        hostname: nacos1
        container_name: nacos1
        image: nacos/nacos-server:latest
        volumes:
          - ./cluster-logs/nacos1:/home/nacos/logs
          - ./init.d/custom.properties:/home/nacos/init.d/custom.properties
        ports:
          - "8848:8848"
          - "9555:9555"
        env_file:
          - ./env/nacos-hostname.env
        restart: on-failure
        external_links:
          - mysql5.7:mysql
        networks:
          - nacos_default
      nacos2:
        hostname: nacos2
        image: nacos/nacos-server:latest
        container_name: nacos2
        volumes:
          - ./cluster-logs/nacos2:/home/nacos/logs
          - ./init.d/custom.properties:/home/nacos/init.d/custom.properties
        ports:
          - "8849:8848"
        env_file:
          - ./env/nacos-hostname.env
        restart: on-failure
        external_links:
          - mysql5.7:mysql
        networks:
          - nacos_default
      nacos3:
        hostname: nacos3
        image: nacos/nacos-server:latest
        container_name: nacos3
        volumes:
          - ./cluster-logs/nacos3:/home/nacos/logs
          - ./init.d/custom.properties:/home/nacos/init.d/custom.properties
        ports:
          - "8850:8848"
        env_file:
          - ./env/nacos-hostname.env
        restart: on-failure
        external_links:
          - mysql5.7:mysql
        networks:
          - nacos_default
      prometheus:
        container_name: prometheus
        image: prom/prometheus:latest
        volumes:
          - ./prometheus/prometheus-cluster.yaml:/etc/prometheus/prometheus.yml
        ports:
          - "9090:9090"
        depends_on:
          - nacos1
          - nacos2
          - nacos3
        restart: on-failure
      grafana:
        container_name: grafana
        image: grafana/grafana:latest
        ports:
            - 3000:3000
        restart: on-failure
    ```
  
- prometheus/prometheus-cluster.yaml

  - ```yaml
    # my global config
    global:
      scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
      evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
      # scrape_timeout is set to the global default (10s).
    
    # Alertmanager configuration
    alerting:
      alertmanagers:
        - static_configs:
            - targets:
              # - alertmanager:9093
    
    # Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
    rule_files:
    # - "first_rules.yml"
    # - "second_rules.yml"
    
    # A scrape configuration containing exactly one endpoint to scrape:
    # Here it's Prometheus itself.
    scrape_configs:
      # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
      - job_name: 'prometheus'
    
        # metrics_path defaults to '/metrics'
        # scheme defaults to 'http'.
    
        static_configs:
          - targets: ['localhost:9090']
    
      - job_name: 'nacos'
        metrics_path: '/nacos/actuator/prometheus'
        static_configs:
          - targets: ["nacos1:8848","nacos2:8848","nacos3:8848"]
    ```


- init.d/custom.properties

  - ```properties
    #spring.security.enabled=false
    #management.security=false
    #security.basic.enabled=false
    #nacos.security.ignore.urls=/**
    #management.metrics.export.elastic.host=http://localhost:9200
    # metrics for prometheus
    management.endpoints.web.exposure.include=*
    
    # metrics for elastic search
    #management.metrics.export.elastic.enabled=false
    #management.metrics.export.elastic.host=http://localhost:9200
    
    # metrics for influx
    #management.metrics.export.influx.enabled=false
    #management.metrics.export.influx.db=springboot
    #management.metrics.export.influx.uri=http://localhost:8086
    #management.metrics.export.influx.auto-create-db=true
    #management.metrics.export.influx.consistency=one
    #management.metrics.export.influx.compressed=true
    ```

#### 参考文档

- [nacos集群部署说明](https://nacos.io/zh-cn/docs/cluster-mode-quick-start.html)
- [nacos docker快速开始](https://nacos.io/zh-cn/docs/quick-start-docker.html)