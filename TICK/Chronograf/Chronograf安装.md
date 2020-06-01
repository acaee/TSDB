### 一. 下载和安装
```
wget https://dl.influxdata.com/chronograf/releases/chronograf_1.8.4_amd64.deb
sudo dpkg -i chronograf_1.8.4_amd64.deb
```

### 二. 启动Chronograf
```
$ systemctl start chronograf.service
$ systemctl status chronograf.service
● chronograf.service - Open source monitoring and visualization UI for the entire TICK stack.
   Loaded: loaded (/lib/systemd/system/chronograf.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2020-05-11 18:43:44 CST; 2 days ago
     Docs: https://www.influxdata.com/time-series-platform/chronograf/
 Main PID: 1361 (chronograf)
    Tasks: 18
   Memory: 82.1M
      CPU: 25.117s
   CGroup: /system.slice/chronograf.service
           └─1361 /usr/bin/chronograf
```


### 三. 连接InfluxDB
打开浏览器访问端口:`8888`
会有以下几个选项：
```
Connection URL #必填(此地址为:http://InfluxDB_data_node_IP:8086)
Connection Name #必填
Username #如果启用请填写
Password #如果启用请填写
Telegraf Dababase Name #选填
Default Retention Policy #选填
Meta Service Connection URL #必填(只有企业版集群才有此项)

```
如果没有安装Kapacitor请直接点击Skip

sdfjl



