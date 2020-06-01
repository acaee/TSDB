#### 一. prometheus.yml配置

```
# my global config
global:
  scrape_interval: 15s # 设置抓取(pull)时间间隔，默认是1m
  evaluation_interval: 15s # 设置rules评估时间间隔，默认是1m
  # scrape_timeout is set to the global default (10s).

# 告警管理配置，默认配置
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - 172.20.4.111:9093 # 这里修改为 alertmanagers 的地址

# 加载rules，并根据设置的时间间隔定期评估
rule_files:
# - "first_rules.yml"
# - "second_rules.yml"
  - "/data0/prometheus/prometheus_server/rules/node_down.yml"                 # 实例存活报警规则文件
  - "/data0/prometheus/prometheus_server/rules/memory_over.yml"               # 内存报警规则文件
  - "/data0/prometheus/prometheus_server/rules/disk_over.yml"                 # 磁盘报警规则文件
  - "/data0/prometheus/prometheus_server/rules/cpu_over.yml"                  # cpu报警规则文件

# 抓取(pull)，即监控目标配置
# 默认只有主机本身的监控配置
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    # 可覆盖全局配置设置的抓取间隔，由15秒重写成5秒。
    scrape_interval: 10s

    static_configs:
      - targets: ['localhost:9090', 'localhost:9100']

  - job_name: 'DMC_HOST'
    file_sd_configs:
      - files: ['./hosts.json']  
      # 被监控的主机，可以通过static_configs罗列所有机器，这里通过file_sd_configs参数加载文件的形式读取
      # 被监控的主机，可以json或yaml格式书写，我这里以json格式书写，target里面写监控机器的ip，labels非必须，可以由你自己定
```

#### 2. 添加规则文件rules
```
vim node_down.yml 
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

```
vim cpu_over.yml 
groups:
- name: CPU报警规则
  rules:
  - alert: CPU使用率告警
    expr: 100 - (avg by (instance)(irate(node_cpu_seconds_total{mode="idle"}[1m]) )) * 100 > 90
    for: 1m
    labels:
      user: prometheus
      severity: warning
    annotations:
      description: "服务器: CPU使用超过90%！(当前值: {{ $value }}%)"
      summary: "机器 {{ $labels.instance }} CPU使用超过90% "
```

```
vim memory_over.yml 
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
      summary: "机器 {{ $labels.instance }} 内存使用超过80%
```

```
vim disk_over.yml 
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
```


#### 三. alertmanager.yml 配置文件设置
```
global:
  resolve_timeout: 5m #处理超时时间，默认为5min
  smtp_smarthost: 'mail.epaylinks.cn:465' # 邮箱smtp服务器代理
  smtp_from: 'zabbix@epaylinks.cn' # 发送邮箱名称
  smtp_auth_username: 'zabbix@epaylinks.cn' # 邮箱名称
  smtp_auth_password: 'Epaylinks@2018' # 邮箱密码或授权码
  smtp_require_tls: false
templates:
  - '/alertmanager/template/*.tmpl'
route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 10m
  receiver: default-receiver
receivers:
- name: 'default-receiver'
  email_configs:
  - to: 'linzhuangrong@epaylinks.cn'
    html: ‘{{ template "alert.html" . }}‘
    headers: { Subject: "[WARN] prometheus 监控报警邮件邮件告警" }
```

#### 四. 定制化邮件告警html模板文件(alert.tmpl)
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
You have new mail in /var/spool/mail/root
```