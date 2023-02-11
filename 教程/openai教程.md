---
title: openai部署教程
description: 
published: true
date: 2023-01-30T07:22:45.053Z
tags: openai, nginx
editor: markdown
dateCreated: 2023-01-30T07:22:45.053Z
---

# openai下载（失败）
路径：`/home/azureuser/openai-quickstart-python`
nginx配置5000端口配置路径：`/etc/nginx/sites-enabled`
关闭nginx：`nginx -s stop`

```python
    location /openai {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                #try_files $uri $uri/ =404;
                proxy_pass http://172.17.0.1:5000;

        }
        
        
           location /openai {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                #try_files $uri $uri/ =404;
                include uwsgi_params;
                uwsgi_pass 20.200.202.65:5000;
        }
```
开启nginx：`nginx`
参考：
https://beta.openai.com/docs/quickstart/build-your-application