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

## Ansible 批量纳管与 Docker 标准化部署复盘

- 🛠️ 核心踩坑与解决方案 (实战总结)
  - **SSH 指纹卡死**：首次连新机器弹 `yes/no`。
    - 方案：控制机注入 `export ANSIBLE_HOST_KEY_CHECKING=False`，跳过 known_hosts 校验。
  - **官方源连接重置**：国内拉取 Docker 官方 GPG 和 Repo 报 Errno 104。
    - 方案：全面替换为国内镜像站（如阿里云），并使用最新的 `/etc/apt/keyrings` 路径存放公钥。
  - **Docker 权限拒绝**：普通用户执行 `docker ps` 报 permission denied。
    - 方案：用 `user` 模块加组 (`append: yes`)，并紧跟 `meta: reset_connection` 强制刷新 SSH 长连接，无需重启节点。
  - **局部单步调试报错**：只想跑单个 Role，混用 YAML 语法报错。
    - 方案：从 `- role_name` 改为标准的字典格式 `- role: role_name` 并对齐加上 `tags: my_tag`，配合 `--tags` 精确执行。
- qa
  - ​        **Q1: 怎么保证批量配免密的安全和剧本幂等性？**      
    - 答：绝对不用 shell echo 强行覆盖。坚持用 `authorized_key` 模块。生产严禁 root 直连，全量用普通账号（如 vagrant）走 `become: yes` 提权。
  - ​        **Q2: 客户机房网络受限（被墙或纯离线），Ansible 怎么跑？**      
    - 答：公网受限就将源地址抽取为变量切到私服或国内源；纯离线就提前打包全量 `.deb` 依赖，用 `copy` 模块推下去做本地离线安装。
  - ​        **Q3: 几百个 Task 的剧本断在中间，怎么高效调试排障？**      
    - 答：拒绝全量无脑重跑。利用规范的 Tags 体系（按组件/动作打标），通过 `--tags` 或 `--start-at-task` 精准打击，控制爆炸半径。如果遇到极端不规范的老剧本，我会直接按认怂话术在测试环境打断点。