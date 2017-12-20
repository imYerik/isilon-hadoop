# Prepare Isilon OneFS

## 1. Install HDFS & SmartConnect Advanced License  

```shell
Isilon-1# isi license activate --key XXXX-XXXX-XXXX-XXXX
Isilon-1# isi license list
```

## 2. Create Access Zone  

```shell
Isilon-1# isi zone zones create --name=zone1-apache-hadoop --path=/ifs/zone1-apache-hadoop --create-path
Isilon-1# isi zone zones list --verbose
```

## 3. SmartConnect Adv Configuration  

### Config SmartConnect Service IP  

```shell
Isilon-1# isi network subnets modify groupnet0.subnet0 --sc-service-addr=192.168.12.20
```

### Create an ip pool and Set SmartConnect ZoneName  

```shell
Isilon-1# isi network pools create --id=groupnet0:subnet0:apache-hadoop-pool --ranges=192.168.12.21-192.168.12.26 --access-zone=apache-hadoop-pool --alloc-method=dynamic --ifaces=1-3:10gige-1,1-3:10gige-2, --sc-subnet=subnet0 --sc-dns-zone=apache-hadoop-pool.isilon.com --description=hadoop
```

### Check ip pool  

```shell
Isilon-1# isi network pools view --id=groupnet0:subnet0:apache-hadoop-pool
```

## 4. DNS Configuration(Bind example)  

### Add NS and A record to bind configuration file  

```shell
zone1-apache-hadoop.isilon.com. NS ssip.isilon.com.
ssip.isilon.com. IN A 192.168.12.20
```  

### Ping test(each ping will return different IP address)  

```shell
Linux01# ping zone1-apache-hadoop.isilon.com
```

## 5. Isilon HDFS Configuration  

```shell
Isilon-1# isi zone zones modify --user-mapping-rules="hdfs=>root" --zone apache-hadoop-pool
```
### Set HDFS blocksize  

```shell
Isilon-1# isi hdfs settings modify --zone=apache-hadoop-pool --default-block-size=128M
```
### Set HDFS root directory  

```shell
Isilon-1# mkdir -p /ifs/apache-hadoop-pool/hdfs
Isilon-1# touch /ifs/apache-hadoop-pool/hdfs/THIS_IS_ISILON_apache-hadoop-pool.txt
Isilon-1# isi hdfs settings modify --zone=apache-hadoop-pool --root-directory=/ifs/apache-hadoop-pool/hdfs
```

### Check HDFS settings  

```shell
Isilon-1# isi hdfs settings view --zone=apache-hadoop-pool
```

### Modify ACL settings   

```shell
Isilon-1# isi auth settings acls modify --group-owner-inheritance=parent 
Isilon-1# isi auth settings acls view 
```

## 6. Create HDFS users, groups and directorys  
It's not recommanded only when you deploy Apache Hadoop.

#### Create user and group in Isilon  

```shell
Isilon-1# isi auth groups create hadoop --zone apache-hadoop-pool --provider local --gid 2000

Isilon-1# isi auth users create hadoop --primary-group hadoop --zone apache-hadoop-pool  --provider local --home-directory /ifs/apache-hadoop-pool/hdfs/user/hadoop --uid 2000

Isilon-1# isi_run -z2 chown hadoop:hadoop /ifs/apache-hadoop-pool/hdfs/user/hadoop 
Isilon-1# isi_run -z2 chmod 755 /ifs/apache-hadoop-pool/hdfs/user/hadoop
```

> z2 is zone ID=2(in this example is zone1-apache-hadoop,you can get zone id with command 'isi zone zones list --verbose')



#### Create user and group in Linux  

```shell
Linux01# groupadd hadoop -g 2000
Linux01# useradd hadoop -u 2000 -g 2000
Linux01# id hadoop
uid=2000(hadoop) gid=2000(hadoop) groups=2000(hadoop)
```

# Install Apache Hadoop
```shell
Linux01# mkdir /opt/apache/
Linux01# tar zxvf hadoop-2.7.3.tar.gz -C /opt/apache
Linux01# chown -R hadoop:hadoop /opt/apache/hadoop-2.7.3
```

## Add environment
```shell
Linux01# su - hadoop
Linux01$ vi .bash_profile 
...
export JAVA_HOME=/usr/lib/jvm/jre-1.7.0-openjdk.x86_64/
export HADOOP_HOME=/opt/apache/hadoop-2.7.3/
export PATH=$HADOOP_HOME/bin:$PATH
...
Linux01$ source .bash_profile
```

## Config hadoop client
```shell
Linux01$ vi /opt/apache/hadoop-2.7.3/etc/hadoop/core-site.xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://zone1-apache-hadoop.isilon.com:8020</value>
    </property>
    <property>
        <name>io.file.buffer.size</name>
        <value>131072</value>
    </property>
    <property> 
        <name>fs.trash.interval</name> 
        <value>0</value> 
    </property>
</configuration>
```


## Test Isilon HDFS
```shell
Linux01$ touch testfile
Linux01$ hadoop fs -mkdir /input
Linux01$ hadoop fs -put testfile /input
```


## Performance Test
```shell
Linux01$ hadoop jar /opt/apache/hadoop-2.7.3/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-2.7.3-tests.jar TestDFSIO -clean
Linux01$ hadoop jar /opt/apache/hadoop-2.7.3/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-2.7.3-tests.jar TestDFSIO -write -nrFiles 50 -size 1000MB -resFile /tmp/TestDFSIO-$(date +%H-%M-%s).log
```