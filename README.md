# ops-lab

个人运维 / 交付实操实验仓库：

- Jenkins + Docker/Compose 自动化发布链
- Prometheus + Grafana + Alertmanager 监控闭环
- Ansible 主机初始化与网络切换实验

该仓库偏向个人实验环境下的运维实操记录与模块化整理。

---

## 仓库结构

### `ci/jenkins`
Jenkins 发布链相关实验。

当前包含：
- `Jenkinsfile`
- 基于 Dockerfile 的镜像构建
- Docker Compose 部署文件
- `systemd` 服务托管示例

适合查看的内容：
- 镜像构建与发布流程
- Jenkins 驱动部署
- Compose + systemd 的落地方式

### `monitoring/prometheus-grafana-alertmanager`
监控闭环实验。

当前包含：
- Prometheus 采集配置
- Grafana 数据源与面板 provisioning
- Alertmanager 告警配置
- Nginx HTTPS 访问配置

适合查看的内容：
- 监控组件编排
- 告警规则与通知链路
- 监控服务对外暴露方式

### `ansible/dockeinstall`
Ansible 初始化与 Docker 安装实验。

当前包含：
- APT 软件源切换
- 常用基础工具安装
- Docker 官方仓库配置
- Docker Engine / Compose / Buildx 安装
- Docker mirror 配置
- 可选系统初始化项（DNS / Swap / SSH 等）

### `ansible/netplan-toggle`
Ansible role 实现双网卡主机默认路由切换。

支持两种模式：
- `proxy`：默认路由走代理 / NAT 链路
- `direct`：默认路由走内网 / 直连链路

适合查看的内容：
- Netplan 模板化管理
- 默认路由切换
- 重启后模式保持与纠偏逻辑

---

## 技术栈

- Linux
- Docker / Docker Compose
- Jenkins
- Ansible
- Prometheus
- Grafana
- Alertmanager
- Nginx
- systemd

---

## 适合从哪里开始看

建议阅读顺序：

1. `ci/jenkins/README.md`
2. `monitoring/prometheus-grafana-alertmanager/README.md`
3. `ansible/README.md`

---

## 说明

该仓库主要用于个人实验与能力沉淀，重点放在：
- 自动化发布链路
- 运维部署与验证
- 监控闭环搭建
- 常见初始化与网络切换场景的脚本化处理
