### 一. 安装InfluxDB Enterprise

#### 1. 安装meta node
##### （1）概述
InfluxDB Enterprise在您的基础架构和管理UI上提供了高度可扩展的集群（通过Chronograf来使用集群。Production Installation流程是为希望在生产环境中部署InfluxDB Enterprise的用户而设计的。以下步骤可帮助您启动并运行InfluxDB Enterprise集群的第一个基本组件为：meta nodes。
##### （2）需求配置
1.  meta nodes数量必须为3个及以上
2.  meta nodes 个数必须为奇数个
3.  不可以在一个实例节点上安装

##### （3）meta配置
1. 修改/etc文件（将服务器的主机名和IP地址添加到每个群集服务器的/etc/hosts 文件中）
```
<Meta_1_IP> enterprise-meta-01
<Meta_2_IP> enterprise-meta-02
<Meta_3_IP> enterprise-meta-03
```
2. 下载并安装服务
```
wget https://dl.influxdata.com/enterprise/releases/influxdb-meta_1.7.8-c1.7.8_amd64.deb
sudo dpkg -i influxdb-meta_1.7.8-c1.7.8_amd64.deb
```
3. 编辑配置文件
```
vim /etc/influxdb/influxdb-meta.conf
```
- 取消hostname注释并设置为meta node的完整主机名
- `[meta]`下的`internal-shared-secret`取消注释，并为其设置密码，此值在所有meta node必须相同，并且与data node配置文件中`[meta]`下的`meta-internal-shared-secret`完全一致
- 设置license-key或者license-path（license-key和license-path只能使用一个）
- [meta] dir = "/var/lib/influxdb/meta" #集群元数据的存储目录
4. 启动meta node服务
`systemctl start influxdb-meta`

##### （4）将meta node加入集群
```
influxd-ctl add-meta enterprise-meta-01:8091
influxd-ctl add-meta enterprise-meta-02:8091
influxd-ctl add-meta enterprise-meta-03:8091
```
```
influxd-ctl show
```
预期结果为：
```
Data Nodes
==========
ID      TCP Address      Version

Meta Nodes
==========
TCP Address               Version
enterprise-meta-01:8091   1.7.8-c1.7.8
enterprise-meta-02:8091   1.7.8-c1.7.8
enterprise-meta-03:8091   1.7.8-c1.7.8
```
#### 1. 安装 data node
##### （1）概述
InfluxDB Enterprise在基础架构上提供了高度可扩展的集群，以及用于处理集群的管理UI。下一步将使您开始使用InfluxDB Enterprise集群的第二个基本组件：data node。

##### （2）将服务器的主机名和IP地址添加到每个群集服务器的/etc/hosts 文件中
```
<Data_1_IP> enterprise-data-01
<Data_2_IP> enterprise-data-02
```
##### （3）data node配置
1. 下载和安装服务
- Ubuntu和Debain（64）
```
wget https://dl.influxdata.com/enterprise/releases/influxdb-data_1.7.8-c1.7.8_amd64.deb
sudo dpkg -i influxdb-data_1.7.8-c1.7.8_amd64.deb
```
- RedHat和CentOS（64）
```
wget https://dl.influxdata.com/enterprise/releases/influxdb-data-1.7.8_c1.7.8.x86_64.rpm
sudo yum localinstall influxdb-data-1.7.8_c1.7.8.x86_64.rpm
```
2. 编辑配置
```
vim /etc/influxdb/influxdb.conf
```
- 取消hostname注释并设置为data node的完整主机名
- `[meta]`下的`meta-internal-shared-secret`取消注释，并为其设置密码，此值在所有data node必须相同，并且与meta node配置文件中`[meta]`下的`internal-shared-secret`完全一致
- 设置license-key或者license-path（license-key和license-path只能使用一个）
- [meta] dir = "/var/lib/influxdb/meta" #集群元数据的存储目录
- [data] dir = "/var/lib/influxdb/data" 用于存储TSM文件的目录
- [data] wal-dir = "/var/lib/influxdb/wal" 用于存储WAL（write）文件的目录
3. 启动服务
```
systemctl start influxdb
```
4. 验证服务
- 输入以下命令，检查程序是否正在运行
```
ps aux | grep -v grep | grep influxdb
```
将输入以下内容
```
influxdb  2706  0.2  7.0 571008 35376 ? Sl  15:37   0:16 /usr/bin/influxd -config /etc/influxdb/influxdb.conf
```
##### （4） 将data node加入集群
```
influxd-ctl add-data enterprise-data-01:8088
influxd-ctl add-data enterprise-data-02:8088
```
预期的输出为：
```
Added data node y at enterprise-data-0x:8088
```
最后运行`influxd-ctl show`,输出结果为一下内容则表示成功：
```
Data Nodes
==========
ID      TCP Address             Version
4       enterprise-data-01:8088 1.7.8-c1.7.8
5       enterprise-data-02:8088 1.7.8-c1.7.8

Meta Nodes
==========
TCP Address             Version
enterprise-meta-01:8091 1.7.8-c1.7.8
enterprise-meta-02:8091 1.7.8-c1.7.8
enterprise-meta-03:8091 1.7.8-c1.7.8

```
##### 集群安装完成！


### 二. Nginx

1. proxy.conf 文件
```
upstream influxdb_write {
server data_node_01:8086; 
server data_node_02:8086;
}
upstream influxdb_read {
server data_node_01:8086;
server data_node_02:8086;
}
server {
	listen 90;
	#编码格式
        charset utf-8;
        
        #代理配置参数
        proxy_connect_timeout 180;
        proxy_send_timeout 180;
        proxy_read_timeout 180;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarder-For $remote_addr;
	location / {
	proxy_pass http://influxdb_write/api/v1/prom/write?db=metrics&rp=autogen;
	
	}
}
server {
	listen 91;
	 #编码格式
        charset utf-8;

        #代理配置参数
        proxy_connect_timeout 180;
        proxy_send_timeout 180;
        proxy_read_timeout 180;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarder-For $remote_addr;

	location / {
	proxy_pass http://influxdb_read/api/v1/prom/read?db=metrics&rp=autogen;

	}
}
```
2. nginx.conf 主配置文件
```
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;



# Settings for a TLS enabled server.
#
#    server {
#        listen       443 ssl http2 default_server;
#        listen       [::]:443 ssl http2 default_server;
#        server_name  _;
#        root         /usr/share/nginx/html;
#
#        ssl_certificate "/etc/pki/nginx/server.crt";
#        ssl_certificate_key "/etc/pki/nginx/private/server.key";
#        ssl_session_cache shared:SSL:1m;
#        ssl_session_timeout  10m;
#        ssl_ciphers HIGH:!aNULL:!MD5;
#        ssl_prefer_server_ciphers on;
#
#        # Load configuration files for the default server block.
#        include /etc/nginx/default.d/*.conf;
#
#        location / {
#        }
#
#        error_page 404 /404.html;
#            location = /40x.html {
#        }
#
#        error_page 500 502 503 504 /50x.html;
#            location = /50x.html {
#        }
#    }

}
```


### 三. Prometheus

##### （1）注册Prometheus服务
```
vim /lib/systemd/system/prometheus.service
```
输入以下内容：
```
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/
After=network.target

[Service]
Type=simple
User=root
ExecStart=/root/prometheus/prometheus --config.file=/root/prometheus/prometheus.yml --web.enable-lifecycle
ExecStop=/usr/bin/killall -9 prometheus



[Install]
WantedBy=multi-user.target
```
##### （2）更新服务配置
```
systemctl daemon-reload
```


##### （3）Prometheus配置
```
scrape_configs:
  - job_name: 'federate'
    scrape_interval: 15s
    honor_labels: true
    metrics_path: ''
    params:
      'match[]':
        - '{job=~"prometheus.*"}'
        - '{job="agent_name_str_1"}'
        - '{job="agent_name_str_2"}'
    static_configs:
      - targets:
        - '192.168.0.66:9091'
        - '192.168.0.67:9091'
        - '192.168.0.68:9091'
        
remote_write:
        #- url: "http://192.168.0.81:8086/api/v1/prom/write?db=metrics&rp=autogen"
        - url: "http://VRRP:[nginx_write port]"

remote_read:
        #- url: "http://192.168.0.81:8086/api/v1/prom/read?db=metrics&rp=autogen"
        - url: "http://VRRP:[nginx_read port]"
          read_recent: true
```
### 四.安装Grfana
```
sudo apt-get install -y adduser libfontconfig1
wget https://dl.grafana.com/oss/release/grafana_6.7.3_amd64.deb
sudo dpkg -i grafana_6.7.3_amd64.deb
```

### 四. Prometheus-keepalived 高可用

##### （1）MASTER 配置
1. keepalived.conf
```
vrrp_script check_prometheus {
    script "/etc/keepalived/check_prometheus.sh"
    interval 2
    weight -60
    fall 2
    rise 1
}

vrrp_instance VI_prometheus {
    state BACKUP
    nopreempt
    interface ens160
    virtual_router_id 50
    priority 100
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    unicast_src_ip 192.168.0.81
    unicast_peer {
        192.168.0.83
    }

    virtual_ipaddress {
        192.168.0.182
    }
    track_script {
        check_prometheus
    }

    notify_backup "/etc/keepalived/stop_prometheus.sh"
    notify_master "/etc/keepalived/start_prometheus.sh"
}
```
2. check_prometheus.sh
```
#!/bin/bash

VIP=192.168.0.182

IP=`ip addr | grep $VIP | awk  '{ print $2 }' | awk -F/ '{ print $1 }'`

if [[ $IP = $VIP ]]; then

        STATUS=`ps -C prometheus  --no-header|wc -l`

        if [ $STATUS -eq 0 ]; then

                exit 1

        else
                exit 0

        fi

else

        exit 0

fi
```
3. start_prometheus.sh
```
#!/bin/bash
systemctl start prometheus.service
```
4. stop_prometheus.sh
```
#!/bin/bash
systemctl stop prometheus.service
```

##### （1）BUCKUP 配置
1. keepalived.conf
```
vrrp_script check_prometheus {
    script "/etc/keepalived/check_prometheus.sh"
    interval 2
}

vrrp_instance VI_prometheus {
    state BACKUP
    #nopreempt
    interface ens160
    virtual_router_id 50
    priority 90
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    unicast_src_ip 192.168.0.83
    unicast_peer {
        192.168.0.81
    }

    virtual_ipaddress {
        192.168.0.182
    }
    track_script {
        check_prometheus
    }

    notify_backup "/etc/keepalived/stop_prometheus.sh"
    notify_master "/etc/keepalived/start_prometheus.sh"

}
```
2. check_prometheus.sh
```
#!/bin/bash

VIP=192.168.0.182

IP=`ip addr | grep $VIP | awk  '{ print $2 }' | awk -F/ '{ print $1 }'`

if [[ $IP = $VIP ]]; then

        STATUS=`ps -C prometheus  --no-header|wc -l`

        if [ $STATUS -eq 0 ]; then

                exit 1

        else
                exit 0

        fi

else

        exit 0

fi
```
3. start_prometheus.sh
```
#!/bin/bash
systemctl start prometheus.service
```
4. stop_prometheus.sh
```
#!/bin/bash
systemctl stop prometheus.service
```

### 五. Nginx-keepalived 高可用
##### （1）MASTER
1. keepalived.conf
```
vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight -60
    fall 2
    rise 1
}

vrrp_instance VI_nginx {
    state BACKUP
    nopreempt
    interface ens160
    virtual_router_id 51
    priority 100
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    unicast_src_ip 192.168.0.81
    unicast_peer {
        192.168.0.83
    }

    virtual_ipaddress {
        192.168.0.183
    }
    track_script {
        check_nginx
    }
}
```


##### （1）BACKUP
1. keepalived.conf
```
vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
}

vrrp_instance VI_nginx {
    state BACKUP
    #nopreempt
    interface ens160
    virtual_router_id 50
    priority 90
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    unicast_src_ip 192.168.0.83
    unicast_peer {
        192.168.0.81
    }

    virtual_ipaddress {
        192.168.0.182
    }
    track_script {
        check_nginx
    }

    notify_backup "/etc/keepalived/start_nginx.sh"
    notify_master "/etc/keepalived/start_nginx.sh"

}
```

2. check_nginx.sh
```
#!/bin/bash

STATUS=`ps -C nginx  --no-header|wc -l`

if [ $STATUS -eq 0 ]; then

        # systemctl restart nginx

        STATUS=`ps -C nginx  --no-header|wc -l`

        if [ $STATUS -eq 0 ]; then

                exit 1
        fi

else
        exit 0

fi
```
3. start_nginx.sh
```
#!/bin/bash
systemctl start nginx
```

### 常见问题：
#### 1. grafana配置altermanager
1. 配置etc/grafana/grafana.ini文件
```
apt-get install sendmail
vim etc/grafana/grafana.ini 
[smtp]
enabled = true
host = smtp.exmail.qq.com:465
user = *****@***.com
# If the password contains # or ; you have to wrap it with triple quotes. Ex """#password;"""
password = ********
;cert_file =
;key_file =
;skip_verify = false
from_address = *****@***.com
;from_name = Grafana
# EHLO identity in SMTP dialog (defaults to instance_name)
;ehlo_identity = dashboard.example.com

[emails]
;welcome_email_on_sign_up = false
;templates_pattern = emails/*.html
```
2. 在表上配置报警
- 选择图表上的Alter，配置报警规则和发送的地址、邮箱等
- Altering > Alter Rules可以启用和禁用报警规则


#### 4. InfluxDB如果不能启动

检查这些目录，如果未创建，则创建文件夹并授予influxdb:influxdb权限
```
 [meta] dir = "/var/lib/influxdb/meta"
 
 [data] dir = "/var/lib/influxdb/data"
 
 [data] wal-dir = "/var/lib/influxdb/wal"
 
 [hinted-handoff] dir = "/var/lib/influxdb/hh"
```
 
#### 5. prometheus配置altermanager

1. altermanager.yml
```
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.exmail.qq.com:465'
  smtp_from: '******@***.com'
  smtp_auth_username: '******@***.com'
  smtp_auth_password: '******'
  smtp_require_tls: false
templates:
  - '/root/alertmanager-0.20.0.linux-amd64/template/*.tmpl'
route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 10m
  receiver: 'default-receiver'
receivers:
- name: 'default-receiver'
  email_configs:
  - to: '******@***.com'
    html: ‘{{ template "alert.html" . }}‘
    headers: { Subject: "[WARN] prometheus 监控报警邮件邮件告警" }
```
2. alert.tmpl（样式模板）
```
{{ define "alert.html" }}
{{ range .Alerts }}
<table border="1">
  <tr>
    <th> 告警通知 </th>
    <th>  prometheus监控告警通知 </th>
  </tr>
  <tr>
    <td> 告警级别 </td>
    <td>{{ .Labels.severity }}</td>
  </tr>

  <tr>
    <td> 告警类型 </td>
    <td>{{ .Labels.alertname }}</td>
  </tr>
  <tr>
    <td> 故障主机 </td>
    <td>{{ .Labels.instance }}</td>
  </tr>

  <tr>
    <td> 告警主题 </td>
    <td>{{ .Annotations.summary }}</td>
  </tr>

  <tr>
    <td> 告警详情 </td>
    <td>{{ .Annotations.description }}</td>
  </tr>

  <tr>
    <td> 触发时间 </td>
    <td>{{ .StartsAt.Format "2006-01-02 15:04:05" }}</td>
  </tr>

</table>
{{ end }}
{{ end }}
```
3. prometheus.yml配置增加以下内容
```
# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - 192.168.0.83:9093
# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  - "/root/prometheus/rules/node_down.yml"                 # 实例存活报警规则文件
  - "/root/prometheus/rules/memory_over.yml"               # 内存报警规则文件
  - "/root/prometheus/rules/disk_over.yml"                 # 磁盘报警规则文件
  - "/root/prometheus/rules/cpu_over.yml"                  # cpu报警规则文件
```
4. 规则文件编写
```
cat cpu_over.yml disk_over.yml memory_over.yml node_down.yml 
groups:
- name: CPU报警规则
  rules:
  - alert: CPU使用率告警
    expr: 100 - ((avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[1m])))) * 100 > 90
    for: 1m
    labels:
      user: prometheus
      severity: warning
    annotations:
      description: "服务器: CPU使用超过90%！(当前值: {{ $value }}%)"
      summary: "机器 {{ $labels.instance }} CPU使用超过90% "
groups:
- name: 磁盘报警规则
  rules:
  - alert: 磁盘使用率告警
    expr: (node_filesystem_size_bytes - node_filesystem_avail_bytes) / node_filesystem_size_bytes * 100 > 80
    for: 1m
    labels:
      user: prometheus
      severity: warning
    annotations:
      description: "服务器: 磁盘设备: 使用超过80%！(挂载点: {{ $labels.mountpoint }} 当前值: {{ $value }}%)"
      summary: "机器 {{ $labels.instance }} 磁盘使用超过80%"
groups:
- name: 内存报警规则
  rules:
  - alert: 内存使用率告警
    expr: (node_memory_MemTotal_bytes - (node_memory_MemFree_bytes+node_memory_Buffers_bytes+node_memory_Cached_bytes )) / node_memory_MemTotal_bytes * 100 > 80
    for: 1m
    labels:
      user: prometheus
      severity: warning
    annotations:
      description: "服务器: 内存使用超过80%！(当前值: {{ $value }}%)"
      summary: "机器 {{ $labels.instance }} 内存使用超过80%"
groups:
- name: 实例存活告警规则
  rules:
  - alert: 实例存活告警
    expr: up == 0
    for: 1m
    labels:
      user: prometheus
      severity: warning
    annotations:
      description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minutes."
      summary: "机器 {{ $labels.instance }} 挂了"
```

