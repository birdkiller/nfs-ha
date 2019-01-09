# nfs-ha
NFS高可用方案 nfs+keepalived+drbd

本文简要记录对nfs高可用可行性验证的过程
原理不说了，drbd做主备同步，nfs分别作为rip，keepalived对外提供vip。

## 环境
- master 虚拟机 ubuntu18 enp0s3 172.20.166.157
- backup 虚拟机 ubuntu18 enp0s3 172.20.166.103
- vip 172.20.166.245/32

## 安装drbd
1. ```apt-get install drbd8-utils```
2. ```vi /etc/drbd.d/global_common.conf```
3. >global {
        usage-count yes;
}
common {
        protocol C;
}
4. ```vi /etc/drbd.d/r0.res ``` #r0 是要配置的drbd资源名称(非必要)
5. >resource r0 {
  on master { #master为主机名，配置文件里必须和uname–n输出的一致，不一致会导致no resources defined!
    device   /dev/drbd1; #这是drbd虚拟出来的存储
    disk     /dev/sdb; #这是你准备用来做drbd的硬件存储
    address  172.20.166.157:7789; #master
    meta-disk internal;
  }
  on node2 {
    device   /dev/drbd1;
    disk     /dev/sdb;
    address  172.20.166.103:7789;
    meta-disk internal;
  }
}
6. 以上两个配置文件在master/backup上都必须配置
7. 初始化drbd（每台机器仅首次执行）```drbdadm create-md r0 ```
8. 每台机器启动drbd ```service drbd start``` 此时查看```drbd-overview```可读取集群状态，此时两台机器皆为secondary。
9. 在master上执行```drbdsetup /dev/drbd1 primary --force```不带--force可能会报错
10. 格式化&挂载：在master上执行```mkfs.xfs /dev/drbd1``` ```mount /dev/drbd1 /data/drbd``` 一会nfs就会共享这块/data/drbd存储。
11. 查看状态，此时： ```1:r0/0  Connected Primary/Secondary UpToDate/UpToDate /data/drbd xfs 1021M 34M 988M 4%```

## 安装nfs
1. 以下操作都在两台机上执行```apt-get install nfs-kernel-server```
2. 配置```vi /etc/exports``` ```/data/drbd *(fsid=1047,rw,sync,no_root_squash)```
3. 重启服务```service nef-kernel-server restart```

## 安装keepalived
1. 两台机```apt-get install keepalived```
2. 配置master```vi /etc/keepalived/keepalived.conf```
3. >! Configuration File for keepalived
    global_defs {
        router_id NFS_TEST
    }
    vrrp_script chk_nfs {
        script "/etc/keepalived/check_nfs.sh"
        interval 5
    }
    vrrp_instance VI_1 {
        state MASTER
        interface enp0s3
        virtual_router_id 51
        priority 100
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass 1111
        }
        track_script {
            chk_nfs
        }
        notify_backup /etc/keepalived/notify_stop.sh          #当本机成为backup时执行的脚本
        notify_master /etc/keepalived/notify_master.sh      #当本机成为master时执行的脚本
        virtual_ipaddress {
            172.20.166.245
        }}
4. 配置backup```vi /etc/keepalived/keepalived.conf```
5. >! Configuration File for keepalived
    global_defs {
        router_id NFS_TEST
    }
    vrrp_instance VI_1 {
        state BACKUP
        interface enp0s3
        virtual_router_id 51
        priority 90
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass 1111
        }
        notify_master /etc/keepalived/notify_master.sh         
        notify_backup /etc/keepalived/notify_backup.sh         
        virtual_ipaddress {
            172.20.166.245
    }}
6. 注意，keepalived脚本默认用keepalived_script用户执行，需要新建此用户并授权
7. 以下是脚本内容
  - master /etc/keepalived/check_nfs.sh
  - ```#!/bin/sh
/usr/sbin/service nfs-kernel-server status &>/dev/null
if [ $? -ne 0 ];then
    sudo /usr/sbin/service nfs-kernel-server restart
    sudo /usr/sbin/service nfs-kernel-server status &>/dev/null
    if [ $? -ne 0 ];then
        sudo umount /dev/drbd1
        sudo drbdadm secondary r0
        sudo /sbin/usr/service keepalived stop
    fi
fi```
  - master /etc/keepalived/notify_master.sh 
  - ```#!/bin/bash
time=`date "+%F  %H:%M:%S"`
sudo echo -e "$time    ------notify_master------\n" >> /etc/keepalived/logs/notify_master.log
sudo /usr/sbin/service drbd start &>> /etc/keepalived/logs/notify_master.log
sudo /sbin/drbdadm primary r0 &>> /etc/keepalived/logs/notify_master.log
sudo /bin/mount /dev/drbd1 /data/drbd &>> /etc/keepalived/logs/notify_master.log
sudo /usr/sbin/service nfs-kernel-server restart &>> /etc/keepalived/logs/notify_master.log
sudo echo -e "\n" >> /etc/keepalived/logs/notify_master.log```
  - master /etc/keepalived/notify_stop.sh 
  - ```#!/bin/bash
time=`date "+%F  %H:%M:%S"`
sudo echo -e "$time  ------notify_stop------\n" >> /etc/keepalived/logs/notify_stop.log
sudo /usr/sbin/service nfs-kernel-server stop &>> /etc/keepalived/logs/notify_stop.log
sudo /bin/umount /data/drbd &>> /etc/keepalived/logs/notify_stop.log
sudo /sbin/drbdadm secondary r0 &>> /etc/keepalived/logs/notify_stop.log
sudo echo -e "\n" >> /etc/keepalived/logs/notify_stop.log```
  - backup /etc/keepalived/notify_master.sh
  - ```#!/bin/bash
time=`date "+%F  %H:%M:%S"`
echo -e "$time    ------notify_master------\n" >> /etc/keepalived/logs/notify_master.log
sudo /usr/sbin/service drbd start &>> /etc/keepalived/logs/notify_master.log
sudo /sbin/drbdadm primary r0 &>> /etc/keepalived/logs/notify_master.log
sudo /bin/mount /dev/drbd1 /data/drbd &>> /etc/keepalived/logs/notify_master.log
sudo /usr/sbin/service nfs-kernel-server restart &>> /etc/keepalived/logs/notify_master.log
echo -e "\n" >> /etc/keepalived/logs/notify_master.log```
  - backup /etc/keepalived/notify_backup.sh
  - ```#!/bin/bash
time=`date "+%F  %H:%M:%S"`
echo -e "$time    ------notify_backup------\n" >> /etc/keepalived/logs/notify_backup.log
sudo /usr/sbin/service nfs-kernel-server stop &>> /etc/keepalived/logs/notify_backup.log
sudo /bin/umount /dev/drbd1 &>> /etc/keepalived/logs/notify_backup.log
sudo /sbin/drbdadm secondary r0 &>> /etc/keepalived/logs/notify_backup.log
echo -e "\n" >> /etc/keepalived/logs/notify_backup.log```
8. 双机启动 ```service keepalived start```

## 测试
1. 安装nfs client端 ```apt-get install nfs-common```
2. 挂载 ```mount -t nfs 172.20.166.245:/data/drbd /data/test ```
3. ```echo 2222 >/data/test/a.txt```
4. master依次执行notify_stop.sh命令,关闭keepalived```service keepalived stop```
5. backup依次执行nofity_master.sh命令，漂移完成.
