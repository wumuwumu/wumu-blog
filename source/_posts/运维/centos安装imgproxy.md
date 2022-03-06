---
title: centos安装imgproxy
abbrlink: 7a004090
date: 2021-04-06 12:00:00
---
# 通过docker安装
```bash
docker pull darthsim/imgproxy:latest

docker run -e IMGPROXY_USE_S3=true -e IMGPROXY_S3_ENDPOINT=http://192.168.100.11:9228 -e AWS_ACCESS_KEY_ID=AKIAIOSFSDODNN7EXAMPLE -e AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEDXXAMPLEKEY -p 9340:8080 -d  darthsim/imgproxy
```

