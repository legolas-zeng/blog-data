# HDFS 常用操作

拉数据

```bash
$ hdfs dfs -get /user/abc/ ./
```

查看大小

```bash
$ hdfs dfs -du -s -h /user/adc/
```

上传

```bash
$ hdfs dfs -put file /user/abc/
```

移动数据：

```bash
$ hadoop fs -mv < hdfs file >  < hdfs file >
```



## 修改 HDFS 目录权限

```
su - hdfs -s /bin/bash -c "hadoop fs -chmod 755 /user/spark"
```

