## 容器监控解决方案 Alertmanager Prometheus Grafana Exporter cAdvisor

# 1.邮箱和slack准备
```
slack: 配置方法
1. 申请slack账号
2. 创建channel #prometheus
3. 添加app incoming-webhook ,点击View--> Webhook URL [复制这里得link，后面有用]

QQ邮箱生成授权码
1.设置-->账户-->POP3/SMTP服务，启用
2.点击“生成授权码” 这里得授权码就是登陆QQ时用的 

```

# 2.获取配置文件
```
git clone https://github.com/JasonYLong/docker-compose-alertmanager-prometheus.git

修改如下配置文件
alertmanager.yml  修改smtp_auth_password为QQ授权码，并修改其中的smtp_from smtp_auth_username邮箱名
                  修改api_url 为slack webhook地址
load_over.yml     修改link中IP  http://IP:9090/graph?g0.range_input=1h&g0.expr=node_load1%20%3E%200.8&g0.tab=1
memory_over.yml   修改link中IP 
node_down.yml     修改link中IP 
prometheus.yml    不用修改

```

# 3.启动容器
```
创建网络my_net2
   # docker network create --driver bridge --subnet 172.22.17.0/24 --gateway 172.22.17.1 my_net2
   # brctl show
   # docker inspect my_net2
   
1.启动node-exporter
docker run -d -p 9100:9100 \
  -v "/proc:/host/proc" \
  -v "/sys:/host/sys" \
  -v "/:/rootfs" \
  --net=my_net2 \
  --cap-add=SYS_TIME \
  -h node-exporter \
  --name=node-exporter \
  prom/node-exporter \
  --path.procfs /host/proc \
  --path.sysfs /host/sys \
  --collector.filesystem.ignored-mount-points "^/(sys|proc|dev|host|etc)($|/)"

Node Exporter 启动后，将通过 9100 提供 host 的监控数据。在浏览器中通过 http://IP:9100/metrics 测试一下。

------------------------------------
2.启动cadvisor
docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:rw \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  -p 8080:8080 \
  --detach=true \
  -h cadvisor \
  --name=cadvisor \
  --net=my_net2 \
  google/cadvisor:latest

cAdvisor 启动后，将通过 8080 提供 host 的监控数据。在浏览器中通过 http://IP:8080/metrics 测试一下。

如果启动cAdvisor报错
docker logs container_id
F0615 05:57:12.024944       1 cadvisor.go:156] Failed to start container manager: inotify_add_watch /sys/fs/cgroup/cpuacct,cpu: no such file or directory

fixed:
    mount -o remount,rw '/sys/fs/cgroup'
    ln -s /sys/fs/cgroup/cpu,cpuacct /sys/fs/cgroup/cpuacct,cpu

------------------------------------
3.启动alertmanager
docker run -d -p 9093:9093 -v /root/docker/prometheus/alertmanager.yml:/etc/alertmanager/alertmanager.yml --net=my_net2 -h alertmanager --name alertmanager prom/alertmanager

如果配置文件加载成功，在 http://IP:9093/#/status 会看到Config中是你的配置文件中的配置

------------------------------------
4.启动prometheus
docker run -d -p 9090:9090 \
  -v /root/docker/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v /root/docker/prometheus/node_down.yml:/etc/prometheus/node_down.yml \
  -v /root/docker/prometheus/memory_over.yml:/etc/prometheus/memory_over.yml \
  -v /root/docker/prometheus/load_over.yml:/etc/prometheus/load_over.yml \
  -h prometheus \
  --name prometheus \
  --net=my_net2 \
  prom/prometheus

通过 http://IP:9090/metrics 测试一下
在浏览器中打开 http://IP:9090 ，点击菜单 Status -> Targets
所有 Target 的 State 都是 UP，说明 Prometheus Server 能够正常获取监控数据。
报警规则配置成功在 http://IP:9090/alerts 可以看到报警规则已经添加到prometheus的Alerts中

测试报警功能
停掉cAdvisor容器
docker stop cadvisor
等待一会，看是否会给你配置的邮件报警

------------------------------------
5.启动grafana
docker run -d -i -p 3000:3000 \
  -h grafana \
  --name=grafana \
  --net=my_net2 \
  grafana/grafana      

Grafana 启动后。在浏览器中打开 http://IP:3000/  
grafana默认密码 admin admin

```

# 4.docker-compose 方式
```

启动容器
docker-compose up -d

删除容器
docker-compose down

查看容器
docker ps
CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS              PORTS                    NAMES
a32ea8d61155        grafana/grafana:latest      "/run.sh"                5 minutes ago       Up 5 minutes        0.0.0.0:3000->3000/tcp   grafana
369db70a0c3f        prom/prometheus:latest      "/bin/prometheus -..."   5 minutes ago       Up 5 minutes        0.0.0.0:9090->9090/tcp   prometheus
9be4ba9393b7        prom/node-exporter:latest   "/bin/node_exporte..."   5 minutes ago       Up 5 minutes        0.0.0.0:9100->9100/tcp   node-exporter
bed69a3e13d1        prom/alertmanager           "/bin/alertmanager..."   5 minutes ago       Up 5 minutes        0.0.0.0:9093->9093/tcp   alertmanager
cb1158e0984d        google/cadvisor:latest      "/usr/bin/cadvisor..."   5 minutes ago       Up 5 minutes        0.0.0.0:8080->8080/tcp   cadvisor

```

# 5.alertmanager报警测试方法
```
符合 user: jason 发送邮件给 email  XXX@foxmail.com
符合 user: long  发送消息给 slack  channel: "#prometheus"
以上都不符合，发送邮件给 default-email XXX@139.com

user: jason  对应 node_down.yml   满足cadvisor instance down 发邮件给 email            
user: long   对应 memory_over.yml 满足Memory usage is above 30% 发消息给 slack
user: jasonlong 对应 load_over.yml 满足OS load is over 0.8 发邮件给 default-email

``` 

![result](https://github.com/JasonYLong/docker-compose-alertmanager-prometheus/tree/master/images/result.jpg)
