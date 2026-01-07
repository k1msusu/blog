---
layout:	post
title:	"Welcome to monitoring!!"
date:	2026-01-06 20:20:10 +0800
categories: jekyll update
---
关于系统监控和日志分析的三个工具：Zabbix、Prometheus、ELK

## Zabbix
* Zabbix -> Maridb -> Apache -> Nginx -> Agent

### localhost管理

准备环境（要注意 zabbix 和数据库是否相互支持，以及数据库版本的语法改变 ）

```
# 关闭防火墙和 Selinux
systemctl stop firewalld && systemctl disable firewalld
setenforce 0 && sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config

# 安装zabbix官方源
# CentOS 7
rpm -ivh https://repo.zabbix.com/zabbix/6.4/rhel/7/x86_64/zabbix-release-6.4-1.el7.noarch.rpm
# CentOS 8
rpm -ivh https://repo.zabbix.com/zabbix/6.4/rhel/8/x86_64/zabbix-release-6.4-1.el8.noarch.rpm
yum clean all && yum makecache

# 安装 MySQL/MariaDB（选其一，推荐 MariaDB）
yum install -y mariadb-server mariadb
# 安装 Web 环境（Nginx + PHP 及扩展）
# 如果使用 apache 的话 httpd ,路径多了/zabbix
yum install -y nginx php php-mysqlnd php-gd php-xml php-bcmath php-mbstring php-ldap php-json
# 安装 Zabbix 核心依赖
yum install -y zabbix-server-mysql zabbix-web-mysql zabbix-nginx-conf zabbix-agent

```

配置数据库

```
# 初始化数据库，可能遇到初始密码的情况，先创建账户密码
systemctl start mariadb && systemctl enable mariadb
mysql_secure_installation  # 按提示设置 root 密码，删除匿名用户
# 创建 zabbix 数据库和用户
mysql -uroot -p
# 执行 SQL 语句
create database zabbix character set utf8mb4 collate utf8mb4_bin;
create user zabbix@localhost identified by 'Zabbix@123456';
grant all privileges on zabbix.* to zabbix@localhost;
flush privileges;
exit;
# 导入 zabbix数据库
# 导入数据库时，注意看数据库信息是否被拆分以及数据库文件所在位置
zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -uzabbix -p zabbix # 未被拆分
# 输入 Zabbix 数据库用户密码
# 一键按顺序导入核心脚本（密码直接传入，无需手动输入）
mysql -uzabbix -pZabbix@123456 zabbix < schema.sql && mysql -uzabbix -pZabbix@123456 zabbix < images.sql && mysql -uzabbix -pZabbix@123456 zabbix < data.sql # 被拆分
```

配置zabbix组件

```
# 配置 /etc/zabbix/zabbix_server.conf 
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=Zabbix@123456  # 填写实际密码
# 可选优化参数（生产环境）
CacheSize=256M
StartPollers=50
Timeout=15

# 配置 PHP 时区
date.timezone = Asia/Shanghai

# 配置nginx 反向代理
server {
    listen 80;
    server_name your-server-ip;  # 替换为宿主机 IP/域名

    root /usr/share/zabbix;
    index index.php;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

启动服务

```
systemctl start php-fpm && systemctl enable php-fpm
systemctl start nginx && systemctl enable nginx
systemctl start zabbix-server zabbix-agent && systemctl enable zabbix-server zabbix-agent
```

需要监控其他服务器的 Agent

```
# 添加官方源
yum install -y zabbix-agent # 安装 Agent

# 配置 Agent 由于 Zabbix 安装后默认配置仅适配本机监控，宿主机不需手动配置
vim /etc/zabbix/zabbix_agentd.conf
# 1. Zabbix Server 的 IP 或主机名
Server=192.168.1.100  # 替换为你的 Zabbix Server IP
# 2. 被动模式下，允许接收的 Server 地址（和上面一致）
ServerActive=192.168.1.100
# 3. 目标服务器的主机名（需和 Web UI 添加时的名称一致）
Hostname=Target-Server-01  # 自定义，如 web-server-01
# 4. 可选：开启远程命令（如需执行脚本/命令）
EnableRemoteCommands=1
UnsafeUserParameters=1
```

启动 Agent 服务

```
# 启动 Agent
systemctl start zabbix-agent
# 设置开机自启
systemctl enable zabbix-agent
# 检查状态（确保 running）
systemctl status zabbix-agent
# 放行防火墙 10050 端口（Zabbix Agent 端口）
firewall-cmd --add-port=10050/tcp --permanent
firewall-cmd --reload
```

### Container

```
# Zabbix + MySQL + Nginx Web UI 生产级 Docker Compose 编排
# 持久化存储、时区统一、资源限制，适配生产环境
version: '3.8'

services:
  # MySQL 数据库服务（Zabbix 后端存储）
  mysql-server:
    image: mysql:8.0
    container_name: zabbix-mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: "Root@123456"
      MYSQL_DATABASE: "zabbix"
      MYSQL_USER: "zabbix"
      MYSQL_PASSWORD: "Zabbix@123456"
      TZ: "Asia/Shanghai"
    volumes:
      - ./mysql-data:/var/lib/mysql
      - ./mysql-conf:/etc/mysql/conf.d
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --default-authentication-plugin=mysql_native_password
    deploy:
      resources:
        limits:
          cpus: '0.8'
          memory: 1G
    networks:
      - zabbix-net
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-pRoot@123456"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Zabbix Server 核心服务
  zabbix-server:
    image: zabbix/zabbix-server-mysql:alpine-latest
    container_name: zabbix-server
    restart: always
    depends_on:
      mysql-server:
        condition: service_healthy
    environment:
      TZ: "Asia/Shanghai"
      DB_SERVER_HOST: "mysql-server"
      MYSQL_DATABASE: "zabbix"
      MYSQL_USER: "zabbix"
      MYSQL_PASSWORD: "Zabbix@123456"
      ZBX_CACHESIZE: "256M"
      ZBX_STARTPOLLERS: "50"
      ZBX_TIMEOUT: "15"
      ZBX_HISTORYCACHESIZE: "128M"
      ZBX_TRENDCACHESIZE: "64M"
    volumes:
      - ./zabbix-alertscripts:/usr/lib/zabbix/alertscripts
      - ./zabbix-exportscripts:/usr/lib/zabbix/exportscripts
      - ./zabbix-modules:/usr/lib/zabbix/modules
    ports:
      - "10051:10051"
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1.5G
    networks:
      - zabbix-net

  # Zabbix Web UI（Nginx + PHP）
  zabbix-web-nginx:
    image: zabbix/zabbix-web-nginx-mysql:alpine-latest
    container_name: zabbix-web-nginx
    restart: always
    depends_on:
      - zabbix-server
    environment:
      TZ: "Asia/Shanghai"
      DB_SERVER_HOST: "mysql-server"
      MYSQL_DATABASE: "zabbix"
      MYSQL_USER: "zabbix"
      MYSQL_PASSWORD: "Zabbix@123456"
      ZBX_SERVER_HOST: "zabbix-server"
      PHP_TZ: "Asia/Shanghai"
      ZBX_SERVER_PORT: "10051"
    ports:
      - "80:8080"
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    networks:
      - zabbix-net

  # Zabbix Agent（监控容器宿主机）
  zabbix-agent:
    image: zabbix/zabbix-agent:alpine-latest
    container_name: zabbix-agent
    restart: always
    privileged: true
    environment:
      TZ: "Asia/Shanghai"
      ZBX_SERVER_HOST: "zabbix-server"
      ZBX_HOSTNAME: "Zabbix-Server-Host"
    volumes:
      - /proc:/data/proc:ro
      - /sys:/data/sys:ro
      - /var/run:/var/run
    networks:
      - zabbix-net

networks:
  zabbix-net:
    driver: bridge

volumes:
  mysql-data:
  zabbix-alertscripts:
  zabbix-exportscripts:
```

```
cd ./zabbix-prod
vim Bocker-compose.yml
docker compose up -d 启动所有服务
http://ip 
账号 Admin，密码 zabbix
敏感密码建议通过 Docker Secrets 或环境变量文件（.env）注入，避免明文
```



## Prometheus
* Grafana -> Data Sources -> Dashboards -> Import

### Localhost

配置 Grafana

```
# 添加yum源
cat > /etc/yum.repos.d/grafana.repo << EOF
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF
# 安装指定版本
yum clean all && yum makecache fast
yum install -y grafana-9.5.17
# 设置开机自启
systemctl start grafana-server
systemctl enable grafana-server
systemctl status grafana-server
ss -tnlp | grep 3000
```

配置 prometheus + nginx-explore-prometheus

```
# 确认安装 nginx 和 stub_status 模块
# stub_status 为 nginx 内置模块，精简版可能未安装
yum install -y nginx
# 开启stub_status模块（修改Nginx配置）
cat > /etc/nginx/nginx.conf << EOF
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    log_format  main  '\$remote_addr - \$remote_user [\$time_local] "\$request" '
                      '\$status \$body_bytes_sent "\$http_referer" '
                      '"\$http_user_agent" "\$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # 核心：开启stub_status
    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  _;
        root         /usr/share/nginx/html;

        location /stub_status {
            stub_status on;
            access_log off;
        }

        include /etc/nginx/default.d/*.conf;

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
}
EOF

# 启动并开机自启Nginx
systemctl start nginx
systemctl enable nginx

# 验证stub_status是否生效
curl http://localhost/stub_status
# 输出包含 Active connections 即正常
```

```
systemctl start nginx
systemctl enable nginx
curl http://localhost/stub_status
```

```
cd /usr/local/prometheus/exporters
# 下载v1.1.0版本（稳定版，替代latest）
wget https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v1.1.0/nginx-prometheus-exporter_1.1.0_linux_amd64.tar.gz
# 解压
tar -zxvf nginx-prometheus-exporter_1.1.0_linux_amd64.tar.gz
mv nginx-prometheus-exporter_1.1.0_linux_amd64 nginx-exporter
```

```
# 创建系统服务
cat > /usr/lib/systemd/system/nginx-exporter.service << EOF
[Unit]
Description=Nginx Prometheus Exporter
After=network.target nginx.service

[Service]
User=root
# 指定采集Nginx的stub_status地址
ExecStart=/usr/local/prometheus/exporters/nginx-exporter/nginx-prometheus-exporter \
  --nginx.scrape-uri=http://localhost:80/stub_status
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

```
# 开启开机自启
systemctl daemon-reload
systemctl start nginx-exporter
systemctl enable nginx-exporter
systemctl status nginx-exporter
curl http://localhost:9113/metrics
```

配置 prometheus + node-exports

```
# 下载并解压
mkdir -p /usr/local/prometheus/exporters
cd /usr/local/prometheus/exporters、
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
tar -zxvf node_exporter-1.8.1.linux-amd64.tar.gz
mv node_exporter-1.8.1.linux-amd64 node_exporter
```

```
# 创建系统服务（开机自启）
# 编写service文件
cat > /usr/lib/systemd/system/node-exporter.service << EOF
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=root
ExecStart=/usr/local/prometheus/exporters/node_exporter/node_exporter \
  --path.procfs=/proc \
  --path.sysfs=/sys \
  --collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($|/)
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

```

```
# 启动设置开机自启
systemctl daemon-reload# 启动服务
systemctl start node-exporter
systemctl enable node-exporter
systemctl status node-exporter
ss -tnlp | grep 9100
```

配置 Proemthues

```
# 添加 Prometheus 的官方镜像源
cat > /etc/yum.repos.d/prometheus.repo << EOF
[prometheus]
name=Prometheus Repository
baseurl=https://packagecloud.io/prometheus-rpm/release/el/7/\$basearch
enabled=1
gpgcheck=0  # 关闭 GPG 校验
repo_gpgcheck=0
EOF

# 下载最新稳定版二进制包
wget https://github.com/prometheus/prometheus/releases/download/v2.53.0/prometheus-2.53.0.linux-amd64.tar.gz
# 解压并启动
tar -zxvf prometheus-2.53.0.linux-amd64.tar.gz
cd prometheus-2.53.0.linux-amd64
./prometheus --config.file=prometheus.yml &  # 后台运行

yum install prometheus 
```

配置主要配置

```
# 此 prometheus.yml 并非规则而是主配置 systemctl 启动问题删除最后一行
# 启动命令并指定配置 ./prometheus --config.file=/usr/local/prometheus/prometheus.yml &
# 上面的命令可以启动配置但会加载新端口，需强制终止源进程 ss -tnlp | grep 9090
cat prometheus.yml 
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets: ["localhost:9093"]
                 # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
    - "rules/node_rules.yml"
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]
  # 监控服务器节点
  - job_name: "node_exporter"
    static_configs:
      - targets: ["localhost:9100"]  # 若监控其他服务器，替换为对应IP:9100

  # 监控nginx
  - job_name: "nginx"
    static_configs:
      - targets: ["localhost:9113"]
    scrape_interval: 5s  #高频抓取
```

配置规则

```
# 配置 node_rule.yml
groups:
- name: node_alerts
  rules:
  - alert: 服务器内存使用率过高
    expr: 100 - (free_memory / total_memory) * 100 > 85  # 内存使用率>85%触发
    for: 5m  # 持续5分钟才触发
    labels:
      severity: warning
    annotations:
      summary: "服务器 {{ $labels.instance }} 内存使用率过高"
      description: "内存使用率: {{ $value }}% (阈值: 85%)，持续5分钟以上"
```

配置反向代理

```
# Nginx 配置
server {
    listen 80;
    # 建议同时写IP+localhost，确保192.168.100.128/localhost访问均生效
    server_name localhost 192.168.100.128;

    # 限制访问（可选，提高安全性）nginx prometheus-exporter
    location /nginx_status {
        stub_status on;       # 开启状态页
        access_log off;       # 关闭状态页日志
        # 限制访问（仅允许 exporter 所在机器访问，提高安全性）
        allow 127.0.0.1;     # 本机（如果 exporter 和 Nginx 同机器）
        allow 192.168.100.0/24; # exporter 所在网段（替换为你的实际网段）
        deny all;
    }
    #限制访问
    autoindex off;
    # 全局根目录，所有子location统一继承，无需重复写
    root /var/www/html/blog;
    index index.html;
  

    # ========== 1. 最高优先级：放行静态资源目录（样式/图片等），缓存7天 ==========
    location ^~ /assets/ {
        expires 7d;
        add_header Cache-Control "public, max-age=604800";
    }

    # ========== 2. 静态资源后缀匹配（缓存+放行） ==========
    location ~* \.(css|js|map|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf)$ {
        expires 7d;
        add_header Cache-Control "public, max-age=604800";
    }

    # ========== 3. 核心修复：所有路由请求兜底转发/重定向（解决history路由404） ==========
    location / {
        # 优先找本地文件 → 找不到文件 → 兜底到index.html（解决前端history路由404）
        try_files $uri $uri/ @fallback;
    }

    # ========== 4. 反向代理兜底规则（统一转发Apache 8080） ==========
    location @fallback {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 60;
        proxy_send_timeout 60;
        proxy_read_timeout 60;
    }
}

```

### Container
```#
docker pull prom/prometheus:v2.53.0
docker pull grafana/grafana:9.5.17
docker pull nginx:1.25.3
docker pull nginx/nginx-prometheus-exporter:1.1.0
docker pull prom/node-exporter:v1.8.1
```

```
# 编排 docker-copmose
# 使用容器注意 Grafana 的网络模式（桥接和主机网络问题）
version: '3.8'

# 定义网络（让所有组件互通）
networks:
  monitor-network:
    driver: bridge

services:
  # 1. Nginx（带监控模块配置）
  nginx:
    image: nginx:1.25.3
    container_name: nginx
    ports:
      - "80:80"  # 对外暴露Nginx服务端口
    volumes:
      # 挂载自定义Nginx配置（开启status模块，供exporter采集）
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      # 挂载Nginx日志目录（可选，用于排查问题）
      - ./nginx/logs:/var/log/nginx
    networks:
      - monitor-network
    restart: always

  # 2. Nginx Prometheus Exporter（采集Nginx监控数据）
  nginx-exporter:
    image: nginx/nginx-prometheus-exporter:1.1.0
    container_name: nginx-exporter
    ports:
      - "9113:9113"  # exporter暴露的监控端口
    command:
      - "--nginx.scrape-uri=http://nginx:80/stub_status"  # 采集Nginx的status接口
    depends_on:
      - nginx  # 依赖Nginx，确保先启动Nginx
    networks:
      - monitor-network
    restart: always

  # 3. Node Exporter（采集服务器系统数据）
  node-exporter:
    image: prom/node-exporter:v1.8.1
    container_name: node-exporter
    ports:
      - "9100:9100"  # node-exporter暴露的监控端口
    volumes:
      # 挂载主机系统目录，用于采集CPU、内存、磁盘等数据
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - "--path.procfs=/host/proc"
      - "--path.sysfs=/host/sys"
      - "--collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($$|/)"
    networks:
      - monitor-network
    restart: always

  # 4. Prometheus（存储监控数据，配置采集规则）
  prometheus:
    image: prom/prometheus:v2.53.0
    container_name: prometheus
    ports:
      - "9090:9090"  # Prometheus访问端口
    volumes:
      # 挂载Prometheus配置文件（定义采集目标）
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      # 挂载Prometheus数据目录（持久化存储监控数据）
      - ./prometheus/data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--web.console.libraries=/usr/share/prometheus/console_libraries"
      - "--web.console.templates=/usr/share/prometheus/consoles"
    depends_on:
      - nginx-exporter
      - node-exporter
    networks:
      - monitor-network
    restart: always

  # 5. Grafana（可视化监控数据）
  grafana:
    image: grafana/grafana:9.5.17
    container_name: grafana
    ports:
      - "3000:3000"  # Grafana访问端口
    volumes:
      # 挂载Grafana数据目录（持久化配置、仪表盘等）
      - ./grafana/data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin  # Grafana默认管理员账号
      - GF_SECURITY_ADMIN_PASSWORD=admin  # 默认密码，建议首次登录后修改
    depends_on:
      - prometheus
    networks:
      - monitor-network
    restart: always

```

编写关键配置文件

```
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    keepalive_timeout  65;

    # 核心：开启Nginx状态监控接口
    server {
        listen       80;
        server_name  localhost;

        # 监控接口路径
        location /stub_status {
            stub_status on;
            access_log off;
            allow all;  # 生产环境建议限制仅监控服务器访问
        }

        # 测试页面（可选）
        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }
    }
}
```

配置 proemthues 主要文件

```
global:
  scrape_interval: 15s  # 采集间隔，默认15秒
  evaluation_interval: 15s

rule_files:
  # - "alert_rules.yml"  # 告警规则文件（可选）

scrape_configs:
  # 采集Prometheus自身数据（可选）
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  # 采集Nginx Exporter数据
  - job_name: "nginx"
    static_configs:
      - targets: ["nginx-exporter:9113"]

  # 采集Node Exporter数据（服务器系统监控）
  - job_name: "node"
    static_configs:
      - targets: ["node-exporter:9100"]
```

```
# 启动所有容器（后台运行）
docker-compose up -d
# 查看容器状态（确认所有容器都是Up状态）
docker-compose ps
# 查看日志（若有容器启动失败，可查看日志排查）
docker-compose logs -f
```

## ELK
* Kibana：Stack Management  -> Data views -> Creat data view -> Discover -> Stream

```
使用 docker 部署 ELK ，并通过 Filebeat 采集日志 Nginx 和系统日志，用 Kibana 排障
Filebeat：日志搬运工
Elasticsearch：日志数据库 + 搜索引擎
Kibana：日志可视化 + 排障工工具

使用 Docker-compose.yml 部署ELK
```

```
# mkdir elk && cd elk && vim Docker-compose.yml
# 均采用版本号为8.10.4的容器，有些容器由于依赖不完全，导致运行时无法启动需要的容器内的环境命令
# 需要在意容器的内存，不可过大导致虚拟主机卡顿
# xpack.security.http.ssl.enabled=false为防止http信任度不够，直接绕过
# 不使用docker.1ms.run 是因为该毫秒镜像官网下载 elk 镜像需要会员
# /var/log:/var/log:ro 前面为挂载，后面为可读
# 一定要将 filedate.yml 的挂载在指定位置
# "/usr/share/filebeat/filebeat" Filebeat 的命令行参数 指定前台模式

version: "3.8"

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.10.4
    container_name: elasticsearch
    environment: 
      - discovery.type=single-node
      - xpack.security.enabled=false
      - xpack.security.http.ssl.enabled=false
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ulimits: 
      memlock: 
        soft: -1
        hard: -1
    volumes: 
      - esdata:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  kibana:
    image: docker.elastic.co/kibana/kibana:8.10.4
    container_name: kibana
    depends_on:
      - elasticsearch
    ports: 
      - "5601:5601"
    environment: 
      - xpack.security.enabled=false
      - elasticsearch.version=8.10.4

  filebeat: 
    image: docker.elastic.co/beats/filebeat:8.10.4
    container_name: filebeat
    user: root
    volumes: 
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/log:/var/log:ro
      - ./filebeat/data:/usr/share/filebeat/data
    depends_on:
      - elasticsearch
    command: ["/usr/share/filebeat/filebeat","-e","-c","/usr/share/filebeat/filebeat.yml"]

volumes:
  esdata:

```

```
# 验证安装
docker compose up -d
docker compose ps 
docker compose down -v && docker compose up -d
curl http://localhost:9200
curl http://localhost:5601
hhtp://localhost:5601 
```

```
# 配置 filebeat 信息 filebeat.yml
# filestream 的前身来自于 log ，在之前的版本中有使用过
# 其中index 为采集日志的文件头，下面的代码块将演示

filebeat.inputs:
  - type: filestream
    enabled: true
    paths:
      - /var/log/syslog
      - /var/log/messages
  - type: filestream
    enabled: true
    paths:
      - /var/log/nginx/access.log
      - /var/log/nginx/error.log
    fields:
      service: nginx
    fields_under_root: true
output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "filebeat-8.10.4-%{+yyyy.MM.dd}"
setup.kibana:
  host: ["kibana:5601"]


setup.ilm.enabled: false
setup.template.enabled: false
setup.template.overwrite: true
setup.template.name: "filebeat-8.10.4"
setup.template.pattern: "filebeat-8.10.4-*"
logging.level: info
logging.to_stdout: true

```

```
# 使用docker logs --tail 5 filebeat 后的日志 
# 注意没有初始日志的时候，强制写入测试（宿主机）
# echo "TEST LOG 1: $(date) - Filebeat to ES test" >> /var/log/syslog
# echo "TEST LOG 2: $(date) - Nginx access test" >> /var/log/nginx/access.log
# echo "TEST LOG 3: $(date) - Nginx error test" >> /var/log/nginx/error.log
# sleep 15
# curl http://localhost:9200/_cat/indices/filebeat*

{"log.level":"info","@timestamp":"2026-01-06T16:23:03.135Z","log.logger":"monitoring","log.origin":{"file.name":"log/log.go","file.line":187},"message":"Non-zero metrics in the last 30s","service.name":"filebeat","monitoring":{"metrics":{"beat":{"cgroup":{"cpuacct":{"total":{"ns":18259331}},"memory":{"mem":{"usage":{"bytes":63574016}}}},"cpu":{"system":{"ticks":7020,"time":{"ms":10}},"total":{"ticks":10580,"time":{"ms":10},"value":10580},"user":{"ticks":3560}},"handles":{"limit":{"hard":1048576,"soft":1048576},"open":12},"info":{"ephemeral_id":"0aae5277-e858-46b0-bb2b-f305bd6fb3e4","uptime":{"ms":14670037},"version":"8.10.4"},"memstats":{"gc_next":41719672,"memory_alloc":24475000,"memory_total":943653072,"rss":80916480},"runtime":{"goroutines":36}},"filebeat":{"events":{"active":0,"added":6,"done":6},"harvester":{"open_files":2,"running":2}},"libbeat":{"config":{"module":{"running":0}},"output":{"events":{"acked":6,"active":0,"batches":6,"total":6},"read":{"bytes":2066},"write":{"bytes":5922}},"pipeline":{"clients":2,"events":{"active":0,"published":6,"total":6},"queue":{"acked":6}}},"registrar":{"states":{"current":0}},"system":{"load":{"1":0.02,"15":0.21,"5":0.13,"norm":{"1":0.005,"15":0.0525,"5":0.0325}}}},"ecs.version":"1.6.0"}}
```


