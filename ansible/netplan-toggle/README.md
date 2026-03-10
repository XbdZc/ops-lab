# netplan-toggle：在 direct / clash 两种网络模式之间切换（IPv4）

这个 playbook/role 用来在「双网卡」主机上切换 IPv4 默认路由（default route）的出口：
- **clash 模式**：默认路由走 `eth0`（通常是 Vagrant NAT / 代理网关那条链路），常用于“走 Clash 出口（US）”
- **direct 模式**：默认路由走 `eth1`（通常是内网/物理网卡直连），常用于“直连（CN）”

并且新增了“**重启后也保持模式**”的机制：开机后自动把默认路由纠偏到你选择的模式。

> 注意：本方案主要管理 **IPv4 路由**；DNS 与 fake-ip 相关问题属于“DNS 流量是否被 Clash 接管/劫持”的范畴，见下方排错。

---

## 适用环境/前提

- Ubuntu/Debian 系（使用 **netplan + systemd-networkd + systemd-resolved**）
- 目标机器不是 NetworkManager 管理（否则 `networkctl status` 可能不准确）
- 两张网卡（示例）：
  - `eth0`：DHCP（例如网关 `10.0.2.2`）
  - `eth1`：直连网关（例如 `192.168.0.1`）

---

## 两种模式做了什么

### direct 模式（直连）
目标：让 **eth1 成为默认出口**，同时阻止 eth0 的 DHCP 抢网关/路由/DNS。

做法：
1. 写入自定义 netplan 文件（默认：`/etc/netplan/99-zz-custom.yaml`）：
   - 给 `eth1` 增加 **默认路由**（metric 通常较大，如 50）
   - 给 `eth1` 配置 DNS（如 `192.168.0.1`、`114.114.114.114`）
2. 自动识别 `eth0` 当前由哪个 `.network` 文件管理，然后写入 drop-in：
   - `/etc/systemd/network/<eth0_network_file>.d/override.conf`
   - 内容为禁用 DHCPv4 注入网关/路由/DNS：
     ```ini
     [DHCPv4]
     UseDNS=false
     UseRoutes=false
     UseGateway=false

1. `netplan apply` + 重启 `systemd-networkd`/`systemd-resolved`
2. 运行时清理：删除可能残留在 `eth0` 上的 default route（IPv4）

### clash 模式

目标：让 **eth0(DHCP) 成为默认出口**，并撤销 direct 模式对 eth0 的限制。

做法：

1. 删除 `/etc/netplan/99-zz-custom.yaml`（撤销对 eth1 的额外默认路由/DNS）
2. 删除 eth0 的 drop-in override（恢复 DHCP 默认行为）
3. `netplan apply` + 重启 networkd/resolved
4. 运行时清理：删除可能残留在 `eth1` 上的 default route（IPv4）

------

## 关键增强：重启后保持模式（本次会话新增）

之前的版本只用 `ip route del` 做“运行时删除”，重启后会被网络配置重新生成，导致模式回退。

现在新增三件东西来保证“重启后也纠偏到目标模式”：

1. 模式状态文件：
   - `/etc/netplan_toggle/mode`
   - 内容为 `direct` 或 `clash`
2. 纠偏脚本：
   - `/usr/local/sbin/netplan-toggle-enforce.sh`
   - 开机或手动执行时读取 mode，并做：
     - clash：确保默认路由在 `eth0`（从 DHCP lease 中取网关），并删除 `eth1` 默认路由
     - direct：删除 `eth0` 默认路由（默认路由交给 eth1 的配置）
3. systemd oneshot 服务：
   - `/etc/systemd/system/netplan-toggle-enforce.service`
   - `After=network-online.target`，保证 DHCP lease/网络准备好后再纠偏
   - 已 `enable`，所以每次开机会自动执行一次纠偏脚本

------

## 主要可配置变量（示例）

- `netplan_mode`: `"direct"` 或 `"clash"`
- `eth0_if`: `"eth0"`
- `eth1_if`: `"eth1"`
- `eth1_gw4`: `"192.168.0.1"`
- `eth1_dns4`: `["192.168.0.1", "114.114.114.114"]`
- `custom_netplan_file`: `"/etc/netplan/99-zz-custom.yaml"`

------

## 使用方法

切到 clash：

```
ansible-playbook -i inventory toggle.yml -b -e netplan_mode=clash
```

切到 direct：

```
ansible-playbook -i inventory toggle.yml -b -e netplan_mode=direct
```

> 建议在有控制台的情况下操作（或确保不会断 SSH）。因为会 `netplan apply` + 重启网络服务。

------

## 验证方式

1. 只应存在 **1 条** IPv4 默认路由：

```
ip -4 route show default

ip -4 route show default | wc -l
```

1. 查看当前模式状态：

```
cat /etc/netplan_toggle/mode
```

1. 查看开机纠偏服务：

```
systemctl status netplan-toggle-enforce.service --no-pager

journalctl -u netplan-toggle-enforce.service -b --no-pager
```

1. 外网出口验证（示例）：

```
curl -4 -s https://ipinfo.io/country; echo
```

------

## DNS / fake-ip 说明（常见疑问）

- `ping baidu.com` 得到 `198.18.x.x`，基本可以判定：**DNS 回答来自 Clash 的 fake-ip 引擎（或透明 DNS 劫持）**
- 这件事不一定会体现在 `/etc/resolv.conf` 上，因为有些实现是“透明劫持 53 端口”，你看起来 nameserver 没变，但流量被重定向了。
- direct/clash 模式本方案主要切的是“默认路由出口”，所以会出现：
  - 走 eth0 的时候更容易触发 fake-ip（如果 eth0 路径上有 DNS 劫持/Clash 网关）
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