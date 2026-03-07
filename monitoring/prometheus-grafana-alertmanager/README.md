# prometheus-grafana-alertmanager

两台虚拟机监控实验环境：

- host4: Prometheus / Grafana / Alertmanager / Nginx
- host1: node-exporter / mysqld-exporter / Tomcat JMX exporter

## 功能
- 主机 CPU / 内存监控
- MySQL / Tomcat 状态监控
- 邮件告警
- 恢复通知
- Nginx HTTPS 统一入口

## 访问
- Grafana: https://monitor.host4/grafana/
- Prometheus: https://monitor.host4/prometheus/
