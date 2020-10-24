---
title: docker安装portainer
date: 2020-10-23 15:00:00
---

```
$ docker volume create portainer_data
$ docker run -d -p 9220:8000 -p 9221:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
```

