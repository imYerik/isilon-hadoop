# Installation Plan  

## Software Version  
```shell
CentOS: 6.6 x64
HDP: ambari 2.1.0
Jdk: 1.8
Isilon: OneFS 8.0.1.2
```

## Isilon 
```shell
Isilon IP:192.168.64.201-192.168.64.203
Isilon SSIP:192.168.64.200
Isilon SmartConnect Name:zone1-hdp.hdfs.isilon.com
```

## HDP   

```shell
DNS Records
192.168.64.12 ambari.isilon.com
192.168.64.12 linux02.isilon.com
192.168.64.13 linux03.isilon.com
192.168.64.14 linux04.isilon.com
```

## Others  
```shell
DNS: 192.168.64.12
NTP: 192.168.64.12
YUM Server:192.168.64.12
```
# Prepare Isilon OneFS

## 1. Install HDFS & SmartConnect Advanced License  

```shell
Isilon-1# isi license activate --key XXXX-XXXX-XXXX-XXXX
Isilon-1# isi license list
```

## 2. Create Access Zone  
```shell
Isilon-1# isi zone zones create --name=zone1-hdp --path=/ifs/zone1-hdp --create-path
Isilon-1# isi zone zones list --verbose
```

## 3. SmartConnect Adv Configuration  

```shell
Isilon-1# isi network subnets modify groupnet0.subnet0 --sc-service-addr=192.168.64.200
Isilon-1# isi network pools create --id=groupnet0:subnet0:hadoop-pool-hdp --ranges=192.168.64.201-192.168.64.203 --access-zone=zone1-hdp --alloc-method=dynamic --ifaces=1-3:10gige-1,1-3:10gige-2, --sc-subnet=subnet0 --sc-dns-zone=zone1-hdp.hdfs.isilon.com --description=hadoop

Isilon-1# isi network pools view --id=groupnet0:subnet0:hadoop-pool-hdp
```


## 4. DNS Configuration(Bind example)   
```shell
hdfs.isilon.com. NS ssip.isilon.com.
ssip.isilon.com. IN A 192.168.64.200
```

```shell
Linux02# ping zone1-hdp.hdfs.isilon.com
```

## 5. Isilon HDFS Configuration  

### Isilon HDFS Basic Configuration  
```shell
Isilon-1# isi zone zones modify --user-mapping-rules="hdfs=>root" --zone zone1-hdp
Isilon-1# isi hdfs settings modify --zone=zone1-hdp --default-block-size=128M
Isilon-1# mkdir -p /ifs/zone1-hdp/hdfs
Isilon-1# touch /ifs/zone1-hdp/hdfs/THIS_IS_ISILON_zone1-hdp.txt
Isilon-1# isi hdfs settings modify --zone=zone1-hdp --root-directory=/ifs/zone1-hdp/hdfs
```

### Isilon HDFS Ambari Configuration  
```shell
Isilon-1# isi hdfs settings modify --zone=zone1-hdp --ambari-namenode=zone1-hdp.hdfs.isilon.com
Isilon-1# isi hdfs settings modify --zone=zone1-hdp --ambari-server=ambari.isilon.com
Isilon-1# isi hdfs settings view --zone=zone1-hdp
```


### Modify ACL settings  
```shell
Isilon-1# isi auth settings acls modify --group-owner-inheritance=parent 
Isilon-1# isi auth settings acls view 
```

## 6. Create HDFS users, groups and directorys  
```shell
Isilon-1# mkdir -p /ifs/zone1-hdp/scripts
Isilon-1# cd /ifs/zone1-hdp/scripts
```

Download scripts from https://github.com/Isilon/isilon_hadoop_tools
Upload to isilon script directory with scp.

### Create users and groups  
This will generate zone1-hdp1.passwd and zone1-hdp1.group file.  
```shell
Isilon-1# unzip isilon_hadoop_tools.zip
Isilon-1# cd isilon_hadoop_tools
Isilon-1# bash isilon_create_users.sh --dist hwx --startuid 1001 --startgid 1001 --zone zone1-hdp
```

### Create directorys  
```shell
isilon-1# bash isilon_create_directories.sh --dist hwx --fixperm --zone zone1-hdp
Isilon-1# ls -al /ifs/zone1-hdp/hdfs
Isilon-1# ls -al /ifs/zone1-hdp/hdfs/user
```

### Deploy passwd and group to client  
Upload zone1-hdp1.passwd and zone1-hdp1.group to all hdp nodes, append the content to passwd and group file, hdp will not create these users and groups during deployment.  
```shell
Linux01# cat zone1-hdp1.passwd >> /etc/passwd
Linux01# cat zone1-hdp1.group >> /etc/group
Linux02# cat zone1-hdp1.passwd >> /etc/passwd
Linux02# cat zone1-hdp1.group >> /etc/group
...
```

### You can also manually create users.(Not recommanded)  
Isilon side
```shell
Isilon-1# isi auth groups create hadoop --zone zone1-hdp --provider local --gid 2000
Isilon-1# isi auth users create hadoop --primary-group hadoop --zone zone1-hdp  --provider local --home-directory /ifs/zone1-hdp/hdfs/user/hadoop --uid 2000
Isilon-1# isi_run -z2 chown hadoop:hadoop /ifs/zone1-hdp/hdfs/user/hadoop 
Isilon-1# chmod 755 /ifs/zone1-hdp/hdfs/user/hadoop
```

Linux Client side
```shell
Linux01# groupadd hadoop -g 2000
Linux01# useradd hadoop -u 2000 -g 2000
Linux01# id hadoop
uid=2000(hadoop) gid=2000(hadoop) groups=2000(hadoop)
```

# Deploy HDP  

## 1.Prepare Linux Environment  

### Keep default /etc/hosts
```shell
# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
```

### Change Hostname to FQDN  
```shell
[root@linux02 ~]# vi /etc/sysconfig/network
HOSTNAME=linux02.isilon.com
```

### Verify the Hostname  
```shell
[root@linux02 ~]# hostname linux02.isilon.com
[root@linux02 ~]# hostname -f
linux02.isilon.com
[root@linux02 ~]# hostname -d
isilon.com
[root@linux02 ~]# hostname -s
linux02
```

### Generate RSA Token and Distribute to All Linux Client
```shell
[root@linux02 ~]# ssh-keygen
[root@linux02 ~]# ssh-copy-id linux02.isilon.com
[root@linux02 ~]# ssh-copy-id linux03.isilon.com
[root@linux02 ~]# ssh-copy-id linux04.isilon.com
```

### Disable Transparent Hugepage(All Linux Client)
```shell
[root@linux02 ~]# vi /etc/rc.local
echo never > /sys/kernel/mm/redhat_transparent_hugepage/enabled
[root@linux02 ~]# echo never > /sys/kernel/mm/redhat_transparent_hugepage/enabled
[root@linux02 ~]# grep -i HugePages_Total /proc/meminfo 
HugePages_Total:       0
```

### YUM Servicve  
All the server has configured local YUM with centos iso or point to a YUM server
```shell
[root@linux02 ~]# cat /etc/yum.repos.d/CentOS-Media.repo 
[c6-media]
name=CentOS-$releasever - Media
baseurl=file:///media/CentOS/
        file:///media/cdrom/
        file:///media/cdrecorder/
gpgcheck=0
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-6

```

## 2.Install Ambari Server  

### Install throuth Internet  
```shell
[root@linux02 ~]# wget http://public-repo-1.hortonworks.com/ambari/centos6/2.x/updates/2.1.0/ambari.repo -O /tmp/ambari.repo
[root@linux02 ~]# yum install ambari-server
```

### Setup Ambari Server  
The setup command will download Oracle JDK and Configure Database.

```shell
[root@linux02 ~]# ambari-server setup
Using python  /usr/bin/python2.6
Setup ambari-server
Checking SELinux...
SELinux status is 'disabled'
Customize user account for ambari-server daemon [y/n] (n)? 
Adjusting ambari-server permissions and ownership...
Checking firewall status...
Checking JDK...
[1] Oracle JDK 1.8 + Java Cryptography Extension (JCE) Policy Files 8
[2] Oracle JDK 1.7 + Java Cryptography Extension (JCE) Policy Files 7
[3] Custom JDK
==============================================================================
Enter choice (1): 1
To download the Oracle JDK and the Java Cryptography Extension (JCE) Policy Files you must accept the license terms found at http://www.oracle.com/technetwork/java/javase/terms/license/index.html and not accepting will cancel the Ambari Server setup and you must install the JDK and JCE files manually.
Do you accept the Oracle Binary Code License Agreement [y/n] (y)? y
Downloading JDK from http://public-repo-1.hortonworks.com/ARTIFACTS/jdk-8u40-linux-x64.tar.gz to /var/lib/ambari-server/resources/jdk-8u40-linux-x64.tar.gz
jdk-8u40-linux-x64.tar.gz... 100% (165.2 MB of 165.2 MB)
Successfully downloaded JDK distribution to /var/lib/ambari-server/resources/jdk-8u40-linux-x64.tar.gz
Installing JDK to /usr/jdk64/
Successfully installed JDK to /usr/jdk64/
Downloading JCE Policy archive from http://public-repo-1.hortonworks.com/ARTIFACTS/jce_policy-8.zip to /var/lib/ambari-server/resources/jce_policy-8.zip

Successfully downloaded JCE Policy archive to /var/lib/ambari-server/resources/jce_policy-8.zip
Installing JCE policy...
Completing setup...
Configuring database...
Enter advanced database configuration [y/n] (n)? 
Configuring database...
Default properties detected. Using built-in database.
Configuring ambari database...
Checking PostgreSQL...
Running initdb: This may take upto a minute.
Initializing database: [  OK  ]

About to start PostgreSQL
Configuring local database...
Connecting to local database...done.
Configuring PostgreSQL...
Restarting PostgreSQL
Extracting system views...
...ambari-admin-2.1.0.1470.jar
...
Adjusting ambari-server permissions and ownership...
Ambari Server 'setup' completed successfully.
```

### Start Ambari Servicve  
```shell
[root@linux02 ~]# service ambari-server start
Using python  /usr/bin/python2.6
Starting ambari-server
Ambari Server running with administrator privileges.
Organizing resource files at /var/lib/ambari-server/resources...
Server PID at: /var/run/ambari-server/ambari-server.pid
Server out at: /var/log/ambari-server/ambari-server.out
Server log at: /var/log/ambari-server/ambari-server.log
Waiting for server start....................
Ambari Server 'start' completed successfully.
```


## 3.Configuring HDP Services  

### Login Ambari Server and Launch the Wizard  

Ambari Management Console: http://192.168.64.12:8080  
User and Password: admin/admin

### Add HDP Host with RSA Private Key  
```shell
[root@linux02 ~]# ssh-keygen 
[root@linux02 ~]# cat ~/.ssh/id_rsa
```
```
linux02.isilon.com
linux03.isilon.com
linux03.isilon.com
```

### Add Isilon node  
Back and add Isilon smartconnect zone name without Authoration.
zone1-hdp.hdfs.isilon.com

### Customize the Service you Need  
HDFS, YARN, Zookeeper


### Assign Masters  

On the Assign Masters screen, assign the NameNode and SNameNode components to the Isilon OneFS cluster. Remove the other Isilon OneFS cluster components.
Assign all other master components to Hadoop client host(s).

### Assign Slaves and Clients
On the Assign Slaves and Clients screen, ensure that DataNode is selected for the Isilon OneFS cluster, and cleared for all the other hosts.
Ensure that the Client option is selected for all the other desired hosts and cleared for the Isilon OneFS cluster. Clear any other components that are assigned to the Isilon OneFS cluster.


### On the Customize Services screen
For the HDFS service, on the Advanced settings tab, update the following settings in the Advanced hdfs site settings section:
a. Change the dfs.namenode.http-address PORT to the FQDN for the SmartConnect Zone Name followed by port 8082 from 50070.
b. Change the dfs.namenode.https-address to the FQDN for the SmartConnect Zone Name followed by port 8080 from 50470.
c. Add a property in the Custom hdfs-site field by the name dfs.client-write-packet-size.
d. Set dfs.client-write-packet-size to 131072.
e. Change the dfs.datanode.http.address port from 50075 to 8082. This setting prevents an error that generates a traceback in ambari-server.log each time you log in to the Ambari server.

For YARN Advance Setting
dfs.client-write-packet-size 131072


## 4.Test HDFS  

```shell
[root@linux02 ~]# su - hdfs
-bash-4.1$ hadoop fs -ls /
Found 11 items
-rw-r--r--   3 root   wheel           0 2018-01-10 06:55 /THIS_IS_ISILON_zone1-hdp.txt
drwxrwxrwx   - yarn   hadoop          0 2018-01-10 09:57 /app-logs
drwxr-xr-x   - hdfs   hadoop          0 2018-01-10 07:14 /apps
drwxr-xr-x   - root   hadoop          0 2018-01-10 09:47 /hdp
drwxr-xr-x   - mapred hadoop          0 2018-01-10 07:14 /mapred
drwxrwxrwx   - mapred hadoop          0 2018-01-10 09:47 /mr-history
drwxr-xr-x   - root   hadoop          0 2018-01-10 07:15 /system
drwxrwxrwx   - hdfs   hdfs            0 2018-01-10 09:48 /tmp
drwxr-xr-x   - hdfs   hdfs            0 2018-01-10 07:15 /user
-bash-4.1$ hadoop fs -mkdir /in
-bash-4.1$ hadoop fs -put /var/log/message /in
-bash-4.1$ yarn jar /usr/hdp/2.3.6.0-3796/hadoop-mapreduce/hadoop-mapreduce-examples.jar wordcount /in /out

```

