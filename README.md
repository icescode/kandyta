# Kandyta

Tujuan Kandyta adalah membangun sebuah tata kelola Big Data yang portable, mudah dipakai dan mudah diimplementasi dalam ekosistim Big Data menggunakan teknologi Apache Hadoop, Apache Spark, Apache Parquet , Apache Drill, pustaka Machine Learning Python, Odoo Framework dan lainnya diatas Sistem Operasi Linux (Ubuntu Server) untuk deployment rack to rack ataupun Ubuntu server dengan manajemen container LXD untuk mesin hypervisor

Tautan teknologi enterprise tersebut di atas dapat diakses dibawah ini 

1. Ubuntu Server http://old-releases.ubuntu.com/releases/19.04/ubuntu-19.04-live-server-amd64.iso

2. LXD, https://linuxcontainers.org/lxd/introduction/

3. Apache Hadoop, https://hadoop.apache.org/

4. Apache Spark, https://spark.apache.org/

5. Apache Parquet, https://parquet.apache.org/

6. Apache Drill, https://drill.apache.org/

7. Odoo framework (opsional), https://www.odoo.com/

8. Python library, khususnya yang berkaitan dengan machine learning, big data (hdfs) dan data presentasi, semua komponen adalah open source dengan lisensi yang berbeda beda.

9. Microsoft Power BI (opsional),  https://powerbi.microsoft.com/en-us/

Kandyta dapat diimplementasi dengan berbagai cara :

1. On Premise rack to rack
2. Container (LXD)
3. VPS (Virtual private server)
4. Google Cloud

Sebagai bahan untuk simulasi, di tutorial ini, akan dipakai cara nomor 2 yaitu menggunakan container LXD, dengan maksud agar mempermudah proses pembangunan Kandyta tanpa mengeluarkan biaya untuk perangkat keras yang mahal sebelum masuk ke metode sesungguhnya yaitu on premise rack to rack/vps/Google Cloud. 



## Installasi LXD

LXD mempermudah anda untuk mencoba sistem tata kelola big data Kandyta anda dapat mencobanya di laptop terlebih dahulu. Kebutuhan umum untuk menjalankan sistem ini adalah, Ubuntu 19.04 Desktop atau Ubuntu 18.04 Server dengan ram minimal 16 GB dan prosesor minimal 8 cores serta dan SSD 512GB , di sini dianggap bahwa sistem operasi Ubuntu server sudah terinstall dengan baik. 

Ikuti petunjuk Installasi LXD di host komputer, terdapat dua ekosistem yaitu host dan container, jika perintah diawali dengan **:host** maka perintah dilaksanakan di dalam Ubuntu server, sedangkan jika diawali dengan **:container**, maka perintah dilakukan di dalam container LXD

**:host**

```
myuser@desktop:~$ sudo snap install lxd
```

Diasumsikan LXD sudah terinstall di host komputer , cek apakah group lxd sudah ada

```
myuser@desktop:~$ groups
```

hasil perintah di atas adalah :

> `myuser adm cdrom sudo dip plugdev lpadmin lxd sambashare`

cek apakah user myuser sudah masuk ke dalam group lxd

```
myuser@desktop:~$ id
```

hasil dari perintah diatas adalah

> `uid=1000(myuser) gid=1000(myuser) groups=1000(myuser),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),119(lpadmin),**130(lxd)**,131(sambashare)`

jika myuser belum masuk ke dalam group lxd, lakukan perintah di bawah ini, untuk beberapa kasus harus dilakukan reboot agar user bisa masuk ke dalam group

```
myuser@desktop:~$ usermod -a -G lxd username-di-host
myuser@desktop:~$ sudo reboot
```

jalankan inisialisasi lxd

```
myuser@desktop:~$ lxd init
```

#### Konfigurasi ip address server LXD

---

Setting dhcp lxd server, pada keaadan default interface adalah **lxdbr0**. pastikan semua container tidak ada yang jalan!

**:host**

```
myuser@desktop:~$ lxc list
myuser@desktop:~$ lxc network edit lxdbr0
```

rubah ipv4.address dan sesuaikan dengan ip yg dkehendaki, tambahkan dhcp entry bila diperlukan, misalnya seperti dibawah ini

```
config:
  ipv4.address: 10.10.10.100/24
  ipv4.nat: "true"
  ipv6.address: fd42:b943:aa56:2293::1/64
  ipv6.nat: "true"
description: ""
name: lxdbr0
type: bridge
used_by:
- /1.0/containers/barebones
  managed: true
  status: Created
  locations:
- none
```

jika diperlukan untuk dhcp bisa tambahkan entry ke section config

```
  ipv4.dhcp: "true"
  ipv4.dhcp.ranges: 10.10.10.105-10.10.10.120
```

Reload lxd

```
myuser@desktop:~$ sudo systemctl reload snap.lxd.daemon.service
```

#### Download dan buat container Barebone-Linux

---

Agar proses cepat, kita akan simpan barebone Ubuntu 16.04 dari images repository ubuntu ke local image sebagai barebone bahan baku

**:host**

```
myuser@desktop:~$ lxc launch ubuntu:16.04 ubuntu
myuser@desktop:~$ lxc stop ubuntu
myuser@desktop:~$ lxc publish ubuntu -- alias barebone
myuser@desktop:~$ lxc image delete ubuntu
```

di tahap ini kita sudah mempunyai image baru bernama barebone

#### Buat Hadoop Container

---

**:host**

```
myuser@desktop:~$ lxc launch barebone hadoop-raw
myuser@desktop:~$ lxc exec hadoop-raw -- /bin/bash
```

#### Buat user penanggung jawab Hadoop

---

**:container**

```
root@hadoop-raw:# useradd -m -s /bin/bash hadoop
root@hadoop-raw:# passwd hadoop 
```

#### Rubah konfigurasi ssh server

---

edit */etc/ssh/ssh_config*

```
Host *
StrictHostKeyChecking no
SendEnv LANG LC_*
HashKnownHosts yes
GSSAPIAuthentication yes
GSSAPIDelegateCredentials no
```

edit */etc/ssh/sshd_config*

```
Port 22
Protocol 2
HostKey /etc/ssh/ssh_host_rsa_key
HostKey /etc/ssh/ssh_host_dsa_key
HostKey /etc/ssh/ssh_host_ecdsa_key
HostKey /etc/ssh/ssh_host_ed25519_key
UsePrivilegeSeparation yes

KeyRegenerationInterval 3600
ServerKeyBits 1024
SyslogFacility AUTH
LogLevel INFO
LoginGraceTime 120
PermitRootLogin prohibit-password
StrictModes yes
RSAAuthentication yes
PubkeyAuthentication yes
IgnoreRhosts yes
RhostsRSAAuthentication no
HostbasedAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
PasswordAuthentication yes
X11Forwarding yes
X11DisplayOffset 10
PrintMotd no
PrintLastLog yes
TCPKeepAlive yes
AcceptEnv LANG LC_*
Subsystem sftp /usr/lib/openssh/sftp-server
UsePAM yes
```

restart ssh server

```
root@hadoop-raw:# /etc/init.d/ssh restart
root@hadoop-raw:# exit
```

Setelah exit, login dengan ssh menggunakan hadoop user, untuk mencari ip address dari hadoop raw ini, di komputer host jalankan

**:host**

```
myuser@desktop:~$ lxc list 
```

login lah dengan user baru

```
myuser@desktop:~$ ssh hadoop@ipaddress-nya-si-hadoop-raw
```

#### Upgrade Container

---

**:container**

```
hadoop@hadoop-raw:~$ sudo apt-get update
hadoop@hadoop-raw:~$ sudo apt-get upgrade
```

#### Sesuaikan Zone Waktu

---

```
hadoop@hadoop-raw:~$ sudo dpk-reconfigure tzdata
```

#### Install Java

---

```
hadoop@hadoop-raw:~$ sudo apt install openjdk-8-jdk
hadoop@hadoop-raw:~$ sudo apt-get install openjdk-8-jre
```

#### Install Hadoop Custom Package

---

download deb package HadoopCustom-2.7.3.deb dan install

```
hadoop@hadoop-raw:~$ cd ~/
hadoop@hadoop-raw:~$ wget https://aihub.id/repository/HadoopCustom-2.7.3.deb
hadoop@hadoop-raw:~$ sudo dpkg -i ~/HadoopCustom-2.7.3.deb
```

Hadoop custom ini akan terinstall di */opt/BigQ/hadoop/*

edit */etc/environment*

```
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/opt/BigQ/hadoop/bin:/opt/BigQ/hadoop/sbin"
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-amd64/jre
export HADOOP_HOME=/opt/BigQ/hadoop
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre
export HADOOP_PREFIX=/opt/BigQ/hadoop
export HADOOP_COMMON_LIB_NATIVE_DIR=/opt/BigQ/hadoop/lib/native
export HADOOP_OPTS="-Djava.library.path=/opt/BigQ/hadoop/lib/native"
```

#### Konfigurasi Hadoop

---

edit */opt/BigQ/hadoop/etc/hadoop/core-site.xml*

```
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>fs.default.name</name>
        <value>hdfs://hadoop-raw:9000</value>
     </property>
     <property>
        <name>hadoop.tmp.dir</name>
        <value>/var/tmp</value>
        <description>A base for other temporary directories.</description>
     </property>
</configuration>
```

create storage folder

```
hadoop@hadoop-raw:~$ sudo mkdir /var/local/hadoop/
hadoop@hadoop-raw:~$ sudo chown hadoop:hadoop /var/local/hadoop
```

edit */opt/BigQ/hadoop/etc/hadoop/hdfs-site.xml*

```
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
            <name>dfs.name.dir</name>
            <value>file:///var/local/hadoop/HDFS/data/namenode</value>
    </property>
    <property>
            <name>dfs.data.dir</name>
            <value>file:///var/local/hadoop/HDFS/data/datanode</value>
    </property>
<property>
        <name>dfs.replication</name>
        <value>1</value>
        <description>Default block replication.</description>
</property>
</configuration>
```

edit */opt/BigQ/hadoop/etc/hadoop/mapred-site.xml*

```
<configuration>
    <property>
            <name>mapreduce.framework.name</name>
            <value>yarn</value>
    </property>
    <property>
            <name>yarn.app.mapreduce.am.env</name>
            <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
    </property>
    <property>
            <name>mapreduce.map.env</name>
            <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
    </property>
    <property>
            <name>mapreduce.reduce.env</name>
            <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
    </property>
<property>
        <name>yarn.app.mapreduce.am.resource.mb</name>
        <value>512</value>
</property>
<property>
        <name>mapreduce.map.memory.mb</name>
        <value>256</value>
</property>
<property>
        <name>mapreduce.reduce.memory.mb</name>
        <value>256</value>
</property>
<property>
        <name>mapreduce.jobtracker.address</name>
        <value>localhost:54311</value>
        <description>MapReduce job tracker runs at this host and port.</description>
</property>
</configuration>
```

format hdfs

```
hadoop@hadoop-raw:~$ hadoop namenode -format
```

buat authorized keys untuk akses ke datanode dari hadoop-raw tanpa login, pada saat proses pembuatan key jangan memasukan pass phrase.

```
hadoop@hadoop-raw:~$ ssh-keygen - t rsa
hadoop@hadoop-raw:~$ cp ~/.ssh/id_rsa.pub ~/.ssh/authorized_keys
```

Clean up history dan lainnya

```
hadoop@hadoop-raw:~$ rm ~/HadoopCustom-2.3.1.deb
hadoop@hadoop-raw:~$ history -c
hadoop@hadoop-raw:~$ exit
```

#### Create Image Template Hadoop dari Container

---

**:host**

```
myuser@desktop:~$ lxc stop hadoop-raw
myuser@desktop:~$ lxc publish hadoop-raw --alias hadoop-image
myuser@desktop:~$ lxc image list
myuser@desktop:~$ lxc delete hadoop-raw
```

Sampai dengan tahap ini kita sudah mempunyai beberapa image yaitu **barebone** dan **hadoop-image**

### Installasi Apache Spark

---

**:host**

```
myuser@desktop:~$ lxc launch hadoop-image spark-raw
myuser@desktop:~$ lxc exec spark-raw -- /bin/bash
```

#### Buat user penanggung jawab Spark

---

**:container**

```
root@spark-raw:# useradd -m -s /bin/bash spark
root@spark-raw:# passwd spark
root@spark-raw:# exit
```

Setelah exit, login dengan ssh menggunakan spark user, untuk mencari ip address dari spark-raw ini, di komputer host jalankan

**:host**

```
myuser@desktop:~$ lxc list
```

login lah dengan user baru

```
myuser@desktop:~$ ssh spark@ipaddress-nya-si-spark-raw
```

#### Install Spark Custom Package

---

**:container**

masih dengan login dengan username spark, download deb package SparkCustom-2.3.1.deb dan install

```
spark@spark-raw:~$ wget http://aihub.id/repository/SparkCustom-2.3.1.deb
spark@spark-raw:~$ sudo dpkg -i ~/SparkCustom-2.3.1.deb
```

Spark custom ini akan terinstall di */opt/BigQ/spark/*

edit */etc/environment*

```
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/opt/BigQ/hadoop/bin:/opt/BigQ/hadoop/sbin:/opt/BigQ/spark/bin:/opt/BigQ/spark/sbin"
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-amd64/jre
export HADOOP_HOME=/opt/BigQ/hadoop
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre
export HADOOP_PREFIX=/opt/BigQ/hadoop
export HADOOP_COMMON_LIB_NATIVE_DIR=/opt/BigQ/hadoop/lib/native
export HADOOP_OPTS="-Djava.library.path=/opt/BigQ/hadoop/lib/native"
export SPARK_HOME=/opt/BigQ/spark 
export PYSPARK_PYTHON=python3.5 
export SPARK_CLASSPATH=/opt/BigQ/spark/jdbc
```

edit */opt/BigQ/spark/spark-env.sh* Kosongkan isian SPARK_MASTER_HOST.

```
#!/usr/bin/env bash
SPARK_MASTER_HOST=
```

Finishing dan clean up

```
spark@spark-raw:~$ sudo rm /home/hadoop/.ssh/authorized_keys
spark@spark-raw:~$ sudo rm /home/hadoop/.ssh/id_rsa
spark@spark-raw:~$ sudo /home/hadoop/.ssh/id_rsa.pub
spark@spark-raw:~$ sudo echo "" > /home/hadoop/.ssh/known_hosts
spark@spark-raw:~$ rm ~/SparkCustom-2.3.1.deb
spark@spark-raw:~$ history -c
spark@spark-raw:~$ exit
```

#### Create Image Template Spark dari Container

---

**:host**

```
myuser@desktop:~$ lxc stop spark-raw
myuser@desktop:~$ lxc publish spark-raw --alias spark-image
myuser@desktop:~$ lxc image list
myuser@desktop:~$ lxc delete spark-raw
```

Sampai dengan tahap ini kita sudah mempunyai beberapa image yaitu **barebone**, **spark-image** dan **hadoop-image**

### Installasi BigQ Manager

---

Front end dari BigQ Manager adalah Odoo Framework 11.0 , untuk membuat container BigQ Manager, yang harus dijadikan bahan baku image adalah spark-image

**:host**

```
myuser@desktop:~$ lxc launch spark-image bigq-raw
myuser@desktop:~$ lxc exec bigq-raw -- /bin/bash
```

**:container**

```
root@bigq-raw:# useradd -m -s /bin/bash masteruser
root@bigq-raw:# passwd spark
root@bigq-raw:# exit
```

**:host**

lihat ip address bigq-raw dengan cara :

```
myuser@desktop:~$ lxc list
```

login dengan ssh ke dalam bigq-raw dengan menggunakan masteruser.

```
myuser@desktop:~$ ssh masteruser@ipaddress-nya-si-bigq-raw
```

di dalam image baru bigq-raw ini terdapat beberapa user hasil dari pembuatan image hadoop-image dan spark image terdahulu, proses pertama adalah rekonfigurasi server.

**:container**

```
masteruser@bigq-raw:-$ sudo chown -R masteruser:masteruser /opt/BigQ/hadoop
masteruser@bigq-raw:-$ sudo chown -R masteruser:masteruser /opt/BigQ/spark
masteruser@bigq-raw:-$ sudo rm /var/tmp/*
masteruser@bigq-raw:-$ sudo rm -r /var/local/hadoop
masteruser@bigq-raw:-$ sudo userdel -r spark
masteruser@bigq-raw:-$ sudo userdel -r hadoop
```

#### Installasi python3 development

---

```
masteruser@bigq-raw:-$ sudo apt-get install python3-pip
masteruser@bigq-raw:-$ sudo apt-get install npm
masteruser@bigq-raw:-$ sudo ln -s /usr/bin/nodejs /usr/bin/node
masteruser@bigq-raw:-$ sudo npm install -g less less-plugin-clean-css
masteruser@bigq-raw:-$ sudo apt-get install node-less
masteruser@bigq-raw:-$ sudo apt-get install postgresql
```

#### Installasi Custom Odoo 11 Server

---

**:container**

```
masteruser@bigq-raw:-$ sudo adduser --system --home=/opt/BigQ/odoo --group odoo
```

Konfigurasi postgresql buat beberapa user postgres yaitu odoo (user odoo), masteruser (user masteruser) dan jdbcmaster (user yang akan bertindak sebagai proxy lintas komunikasi system BigQ Manager dengan framework Odoo 11)

```
masteruser@bigq-raw:-$ sudo su postgres
postgres@bigq-raw:/home/masteruser$ cd
postgres@bigq-raw:~$ create user -s odoo
postgres@bigq-raw:~$ create user -s masteruser
postgres@bigq-raw:~$ create user -s jdbcmaster

postgres@bigq-raw:~$ psql
postgres=# ALTER USER jdbcmaster WITH PASSWORD 'jdbcmaster';
postgres=# \q

postgres@bigq-raw:~$ exit
```

edit */etc/postgresql/9.5/main/pg_hba.conf*

```
local   all             postgres                                peer
local   all             odoo                                    peer
local   all             masteruser                              peer
local   all             jdbcmaster                              md5

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5

# IPv6 local connections:
host    all             all             ::1/128                 md5
```

edit */etc/postgresql/9.5/main/postgresql.conf*

```
listen_address = '*'  menjadi listen_address = 'localhost' (dan buang '#')
```

restart postgresql

```
masteruser@bigq-raw:-$ sudo /etc/init.d/postgresql restart
```

Test proxy connection dengan user jdbcmaster

```
masteruser@bigq-raw:-$ createdb testing
masteruser@bigq-raw:-$ psql testing
testing=# \du
testing=# \q
masteruser@bigq-raw:-$ psql -U jdbcmaster -W
testing=# \q
masteruser@bigq-raw:-$ dropdb testing
```

masih dengan login dengan username masteruser, download deb package OdooCustom-11.deb dan install

```
masteruser@bigq-raw:-$ cd ~/
masteruser@bigq-raw:-$ wget http://aihub.id/repository/OdooCutom-11.deb
masteruser@bigq-raw:-$ sudo dpkg -i ~/OdooCustom-2.3.1.deb
```

Odoo 11 custom ini akan terinstall di */opt/BigQ/odoo/server*, download deb package AdditionalConfig-1.0.deb

```
masteruser@bigq-raw:-$ cd ~/
masteruser@bigq-raw:-$ wget http://aihub.id/repository/AdditionalConfig-1.0.deb
masteruser@bigq-raw:-$ sudo dpkg -i ~/AdditionalConfig-1.0.deb
```

Installasi AdditionalConfig-1.0.deb tersebut akan menghasilkan beberapa direktori di /opt/BigQ yaitu

> /opt/BigQ/payload /opt/BigQ/payload/config /opt/BigQ/payload/data /opt/BigQ/payload/images /opt/BigQ/payload/modules /opt/BigQ/payload/scripts

Install python library

```
masteruser@bigq-raw:-$ sudo -H pip3 install Cython Babel decorator docutils ebaysdk feedparser gevent greenlet html2text Jinja2 lxml Mako MarkupSafe mock num2words ofxparse passlib Pillow psutil psycogreen pydot pyparsing PyPDF2 pyserial python-dateutil python-openid pytz pyusb PyYAML qrcode reportlab requests six suds-jurko vatnumber vobject Werkzeug XlsxWriter xlwt xlrd  psycopg2-binary phonenumbers pandoc gdata
```

Install BigQ Manager Dependencies

```
masteruser@bigq-raw:-$ cd ~/
masteruser@bigq-raw:-$ wget https://aihub.id/repository/pyspark-2.3.1.tar.gz
masteruser@bigq-raw:-$ tar -xvf pyspark-2.3.1.tar.gz
masteruser@bigq-raw:-$ mv pyspark-2.3.1 pyspark
masteruser@bigq-raw:-$ cd pyspark
masteruser@bigq-raw:-$ sudo python3 setup.py install
masteruser@bigq-raw:-$ sudo -H pip3 install python-dotenv crypthography uuid
masteruser@bigq-raw:-$ sudo -H pip3 install pandas==0.24
masteruser@bigq-raw:-$ sudo -H pip3 install pyarrow
```

Test jalankan BigQ Manager

```
masteruser@bigq-raw:-$ /opt/BigQ/odoo/server/odoo-bin
```

**:host**

Di komputer host, download BigQ Manager Data, BigQ Manager Data tersedia dalam beberapa rilis, download sesuai dengan lisensi yang anda punya

1. BigQ_Manager_Standard_V.2.1.zip
2. BigQ_Manager_Government_V.2.1.zip

```
myuser@desktop:-$ cd ~/Download
myuser@desktop:-$ wget https://aihub.id/repository/BigQ_Manager_Standard_V.2.1.zip
```

Cari ip address bigq-raw

```
myuser@desktop:-$ lxc list
```

> Buka browser (chrome atau Firefox), ketikan url http://ip-address-nya-bigq-raw:8069/web/database/manager

Klik button Restore Database, di ops "Choose File" klik dan pilih file BigQ Manager Data yang telah didownload sebelumnya (nomor 1-3 sesuai dengan versi release masing masing). Setelah proses restore selesai, loginlah dengan username 'admin' dan password '1234567' jika semua berjalan dengan baik, logout dari system BigQ Manager.

Fokuskan kembali ke terminal container yang sedang berjalan dan ketikan ctrl+c dua kali

**:container**

Lihat services yang sedang berjalan, dan pastikan hanya port 22 dan 5432 yang masih berjalan

```
masteruser@bigq-raw:-$ netstat -anpt
```

Lakukan pembersihan akhir di container

```
masteruser@bigq-raw:-$ rm ~/OdooCustom-2.3.1.deb
masteruser@bigq-raw:-$ rm ~/AdditionalConfig-1.0.deb
masteruser@bigq-raw:-$ rm -r ~/pyspark
masteruser@bigq-raw:-$ rm pyspark-2.3.1.tar.gz
masteruser@bigq-raw:-$ history -c
masteruser@bigq-raw:-$ exit
```

**:host**

```
myuser@desktop:-$ lxc stop bigq-raw
myuser@desktop:-$ lxc publish bigq-raw --alias BigQ-image
myuser@desktop:-$ lxc delete bigq-raw
```

Sampai dengan tahap ini kita sudah mempunyai beberapa image yaitu **barebone**, **hadoop-image** , **spark-image** dan **BigQ-image**.



### Post Install

---

Seperti diterangkan dalam proses akhir installasi, image yang sudah dibuat adalah :

1. barebone, Image Ubuntu 16.04
2. haddop-image
3. spark-image
4. dan BigQ-image

image barebone cukup disimpan untuk kepentingan lainya dikemudian waktu, hadoop-image, spark-image dan BigQ Image lah yang akan kita pakai.

##### Perencanaan Kapasitas (Capacity Planning)

---

BigQ Image akan bertindak sebagai Command Center, untuk Hadoop kita akan pakai 3 (1 master dan 2 data node), Spark akan kita pakai 3 (1 master dan 2 slaves) dengan konfigurasi host dan static ip-address (sesuaikan kondisi dengan konfigurasi anda sendiri) adalah sebagai berikut :

| Image         | Hostname       | Static Ip    |
| ------------- | -------------- | ------------ |
| BigQ-image    | BigQ-Manager1  | 10.10.10.100 |
| hadoop-image  | hadoop-master1 | 10.10.10.101 |
| hadoop-image  | data-node1     | 10.10.10.102 |
| hadoop-image  | data-node2     | 10.10.10.103 |
| spark-image   | spark-master1  | 10.10.10.104 |
| spark-image   | spark-slave1   | 10.10.10.104 |
| spark-image   | spark-slave2   | 10.10.10.105 |
| Komputer Host | desktop        | 192.168.1.10 |

Kita hanya perlu membuat beberapa container dari image yang sudah kita buat diproses installasi, dimana sumber image nya adalah yaitu BigQ-image, hadoop-image dan spark-image. Pastikan semua container tidak ada yang jalan, untuk memastikannya dapat dilihat dengan perintah :

**:host**

```
myuser@desktop:~$ lxc list
```

Jika ada container yang sedang berjalan, stop container tersebut dengan perintah

```
myuser@desktop:~$  lxc stop <nama-container>
```

##### Pengaturan Container dan IP Address

---

Buat container BigQ-Manager dan set static ip address untuk dirinya

```
myuser@desktop:~$ lxc launch BigQ-image BigQ-Manager1
myuser@desktop:~$ lxc stop BigQ-Manager1
myuser@desktop:~$ lxc network attach lxdbr0 BigQ-Manager1 eth0 eth0
myuser@desktop:~$ config device set BigQ-Manager1 eth0 ipv4.address 10.10.10.100
```

Buat container hadoop-master.

```
myuser@desktop:~$ lxc launch hadoop-image hadoop-master1
myuser@desktop:~$ lxc stop hadoop-master1
myuser@desktop:~$ lxc network attach lxdbr0 hadoop-master1 eth0 eth0
myuser@desktop:~$ config device set hadoop-master1 eth0 ipv4.address 10.10.10.101
```

Buat container data-node1.

```
myuser@desktop:~$ lxc launch hadoop-image data-node1
myuser@desktop:~$ lxc stop data-node1
myuser@desktop:~$ lxc network attach lxdbr0 data-node1 eth0 eth0
myuser@desktop:~$ config device set data-node1 eth0 ipv4.address 10.10.10.102
```

Buat container data-node2.

```
myuser@desktop:~$ lxc launch hadoop-image data-node2
myuser@desktop:~$ lxc stop data-node2
myuser@desktop:~$ lxc network attach lxdbr0 data-node2 eth0 eth0
myuser@desktop:~$ config device set data-node1 eth0 ipv4.address 10.10.10.103
```

Buat container spark-master1

```
myuser@desktop:~$ lxc launch spark-image spark-master1
myuser@desktop:~$ lxc stop spark-master1
myuser@desktop:~$ lxc network attach lxdbr0 spark-master1 eth0 eth0
myuser@desktop:~$ config device set spark-master1 eth0 ipv4.address 10.10.10.104
```

Buat container spark-slave1

```
myuser@desktop:~$ lxc launch spark-image spark-slave1
myuser@desktop:~$ lxc stop spark-slave1
myuser@desktop:~$ lxc network attach lxdbr0 spark-slave1 eth0 eth0
myuser@desktop:~$ config device set spark-slave1 eth0 ipv4.address 10.10.10.105
```

Buat container spark-slave2

```
myuser@desktop:~$ lxc launch spark-image spark-slave2
myuser@desktop:~$ lxc stop spark-slave2
myuser@desktop:~$ lxc network attach lxdbr0 spark-slave2 eth0 eth0
myuser@desktop:~$ config device set spark-slave2 eth0 ipv4.address 10.10.10.106
```

Buat hosts di komputer host, diasumsikan mengedit dengan aplikasi text editor nano :

```
myuser@desktop:~$ sudo nano /etc/hosts
```

Tambahkan semua entry container yang sudah dibuat, dan simpan dengan CTRL-O

```
127.0.0.1           localhost
10.10.10.100        BigQ-Manager1
10.10.10.101        hadoop-master1
10.10.10.102        data-node1
10.10.10.103        data-node2
10.10.10.104        spark-master1
10.10.10.105        spark-slave1
10.10.10.106        spark-slave2
```

artinya, komputer host dapat mengakses semua container berdasarkan host masing masing agar lebih mudah.

##### Template Konfigurasi

---

Agar lebih mudah dalam mereplikasi, buatlah sebuah projek, simpan dalam folder khusus dan buat beberapa konfigurasi dalam folder tersebut, home direktori pengguna, dalam artikel ini adalah /home/myuser

```
myuser@desktop:~$ mkdir /home/myuser/project1
```

buat file core-site.xml, sunting dan simpan

```
myuser@desktop:~$ nano /home/myuser/project1/core-site.xml
```

Sunting seperti di bawah ini :

```
<?xml version="1.0" encoding="UTF-8"?> 
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?> 
<configuration>     
    <property>         
        <name>fs.default.name</name>         
        <value>hdfs://hadoop-master1:9000</value>      
    </property>      
    <property>         
        <name>hadoop.tmp.dir</name>         
        <value>/var/tmp</value>
        <description>A base for other temporary directories.</description>                  </property> 
</configuration>
```

buat file mapred-site.xml, sunting dan simpan.

```
myuser@desktop:~$ nano /home/myuser/project1/mapred-site.xml
```

Sunting seperti di bawah ini :

```
<configuration>
    <property>
            <name>mapreduce.framework.name</name>
            <value>yarn</value>
    </property>
    <property>
            <name>yarn.app.mapreduce.am.env</name>
            <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
    </property>
    <property>
            <name>mapreduce.map.env</name>
            <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
    </property>
    <property>
            <name>mapreduce.reduce.env</name>
            <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
    </property>
<property>
        <name>yarn.app.mapreduce.am.resource.mb</name>
        <value>512</value>
</property>
<property>
        <name>mapreduce.map.memory.mb</name>
        <value>256</value>
</property>
<property>
        <name>mapreduce.reduce.memory.mb</name>
        <value>256</value>
</property>
<property>
        <name>mapreduce.jobtracker.address</name>
        <value>hadoop-master1:54311</value>
        <description>MapReduce job tracker runs at this host and port.</description>
</property>
</configuration>
```

Buat file hadoop-hosts, sunting dan simpan

```
myuser@desktop:~$ nano /home/myuser/project1/hadoop-hosts
```

Sunting seperti di bawah ini :

```
127.0.0.1           localhost
10.10.10.101        hadoop-master1
10.10.10.102        data-node1
10.10.10.103        data-node2

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Biasanya bari ipv6 sudah otomatis diberikan.

Buat file hadoop slaves, sunting dan simpan

```
myuser@desktop:~$ nano /home/myuser/project1/slaves
```

Sunting seperti di bawah ini :

```
10.10.10.102        data-node1
10.10.10.103        data-node2
```

Buat file hadoop workers, sunting dan simpan

```
myuser@desktop:~$ nano /home/myuser/project1/workers
```

Sunting seperti di bawah ini :

```
10.10.10.102        data-node1
10.10.10.103        data-node2
```

Buat file spark-hosts, sunting dan simpan

```
myuser@desktop:~$ nano /home/myuser/project1/spark-hosts
```

Sunting seperti di bawah ini :

```
10.10.10.104        spark-master1
10.10.10.105        spark-slave1
10.10.10.106        spark-slave2

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Buat file spark-env.sh, sunting dan edit

```
#!/usr/bin/env bash
SPARK_MASTER_HOST = spark-master1
```

beri mode eksekusi file spark-env.sh tersebut :

```
myuser@desktop:~$ chmod +x /home/myuser/project1/spark-env.sh
```

dari semua proses pembuatan konfigurasi di atas, jika dilihat secara tree view adalah sebagai berikut :

```
-/home/myuser/project1/
--- core-site.xml
--- mapred-site.xml
--- hadoop-hosts
--- slaves
--- workers
--- spark-hosts
--- spark-env.sh
```

##### Test Running

---

Jalankan secara sequential (berurutan) masing masing container :

```
myuser@desktop:~$ cd /home/myuser/project1
myuser@desktop:~$ lxc start data-node2
myuser@desktop:~$ lxc start data-node1
myuser@desktop:~$ lxc start hadoop-master1
myuser@desktop:~$ lxc start spark-slave2
myuser@desktop:~$ lxc start spark-slave1
myuser@desktop:~$ lxc start spark-master1
myuser@desktop:~$ lxc start BigQ-Manager1
```

Upload file core-site.xml yang telah dibuat di atas :

```
myuser@desktop:~$ lxc file push core-site.xml hadoop-master1/opt/BigQ/hadoop/etc/hadoop/core-site.xml .

myuser@desktop:~$ lxc file push core-site.xml data-node1/opt/BigQ/hadoop/etc/hadoop/core-site.xml .

myuser@desktop:~$ lxc file push core-site.xml data-node2/opt/BigQ/hadoop/etc/hadoop/core-site.xml .
```

perhatikan tanda . (titik) diakhir perintah, tanda . (titik) tersebut merupakan bagian dari perintah

Upload file mapred-site.xml yang telah dibuat di atas :

```
myuser@desktop:~$ lxc file push mapred-site.xml hadoop-master1/opt/BigQ/hadoop/etc/hadoop/mapred-site.xml .

myuser@desktop:~$ lxc file push mapred-site.xml data-node1/opt/BigQ/hadoop/etc/hadoop/mapred-site.xml .

myuser@desktop:~$ lxc file push mapred-site.xml data-node2/opt/BigQ/hadoop/etc/hadoop/mapred-site.xml .
```

Upload file slaves dan workers yang telah dibuat di atas :

```
myuser@desktop:~$ lxc file push slaves hadoop-master1/opt/BigQ/hadoop/etc/hadoop/slaves .
myuser@desktop:~$ lxc file push workers hadoop-master1/opt/BigQ/hadoop/etc/hadoop/workers .
```

Upload file hadoop-hosts yang telah dibuat di atas, sebagai file hosts :

```
myuser@desktop:~$ lxc file push hadoop-hosts hadoop-master1/etc/hosts .
myuser@desktop:~$ lxc file push hadoop-hosts data-node1/etc/hosts .
myuser@desktop:~$ lxc file push hadoop-hosts data-node2/etc/hosts .
```

Upload file spark-hosts yang telah dibuat di atas, sebagai file hosts :

```
myuser@desktop:~$ lxc file push spark-hosts spark-master1/etc/hosts .
myuser@desktop:~$ lxc file push spark-hosts spark-slave1/etc/hosts .
myuser@desktop:~$ lxc file push spark-hosts spark-slave2/etc/hosts .
```

Upload file spark-env.sh yang telah dibuat di atas :

```
myuser@desktop:~$ lxc file push spark-env.sh spark-master1/BigQ/spark/conf/spark-env.sh .
myuser@desktop:~$ lxc file push spark-env.sh spark-slave1/BigQ/spark/conf/spark-env.sh .
myuser@desktop:~$ lxc file push spark-env.sh spark-slave2/BigQ/spark/conf/spark-env.sh .
```

Rubah kepemilikan core-site.xml, mapred-site.xml dan spark-env.sh

```
myuser@desktop:~$ lxc exec hadoop-master1 bash -ilc "/usr/bin/chown <user-hadoop>:<group hadoop> /opt/BigQ/hadoop/etc/hadoop/core-site.xml"

myuser@desktop:~$ lxc exec data-node1 bash -ilc "/usr/bin/chown <user-hadoop>:<group hadoop> /opt/BigQ/hadoop/etc/hadoop/core-site.xml"

myuser@desktop:~$ lxc exec data-node1 bash -ilc "/usr/bin/chown <user-hadoop>:<group hadoop> /opt/BigQ/hadoop/etc/hadoop/core-site.xml"
```

Lakukan perintah di atas untuk mapred-site.xml (hadoop) dan spark-env.sh (spark), adapun lokasi config file hadoop adalah **/opt/BigQ/hadoop/etc/hadoop** sementara config file spark adalah **/opt/BigQ/spark/conf**

##### Running Semua Sistem

---

Jalankan semua service dengan perintah sebagai berikut :

```
myuser@desktop:~$ lxc exec hadoop-master1 -- sudo --login --user hadoop bash -ilc ". /etc/environment && /opt/BigQ/hadoop/sbin/start-dfs.sh"

myuser@desktop:~$ lxc exec hadoop-master1 -- sudo --login --user hadoop bash -ilc ". /etc/environment && /opt/BigQ/hadoop/sbin/start-yarn.sh"

myuser@desktop:~$ lxc exec spark-master1 -- sudo --login --user spark bash -ilc ". /etc/environment && /opt/BigQ/spark/sbin/start-master.sh"

myuser@desktop:~$ lxc exec spark-slave1 -- sudo --login --user spark bash -ilc  ". /etc/environment && /opt/BigQ/spark/sbin/start-slave.sh spark://10.10.10.105:7077"

myuser@desktop:~$ lxc exec spark-slave2 -- sudo --login --user spark bash -ilc  ". /etc/environment && /opt/BigQ/spark/sbin/start-slave.sh spark://10.10.10.106:7077"

myuser@desktop:~$ lxc exec odoo-server -- sudo --login --user masteruser bash -ilc ". /etc/environment && /opt/BigQ/odoo/server/odoo-bin &"
```

Cek ricek apakah hadoop dan spark sudah berjalan dengan baik, pada komputer host, buka browser dan arahkan url ke http://hadoop-master1:9000 serta http://spark-master1:8080 , untuk diingat bahwa layanan (services) hadoop dan spark beserta informasi di url di atas hanya dapat dilihat di host komputer dikarenakan sistem keamanan subnet isolated.

Kandyta BigQ-Manager dilengkapi dengan sistem one touch delivery, artinya semua proses post install dari atas sampai ke bawah ini, cukup dengan pengaturan web interface yang dapat dijalankan dengan mudah.

##### Memberhentikan Layanan

---

Hati hati dalam memberhentikan layanan, terutama di Hadoop, karena terkait dengan data node dan blok blok penyimpanan, jika memang harus, ikuti perintah berurutan sebagai berikut :

```
myuser@desktop:~$ lxc exec hadoop-master1 -- sudo --login --user hadoop bash -ilc ". /etc/environment && /opt/BigQ/hadoop/sbin/stop-yarn.sh"

myuser@desktop:~$ lxc exec hadoop-master1 -- sudo --login --user hadoop bash -ilc ". /etc/environment && /opt/BigQ/hadoop/sbin/stop-dfs.sh"

myuser@desktop:~$ lxc exec spark-slave1 -- sudo --login --user spark bash -ilc ". /etc/environment && /opt/BigQ/spark/sbin/stop-slave.sh"

myuser@desktop:~$ lxc exec spark-slave2 -- sudo --login --user spark bash -ilc ". /etc/environment && /opt/BigQ/spark/sbin/stop-slave.sh"

myuser@desktop:~$ lxc exec spark-master -- sudo --login --user spark bash -ilc ". /etc/environment && /opt/BigQ/spark/sbin/stop-master.sh"

myuser@desktop:~$ lxc stop data-node1
myuser@desktop:~$ lxc stop data-node2
myuser@desktop:~$ lxc stop hadoop-master1
myuser@desktop:~$ lxc stop spark-slave1
myuser@desktop:~$ lxc stop spark-slave2
myuser@desktop:~$ lxc stop BigQ-Manager1
```