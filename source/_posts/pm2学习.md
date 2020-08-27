---
title: pm2学习
date: 2019-11-02 10:10:05
tags:
- node
---

# pm2基本命令

```bash
# 启动程序
pm2 start app.js
pm2 start npm --name pro -- run dev

# 查看程序
pm2 start list
pm2 monit
pm2 logs

# 重启
pm2 restart all
pm2 reload all
pm2 restartt 0

# 停止
pm2 stop all
pm2 stop 0

# 杀死
pm2 delete all
pm2 delete 0

# 集群
pm2 start app.js -i max # 根据cpu数目启动线程
pm2 start app.js -i 3 # 启动3个进程
pm2 start app.js -x  # 使用fork模式启动
pm2 start app.json
```

# 日志问题

日志系统对于任意应用而言，通常都是必不可少的一个辅助功能。pm2的相关文件默认存放于$HOME/.pm2/目录下，其日志主要有两类：

a. pm2自身的日志，存放于$HOME/.pm2/pm2.log；

b. pm2所管理的应用的日志，存放于$HOME/.pm2/logs/目录下，标准谁出日志存放于${APP_NAME}_out.log，标准错误日志存放于${APP_NAME}_error.log；

这里之所以把日志单独说明一下是因为，如果程序开发不严谨，为了调试程序，导致应用产生大量标准输出，使服务器本身记录大量的日志，导致服务磁盘满载问题。一般而言，pm2管理的应用本身都有自己日志系统，所以对于这种不必要的输出内容需禁用日志，重定向到/dev/null。

与crontab比较，也有类似情况，crontab自身日志，与其管理的应用本身的输出。应用脚本输出一定需要重定向到/dev/null，因为该输出内容会以邮件的形式发送给用户，内容存储在邮件文件，会产生意向不到的结果，或会导致脚本压根不被执行；

# 开机启动

```bash
pm2 startup
systemctl enable pm2-root
```



# 参考

<https://pm2.keymetrics.io/docs/usage/monitoring/>