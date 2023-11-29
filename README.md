# Oracle APEX Container

[[_TOC_]]

## Description

This project contains the steps to create a custom Oracle APEX container image.
The image is built from the [Oracle Container Registry](https://container-registry.oracle.com/) `free:latest` base image with custom configuration to add APEX and ORDS software.
The built container will be used to store and run Oracle APEX applications.

Download the image from Docker Hub [https://hub.docker.com/r/dbeniteza/oracle-apex-ords](https://hub.docker.com/r/dbeniteza/oracle-apex-ords)


## Container deployment

### Base Image
[Oracle Database Free](https://container-registry.oracle.com/ords/ocr/ba/database/free) 

| Tag | OS/Architecture| Size | Pull Command | Image ID |
| ------ | ------ | ------ | ------ | ------ |
| latest | linux/amd64 | 3 GB | docker pull container-registry.oracle.com/database/free:latest | 39cabc8e6db0 |
| 23.3.0.0 | linux/amd64 | 3 GB | docker pull container-registry.oracle.com/database/free:latest | 39cabc8e6db0 |

### Volumes

- `apex-oradata`: Stores Oracle database data
- `apex-ords-config`: Stores ORDS configuration files
- `apex-sw`: Stores APEX and ORDS installation files

### Ports

- 1521: default Oracle Database listener port. It will be mapped to 8521 host port.
- 8080: default ORDS port. It will be mapped to 8023 host port.

### Software Download

- [`APEX latest version`](https://download.oracle.com/otn_software/apex/apex-latest.zip)
- [`ORDS latest version`](https://download.oracle.com/otn_software/java/ords/ords-latest.zip)


## Build container

### Requirements

1. Verify volumes
```
docker volume ls
DRIVER    VOLUME NAME
local     apex-oradata
local     apex-ords-config
local     apex-sw
```

2. Login into Oracle Container Registry using your Oracle account
```
docker login container-registry.oracle.com
Username: your_email@domain.com
Password:
Login Succeeded
```

3. Pull Oracle Database Free image
```
docker pull container-registry.oracle.com/database/free
Using default tag: latest
latest: Pulling from database/free
089fdfcd47b7: Pull complete
43c899d88edc: Pull complete
47aa6f1886a1: Pull complete
f8d07bb55995: Pull complete
c31c8c658c1e: Pull complete
b7d28faa08b4: Pull complete
1d0d5c628f6f: Pull complete
db82a695dad3: Pull complete
25a185515793: Pull complete
Digest: sha256:5ac0efa9896962f6e0e91c54e23c03ae8f140cf6ed43ca09ef4354268a942882
Status: Downloaded newer image for container-registry.oracle.com/database/free:latest
container-registry.oracle.com/database/free:latest
```

4. Verify pulled image
```
# docker images
REPOSITORY                                    TAG       IMAGE ID       CREATED        SIZE
container-registry.oracle.com/database/free   latest    39cabc8e6db0   2 months ago   9.16GB
```

### Create base container

1. Create container
```
docker run -d --name apex-db --restart always -p 8521:1521 -p 8023:8080 -v apex-oradata:/opt/oracle/oradata -v apex-ords-config:/etc/ords/config -v apex-sw:/opt/sw  container-registry.oracle.com/database/free:latest
```

2. Login to container and verify Oracle database is running
```
docker exec -it apex-db bash

bash-4.4$ netstat -tapln | grep tnslsnr
tcp        0      0 0.0.0.0:1521            0.0.0.0:*               LISTEN      24/tnslsnr
tcp        0      0 172.17.0.2:1521         172.17.0.2:37066        ESTABLISHED 24/tnslsnr
```

3. Change the default password for SYS User. When the container is started, a random password is used for the SYS, SYSTEM and PDBADMIN users. This is termed as the default password. To change the password for these accounts, run the setPassword.sh script that is found in the container.
```
bash-4.4$ ./setPassword.sh Welcome1##
The Oracle base remains unchanged with value /opt/oracle

SQL*Plus: Release 23.0.0.0.0 - Production on Tue Nov 14 08:24:41 2023
Version 23.3.0.23.09

Copyright (c) 1982, 2023, Oracle.  All rights reserved.


Connected to:
Oracle Database 23c Free Release 23.0.0.0.0 - Develop, Learn, and Run for Free
Version 23.3.0.23.09

SQL>
User altered.

SQL>
User altered.

SQL>
Session altered.

SQL>
User altered.

SQL> Disconnected from Oracle Database 23c Free Release 23.0.0.0.0 - Develop, Learn, and Run for Free
Version 23.3.0.23.09
```

### Install Java

ORDS requires Java 11 and above to run. Configure yum repository for the container (Oracle Linux) and install OpenJDK 17.

```
bash-4.4$ cat /etc/oracle-release
Oracle Linux Server release 8.8

bash-4.4$ su -
[root@bffe2831ca7a ~]# yum-config-manager --add-repo=http://yum.oracle.com/repo/OracleLinux/OL8/oracle/software/
x86_64
Adding repo from: http://yum.oracle.com/repo/OracleLinux/OL8/oracle/software/x86_64

[root@bffe2831ca7a ~]# yum install java-17-openjdk
Oracle Linux 8 BaseOS Latest (x86_64)                                            15 kB/s | 3.6 kB     00:00
Oracle Linux 8 BaseOS Latest (x86_64)                                            19 MB/s |  66 MB     00:03
Oracle Linux 8 Application Stream (x86_64)                                      105 kB/s | 3.9 kB     00:00
Oracle Linux 8 Application Stream (x86_64)                                       22 MB/s |  53 MB     00:02
Oracle Linux 8 Development Packages (x86_64)                                     13 kB/s | 3.3 kB     00:00
Oracle Linux 8 Development Packages (x86_64)                                     22 MB/s | 109 MB     00:04
created by dnf config-manager from http://yum.oracle.com/repo/OracleLinux/OL8/o  34 kB/s | 105 kB     00:03
Last metadata expiration check: 0:00:01 ago on Mon Nov 27 11:18:21 2023.
Dependencies resolved.
[...]
Complete!

bash-4.4$ java -version
openjdk version "17.0.9" 2023-10-17 LTS
OpenJDK Runtime Environment (Red_Hat-17.0.9.0.9-2.0.1) (build 17.0.9+9-LTS)
OpenJDK 64-Bit Server VM (Red_Hat-17.0.9.0.9-2.0.1) (build 17.0.9+9-LTS, mixed mode, sharing)
```

### Install Oracle APEX

1. Download APEX software
```
bash-4.4$ curl -o apex-latest.zip https://download.oracle.com/otn_software/apex/apex-latest.zip
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  244M  100  244M    0     0  27.4M      0  0:00:08  0:00:08 --:--:-- 29.7M

bash-4.4$ su -
[root@bffe2831ca7a ~]# chown oracle:oinstall /opt/sw
[root@bffe2831ca7a ~]# exit
bash-4.4$ unzip apex-latest.zip -d /opt/sw/
Archive:  apex-latest.zip
  inflating: META-INF/MANIFEST.MF
[...]
bash-4.4$ rm apex-latest.zip
```

2. Run `apexins.sql` script
```
bash-4.4$ cd /opt/sw/apex/
bash-4.4$ sqlplus / as sysdba

SQL*Plus: Release 23.0.0.0.0 - Production on Tue Nov 14 08:15:48 2023
Version 23.3.0.23.09

Copyright (c) 1982, 2023, Oracle.  All rights reserved.


Connected to:
Oracle Database 23c Free Release 23.0.0.0.0 - Develop, Learn, and Run for Free
Version 23.3.0.23.09

SQL> show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 FREEPDB1                       READ WRITE NO

SQL> ALTER SESSION SET CONTAINER = FREEPDB1;
SQL> @apexins.sql SYSAUX SYSAUX TEMP /i/
[..]

timing for: Complete Installation
Elapsed:    8.65
```

3. Unlock `APEX_PUBLIC_USER` user account and set a password (Welcome1##)
```
SQL> select username, account_status from dba_users where username like 'APEX%';

USERNAME
--------------------------------------------------------------------------------
ACCOUNT_STATUS
--------------------------------
APEX_PUBLIC_USER
LOCKED

APEX_230200
LOCKED

SQL> ALTER USER APEX_PUBLIC_USER ACCOUNT UNLOCK;

User altered.

SQL> ALTER USER APEX_PUBLIC_USER IDENTIFIED BY Welcome1##;

User altered.
```

4. Set APEX `ADMIN` account password by running `apxchpwd.sql` script. Please note this password (Welcome1##), it will be used later to access APEX internal Workspace.
```
SQL> @apxchpwd.sql
...set_appun.sql
================================================================================
This script can be used to change the password of an Oracle APEX
instance administrator. If the user does not yet exist, a user record will be
created.
================================================================================
Enter the administrator's username [ADMIN]
User "ADMIN" does not yet exist and will be created.
Enter ADMIN's email [ADMIN]
Enter ADMIN's password []
Created instance administrator ADMIN.
```

### Install ORDS

1. Download ORDS software
```
bash-4.4$ curl -o ords-latest.zip  https://download.oracle.com/otn_software/java/ords/ords-latest.zip
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  108M  100  108M    0     0  18.0M      0  0:00:06  0:00:06 --:--:-- 18.7M

bash-4.4$ unzip ords-latest.zip -d /opt/sw/
Archive:  ords-latest.zip
[...]

bash-4.4$ rm ords-latest.zip
```

2. Add ORDS binaries to `PATH` variable
```
bash-4.4$ echo -e 'export PATH="$PATH:/opt/sw/ords/bin"' >> ~/.bash_profile
bash-4.4$ source ~/.bash_profile
```

3. Configure ORDS on port `8080` for the database `jdbc:oracle:thin:@//localhost:1521/FREEPDB1`. Also specify `/opt/sw/apex/images/` path for APEX static resources location
```
bash-4.4$ su -
[root@bffe2831ca7a ~]# chown oracle:oinstall /etc/ords/config
[root@bffe2831ca7a ~]# exit

bash-4.4$ ords --config /etc/ords/config install

ORDS: Release 23.3 Production on Tue Nov 14 08:25:30 2023

Copyright (c) 2010, 2023, Oracle.

Configuration:
  /etc/ords/config/

The configuration folder /etc/ords/config does not contain any configuration files.

Oracle REST Data Services - Interactive Install

  Enter a number to select the type of installation
    [1] Install or upgrade ORDS in the database only
    [2] Create or update a database pool and install/upgrade ORDS in the database
    [3] Create or update a database pool only
  Choose [2]:
  Enter a number to select the database connection type to use
    [1] Basic (host name, port, service name)
    [2] TNS (TNS alias, TNS directory)
    [3] Custom database URL
  Choose [1]:
  Enter the database host name [localhost]:
  Enter the database listen port [1521]:
  Enter the database service name [FREE]: FREEPDB1
  Provide database user name with administrator privileges.
    Enter the administrator username: sys
  Enter the database password for SYS AS SYSDBA:
Connecting to database user: SYS AS SYSDBA url: jdbc:oracle:thin:@//localhost:1521/FREEPDB1

Retrieving information.
  Enter the default tablespace for ORDS_METADATA and ORDS_PUBLIC_USER [SYSAUX]:
  Enter the temporary tablespace for ORDS_METADATA and ORDS_PUBLIC_USER [TEMP]:
  Enter a number to select additional feature(s) to enable:
    [1] Database Actions  (Enables all features)
    [2] REST Enabled SQL and Database API
    [3] REST Enabled SQL
    [4] Database API
    [5] None
  Choose [1]:
  Enter a number to configure and start ORDS in standalone mode
    [1] Configure and start ORDS in standalone mode
    [2] Skip
  Choose [1]:
  Enter a number to select the protocol
    [1] HTTP
    [2] HTTPS
  Choose [1]:
  Enter the HTTP port [8080]: 8080
  Enter the APEX static resources location: /opt/sw/apex/images/
The setting named: db.connectionType was set to: basic in configuration: default
The setting named: db.hostname was set to: localhost in configuration: default
The setting named: db.port was set to: 1521 in configuration: default
The setting named: db.servicename was set to: FREEPDB1 in configuration: default
The setting named: plsql.gateway.mode was set to: proxied in configuration: default
The setting named: db.username was set to: ORDS_PUBLIC_USER in configuration: default
The setting named: db.password was set to: ****** in configuration: default
The setting named: feature.sdw was set to: true in configuration: default
The global setting named: database.api.enabled was set to: true
The setting named: restEnabledSql.active was set to: true in configuration: default
The setting named: security.requestValidationFunction was set to: ords_util.authorize_plsql_gateway in configuration: default
The global setting named: standalone.http.port was set to: 8080
The global setting named: standalone.static.path was set to: /home/oracle/apex/images/
The global setting named: standalone.static.context.path was set to: /i
The global setting named: standalone.context.path was set to: /ords
The global setting named: standalone.doc.root was set to: /etc/ords/config/global/doc_root
2023-11-14T08:34:31.517Z INFO        Updating ORDS_PUBLIC_USER database password in FREEPDB1
------------------------------------------------------------
Date       : 14 Nov 2023 08:34:30
Release    : Oracle REST Data Services 23.3.0.r2891830

[...]

[*** Info: Completed updating database password for ORDS_PUBLIC_USER. Elapsed time: 00:00:14.290
 ]
2023-11-14T08:34:46.814Z INFO        HTTP and HTTP/2 cleartext listening on host: 0.0.0.0 port: 8080
2023-11-14T08:34:46.950Z INFO        Disabling document root because the specified folder does not exist: /etc/ords/config/global/doc_root
2023-11-14T08:34:46.952Z INFO        Default forwarding from / to contextRoot configured.
2023-11-14T08:34:56.804Z INFO        Configuration properties for: |default|lo|
db.servicename=FREEPDB1
standalone.context.path=/ords
db.hostname=localhost
db.password=******
conf.use.wallet=true
security.requestValidationFunction=ords_util.authorize_plsql_gateway
standalone.static.context.path=/i
database.api.enabled=true
db.username=ORDS_PUBLIC_USER
standalone.http.port=8080
standalone.static.path=/home/oracle/apex/images/
restEnabledSql.active=true
resource.templates.enabled=false
plsql.gateway.mode=proxied
db.port=1521
feature.sdw=true
config.required=true
db.connectionType=basic
standalone.doc.root=/etc/ords/config/global/doc_root

2023-11-14T08:34:56.807Z WARNING     *** jdbc.MaxLimit in configuration |default|lo| is using a value of 20, this setting may not be sized adequately for a production environment ***
2023-11-14T08:34:56.808Z WARNING     *** jdbc.InitialLimit in configuration |default|lo| is using a value of 3, this setting may not be sized adequately for a production environment ***
2023-11-14T08:35:15.424Z INFO

Mapped local pools from /etc/ords/config/databases:
  /ords/                              => default                        => VALID


2023-11-14T08:35:15.676Z INFO        Oracle REST Data Services initialized
Oracle REST Data Services version : 23.3.0.r2891830
Oracle REST Data Services server info: jetty/10.0.17
Oracle REST Data Services java info: OpenJDK 64-Bit Server VM 17.0.9+9-LTS
```

4. Verify installation. The last message seen on the screen should be the OpenJDK server up and running. In a web browser go to http://localhost:8023/ords and check that APEX can be found there. Please note that port `8023` is the host port (mapped to container port 8080).
```
bash-4.4$ sqlplus / as sysdba

SQL*Plus: Release 23.0.0.0.0 - Production on Tue Nov 14 08:15:48 2023
Version 23.3.0.23.09

Copyright (c) 1982, 2023, Oracle.  All rights reserved.


Connected to:
Oracle Database 23c Free Release 23.0.0.0.0 - Develop, Learn, and Run for Free
Version 23.3.0.23.09

SQL> ALTER SESSION SET CONTAINER = FREEPDB1;

Session altered.

SQL> SELECT username FROM all_users WHERE username LIKE 'APEX%';

USERNAME
--------------------------------------------------------------------------------
APEX_230200
APEX_PUBLIC_USER
```

### Access Oracle APEX

In a web browser go to `http://localhost:8023/ords/apex`. Initially you only can access to `internal` APEX Workspace, from where you can create regular workspaces.
- Workspace: internal
- Username: ADMIN
- Password: (the one setup when running `apxchpwd.sql` script)

### Post Installation

- Create ORDS start and stop scripts
```
bash-4.4$ mkdir /home/oracle/logs /home/oracle/scripts

bash-4.4$ vi /home/oracle/scripts/start_ords.sh

export ORDS_HOME=/opt/sw/ords/bin/ords
export _JAVA_OPTIONS="-Xms512M -Xmx512M"
LOGFILE=/home/oracle/logs/ords-`date +"%Y""%m""%d"`.log
nohup ${ORDS_HOME} --config /etc/ords/config serve >> $LOGFILE 2>&1 & echo "View log file with : tail -f $LOGFILE"

bash-4.4$ vi /home/oracle/scripts/stop_ords.sh

kill `ps -ef | grep [o]rds.war | awk '{print $2}'`
```

- Start ORDS with `start_ords.sh` script
```
bash-4.4$ sh /home/oracle/scripts/start_ords.sh
View log file with : tail -f /home/oracle/logs/ords-20231127.log

bash-4.4$ tail -f /home/oracle/logs/ords-20231127.log

Mapped local pools from /etc/ords/config/databases:
  /ords/                              => default                        => VALID


2023-11-27T15:28:04.271Z INFO        Oracle REST Data Services initialized
Oracle REST Data Services version : 23.3.0.r2891830
Oracle REST Data Services server info: jetty/10.0.17
Oracle REST Data Services java info: OpenJDK 64-Bit Server VM 17.0.9+9-LTS
```

### Extra configuration

In order to allow APEX instance to access remote endpoints, such as REST APIs, it is necessary to allow incoming connections to `APEX_230200`. 

1. Run the following PL/SQL Block only once:
```
bash-4.4$ sqlplus / as sysdba

SQL> ALTER SESSION SET CONTAINER = FREEPDB1;

Session altered.

SQL> BEGIN
  2        DBMS_NETWORK_ACL_ADMIN.APPEND_HOST_ACE(
  3            host => '*',
  4            ace => xs$ace_type(privilege_list => xs$name_list('connect', 'resolv
e'),
  5                principal_name => 'APEX_230200',
  6                principal_type => xs_acl.ptype_db
  7            )
  8        );
  9    END;
 10  /

PL/SQL procedure successfully completed.
```

2. Test ACL by accessing any URL
```
SQL> select utl_http.request('http://google.es') from dual;

UTL_HTTP.REQUEST('HTTP://GOOGLE.ES')
--------------------------------------------------------------------------------
<!doctype html><html itemscope="" itemtype="http://schema.org/WebPage" lang="es"
><head><meta content="Google.es permite acceder a la informacion mundial en cast
ellano, catalan, gallego, euskara e ingles." name="description"><meta content="n
oodp" name="robots"><meta content="text/html; charset=UTF-8" http-equiv="Content
-Type"><meta content="/images/branding/googleg/1x/googleg_standard_color_128dp.p
ng" itemprop="image"><title>Google</title><script nonce="p1kX3w87n199hNE39jj3zw"
>(function(){var _g={kEI:'WrRkZd6KIbuehbIPrPyEwAY',kEXPI:'0,18168,1347300,206,24
15,2389,2316,383,1129371,1805,115,1576571,16114,28684,22430,1362,284,12031,17584
,4998,17075,38444,2872,2891,3926,7828,606,60690,2614,13491,230,20583,4,59617,443
7,22613,6624,7596,1,42154,2,39761,5679,1021,31122,4568,6258,23419,1251,59701,815
5,23350,873,14672,4963,6,1922,9779,42459,3141,17058,5796,3,14433,20206,27365,537
```

## Publish container image

### Create image from existing container

Add environment variables `APEX_SW`, `ORDS_SW`, `ORDS_CONFIG` and `ORDS_START_FILE`
```
docker commit --change='ENV APEX_SW=/opt/sw/apex' --change='ENV ORDS_SW=/opt/sw/ords' --change='ENV ORDS_CONFIG=/etc/ords/config'--change='ENV ORDS_START_FILE=/home/oracle/scripts/start_ords.sh' apex-db apex-ords:latest
```

### Push image to container registry

```
docker login
docker tag apex-ords:latest dbeniteza/oracle-apex-ords:latest
docker push dbeniteza/oracle-apex-ords:latest
```

## Create container with new image

Now you can download the image from Docker Hub [https://hub.docker.com/r/dbeniteza/oracle-apex-ords](https://hub.docker.com/r/dbeniteza/oracle-apex-ords)

```
docker run -d --name apex --restart always -p 8521:1521 -p 8023:8080 -v apex-oradata:/opt/oracle/oradata -v apex-ords-config:/etc/ords/config -v apex-sw:/opt/sw  dbeniteza/oracle-apex-ords:latest
```

## Access Oracle APEX

Access [Oracle APEX workspaces login page](http://localhost:8023/ords/r/apex/workspace-sign-in/oracle-apex-sign-in)

## Useful links

- [Oracle Database Free FAQs](https://www.oracle.com/es/database/free/faq/)
- [Oracle Docs - Downloading and Installing APEX](https://docs.oracle.com/en/database/oracle/apex/23.1/htmig/downloading-installing-apex.html#GUID-B5A5B38D-586C-488A-AE27-A168FAA28FEE)
- [Oracle Docs - Installing Oracle REST Data Services](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/23.3/ordig/installing-and-configuring-oracle-rest-data-services.html#GUID-D86804FC-4365-4499-B170-2F901C971D30)
- [APEX latest version](https://download.oracle.com/otn_software/apex/apex-latest.zip)
- [ORDS latest version](https://download.oracle.com/otn_software/java/ords/ords-latest.zip)