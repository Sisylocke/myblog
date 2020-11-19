---
title: "使用docker部署rsshub"
tags: ["折腾"]
date: 2020-11-19T02:16:00+08:00
draft: false
categories: "萨达姆做好了战斗准备"
---



## 首先是docker compose

具体步骤可以参考rsshub[文档](https://docs.rsshub.app/install/)，总之使用docker compose一把梭就OK啦。

- 先下载docker-compose.yml
```
weget https://raw.githubusercontent.com/DIYgod/RSSHub/master/docker-compose.yml
```

这里可以先根据个人喜好配置一下`docker-compose.yml`的[环境变量](https://docs.rsshub.app/install/#pei-zhi)：
```
services:
    rsshub:
    ......
        environment:
            # 因为部署在共网上，还是不要debug信息了。注意false的引号...
            DEBUG_INFO: "false"
            ACCESS_KEY: ILoveRSSHub
    ......
```

- 然后是持久化redis

```
docker volume redis-data
```
- 接着就可以run起来
```
docker-compose up -d
```

## 然后是用Nginx反向代理
首先先找到rsshub在docker容器的ip地址，这里用的是`docker inspect`，`rsshub_rsshub_1`是rsshub在docker的名称，用容器ID也行，它们需要根据实际情况更改，可以用`docker ps`找到。
```
docker inspect rsshub_rsshub_1 | grep IPAddress
>>>
    "SecondaryIPAddresses": null,
    "IPAddress": "",
             "IPAddress": "172.19.0.4",
```
额，这结果看起来不是那么优雅，所以还可以这样：
```
docker inspect --format='{{.Name}} - {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(docker ps -aq)
>>>
    /rsshub_rsshub_1 - 172.19.0.4
    /rsshub_browserless_1 - 172.19.0.3
    /rsshub_redis_1 - 172.19.0.2
```

这看起来就舒服多了，只不过就是这命令有点长不太好记。不过总之是得到了ip地址，然后就可以开始修改nginx的配置文件了，这里是将rsshub的页面设置为网站的一个二级目录，用`https://xxx.xxx/rsshub`就可以访问，所以修改起来也很简单，在nginx原先网站的配置文件后面加上一个`location`即可：
```
server {
    ......
    location /rsshub/ {
                # 注意端口后面要加斜线
                proxy_pass http://172.19.0.4:1200/;
                proxy_redirect off;
                proxy_set_header        Host    $host;
                proxy_set_header        X-Real-IP       $remote_addr;
                proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
                    }
   ......
}
```
这就搞定啦～
## 最后是docker镜像的自动更新
......