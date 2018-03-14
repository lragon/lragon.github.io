---
title: "Docker Toolbox无法访问容器服务"
date: 2018-03-14 14:18:59
tags:
---

最近在看Docker，感觉很适合开发者用，部署服务器不要太方便，只是学习的成本稍微有点高，在公司的windows上装了Docker Toolbox，起了一个nginx容器

    $ docker container run \
	  -d \
	  -p 8080:80 \
	  --rm \
	  --name mynginx \
	  nginx

然后通过`localhost:8080`死活访问不了，检查命令看端口映射也没问题，网上搜了半天才发现原来是因为Toolbox用到了虚拟机，容器运行在虚拟机中，端口也是映射到虚拟机上，相当于中间隔了一层，把localhost换成虚拟机的IP就可以访问了。