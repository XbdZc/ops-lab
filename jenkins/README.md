# Jenkins + Harbor + host1 部署链

## 1. 简介

目标：

- 在 **host3** 上由 Jenkins 本机构建 Tomcat 镜像
- 推送到 **Harbor**
- 在 **host1** 上通过 `docker compose + systemd` 按 `TAG` 部署
- 通过 **host2** 的 HTTPS 入口做验收
- 通过容器内 `BUILD_INFO.txt` 做版本核验

自用留档

---

## 2. 当前目录结构

```text
jenkins/
├── deploy
│   ├── docker-compose.yml
│   └── ry-stack.service
├── Jenkinsfile
└── tomcat9
    └── Dockerfile
```

说明：

- `Jenkinsfile`
  - 当前发布链的流水线脚本
  - 这套逻辑已经真实跑通过
- `deploy/docker-compose.yml`
  - host1 使用的 Compose 文件
- `deploy/ry-stack.service`
  - host1 上用于托管 Compose 的 systemd 服务
- `tomcat9/Dockerfile`
  - Tomcat 镜像构建文件

------

## 3. 当前已验证的架构状态

- **host1**
  - 使用 `ry-stack.service` 托管 `docker compose`
  - `ry-tomcat` 已按 Harbor 镜像方式部署
  - 通过 `/opt/ry-stack/.env` 传递 `TAG`
- **host2**
  - Nginx HTTPS 反代 host1 业务入口
- **host3**
  - Jenkins + Harbor
  - Jenkins 已具备 Docker 权限
  - Pipeline 内 Harbor 登录已验证可用
- **host4**
  - Prometheus + Grafana + Alertmanager + Nginx HTTPS

------

## 4. 做了什么

1. Jenkins 在 host3 本机构建镜像
2. Jenkins 登录 Harbor
3. Jenkins 推送镜像到 Harbor
4. Jenkins 远程连接 host1
5. host1 写入 `/opt/ry-stack/.env`
6. host1 按指定 tag `docker pull`
7. host1 `systemctl restart ry-stack`
8. 通过 host2 入口验收
9. 进入容器读取 `BUILD_INFO.txt` 核验版本

**host3 build/push -> host1 pull/deploy -> host2 验收 -> 容器内版本核验**

------

## 5. 已经踩过并验证过的关键点

### 5.1 Harbor 证书信任

host1 访问 Harbor 时，曾遇到过：

```bash
x509: certificate signed by unknown authority
```

最终可用做法已经验证：

```bash
/etc/docker/certs.d/harbor.host3/ca.crt
/usr/local/share/ca-certificates/harbor.host3.crt
sudo update-ca-certificates
sudo systemctl restart docker
```

### 5.2 Jenkins 用户 Docker 权限

Jenkins 最开始不能直接访问 Docker：

```bash
permission denied while trying to connect to the docker API at unix:///var/run/docker.sock
```

最终可用做法已经验证：

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### 5.3 Compose 的 TAG 传递

`docker-compose.yml` 使用 Harbor 镜像时，如果写成：

```yaml
image: harbor.host3/ruoyi/ry-tomcat:${TAG:-latest}
```

那么部署阶段必须显式给 `TAG`，否则 Compose 会回退到 `latest`。

当前已验证可用的方式是：

```bash
/opt/ry-stack/.env
```

内容示例：

```bash
TAG=11
```

### 5.4 入口验收不能只看 restart 成功

实际验证过：

- `systemctl restart ry-stack` 成功
- 不代表业务已经 ready
- Java/Tomcat 冷启动阶段，入口可能先出现 502
- 因此 Pipeline 里保留了重试探测

------

## 6. 当前文件对应关系

### `Jenkinsfile`

负责：

- Harbor 登录
- 本机构建镜像
- Push Harbor
- 远程写 `.env`
- Pull 指定 tag
- 重启 `ry-stack`
- 入口验收
- 容器内版本核验

### `deploy/docker-compose.yml`

负责：

- host1 上的容器编排
- `ry-tomcat` 使用 Harbor 镜像 + `${TAG:-latest}`
- 保留现有挂载、JMX、网络、端口等配置

### `deploy/ry-stack.service`

负责：

- 由 systemd 托管 Compose
- 通过 `systemctl start/stop/restart/status ry-stack` 管理业务栈

### `tomcat9/Dockerfile`

负责：

- 构建业务使用的 Tomcat 镜像

------

## 7. 当前实际使用方式

### 7.1 Jenkins 侧

当前跑通的是：

- Jenkins Job 使用 **Pipeline script**
- `Jenkinsfile` 先作为仓库留档 / 自用备份

说明：

- 这份 `Jenkinsfile` 内容已经真实跑通过
- 但当前实验里，**没有**真正切成 `Pipeline script from SCM`
- 如果后面要改成从仓库直接读取，需要先配置真实 SCM

### 7.2 host1 侧

host1 当前依赖：

- `deploy/docker-compose.yml`
- `deploy/ry-stack.service`
- `/opt/ry-stack/.env`

------

## 8. 最关键的验收命令

### 8.1 看当前部署 tag

```bash
cat /opt/ry-stack/.env
```

### 8.2 看服务状态

```bash
systemctl status ry-stack --no-pager -l
```

### 8.3 看入口是否正常

```bash
curl -kI https://app.host2/login
```

返回 `200` 或 `302` 视为通过。

### 8.4 看容器内版本信息

```bash
docker exec ry-tomcat sh -c 'cat /opt/tomcat/webapps/ROOT/BUILD_INFO.txt'
```

------

## 9. 常用排障命令

### systemd / compose

```bash
systemctl status ry-stack --no-pager -l
journalctl -u ry-stack -n 50 --no-pager
cd /opt/ry-stack && docker compose config --environment
cd /opt/ry-stack && docker compose config | sed -n '/ry-tomcat:/,/^[^ ]/p'
```

### docker / Harbor

```bash
docker login harbor.host3
docker pull harbor.host3/ruoyi/ry-tomcat:<tag>
docker image ls | grep 'harbor.host3/ruoyi/ry-tomcat'
```

### Jenkins 用户权限

```bash
sudo -u jenkins -H bash -lc 'id && docker version'
getent group docker
systemctl status jenkins --no-pager -l
```

### 入口与端口

```bash
curl -kI https://app.host2/login
ss -lntp
tail -n 50 /var/log/nginx/error.log
```

