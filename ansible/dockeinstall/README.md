# Ansible

- 读取roles命令[[文件树]]
  - find playbook -type f -exec echo -e "n### File: {} ###" ; -exec cat {} ;

## 安装

- ```
  			  #安装pipx
  			  sudo apt update
  			  sudo apt install python3-pip -y
  			  python3 -m pip install --user pipx
  			  python3 -m pipx ensurepath # 把 ~/.local/bin 写进 PATH
  			  export PATH="$HOME/.local/bin:$PATH"
  			  pipx --version
  			  #安装完整的 Ansible 软件包
  			  #apt install python3.8-venv
  			  pipx install --include-deps ansible
  ```

## 配置密钥

- ```
  			  # 1. 静默安装 Ansible 及其依赖 (屏蔽 needrestart 弹窗)
  			  sudo NEEDRESTART_MODE=a apt update -y
  			  sudo NEEDRESTART_MODE=a apt install -y python3-pip pipx sshpass
  			  pipx ensurepath
  			  export PATH="$HOME/.local/bin:$PATH"
  			  pipx install --include-deps ansible
  			  
  			  # 2. 生成中控机专属密钥 (免回车)
  			  ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
  			  export ANSIBLE_HOST_KEY_CHECKING=False
  			  
  			  # 3. 模拟生产环境：把中控机公钥批量推给业务机 (假设 Vagrant 默认密码为 vagrant)
  			  # 生产中这一步通常由装机模板做，拿到裸机手动做的话用 sshpass 最快
  			  for ip in 192.168.0.221 192.168.0.222 192.168.0.223; do
  			    sshpass -p "vagrant" ssh-copy-id -o StrictHostKeyChecking=no vagrant@$ip
  			  done
  			  
  			  # 4. 创建极简且脱敏的 hosts 资产清单
  			  cat > hosts << 'EOF'
  			  [web]
  			  192.168.0.221
  			  192.168.0.222
  			  192.168.0.223
  			  
  			  [all:vars]
  			  ansible_user=vagrant
  			  EOF
  			  
  			  # 5. 终极验证
  			  ansible all -i hosts -m ping
  ```

# Linux初始化

## 一键替换apt源脚本

- ipv4映射地址优先级提高

  - `sed -i 's/^#precedence ::ffff:0:0\/96  100/precedence ::ffff:0:0\/96  100/' /etc/gai.conf`

- 替换更新源

  - ```
    			  #!/usr/bin/env bash
    			  #
    			  # switch-apt-source.sh
    			  # 用于在 Debian/Ubuntu 系统上一键切换 APT 源
    			  #
    			  # Usage:
    			  #   ./switch-apt-source.sh -s  [-r] [--dry-run] [--clean-cache] [--remove-extra-sources] [-y]
    			  #
    			  # 可选 source_name:
    			  #   huawei      -> 华为云镜像 (HTTP)
    			  #   aliyun      -> 阿里云镜像 (HTTP)
    			  #   tsinghua    -> 清华大学 TUNA 镜像 (HTTPS)
    			  #   ustc        -> 中国科学技术大学镜像 (HTTPS)
    			  #   official    -> 官方默认镜像 (HTTP)
    			  #   （默认使用 huawei）
    			  #
    			  # 主要优化：
    			  # 1. 智能处理 HTTPS 源依赖问题
    			  # 2. 减少不必要的包安装
    			  # 3. 提高切换速度
    			  # 4. 增强最小化系统兼容性
    			  
    			  set -euo pipefail
    			  
    			  #################################
    			  # 0. 常量与全局变量            #
    			  #################################
    			  SCRIPT_VERSION="1.0.7"
    			  BACKUP_DIR="/var/backups/apt-sources"
    			  SRC_FILE="/etc/apt/sources.list"
    			  SRC_DIR="/etc/apt/sources.list.d"
    			  TIMESTAMP=$(date +%Y%m%d-%H%M%S)
    			  
    			  # 颜色定义
    			  RED='\033[1;31m'
    			  GREEN='\033[1;32m'
    			  BLUE='\033[1;34m'
    			  YELLOW='\033[1;33m'
    			  NC='\033[0m'  # No Color
    			  
    			  #################################
    			  # 1. 镜像源配置                #
    			  #################################
    			  declare -A MIRROR_CONFIG=(
    			      # 名称  Debian基础URL                           Debian安全URL                        Ubuntu基础URL                           Ubuntu安全URL
    			      [huawei]="http://repo.huaweicloud.com/debian         http://repo.huaweicloud.com/debian-security    http://repo.huaweicloud.com/ubuntu         http://repo.huaweicloud.com/ubuntu"
    			      [aliyun]="http://mirrors.aliyun.com/debian           http://mirrors.aliyun.com/debian-security      http://mirrors.aliyun.com/ubuntu           http://mirrors.aliyun.com/ubuntu"
    			      [tsinghua]="https://mirrors.tuna.tsinghua.edu.cn/debian     https://mirrors.tuna.tsinghua.edu.cn/debian-security    https://mirrors.tuna.tsinghua.edu.cn/ubuntu     https://mirrors.tuna.tsinghua.edu.cn/ubuntu"
    			      [ustc]="https://mirrors.ustc.edu.cn/debian           https://mirrors.ustc.edu.cn/debian-security    https://mirrors.ustc.edu.cn/ubuntu         https://mirrors.ustc.edu.cn/ubuntu"
    			      [official]="http://deb.debian.org/debian             http://security.debian.org/debian-security      http://archive.ubuntu.com/ubuntu           http://security.ubuntu.com/ubuntu"
    			  )
    			  
    			  # HTTP 回退源（当 HTTPS 失败时使用）
    			  declare -A HTTP_FALLBACKS=(
    			      [tsinghua]="http://mirrors.tuna.tsinghua.edu.cn"
    			      [ustc]="http://mirrors.ustc.edu.cn"
    			  )
    			  
    			  # 定义 HTTP 源列表
    			  HTTP_SOURCES=("huawei" "aliyun" "official")
    			  
    			  #################################
    			  # 2. 函数：打印使用说明         #
    			  #################################
    			  print_usage() {
    			      echo -e "${BLUE}脚本版本: switch-apt-source.sh v${SCRIPT_VERSION}${NC}"
    			      echo -e "${BLUE}可用镜像源：${NC}"
    			      for source in "${!MIRROR_CONFIG[@]}"; do
    			          if [[ " ${HTTP_SOURCES[*]} " =~ " ${source} " ]]; then
    			              echo "  - $source (HTTP)"
    			          else
    			              echo "  - $source (HTTPS)"
    			          fi
    			      done
    			      cat << EOF
    			  
    			  Usage: $0 [ -s  ] [ -r ] [ --dry-run ] [ --clean-cache ] [ --remove-extra-sources ] [-y]
    			  
    			    -s|--source           指定要切换的镜像源（默认为 huawei）
    			    -r|--with-src         额外写入 deb-src 源行（源码包）
    			    --dry-run             仅打印最终会写入的内容，不实际覆盖源文件
    			    --clean-cache         切换完源后自动执行 apt-get update && apt-get autoclean
    			    --remove-extra-sources 移除 /etc/apt/sources.list.d/ 目录下的所有额外源
    			    -y|--assume-yes       自动回答所有提示为 yes（适用于自动化场景）
    			  
    			  示例：
    			    $0 -s aliyun
    			    $0 --source tsinghua --with-src -y
    			    $0 -s ustc --dry-run
    			    $0 -s official --clean-cache --remove-extra-sources -y
    			  EOF
    			      exit 0
    			  }
    			  
    			  #################################
    			  # 3. 函数：列出可选镜像源       #
    			  #################################
    			  list_sources() {
    			      echo -e "${BLUE}支持的镜像源列表：${NC}"
    			      for source in "${!MIRROR_CONFIG[@]}"; do
    			          if [[ " ${HTTP_SOURCES[*]} " =~ " ${source} " ]]; then
    			              echo "  - $source (HTTP)"
    			          else
    			              echo "  - $source (HTTPS)"
    			          fi
    			      done
    			      exit 0
    			  }
    			  
    			  #################################
    			  # 4. 函数：检查 HTTPS 依赖     #
    			  #################################
    			  check_https_dependencies() {
    			      # 仅对 HTTPS 源检查依赖
    			      if [[ " ${HTTP_SOURCES[*]} " =~ " ${SOURCE_NAME} " ]]; then
    			          return 0  # HTTP 源不需要额外依赖
    			      fi
    			      
    			      # 检查是否已安装 ca-certificates
    			      if ! dpkg -s ca-certificates &>/dev/null; then
    			          echo -e "${RED}错误：系统缺少 ca-certificates 包，无法访问 HTTPS 源${NC}"
    			          echo -e "${YELLOW}请先安装 ca-certificates：apt-get install -y ca-certificates${NC}"
    			          
    			          # 检查是否有 HTTP 回退源
    			          if [[ -n "${HTTP_FALLBACKS[$SOURCE_NAME]:-}" ]]; then
    			              if [[ "$ASSUME_YES" == true ]]; then
    			                  echo -e "${YELLOW}自动回退到 HTTP 源...${NC}"
    			                  BASE_URL="http://${BASE_URL#*://}"
    			                  SECURITY_URL="http://${SECURITY_URL#*://}"
    			                  return 0
    			              else
    			                  echo -e "${YELLOW}是否回退到 HTTP 源？(y/N)${NC}"
    			                  read -rp " " yn
    			                  if [[ "$yn" =~ ^[Yy]$ ]]; then
    			                      echo -e "${YELLOW}正在回退到 HTTP 源...${NC}"
    			                      BASE_URL="http://${BASE_URL#*://}"
    			                      SECURITY_URL="http://${SECURITY_URL#*://}"
    			                      return 0
    			                  fi
    			              fi
    			          fi
    			          
    			          exit 1
    			      fi
    			      
    			      return 0
    			  }
    			  #################################
    			  # 5. 参数解析                  #
    			  #################################
    			  SOURCE_NAME="huawei"   # 默认华为源
    			  ADD_DEBSRC=false       # 是否写入 deb-src
    			  DRY_RUN=false          # 是否仅打印，不实际覆盖
    			  CLEAN_CACHE=false      # 是否切换完后自动清理缓存
    			  REMOVE_EXTRA_SOURCES=false # 是否移除额外源
    			  ASSUME_YES=false       # 是否自动回答 yes
    			  
    			  while [[ $# -gt 0 ]]; do
    			      case "$1" in
    			          -s|--source)
    			              if [[ -n "${2-}" && ! "$2" =~ ^- ]]; then
    			                  SOURCE_NAME="$2"
    			                  shift
    			              else
    			                  echo -e "${RED}错误：-s/--source 选项缺少参数${NC}"
    			                  exit 1
    			              fi
    			              ;;
    			          --source=*)
    			              SOURCE_NAME="${1#*=}"
    			              ;;
    			          -r|--with-src)
    			              ADD_DEBSRC=true
    			              ;;
    			          --dry-run)
    			              DRY_RUN=true
    			              ;;
    			          --clean-cache)
    			              CLEAN_CACHE=true
    			              ;;
    			          --remove-extra-sources)
    			              REMOVE_EXTRA_SOURCES=true
    			              ;;
    			          -y|--assume-yes)
    			              ASSUME_YES=true
    			              ;;
    			          -l|--list)
    			              list_sources
    			              ;;
    			          -h|--help)
    			              print_usage
    			              ;;
    			          *)
    			              echo -e "${RED}错误：未知选项 '$1'${NC}"
    			              print_usage
    			              exit 1
    			              ;;
    			      esac
    			      shift
    			  done
    			  
    			  #################################
    			  # 6. 验证镜像源名称是否有效     #
    			  #################################
    			  if [[ -z "${MIRROR_CONFIG[$SOURCE_NAME]-}" ]]; then
    			      echo -e "${RED}错误：不支持的镜像源 '${SOURCE_NAME}'${NC}"
    			      echo -e "${BLUE}可用源：${NC}"
    			      for source in "${!MIRROR_CONFIG[@]}"; do
    			          echo "  - $source"
    			      done
    			      exit 1
    			  fi
    			  
    			  #################################
    			  # 7. 检查是否以 root 执行       #
    			  #################################
    			  if [[ $EUID -ne 0 ]]; then
    			     echo -e "${RED}错误：请使用 root 权限运行此脚本${NC}"
    			     exit 1
    			  fi
    			  
    			  #################################
    			  # 8. 获取操作系统信息          #
    			  #################################
    			  if [[ ! -f /etc/os-release ]]; then
    			      echo -e "${RED}错误：无法识别操作系统，请确保是 Debian/Ubuntu 系统。${NC}"
    			      exit 1
    			  fi
    			  
    			  # 加载系统信息
    			  . /etc/os-release
    			  DIST_ID=$(echo "${ID:-unknown}" | tr '[:upper:]' '[:lower:]')
    			  DIST_CODENAME=$(echo "${VERSION_CODENAME:-}" | tr '[:upper:]' '[:lower:]')
    			  
    			  if [[ -z "$DIST_CODENAME" ]]; then
    			      # 尝试使用 lsb_release 获取版本代号
    			      if command -v lsb_release &>/dev/null; then
    			          DIST_CODENAME=$(lsb_release -cs)
    			      else
    			          echo -e "${YELLOW}警告：无法获取系统版本代号，使用默认值 bookworm${NC}"
    			          DIST_CODENAME="bookworm"
    			      fi
    			  fi
    			  
    			  # 获取 Debian 主版本号
    			  DEB_MAJOR_VERSION=""
    			  if [[ "$DIST_ID" == "debian" ]]; then
    			      DEB_MAJOR_VERSION=$(echo "${VERSION_ID:-}" | cut -d. -f1)
    			  fi
    			  
    			  # 系统架构
    			  DIST_ARCH=$(dpkg --print-architecture)
    			  
    			  #################################
    			  # 9. 解析镜像配置              #
    			  #################################
    			  read -r -a URLS <<< "${MIRROR_CONFIG[$SOURCE_NAME]}"
    			  if [[ ${#URLS[@]} -lt 4 ]]; then
    			      echo -e "${RED}错误：镜像源配置不完整，请检查脚本配置${NC}"
    			      exit 1
    			  fi
    			  
    			  # 根据系统类型选择 URL
    			  if [[ "$DIST_ID" == "debian" ]]; then
    			      BASE_URL="${URLS[0]}"
    			      SECURITY_URL="${URLS[1]}"
    			  else
    			      BASE_URL="${URLS[2]}"
    			      SECURITY_URL="${URLS[3]}"
    			  fi
    			  
    			  # 检查 HTTPS 依赖
    			  HAS_CURL=false
    			  if check_https_dependencies; then
    			      HAS_CURL=true
    			  fi
    			  
    			  #################################
    			  # 10. 设置组件和安全源路径     #
    			  #################################
    			  if [[ "$DIST_ID" == "debian" ]]; then
    			      COMPONENTS="main contrib non-free"
    			      # Debian 12+ 默认再加 non-free-firmware
    			      if [[ -n "$DEB_MAJOR_VERSION" && "$DEB_MAJOR_VERSION" -ge 12 ]]; then
    			          COMPONENTS="${COMPONENTS} non-free-firmware"
    			      fi
    			      # 安全源路径：Debian 9 及更早 "/updates"，10+ "-security"
    			      if [[ -n "$DEB_MAJOR_VERSION" && "$DEB_MAJOR_VERSION" -lt 10 ]]; then
    			          SECURITY_SUITE="${DIST_CODENAME}/updates"
    			      else
    			          SECURITY_SUITE="${DIST_CODENAME}-security"
    			      fi
    			  else
    			      # Ubuntu 系列
    			      COMPONENTS="main restricted universe multiverse"
    			      SECURITY_SUITE="${DIST_CODENAME}-security"
    			  fi
    			  
    			  #################################
    			  # 11. 备份与旧备份自动清理      #
    			  #################################
    			  mkdir -p "$BACKUP_DIR"
    			  
    			  # 自动清理旧备份
    			  if [[ $(find "$BACKUP_DIR" -type f -name 'sources.list.bak.*' 2>/dev/null | wc -l) -gt 0 ]]; then
    			      find "$BACKUP_DIR" -type f -name 'sources.list.bak.*' -mtime +30 -delete 2>/dev/null || true
    			  fi
    			  
    			  BACKUP_FILE="${BACKUP_DIR}/sources.list.bak.${TIMESTAMP}"
    			  
    			  # 检查源文件状态
    			  if [[ ! -e "$SRC_FILE" ]]; then
    			      [[ "$ASSUME_YES" == false ]] && echo -e "${YELLOW}源文件不存在，自动创建新文件${NC}"
    			      touch "$SRC_FILE"
    			      chmod 644 "$SRC_FILE"
    			  elif [[ ! -s "$SRC_FILE" ]]; then
    			      [[ "$ASSUME_YES" == false ]] && echo -e "${YELLOW}源文件为空，继续写入新配置${NC}"
    			  fi
    			  
    			  # 文件权限检查
    			  if [[ -e "$SRC_FILE" && ! -w "$SRC_FILE" ]]; then
    			      chmod u+w "$SRC_FILE"
    			  fi
    			  
    			  echo "正在备份原文件: ${SRC_FILE} -> ${BACKUP_FILE}"
    			  cp -f "$SRC_FILE" "$BACKUP_FILE"
    			  
    			  #################################
    			  # 12. 处理额外源 (可选)        #
    			  #################################
    			  if [[ "$REMOVE_EXTRA_SOURCES" == true ]]; then
    			      [[ "$ASSUME_YES" == false ]] && echo -e "${YELLOW}正在移除额外源...${NC}"
    			      
    			      if [[ -d "$SRC_DIR" ]]; then
    			          BACKUP_EXTRA_DIR="${BACKUP_DIR}/sources.list.d.${TIMESTAMP}"
    			          mkdir -p "$BACKUP_EXTRA_DIR"
    			          
    			          if find "$SRC_DIR" -maxdepth 1 -type f | grep -q .; then
    			              mv -f "${SRC_DIR}"/* "$BACKUP_EXTRA_DIR/" 2>/dev/null || true
    			              [[ "$ASSUME_YES" == false ]] && echo -e "${GREEN}已备份并移除额外源到: ${BACKUP_EXTRA_DIR}${NC}"
    			          fi
    			      fi
    			  fi
    			  
    			  #################################
    			  # 13. 生成临时文件并写入新源   #
    			  #################################
    			  TMPFILE=$(mktemp "/tmp/sources.list.XXXXXX")
    			  chmod 644 "$TMPFILE"
    			  
    			  {
    			      echo "# ${SOURCE_NAME} 镜像源 (自动生成) - ${PRETTY_NAME} [${DIST_ARCH}]"
    			      echo "# 生成时间: $(date '+%Y-%m-%d %H:%M:%S')"
    			      echo "# 脚本版本: switch-apt-source.sh v${SCRIPT_VERSION}"
    			      echo "# 协议: $(echo "$BASE_URL" | cut -d: -f1)"
    			      echo
    			      echo "deb [arch=${DIST_ARCH}] ${BASE_URL} ${DIST_CODENAME} ${COMPONENTS}"
    			      $ADD_DEBSRC && echo "deb-src [arch=${DIST_ARCH}] ${BASE_URL} ${DIST_CODENAME} ${COMPONENTS}"
    			      echo
    			      echo "deb [arch=${DIST_ARCH}] ${SECURITY_URL} ${SECURITY_SUITE} ${COMPONENTS}"
    			      $ADD_DEBSRC && echo "deb-src [arch=${DIST_ARCH}] ${SECURITY_URL} ${SECURITY_SUITE} ${COMPONENTS}"
    			      echo
    			      echo "deb [arch=${DIST_ARCH}] ${BASE_URL} ${DIST_CODENAME}-updates ${COMPONENTS}"
    			      $ADD_DEBSRC && echo "deb-src [arch=${DIST_ARCH}] ${BASE_URL} ${DIST_CODENAME}-updates ${COMPONENTS}"
    			      echo
    			      echo "deb [arch=${DIST_ARCH}] ${BASE_URL} ${DIST_CODENAME}-backports ${COMPONENTS}"
    			      $ADD_DEBSRC && echo "deb-src [arch=${DIST_ARCH}] ${BASE_URL} ${DIST_CODENAME}-backports ${COMPONENTS}"
    			  } > "$TMPFILE"
    			  
    			  #################################
    			  # 14. Dry-run 模式处理         #
    			  #################################
    			  if [[ "$DRY_RUN" == true ]]; then
    			      echo -e "\n${BLUE}---- Dry-run 模式：以下内容将写入 ${SRC_FILE} ----${NC}"
    			      cat "$TMPFILE"
    			      rm -f "$TMPFILE"
    			      echo -e "\n${GREEN}✓ Dry-run 完成，未实际覆盖任何文件。${NC}"
    			      exit 0
    			  fi
    			  
    			  #################################
    			  # 15. 原子替换 sources.list    #
    			  #################################
    			  mv -f "$TMPFILE" "$SRC_FILE"
    			  
    			  echo -e "\n${GREEN}✓ 已成功将 APT 源切换为: ${SOURCE_NAME}${NC}"
    			  echo "主仓库: ${BASE_URL}"
    			  echo "安全更新: ${SECURITY_URL}"
    			  echo "组件: ${COMPONENTS}"
    			  $ADD_DEBSRC && echo "已额外写入 deb-src 行。"
    			  
    			  #################################
    			  # 16. 后续操作提示             #
    			  #################################
    			  echo -e "\n${BLUE}▷ 下一步操作建议:${NC}"
    			  echo "  1. 更新软件包列表: apt update"
    			  echo "  2. 升级所有软件包: apt upgrade"
    			  
    			  echo -e "\n${BLUE}▷ 回滚方法:${NC}"
    			  echo "  cp -v ${BACKUP_FILE} ${SRC_FILE}"
    			  [[ "$REMOVE_EXTRA_SOURCES" == true ]] && echo "  mv ${BACKUP_EXTRA_DIR}/* ${SRC_DIR}/"
    			  
    			  #################################
    			  # 17. 清理缓存（可选）         #
    			  #################################
    			  if [[ "$CLEAN_CACHE" == true ]]; then
    			      echo -e "\n${BLUE}正在执行 apt-get update && apt-get autoclean ...${NC}"
    			      if apt-get update -y; then
    			          apt-get autoclean -y
    			          echo -e "${GREEN}✓ 缓存清理完成。${NC}"
    			      else
    			          echo -e "${YELLOW}警告：apt-get update 失败，跳过 autoclean${NC}"
    			      fi
    			  fi
    			  
    			  exit 0
    			  
    			  
    ```

## 初始化(包括设置root登录、安装常用工具、时区、虚拟内存

- ```
  			  #!/bin/bash
  			  
  			  # 确保脚本以 root 权限运行
  			  if [ "$EUID" -ne 0 ]; then
  			    echo "请使用 root 权限运行此脚本 (例如: sudo bash $0)"
  			    exit 1
  			  fi
  			  
  			  echo "========== 开始执行基础配置 (通用版) =========="
  			  
  			  # 1. 设置 root 密码
  			  #echo -e "\n---> 1. 正在设置 root 密码，请根据提示输入："
  			  #passwd root
  			  
  			  # 2. 修改 SSH 配置文件 (允许 root 登录 & 启用密码认证)
  			  echo -e "\n---> 2. 正在配置 SSH..."
  			  SSHD_CONFIG="/etc/ssh/sshd_config"
  			  cp $SSHD_CONFIG ${SSHD_CONFIG}.bak
  			  
  			  # 处理 PermitRootLogin 和 PasswordAuthentication
  			  sed -i 's/^#*PermitRootLogin.*/PermitRootLogin yes/' $SSHD_CONFIG
  			  if ! grep -q "^PermitRootLogin yes" $SSHD_CONFIG; then echo "PermitRootLogin yes" >> $SSHD_CONFIG; fi
  			  
  			  sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' $SSHD_CONFIG
  			  if ! grep -q "^PasswordAuthentication yes" $SSHD_CONFIG; then echo "PasswordAuthentication yes" >> $SSHD_CONFIG; fi
  			  
  			  # 覆盖 Ubuntu 22.04+ 默认的包含文件配置
  			  if [ -d "/etc/ssh/sshd_config.d" ]; then
  			      echo "PasswordAuthentication yes" > /etc/ssh/sshd_config.d/50-cloud-init.conf 2>/dev/null
  			  fi
  			  
  			  # 重启 SSH 服务
  			  if systemctl restart ssh 2>/dev/null || systemctl restart sshd 2>/dev/null; then
  			      echo "SSH 服务已重启。"
  			  fi
  			  
  			  # 3 & 4. 自动检测包管理器进行系统更新与基础工具安装
  			  echo -e "\n---> 3 & 4. 正在检测系统环境并更新软件包..."
  			  
  			  if command -v apt >/dev/null 2>&1; then
  			      echo "检测到 Debian/Ubuntu (apt) 系统"
  			      export DEBIAN_FRONTEND=noninteractive
  			      apt-get update -y
  			      apt-get upgrade -y -o Dpkg::Options::="--force-confdef" -o Dpkg::Options::="--force-confold"
  			      echo "正在安装常用工具..."
  			      apt-get install -y curl wget vim git htop net-tools dnsutils tree unzip tar jq
  			  
  			  elif command -v dnf >/dev/null 2>&1 || command -v yum >/dev/null 2>&1; then
  			      echo "检测到 RHEL/CentOS/Rocky/Alma (dnf/yum) 系统"
  			      # 优先使用 dnf (新版系统)，如果没有则退化使用 yum
  			      PKG_MANAGER=$(command -v dnf || command -v yum)
  			      
  			      $PKG_MANAGER update -y
  			      # RHEL 体系下，安装 htop 和 jq 通常需要先安装 epel-release 扩展源
  			      $PKG_MANAGER install -y epel-release
  			      # 注意：Debian下的 dnsutils 在 RHEL 下叫 bind-utils
  			      echo "正在安装常用工具..."
  			      $PKG_MANAGER install -y curl wget vim git htop net-tools bind-utils tree unzip tar jq sshpass
  			  else
  			      echo "警告: 未知的包管理器，跳过系统更新和工具安装。"
  			  fi
  			  
  			  # 5. 设置时区为亚洲/上海
  			  echo -e "\n---> 5. 正在配置时区为 Asia/Shanghai..."
  			  timedatectl set-timezone Asia/Shanghai
  			  echo "当前系统时间：$(date)"
  			  
  			  # 6. 配置虚拟内存 (Swap) - 分配 1GB
  			  echo -e "\n---> 6. 正在检查并配置 1GB 虚拟内存(Swap)..."
  			  if swapon --show | grep -q "/swapfile"; then
  			      echo "Swap 已经存在，跳过创建。"
  			  else
  			      # 优先尝试 fallocate，如果不支持则退化使用 dd
  			      if fallocate -l 1G /swapfile 2>/dev/null; then
  			          echo "使用 fallocate 创建了 Swap 文件。"
  			      else
  			          echo "fallocate 失败，使用 dd 创建 Swap 文件 (这可能需要几秒钟)..."
  			          dd if=/dev/zero of=/swapfile bs=1M count=1024 status=none
  			      fi
  			      chmod 600 /swapfile
  			      mkswap /swapfile
  			      swapon /swapfile
  			      if ! grep -q "/swapfile" /etc/fstab; then
  			          echo '/swapfile none swap sw 0 0' >> /etc/fstab
  			      fi
  			      echo "1GB Swap 创建并挂载成功。"
  			  fi
  			  
  			  echo -e "\n========== 所有基础配置执行完毕！ =========="
  ```