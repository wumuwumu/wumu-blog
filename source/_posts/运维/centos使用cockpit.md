---
title: centos使用cockpit
date: 2020-9-05 21:40:23
tags:
- linux
---

```bash
sudo systemctl enable --now cockpit.socket

[leiakun@centos8 ~]$ sudo systemctl enable --now cockpit.socket
[sudo] leiakun 的密码：
Created symlink /etc/systemd/system/sockets.target.wants/cockpit.socket → /usr/lib/systemd/system/cockpit.socket.
[leiakun@centos8 ~]$ 

[leiakun@centos8 ~]$ sudo firewall-cmd --get-services |grep cockpit
RH-Satellite-6 amanda-client amanda-k5-client amqp amqps apcupsd audit
bacula bacula-client bb bgp bitcoin bitcoin-rpc bitcoin-testnet bitcoin-testnet-rpc
bittorrent-lsd ceph ceph-mon cfengine cockpit condor-collector ctdb dhcp dhcpv6 
dhcpv6-client distcc dns dns-over-tls docker-registry docker-swarm dropbox-lansync 
elasticsearch etcd-client etcd-server finger freeipa-4 freeipa-ldap freeipa-ldaps 
freeipa-replication freeipa-trust ftp ganglia-client ganglia-master git grafana 
gre high-availability http https imap imaps ipp ipp-client ipsec irc ircs 
iscsi-target isns jenkins kadmin kdeconnect kerberos kibana klogin kpasswd 
kprop kshell ldap ldaps libvirt libvirt-tls lightning-network llmnr managesieve
matrix mdns memcache minidlna mongodb mosh mountd mqtt mqtt-tls ms-wbt mssql
murmur mysql nfs nfs3 nmea-0183 nrpe ntp nut openvpn ovirt-imageio ovirt-storageconsole 
ovirt-vmconsole plex pmcd pmproxy pmwebapi pmwebapis pop3 pop3s postgresql privoxy
prometheus proxy-dhcp ptp pulseaudio puppetmaster quassel radius rdp redis 
redis-sentinel rpc-bind rsh rsyncd rtsp salt-master samba samba-client samba-dc
sane sip sips slp smtp smtp-submission smtps snmp snmptrap spideroak-lansync 
spotify-sync squid ssdp ssh steam-streaming svdrp svn syncthing syncthing-gui 
synergy syslog syslog-tls telnet tentacle tftp tftp-client tile38 tinc tor-socks 
transmission-client upnp-client vdsm vnc-server wbem-http wbem-https wsman wsmans
xdmcp xmpp-bosh xmpp-client xmpp-local xmpp-server zabbix-agent zabbix-server

sudo firewall-cmd --add-service=cockpit --permanent
sudo firewall-cmd --reload
```

# 多主机管理

```bash
yum install -y cockpit-dashboard
```

