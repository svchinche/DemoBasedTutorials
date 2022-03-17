# Linux

Table of contents
=================

<!--ts-->
   * [Unit Files](#unit-files)
   * [tmpfs vs ramfs and swap](#tmpfs-vs-ramfs-and-swap)
   * [Double vs Sigle square bracket for if condition](#double-vs-single-square-bracket-for-if-condition)
   * [cut](#cut)
   * [xargs](#xargs)
   * [sort](#sort)
   * [tr](#tr)
   * [sed](#sed)
   * [awk](#awk)
   * [getops](#getops)
   * [How ssl works](#how-ssl-works)
   * [Selinux](#selinux)
   * [Date and NTP](#date-and-ntp)
   * [tricky interview quetions](#tricky-interview-quetions)
  

<!--te-->

Unit Files
==========

**Use Case:: Script or service should start at the time of machine boot**

We can do this by two ways.
* Using reboot annotation in crontab
* System unit file 

System Unit file
----------------
1. Create a script whch do a certain task, and that task we want it to be in running state after system boot.
```shell
[root@worker-node1 ansible_hosts]# cat  /root/startup_scripts/start_containers.sh 
#!/bin/bash

## Starting sonar cube container , no need to handle this case as since if container is in running mode it wont start 
docker start sonarqube-article

## starting nexus repo service is not required since it is enabled as part runlevels

## Starting elk containers 
cd /root/docker_projects/elk_project/docker-elk ; 
docker-compose up -d

## Starting ansible hosts
cd /root/docker_projects/ansible_hosts/ ; docker-compose start


###starting filebeat services
/etc/init.d/filebeat start
```

2. Create unit file for the resource/service that you want to control using systemctl command

```linux

# /etc/systemd/system/start_container.service
[Unit]
Description=Description for sample script goes here
After=docker.service

[Service]
Type=simple
ExecStart=/root/startup_scripts/start_containers.sh
TimeoutStartSec=0

[Install]
WantedBy=default.target
```
**Note:** File can be viewed directly from systemctl cat command
```
[root@worker-node1 ansible_hosts]# systemctl cat start_container.service
```
3. Reload the system daemon to get the new changes for systemctl command
systemctl daemon-reload

4. Enable the service
```
[root@worker-node1 ansible_hosts]# systemctl enable start_container.service
```

5. Check the logs of service 
```
[root@worker-node1 ansible_hosts]# journalctl -u start_container.service
-- Logs begin at Tue 2019-09-03 09:48:38 IST, end at Tue 2019-09-03 15:29:28 IST. --
Sep 03 09:50:08 worker-node1 systemd[1]: Started Description for sample script goes here.
Sep 03 09:50:15 worker-node1 start_containers.sh[5060]: sonarqube-article
Sep 03 09:50:17 worker-node1 start_containers.sh[5060]: Starting docker-elk_elasticsearch_1 ...
Sep 03 09:50:22 worker-node1 start_containers.sh[5060]: [97B blob data]
Sep 03 09:50:22 worker-node1 start_containers.sh[5060]: Starting docker-elk_logstash_1      ...
Sep 03 09:50:40 worker-node1 start_containers.sh[5060]: [134B blob data]
Sep 03 09:50:40 worker-node1 start_containers.sh[5060]: Starting host2 ...
Sep 03 09:50:40 worker-node1 start_containers.sh[5060]: Starting host3 ...
Sep 03 09:51:03 worker-node1 start_containers.sh[5060]: [155B blob data]
[root@worker-node1 ansible_hosts]#
```

tmpfs vs ramfs and swap
=======================

Ram based storage can be created using
----------------------------------------
- tmpfs [more recent ramfs]  </br>
- ramfs [it will use memory storage until systtem run out of of the ram- no limit on size ]  </br>
It is intended to appear mounted fs, but the data is stored at volatile memory  </br>
The major benefit to memory based file systems is that they are very fast – 10s of times faster than modern SSDs.  </br>
Read and write performance is massively increased for all workload types.  </br>

tmpfs vs swap
-------------
tmpfs may use swap fs space.  If your system runs out of physical RAM, files in your tmpfs partitions may be written to disk based SWAP partitions and  </br>
will have to be read from disk when the file is next accessed

SWAP
----
Linux kernel uses RAM to store temparary info.  </br>
useful to extend the ram, when ram space is exhausted and process have to be continued.  </br>
Process which are waiting for other resource since too much time will get moved to swap.  </br>
Zombie processes get moved into the swap.  </br>

How to create swap
------------------
* Create file using fallocate or dd command
```
fallocate -l 1G /swapfile
dd if=/dev/zero of=/swapfile bs=1024 count=1048576
```

* assign file system to that file -- setup swap on file
```
mkswap /swapfile
```
* add this file to swap space-- enabling
```
swapon /swapfile
```
* make permanent entry in etc fstab conf file

How to adjust the swappiness value
---------------------------------
Swappiness is a Linux kernel property that defines how often the system will use the swap space. </br>
Swappiness can have a value between 0 and 100. A low value will make the kernel to try to avoid 
swapping whenever possible while a higher value will make the kernel to use the swap space more aggressively.
The default swappiness value is 60. You can check the current swappiness value by typing the following command:</br>
```
[root@mum00aqm]# cat /proc/sys/vm/swappiness
30
```

Purpose of tmpfs?
-----------------
This memory is generally used by Linux to cache recently accessed files so that the next time they are requested then can be fetched from RAM very quickly

what is tmpfs? (initially it was RAM disk - virtual disk)
-------------
- temparary file system data is stored on volatile memory rather than physical memory.

why swap is used even if RAM is free ?
--------------------------
Inactive memory pages(that are very seldom use)will get swapped into swap memory. 
Swap space usage becomes an issue only when there is no RAM available -  Since kernel is force to send memory pages to swap and back to RAM.

what is buff/cache ?
-------------------
```
free  -m
              total        used        free      shared  buff/cache   available
Mem:          29172       15139         655          47       13377       12817
Swap:         21999        3870       18129
```

* buffers - Memory used by kernel buffers (Buffers in /proc/meminfo)
buffers refers to data that is being written -- that memory cannot be reclaimed until the write is complete.

* cache -Memory used by the page cache and slabs (Cached and SReclaimable in /proc/meminfo)
refers to data that has been read -- it is kept around in case it needs to be read again, but can be immediately reclaimed since it can always be re-read from disk.


Difference of free vs available ?
----------------------------------
* free - Free / Unused memory.
* available - An estimate of the amount of memory that is available for starting new applications, without swapping

Double vs Single square bracket for if condition
=========================

```linux
[root@mum00aqm test]# [[ $name =~ "suyo" ]] && echo "hi"
hi
[root@mum00aqm test]# [[ $name == "suyo" ]] && echo "hi"
[root@mum00aqm test]# [[ $name == "suyog" ]] && echo "hi"
hi
```
In above command, you can see == is for strict comparsion while ~ for patter matching, one more benefit of using double square bracket is its compact way to write if conditon, we can also use sigle square bracket

Difference between [ and [[ is, [ is command test command and command completion should be marked as  ] </b>
benefit of using [[ is, its not a command but its syntax and all these case are handled like unary operator expected mentioned in below command

both command are same, but whenever there whenever there is else condition is required then we need to use second option 
```
[[ $name = "suyog" ]] && echo "match" 
if [[ $name = "suyog" ]]; then echo "match"; fi
```
```linux
[root@mum00aqm test]# test 4 =5
-bash: test: 4: unary operator expected
```

cut
===
used for removing or replacing sections
```
[root@mum00aqm ~]# echo -e "Hi \n this is suyog, \n and my mob no is 9822622279" | cut -d' ' -f1
Hi

[root@mum00aqm ~]# echo -e "Hi \n this is suyog, \n and my mob no is 9822622279" | cut -d' ' -f1-4 --output-delimiter='%'
Hi%
%this%is%suyog,
%and%my%mob
```
xargs
=====

**Note::** xargs command in unix will not work as expected, if any of file name contains space or new line in it.</br>
to avoid this problem we use find -print0 to produce null separated file name and xargs-0 to handle null separated items

```linux
find /tmp -name "*.tmp" -print0 | xargs -0 rm
```




How to avoid "Argument list too long" error
------------------------------------------
xargs in unix or Linux was initially use to avoid "Argument list too long" errors and by using xargs you send sub-list to any command which is shorter than "ARG_MAX" and that's how xargs avoid "Argument list too long" error. You can see current value of "ARG_MAX" by using getconf ARG_MAX. Normally xargs own limit is much smaller than what modern system can allow, default is 4096. You can override xargs sub list limit by using "-s" command line option.

find –exec vs find + xargs
--------------------------
xargs with find command is much faster than using -exec on find. since -exec runs for each file while xargs operates on sub-list level. to give an example if you need to change permission of 10000 files xargs with find will be almost 10K time faster than find with -exec because xargs change permission of all file at once.


```linux
getconf -a . | grep ARG
ARG_MAX                            2097152
NL_ARGMAX                          4096
_POSIX_ARG_MAX                     2097152
LFS64_CFLAGS                       -D_LARGEFILE64_SOURCE
LFS64_LINTFLAGS                    -D_LARGEFILE64_SOURCE
XBS5_ILP32_OFFBIG_CFLAGS           -m32 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64
POSIX_V6_ILP32_OFFBIG_CFLAGS       -m32 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64
POSIX_V7_ILP32_OFFBIG_CFLAGS       -m32 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64
```

tr
==
We use this command to translate and delete characters </br>
Here, you can see it translate character by character </br>
s flag is for squeese repitating character </br>
```linux
[root@k8s-master ~]# echo "{Hi i am suyog::}" | tr '{}:' '()-'
(Hi i am suyog--)

echo " Hi i am suyog" | tr '[:lower:]' '[:upper:]'
 HI I AM SUYOG

 echo " Hi i am suyog" | sed 's/suyog/sachin chinche/g'
 Hi i am sachin chinche
 
[root@k8s-master ~]# echo " Hi i am suyog" | tr 'suyog' 'sachin chinche'
 Hi i am sachi


echo "Hi i am   suyog     chinche" | tr -s '[:space:]' ' '
Hi i am suyog chinche 
```

sort
=====
```linux
[root@mum00aqm writeable]# cat emp.csv
3,amit,15000,500
2,suhas,12000,340
5,shubham,2111,450
17,Shantanu,45672,345
1,suyog,20000,1000
4,Suraj,21000,1100
12,Sanchit,22000,1150
7,Sachin,27000,1300

[root@mum00aqm writeable]# sed 's/,/ /g' emp.csv | sort -k4n
2 suhas 12000 340
17 Shantanu 45672 345
5 shubham 2111 450
3 amit 15000 500
1 suyog 20000 1000
4 Suraj 21000 1100
12 Sanchit 22000 1150
7 Sachin 27000 1300
```
**NOTE:** by default sort uses space as delimitor, we can specify different field sperator or delimeter using -t option as below

```linux
[root@k8s-workernode shell_learning]# cat emp.csv | sort -t, -k4n
3,amit,15000,500
1,suyog,20000,1000
4,Suraj,21000,1100
12,Sanchit,22000,1150
7,Sachin,27000,1300


[root@k8s-workernode shell_learning]# cat emp.csv | sort --field-separator=, -k4n
3,amit,15000,500
1,suyog,20000,1000
4,Suraj,21000,1100
12,Sanchit,22000,1150
7,Sachin,27000,1300
```
sed
===

Substitute/replace ccoms with bcoms
-----------------------------------
```
[root@mum00aqm ~]# sed 's/ccoms/bcoms/g' ext_ccoms.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: bcoms
  namespace: bcoms
spec:
  ports:
  - port: 8111
    targetPort: 8111
    protocol: TCP
  selector:
    app: proxy-ms
  externalIPs:
  - 10.180.86.187
 ```
 Delete the line which matches the pattern
 -----------------------------------------
```shell
[root@mum00aqm ~]# sed '/ccoms/d' ext_ccoms.yaml
---
apiVersion: v1
kind: Service
metadata:
spec:
  ports:
  - port: 8111
    targetPort: 8111
    protocol: TCP
  selector:
    app: proxy-ms
  externalIPs:
  - 10.180.86.187
```
Special characters with sed works very well
--------------------------------
```shell
[root@mum00aqm ~]# sed 's/- port: 8111/- port: 8888/g' ext_ccoms.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: ccoms
  namespace: ccoms
spec:
  ports:
  - port: 8888
    targetPort: 8111
    protocol: TCP
  selector:
    app: proxy-ms
  externalIPs:
  - 10.180.86.187
```
Special character with single quote
-----------------------------------
```shell
[root@mum00aqm ~]# cat ext_ccoms.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: 'ccoms'
  namespace: ccoms
spec:
  ports:
  - port: 8111
    targetPort: 8111
    protocol: TCP
  selector:
    app: proxy-ms
  externalIPs:
  - 10.180.86.187


[root@mum00aqm ~]# sed 's/name: '\''ccoms'\''/name: '\''bcoms'\''/g' ext_ccoms.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: 'bcoms'
  namespace: ccoms
spec:
  ports:
  - port: 8111
    targetPort: 8111
    protocol: TCP
  selector:
    app: proxy-ms
  externalIPs:
  - 10.180.86.187
```

**Some more deletion examples
**For deleting a specific line
```
sed -i '5d' file.txt
```
**Deleting line no ranges from x to y
```
sed -i '2,5d' file.txt
```
**Delete blank line in file
```
sed -i '^$/d' file.txt
```
**Delete the blank line or starting with 
```
sed-i '^$/d;^#/d' file.txt
```
awk
===

One of the best benefit of using awk that i found is, we can do basic calculation in command itself. See the example below
```
[root@mum00aqm ~]# echo "Do the sum of 4 and 5" | awk '{print $5 + $7}'
9
```

This command is similar to cut command, but awk suppresses leading whitespaces in command standard output, this is why we use awk the most

**Note:** We always keep action in curly bracket and before that we use search criteria
```
awk options 'selection _criteria {action }' file.txt
```

**Some builtin variables that we use frequently**
- NF - Last field in a line
- NR - Keep the count of number of input records, useful when you want to list file with line no
```
[root@mum00aqm ~]# echo "Do the sum of 4 and 5" | awk '{print $5 $NF}'
45
```

Getops
=======
```
while getopts 'srd:f:' c
do
  case $c in
    s) ACTION=SAVE ;;
    r) ACTION=RESTORE ;;
    d) DB_DUMP=$OPTARG ;;
    f) TARBALL=$OPTARG ;;
  esac
done
```
Similarly, if you don't pass at least one of -d and -f, then nothing will happen at all. </br>

How SSL Works
=============

**What is trusted and self signed certificate**

Self Signed Certificate
-----------------------
openssl req -newkey rsa:2048 -nodes -keyout domain.key-x509 -days 365 -out domain.crt

Trusted Certificate
-----------------
Raise a csr 

openssl req  -new -newkey rsa:2048 -nodes -keyout privatekey.key –out certificatesigningrequest.csr



How to check validity of certificate?
------------------------------------
```
PROTOCOLS=(ssl2 ssl3 tls1  tls1_1  tls1_2)
CIPHERS=(RC4 DES DES:3DES)

echo | openssl s_client  -connect <hostname>:<port no> -tls1_2  2>/dev/null | grep "public key is" | awk '{print $5}'
```

How to check cipher and protocol information?
------------------------------------------
```
echo | openssl s_client  -connect <hostname>:<port no> -"<protocol>" 2>/dev/null | grep -i supported

echo|  openssl s_client  -connect <hostname>:<port no>  -cipher <cipher> 2>/dev/null | grep -i supported
```


| Command purpose                                                                                                | Command                                                                                                                                                                                                                                                                                                                                                    |
|----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Step 1 \- Create private key and a private key store for aforementioned created key \(\.\.identity\.jck file\) | keytool \-genkey \-alias \`hostname \-f` \-keyalg RSA \-keysize 2048 \-sigalg SHA256withRSA \-validity 1095 \-keypass "GkTjn03zfpJ8Weq9" \-storetype jceks \-keystore `hostname \-f`\_identity\.jck \-storepass "GkTjn03zfpJ8Weq9"   \-dname "CN=`hostname \-f`, OU=Oracle Cloud for Industry, O=Oracle Corporation, L=Redwood Shores, S=California, C=US" |
| Step 2 \- Create public key certificate \(\.cer file\)                                                         | keytool \-export \-alias \`hostname \-f` \-file `hostname \-f`\.cer \-keystore `hostname \-f`\_identity\.jck \-storetype jceks \-storepass "GkTjn03zfpJ8Weq9"                                                                                                                                                                                              |
| Step 3 \- Import the created public key certificate into the trust store \(\.\.trust\.jck file\)               | keytool \-import \-alias \`hostname \-f` \-file `hostname \-f`\.cer \-keystore `hostname \-f`\_trust\.jck \-storetype jceks \-storepass "GkTjn03zfpJ8Weq9" \-noprompt                                                                                                                                                                                      |
| List all private \(identity\) keys in private \(identity\) key store                                           | keytool \-list \-keystore \`hostname \-f`\_identity\.jck \-storetype jceks \-storepass "GkTjn03zfpJ8Weq9"                                                                                                                                                                                                                                                  |
| List all public \(trust\) keys in public \(trust\) key store                                                   | keytool \-list \-keystore \`hostname \-f`\_trust\.jck \-storetype jceks \-storepass "GkTjn03zfpJ8Weq9"                                                                                                                                                                                                                                                     |




```
```

SELinux
======== 
Policy core component of selinux (Set of rules)
* Users - Policy defines users access to roles (users -> roles) 
* roles - and roles access to domain (roles -> domain)
* domains - and domain access a file

Commands to manage policies
-------

* List and manage all selinux modules
```
semodule -l
semanage boolean -l
getsebool <name of boolean>
setsebool <name of boolean> true/false

```

Date and NTP
============

Date
----
```
[root@mum00cuc ~]# date +%Y
2020
[root@mum00cuc ~]# date +%Y-%d
2020-18
[root@mum00cuc ~]# date +%Y-%d-%m
2020-18-01
```

NTP
----
Just start ntpdate service, it will tell you when server is synched with NTP server
```
[root@mum00cuc ~]# service ntpdate status
Redirecting to /bin/systemctl status  ntpdate.service
● ntpdate.service - Set time via NTP
   Loaded: loaded (/usr/lib/systemd/system/ntpdate.service; disabled; vendor preset: disabled)
   Active: active (exited) since Sat 2020-01-18 08:29:08 EST; 1h 8min ago
  Process: 20771 ExecStart=/usr/libexec/ntpdate-wrapper (code=exited, status=0/SUCCESS)
 Main PID: 20771 (code=exited, status=0/SUCCESS)

Jan 17 13:51:32 mum00cuc.in.oracle.com systemd[1]: Starting Set time via NTP...
Jan 18 08:29:08 mum00cuc.in.oracle.com systemd[1]: Started Set time via NTP.
You have new mail in /var/spool/mail/root
```

Tricky interview quetions
=========================

Command to search content in file, if matches it should show next 3 lines
--------------------------------------------------------------------------
```shell
[root@mum00aqm ~]# grep -A 3 '8111' ext_ccoms.yaml
  - port: 8111
    targetPort: 8111
    protocol: TCP
  selector:
    app: proxy-ms
```
*-A* : means after search context

How to find count of files and directories from current directory ?
-----------------------------------------------------
```shell
[root@k8s-workernode shell_learning]# find . -type d | expr $(wc -l) - 1
3
[root@k8s-workernode shell_learning]# find . -type f | expr $(wc -l) - 1
6
```

Count the occurences of special character in file ?
---------------------------------------------------
```
[root@mum00aqm ~]# cat test.date
*%$#%%$#%$#$@#$@#
[root@mum00aqm ~]# grep  -o "#" test.date | wc -l
5
```

Move all .txt files with .txt1, if some files already moved then command should ignore that since they are already moved
-----------------------------------------------------------------------------------------------

for eg. in below  xyz txt1 file already moved, command should ignore that
```
[root@mum00aqm test]# ll
total 0
-rw-r--r-- 1 root root 0 Nov 19 17:46 abc.txt
-rw-r--r-- 1 root root 0 Nov 19 17:46 cde.txt
-rw-r--r-- 1 root root 0 Nov 19 17:46 fgh.txt
-rw-r--r-- 1 root root 0 Nov 19 17:46 ghj.txt
-rw-r--r-- 1 root root 0 Nov 19 17:48 xyz.txt1
```

**Solution::**
```
[root@mum00aqm test]# for filename in $( ls *.txt |sed 's/.txt1$//g');do  mv $filename ${filename}1 ; done
[root@mum00aqm test]# ll
total 0
-rw-r--r-- 1 root root 0 Nov 19 17:46 abc.txt1
-rw-r--r-- 1 root root 0 Nov 19 17:46 cde.txt1
-rw-r--r-- 1 root root 0 Nov 19 17:46 fgh.txt1
-rw-r--r-- 1 root root 0 Nov 19 17:46 ghj.txt1
-rw-r--r-- 1 root root 0 Nov 19 17:48 xyz.txt1
```

How to change ownership of file created by one NIS user to other user in /tmp directory?(ex. /tmp/t1 is file created by abc nid user and xyz user has to login into the system and need to change its permission
-----------------------------------------------------------------------
Conditions are
--------------
* he dont need to use sudo or no need to swith to other user and dont change permission of /tmp to 777 )
Answer: 

tempfs by default permission is 1777
```linux
[root@mum00aqm ~]# getfacl /tmp
getfacl: Removing leading '/' from absolute path names
# file: tmp
# owner: root
# group: root
# flags: --t
user::rwx
group::rwx
other::rwx

touch /tmp/svchinch

setfacl -m u:anjane:rw /tmp/svchinch

getfacl /tmp/svchinch
getfacl: Removing leading '/' from absolute path names
# file: tmp/svchinch
# owner: svchinch
# group: dba
user::rw-
user:anjane:rw-
group::r--
mask::rw-
other::r--
```

what is the difference between $* vs $@ in shell ?
-------------------------------------------------
 the values of $@ and $* are same.
However, the values of "$@" and "$*" are different (note double quotes).
```
$@ expanded as "$1" "$2" "$3" ... "$n"
$* expanded as "$1IFS$2IFS$3IFS...$n", "$*" is one long string and $IFS act as an separator or token delimiters.

[root@mum00ban ~]# cat data.sh
#!/bin/bash
IFS='|'
echo "--------------------------------"
echo "--------------------------------"
echo "${*}"
echo "--------------------------------"
echo "${@}"
echo "--------------------------------"
echo $*
echo "--------------------------------"
echo $@
echo "--------------------------------"
echo "--------------------------------"


sh data.sh 1 2 3 4 5 6 7 8 9 10 12 34 767  Suyog Suraj 12 3483 54893 adsjkdkaj asdkjlfaslfj  qwhjwhkjwkef
--------------------------------
--------------------------------
1|2|3|4|5|6|7|8|9|10|12|34|767|Suyog|Suraj|12|3483|54893|adsjkdkaj|asdkjlfaslfj|qwhjwhkjwkef
--------------------------------
1 2 3 4 5 6 7 8 9 10 12 34 767 Suyog Suraj 12 3483 54893 adsjkdkaj asdkjlfaslfj qwhjwhkjwkef
--------------------------------
1 2 3 4 5 6 7 8 9 10 12 34 767 Suyog Suraj 12 3483 54893 adsjkdkaj asdkjlfaslfj qwhjwhkjwkef
--------------------------------
1 2 3 4 5 6 7 8 9 10 12 34 767 Suyog Suraj 12 3483 54893 adsjkdkaj asdkjlfaslfj qwhjwhkjwkef
--------------------------------
--------------------------------

```

How to find filename starting with suyog and ending with suyog and between we can have alphanumeric with lowercaase
----------------------------------------------------------------------------

```
 find . -type f -regex "\.\/suyog.[a-z1-9]*suyog"
```

How to get filename or directory name in column without using pipe or awk ?
--------------------------------------------------------------------
```
[root@mum00ban ~]# ls -1
anaconda-ks.cfg
chef-15.2.26-1.el7.x86_64.rpm
chef-repo
chef-repo.zip
data
data.sh
pre-req.sh
R6
```

How can we run grep command without standard output? (condition - Dont redirect the logs in dev null )
------------------------------------------------------

```
[root@mum00ban ~]# grep -inr "Jan" data
1:Wed Jan 15 13:43:15 UTC 2020
2:Wed Jan 15 13:43:35 UTC 2020
[root@mum00ban ~]# grep -inrq "Jan" data

[root@mum00ban ~]# grep -inrq "Jan" data;echo $?
0

```

Special variables in shell
--------------------------
```shell
$1, $2, $3, ... are the positional parameters.
"$@" is an array-like construct of all positional parameters, {$1, $2, $3 ...}.
"$*" is the IFS expansion of all positional parameters, $1 $2 $3 ....
$# is the number of positional parameters.
$- current options set for the shell.
$$ pid of the current shell (not subshell).
$_ most recent parameter (or the abs path of the command to start the current shell immediately after startup).
$IFS is the (input) field separator.
$? is the most recent foreground pipeline exit status.
$! is the PID of the most recent background command.
$0 is the name of the shell or shell script.
```

What is Subshell
-----------------
Whenever you run a shell script, it creates a new process called subshell and your script will get executed using a subshell.</br>
A Subshell can be used to do parallel processing.</br>
If you start another shell on top of your current shell, it can be referred to as a subshell. Type the following command to see subshell value:</br>


What is sh and bash
-------------------
Shell" is a program, which facilitates the interaction between the user and operating system (kernel) </br>
Bash (bash) is one of many available (yet the most commonly used) Unix shells.  </br>
Bash stands for "Bourne Again SHell", and is a replacement/improvement of the original Bourne shell (sh). </br>


Explain about "s" permission bit in a file?
----------------------------------------------

"s" bit is called "set user id" (SUID) bit.</br>
"s" bit on a file causes the process to have the privileges of the owner of the file during the instance of the program.</br>
For example, executing "passwd" command to change current password causes the user to writes its new password to shadow file even though it has "root" as its owner.

The dot command allows you to modify current shell variables.
-----------------------------------------------------------

exec command to avoid subshell
------------------------------
