# Docker安装搭建loki+promtail+grafana收集容器日志


<!--more-->

<div align='center' ><font size='6'>docker安装搭建loki+promtail+grafana 收集容器日志</font></div>

# 安装部署docker环境

## 1.卸载旧版本

旧版本的 Docker 被称为`docker`或`docker-engine`. 如果安装了这些，请卸载它们以及相关的依赖项。

```
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

## 2.安装docker

您可以根据需要以不同的方式安装 Docker Engine：

- 大多数用户 [设置 Docker 的存储库](https://docs.docker.com/engine/install/centos/#install-using-the-repository)并从中安装，以便于安装和升级任务。这是推荐的方法。
- 一些用户下载 RPM 包并 [手动安装它](https://docs.docker.com/engine/install/centos/#install-from-a-package)并完全手动管理升级。这在诸如在无法访问 Internet 的气隙系统上安装 Docker 等情况下很有用。
- 在测试和开发环境中，一些用户选择使用自动化 [便利脚本](https://docs.docker.com/engine/install/centos/#install-using-the-convenience-script)来安装 Docker。

安装`yum-utils`包（提供`yum-config-manager` 实用程序）并设置存储库。

```
yum install -y yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum clean all && yum makecache
yum install -y docker-ce
```

> 列出并排序您的存储库中可用的版本。此示例按版本号从最高到最低对结果进行排序，并被截断：
>
> ```
> $ yum list docker-ce --showduplicates | sort -r
> 
> docker-ce.x86_64  3:18.09.1-3.el7                     docker-ce-stable
> docker-ce.x86_64  3:18.09.0-3.el7                     docker-ce-stable
> docker-ce.x86_64  18.06.1.ce-3.el7                    docker-ce-stable
> docker-ce.x86_64  18.06.0.ce-3.el7                    docker-ce-stable
> ```
>
> 返回的列表取决于启用了哪些存储库，并且特定于您的 CentOS 版本（`.el7`在本例中由后缀表示）。

## 3.启动docker

```
systemctl start docker
systemctl enable docker
```

## 4.安装docker-compose工具

```bash
#国外
curl -L https://github.com/docker/compose/releases/download/v2.10.2/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
#国内
curl -L https://get.daocloud.io/docker/compose/releases/download/v2.10.2/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

> docker-compose 版本：https://github.com/docker/compose/releases

# 安装部署Grafana Loki

## 1.Grafana Loki介绍

**Grafana Loki 是什么？**

Loki 是一个水平可扩展，高可用性，多租户的 `日志聚合系统`
。它的设计非常经济高效且易于操作，因为它不会为日志内容编制索引，而是为每个日志流编制一组标签。

Loki 组成：

1. loki : 主服务器，负责存储日志和处理查询
2. promtail : 代理，负责收集日志并将其发送给 loki
3. Grafana : Go语言开发的开源数据可视化工具，可以做数据监控和数据统计，带有告警功能

官网：https://grafana.com/oss/loki         Loki : https://github.com/grafana/loki      

> 与其他日志聚合系统相比，Loki具有下面的一些特性：
>
> - 不对日志进行全文索引（vs ELK技）。通过存储压缩非结构化日志和仅索引元数据，Loki 操作起来会更简单，更省成本。
> - 通过使用与 Prometheus 相同的标签记录流对日志进行索引和分组，这使得日志的扩展和操作效率更高。
> - 特别适合储存 Kubernetes Pod 日志; 诸如 Pod 标签之类的元数据会被自动删除和编入索引。受 Grafana 原生支持。

## 2.系统架构

![](http://cdn.linuxwf.com/img/loki.png)

Promtail收集并将日志发送给Loki的 Distributor 组件

Distributor会对接收到的日志流进行正确性校验，并将验证后的日志分批并行发送到Ingester

Ingester 接受日志流并构建数据块，压缩后存放到所连接的存储后端

Querier 收到HTTP查询请求，并将请求发送至Ingester 用以获取内存数据 ，Ingester 收到请求后返回符合条件的数据 ；

如果 Ingester 没有返回数据，Querier 会从后端存储加载数据并遍历去重执行查询 ，通过HTTP返回查询结果。

**Loki与ELK比较**

- ELK功能丰富，但是架构复杂，资源占用高，很多功能系统用不上，造成很多资源浪费。
- ELK进行全文索引。安装部署复杂。
- Loki不对日志全文索引。通过存储压缩非结构化日志和仅索引元数据，Loki 操作起来会更简单，更省成本。
- Loki通过使用与 Prometheus 相同的标签记录流对日志进行索引和分组，这使得日志的扩展和操作效率更高。
- Loki安装部署简单快速，且受 Grafana 原生支持。

假如系统依赖于ES，建议使用ELK作为日志系统。若系统不依赖ES，选择用Loki。

## 3.安装部署 Loki

[官网安装文档](https://grafana.com/docs/loki/latest/installation/docker/)

### 3.1使用docker安装

```bash
#创建目录
mkdir -p /data/loki/conf
cd /data/loki/conf
#下载配置文件
wget https://raw.githubusercontent.com/grafana/loki/v2.6.1/cmd/loki/loki-local-config.yaml -O loki-config.yaml
wget https://raw.githubusercontent.com/grafana/loki/v2.6.1/clients/cmd/promtail/promtail-docker-config.yaml -O promtail-config.yaml
```

**修改loki-config配置文件**

```bash
vim loki-config.yaml
```

```bash
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096
  grpc_server_max_recv_msg_size: 15728640  #grpc最大接收消息值，默认4m
  grpc_server_max_send_msg_size: 15728640  #grpc最大发送消息值，默认4m

common:
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  ingestion_rate_mb: 30
  ingestion_burst_size_mb: 15
  
chunk_store_config:
  max_look_back_period: 168h #回看日志行的最大时间，只适用于即时日志
 
table_manager:
  retention_deletes_enabled: true #日志保留周期开关，默认为false
  retention_period: 168h #日志保留周期

ruler:
  alertmanager_url: http://localhost:9093
```

#### 修改promtail的配置文件

```bash
vim promtail-config.yaml
```

```bash
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml
  sync_period: 10s #10秒钟同步一次

#Loki服务器的地址，修改IP和端口即可，后面Grafana要从这个地方取数据
clients:
  - url: http://YOUR_IP:3100/loki/api/v1/push

# 数据抓取配置
scrape_configs:
- job_name: system    			#任务名称，应该具有唯一性
  static_configs:
  - targets:
      - localhost				#抓取目标，本地
    labels:
      job: varlogs				#标签名，用于筛选数据，最好取一个易于识别的名称
      __path__: /var/log/*log	#收集日志的路径，“*”是通配符，支持正则匹配
```

**安装Loki**

```bash
docker run --name loki -d -v $(pwd):/mnt/config -p 3100:3100 grafana/loki:2.6.1 -config.file=/mnt/config/loki-config.yaml

#查看metrics数据
http://IP:3100/metrics

#查看loki运行状态
http://IP:3100/ready
```

**安装promtail**

```bash
docker run --name promtail -d -v $(pwd):/mnt/config -v /var/log:/var/log --link loki grafana/promtail:2.6.1 -config.file=/mnt/config/promtail-config.yaml
```

**安装grafana**

```bash
docker run -d --name grafana \
  --restart always \
  --privileged=true \
  --user root \
  -p 3000:3000 \
  -v /data/loki/grafana-storage:/var/lib/grafana \
  grafana/grafana
```

> <span style="display:block;color:orangered;">安装报错：</span>
>
> GF_PATHS_DATA='/var/lib/grafana' is not writable.
> You may have issues with file permissions, more information here: http://docs.grafana.org/installation/docker/#migrate-to-v51-or-later
> mkdir: can't create directory '/var/lib/grafana/plugins': Permission denied
>
> <span style="display:block;color:orangered;">解决办法：</span>
>
> 启动容器时加上一行 `--user root`

打开 http://IP:3000 访问grafana，默认用户密码为admin/admin。

选择左侧设置—>Data Sources—>Add data source，搜索Loki配置HTTP URL为 http://IP:3100

完成后选择左侧设置—>Preferences，修改底部默认时区为 Asia/Shanghai。

选择左侧Explore查看日志，可以基于文件名或标签查看.

### 3.2 使用docker-compose安装

```bash
mkdir -p /data/{loki,loki-config,promtail-config,grafana-data}
mkdir /var/log/promtail

/data/
├── docker-compose.yaml
├── docker-compose.yaml.bak
├── grafana-data 
├── loki
├── loki-config
│   └── local-config.yaml
└── promtail-config
    └── config.yml

```

**修改loki配置文件 local-config.yaml**

```bash
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096
  grpc_server_max_recv_msg_size: 15728640  #grpc最大接收消息值，默认4m
  grpc_server_max_send_msg_size: 15728640  #grpc最大发送消息值，默认4m

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  ingestion_rate_mb: 30
  ingestion_burst_size_mb: 15
  
chunk_store_config:
  max_look_back_period: 168h #回看日志行的最大时间，只适用于即时日志
 
table_manager:
  retention_deletes_enabled: true #日志保留周期开关，默认为false
  retention_period: 168h #日志保留周期

ruler:
  alertmanager_url: http://localhost:9093
```



#### 下载docker-compose.yml

```bash
cd /data
wget https://raw.githubusercontent.com/grafana/loki/v2.6.1/production/docker-compose.yaml -O docker-compose.yaml
```

#### 修改docker-compose.yml

```bash
version: "3"

networks:
  loki:

services:
  loki:
    image: grafana/loki:2.6.1
    container_name: loki
    environment:
      - TZ=Asia/Shanghai
      - LANG=zh_CN.UTF-8
    restart: always
    privileged: true
    user: root
    ports:
      - "3100:3100"
    volumes:
          - "/data/loki-config:/etc/loki"
          - "/data/loki:/loki"
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - loki

  promtail:
    image: grafana/promtail:2.6.1
    container_name: promtail
    environment:
      - TZ=Asia/Shanghai
      - LANG=zh_CN.UTF-8
    restart: always
    privileged: true
    user: root
    volumes:
          - "/var/log/promtail:/var/log"
          - "/data/promtail-config:/etc/promtail"
    command: -config.file=/etc/promtail/config.yml
    networks:
      - loki

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      - TZ=Asia/Shanghai
      - LANG=zh_CN.UTF-8
    restart: always
    privileged: true
    user: root
    ports:
      - "3000:3000"
    volumes:
          - "/data/grafana-data:/var/lib/grafana"
    networks:
      - loki
```

#### 启动

```bash
docker-compose -f docker-compose.yaml up -d
```

#### 配置grafana

1. 打开 http://IP:3000 访问grafana，默认用户密码为admin/admin。
2. 选择左侧设置—>Data Sources—>Add data source，搜索Loki配置HTTP URL为 http://IP:3100
3. 完成后选择左侧设置—>Preferences，修改底部默认时区为 Asia/Shanghai。

4. 选择左侧Explore查看日志，可以基于文件名或标签查看.

![](http://cdn.linuxwf.com/img/20220909131103.png)

![](http://cdn.linuxwf.com/img/20220909131410.png)

![](http://cdn.linuxwf.com/img/20220909130741.png)

# 监控docker日志

## 1.被监控机器安装Loki日志插件

```bash
docker plugin install grafana/loki-docker-driver:latest --alias loki --grant-all-permissions
docker plugin ls
```

当有新版本时, 更新plugins

```bash
docker plugin disable loki --force
docker plugin upgrade loki grafana/loki-docker-driver:latest --grant-all-permissions
docker plugin enable loki
systemctl restart docker
```

## 2.全局日志搜集设置

**对于loki的docker plugin有两种使用方式**

配置daemon.json,收集此后创建的所有容器的日志(注意，是配置daemon.json后重启docker服务后创建的容器才会把日志输出到loki)。
新建容器时指定logging类型为loki，这样只有指定了logging的容器才会输出到loki

编辑daemon.json。linux下默认路径是/etc/docker/daemon.json (需要sudo)， windows则默认是%userprofile%.docker\daemon.json

```bash
vim /etc/docker/daemon.json

{
    "log-driver": "loki",
    "log-opts": {
        "loki-url": "http://YOUR_IP:3100/loki/api/v1/push",
        "loki-batch-size": "400",
        "max-size": "50m",
        "max-file": "10"
    }
}

systemctl daemon-reload
systemctl restart docker
```

# 好用的grafana模板下载

https://grafana.com/dashboards

```bash
Spring Boot Statistics
	6756
1 Node Exporter for Prometheus Dashboard EN 20201010
	11074
Docker and system monitoring
	893
Docker Container & Host Metrics
	10619
```




