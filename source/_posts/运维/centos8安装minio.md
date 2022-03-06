---
title: centos8安装minio
abbrlink: f8557ed6
date: 2020-10-24 16:00:00
---

```bash
docker run -p 9228:9000 --name miniostorage -di -e "MINIO_ACCESS_KEY=AKIAIOSFODNN7EXAMPLE" -e "MINIO_SECRET_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"   -v /opt/minio/storage/data:/data -v /opt/minio/storage/config:/root/.minio --restart=always minio/minio server /data
```

