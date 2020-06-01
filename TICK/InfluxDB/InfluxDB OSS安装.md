### 一. 下载和安装
```
wget https://dl.influxdata.com/influxdb/releases/influxdb_1.7.10_amd64.deb
sudo dpkg -i influxdb_1.7.10_amd64.deb
```

### 二. 配置(编辑/etc/influxdb/influxdb.conf文件，更改以下路径)
```
...

[meta]
  dir = "/mnt/influxdb/meta"
  ...

...

[data]
  dir = "/mnt/influxdb/data"
  ...
wal-dir = "/mnt/influxdb/wal"
  ...

```
### 三. 目录权限
```
chown influxdb:influxdb /mnt/influx

```

### 四. 启动并创建数据库和保留策略

```
$ systemctl start influxdb.service
$ systemctl status influxdb.service
● influxdb.service - InfluxDB is an open-source, distributed, time series database
   Loaded: loaded (/lib/systemd/system/influxdb.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2020-04-07 11:30:06 UTC; 1 minutes ago
     Docs: https://docs.influxdata.com/influxdb/
 Main PID: 29654 (influxd)
    Tasks: 15 (limit: 19190)
   CGroup: /system.slice/influxdb.service
           └─29654 /usr/bin/influxd -config /etc/influxdb/influxdb.conf
$ influx
Connected to http://localhost:8086 version 1.7.10
InfluxDB shell version: 1.7.10
> create database metric
> CREATE RETENTION POLICY "180d.events" ON "metric" DURATION 180d REPLICATION 1 SHARD DURATION 7d
> SHOW RETENTION POLICIES ON "metric"
name        duration  shardGroupDuration replicaN default
----        --------  ------------------ -------- -------
autogen     0s        168h0m0s           1        true
180d.events 4320h0m0s 168h0m0s           1        false
```