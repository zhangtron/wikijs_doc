---
title: 部署wiki.js
description: 部署自己的知识库
published: true
date: 2023-01-17T15:31:25.607Z
tags: 计算机教程, 部署wiki
editor: markdown
dateCreated: 2023-01-06T13:13:11.809Z
---

# 白嫖一个服务器
## 申请双币支付信用卡
## 申请服务器免费试用
当前支持谷歌、微软、甲骨文和亚马逊，一个公司白嫖一年，那就是免费试用四年哈哈！

# 部署wikijs

参考：
https://docs.requarks.io/dev/themes


# 反向代理

修改`/etc/nginx/sites-enabled`路径下的default文件

```
 location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                #try_files $uri $uri/ =404;
                proxy_pass http://172.17.0.1:8080;
        }
```

- 查看docker的ip `ip addr show docker0` 
- 后面一定要加分号，否则无法验证通过

- 验证 `sudo nginx -t`  检查语法是否有错误

- 重新启动nginx `nginx -s reload` 

参考：

[2_配置nginx实现反向代理_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1mU4y1g74Y/?p=2&vd_source=4246d5faa0e7abd672adc236259eee2e)

# nginx 配置证书



证书位置`/home/azureuser/yydscpa.club_nginx` 

```
server {
        # listen 80 default_server;
        listen [::]:80 default_server;
        # listen 443 ssl default_server;
        listen 443 ssl;
        # ssl证书绑定的域名
        server_name www.yydscpa.club;
        # 证书pem文件
        ssl_certificate /home/azureuser/yydscpa.club_nginx/yydscpa.club_bundle.pem;  # pem文件的路径

        # 证书key文件
        ssl_certificate_key /home/azureuser/yydscpa.club_nginx/yydscpa.club.key; # key文件的路径
        # 启用ssl session缓存
        # 缓存握手产生的参数和加密密钥的时长
        ssl_session_timeout 5m;    #缓存有效期
        # 使用的加密套件的类型
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;    #加密算法
        # 表示使用的tls协议的类型
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;    #安全链接可选的加密协议
        }
```

参考：

[Nginx 安装 SSL 配置 HTTPS 超详细完整教程全过程-阿里云开发者社区 (aliyun.com)](https://developer.aliyun.com/article/766958)

[Nginx-SSL证书配置_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1jd4y1X7we/?spm_id_from=333.337.search-card.all.click&vd_source=4246d5faa0e7abd672adc236259eee2e)

# 配置wiki.js 中文搜索

查看运行实例：`docker ps` 

进入bash`sudo docker exec -it wikijs bash` 

通过`find -name `先找到配置文件位置

修改配置文件

```
vi /app/wiki/server/modules/search/postgres/definition.yml
enum:
      ...
      - turkish
      - chinese_zh
```
参考：
[(67条消息) 支持中文全文搜索的wiki.js_surfing！！！的博客-CSDN博客_wiki.js中文搜索](https://blog.csdn.net/placidLife/article/details/115313314)


[Wiki.js 本地部署 - 我的全新 Hugo 网站 (xja.github.io)](https://xja.github.io/wikijs-deployment/)

# http重定向为https

```
server{
        listen 80;

        # 绑定证书的域名
        server_name www.yydscpa.club;

        # 把http的域名请求转成https
        return 301 https://$host$request_uri;

}
```
参考：	
[Nginx配置HTTP跳转到HTTPS](https://juejin.cn/post/7044911075480829959)


# 配置GitHub同步

主要参考官方文档
>注意点：
>1.在虚拟机设置ssh不用进入到docker
>2.密钥默认生成在~/.ssh 文件夹下，如果改了名字或改了路径需要重新设置
>3.设置完后只有修改部分可以同步，暂没发现其他同步方式


参考：
https://docs.requarks.io/storage/git










