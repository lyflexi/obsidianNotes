 监控案例落地：基于 Prometheus + Grafana
```shell
#安装prometheus:时序数据库
docker run -p 9090:9090 -d \
-v pc:/etc/prometheus \
prom/prometheus

#安装grafana；默认账号密码 admin:admin
docker run -d --name=grafana -p 3000:3000 grafana/grafana
```
![[Pasted image 20240130113027.png]]
springboot应用导入依赖：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <version>1.10.6</version>
</dependency>
```
配置信息：
```yaml
management:
  endpoints:
    web:
      exposure: #暴露所有监控的端点
        include: '*'
```
本地测试访问： http://localhost:9999/actuator/prometheus 验证，返回 prometheus 格式的所有指标
# 本地应用打包
接下来把springboot打成jar包，也部署在线上
```shell
#安装上传工具
yum install lrzsz

#安装openjdk
# 下载openjdk
wget https://download.oracle.com/java/17/latest/jdk-17_linux-x64_bin.tar.gz

mkdir -p /opt/java
tar -xzf jdk-17_linux-x64_bin.tar.gz -C /opt/java/
sudo vi /etc/profile
#加入以下内容
export JAVA_HOME=/opt/java/jdk-17.0.7
export PATH=$PATH:$JAVA_HOME/bin

#环境变量生效
source /etc/profile


# 后台启动java应用
nohup java -jar boot3-14-actuator-0.0.1-SNAPSHOT.jar > output.log 2>&1 &

```
测试线上springboot应用启动情况，确认可以访问到： http://8.130.32.70:9999/actuator/prometheus
# 配置 Prometheus
配置 Prometheus 拉取springboot应用的actuator数据
```yaml
## 修改 prometheus.yml 配置文件
scrape_configs:
#job_name随意起
  - job_name: 'spring-boot-actuator-exporter'
    metrics_path: '/actuator/prometheus' #指定抓取的路径
    static_configs:
      - targets: ['8.130.32.70:9999']
        labels:
          nodename: 'app-demo'
```
# 配置 Grafana 监控面板

- 添加数据源，指向我们自己部署的Prometheus
- 添加面板。可去 dashboard 市场找一个自己喜欢的面板，https://grafana.com/grafana/dashboards/?plcmt=footer