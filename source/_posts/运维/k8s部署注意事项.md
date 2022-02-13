---
title: k8s部署总结
date: 2021-09-25 12:12:12
- tags： k8s
---

# 安装

> https://kuboard.cn/install/install-k8s.html



# 卸载kubernetes-dashboard

```cpp
sudo kubectl delete deployment kubernetes-dashboard --namespace=kube-system 
sudo kubectl delete service kubernetes-dashboard  --namespace=kube-system 
sudo kubectl delete role kubernetes-dashboard-minimal --namespace=kube-system 
sudo kubectl delete rolebinding kubernetes-dashboard-minimal --namespace=kube-system
sudo kubectl delete sa kubernetes-dashboard --namespace=kube-system 
sudo kubectl delete secret kubernetes-dashboard-certs --namespace=kube-system
sudo kubectl delete secret kubernetes-dashboard-csrf --namespace=kube-system
sudo kubectl delete secret kubernetes-dashboard-key-holder --namespace=kube-system
```

# 总结

1. k8s新版本开始放弃docker,安装过程中可以不按照docker,注意网上教程的版本，本次安装的时1.21.4