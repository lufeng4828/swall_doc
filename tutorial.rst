安装说明
=====================

安装swall比较简单，首先安装下zookeeper，安装好了以后安装swall即可

    ===========   =========   =============
    -----------   ---------   -------------
    名称           配置        IP地址
    ===========   =========   =============
    zookeeper1    centos6.2   192.168.4.181
    swall1        centos6.2   192.168.4.180
    ===========   =========   =============



安装jdk和配置java环境
------------------------


下载 jdk7_ （选择jdk-7u55-linux-x64.gz进行下载）

上传jdk-7u55-linux-x64.gz到服务器，解压::

    [root@zookeeper1 ~]# tar xf jdk-7u55-linux-x64.gz -C /usr/local/
    [root@zookeeper1 ~]# ls /usr/zookeeper/
    /usr/local/jdk1.7.0_55
    [root@zookeeper1 ~]# mv /usr/local/jdk1.7.0_55 /usr/local/java
    [root@zookeeper1 ~]#

配置jdk环境::

    [root@zookeeper1 ~]# cat >> /etc/bashrc <<\eof
    > export JAVA_HOME=/usr/local/java
    > export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
    > export PATH=$PATH:$JAVA_HOME/bin
    > eof
    [root@zookeeper1 ~]# source /etc/bashrc

检查java是否安装成功::

    [root@zookeeper1 ~]# java -version
    java version "1.7.0_55"
    Java(TM) SE Runtime Environment (build 1.7.0_55-b13)
    Java HotSpot(TM) 64-Bit Server VM (build 24.55-b03, mixed mode)



安装zookeeper集群
------------------------


下载 zookeeper_ （当前最新版本zookeeper-3.4.5-cdh5.0.0.tar.gz）

上传zookeeper-3.4.5-cdh5.0.0.tar.gz到服务器，解压::

    [root@zookeeper1 ~]# tar xf zookeeper-3.4.5-cdh5.0.0.tar.gz -C /usr/local/
    [root@zookeeper1 ~]# ls -d /usr/local/zookeeper-3.4.5-cdh5.0.0
    /usr/local/zookeeper-3.4.5-cdh5.0.0
    [root@zookeeper1 ~]# mv /usr/local/zookeeper-3.4.5-cdh5.0.0 /usr/local/zookeeper

配置zookeeper，这里配置的是单节点::

    [root@zookeeper1 ~]# cd /usr/local/zookeeper/conf/
    [root@zookeeper1 conf]# mv zoo_sample.cfg zoo.cfg
    [root@zookeeper1 conf]# vim zoo.cfg #修改dataDir=/data/database/zookeeper
    [root@zookeeper1 conf]# mkdir -p /data/database/zookeeper
    [root@zookeeper1 conf]# cd ..
    [root@zookeeper1 zookeeper]# cd bin/
    [root@zookeeper1 bin]# ./zkServer.sh start
    JMX enabled by default
    Using config: /usr/local/zookeeper/bin/../conf/zoo.cfg
    Starting zookeeper ... STARTED
    [root@zookeeper1 bin]# ./zkServer.sh status
    JMX enabled by default
    Using config: /usr/local/zookeeper/bin/../conf/zoo.cfg
    Mode: standalone #说明配置成功
    [root@zookeeper1 bin]#

配置防火墙，允许访问2181端口（这里以RH-Firewall-1-INPUT规则为例）::

    [root@zookeeper1 bin]# iptables -A INPUT -p tcp --dport 2181 -j ACCEPT
    [root@zookeeper1 bin]# iptables -L -n | grep 2181
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           tcp dpt:2181
    [root@zookeeper1 bin]# iptables-save > /etc/sysconfig/iptables


安装swall
------------------------

下载swall最新版本，内部下载地址是： swall_ 或者通过git命令::

    [root@swall1 ~]# mkdir /data
    [root@swall1 data]# git clone https://github.com/lufeng4828/swall.git
    [root@swall1 data]# cd swall/conf

    主配置swall.conf：

    node_role：   定义swall角色，角色有相应的模块组，一个swall节点可以有多个角色，用逗号分开
    node_ip：     swall节点ip
    cache：       用来存放临时文件的，主要是用来加速文件传输
    backup：      拷贝文件的时候，默认会保存一份原来的文件放在这个目录下面
    fs_plugin：   文件传输组件存放的地方
    pidfile：     pid存放的文件
    log_file：    日志存放文件
    log_level：   日志打印级别
    token：       信息传输通过这个来加密的，这个地方可以保留默认，修改起来麻烦

    文件传输配置fs.conf

    fs_type：     指定用什么组件来传输数据，默认实现有ssh、ftp、rsync，对应plugins/fservice目录下面的文件名（不带后缀）
    fs_host：     文件服务器ip，我们是通过它提供的上传、下载服务来传输数据的
    fs_port：     服务端口
    fs_user：     文件服务访问的账户
    fs_pass：     文件服务的账户密码
    fs_tmp_dir：  指定一个路径，比如ssh，其他如ftp、rsync用不到这个选项
    fs_failtry：  针对rsync的选项，失败重试次数，其他ftp、ssh用不到

    zookeeper配置zk.conf

    zk_servers：  zookeeper的ip和端口，多个用逗号隔开，如：zk_servers = 192.168.4.181:2181,192.168.4.182:2181
    zk_scheme：   zookeeper认证类型，目前只支持digest
    zk_auth：     zookeeper的degest认证密码，格式如：vcode:swall!@#，要有个冒号

    roles.d配置，目录下面存放的配置是针对角色的，主要是指定，这个角色的节点怎么定义，是通过模块自定义还是直接写死到配置等
    swall.conf配置中node_role配置的角色对应roles.d下面的文件，例如swall.conf中配置了node_role=game,server，roles.d下面就有
    两个配置：game.conf，server.conf

    node_name：   自定该角色节点名列表，可以写死，多个节点名通过逗号分隔，如：vcode_swc_1,vcode_swc_2
                  也可以通过模块自动生成节点名列表，通过加@@标识，如：@@gen.game，意思是调用gen.py的game函数生成，gen.py要放在
                  module/common下面
    project：     如果node_name不是@@格式的，就一定要为你的角色指定一个项目标识，因为公司里面可以有很多项目，很多调用信息带上会很
                  容易处理多个项目的环境
    agent：       可以认为是二级项目标识，作用和project一样，如果node_name不是@@格式的，就一定要为你的角色指定一个二级标识

以server角色为例，配置server.conf角色的节点::

    [main]
    project = xyz
    agent = sa
    node_name = %(project)s_%(agent)s_server_192.168.4.180

.. attention::

    这个形式的配置一定要配置project，agent、node_name

如果一个角色下面有多个节点，比如game角色，一台机器上面有多个游戏服，在swall中，我们把一个游戏当做一个节点。那么上面这种形式的配置需要如下修改::

    [main]
    project = xyz
    agent = sa
    node_name = xyz_sa_600,xyz_sa_700,xyz_sa_750

上面的配置在一台机器上面支持一个游戏代理情况下适用，如果是一个机器上面安装多个代理的多个游戏服，就不能通过上面的方法获取节点了，需要编写模块::

    [main]
    node_name = @@gen.game

上面的@@gen.game是指节点信息通过gen.py中得game函数获取，这个模块是存放在/module/common或者/module/game目录中，而且这个game函数必须用swall.utils.gen_node来修饰，
这个修饰器会对自定义节点获取函数进行约束，主要约束函数的返回值，目前游戏的game节点获取代码如下:

.. code:: python

    import os
    import re
    from swall.utils import gen_node
    from swall.logger import Logger

    log = Logger().logger

    @gen_node
    def game(*args):
        """
        def game(*args) -> 获取游戏节点列表，返回格式是
        @return dict:
        {
            'xyz_elex_9002': {'project': 'xyz', 'agent': 'elex'},
            'xyz_elex_9001': {'project': 'xyz', , 'agent': 'elex'},
            'xyz_fline_1': {'project': 'xyz', 'agent': 'fline'},
            'xyz_fcenter_1': {'project': 'xyz', 'agent': 'fcenter'}
        }
        """
        all_games = {}

        def rep(x):
            project = x.split('_')[0]
            agent = x.split('_')[1]
            sid = x.split('_')[2]
            if len(args) == 1:
                sub_role = args[0]
                return {"%s_%s_%s_%s" % (sub_role, project, agent, sid): {"agent": agent, "project": project, "role": "game"}}
            else:
                return {"%s_%s_%s" % (project, agent, sid): {"agent": agent, "project": project, "role": "game"}}
        for n in [g for g in os.listdir("/data/")
                  if re.match(r'[a-z0-9]+_[0-9a-z]+_[0-9]+$', g)]:
            all_games.update(rep(n))
        return all_games

以rsync为例，rsync需要什么配置，只需要看plugins/fservice/rsync.py中self.fs_conf属性::

    fs_type = rsync
    fs_host = 192.168.4.181
    fs_port = 61768
    fs_user = swall
    fs_pass = vGjeVUncnbPV8CcZ
    fs_tmp_dir = /data/swall_fs
    fs_failtry = 3


配置rsync，一定要192.168.4.181的rsync服务已经正确运行了，下面给出配置rsync过程，这里我们把rsync也配置到192.168.4.181::

    [root@zookeeper1 ~]# useradd swall
    [root@zookeeper1 ~]# mkdir /data/swall_fs
    [root@zookeeper1 ~]# chown -R swall:swall
    [root@zookeeper1 ~]# vim /etc/rsyncd.conf


rsync配置如下::

    secrets file = /etc/rsyncd.secrets
    list = no
    port = 61768
    read only = yes
    uid = swall
    gid = swall
    max connections = 3000
    log file = /var/log/rsyncd_swall.log
    pid file = /var/run/rsyncd_swall.pid
    lock file = /var/run/rsync_swall.lock

    [swall_fs]
    path = /data/swall_fs
    auth users = swall
    read only = no

设置rsync密码::

    [root@zookeeper1 ~]# echo 'swall:vGjeVUncnbPV8CcZ' > /etc/rsyncd.secrets
    [root@zookeeper1 ~]# chmod 600 /etc/rsyncd.secrets


防火墙要允许访问61768端口::

    [root@zookeeper1 bin]# iptables -A INPUT -p tcp --dport 61768 -j ACCEPT
    [root@zookeeper1 bin]# iptables -L -n | grep 61768
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           tcp dpt:61768
    [root@zookeeper1 bin]# iptables-save > /etc/sysconfig/iptables

运行rsync::

    [root@zookeeper1 bin]# rsync --port=61768 --config=/etc/rsyncd.conf --daemon


测试rsync是否正常服务,登录其他机器，这里以192.168.4.180为例::

    [root@swall1 ~]# RSYNC_PASSWORD=vGjeVUncnbPV8CcZ rsync -a --port=61768 --partial /etc/services swall@192.168.8.180::swall_fs/service
    [root@swall1 ~]# echo $?
    0
    [root@swall1 ~]# RSYNC_PASSWORD=vGjeVUncnbPV8CcZ rsync -a --port=61768 --partial swall@192.168.8.180::swall_fs/service /tmp/service
    [root@swall1 ~]# ll /tmp/service
    -rw-r--r-- 1 root root 640999 Jan 12  2010 /tmp/service

如上，说明rsync配置成功。

在启动swall之前，下面给出一个完整配置示例::

    [main]
    #定义角色
    node_role = game,server

    node_ip = 172.17.0.4

    #缓存路径
    cache = var/cache

    #模块路径
    module = module/

    #文件备份路径
    backup = var/backup

    #plugins路径
    fs_plugin = plugins/fservice

    #pid文件
    pidfile = /tmp/.swall.pid

    #日志定义
    log_file = /data/logs/swall.log

    log_level = INFO

    #认证key，数据传输用
    token = yhIC7oenuJDpBxqyP3GSHn7mgQThRHtOnNNwqpJnyPVhR1n9Y9Q+/T3PJfjYCZdiGRrX03CM+VI=

    ###roles.d/server.conf角色配置
    [main]
    project = swall
    agent = sa
    node_name = %(project)s_%(agent)s_server_172.17.0.4

    ###roles.d/game.conf配置
    [main]
    node_name = @@gen.game


第一次配置swall集群下初始化zookeeper目录::

    [root@swall1 ~]# cd /data/swall/bin
    [root@swall1 ~]# ./swall init

启动swall节点程序::

    [root@swall1 ~]# cd /data/swall/bin
    [root@swall1 ~]# ./swall server start

测试命令::

    [root@swall1 ~]# swall ctl server "*"  sys.ping
    ####################
    [server] xyz_sa_server_192.168.8.180 : 1
    ####################
    一共执行了[1]个


swall简单用法
------------------------

swall的管理工具是swall，使用方法如下::

    Usage: cmd.py ctl  <role> [target] <module.function> [arguments]

    Send command to swall server.

    Options:
    -h, --help            show this help message and exit

    Options for swall ctl:
    -e EXCLUDE, --exclude=EXCLUDE
                        Specify the exclude hosts by regix

    -t TIMEOUT, --timeout=TIMEOUT
                        Specify the timeout,the unit is second

    -r, --is_raw        Specify the raw output
    -n NTHREAD, --nthread=NTHREAD
                        Specify running nthread

    Options for conf_dir:
    -c CONFIG_DIR, --config_dir=CONFIG_DIR
                        Pass in an alternative configuration dir. Default: /data/swall/conf

*参数解释*

.. note::

    role：指的是在swall.conf的node_role配置角色，只有配置了对应的role才能接收到命令

    target：通配符或者正则，通配符只支持*号，用来匹配具体的节点，主要去匹配swall.conf的node_name

    module.function：要执行的函数，例如sys.ping，有内置函数和自定义函数

    arguments：传递到module.function中的参数，支持位置参数和关键字参数

*选项解释*

.. note::

    --exclude：  需要从target刷选的列表中排除，支持通配符和正则

    --timeout：  设置超时

    --is_raw:    打印结果需要显示颜色

    --nthread：  需要多少个线程去执行任务，如果为1，代表一个swall接收到的任务只会在一个线程中执行


**下面演示一些功能函数的使用**

查看swall通讯是否正常::

    [root@swall1 ~]# swall ctl server "*"  sys.ping --timeout=10
    ####################
    [server] xyz_sa_server_192.168.8.190 : 1
    [server] xyz_sa_server_192.168.8.191 : 1
    [server] xyz_sa_server_192.168.8.195 : 1
    [server] xyz_sa_server_192.168.8.198 : 1
    [server] xyz_sa_server_192.168.8.203 : 1
    [server] xyz_sa_server_192.168.8.180 : 1
    ####################
    一共执行了[6]个

    
拷贝文件到远程::

    [root@swall1 ~]# swall ctl server "*"  sys.copy /etc/hosts /tmp/xx_hosts --timeout=10
    ####################
    [server] xyz_sa_server_192.168.8.190 : 1
    [server] xyz_sa_server_192.168.8.191 : 1
    [server] xyz_sa_server_192.168.8.195 : 1
    [server] xyz_sa_server_192.168.8.198 : 1
    [server] xyz_sa_server_192.168.8.203 : 1
    [server] xyz_sa_server_192.168.8.180 : 1
    ####################
    一共执行了[6]个
    [root@swall1 ~]# swall ctl server "*"  sys.copy /etc/hosts /tmp/xx_hosts ret_type=full --timeout=10
    ####################
    [server] xyz_sa_server_192.168.8.190 : /tmp/xx_hosts
    [server] xyz_sa_server_192.168.8.191 : /tmp/xx_hosts
    [server] xyz_sa_server_192.168.8.195 : /tmp/xx_hosts
    [server] xyz_sa_server_192.168.8.198 : /tmp/xx_hosts
    [server] xyz_sa_server_192.168.8.203 : /tmp/xx_hosts
    [server] xyz_sa_server_192.168.8.180 : /tmp/xx_hosts
    ####################
    一共执行了[6]个
    [root@swall1 ~]#

从远程拷贝文件到当前::

    [root@swall1 ~]# swall ctl server "xyz_sa_server_192.168.8.190"  sys.get /etc/services /tmp/xxx_service
    ####################
    [server] xyz_sa_server_192.168.8.190 : /tmp/xxx_service
    ####################
    一共执行了[1]个
    [root@swall1 ~]#


执行shell命令::

    [root@swall1 ~]# swall ctl server "xyz_sa_server_192.168.8.190"  cmd.call 'df -h | grep data'
    ####################
    [server] xyz_sa_server_192.168.8.190 : {'pid': 5329, 'retcode': 0, 'stderr': None, 'stdout': '/dev/sda5              73G   15G   55G  21% /data'}
    ####################

    [root@swall1 ~]# swall ctl server "xyz_sa_server_192.168.8.190"  cmd.call 'df -h | grep data' ret_type=stdout
    ####################
    [server] xyz_sa_server_192.168.8.190 : /dev/sda5              73G   15G   55G  21% /data
    ####################
    一共执行了[1]个
    [root@swall1 ~]#

.. raw:: html

    <style> .red {color:red} </style>

.. role:: red

.. note::

    *调用模块的时候如果不知道怎么使用模块，不知道传什么参数，怎么办？*

    :red:`答：每个函数后面加上 help参数都会打印这个函数使用说明`

    ::

        [root@swall1 ~]# swall ctl server "xyz_sa_server_192.168.8.190"  sys.copy help
        ####################
        [server] xyz_sa_server_192.168.8.190 :
            def copy(*args, **kwargs) -> 拷贝文件到远程 可以增加一个ret_type=full，支持返回文件名
            @param args list:支持位置参数，例如 sys.copy /etc/src.tar.gz /tmp/src.tar.gz ret_type=full
            @param kwargs dict:支持关键字参数，例如sys.copy local_path=/etc/src.tar.gz remote_path=/tmp/src.tar.gz
            @return int:1 if success else 0
        ####################
        一共执行了[1]个


.. note::

    *需要查看摸个模块的函数列表，怎么办？*

    :red:`答：提供了一个sys.funcs函数可以解决这个问题，需要输入想要查看的模块名称（不带后缀）`

    ::

        [root@swall1 ~]# swall ctl server "xyz_sa_server_192.168.8.190"  sys.funcs sys
        ####################
        [server] xyz_sa_server_192.168.8.190 : ('sys.rsync_module', 'sys.get', 'sys.job_info', 'sys.exprs', 'sys.copy', 'sys.ping', 'sys.reload_env', 'sys.funcs', 'sys.roles', 'sys.reload_node', 'sys.reload_module')
        ####################
        一共执行了[1]个
        [root@swall1 ~]#



如果写好了模块并且存放如当前节点的/module/{role}，这里的{role}对应你要同步的角色，/module/common是所有角色公用的模块，现在为server同步模块如下::

    [root@swall1 ~]# swall ctl server "xyz_sa_server_192.168.8.190"  sys.rsync_module
    ####################
    [server] xyz_sa_server_192.168.8.190 : 1
    ####################
    一共执行了[1]个

:red:`支持同步个别模块，多个需要用逗号分隔`::

    [root@swall1 ~]# swall ctl server "xyz_sa_server_192.168.8.190"  sys.rsync_module server_tools.py
    ####################
    [server] xyz_sa_server_192.168.8.190 : 1
    ####################
    一共执行了[1]个
    [root@swall1 ~]#



swall高级用法
------------------------

swall提供一些内置变量，使用在参数中，在真正执行的时候会被替换，查看当前系统支持的“系统变量”::

    [root@swall1 ~]# swall ctl server "xyz_sa_server_192.168.8.190"  sys.get_env
    ####################
    [server] xyz_sa_server_192.168.8.190 : ('node', 'ip', 'role')

支持node，ip、role这三个系统变量，使用的时候需要加大括号，如{node}、{ip}，查看系统变量的具体值如下::

    [root@swall1 bin]# swall ctl server "*"  sys.exprs "role:{role},ip:{ip},node:{node}"
    ####################
    [server] xyz_sa_server_192.168.8.190 : role:server,ip:192.168.8.190,node:xyz_sa_server_192.168.8.190
    [server] xyz_sa_server_192.168.8.191 : role:server,ip:192.168.8.191,node:xyz_sa_server_192.168.8.191
    [server] xyz_sa_server_192.168.8.195 : role:server,ip:192.168.8.195,node:xyz_sa_server_192.168.8.195
    [server] xyz_sa_server_192.168.8.198 : role:server,ip:192.168.8.198,node:xyz_sa_server_192.168.8.198
    [server] xyz_sa_server_192.168.8.203 : role:server,ip:192.168.8.203,node:xyz_sa_server_192.168.8.203
    [server] xyz_sa_server_192.168.8.180 : role:server,ip:192.168.8.180,node:xyz_sa_server_192.168.8.180
    ####################
    一共执行了[6]个
    [root@swall1 bin]#

什么场景下使用这些系统变量呢？
例如其他节点获取配置的时候，一般情况下，如果你不加系统变量，获取到当前节点的文件是同一个路径，你根本区分不出来，如下::
    
    [root@swall1 bin]# swall ctl server "*"  sys.get /etc/hosts /tmp/
    ####################
    [server] xyz_sa_server_192.168.8.190 : /etc/hosts
    [server] xyz_sa_server_192.168.8.191 : /etc/hosts
    [server] xyz_sa_server_192.168.8.195 : /etc/hosts
    [server] xyz_sa_server_192.168.8.198 : /etc/hosts
    [server] xyz_sa_server_192.168.8.203 : /etc/hosts
    [server] xyz_sa_server_192.168.8.205 : /etc/hosts
    ####################
    一共执行了[6]个
    [root@swall1 bin]#

这里就有一个问题了，所有获取的文件路径都是/etc/hosts，区分不出是那个节点的文件，如果使用系统变量，就不一样了::

    [root@swall1 bin]# swall ctl server "*"  sys.get /etc/hosts /tmp/hosts.{node}
    ####################
    [server] xyz_sa_server_192.168.8.190 : /tmp/hosts.xyz_sa_server_192.168.8.190
    [server] xyz_sa_server_192.168.8.191 : /tmp/hosts.xyz_sa_server_192.168.8.191
    [server] xyz_sa_server_192.168.8.195 : /tmp/hosts.xyz_sa_server_192.168.8.195
    [server] xyz_sa_server_192.168.8.198 : /tmp/hosts.xyz_sa_server_192.168.8.198
    [server] xyz_sa_server_192.168.8.203 : /tmp/hosts.xyz_sa_server_192.168.8.203
    [server] xyz_sa_server_192.168.8.205 : /tmp/hosts.xyz_sa_server_192.168.8.205
    ####################
    一共执行了[6]个
    [root@swall1 bin]#


还有一种场景，在游戏运维中，针对一机多服，假设游戏有/data/xyz_sa_600,/data/xyz_sa_601,/data/xyz_sa_700三个程序，对应三个game的节点，节点名称就是目录名。
如果我要拷贝文件到/data/xyz_sa_600,/data/xyz_sa_601,/data/xyz_sa_700各个目录下，用swall的系统变量替换就很容易解决::
    
    [root@swall1 bin]# swall ctl game "*"  sys.copy /etc/services /data/{node}/ ret_type=full
    ####################
    [game] xyz_sa_600 : /data/xyz_sa_600/services
    [game] xyz_sa_601 : /data/xyz_sa_601/services
    [game] xyz_sa_700 : /data/xyz_sa_700/services
    ####################
    一共执行了[3]个
    [root@swall1 bin]#


.. _swall: https://github.com/lufeng4828/swall.git
.. _zookeeper: http://archive.cloudera.com/cdh5/cdh/5/zookeeper-3.4.5-cdh5.0.0.tar.gz
.. _jdk7: http://www.oracle.com/technetwork/java/javase/downloads/jdk7-downloads-1880260.html
