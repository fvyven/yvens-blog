---
layout: post
title: "[BIG DATA] CDH 3.2.1 on CentOS 7.5 集群安装(半)离线部署"
tags: 
  - Big Data
  - CDH
---


\* _本文使用的是云服务器 CentOS 7.5 集群, 且使用 root 账户, 所有包管理工具, CDH版本和目录可能不一样, 请自行切换_

\* _本文为可能出现纰漏, 详细可参考访问[官方安装文档](https://docs.cloudera.com/documentation/enterprise/6/6.0/topics/installation.html)和Google_

---


## 一、	集群环境准备, 参考[官方文档](https://docs.cloudera.com/documentation/enterprise/6/latest/topics/installation_reqts.html)

1. ### 集群IP映射

    为所有节点设置:
    ```shell
    echo "172.16.20.245 cdh-0001" >> /etc/hosts
    echo "172.16.20.28 cdh-0002" >> /etc/hosts
    echo "172.16.20.196 cdh-0003" >> /etc/hosts
    echo "172.16.20.170 cdh-0004" >> /etc/hosts 
    ```

2. ### 配置集群ssh免密登录

    获取密钥文件, 所有节点执行: 

    ```shell
    ssh-keygen -t rsa
    ```
    将所有节点的 .ssh/id_rsa.pub文件传到主节点, 然后将每个节点的id_rsa.pub加入到 .ssh/authorized_keys文件中:

    ```shell
    id_rsa.pub >> authorized_keys
    ```
    将这个包含所有节点公钥的authorized_keys文件上传到每个节点的.ssh目录下

    *(这里使用的是root账户, 如使用其他账户, 请注意目录和.ssh文件夹的权限问题)*


3. ### 关闭SELinux

    所有节点设置
    
    先查看SELinux状态, 如果为 *disabled* 则无需修改, 命令:
    ```shell
    /usr/sbin/sestatus -v
    ```
    否则就 */etc/selinux/config* 文件中的 *SELINUX=enforcing* 改为 *SELINUX=disabled*

4. ### 配置集群时间同步

    所有节点安装Chrony:

    ```shell
    yum -y install chrony
    ```
    
    主节点配置 */etc/chrony.conf* 文件: 按需设置 *NTP Server*, 可以保持使用默认4个Server配置, 也配置多个

    允许从节点IP访问: *allow 172.16.20.0/24*
    
    从节点配置 */etc/chrony.conf* 文件: 注释其他server, 只把一个server指向主节点IP: *server 172.16.20.245 iburst*

    所有节点启动Chrony服务:

    ```shell
    systemctl enable chronyd.service && systemctl restart chronyd.service
    ```

5. ### 其他CDH建议配置

    禁用透明大页面压缩, 所有节点写入到 */etc/rc.d/rc.local* 文件:

    ```shell
    echo never > /sys/kernel/mm/transparent_hugepage/defrag
    echo never > /sys/kernel/mm/transparent_hugepage/enabled
    chmod +x /etc/rc.d/rc.local
    ```
    
    优化交换分区, 所有节点执行:

    ```shell
    echo "vm.swappiness = 10" >> /etc/sysctl.conf
    ```


## 二、	所需组件安装及初始化, 参考[官方文档](https://docs.cloudera.com/documentation/enterprise/6/latest/topics/install_cm_cdh.html)

1. ### JDK, 所有节点安装:
    ```shell
    yum -y install java-1.8.0-openjdk.x86_64
    ```

2. ### MySQL, 任选一个节点安装, 建议主节点
    安装, 不做详述, 以下为主要命令:

    ```shell
    $ wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
    $ rpm -ivh mysql-community-release-el7-5.noarch.rpm
    $ yum -y install mysql-server
    $ systemctl enable mysqld && systemctl start mysqld
    $ /usr/bin/mysql_secure_installation
    ```

    按需配置 */etc/my.cnf*, 可参考[CDH推荐配置](https://docs.cloudera.com/documentation/enterprise/6/latest/topics/cm_ig_mysql.html#cmig_topic_5_5)

    创建 CDH 所需 database 和 user:

    ```sql
    CREATE DATABASE scm DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
    CREATE DATABASE amon DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
    CREATE DATABASE rman DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
    CREATE DATABASE hue DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
    CREATE DATABASE metastore DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
    CREATE DATABASE sentry DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
    CREATE DATABASE nav DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
    CREATE DATABASE navms DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
    CREATE DATABASE oozie DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;

    GRANT ALL ON scm.* TO 'scm'@'%' IDENTIFIED BY 'cdh123@test';
    GRANT ALL ON amon.* TO 'amon'@'%' IDENTIFIED BY 'cdh123@test';
    GRANT ALL ON rman.* TO 'rman'@'%' IDENTIFIED BY 'cdh123@test';
    GRANT ALL ON hue.* TO 'hue'@'%' IDENTIFIED BY 'cdh123@test';
    GRANT ALL ON metastore.* TO 'metastore'@'%' IDENTIFIED BY 'cdh123@test';
    GRANT ALL ON sentry.* TO 'sentry'@'%' IDENTIFIED BY 'cdh123@test';
    GRANT ALL ON nav.* TO 'nav'@'%' IDENTIFIED BY 'cdh123@test';
    GRANT ALL ON navms.* TO 'navms'@'%' IDENTIFIED BY 'cdh123@test';
    GRANT ALL ON oozie.* TO 'oozie'@'%' IDENTIFIED BY 'cdh123@test';

    FLUSH PRIVILEGES;
    ```
    
3. ### MySQL所在节点添加对应版本的 jdbc 驱动, 主要命令:

    ```shell
    $ wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.46.tar.gz
    $ tar zxvf mysql-connector-java-5.1.46.tar.gz
    $ cp mysql-connector-java-5.1.46-bin.jar /usr/share/java/mysql-connector-java.jar ##(主要去掉版本后缀)
    ```


## 三、	CDH组件安装, 参考[官方文档](https://docs.cloudera.com/documentation/enterprise/6/latest/topics/install_cm_cdh.html)

1. ### 离线部署Cloudera Manager Server及Agent

    上传 cm6.3.1-redhat7.tar.gz 到所有节点并解压到 */opt/cloudera-manager/* 目录下

    在**主节点**使用 *rpm* 离线安装 daemons, server 和 agent:

    ```shell
    cd /opt/cloudera-manager/cm6.3.1/RPMS/x86_64/
    rpm -ivh cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm --nodeps --force
    rpm -ivh cloudera-manager-server-6.3.1-1466458.el7.x86_64.rpm --nodeps --force
    rpm -ivh cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm --nodeps --force
    ```
    所有**从节点**使用 *rpm* 离线安装 daemons 和 agent:

    ```shell
    cd /opt/cloudera-manager/cm6.3.1/RPMS/x86_64/
    rpm -ivh cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm --nodeps --force
    rpm -ivh cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm --nodeps --force
    ```
    修改所有**从节点**的 agent 配置, server 指向主节点 *cdh-0001*

    修改 */etc/cloudera-scm-agent/config.ini* : *server_host=cdh-0001*

    修改**主节点** */etc/cloudera-scm-server/db.properties* 配置:
      
    *(此处配置为上方 2.2.3 所创建的数据库和账号密码)*

    ```conf
    com.cloudera.cmf.db.host=cdh-0001
    com.cloudera.cmf.db.name=scm
    com.cloudera.cmf.db.user=scm
    com.cloudera.cmf.db.password=cdh123@test
    # If scm-server uses Embedded DB then it is set to EMBEDDED
    # If scm-server uses External DB then it is set to EXTERNAL
    com.cloudera.cmf.db.setupType=EXTERNAL
    ```

    启动 Cloudera Manager Server 和 Agent:

    ```shell
    # 启动CM:
    systemctl enable cloudera-scm-server && systemctl start cloudera-scm-server
    ```

    查看日志, 解决出现的问题: 
    ```shell
    tail -f /var/log/cloudera-scm-server/cloudera-scm-server.log
    ```

    CM 启动成功后, 所有节点启动 Agent:
    ```shell 
    systemctl enable cloudera-scm-agent && systemctl start cloudera-scm-agent
    ```

    浏览器启动 http://chd-0001:7180 页面配置集群

2. ### Cloudera Manager 集群安装和设置
    
    欢迎页面, 一共三个步骤, 同意条款, 选择是否付费版本, 进入集群安装界面(*没有配图, 请自行脑补~*)
    
    **集群安装**

    1). 欢迎界面
    
    2). 自定义集群名称

    3). 指定主机, 如果上面的 Agent 和配置全部正确, 不用搜索新主机, 所有主机会展示在 '当前管理的主机' Tab下, 全选后进入下一步

    4). 选择存储库
    
    &emsp;a. 将离线包: CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel 和 CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.sha (此文件为parcel的hash码, 请不要随意修改文件) 拷贝到 */opt/cloudera/parcel-repo* 目录

    &emsp;b. 修改改目录下所有文件权限: 
    ```shell
    chown -R cloudera-scm:cloudera-scm /opt/cloudera/parcel-repo/*
    ```

    &emsp;c. 点击 '使用Parcel(建议)' 后的更多选项按钮, 移除所有远程Parcel库, 保存

    &emsp;d. 如果没有问题会默认选中拷贝的离线Parcel包

    &emsp;e. 点击继续

    5). 安装 Parcel, 由于上一步配置好了离线 Parcel, 这里只需要等待分配, 解压和激活三个步骤自动完成

    6). 检查集群

    &emsp;*可能会出现的问题包括上方步骤1.5出现两个配置项和修复 Psycopg2 版本较低(忽略)*

    7). 问题全部解决后进入集群设置

    **集群配置**

    1). 选择需要的服务, 常用: HDFS, HIVE, YARN, Spark, Zookeeper 等, 根据需要执行选择

    2). 自定义角色分配, 设置每个服务的角色和运行的节点, 根据情况设置

    3). 数据设置, 如果上一步没有选择‘Activity Monitor’, 这一步骤会被省略, 所以Activity Monitor最好选择cdh-0001(mysql安装的节点)

    &emsp;输入数据库amon, 用户名amon和密码cdh123@test (2.2.3的设置), 测试连接

    4). 审核更改, 此步骤会设置所选角色的部分基础文件目录, 根据情况修改

    5). 执行命令详细信息, 等待自动运行完成, 全部通过后继续

    6). 至此所有安装步骤完成, 可进入 Cloudera Manager 进行集群监控和管理
