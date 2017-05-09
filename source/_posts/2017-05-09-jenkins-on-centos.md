---
title: 在CentOS上安装Jenkins
layout: post
toc: false
comments: true
date: 2017-05-09 12:01:34
tags:
---

项目一直用 Jenkins 打包，都是跑在 Mac 上面。最近在 CentOS 上搭了一个，记录一下。

## 一、安装
参考 [Jenkins wiki](https://wiki.jenkins-ci.org/display/JENKINS/Installing+Jenkins+on+Red+Hat+distributions)，添加 yum 源并安装，安装合适的 Java 版本，启动服务即可，如果访问不到 8080 端口，检查防火墙是否打开这个端口，或者用 Nginx 反向代理。我采取的是用 Nginx 反向代理。

## 二、修改端口
Jenkins 默认监听端口是 8080，为避免和其他服务冲突，我把这个端口修改了。
修改 `/etc/sysconfig/jenkins` 文件中的 `JENKINS_PORT="8080"`，重启 Jenkins 服务即可。

## 三、反向代理
在 `/etc/nginx/conf.d` 中添加 Jenkins 的配置：

```
upstream jenkins{
  server localhost:6008;
}

map $http_accept_language $language {
  default  zh_tw;
  ~*^zh-cn zh_cn;
  ~*^en    en;
  ~*^zh,   zh_cn;
}

server {
  listen  80;
  server_name xxx;

  rewrite_log on;
  access_log  /data/logs/nginx/jenkins.http.log;
  error_log   /data/logs/nginx/jenkins.http.log;

  location / {
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Scheme $scheme;
    proxy_pass http://jenkins;
  }
}
```

## 四、Jenkins 中 shell 脚本调用 SVN 命令的默认 SVN 账号
我们用 Jenkins 打动态更新包，需要在 shell 脚本中调用 SVN 命令，[拉取新旧版本之间的差异](https://github.com/jreading/svn-export-changes.sh)，但是执行命令时，会用 Jenkins 和 svn 这2个默认用户去连接 SVN 服务器，连接失败，拉取不到代码。这时需要在 CentOS 上登录 Jenkins 用户再连接 SVN 服务器，用正确的账号密码通过 SVN 服务器认证并在本地存储密码。找了很多办法，可以这样[在 CentOS 上登录 Jenkins 用户](http://stackoverflow.com/questions/18068358/cant-su-to-user-jenkins-after-installing-jenkins)，然后执行一下 SVN diff 或者别的命令，目的是输入正确的账号密码通过 SVN 服务器的认证。之后就可以在 shell 中正确的执行 SVN 命令了。是否有更好的办法，我目前还不知道。

至此，可以通过 Nginx 中指定的 server_name 访问 Jenkins 界面了。