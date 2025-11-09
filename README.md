# This is a study note
based on a small personal cloud server with a ubuntu system.

这是一个关于Ubuntu服务器部署的详细笔记.

## 1. Linux基本操作

在安装任何软件前，建议先更新系统源和升级软件包，以确保系统是最新的状态。

```
# 安装任何东西之前：
sudo apt update
sudo apt upgrade -y
```

### 文件和目录操作

- 查看目录内容（解释：用于检查文件是否存在或列出目录内容）：
  ```
  # 查看是否存在
  ls -la ~/xxx/
  # 简单列出
  ls
  # 详细列表（显示权限、大小、时间）
  ls -l
  # 显示所有文件（包括隐藏文件）
  ls -a
  ```

- 创建目录和文件（解释：`-p`选项确保父目录存在时不报错，`nano`是简单文本编辑器）：
  ```
  # 创建目录
  mkdir -p ~/xxx/
  # 创建配置文件
  nano ~/xxx/config.json
  ```

- 删除文件和目录（解释：`-i`选项要求确认以避免误删，`-r`用于递归删除非空目录）：
  ```
  # 删除前确认（推荐新手使用）
  rm -i filename.txt
  # 删除空目录
  rmdir directory_name
  # 删除非空目录及其所有内容
  rm -r directory_name
  ```

### 安全删除示例

这些示例强调先预览再删除，以减少错误风险。

- 删除日志文件：
  ```
  # 先列出要删除的文件
  ls *.log
  # 确认后删除
  rm -i *.log
  # 删除备份文件
  rm backup_*
  ```

- 更高级的清理（解释：使用`grep`过滤查看，`find`自动删除旧文件）：
  ```
  # 1. 先查看要删除什么
  ls -la | grep test
  # 2. 确认后删除
  rm -i *test*
  # 3. 清理日志文件（保留最近7天）
  find . -name "*.log" -mtime +7 -delete
  ```

## 2. SSH登录配置

### 在控制台获得.pem的SSH key放在本地

使用PowerShell从Windows登录Ubuntu服务器（解释：`.pem`是SSH私钥文件，用于安全认证）。

```
ssh -i "C:\Users\用户\.ssh\xxxxxsshkey.pem" ubuntu@123456
```

更新系统后，禁用用户名密码登录以提升安全，只允许SSH密钥登录。

```
sudo apt update
sudo apt upgrade -y
```

### 禁用用户名密码登录（为了安全用SSH）

编辑SSH配置文件（解释：修改后只允许密钥认证，禁止root直接登录）。

```
sudo nano /etc/ssh/sshd_config
```

添加或修改为：
```
PasswordAuthentication no
PermitRootLogin prohibit-password
```

保存和退出nano编辑器：Ctrl + O 保存，Enter 确认，Ctrl + X 退出。Ctrl + W 用于查找。

如果修改未生效，检查配置：
```
sudo sshd -T | egrep -i "passwordauthentication|permitrootlogin|challenge|authenticationmethods|pubkeyauthentication"
sudo grep -R "PasswordAuthentication" /etc/ssh/sshd_config.d/
```

重启SSH服务：
```
sudo systemctl restart sshd
```

**注意**：在其他窗口测试新SSH登录有效后再退出当前会话，否则可能把自己锁在外面。

## 3. 配置防火墙UFW

检查是否安装UFW（解释：UFW是Ubuntu的简易防火墙工具）。

```
dpkg -l | grep ufw
```

已安装显示类似：
```
ii  ufw   0.36.2-2   amd64   program for managing a Netfilter firewall
```

未安装显示类似：
```
Command 'ufw' not found
```

安装UFW：
```
sudo apt update
sudo apt install ufw -y
```

检查状态：
```
sudo ufw status verbose
```

如果inactive（未激活），可以不开；如果要启用，先放行SSH端口（解释：否则SSH连接会被防火墙阻挡，导致锁死自己）。

```
sudo ufw allow ssh
```

放行其他端口（示例）：
```
# sudo ufw allow 80/tcp
# sudo ufw allow 443/tcp
```

启用UFW：
```
sudo ufw enable
```

在云提供商控制台查看安全组规则。如果不用Windows远程桌面，可以删除对应规则；可以将22端口入站规则限制为本地公网IP，只允许本人登录。

## 4. Docker安装

### 卸载旧的Docker

移除旧版本以避免冲突。

```
sudo apt remove docker docker-engine docker.io containerd runc
```

### 添加官方Docker GPG key和源

```
sudo apt update
sudo apt install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 安装Docker CE

```
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

### 启动并验证

启用并启动Docker服务，运行测试容器（解释：`hello-world`镜像用于验证安装成功）。

```
sudo systemctl enable docker
sudo systemctl start docker
sudo docker run hello-world
```

如果输出“Hello from Docker!”则成功；失败可能需配置镜像或IPv4。

### 添加镜像

编辑Daemon配置文件以使用国内镜像加速下载（解释：镜像源加快Docker镜像拉取速度）。

```
sudo mkdir -p /etc/docker
sudo nano /etc/docker/daemon.json
```

添加内容：
```
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com",
    "https://dockerproxy.com",
    "https://docker.m.daocloud.io"
  ]
}
```

重启Docker：
```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

验证镜像列表：
```
sudo docker info | grep Mirrors
```

## 5. Docker和Docker Compose的使用（以Nginx为例）

### Docker基本命令

不推荐单独使用Docker运行Nginx，而是用Compose。

拉取镜像：
```
# 单独拉取镜像
docker pull nginx:latest
```

创建容器（解释：`-d`后台运行，`--name`指定容器名，`-p`端口映射）：
```
#创建容器
docker run -d --name mynginx -p 80:80 nginx
```

查看和管理容器/镜像：
```
# 查看正在运行的容器
docker ps
# 查看所有容器（包括停止的）
docker ps -a
# 查看容器的详细配置和信息
docker inspect 容器名或容器ID
# 删除已停止的容器
docker rm 容器名或容器ID
# 查看本地已有镜像
docker images
# 删除所有未使用的镜像
docker image prune -a
# 删除特定镜像
docker rmi nginx:latest
```

### Docker Compose

安装Compose插件（解释：Compose用于管理多容器应用）。

```
sudo apt-get install docker-compose-plugin
docker compose version
```

新建目录并创建Compose文件：
```
mkdir ~/myapp && cd ~/myapp
nano docker-compose.yml
#(用这个文件名不用指定)
```

写入内容（示例Nginx配置）：
```
services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./html:/usr/share/nginx/html
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
```

启动：
```
docker compose up -d
#Docker Compose 默认只会读取 当前目录下名为 docker-compose.yml 的文件
#如果用其他文件名，例如docker-compose-test.yml，启动时：
docker-compose -f docker-compose-test.yml up -d
```

### Docker Compose常见命令

| 操作          | 命令                          | 说明                  |
|---------------|-------------------------------|-----------------------|
| 启动容器      | docker compose up -d          | 后台运行              |
| 查看状态      | docker compose ps             | 看哪些容器在运行      |
| 停止容器      | docker compose stop           | 仅停止，不删          |
| 重启容器      | docker compose restart        | 重启服务              |
| 删除容器      | docker compose rm             | 删除容器              |
| 删除所有容器  | docker compose down           | 删除所有容器与网络    |
| 删除所有容器和卷 | docker compose down -v     | 慎用，会删数据        |
| 查看日志      | docker compose logs -f        | 持续输出日志          |
| 更新镜像重启  | docker compose pull && docker compose up -d | 滚动更新服务 |

### 操作Docker容器中的软件

进入容器执行命令（解释：`exec -it`进入交互模式）。

```
# 1. 进入数据库容器
docker exec -it database_app1 bash
# 2. 现在你在容器内部，可以：
# --------------------------------------------------
# 连接到MySQL/MariaDB命令行
mysql -uroot -p你的密码
# 查看数据库文件
ls -la /var/lib/mysql/
# 退出容器
exit
```

直接执行命令不进入容器：
```
# 直接在容器中执行命令，然后返回宿主机
docker-compose exec database_app1 mysql -uroot -p密码 -e "SHOW DATABASES;"
docker-compose exec web_app nginx -t
docker-compose exec app cat /etc/os-release
```

### 在Docker容器中安装长期软件

在YML根目录创建Dockerfile（解释：自定义镜像，安装额外工具）。

```
FROM python:3.12
# 安装 nano
RUN apt-get update && apt-get install -y nano
# 先升级 pip（可选，但推荐）
RUN pip install -i https://mirrors.aliyun.com/pypi/simple/ \
    --trusted-host mirrors.aliyun.com \
    --timeout=120 \
    --upgrade pip
# 其他自定义（如安装 Python 库）
RUN pip install -i https://mirrors.aliyun.com/pypi/simple/ \
    --trusted-host mirrors.aliyun.com \
    --timeout=120 \
    requests mysql-connector-python pymysql
# 更新 apt 并安装 ping
RUN apt-get update && \
    apt-get install -y iputils-ping && \
    rm -rf /var/lib/apt/lists/*
# 设置工作目录
WORKDIR /scripts
# 拷贝你的 Python 脚本（可选）
# COPY ./scripts /scripts
# 默认命令（可修改）
CMD ["tail", "-f", "/dev/null"]
```

修改YML的image为build：
```
services:
  python-test:
    build: .  # 用 Dockerfile 构建自定义镜像
    container_name: python_test
    volumes:
      - ./scripts:/scripts
    working_dir: /scripts
    tty: true
```

构建并启动：
```
docker stop container_names && docker rm container_names
docker compose build  # 先构建镜像
docker compose up -d  # 启动
docker compose -f xxx.yml build
docker compose -f xxx.yml up -d
```

### 关于容器间网络问题

要写到docker-compose.yml里面，写完了容器要删掉重建（解释：使用自定义网络实现容器间通信）。

```
# 查看现有网络
docker network ls
# 如果还没创建：
docker network create dev-net
```

### 修改数据库密码，删除卷

在mariadb根目录下：
```
docker compose down
```

删除卷（需输入sudo密码）：
```
sudo rm -rf ./mariadb_data
```

修改YML后重新创建：
```
docker compose up -d
```

## 6. Claude Code安装和Git

### Claude Code安装

先安装Node.js（解释：Claude Code依赖Node.js运行）。

```
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo bash -
sudo apt-get install -y nodejs
```

Claude Code本体：参考https://docs.claude.com/en/docs/claude-code/quickstart

Claude Code Router：参考https://github.com/musistudio/claude-code-router

### Linux上配置GitHub SSH密钥

检查是否已有密钥：
```
ls -al ~/.ssh
```

如果看到：
```
id_rsa
id_rsa.pub
#或者
id_ed25519
id_ed25519.pub
```

则已有，可跳过生成。

生成新密钥：
```
ssh-keygen -t ed25519 -C "GitHub邮箱"
```

路径默认，回车；密码空，回车两次。

复制公钥：
```
cat ~/.ssh/id_ed25519.pub
```

输出类似：
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIE4kQ3K... 你的邮箱
```

在GitHub设置中添加公钥：Title如“Ubuntu Cloud Server”，Key type: Authentication Key，粘贴公钥。

测试连接：
```
ssh -T git@github.com
```

第一次提示确认yes，成功输出：
```
Hi your-username! You've successfully authenticated, but GitHub does not provide shell access.
```

配置Git用户：
```
git config --global user.name "GitHub用户名"
git config --global user.email "GitHub邮箱"
```

查看：
```
git config --global --list
```

Clone仓库（SSH方式）：
```
git clone git@github.com:用户名/仓库名.git
```

### Git上传功能隐藏环境变量中的密码

把密码放在项目根目录的.env：
```
# ~/srv/www/project1/.env
DATABASE_HOST=localhost
DATABASE_USER=project1
DATABASE_PASSWORD=verysecretpassword
```

把.env加入.gitignore：
```
# 忽略所有 .env 文件
*.env
backend/.env
frontend/.env
# 忽略 Docker Compose 生成的临时文件
**/docker-compose.override.yml
**/docker-compose.override.yaml
# 忽略日志
**/logs/
*.log
# 忽略 Python 缓存文件
__pycache__/
*.pyc
# 忽略虚拟环境
venv/
.env/
# 忽略编译产物
dist/
build/
```

确保权限：
```
chmod 600 ~/srv/www/project1/.env
```

### Git的备份、分支、撤回

假设项目目录/path/to/project。

进入并确认Git：
```
cd /path/to/project
git --version   # 确认安装
```

初始化并首次提交：
```
git init
git add -A
git commit -m "chore: initial commit — project baseline"
# （可选）把默认分支设为 main
git branch -M main
```

查看当前分支：
```
git branch
```

查看远端：
```
git remote -v
# 输出类似：
# origin  https://github.com/you/repo.git (fetch)
# origin  https://github.com/you/repo.git (push)
# 设置远端（如果还没设置）
# https
git remote add origin https://github.com/you/xxx.git
# ssh(如果设置了ssh密匙就用这个)
git remote add origin git@github.com:you/xxx.git
```

创建分支A并提交：
```
git checkout -b branch-A
#这会从当前 main（即最新提交）新建并切换到 branch-A。你现在在 branch-A 上。
# 假设你/Claude 修改了文件，现在保存
git add -A
# 注意此处的-A指的是所有改动，包括当前目录之外的
git add .   #此处语句为仅仅目录之下的
git commit -m "A-step1: 描述第一步改动"
# 假设你/Claude 修改了文件，但是不满意，丢弃工作区所有未提交改动（回到上次 commit）注意语句后面有个点
git restore .  
# 修改文件...
git add -A
git commit -m "A-step2: 描述第二步改动"
```

查看历史：
```
git log --oneline --graph --decorate
```

输出类似：
```
c3f1a2b (HEAD -> branch-A) A-step3: 描述第三步改动
b2e1d0c A-step2: 描述第二步改动
a1d2c3e A-step1: 描述第一步改动
... (main 的提交)
```

创建分支B：
```
# 先回到 main（从 main 创建 B，确保 B 与 A 的改动独立）：
git checkout main
#创建分支 B：
git checkout -b branch-B
#branch-B 上做n个保存点（每次 Claude 改完就 commit）：
# B-step1
git add -A
git commit -m "B-step1: 描述"
# B-step2
git add -A
git commit -m "B-step2: 描述"
```

回到分支某一步：
```
git checkout branch-A
git log --oneline
```

输出类似：
```
c3f1a2b A-step3: ...
b2e1d0c A-step2: ...
a1d2c3e A-step1: ...
```

基于特定提交创建新分支：
```
# 在 branch-A 上或任意分支都可以执行
git branch feature-from-A-step2 b2e1d0c   # 仅创建分支，不切换
git checkout feature-from-A-step2          # 切换到新分支
# 或一步到位：
# git checkout -b feature-from-A-step2 b2e1d0c
```

删除分支：
```
git branch -d try-A-step2           # 若已合并则删除成功
git branch -D try-A-step2           # 强制删除（危险）
# 若远端有备份，删除远端：
git push origin --delete try-A-step2
```

临时查看（detached HEAD）：
```
git checkout b2e1d0c
# 现在处于 detached HEAD 状态：可以查看、跑程序、测试，但不要做长期提交（除非创建新分支）
```

合并到main：
普通合并：
```
git checkout main
git merge feature-from-A-step2
```

压缩合并：
```
git checkout main
git merge --squash feature-from-A-step2
git commit -m "feat: 从 A-step2 衍生的改动 —— 描述"
```

推送前pull：
```
git checkout main
git pull origin main
# （merge操作）
# ...
# 推送到远端
git push origin main
```

## 7. MariaDB数据库

以宿主上一个Docker容器中的单实例为例。

创建目录并配置Compose：
```
mkdir ~/mariadb-docker
cd ~/mariadb-docker
nano docker-compose.yml
#(用这个文件名不用指定)
```

内容：
```
services:
  mariadb:
    image: mariadb:11.8
    container_name: mariadb_docker  # 容器名字，方便管理
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: xxx!  # root 管理员密码
      MYSQL_DATABASE: mydb  # 自动创建的初始数据库
      MYSQL_USER: user1  # 新建普通用户
      MYSQL_PASSWORD: xxx!  # 普通用户密码
    ports:
      - "127.0.0.1:3306:3306"  # **只允许服务器本地访问**，外网无法访问
    volumes:
      - ./mariadb_data:/var/lib/mysql  # 持久化数据，宿主机上的卷
      - ./conf.d/custom.cnf:/etc/mysql/conf.d/custom.cnf:ro
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci  # 设置默认字符集
    networks:
      - dev-net
volumes:
  mariadb_data:  # 定义宿主机卷，用来存储数据库文件
networks:
  dev-net:
    external: true
```

启动：
```
#启动
docker compose up -d
#Docker Compose 默认只会读取 当前目录下名为 docker-compose.yml 的文件
#如果用其他文件名，例如docker-compose-test.yml，启动时：
docker-compose -f docker-compose-test.yml up -d
```

查看和进入：
```
# 查看容器状态
docker ps
# 进入容器
docker exec -it mariadb_local bash
```

## 8. 小型云服务器架构

以2核4G内存6M带宽为例：1个MariaDB容器、1个统一Nginx反向代理容器、每个后端1个容器（按需启动），前端静态文件由Nginx托管。

示例目录结构：
```
~/srv/
├── mariadb-docker/
│   └── docker-compose.yml
├── nginx-docker/
│   ├── docker-compose.yml
│   ├── conf.d/
│   ├── certs/
│   └── logs/
└── www/
    └── project1/
        ├── .git/
        ├── .gitignore
        ├── backend/
        │   ├── .env    
        │   └── docker-compose.yml
        └── frontend/
```

更详细结构：
```
~/srv/
├── mariadb-docker/
│   └── docker-compose.yml
├── nginx-docker/
│   ├── docker-compose.yml
│   ├── conf.d/
│   ├── certs/
│   └── logs/
└── www/
    ├── xxxx/
    │   └── frontend/
    │       ├── index.html
    │       ├── css/
    │       └── images/
    │
    ├── xxxx/
    │   └── frontend/
    │       └── index.html
    │
    ├── xxxx/
    │   └── frontend/
    │       ├── index.html     
    │       ├── html_game1/
    │       │   └── index.html
    │       └── html_game2/
    │           └── index.html
    │
    ├── xxxx/
    │   └── frontend/
    │       ├── experiment1.html
    │       └── ...
    │
    ├── xxxx/    
    │   ├── .git/
    │   ├── .gitignore
    │   ├── backend/
    │   │   ├── .env    # (DB_NAME=member_db, SECRET_KEY=...)
    │   │   ├── docker-compose.yml
    │   │   └── app/
    │   │       ├── main.py    
    │   │       └── ...
    │   └── frontend/
    │       ├── auth/
    │       │   ├── login.html
    │       │   └── register.html
    │       └── index.html     
    │
    └── private_tools/   
        ├── .git/
        ├── .gitignore
        ├── backend/
        │   ├── .env    # (DB_NAME=private_db, SECRET_KEY=...)
        │   ├── docker-compose.yml
        │   └── app/
        │       └── main.py    
        └── frontend/
            └── index.html     
```

## 9. Nginx Web服务器

### Docker Compose配置

```
services:
  nginx:
    image: nginx:stable
    container_name: nginx_proxy
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /home/ubuntu/srv/nginx-docker/conf.d:/etc/nginx/conf.d:ro
      - /home/ubuntu/srv/www:/var/www:ro
      - /home/ubuntu/srv/nginx-docker/certs:/etc/letsencrypt         # letsencrypt 证书存放（可选）
      - /home/ubuntu/srv/nginx-docker/logs:/var/log/nginx
    networks:
      - dev-net
networks:
  dev-net:
    external: true
```

### 仅80端口Nginx配置（~/srv/nginx-docker/conf.d/default.conf）

```
server {
    listen 80;
    server_name _;  # 或 公网ip
    location / {
        root /var/www/xxxx;  # 指向 ~/srv/www/calendar/
        index index.html index.htm;
        try_files $uri $uri/ =404;
    }
    error_log /var/log/nginx/error.log warn;
}
```

### 443端口（需证书）

```
server {
    listen 80;
    server_name project1.example.com;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl;
    server_name project1.example.com;
    ssl_certificate /etc/letsencrypt/live/project1.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/project1.example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf; # 如果有 certbot 的配置
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
    root /var/www/project1;
    index index.html;
    # 静态文件优先
    location / {
        try_files $uri $uri/ /index.html;
    }
    # /api 转发到内部容器（Docker network 内的容器名）
    location /api/ {
        proxy_pass http://project1_backend:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 国内IP的域名备案

备案成功前不要DNS解析域名！如果已解析，删除解析。

### HTTPS Certbot证书（ICP备案后）

修改Nginx配置文件（域名.conf）：
```
server {
    listen 80;
    # 1. 替换成您的域名
    server_name 域名 www.域名;
    # 2. 为 Certbot 验证准备的特殊路径
    # 当 Let's Encrypt 访问 http://.../.well-known/acme-challenge/ 时
    # Nginx 会从这个目录提供文件
    location /.well-known/acme-challenge/ {
        root /var/www-challenge;
    }
    # 3. 您的网站前端（保持您原来的设置）
    location / {
        root /var/www/default;  # 这是您上次修改的路径
        index default.html default.htm; # 这是您上次修改的 index
        try_files $uri $uri/ =404;
    }
    # 4. 错误日志
    error_log /var/log/nginx/error.log warn;
}
```

集成Certbot（修改nginx-docker的Compose）：
```
services:
  nginx:
    image: nginx:stable
    container_name: nginx_proxy
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /home/ubuntu/srv/nginx-docker/conf.d:/etc/nginx/conf.d:ro
      - /home/ubuntu/srv/www:/var/www:ro
      - /home/ubuntu/srv/nginx-docker/certs:/etc/letsencrypt         # letsencrypt 证书存放（可选）
      - /home/ubuntu/srv/nginx-docker/logs:/var/log/nginx
      # [新增] Certbot 验证文件的共享目录 (Nginx 只读)
      - /home/ubuntu/srv/nginx-docker/letsencrypt-challenge:/var/www-challenge:ro
    networks:
      - dev-net
# [新增] Certbot 服务定义
  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      # 共享证书目录
      - /home/ubuntu/srv/nginx-docker/certs:/etc/letsencrypt
      # 共享验证目录 (Certbot 需要写入)
      - /home/ubuntu/srv/nginx-docker/letsencrypt-challenge:/var/www-challenge
    # Certbot 只是一个工具，不需要端口或重启策略
networks:
  dev-net:
    external: true
```

重启Nginx：
```
docker compose up -d --force-recreate nginx
```

运行Certbot：
```
docker compose run --rm certbot certonly \
    --webroot \
    -w /var/www-challenge \
    -d 域名 \
    -d www.域名 \
    --email your-email@example.com \
    --agree-tos \
    --no-eff-email
```

启用HTTPS最终配置：
```
cd ~/srv/nginx-docker/conf.d/
nano 域名.conf
```

内容：
```
# 1. HTTP (80端口) -> 强制跳转到 HTTPS (443端口)
server {
    listen 80;
    server_name 域名 www.域名;
    # Certbot 续期时仍然需要这个 (路径要正确！)
    location /.well-known/acme-challenge/ {
        root /var/www-challenge;
    }
    # 其他所有请求，永久重定向 (301) 到 HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}
# 2. HTTPS (443端口) 的主服务
server {
    listen 443 ssl http2;
    server_name 域名 www.域名;
    # 证书文件路径 (容器内路径)
    ssl_certificate /etc/letsencrypt/live/域名/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/域名/privkey.pem;
    # (可选) 提高安全性的 SSL 配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers on;
    # 您的网站前端
    location / {
        root /var/www/default; # 您的网站根目录
        index default.html default.htm;
        try_files $uri $uri/ =404;
    }
    # (占位符) 您未来的后端 API 反向代理
    # location /api {
    #    proxy_pass http://backend_app:8000; 
    #    ...
    # }
    # 错误日志
    error_log /var/log/nginx/error.log warn;
}
```

重载Nginx：
```
docker compose exec nginx nginx -s reload
```

设置证书自动续期：创建renew.sh：
```
nano renew.sh
```

内容：
```
#!/bin/bash
# 切换到 docker-compose.yml 所在的目录
cd /home/ubuntu/srv/nginx-docker/
# 1. 尝试续期证书
# 注意：我们使用 /usr/bin/docker 来确保 cron 能找到 docker 命令
/usr/bin/docker compose run --rm certbot renew
# 2. 重载 Nginx 以应用新证书（如果续期了的话）
/usr/bin/docker compose exec nginx nginx -s reload
```

权限：
```
chmod +x renew.sh
```

添加Cron任务：
```
sudo crontab -e
```

添加：
```
# 每天凌晨 3:30 尝试续期 SSL 证书，并把日志写到 cron.log
30 3 * * * /home/ubuntu/srv/nginx-docker/renew.sh > /home/ubuntu/srv/nginx-docker/logs/cron.log 2>&1
```

## 10. 本地Windows端开发环境设置

### 安装WSL2

PowerShell（管理员）：
```
# powershell（管理员模式）
wsl --install
# 查看可用的 Linux 版本：
wsl --list --online
#安装你想要的版本，比如 Ubuntu 24.04：
wsl --install -d Ubuntu-24.04
#设置某个 Ubuntu 版本为默认 WSL2
wsl --set-default Ubuntu-24.04
# 确认是不是 WSL2（而不是 WSL1）
wsl --list --verbose
```

进入WSL家目录：
```
wsl ~
```

### WSL中的Docker配置

在Windows安装Docker Desktop，设置WSL Integration启用Ubuntu。

验证：
```
docker --version
docker ps
# 如果出现“permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.51/containers/json": dial unix /var/run/docker.sock: connect: permission denied”
#添加用户到docker group
sudo usermod -aG docker $USER
#重启终端
```

WSL用国内镜像：在Docker Desktop设置Docker Engine JSON添加：
```
{
  "registry-mirrors": [
    "https://mirror.ccs.tencentyun.com"  ]
}
```

### 测试Nginx

```
cd ~/srv/nginx-docker
nano conf.d/default.conf
```

测试conf：
```
server {
    listen 80;
    server_name localhost;
    # 这个 location 块告诉 Nginx 去哪里找默认的欢迎页面
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
    # 简单的错误页面配置
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

启动：
```
# 这会移除旧的容器
docker compose down
# 这会使用新的配置来启动一个新容器
docker compose up -d
```

Windows访问：
```
#windows上访问：
http://localhost
```

### WSL直接用宿主代理

在~/.wslconfig添加（Windows 11 22H2+）：
```
[wsl2]
networkingMode = mirrored
autoproxy = true
```

关闭WSL：
```
wsl --shutdown
```

### 传文件到WSL

文件资源管理器输入：
```
\\wsl$
```

拖拽文件。

### 用Windows的VSCode连接WSL2

下载Remote - Development插件，点击左下绿色按钮连接。

### 本地WSL2的调试

本地用80端口，无需HTTPS。

Nginx配置local.conf：
```
# ----------------------------------------------------
#  local.conf (仅供本地 http://localhost 开发测试使用)
# ----------------------------------------------------
server {
    listen 80;
    server_name localhost; # <--- 关键改动
    # (我们假设您的项目目录们在 ~/srv/www/ 下)
    # 比如：/home/ubuntu/srv/www/resume/frontend/
    # Nginx 容器会把 /home/ubuntu/srv/www 挂载为 /var/www
    # 根目录 (/)，指向您的简历或作品集
    location = / {
        alias /var/www/resume/frontend/index.html; 
    }
    # 作品集 (/portfolio/)
    location /portfolio/ {
        alias /var/www/portfolio/frontend/;
        try_files $uri $uri/ /portfolio/frontend/index.html;
    }
    # 简历 (/resume/)
    location /resume/ {
        alias /var/www/resume/frontend/;
        try_files $uri $uri/ /resume/frontend/index.html;
    }
    # ... (您其他的静态项目 location) ...
    # ----------------------------------------------------
    # (B) 动态项目 (会员工具，即 project1 或 member_tools)
    # ----------------------------------------------------
    # 会员工具的前端页面 (比如 /tools/auth/login.html)
    location /tools/ {
        alias /var/www/member_tools/frontend/;
        try_files $uri $uri/ /tools/frontend/index.html; 
    }
    # 会员工具的后端 API (登录、注册、工具API)
    location /api/auth/ {
        proxy_pass http://member_tools_backend:8000; # <--- 转发给本地后端容器
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    location /api/tools/ {
        proxy_pass http://member_tools_backend:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    # ... (您其他的后端项目 location) ...
}
```

加入.gitignore：
```
# 忽略本地开发配置文件
conf.d/local.conf
```

本地启动测试：
```
# 数据库
cd ~/srv/mariadb-docker
docker compose up -d
# 代理
cd ~/srv/nginx-docker
docker compose up -d
# 后端
cd ~/srv/www/member_tools/backend
docker compose up -d
```

浏览器访问：
```
http://localhost/xxx/frontend/index.html
```

## 11. 新项目初始化

### 新建目录结构

### 创建新数据库

```
# 确保在正确的目录下
cd ~/srv/mariadb-docker
# 进入容器，并直接启动 mysql 客户端
docker compose exec mariadb mariadb -u root -p
```

SQL命令：
```
CREATE DATABASE project1_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
GRANT ALL PRIVILEGES ON project1_db.* TO 'user1'@'%';
FLUSH PRIVILEGES;
exit
```

### 更新.env

在backend下创建：
```
nano .env
```

内容：
```
# .env 文件内容
# 数据库连接信息
DB_HOST=mariadb # Docker Compose 服务名
DB_NAME=project1_db  # <--- 数据库名
DB_USER=user1              # <--- 用户名
DB_PASSWORD=a_very_secret_password_for_user1 # <--- 密码
# JWT 密钥 (这个是新的，自己设置一个复杂的字符串)
SECRET_KEY=a_new_random_strong_secret_for_your_app
```

在根目录.gitignore：
```
nano .gitignore
```

内容：
```
# .gitignore 文件内容示例
# 忽略所有 .env 文件
*.env
backend/.env
frontend/.env
# 忽略 Docker Compose 生成的临时文件
**/docker-compose.override.yml
**/docker-compose.override.yaml
# 忽略日志
**/logs/
*.log
# 忽略 Python 缓存文件
__pycache__/
*.pyc
# 忽略虚拟环境
venv/
.env/
# 忽略编译产物
dist/
build/
# Node.js
node_modules/
```

### 设置CLAUDE.md

示例：
```
## 主要目标
## 服务器环境
- Ubuntu 24.04，腾讯轻量应用服务器（2核心、4GB内存、6M峰值带宽）
- mariadb的docker容器已经在其他地方创建好并已经在运行了，所以不要在项目中创建mariadb容器。请直接使用现成的：
服务：
容器：
数据库：
用户：
网络：
- 反向代理nginx的docker容器已经在其他地方创建好并已经在运行了，所以不要在项目中创建nginx容器。nginx反向代理的配置如下：
## 项目结构
请严格遵守项目结构：
## 技术栈
- 后端：
- 前端：
- 数据库：
- 反向代理：
## 开发标准
编写代码时，请遵循以下原则：
- 使用context7 mcp以应用最新的示例和文档。
- 优先考虑简洁性和易读性，避免复杂依赖。
- 从最小功能入手，验证其有效性后再增加复杂性。
- 保持核心逻辑清晰，并将实现细节推到边缘。
- 在整个代码库中保持一致的风格（缩进、命名、模式）。
- 适时使用github mcp连接到GitHub平台来读取代码库和代码文件和PR、分析代码以及自动化工作流程。
- 适时使用chrome-devtools mcp调试网页。
## 环境变量
- xxxx/.env是环境变量文件，包含密匙，请勿读取。
- 请直接使用xxxx/.env，其中变量：
# 默认版本
Python: 3.12
## 互动规则
使用中文和我交流。
```

### Git初始化

（解释：后续开发使用Git管理代码。）

## 12. Claude Code使用

### 忽略涉密文件

settings.json（解释：权限设置，避免读取敏感文件）：
```
{
  "permissions": {
    "deny": [
      "Read(**/.env*)",
      "Read(./**/.env*)",
      "Read(~/.env*)",
      "Read(**/.aws/**)",
      "Read(~/.aws/**)",
      "Read(**/.ssh/**)",
      "Read(~/.ssh/**)",
      "Read(./secrets/**)",
      "Read(./secret_keys/**)",
      "Read(**/*.pem)",
      "Read(**/*.key)",
      "Read(**/credentials.json)",
      "Read(**/database.yml)",
      "Bash(curl:*)",
      "Bash(wget:*)",
      "WebFetch"
    ],
    "allow": [
      "Bash(git status)",
      "Bash(npm install)",
      "Bash(npm run test)"
    ],
    "ask": [
      "Bash(git push:*)",
      "Bash(git commit:*)",
      "Bash(rm:*)",
      "Bash(mv:*)",
      "Edit(./backend/**)",
      "Edit(./frontend/**)"
    ]
  }
 }
```

### MCP

使用时注意涉密问题。参考https://docs.claude.com/en/docs/claude-code/mcp

常用MCP示例：
```
claude mcp add --transport http context7 https://mcp.context7.com/mcp --header "CONTEXT7_API_KEY: YOUR_API_KEY"
claude mcp add chrome-devtools npx chrome-devtools-mcp@latest
claude mcp add --transport http github https://api.githubcopilot.com/mcp -H "Authorization: Bearer YOUR_GITHUB_PAT"
```

## 13. 以登录系统为例的开发流程

流程总结：VSCode+WSL > Git > 服务器调试。

详细流程：

后端Compose环境变量（解释：从.env加载变量，连接外部网络）：
```
services:
  后端服务名称自定义一个:
    # ... 你的后端服务定义 ...
    environment:
      - DB_HOST=${DB_HOST} # mariadb的Docker Compose 服务名
      - DB_NAME=${DB_NAME}  # <--- 数据库名
      - DB_USER=${DB_USER}              # <--- 用户名
      - DB_PASSWORD=${DB_PASSWORD}
      - SECRET_KEY=${SECRET_KEY}
    networks:
      - dev-net  # 连接到外部网络
networks:
  dev-net:
    external: true # 声明 dev-net 是一个外部已存在的网络
```

### 上线，部署到服务器

Git到服务器：
```
cd ~/srv/www/
# (把 [ Git 仓库地址] 换成您自己的)
git clone [您的 Git 仓库地址] xxxxx
# (如果已经克隆过了，就进入目录并拉取最新代码)
# cd member_tools
# git pull origin main
```

创建生产数据库：
```
cd ~/srv/mariadb-docker
# 使用您服务器的 root 密码登录
docker compose exec mariadb mariadb -u root -p
```

SQL：
```
CREATE DATABASE xxx_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
-- (假设您服务器的 user1 用户已存在)
GRANT ALL PRIVILEGES ON xxx_db.* TO 'user1'@'%';
FLUSH PRIVILEGES;
exit
```

创建生产.env：
```
# 到对应路径下
nano .env
```

内容：
```
# --- 生产环境数据库连接 ---
DB_HOST=mariadb
DB_NAME=xxx_db
DB_USER=user1
# (!!!) 填入您在 *服务器* mariadb-docker/docker-compose.yml 中
# 为 user1 设置的那个“高强度”密码
DB_PASSWORD=YOUR_STRONG_PRODUCTION_PASSWORD 
# --- 生产环境安全密钥 ---
# (!!!) 绝不要用 'jwt'。请在这里生成一个64位长的随机字符串
# (您可以在本地用工具生成，或者随手打一长串)
SECRET_KEY=A_VERY_LONG_AND_RANDOM_SECRET_KEY_f0r_HS256_!@#$
```

启动后端：
```
docker compose up -d --build
docker compose ps
```

如果网络卡住，使用镜像。

### 配置Nginx

直接问AI配置。本地WSL2（忽略域名.conf、override.yml）：
```
nginx-docker/
├── docker-compose.yml              # Docker Compose 主配置文件
├── docker-compose.override.yml     # 为空，实际在服务器上添加专属服务器的内容， 本地开发环境覆盖配置（可选）
├── conf.d/                         # Nginx 配置文件目录
│   ├── local.conf                  # 本地开发环境配置（HTTP）
│   ├── 域名.conf.production  # 生产环境配置（HTTPS，需在服务器上重命名）
│   └── includes/                   # 共享配置片段
│       ├── common_locations.conf   # 通用 location 块配置
│       └── proxy_params.conf       # 代理参数配置
├── logs/                           # Nginx 访问和错误日志
└── README.md                       # 本文档
```

.gitignore：
```
# ----------------------------------
# .gitignore for nginx-docker
# ----------------------------------
# 1. 忽略本地开发专属的 "外壳" 配置文件
conf.d/local.conf
# 2. 忽略 Nginx 运行时生成的日志文件
/logs/
*.log
# 3. 忽略 Certbot 运行时生成的临时文件
/letsencrypt-challenge/
# 4. 忽略 SSL 证书
/certs/
# 忽略所有 .env 文件 (一个好的安全兜底)
*.env
# 忽略服务器专属的容器内证书相关内容
docker-compose.override.yml
# 忽略服务器专属的证书续期脚本
renew.sh
conf.d/域名.conf
```

服务器结构：
```
nginx-docker/
├── docker-compose.yml              # Docker Compose 主配置文件
├── docker-compose.override.yml     # 服务器上添加专属服务器的内容， 本地开发环境覆盖配置（可选）
├── conf.d/                         # Nginx 配置文件目录
│   ├── local.conf                  # 本地开发环境配置（HTTP）
│   ├── 域名.conf          # 生产环境配置（HTTPS，需在服务器上重命名）
│   └── includes/                   # 共享配置片段
│       ├── common_locations.conf   # 通用 location 块配置
│       └── proxy_params.conf       # 代理参数配置
├── certs/                          # SSL 证书存放目录（Let's Encrypt）
├── logs/                           # Nginx 访问和错误日志
└── README.md                       # 本文档
```

本地上传仓库，服务器拉取。

### 验证上线
