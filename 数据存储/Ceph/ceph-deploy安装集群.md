# ceph-deploy 安装集群

ceph-deploy 安装集群的方式比较流行，不过官方决定放弃这种方式了。。。

在这里为了学习，还是学一下吧。

官方教程：https://docs.ceph.com/docs/master/install/ceph-deploy/quick-ceph-deploy/

---

## 安装

首先安装 ceph-deploy：

```bash
$ sudo yum install ceph-deploy
```

更新 ceph-deploy ：

```bash
$ pip install --upgrade ceph-deploy
```

添加 `/etc/yum.repos.d/ceph.repo`：

```bash
[ceph]
name=Ceph packages
baseurl=http://mirrors.163.com/ceph/rpm-15.2.1/el7/x86_64/
enabled=1
gpgcheck=1
priority=2
type=rpm-md
gpgkey=http://mirrors.163.com/ceph/keys/release.asc

[ceph-noarch]
name=Ceph noarch packages
baseurl=http://mirrors.163.com/ceph/rpm-15.2.1/el7/noarch/
enabled=1
gpgcheck=1
priority=2
type=rpm-md
gpgkey=http://mirrors.163.com/ceph/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.163.com/ceph/rpm-15.2.1/el7/SRPMS/
enabled=0
gpgcheck=1
priority=2
type=rpm-md
gpgkey=http://mirrors.163.com/ceph/keys/release.asc
```

root 用户之间要配置免密，过程省略

安装集群：

```bash
$ sudo ceph-deploy new fueltank-1 fueltank-2 fueltank-3 # 会在当前目录生成三个配置文件
# 在 ceph.conf 中添加 rbd_default_features = 1
$ sudo yum -y install ceph ceph-radosgw
$ sudo ceph-deploy install --no-adjust-repos fueltank-1 fueltank-2 fueltank-3
```

完成后，每台机器上的 `ceph-crash` 服务都会启动起来：

```bash
$ systemctl status ceph-crash
```

然后：

```bash
$ sudo ceph-deploy --overwrite-conf mon create-initial
```

这步完成后，会在当前目录生成以下文件：

- `ceph.client.admin.keyring`
- `ceph.bootstrap-mgr.keyring`
- `ceph.bootstrap-osd.keyring`
- `ceph.bootstrap-mds.keyring`
- `ceph.bootstrap-rgw.keyring`
- `ceph.bootstrap-rbd.keyring`
- `ceph.bootstrap-rbd-mirror.keyring`

然后再执行：

```bash
$ sudo ceph-deploy admin fueltank-1 fueltank-2 fueltank-3
```

这会把 `ceph.client.admin.keyring` 文件拷贝到各主机的 `/etc/ceph` 目录。

创建管理守护程序：

```bash
$ sudo ceph-deploy mgr create fueltank-1
```

创建 OSD，磁盘需要自己准备，`fdisk -l` 查看磁盘，我这里创建三个 OSD:

```bash
$ sudo ceph-deploy osd create --data /dev/vdb fueltank-1
$ sudo ceph-deploy osd create --data /dev/vdb fueltank-2
$ sudo ceph-deploy osd create --data /dev/vdb fueltank-3
```

查看状态：

```bash
$ sudo ceph health
$ sudo ceph -s
```



---



## 部署 RGW

```bash
$ sudo ceph-deploy rgw create fueltank-1
```

查看状态：

```bash
$ sudo systemctl status ceph-radosgw@rgw.fueltank-1
```

访问：

```bash
$ curl http://fueltank-1:7480
```





---



## 安装 Dashboard

```bash
$ sudo yum install ceph-mgr-dashboard -y
```

我靠，新版本有坑，有一个依赖 CentOS 不支持，放弃，反正也很鸡肋，不要了。

```bash
$ docker run -p 5000:5000 -e CEPHMONS='172.20.21.16,172.20.21.17,172.20.20.115' -e KEYRING="$(sudo cat /etc/ceph/ceph.client.admin.keyring)" crapworks/ceph-dash:latest
```





---



## 测试储存一个对象

创建 pool ，并往里面添加一个对象。

```bash
$ echo "Hello BBD" > testfile.txt
$ sudo ceph osd pool create mytest
$ sudo rados put test-object testfile.txt --pool=mytest
```

查看对象列表：

```bash
$ sudo rados -p mytest ls
```

下载对象：

```bash
$ sudo rados -p mytest get test-object abc.txt
```

确定对象储存的位置：

```bash
$ sudo ceph osd map mytest test-object
```

删除对象：

```bash
$ sudo rados rm test-object --pool=mytest
```

再次查看对象列表，发现刚才的对象被删除了：

```bash
$ sudo rados -p mytest ls
```

删除 pool：

```bash
$ sudo ceph osd pool rm mytest
```

报错，说：

```
Error EPERM: WARNING: this will *PERMANENTLY DESTROY* all data stored in pool mytest.  If you are *ABSOLUTELY CERTAIN* that is what you want, pass the pool name *twice*, followed by --yes-i-really-really-mean-it.
```

意思是 pool 名字要写两次，并且要带 `--yes-i-really-really-mean-it` 参数：

```bash
$ sudo ceph osd pool rm mytest mytest --yes-i-really-really-mean-it
```

又报错，说：

```
Error EPERM: pool deletion is disabled; you must first set the mon_allow_pool_delete config option to true before you can destroy a pool
```

这样解决：

```bash
$ sudo ceph tell mon.\* injectargs '--mon-allow-pool-delete=true'
$ sudo ceph osd pool rm mytest mytest --yes-i-really-really-mean-it
```

查看 pool 列表：

```bash
$ sudo ceph osd pool ls
```



---



## 卸载

```bash
$ sudo ceph-deploy purge fueltank-1 fueltank-2 fueltank-3
$ sudo ceph-deploy purgedata fueltank-1 fueltank-2 fueltank-3
$ sudo ceph-deploy forgetkeys
```



---



## 错误记录

有一个错误说：

````
Module 'restful' has failed dependency: No module named 'werkzeug'
````

解决方案，在各个节点下执行：

```
pip3 install werkzeug
```

然后重启各节点的 ceph-mon 和 ceph-mgr





