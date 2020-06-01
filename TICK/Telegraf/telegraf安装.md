### 一. 下载和安装
```
wget https://dl.influxdata.com/telegraf/releases/telegraf_1.14.2-1_amd64.deb
sudo dpkg -i telegraf_1.14.2-1_amd64.deb
```

### 二. 配置(编辑/etc/telegraf/telegraf.conf文件)
```
...
[[outputs.influxdb]]
  urls = ["http://InfluxDB_data_node_IP:8086"] #修改为data node的ip和port
...
[[inputs.influxdb]] #删除此行的注释
...
```
### 三. 启动telegraf
```
$ systemctl start telegraf.service
$ systemctl status telegraf.service
● telegraf.service - The plugin-driven server agent for reporting metrics into InfluxDB
   Loaded: loaded (/lib/systemd/system/telegraf.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2020-05-13 03:41:09 UTC; 1min ago
     Docs: https://github.com/influxdata/telegraf
 Main PID: 21128 (telegraf)
    Tasks: 17 (limit: 19190)
   CGroup: /system.slice/telegraf.service
           └─21128 /usr/bin/telegraf -config /etc/telegraf/telegraf.conf -config-directory /etc/telegraf/telegraf.d
```

