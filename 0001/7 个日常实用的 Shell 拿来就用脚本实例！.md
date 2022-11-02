## 7 个日常实用的 Shell 拿来就用脚本实例！

今天再来给大家分享 7 个日常实用脚本；

#### 1、list_sys_status.sh

显示系统使用的以下信息：

主机名、IP地址、子网掩码、网关、DNS服务器IP地址信息

```
#!/bin/bash
IP=`ifconfig eth0 | head -2 | tail -1 | awk '{print $2}' | awk -F":" '{print $2}'`
ZW=` ifconfig eth0 | head -2 | tail -1 | awk '{print $3}' | awk -F":" '{print $2}'`
GW=`route -n | tail -1 | awk '{print $2}'`
HN=`hostname`
DNS=`head -1 /etc/resolv.conf | awk '{print $2}'`
echo '此机IP地址是' $IP
echo '此机子网掩码是' $ZW
echo '此机网关是' $GW
echo '此机主机名是' $HN
echo '此机DNS是' $DNS
```

#### 2、mysqlbak.sh备份数据库目录脚本

```
#!/bin/bash
DAY=`date +%Y%m%d`
SIZE=`du -sh /var/lib/mysql`
echo "Date: $DAY" >> /tmp/dbinfo.txt
echo "Data Size: $SIZE" >> /tmp/dbinfo.txt
cd /opt/dbbak &> /dev/null || mkdir /opt/dbbak
tar zcf /opt/dbbak/mysqlbak-${DAY}.tar.gz /var/lib/mysql /tmp/dbinfo.txt &> /dev/null
rm -f /tmp/dbinfo.txt

crontab-e
55 23 */3 * * /opt/dbbak/dbbak.sh
```

#### 3、每周日半夜23点半，对数据库服务器上的webdb库做完整备份

每备份文件保存到系统的/mysqlbak目录里

用系统日期做备份文件名 webdb-YYYY-mm-dd.sql

每次完整备份后都生成新的binlog日志

把当前所有的binlog日志备份到/mysqlbinlog目录下

```
#mkdir /mysqlbak 
#mkdir /mysqlbinlog
#service mysqld start
cd /shell
#vi webdb.sh
#!/bin/bash
day=`date +%F`
mysqldump -hlocalhost -uroot -p123 webdb > /mysqlbak/webdb-${day}.sql
mysql -hlocalhost -uroot -p -e "flush logs"
tar zcf /mysqlbinlog.tar.gz /var/lib/mysql/mysqld-bin.0*
#chmod +x webdb.sh 
#crontab -e
30 23 * * 7 /shell/webdb.sh
```

#### 4、very.ser.sh(检查任意一个服务的运行状态)

只检查服务vsftpd httpd sshd crond、mysql中任意一个服务的状态

如果不是这5个中的服务，就提示用户能够检查的服务名并退出脚本

如果服务是运行着的就输出 "服务名 is running"

如果服务没有运行就启动服务

**方法1：使用read写脚本**

```
#!/bin/bash
read -p "请输入你的服务名:" service
if [ $service != 'crond' -a $service != 'httpd' -a $service != 'sshd' -a $service != 'mysqld' -a $service != 'vsftpd' ];then
echo "只能够检查'vsftpd,httpd,crond,mysqld,sshd"
exit 5
fi
service $service status &> /dev/null

if [ $? -eq 0 ];thhen
echo "服务在线"
else
service $service start
fi
```

**方法2：使用位置变量来写脚本**

```
if [ -z $1 ];then
echo "You mast specify a servername!"
echo "Usage: `basename$0` servername"
exit 2
fi
if [ $1 == "crond" ] || [ $1 == "mysql" ] || [ $1 == "sshd" ] || [ $1 == "httpd" ] || [ $1 == "vsftpd" ];then
service $1 status &> /dev/null
if [ $? -eq 0 ];then
echo "$1 is running"
else
service $1 start
fi
else
echo "Usage:`basename $0` server name"
echo "But only check for vsftpd httpd sshd crond mysqld" && exit2
fi
```

#### 5、pc_noline.sh

输出192.168.1.0/24网段内在线主机的ip地址

统计不在线主机的台数，并把不在线主机的ip地址和不在线时的时间保存到/tmp/ip.txt文件里

```
#!/bin/bash
ip=192.168.1.
j=0
for i in `seq 10 12`
do
ping -c 3 $ip$i &> /dev/null
if [ $? -eq 0 ];then
echo 在线的主机有：$ip$i
else
let j++
echo $ip$i >> /tmp/ip.txt
date >> /tmp/ip.txt
fi
done
echo 不在线的主机台数有 $j
```

#### 6、一个简单的网站论坛测试脚本

用交互式的输入方法实现自动登录论坛数据库，修改用户密码

```
[root@test1 scripts]# vim input.sh

#!/bin/bash

End=ucenter_members
MYsql=/home/lnmp/mysql/bin/mysql

read -p "Enter a website directory : " webdir
WebPath=/home/WebSer/$webdir/config
echo $WebPath

read -p "Enter dbuser name : " dbuser
echo $dbuser

read -sp "Enter dbuser password : " dbpass

read -p "Enter db name : " dbname
echo $dbname

read -p "Enter db tablepre : " dbtablepre
echo $dbtablepre

Globalphp=`grep "tablepre*" $WebPath/config_global.php |cut -d "'" -f8`
Ucenterphp=`grep "UC_DBTABLEPRE*" $WebPath/config_ucenter.php |cut -d '.' -f2 | awk -F "'" '{print $1}'`

if [ $dbtablepre == $Globalphp ] && [ $dbtablepre == $Ucenterphp ];then

     Start=$dbtablepre
     Pre=`echo $Start$End`

     read -p "Enter you name : " userset
     echo $userset

     Result=`$MYsql -u$dbuser -p$dbpass $dbname -e "select username from $Pre where username='$userset'\G"|cut -d ' ' -f2|tail -1`
     echo $Result
     if [ $userset == $Result ];then
           read -p "Enter your password : " userpass
           passnew=`echo -n $userpass|openssl md5|cut -d ' ' -f2`

           $MYsql -u$dbuser -p$dbpass $dbname -e "update $Pre set password='$passnew' where username='$userset';"
           $MYsql -u$dbuser -p$dbpass $dbname -e "flush privileges;"
     else
           echo "$userset is not right user!"
           exit 1
     fi
else
     exit 2
fi
```

#### 7、slave_status.sh（检查mysql主从从结构中从数据库服务器的状态）

1）本机的数据库服务是否正在运行

2）能否与主数据库服务器正常通信

3）能否使用授权用户连接数据库服务器

4）本机的slave_IO进程是否处于YES状态

本机的slave_SQL进程是否处于YES状态

```
[root@test1 scripts]# vim test.sh

#!/bin/bash
netstat -tulnp | grep :3306 > /dev/null
if [ $? -eq 0 ];then
echo "服务正在运行" 
else
service mysqld start
fi
ping -c 3 192.168.1.100 &> /dev/null
if [ $? -eq 0 ];then
echo "网络连接正常" 
else
echo "网络连接失败"
fi
mysql -h192.168.1.100 -uroot -p123456 &> /dev/null
if [ $? -eq 0 ];then
echo "数据库连接成功" 
else
echo "数据库连接失败"
fi
IO= mysql -uroot -p123 -e "show slave status\G" | grep Slave_IO_Running | awk '{print $2}' > /dev/null
SQL= mysql -uroot -p123 -e "show slave status\G" | grep Slave_SQL_Running | awk '{print $2}' /dev/null
if [ IO==Yes ] && [ SQL==Yes ];then
echo “IO and SQL 连接成功”
else
echo "IO线程和SQL线程连接失败"
fi
```

以上就是今天分享的全部内容，建议大家使用上面这几种案例动手试试。

