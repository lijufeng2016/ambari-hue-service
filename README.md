## 版本信息

Ambari：2.7.4

HDP：3.1.4

HUE：4.6.0

## 环境准备

1.hue的master节点上执行，为编译环境做准备

```shell
yum install sqlite-devel  libxslt-devel.x86_64 python-devel openldap-devel asciidoc cyrus-sasl-gssapi  libxml2-devel.x86_64 mysql-devel gcc gcc-c++ kernel-devel openssl-devel gmp-devel libffi-devel install npm
```

2.所有机器上创建用户和组

```shell
useradd -g hue hue
```

3.提前在mysql创建好hue的库并授权

```sql
CREATE DATABASE hue;
GRANT ALL PRIVILEGES ON hue.* TO hue@'%' IDENTIFIED BY '123456';
FLUSH PRIVILEGES;
```

4.提前建好hue在hdfs的HOME目录

```shell
hadoop fs -mkdir /user/hue
hadoop fs -chown hue:hue /user/hue
```

5.下载插件源码

在ambari server节点执行

```shell
VERSION=`hdp-select status hadoop-client | sed 's/hadoop-client - \([0-9]\.[0-9]\).*/\1/'`
rm -rf /var/lib/ambari-server/resources/stacks/HDP/$VERSION/services/HUE  
sudo git clone https://github.com/lijufeng2016/ambari-hue-service.git /var/lib/ambari-server/resources/stacks/HDP/$VERSION/services/HUE
```

6.hue的安装包并放到你的Apache服务器上

![15837480895742](https://github.com/lijufeng2016/ambari-hue-service/blob/master/screenshots/15837480895742.png)


在ambari server节点执行

## 代码修改

1. package/files/configs.sh文件

   ```
   USERID='ambari的管理员账号'
   PASSWD='ambari的管理员密码'
   ```

2. package/scripts/params.py文件

   第32行 download_url 改成你自己的地址，可以跟hdp的本地仓库放一起

   第40行 ambari_server_hostname 改成你自己的地址

## 部署安装

重启ambari

```shell
ambari-server restart
```

ambari界面操作

界面左侧 >> services >> Add service >> Hue >> NEXT >> 选择Hue Server >> NEXT >> 配置 

数据库配置，这里选了mysql：

![1583732152511](https://github.com/lijufeng2016/ambari-hue-service/blob/master/screenshots/1583732152511.png)

![1583732214223](https://github.com/lijufeng2016/ambari-hue-service/blob/master/screenshots/1583732214223.png)



一路next， 启动失败先忽视

![1583733010169](https://github.com/lijufeng2016/ambari-hue-service/blob/master/screenshots/1583733010169.png)

**启动hue报错**：

```
UnicodeEncodeError: 'ascii' codec can't encode character u'\u201c' in position 3462: ordinal not in range(128)
```

![1583746893993](https://github.com/lijufeng2016/ambari-hue-service/blob/master/screenshots/1583746893993.png)

**解决办法**：

在hue的安装节点上：

```shell
vim /usr/lib/ambari-agent/lib/resource_management/core/sudo.py
```

**网上的说的/usr/lib/python2.7/site-packages/resource_management/core/sudo.py文件在新版本中不适用！**

添加如下代码：

  import sys
  reload(sys)
  sys.setdefaultencoding('utf-8')

![1583746926853](https://github.com/lijufeng2016/ambari-hue-service/blob/master/screenshots/1583746926853.png)



## 编译

```shell
cd /usr/hdp/3.1.4.0-315/hue/
make apps
```

**注意：在准备工作的第一步的包必须安装才能编译成功**



**再次启动报错：Invalid HTTP_HOST header: 'sh05-hdp3-manage002:8888'. You may need to add u'sh05-hdp3-manage002' to ALLOWED_HOSTS.**

![1583737741804](https://github.com/lijufeng2016/ambari-hue-service/blob/master/screenshots/1583737741804.png)

解决办法：找到Advanced pseudo-distributed.ini 配置

  allowed_hosts=*

![1583737898638](https://github.com/lijufeng2016/ambari-hue-service/blob/master/screenshots/1583737898638.png)



改完重启，终于看到人样的页面了，输入hue hue，你随意

![1583737992211](https://github.com/lijufeng2016/ambari-hue-service/blob/master/screenshots/1583737992211.png)

进去发现加载数据库错误

![1583738042687](https://github.com/lijufeng2016/ambari-hue-service/blob/master/screenshots/1583738042687.png)

解决方法：

```shell
vim /usr/hdp/3.1.4.0-315/hue/desktop/core/src/desktop/lib/conf.py
```

第293行改为：

```python
if raw is None or raw=='':
```



再次重启hue，又报错：ERROR    Error running create_session

![1583738358751](https://github.com/lijufeng2016/ambari-hue-service/blob/master/screenshots/1583738358751.png)

明显的哪里把端口号当成字符串输入了

解决方法：

把hiveserver2的host和端口号手动设置一下

![1583739139434](https://github.com/lijufeng2016/ambari-hue-service/blob/master/screenshots/1583739139434.png)

重启，又报错：**TSocket read 0 bytes (code THRIFTTRANSPORT): TTransportException('TSocket read 0 bytes',)**

![1583743353941](https://github.com/lijufeng2016/ambari-hue-service/blob/master/screenshots/1583743353941.png)

解决办法：

在beeswax的配置下面加上  use_sasl=true

从哪里跌倒就从哪里爬起，再重启，页面终于正常啦！尽情的玩耍了！咦？不对啊？

![1583743457835](https://github.com/lijufeng2016/ambari-hue-service/blob/master/screenshots/1583743457835.png)

怎么会显示无列呢？

新版的hue4.6.0与hdp3.1.4这种双新组合往往自带坑位，网上也找不到任何答案，经过一步步推测排查，首先可以确定的是后端返回字段的时候有问题，与之相关的是hive相关的包，经过漫长的一步一步排查，确定到了哪一行代码，python真不习惯，也没开发过，太难了，中间过程就不细说了，直接上解决方法：

```shell
vim /usr/hdp/3.1.4.0-315/hue/apps/beeswax/src/beeswax/server/hive_server2_lib.py
```

118行和119行的2改成1即可

![1583743805356](https://github.com/lijufeng2016/ambari-hue-service/blob/master/screenshots/1583743805356.png)



重启，想查看hdfs，又遇到问题

![1583743909954](https://github.com/lijufeng2016/ambari-hue-service/blob/master/screenshots/1583743909954.png)

解决方法：

webhdfs_url=http://sh05-hdp3-manage001:50070/webhdfs/v1

![1583743978311](https://github.com/lijufeng2016/ambari-hue-service/blob/master/screenshots/1583743978311.png)



继续，查看hbase报错

**Api 错误：HTTPConnectionPool(host='sh05-hdp3-manage003', port=9090): Max retries exceeded with url: / (Caused by NewConnectionError(': Failed to establish a new connection: [Errno 111] Connection refused',))**

![1583744318350](https://github.com/lijufeng2016/ambari-hue-service/blob/master/screenshots/1583744318350.png)

解决办法：

hdp3中，hbase的thrift默认不开启，需要手动在各个**Hmaster节点**启动，注意，一定要**使用hbase用户**启动**thrift**，而不是**thrift2**！！否则后面还会有问题，上代码

```shell
su hbase
/usr/hdp/current/hbase-master/bin/hbase-daemon.sh start thrift
```

建议把启动thrift的命令写到启动hbase master的脚本里，这样就不用每次手动起了



到这里基本上差不多解决了，记住一定要按步骤来，一步棋错全盘皆输！在解决这些问题的时候折腾了很久。在ambari-hue-service插件上面的改造上面画了比较长的时间，由于是全新版本，只能站在巨人肩膀上。安装过程中，兼容问题频频发生，需要耐心的从原理源码角度出发解决问题，所有问题都不是问题！