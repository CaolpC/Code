## Oracle 数据库重启

### 开发环境登录数据库
```shell
host: 10.0.250.175 - db1-f1
用户名：root
密码：1q2w3e4R
```

### 开发环境重启数据库

```shell
su - oracle
sqlplus / as sysdba
SQL> shutdown immediate;
如果命令发不出去，就直接shutdown abort(直接关闭数据库，正在访问数据库的会话会被突然终止跟kill 进程是一样的效果，生产环境不要用，所有正在运行的SQL语句都将立即中止。所有未提交的事务将不回滚)
SQL> startup    启动服务
SQL> alter pluggable database bosspdb open;
如果不执行这个，dataGrip连接的时候会报：ora-01033:oracle initialization or shutdown in progress；

如果是机器重启，那么需要重新进行监听
启动监听
lsnrctl start
lsnrctl reload
lsnrctl status
```

![20211008153747](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20211008153747.png)

### SQL 字符串截取函数
```sql
    select substr('12345678', -8, 8) FROM dual; #运行结果：'12345678'
    select substr('12345678', -9, 9) FROM dual; #运行结果：null

    select 1 from dual where null like '%019';  #无运行结果
    select 1 from dual WHERE '019' like '%' || null #运行结果：1
```