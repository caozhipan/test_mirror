# mysql data_dir迁移



*mysql 默认data目录放在/var/lib/mysql 由于空间限制，挂载了新的硬盘，希望把mysqldata目录迁移到新的硬盘上*

```nginx
#old config
datadir         = /var/lib/mysql
 log_error = /var/log/mysql/error.log
```

``` shell
root@sj-test:/etc/mysql# du -sh /var/lib/mysql
4.8G	/var/lib/mysql
```



## 迁移步骤

* 修改配置

``` nginx
# new config
datadir         = /data/mysql
 log_error = /data/logs/mysql/error.log
```

* 停止服务

``` shell
service mysql stop
```

* 迁移数据目录

```shell
cp -r /var/lib/mysql /tmp/mysql
mv /var/lib/mysql /data/mysql

```



* 修改apparmor配置

```shell
sed -ie '/network/a\\n  \/data\/mysql\/ r,\n  \/data\/mysql\/** rwk,\n  \/data\/logs\/mysql\/** rw,' /etc/apparmor.d/usr.sbin.mysqld
```

* 重启服务

```shell
/etc/init.d/apparmor restart
service mysql start
```



