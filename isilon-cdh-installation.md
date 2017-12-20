#Prepare Isilon OneFS

##1. Install HDFS & SmartConnect Advanced License  

```shell
Isilon-1# isi license activate --key XXXX-XXXX-XXXX-XXXX
Isilon-1# isi license list
```

##2. Create Access Zone  

```shell
Isilon-1# isi zone zones create --name=zone1-cdh --path=/ifs/zone1-cdh --create-path
Isilon-1# isi zone zones list --verbose
```

##3. SmartConnect Adv Configuration  

###Config SmartConnect Service IP  

```shell
Isilon-1# isi network subnets modify groupnet0.subnet0 --sc-service-addr=192.168.12.20
```

###Create an ip pool and Set SmartConnect ZoneName  

```shell
Isilon-1# isi network pools create --id=groupnet0:subnet0:hadoop-pool-cdh --ranges=192.168.12.21-192.168.12.26 --access-zone=zone1-cdh --alloc-method=dynamic --ifaces=1-3:10gige-1,1-3:10gige-2, --sc-subnet=subnet0 --sc-dns-zone=zone1-cdh.isilon.com --description=hadoop
```

###Check ip pool  

```shell
Isilon-1# isi network pools view --id=groupnet0:subnet0:hadoop-pool-cdh
```

##4. DNS Configuration(Bind example)  

### Add NS and A record to bind configuration file  

```shell
zone1-cdh.isilon.com. NS ssip.isilon.com.
ssip.isilon.com. IN A 192.168.12.20
```  

### Ping test(each ping will return different IP address)  

```shell
Linux01# ping zone1-cdh.isilon.com
```

##5. Isilon HDFS Configuration  

```shell
Isilon-1# isi zone zones modify --user-mapping-rules="hdfs=>root" --zone zone1-cdh
```
###Set HDFS blocksize  

```shell
Isilon-1# isi hdfs settings modify --zone=zone1-cdh --default-block-size=128M
```
###Set HDFS root directory  

```shell
Isilon-1# mkdir -p /ifs/zone1-cdh/hdfs
Isilon-1# touch /ifs/zone1-cdh/hdfs/THIS_IS_ISILON_zone1-cdh.txt
Isilon-1# isi hdfs settings modify --zone=zone1-cdh --root-directory=/ifs/zone1-cdh/hdfs
```

###Check HDFS settings  

```shell
Isilon-1# isi hdfs settings view --zone=zone1-cdh
```

###Modify ACL settings   

```shell
Isilon-1# isi auth settings acls modify --group-owner-inheritance=parent 
Isilon-1# isi auth settings acls view 
```

##6. Create HDFS users, groups and directorys  

###Download scripts  

Link https://github.com/Isilon/isilon_hadoop_tools.
Download and upload it to isilon directory with scp.

```shell
Isilon-1# mkdir -p /ifs/zone1-cdh/scripts
Isilon-1# cd /ifs/zone1-cdh/scripts
Isilon-1# unzip isilon_hadoop_tools.zip
Isilon-1# cd isilon_hadoop_tools
```

###Create users and groups  

This will generate zone1-cdh1.passwd and zone1-cdh1.group file. 
```shell
Isilon-1# bash isilon_create_users.sh --dist cdh --startuid 1001 --startgid 1001 --zone zone1-cdh
```

###Create directorys  

```shell
Isilon-1# bash isilon_create_directories.sh --dist cdh --zone zone1-cdh
Isilon-1# ls -al /ifs/zone1-cdh/hdfs
Isilon-1# ls -al /ifs/zone1-cdh/hdfs/user
```

###Config Linux Clients Users and Group  

Upload zone1-cdh1.passwd and zone1-cdh1.group to all CDH nodes, append the content to passwd and group file, CDH will not create these users and groups during deployment.
```shell
Linux01# cat zone1-cdh1.passwd >> /etc/passwd
Linux01# cat zone1-cdh1.group >> /etc/group
Linux02# cat zone1-cdh1.passwd >> /etc/passwd
Linux02# cat zone1-cdh1.group >> /etc/group
```

###You can also manually create users.(it's not recommanded only when you deploy Apache Hadoop)  

####Create user and group in Isilon  

```shell
Isilon-1# isi auth groups create hadoop --zone zone1-cdh --provider local --gid 2000

Isilon-1# isi auth users create hadoop --primary-group hadoop --zone zone1-cdh  --provider local --home-directory /ifs/zone1-cdh/hdfs/user/hadoop --uid 2000

Isilon-1# isi_run -z2 chown hadoop:hadoop /ifs/zone1-cdh/hdfs/user/hadoop 
Isilon-1# isi_run -z2 chmod 755 /ifs/zone1-cdh/hdfs/user/hadoop
```

####Create user and group in Linux  

```shell
Linux01# groupadd hadoop -g 2000
Linux01# useradd hadoop -u 2000 -g 2000
Linux01# id hadoop
uid=2000(hadoop) gid=2000(hadoop) groups=2000(hadoop)
```

#Deploy Cloudera  

```
default_fs_name hdfs://zone1-cdh.isilon.com:8020 
webhdfs_url http://zone1-cdh.isilon.com:8082/webhdfs/v1
```