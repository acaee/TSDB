前面已经对`alertmanager.yml`的做了报警的配置，本文只对alertmanager的部署方式做一个简单的描述
### 一. 注册altermanager服务
本次使用了2个alertmanager节点

192.168.0.83的服务配置如下：
```
vim /lib/systemd/system/alertmanager.service

[Unit]
Description=Alertmanager
After=network-online.target

[Service]
Restart=on-failure
ExecStart=/root/alertmanager-0.20.0.linux-amd64/alertmanager --web.external-url=http://192.168.0.83:9093 --cluster.listen-address=192.168.0.83:9094 --cluster.peer=192.168.0.83:9094 --cluster.peer=192.168.0.81:9094 --config.file=/root/alertmanager-0.20.0.linux-amd64/alertmanager.yml

[Install]
WantedBy=multi-user.target
```
192.168.0.81的服务配置如下：
```
vim /lib/systemd/system/alertmanager.service

[Unit]
Description=Alertmanager
After=network-online.target

[Service]
Restart=on-failure
ExecStart=/root/alertmanager-0.20.0.linux-amd64/alertmanager --web.external-url=http://192.168.0.81:9093 --cluster.listen-address=192.168.0.81:9094 --cluster.peer=192.168.0.81:9094 --cluster.peer=192.168.0.83:9094 --config.file=/root/alertmanager-0.20.0.linux-amd64/alertmanager.yml

[Install]
WantedBy=multi-user.target
```

### 二. 在`prometheus.yml`中alertmanager配置修改成如下内容

```
# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - 192.168.0.81:9093
      - 192.168.0.83:9093
```

### 三. 重启让配置生效

1. 重启`prometheus`
```
systemctl restart prometheus.service
```
2. 启动alertmanager(在两台节点上分别启动)
```
systemctl start alertmanager.service
```
### 四. 查看集群状态

使用浏览器打开 http://192.168.0.81:9093/#/status 或 http://192.168.0.82:9093/#/status

可以看到集群状态

```
Cluster Status
Name:       01E7Q1NAXF3H663CJFQ4EYSNB5
Status:     ready
Peers:
            Name:01E7Q1NAXF3H663CJFQ4EYSNB5
            Address:192.168.0.81:9094
            Name: 01E7Q24F7TJNHGZM8M4V8HAZMX
            Address: 192.168.0.83:9094
```