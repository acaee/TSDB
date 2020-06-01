
### 基本条件
- 运行InfluxDB 1.7.10或更高版本的InfluxDB OSS实例
- 运行InfluxDB Enterpries 1.7.10或更高版本的InfluxDB Enterprise集群
- OSS实例与所有data node和meta node之间的网络可以通讯

**迁移需要执行以下操作**
- 删除现有InfluxDB Enterprise data node中的数据
- 将所有用户从OSS实例转移到InfluxDB Enterprise集群
- 需要OSS实例停机

### 迁移到InfluxDB Enterprise
完成以下内容：
1. 将InfluxDB升级到最新版本
2. 配置InfluxDB Enterprise meta node
3. 配置InfluxDB Enterprise data node
4. 在OSS实例上升级InfluxDB二进制文件
5. 将升级的OSS实例添加到InfluxDB Enterprise集群
6. 将现有data node添加到集群
7. 重新平衡集群