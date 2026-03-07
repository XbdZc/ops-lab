# Ansible

- 读取roles命令[[文件树]]
  - `find playbook -type f -exec echo -e "\n### File: {} ###" \; -exec cat {} \;`

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
