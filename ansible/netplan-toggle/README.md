# netplan-toggle：在 direct / proxy 两种网络模式之间切换（IPv4）

本项目使用 Ansible role 在“双网卡”主机上切换 IPv4 默认路由（default route）的出口，并提供“重启后保持模式”的纠偏机制。

- proxy 模式：默认路由走 eth0（通常是 NAT/代理/proxy 网关那条链路）
- direct 模式：默认路由走 eth1（通常是内网/物理网卡直连）

> 说明：本方案主要管理“IPv4 默认路由（以及 direct 下的 DNS）”。DNS fake-ip/透明劫持这类现象，更多取决于你链路上的 proxy/网关行为。

---

## 适用环境/前提

- Ubuntu/Debian 系（netplan）
- 使用 systemd-networkd 管理网卡（role 依赖 networkctl / drop-in）
  - `systemctl is-active systemd-networkd` 需为 `active`
- 双网卡（示例）：
  - eth0：DHCP（NAT/代理链路）
  - eth1：直连内网（例如网关 192.168.0.1）

---

## 仓库结构

- hosts：Ansible inventory（示例组为 `[vms]`）
- toggle-netplan.yml：入口 playbook（调用 role）
- roles/netplan_toggle：核心逻辑
- group_vars/vms.yml：统一配置（对 `[vms]` 组内所有主机生效）
- host_vars/<hostname>.yml：单机覆盖（只有少数机器不同才需要）

如果仓库里存在 `netplan-toggle.yml.legacy`，那是早期“单文件 playbook”，变量名体系不同，不再推荐使用。

---

## 统一配置

统一变量放在：`group_vars/vms.yml`
新增主机时，只要把主机加入 inventory 的 `[vms]` 组，一般就不需要再写 `host_vars`。

常用可配置项（均可在 group_vars/host_vars 或运行时 `-e` 覆盖）：

- 模式：
  - `netplan_toggle_mode`: `direct` / `proxy`（建议运行时 `-e` 指定）
- 网卡名：
  - `netplan_toggle_eth0_if`
  - `netplan_toggle_eth1_if`
- direct 模式下 eth1 默认出口配置：
  - `netplan_toggle_eth1_gw4`
  - `netplan_toggle_eth1_dns4`
  - `netplan_toggle_eth1_metric`
- 生成/管理的文件名：
  - `netplan_toggle_custom_netplan_file`（默认 `/etc/netplan/99-netplan-toggle.yaml`）
  - `netplan_toggle_dropin_name`（默认 `99-netplan-toggle.conf`）
- 切换后校验：
  - `netplan_toggle_verify` / `netplan_toggle_assert_single_default`
  - `netplan_toggle_verify_probe_ip` / `retries` / `delay`

---

## 使用方法

切到 proxy：

```bash
ansible-playbook -i hosts toggle-netplan.yml -b -e netplan_toggle_mode=proxy
```

切到 direct：

```bash
ansible-playbook -i hosts toggle-netplan.yml -b -e netplan_toggle_mode=direct
```

只操作一台（例：host1）：

```bash
ansible-playbook -i hosts toggle-netplan.yml -b -l host1 -e netplan_toggle_mode=direct
```

> playbook 默认 `serial: 1`（逐台切换），降低集体断网风险。建议在有控制台的情况下操作。

---

## 两种模式做了什么

### direct 模式（eth1 做默认出口）

目标：

- 让 eth1 成为 IPv4 默认路由出口
- 阻止 eth0 的 DHCP 抢默认网关/路由/DNS

主要动作：

1. 写入自定义 netplan 文件（默认：`/etc/netplan/99-netplan-toggle.yaml`）
   - 给 eth1 增加默认路由：`to: default via: <netplan_toggle_eth1_gw4> metric: <netplan_toggle_eth1_metric>`
   - 配置 eth1 DNS：`netplan_toggle_eth1_dns4`
2. 自动识别 eth0 对应的 `.network` 文件，并写入 drop-in：
   - `/etc/systemd/network/<eth0_network_file>.d/<netplan_toggle_dropin_name>`
   - 禁止 DHCPv4 注入 gateway/routes/dns（UseGateway/UseRoutes/UseDNS = false）
3. `netplan apply`
4. 运行时清理：删除 eth0 上可能残留的 IPv4 default route
5. 清理由 proxy 模式写入的 eth1 “清空 Gateway” drop-in（如果存在）

### proxy 模式（eth0 做默认出口）

目标：

- 恢复 eth0 的 DHCP 默认行为，让 eth0 成为 IPv4 默认出口
- 确保 eth1 不成为默认路由

主要动作：

1. 删除自定义 netplan 文件（撤销 direct 模式对 eth1 的默认路由/DNS）
2. 删除 eth0 的 “禁 DHCP 注入” drop-in
3. `netplan apply`
4. 运行时清理：删除 eth1 上可能残留的 IPv4 default route
5. 强化 eth0 DHCP 生效（写入 eth0 drop-in 强制 UseRoutes/UseGateway/UseDNS = yes，并重启 networkd、renew）
6. 若 eth0 仍未生成默认路由：从 systemd lease 里读 ROUTER，fallback 添加默认路由
7. 为 eth1 写入 drop-in 清空 Gateway（防止 eth1 变 default）

---

## 重启后保持模式（纠偏机制）

role 会写入并启用：

- 模式状态文件：`/etc/netplan_toggle/mode`（内容为 `direct` 或 `proxy`）
- 纠偏脚本：`/usr/local/sbin/netplan-toggle-enforce.sh`
- systemd oneshot 服务：`/etc/systemd/system/netplan-toggle-enforce.service`

开机后会读取 mode 并自动纠偏，确保默认路由符合目标模式。

---

## 验证/排查

查看默认路由（建议只有 1 条）：

```bash
ip -4 route show default
ip -4 route show default | wc -l
```

查看当前保存的模式：

```bash
cat /etc/netplan_toggle/mode
```

查看纠偏服务状态与日志：

```bash
systemctl status netplan-toggle-enforce.service --no-pager
journalctl -u netplan-toggle-enforce.service -b --no-pager
```

验证某个目的地址实际走哪张网卡：

```bash
ip -4 route get 1.1.1.1
```

验证ip

``````
curl -4 -s https://ipinfo.io/country; echo
``````

---

## 新增主机怎么做？

1. 把主机加到 `hosts` 的 `[vms]` 组
2. 直接运行 playbook（会自动继承 `group_vars/vms.yml`）

只有当某台主机参数不同（接口名/网关/DNS/metric 等）时，才需要单独写：

`host_vars/<hostname>.yml`（覆盖差异项即可），例如：

```yaml
netplan_toggle_eth0_if: "ens160"
netplan_toggle_eth1_if: "ens192"
```

## DNS / fake-ip 说明（常见疑问）

- `ping baidu.com` 得到 `198.18.x.x`，基本可以判定：**DNS 回答来自 proxy 的 fake-ip 引擎（或透明 DNS 劫持）**
- 这件事不一定会体现在 `/etc/resolv.conf` 上，因为有些实现是“透明劫持 53 端口”，你看起来 nameserver 没变，但流量被重定向了。
- direct/proxy 模式本方案主要切的是“默认路由出口”，所以会出现：
  - 走 eth0 的时候更容易触发 fake-ip（如果 eth0 路径上有 DNS 劫持/proxy 网关）
  - 走 eth1 时解析回真实 IP

快速定位（可选）：

```
ip -4 route get 198.18.0.34

dig @8.8.8.8 baidu.com A +short
```

------

## 限制

- 仅处理 IPv4 default route（IPv6 未纳入切换）
- 默认假设 systemd-networkd 管理网卡；如果是 NetworkManager，需要改检测方式