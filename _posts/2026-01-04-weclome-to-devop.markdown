---
layout:	post
title:	"Welcome to devop!!"
date:	2026-01-05 20:25:80 +0800
categories:	jekyll update
---
基于ruby语言搭建的jekyll个人博客系统，采用本地代码编写，通过github存储数据，利用jenkins的自动化CI/CD代码拉取推送，将文章远程推送到到云服务器中上线。

|   终端   | 中间件  | 环境               |
| :-------: | ------- | ------------------ |
| 个人用户 | vscode  | jekyll博客本地     |
| web服务器 | github  | git代码拉取和提交  |
|  服务器1  | jenkins | podman镜像容器管理 |
|  服务器2  | nginx   | nginx+apache       |

## 本地搭建

环境：ruby3.3.5[ruby官方网站](https://www.ruby-lang.org/zh_cn/news/2024/09/03/3-3-5-released/) + bundle3.0.4[bundle托管中心](https://rubygems.org/gems/bundler/versions?locale=zh-CN)

```
# 通过源码安装==>下载二进制包==>解压包==>
$ ./configure
$ make
$ sudo make install
```

```
$ sudo yum install ruby # apt
$ sudo apt-get install ruby-full # apt
$ sudo emerge dev-lang/ruby # portage
$ sudo pacman -S ruby # pacman
```

```
ruby -v
gem -v
gcc -v
g++ -v
make -v
gem install bundler jeklyll
jekyll new myblog
cd myblog
bundle build #更新Gemfile文件
bundle exec jekyll serve

```


```
# 编辑Gemfile文件得到错误信息少的编译后的本地网站
source "https://gems.ruby-china.com"

# Hello! This is where you manage which Jekyll version is used to run.
# When you want to use a different version, change it below, save the
# file and run `bundle install`. Run Jekyll with `bundle exec`, like so:
#
#     bundle exec jekyll serve
#
# This will help ensure the proper Jekyll version is running.
# Happy Jekylling!
gem "jekyll", "~> 4.4.1"
# This is the default theme for new Jekyll sites. You may change this to anything you like.
gem "minima", "~> 2.5"
# If you want to use GitHub Pages, remove the "gem "jekyll"" above and
# uncomment the line below. To upgrade, run `bundle update github-pages`.
# gem "github-pages", group: :jekyll_plugins
# If you have any plugins, put them here!
group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.12"
end

# Windows and JRuby does not include zoneinfo files, so bundle the tzinfo-data gem
# and associated library.
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

# Performance-booster for watching directories on Windows
gem "wdm", "~> 0.1", :platforms => [:mingw, :x64_mingw, :mswin]

# Lock `http_parser.rb` gem to `v0.6.x` on JRuby builds since newer versions of the gem
# do not have a Java counterpart.
gem "http_parser.rb", "~> 0.6.0", :platforms => [:jruby]

gem 'logger'
gem 'sass-embedded','~> 1.69.5'
gem "jekyll-sass-converter", "~> 2.0"

```

以上本地jekyll博客已完成搭建

## github

首先注册个人账户，再成功创建一个新项目，设置为项目私有，关闭自动add  .gitignore，这样就有一个自己的博客项目，命名blog

```

#用于将本地的myblog推送到github上
echo "### myblog" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:xxxxxxx/myblog.git
git push -u origin main
#第二次推送
git add .
git commmit -m "这是我这一次推送更改的内容"
git push
#下拉仓库的代码
git clone <URL>
```

想要 github 关联 jekyll 建立web 服务，使用 github 的 page 工具，在Setting 中 building and deployment 选择 main / (root)

## jenkins and registry

```
#使用podman
yum install podman # yum 
```

[podman的安装方法](https://podman.io/docs/installation)
[podman的官方文档](https://docs.podman.org.cn/en/latest/markdown/podman-push.1.html)
[brew的安装方法](https://docs.brew.sh/Installation)

### 镜像容器
```
# 构建运行镜像
podman run docker.1ms.run/jenkins/jenkins:lts
podman run docker.1ms.run/registry:2
# 构建容器
podman run -d --name jenkins -p 8080:8080 -p 50000:50000 -v /data/jenkins:/var/jenkins_home -v /usr/bin/ruby:/usr/bin/ruby -u 10000:1000 jenkins docker.1ms.run/jenkins/jenkins:lts     #持久化（宿主机挂载数据和环境）
podman run -d --name registry -p 5000 -v /data/registry:/var/registry docker.1ms.run/registry:2 #持久化
#启动容器
podman start jenkins
podman start registry
podman restart jenkins
podman restart registry
podman stop jenkins
podman stop registry
#podman基础命令
podman ps -a
podman rm <id>
podman rmi <id>
podman exec -it <id> /bin/bash #启动时才能进入
podman inspect registry | grep -E "Source|Destination" -A1 -B1 #查看容器数据位置


#镜像私有
podman tag jenkins-ruby-bundle:v1 localhost:5000/jenkins-jekyell:v1 #镜像打标签
podman push localhost:5000/jenkins-jekyell:v1 Getting image source signatures # Podman/Docker 强制用 HTTPS 访问仓库
podman push --tls-verify=false localhost:5000 jenkins-jekyell:v1 Getting image source signatures # HTTP 访问跳过校验
curl http://localhost:5000/v2/_catalog # 查看镜像仓库中的镜像 #V1接口已经被废弃（podman/docker）
```
```
#关于ruby+bundle环境的jenkins容器构建Dockerfile
FROM docker.1ms.run/jenkins/jenkins:lts
USER root

# 直接安装系统源默认的ruby-full（无需指定版本）
RUN apt-get update -y && \
    apt-get install -y ruby-full build-essential libssl-dev && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# 安装Bundler4.0.3
RUN gem install bundler:4.0.3 --no-document

USER jenkins
RUN ruby -v && bundle -v
```
```
podman build -t jenkins-ruby-bundle:v1 .
```
现在已经启动Jenkins,Registry 进入本地Localhost:8080
先安装一系列插件，构建选择自由式风格，✔访问仓库拉取代码 ✔代码远程转发
```
#jinkins服务器公钥私钥配置
ssh-keygen -t rsa
~/.ssh/id_rsa # 私钥
~/.ssh/id_rsa.pub # 公钥
ssh -T git@github.com # 1.验证GitHub SSH服务，执行后自动创建known_hosts
mkdir -p ~/.ssh && touch ~/.ssh/known_hosts # 2.手动创建known_hosts 适用于无人交互
ssh-keyscan github.com >> ~/.ssh/known_hosts #2..
ssh -T git@github.com # 2...
#===权限管理===
sudo chown -R jenkins:jenkins ~/.ssh
chmod 700 ~/.ssh
chmod 644 ~/.ssh/known_hosts
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
```
```
ls -l
# ==============编译脚本==========
cd ${WORKSPACE}/
# 安装依赖
bundle config set path 'vendor/bundle' && bundle config set disable_shared_gems 'true' && bundle install --quiet
# 编译jekyll，生成纯静态文件->输出到_site目录下
bundle exec jekyll build --verbose
# 验证编译结果
echo "＜（＾－＾）＞ jekyll 编译已经完成，静态文件列表如下："
ls -lh ${WORKSPACE}/_site/
```
此时需要考虑将Jenkins服务器上的文件传输到Nignx服务器上/var/www/html/blog
在容器内做ssh远程传输
```
# jenkins容器内部
keygen -t rsa
ls -l ~/.ssh/
ssh-copy-id root@192.168.1.100
yum install openssh # scp是openssh的原生内置工具
```
```
# ============推送纯净静态文件给服务器=======
# 仅推送_site目录下的所有文件，无任何冗余
scp -r ${WORKSPACE}/_site/*  root@192.168.100.128:/var/www/html/blog
# 可选优化，给服务器赋权，便面访问权限问题
ssh root@192.168.100.128 "chmod -R 755 /var/www/html/blog && chown -R nginx:nginx /var/www/html/blog"
# 可选优化，删除服务器上已经失效的文件
# ssh root@192.168.100.128 "find /var/www/html -type f ! -name '*.html' ! -name '*.css' ! -name '*.js' ! -name '*.jpg' ! -name '*.png' -delete"

echo "＜（＾－＾）＞ 部署成功！服务器仅存在纯净静态文件，无任何冗余"
```
保存应用，构建Jenkins，更改Jenkins时区，admin-->account-->timezone

## Nginx服务器
```
# 反向代理，Jekyll是静态博客，可以不使用Apache
server {
    listen 80;
    # 建议同时写IP+localhost，确保192.168.100.128/localhost访问均生效
    server_name localhost 192.168.100.128;
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