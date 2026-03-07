# Ansible Docker Install Playbook

用于在 Ubuntu 主机上完成以下初始化与 Docker 安装工作：

- 替换 APT 软件源
- 安装常用基础工具
- 配置 Docker 官方仓库（当前使用阿里云 Docker 镜像）
- 安装 Docker Engine / Compose / Buildx
- 配置 Docker registry mirror
- 可选配置 DNS
- 可选配置 Swap
- 可选配置 SSH（root 登录 / 密码认证）

## 当前已验证环境

已完成验证：

- Ubuntu 22.04
- Ubuntu 24.04

已验证内容：

- APT 源替换
- Docker 仓库配置
- Docker 安装
- Docker registry mirror 配置
- bash-completion
- Swap 可选开关
- DNS 可选开关
- SSH 可选开关
- 幂等性（二次执行 `changed=0`）

## 关键开关说明

> 当前 `docker_repo` role 仅支持 Ubuntu。  
> 如果目标系统是 Debian，APT 源替换 role 可以工作，但 Docker 仓库 role 还需要单独适配。

### apt_remove_extra_sources
是否删除 `/etc/apt/sources.list.d/` 下的额外源文件。

- Ubuntu 22.04：通常不开也能正常使用
- Ubuntu 24.04：建议开启，否则系统可能继续使用默认的 `ubuntu.sources`，导致 APT 混用官方源和镜像源

注意：如果目标机器依赖其他第三方仓库，请谨慎开启。

### apt_with_src
是否写入 `deb-src` 源。

默认关闭即可。仅在以下场景建议开启：

- 需要执行 `apt source`
- 需要执行 `apt build-dep`
- 需要在目标机上编译 Debian/Ubuntu 软件
- 需要排查源码级问题

普通 Docker 安装和日常包安装不需要开启。

### common_manage_ssh
是否由 playbook 管理 SSH 配置。

开启后会修改：

- `PermitRootLogin`
- `PasswordAuthentication`
- 可选覆盖 cloud-init 下发的 SSH 配置

建议仅在测试环境或受控内网环境开启。

### common_manage_swap
是否创建 `/swapfile`。

适合内存较小、构建/安装易 OOM 的机器。  
如果系统本身已有其他 swap（如 `/swap.img`），当前逻辑不会自动移除原有 swap，可能并存。

### manage_dns_config
是否管理 `systemd-resolved` 的 DNS 配置。

开启后会修改：

- `/etc/systemd/resolved.conf`
- `/etc/resolv.conf`

建议在明确知道当前系统使用 `systemd-resolved` 时开启。

## 目录结构

```
playbook/
├── hosts
├── site.yml
├── vars/
│   └── main.yml
└── roles/
    ├── apt_source/
    ├── common_init/
    ├── docker_repo/
    ├── docker_install/
    ├── mirror_config/
    ├── completion/
    └── dns_config/
```

## 变量说明

主要变量位于：

```
playbook/vars/main.yml
```

### APT 源相关

- `apt_source_name`：选择 APT 镜像源，支持 `aliyun` / `huawei` / `tsinghua` / `ustc` / `official`
  - apt_source_name：选择 APT 镜像源，支持 aliyun / 华为 / 清华 / ustc / official
- `apt_with_src`：是否同时写入 `deb-src`
- `apt_remove_extra_sources`：是否删除 `/etc/apt/sources.list.d/` 下额外源文件
  - Ubuntu 24.04 建议设为 `true`
  - 如果机器有其他第三方仓库，请谨慎开启

### 基础初始化相关

- `common_timezone`：系统时区
- `common_manage_ssh`：是否由 playbook 管理 SSH 配置
- `common_manage_swap`：是否创建 `/swapfile`
- `swap_file_size_mb`：swap 文件大小（MB）

### SSH 相关

- `ssh_permit_root_login`：是否允许 root 登录
- `ssh_password_authentication`：是否启用密码认证
- `ssh_cloud_init_override`：是否覆盖 cloud-init 的 SSH 配置

### DNS 相关

```
manage_dns_config: false
dns_servers:
  - 8.8.8.8
  - 114.114.114.114
fallback_dns: 1.1.1.1
```

- `manage_dns_config`：是否管理 `systemd-resolved`
- `dns_servers`：主 DNS
- `fallback_dns`：备用 DNS

### Docker mirror 相关

```
registry_mirrors:
  - "https://docker.1ms.run"
  - "https://docker.1panel.live"
  - "https://dockerproxy.net"
```

用于生成 `/etc/docker/daemon.json` 中的 `registry-mirrors`。

## 使用方式

### 1. 语法检查

```
ansible-playbook -i playbook/hosts playbook/site.yml --syntax-check
```

### 2. 执行全部 role

```
ansible-playbook -i playbook/hosts playbook/site.yml -b
```

### 3. 仅执行某台主机

```
ansible-playbook -i playbook/hosts playbook/site.yml -l host5 -b
```

### 4. 仅执行 Docker 安装 role

```
ansible-playbook -i playbook/hosts playbook/site.yml --tags docker_install -b
```

## 推荐测试顺序

建议按以下顺序测试：

1. Ubuntu 22.04
2. Ubuntu 24.04
3. 主流程先测试（SSH / DNS / Swap 默认关闭）
4. 再分别单独开启：
   - `common_manage_swap: true`
   - `manage_dns_config: true`
   - `common_manage_ssh: true`

## 6. 总结

- **SSH 指纹卡死**：首次连接新机器时会弹 `yes/no` 确认，影响自动化执行。  
  - 方案：控制机临时注入 `export ANSIBLE_HOST_KEY_CHECKING=False`，跳过 known_hosts 校验。生产环境建议使用受控的 known_hosts 管理方式，而不是长期关闭校验。

- **官方源连接重置 / 下载慢**：国内环境拉取 Docker 官方 GPG 和 Repo 可能出现连接重置、超时或速度极慢。  
  - 方案：统一切换为国内镜像站（如阿里云），并使用新的 `/etc/apt/keyrings` 路径存放 GPG 公钥。

- **Docker 权限拒绝**：普通用户执行 `docker ps` 报 `permission denied while trying to connect to the docker API`。  
  - 方案：使用 `user` 模块将用户加入 `docker` 组（`append: yes`），并在 Play 中使用 `meta: reset_connection` 刷新 Ansible 连接。  
  - 注意：交互式 SSH 会话通常仍需重新登录一次，组权限才会在当前 shell 中生效。

- **局部单步调试报错**：只想跑单个 Role 或局部任务时，容易因为 roles 语法不规范导致 YAML 报错。  
  - 方案：统一使用标准写法 `- role: role_name`，需要打标签时使用：
    ```yml
    - role: docker_install
      tags: only_install
    ```
    配合 `--tags` 精确执行。

- **Ubuntu 24.04 源替换不彻底**：仅修改 `/etc/apt/sources.list` 后，`apt update` 仍可能命中官方源。  
  - 原因：Ubuntu 24.04 默认可能使用 `/etc/apt/sources.list.d/ubuntu.sources`。  
  - 方案：开启 `apt_remove_extra_sources: true`，删除额外源文件后再执行 `apt update`。

- **幂等性不只是“能重复跑”**：有些 task 虽然功能正确，但会在每次执行时都显示 `changed`。  
  - 方案：避免无条件时间戳备份；像时区设置这类 task，先查询当前状态，再按需修改。最终将主流程收敛到二次执行 `changed=0`。

 ### QA

- **Q1: 怎么保证批量配免密的安全和剧本幂等性？**  
  - 答：绝对不要用 shell + echo 强行覆盖。坚持使用 `authorized_key` 模块。生产环境不建议 root 直连，统一使用普通账号（如 `vagrant` / `ubuntu`）配合 `become: yes` 提权。

- **Q2: 客户机房网络受限（被墙或纯离线），Ansible 怎么跑？**  
  - 答：公网受限时，将源地址抽成变量切到国内镜像或私有仓库；纯离线环境则提前打包 `.deb` 依赖，通过 `copy` 或本地仓库方式安装。

- **Q3: 几百个 Task 的剧本断在中间，怎么高效调试排障？**  
  - 答：不要全量无脑重跑。给 Role 和关键任务打好 Tags，用 `--tags`、`--skip-tags`、`--start-at-task` 精确定位问题，减少回归成本。

- **Q4: `apt_with_src` 什么时候需要开启？**  
  - 答：只有在需要源码包或构建依赖时才需要，例如执行 `apt source`、`apt build-dep`、源码调试、软件二次打包。普通软件安装和 Docker 安装不需要开启。

- **Q5: `apt_remove_extra_sources` 为什么在 Ubuntu 24.04 建议开启？**  
  - 答：因为 Ubuntu 24.04 默认可能启用 `/etc/apt/sources.list.d/ubuntu.sources`。如果只替换 `/etc/apt/sources.list`，APT 仍会混用系统默认官方源。开启后可以更彻底地完成源替换。

- **Q6: 为什么用户已经加入 docker 组了，还是提示 permission denied？**  
  - 答：这是 Linux 用户组变更后的常见现象。Ansible 侧的连接可以通过 `meta: reset_connection` 刷新，但交互式 SSH shell 通常仍需重新登录一次，新的组身份才会生效。