# greenplum-single-node

```
# 一个单机版的安装指南
http://www.greenplumguru.com/?p=1111

The install guide has detailed instructions on how to install a multi-node cluster but it doesn’t cover a single node installation. That is why I created this blog post to cover just a single node.

So look at the install guide but here is a quick list of things you’ll need to change:
– Follow the XFS recommendations in the install guide.
– Step 4 needs to be done on all nodes.
– The /etc/hosts file should have the real IP address, not 127.0.0.1 and it should contain all hosts in the cluster.
– The /data/primary directories need to be created on the segment hosts and /data/master on the master node.
– The hostfile in Step 7 should contain all segment hosts in the cluster.
– You may want to change step 8 to have more data directories. One directory equals one segment per host.
– Step 9 is run on the master only.
```



```
Only need to make 2 hostname entries in host file for same server.

I have created mdw for master and sdw1,sdw2 for segments(1+2)
```

## ubuntu安装单机版

```
# 按照这个文档来的，ubuntu选的腾讯云 18.04 lts公共镜像
https://greenplum.org/install-greenplum-oss-on-ubuntu/
最后安装的greenplum是 （用命令：select version() ; 来检查，输出如下面 ）
                                                                                                            PostgreSQL 9.4.24 (Greenplum Database 6.8.1 build commit:7118e8aca825b743dd9477d19406fcc06fa53852) on x86_64-unknown-linux-gnu, compiled by gcc (Ubuntu 7.5.0-3ubuntu1~18.04) 7.5.0,
 64-bit compiled on Jun 16 2020 18:53:13
```



1. Add the Greenplum PPA repository to your Ubuntu System, like this:

```
sudo add-apt-repository ppa:greenplum/db

# 出现一大屏提示和说明，最后一行提示enter开始，或者ctrl-c中止
# 在腾讯云成都，这一步还挺快，大概十几秒
```

2. Update your Ubuntu system to retrieve information from the recently added repository, like this:

   ```
   sudo apt update
   # 这一步也挺快，大概几秒
   ```

3. Install the Greenplum Database software, like this:

   ```
   sudo apt install greenplum-db
   
   # 这一步也不慢。以上在腾讯云，实际都是自动用了腾讯云的镜像源
   # 如果不是腾讯云的ecs，可能需要自已配置ubuntu的源，否则从国外下载很可能会慢
   # 安装完成后可以检查一下，能看到/opt目录下的greenplum-db-:
   
   ubuntu@VM-0-6-ubuntu:~$ ls -l /opt
   drwxr-xr-x 11 root root 4096 Jul 24 00:39 greenplum-db-6.8.1
   ```

4. Load Greenplum Database software into your environment with the following command. Note you should pick the exact path of the greenplum software directory based on the version of Greenplum:

   ```
   $ . /opt/greenplum-db-6.0.1/greenplum_path.sh
   $ which gpssh
   /opt/greenplum-db-6.0.1/bin/gpssh
   
   # 这里要注意下，这个文件设定了环境变量 $GPHOME，所以后面跟着就在这个term里继续，不要去其它term干活，因为其它term里没有这个$GPHOME环境变量，肯定会出错的
   ```

5. You can see the software is on the path by testing using the **which** command as above. Now you can copy a Greenplum cluster configuration file template into your local directory for editing like this:

   ```
   cp $GPHOME/docs/cli_help/gpconfigs/gpinitsystem_singlenode .
   # 这里我先把这个文件复制到当前用户（默认是ubuntu用户，非root用户）的目录里
   ```

6. #### Edit gpinitsystem Configuration File

   The following edits can be made for the most simple cluster configuration running locally.

   Create this file and put only your hostname into the file:
   **MACHINE_LIST_FILE=./hostlist_singlenode**

   Update this line to have a directory you want to use for primaries for example:
   **declare -a DATA_DIRECTORY=(/gpdata1 /gpdata2)**
   **declare -a DATA_DIRECTORY=(/home/inovick/primary /home/inovick/primary)**
   And make sure the directory mentioned above exists.

   Update this line to have the hostname of your machine, in my case, the hostname is ‘ubuntu’:
   **MASTER_HOSTNAME=hostname_of_machine**
   **MASTER_HOSTNAME=ubuntu**

   Update the master data directory entry in the file and ensure it exists by making the directory:
   **MASTER_DIRECTORY=/home/inovick/master**

   That’s enough to get the database initialized and up running, so close the file and let’s initialize the cluster. We will have a master segment instance and two primary segment instances with this configuration. In more advanced setups you would configure a standby master and segment mirrors on additional hosts, and the data would be automatically both sharded (distributed) between the primary segments and mirrored from primaries to mirrors.

   

   ```
   MACHINE_LIST_FILE=./hostlist_singlenode
   # 这行已经有了，不动
   # 但是这里要新建一个文件在默认目录（也就是用户目录）: touch hostlist_singlenode
   # 然后这个hostlist_singlenode文件里，有且仅有本机的hostname就行（我的是VM-0-6-ubuntu)
   
   declare -a DATA_DIRECTORY=(/gpdata1 /gpdata2)
   # 这行也已经有了，不动
   
   declare -a DATA_DIRECTORY=(/home/inovick/primary /home/inovick/primary)
   # 这行没有，加在上一个declare那行之后
   
   # /gpdata1  /gpdata2  /home/inovick这三个目录都没有，因此都mkdir加上，并且chown给用户ubuntu
   # /home/inovick/primary 这个目录也没有，加上；权限不用改，已经都是ubuntu的权限了
   
   # MASTER_DIRECTORY=/gpmaster，这是原来的配置，改成下面这个
   MASTER_DIRECTORY=/home/inovick/master 
   
   # 同时要mkdir建那个 /home/inovick/master目录
   
   # MASTER_HOSTNAME=hostname_of_machine 这行要改，我的机器hostname是 VM-0-6-ubuntu
   MASTER_HOSTNAME=VM-0-6-ubuntu
   ```

   

   ```
   sudo mkdir /gpdata1
   sudo mkdir /gpdata2
   sudo chown ubuntu /gpdata1
   sudo chown ubuntu /gpdata2
   sudo mkdir /home/inovick/
   sudo chown ubuntu /home/inovick/
   # 验证下
   ls -l /home |grep inovi
   drwxr-xr-x 2 ubuntu root   4096 Jul 24 00:56 inovick
   ```

7. #### Run gpinitsystem

   确认上一步该建的目录都建好，目录权限都给了用户，配置文件修改完成，然后跑下面这个gpssh-exkeys命令

   First, let’s make sure ssh keys are exchanged by running the following command. Screenshot from my system is shown below:

   ```
   # 注意这里应该在前面那个term里执行，然后应该在默认目录执行（我是在默认用户ubuntu的目录执行，hostlist_singlenode文件，和前面这个配置文件也都在这个默认用户目录）
   gpssh-exkeys -f hostlist_singlenode
   
   # 执行后我的输出如下：
   ubuntu@VM-0-6-ubuntu:~$ gpssh-exkeys -f hostlist_singlenode
   [STEP 1 of 5] create local ID and authorize on local host
   
   [STEP 2 of 5] keyscan all hosts and update known_hosts file
   
   [STEP 3 of 5] retrieving credentials from remote hosts
   
   [STEP 4 of 5] determine common authentication file content
   
   [STEP 5 of 5] copy authentication files to all remote hosts
   
   [INFO] completed successfully
   ```

8. Ok, we need to start the cluster, let’s get started. Run the following command:

   ```
   # 这里还是在前面那个环境变量$GPHOME同一个term里执行
   gpinitsystem -c gpinitsystem_singlenode
   
   # 一串运行信息，最后要确认才继续：
   20200724:01:17:29:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-Greenplum Primary Segment Configuration
   20200724:01:17:29:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:----------------------------------------
   20200724:01:17:29:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-VM-0-6-ubuntu        6000    VM-0-6-ubuntu   /home/inovick/primary/gpsne0    2
   20200724:01:17:29:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-VM-0-6-ubuntu        6001    VM-0-6-ubuntu   /home/inovick/primary/gpsne1    3
   
   Continue with Greenplum creation Yy|Nn (default=N):
   ```

   接着按y确认之后继续，最后出现提示：

   ```
   20200724:01:19:49:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-Greenplum Database instance successfully created
   20200724:01:19:49:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-------------------------------------------------------
   20200724:01:19:49:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-To complete the environment configuration, please
   20200724:01:19:49:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-update ubuntu .bashrc file with the following
   20200724:01:19:49:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-1. Ensure that the greenplum_path.sh file is sourced
   20200724:01:19:49:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-2. Add "export MASTER_DATA_DIRECTORY=/home/inovick/master/gpsne-1"
   20200724:01:19:49:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-   to access the Greenplum scripts for this instance:
   20200724:01:19:50:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-   or, use -d /home/inovick/master/gpsne-1 option for the Greenplum scripts
   20200724:01:19:50:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-   Example gpstate -d /home/inovick/master/gpsne-1
   20200724:01:19:50:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-Script log file = /home/ubuntu/gpAdminLogs/gpinitsystem_20200724.log
   20200724:01:19:50:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-To remove instance, run gpdeletesystem utility
   20200724:01:19:50:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-To initialize a Standby Master Segment for this Greenplum instance
   20200724:01:19:50:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-Review options for gpinitstandby
   20200724:01:19:50:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-------------------------------------------------------
   20200724:01:19:50:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-The Master /home/inovick/master/gpsne-1/pg_hba.conf post gpinitsystem
   20200724:01:19:50:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-has been configured to allow all hosts within this new
   20200724:01:19:50:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-array to intercommunicate. Any hosts external to this
   20200724:01:19:50:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-new array must be explicitly added to this file
   20200724:01:19:50:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-Refer to the Greenplum Admin support guide which is
   20200724:01:19:50:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-located in the /opt/greenplum-db-6.8.1/docs directory
   20200724:01:19:50:001916 gpinitsystem:VM-0-6-ubuntu:ubuntu-[INFO]:-------------------------------------------------------
   ```

9. 测试一下

   ```
   # 还是要在前面设了环境变量的$GPHOME那个term里，不然找不到creatdb命令（因为是在$GPHOME的子目录下）
   
   createdb demo
   psql demo
   psql (9.4.24)
   Type "help" for help.
   
   demo=# select * from gp_segment_configuration ;
    dbid | content | role | preferred_role | mode | status | port |   hostname    |    address    |           datadir
   ------+---------+------+----------------+------+--------+------+---------------+---------------+------------------------------
       1 |      -1 | p    | p              | n    | u      | 5432 | VM-0-6-ubuntu | VM-0-6-ubuntu | /home/inovick/master/gpsne-1
       2 |       0 | p    | p              | n    | u      | 6000 | VM-0-6-ubuntu | VM-0-6-ubuntu | /home/inovick/primary/gpsne0
       3 |       1 | p    | p              | n    | u      | 6001 | VM-0-6-ubuntu | VM-0-6-ubuntu | /home/inovick/primary/gpsne1
   (3 rows)
   ```

   这应该是安装成功了。后面可以进行功能性的操作及检验了。

> 注意设定了$GPHOME变量之后，操作都在这个term里做，因为好多都要用到这个环境变量
>
> 建那那些目录之后，chown给当前用户（chgrp没有管，还是root，看起来没有影响）
>
> 用了一个 1CPU / 2G 的腾讯云ecs，安装成功；如果后面数据量上来了，不知道内存是不是不够；万一确定内存不够，到时再开swap试试！



