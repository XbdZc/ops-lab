# ops-lab

个人运维实验项目仓库。

## 目录
- monitoring/prometheus-grafana-alertmanager
  - Prometheus + Grafana + Alertmanager + Nginx HTTPS 监控闭环实验
- ansible/dockerinstall 
  - 用于在 Ubuntu 主机上完成以下初始化与 Docker 安装工作：
    - 替换 APT 软件源, 支持 aliyun / 华为 / 清华 / ustc / official
    - 安装常用基础工具
    - 配置 Docker 官方仓库（当前使用阿里云 Docker 镜像）
    - 安装 Docker Engine / Compose / Buildx
    - 配置 Docker registry mirror
    - 可选配置 DNS
    - 可选配置 Swap
    - 可选配置 SSH（root 登录 / 密码认证）
