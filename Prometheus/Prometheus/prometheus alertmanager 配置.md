### 一. alertmanager配置
#### 1. 编辑 `alertmanager.yml`文件
```
# 全局配置，设置邮箱等参数
global:
  resolve_timeout: 5m # 处理超时时间，默认为5min
  smtp_smarthost: 'smtp.exmail.qq.com:465' # 邮箱smtp服务器代理
  smtp_from: 'gaojianyue@sqhtech.com' # 发送邮箱名称
  smtp_auth_username: 'gaojianyue@sqhtech.com' # 邮箱名称
  smtp_auth_password: '*********' # 邮箱授权码
  smtp_require_tls: false
  

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 10m
  receiver: 'email'
receivers:
- name: 'email'
  email_configs:
  - to: 'gaojyue@163.com' # 接收报警的邮箱
    send_resolved: true
# templates: # 配置自定义邮箱内容格式，如不指定，则使用默认邮箱内容格式
#   - '/alertmanager/template/*.tmpl'
```

### 二. prometheus配置
#### 1. 在prometheus文件夹下新建rules文件夹
```
cd prometheus
mkdir rules
```
#### 2. 编写规则文件
```
cd rules
```
1. node_exporter.yml(用来监控主机和性能的规则文件)(已测试)
```
groups:
- name: 主机和硬件
  rules:
  # Host out of memory
  - alert: HostOutOfMemory
    expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100 < 80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host out of memory (instance {{ $labels.instance }})"
      description: "Node memory is filling up (< 10% left)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Host memory under memory pressure
  - alert: HostMemoryUnderMemoryPressure
    expr: rate(node_vmstat_pgmajfault[1m]) > 1000
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host memory under memory pressure (instance {{ $labels.instance }})"
      description: "The node is under heavy memory pressure. High rate of major page faults\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"
  
  # Host unusual network throughput in
  - alert: HostUnusualNetworkThroughputIn
    expr: sum by (instance) (irate(node_network_receive_bytes_total[2m])) / 1024 / 1024 > 100
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host unusual network throughput in (instance {{ $labels.instance }})"
      description: "Host network interfaces are probably receiving too much data (> 100 MB/s)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Host unusual network throughput out
  - alert: HostUnusualNetworkThroughputOut
    expr: sum by (instance) (irate(node_network_transmit_bytes_total[2m])) / 1024 / 1024 > 100
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host unusual network throughput out (instance {{ $labels.instance }})"
      description: "Host network interfaces are probably sending too much data (> 100 MB/s)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Host unusual disk read rate
  - alert: HostUnusualDiskReadRate
    expr: sum by (instance) (irate(node_disk_read_bytes_total[2m])) / 1024 / 1024 > 50
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host unusual disk read rate (instance {{ $labels.instance }})"
      description: "Disk is probably reading too much data (> 50 MB/s)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Host unusual disk write rate
  - alert: HostUnusualDiskWriteRate
    expr: sum by (instance) (irate(node_disk_written_bytes_total[2m])) / 1024 / 1024 > 50
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host unusual disk write rate (instance {{ $labels.instance }})"
      description: "Disk is probably writing too much data (> 50 MB/s)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Host out of disk space
  - alert: HostOutOfDiskSpace
    expr: (node_filesystem_avail_bytes{mountpoint="/rootfs"}  * 100) / node_filesystem_size_bytes{mountpoint="/rootfs"} < 10
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host out of disk space (instance {{ $labels.instance }})"
      description: "Disk is almost full (< 10% left)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Host disk will fill in 4 hours
  - alert: HostDiskWillFillIn4Hours
    expr: predict_linear(node_filesystem_free_bytes{fstype!~"tmpfs"}[1h], 4 * 3600) < 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host disk will fill in 4 hours (instance {{ $labels.instance }})"
      description: "Disk will fill in 4 hours at current write rate\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Host out of inodes
  - alert: HostOutOfInodes
    expr: node_filesystem_files_free{mountpoint ="/rootfs"} / node_filesystem_files{mountpoint ="/rootfs"} * 100 < 10
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host out of inodes (instance {{ $labels.instance }})"
      description: "Disk is almost running out of available inodes (< 10% left)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Host unusual disk read latency
  - alert: HostUnusualDiskReadLatency
    expr: rate(node_disk_read_time_seconds_total[1m]) / rate(node_disk_reads_completed_total[1m]) > 100
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host unusual disk read latency (instance {{ $labels.instance }})"
      description: "Disk latency is growing (read operations > 100ms)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Host unusual disk write latency
  - alert: HostUnusualDiskWriteLatency
    expr: rate(node_disk_write_time_seconds_total[1m]) / rate(node_disk_writes_completed_total[1m]) > 100
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host unusual disk write latency (instance {{ $labels.instance }})"
      description: "Disk latency is growing (write operations > 100ms)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Host high CPU load
  - alert: HostHighCpuLoad
    expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host high CPU load (instance {{ $labels.instance }})"
      description: "CPU load is > 80%\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Host context switching
  # 1000 context switches is an arbitrary number.
  # Alert threshold depends on nature of application.
  # Please read: https://github.com/samber/awesome-prometheus-alerts/issues/58
  - alert: HostContextSwitching
    expr: (rate(node_context_switches_total[5m])) / (count without(cpu, mode) (node_cpu_seconds_total{mode="idle"})) > 1000
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host context switching (instance {{ $labels.instance }})"
      description: "Context switching is growing on node (> 1000 / s)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Host swap is filling up
  - alert: HostSwapIsFillingUp
    expr: (1 - (node_memory_SwapFree_bytes / node_memory_SwapTotal_bytes)) * 100 > 80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host swap is filling up (instance {{ $labels.instance }})"
      description: "Swap is filling up (>80%)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Host SystemD service crashed
  - alert: HostSystemdServiceCrashed
    expr: node_systemd_unit_state{state="failed"} == 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host SystemD service crashed (instance {{ $labels.instance }})"
      description: "SystemD service crashed\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Host physical component too hot
  - alert: HostPhysicalComponentTooHot
    expr: node_hwmon_temp_celsius > 75
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host physical component too hot (instance {{ $labels.instance }})"
      description: "Physical hardware component too hot\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Host node overtemperature alarm
  - alert: HostNodeOvertemperatureAlarm
    expr: node_hwmon_temp_alarm == 1
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Host node overtemperature alarm (instance {{ $labels.instance }})"
      description: "Physical node temperature alarm triggered\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Host RAID array got inactive
  - alert: HostRaidArrayGotInactive
    expr: node_md_state{state="inactive"} > 0
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Host RAID array got inactive (instance {{ $labels.instance }})"
      description: "RAID array {{ $labels.device }} is in degraded state due to one or more disks failures. Number of spare drives is insufficient to fix issue automatically.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Host RAID disk failure
  - alert: HostRaidDiskFailure
    expr: node_md_disks{state="fail"} > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host RAID disk failure (instance {{ $labels.instance }})"
      description: "At least one device in RAID array on {{ $labels.instance }} failed. Array {{ $labels.md_device }} needs attention and possibly a disk swap\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Host kernel version deviations
  - alert: HostKernelVersionDeviations
    expr: count(sum(label_replace(node_uname_info, "kernel", "$1", "release", "([0-9]+.[0-9]+.[0-9]+).*")) by (kernel)) > 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host kernel version deviations (instance {{ $labels.instance }})"
      description: "Different kernel versions are running\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Host OOM kill detected
  - alert: HostOomKillDetected
    expr: increase(node_vmstat_oom_kill[30m]) > 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Host OOM kill detected (instance {{ $labels.instance }})"
      description: "OOM kill detected\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"
```
2. ceph_exporter.yml(监控ceph的规则文件)(未测试)
```
groups:
- name: Ceph
  rules:
  # Ceph State
  - alert: CephState
    expr: ceph_health_status != 0
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Ceph State (instance {{ $labels.instance }})"
      description: "Ceph instance unhealthy\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"
  
  # Ceph monitor clock skew
  - alert: CephMonitorClockSkew
    expr: abs(ceph_monitor_clock_skew_seconds) > 0.2
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Ceph monitor clock skew (instance {{ $labels.instance }})"
      description: "Ceph monitor clock skew detected. Please check ntp and hardware clock settings\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Ceph monitor low space
  - alert: CephMonitorLowSpace
    expr: ceph_monitor_avail_percent < 10
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Ceph monitor low space (instance {{ $labels.instance }})"
      description: "Ceph monitor storage is low.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Ceph OSD Down
  - alert: CephOsdDown
    expr: ceph_osd_up == 0
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Ceph OSD Down (instance {{ $labels.instance }})"
      description: "Ceph Object Storage Daemon Down\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Ceph high OSD latency
  - alert: CephHighOsdLatency
    expr: ceph_osd_perf_apply_latency_seconds > 10
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Ceph high OSD latency (instance {{ $labels.instance }})"
      description: "Ceph Object Storage Daemon latetncy is high. Please check if it doesn't stuck in weird state.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Ceph OSD low space
  - alert: CephOsdLowSpace
    expr: ceph_osd_utilization > 90
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Ceph OSD low space (instance {{ $labels.instance }})"
      description: "Ceph Object Storage Daemon is going out of space. Please add more disks.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Ceph OSD reweighted
  - alert: CephOsdReweighted
    expr: ceph_osd_weight < 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Ceph OSD reweighted (instance {{ $labels.instance }})"
      description: "Ceph Object Storage Daemon take ttoo much time to resize.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Ceph PG down
  - alert: CephPgDown
    expr: ceph_pg_down > 0
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Ceph PG down (instance {{ $labels.instance }})"
      description: "Some Ceph placement groups are down. Please ensure that all the data are available.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Ceph PG incomplete
  - alert: CephPgIncomplete
    expr: ceph_pg_incomplete > 0
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Ceph PG incomplete (instance {{ $labels.instance }})"
      description: "Some Ceph placement groups are incomplete. Please ensure that all the data are available.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Ceph PG inconsistant
  - alert: CephPgInconsistant
    expr: ceph_pg_inconsistent > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Ceph PG inconsistant (instance {{ $labels.instance }})"
      description: "Some Ceph placement groups are inconsitent. Data is available but inconsistent across nodes.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Ceph PG activation long
  - alert: CephPgActivationLong
    expr: ceph_pg_activating > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Ceph PG activation long (instance {{ $labels.instance }})"
      description: "Some Ceph placement groups are too long to activate.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Ceph PG backfill full
  - alert: CephPgBackfillFull
    expr: ceph_pg_backfill_toofull > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Ceph PG backfill full (instance {{ $labels.instance }})"
      description: "Some Ceph placement groups are located on full Object Storage Daemon on cluster. Those PGs can be unavailable shortly. Please check OSDs, change weight or reconfigure CRUSH rules.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Ceph PG unavailable
  - alert: CephPgUnavailable
    expr: ceph_pg_total - ceph_pg_active > 0
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Ceph PG unavailable (instance {{ $labels.instance }})"
      description: "Some Ceph placement groups are unavailable.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"
```
3. kubernetes_exporter.yml(监控Kubernetes的规则文件)(未测试)
```
groups:
- name: kubernetes报警指标
  rules:
  # Kubernetes Node ready
  - alert: KubernetesNodeReady
    expr: kube_node_status_condition{condition="Ready",status="true"} == 0
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Kubernetes Node ready (instance {{ $labels.instance }})"
      description: "Node {{ $labels.node }} has been unready for a long time\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes memory pressure
  - alert: KubernetesMemoryPressure
    expr: kube_node_status_condition{condition="MemoryPressure",status="true"} == 1
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Kubernetes memory pressure (instance {{ $labels.instance }})"
      description: "{{ $labels.node }} has MemoryPressure condition\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes disk pressure
  - alert: KubernetesDiskPressure
    expr: kube_node_status_condition{condition="DiskPressure",status="true"} == 1
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Kubernetes disk pressure (instance {{ $labels.instance }})"
      description: "{{ $labels.node }} has DiskPressure condition\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes out of disk
  - alert: KubernetesOutOfDisk
    expr: kube_node_status_condition{condition="OutOfDisk",status="true"} == 1
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Kubernetes out of disk (instance {{ $labels.instance }})"
      description: "{{ $labels.node }} has OutOfDisk condition\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes Job failed
  - alert: KubernetesJobFailed
    expr: kube_job_status_failed > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kubernetes Job failed (instance {{ $labels.instance }})"
      description: "Job {{$labels.namespace}}/{{$labels.exported_job}} failed to complete\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes CronJob suspended
  - alert: KubernetesCronjobSuspended
    expr: kube_cronjob_spec_suspend != 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kubernetes CronJob suspended (instance {{ $labels.instance }})"
      description: "CronJob {{ $labels.namespace }}/{{ $labels.cronjob }} is suspended\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes PersistentVolumeClaim pending
  - alert: KubernetesPersistentvolumeclaimPending
    expr: kube_persistentvolumeclaim_status_phase{phase="Pending"} == 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kubernetes PersistentVolumeClaim pending (instance {{ $labels.instance }})"
      description: "PersistentVolumeClaim {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }} is pending\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes Volume out of disk space
  - alert: KubernetesVolumeOutOfDiskSpace
    expr: kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes * 100 < 10
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kubernetes Volume out of disk space (instance {{ $labels.instance }})"
      description: "Volume is almost full (< 10% left)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes Volume full in four days
  - alert: KubernetesVolumeFullInFourDays
    expr: predict_linear(kubelet_volume_stats_available_bytes[6h], 4 * 24 * 3600) < 0
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Kubernetes Volume full in four days (instance {{ $labels.instance }})"
      description: "{{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }} is expected to fill up within four days. Currently {{ $value | humanize }}% is available.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes PersistentVolume error
  - alert: KubernetesPersistentvolumeError
    expr: kube_persistentvolume_status_phase{phase=~"Failed|Pending",job="kube-state-metrics"} > 0
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Kubernetes PersistentVolume error (instance {{ $labels.instance }})"
      description: "Persistent volume is in bad state\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes StatefulSet down
  - alert: KubernetesStatefulsetDown
    expr: (kube_statefulset_status_replicas_ready / kube_statefulset_status_replicas_current) != 1
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Kubernetes StatefulSet down (instance {{ $labels.instance }})"
      description: "A StatefulSet went down\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes HPA scaling ability
  - alert: KubernetesHpaScalingAbility
    expr: kube_hpa_status_condition{condition="false", status="AbleToScale"} == 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kubernetes HPA scaling ability (instance {{ $labels.instance }})"
      description: "Pod is unable to scale\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes HPA metric availability
  - alert: KubernetesHpaMetricAvailability
    expr: kube_hpa_status_condition{condition="false", status="ScalingActive"} == 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kubernetes HPA metric availability (instance {{ $labels.instance }})"
      description: "HPA is not able to colelct metrics\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes HPA scale capability
  - alert: KubernetesHpaScaleCapability
    expr: kube_hpa_status_desired_replicas >= kube_hpa_spec_max_replicas
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kubernetes HPA scale capability (instance {{ $labels.instance }})"
      description: "The maximum number of desired Pods has been hit\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes Pod not healthy
  - alert: KubernetesPodNotHealthy
    expr: min_over_time(sum by (namespace, pod) (kube_pod_status_phase{phase=~"Pending|Unknown|Failed"})[1h:]) > 0
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Kubernetes Pod not healthy (instance {{ $labels.instance }})"
      description: "Pod has been in a non-ready state for longer than an hour.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes pod crash looping
  - alert: KubernetesPodCrashLooping
    expr: rate(kube_pod_container_status_restarts_total[15m]) * 60 * 5 > 5
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kubernetes pod crash looping (instance {{ $labels.instance }})"
      description: "Pod {{ $labels.pod }} is crash looping\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes ReplicasSet mismatch
  - alert: KubernetesReplicassetMismatch
    expr: kube_replicaset_spec_replicas != kube_replicaset_status_ready_replicas
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kubernetes ReplicasSet mismatch (instance {{ $labels.instance }})"
      description: "Deployment Replicas mismatch\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes Deployment replicas mismatch
  - alert: KubernetesDeploymentReplicasMismatch
    expr: kube_deployment_spec_replicas != kube_deployment_status_replicas_available
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kubernetes Deployment replicas mismatch (instance {{ $labels.instance }})"
      description: "Deployment Replicas mismatch\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes StatefulSet replicas mismatch
  - alert: KubernetesStatefulsetReplicasMismatch
    expr: kube_statefulset_status_replicas_ready != kube_statefulset_status_replicas
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kubernetes StatefulSet replicas mismatch (instance {{ $labels.instance }})"
      description: "A StatefulSet has not matched the expected number of replicas for longer than 15 minutes.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes Deployment generation mismatch
  - alert: KubernetesDeploymentGenerationMismatch
    expr: kube_deployment_status_observed_generation != kube_deployment_metadata_generation
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Kubernetes Deployment generation mismatch (instance {{ $labels.instance }})"
      description: "A Deployment has failed but has not been rolled back.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes StatefulSet generation mismatch
  - alert: KubernetesStatefulsetGenerationMismatch
    expr: kube_statefulset_status_observed_generation != kube_statefulset_metadata_generation
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Kubernetes StatefulSet generation mismatch (instance {{ $labels.instance }})"
      description: "A StatefulSet has failed but has not been rolled back.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes StatefulSet update not rolled out
  - alert: KubernetesStatefulsetUpdateNotRolledOut
    expr: max without (revision) (kube_statefulset_status_current_revision unless kube_statefulset_status_update_revision) * (kube_statefulset_replicas != kube_statefulset_status_replicas_updated)
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Kubernetes StatefulSet update not rolled out (instance {{ $labels.instance }})"
      description: "StatefulSet update has not been rolled out.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes DaemonSet rollout stuck
  - alert: KubernetesDaemonsetRolloutStuck
    expr: kube_daemonset_status_number_ready / kube_daemonset_status_desired_number_scheduled * 100 < 100 or kube_daemonset_status_desired_number_scheduled - kube_daemonset_status_current_number_scheduled > 0
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Kubernetes DaemonSet rollout stuck (instance {{ $labels.instance }})"
      description: "Some Pods of DaemonSet are not scheduled or not ready\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes DaemonSet misscheduled
  - alert: KubernetesDaemonsetMisscheduled
    expr: kube_daemonset_status_number_misscheduled > 0
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Kubernetes DaemonSet misscheduled (instance {{ $labels.instance }})"
      description: "Some DaemonSet Pods are running where they are not supposed to run\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes CronJob too long
  - alert: KubernetesCronjobTooLong
    expr: time() - kube_cronjob_next_schedule_time > 3600
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kubernetes CronJob too long (instance {{ $labels.instance }})"
      description: "CronJob {{ $labels.namespace }}/{{ $labels.cronjob }} is taking more than 1h to complete.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes job completion
  - alert: KubernetesJobCompletion
    expr: kube_job_spec_completions - kube_job_status_succeeded > 0 or kube_job_status_failed > 0
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Kubernetes job completion (instance {{ $labels.instance }})"
      description: "Kubernetes Job failed to complete\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes API server errors
  - alert: KubernetesApiServerErrors
    expr: sum(rate(apiserver_request_count{job="apiserver",code=~"^(?:5..)$"}[2m])) / sum(rate(apiserver_request_count{job="apiserver"}[2m])) * 100 > 3
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Kubernetes API server errors (instance {{ $labels.instance }})"
      description: "Kubernetes API server is experiencing high error rate\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes API client errors
  - alert: KubernetesApiClientErrors
    expr: (sum(rate(rest_client_requests_total{code=~"(4|5).."}[2m])) by (instance, job) / sum(rate(rest_client_requests_total[2m])) by (instance, job)) * 100 > 1
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Kubernetes API client errors (instance {{ $labels.instance }})"
      description: "Kubernetes API client is experiencing high error rate\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes client certificate expires next week
  - alert: KubernetesClientCertificateExpiresNextWeek
    expr: apiserver_client_certificate_expiration_seconds_count{job="apiserver"} > 0 and histogram_quantile(0.01, sum by (job, le) (rate(apiserver_client_certificate_expiration_seconds_bucket{job="apiserver"}[5m]))) < 7*24*60*60
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kubernetes client certificate expires next week (instance {{ $labels.instance }})"
      description: "A client certificate used to authenticate to the apiserver is expiring next week.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes client certificate expires soon
  - alert: KubernetesClientCertificateExpiresSoon
    expr: apiserver_client_certificate_expiration_seconds_count{job="apiserver"} > 0 and histogram_quantile(0.01, sum by (job, le) (rate(apiserver_client_certificate_expiration_seconds_bucket{job="apiserver"}[5m]))) < 24*60*60
    for: 5m
    labels:
      severity: error
    annotations:
      summary: "Kubernetes client certificate expires soon (instance {{ $labels.instance }})"
      description: "A client certificate used to authenticate to the apiserver is expiring in less than 24.0 hours.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  # Kubernetes API server latency
  - alert: KubernetesApiServerLatency
    expr: histogram_quantile(0.99, sum(apiserver_request_latencies_bucket{verb!~"CONNECT|WATCHLIST|WATCH|PROXY"}) WITHOUT (instance, resource)) / 1e+06 > 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kubernetes API server latency (instance {{ $labels.instance }})"
      description: "Kubernetes API server has a 99th percentile latency of {{ $value }} seconds for {{ $labels.verb }} {{ $labels.resource }}.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"
```
#### 3. 编辑prometheus.yml文件
1. 增加alertmanager配置
```
# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - 192.168.0.83:9093
```
2. 引用规则文件
```
rule_files:
  - "/root/prometheus/rules/node_exporter.yml"
  - "/root/prometheus/rules/ceph_exporter.yml"
  - "/root/prometheus/rules/kubernetes_exporter.yml"
```