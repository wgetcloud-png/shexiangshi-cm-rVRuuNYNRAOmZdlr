
### 事件背景


起因是有开发人员报障，程序在发布后无法正常运行，一直处于在重启的状态。


一开始我以为是程序本身的问题，但在查看服务日志后，并未发现程序有任何错误。


在查看监控系统时，发现该服务器节点CPU 利用率达到了100%，难怪程序已经无法运行。并且，还发现有这种情况的节点不止一个，整个环境中有好几台服务器都是CPU 100%的情况


![](https://img2024.cnblogs.com/blog/2829682/202412/2829682-20241209135416663-1946687056.png)


### 一、查看进程


使用Top命令查看进程 ，可以看到CPU的使用率已经跑满。但在进程列表中却未发现有异常进程 。除有个别业务程序占用CPU较多，但关掉后情况并未改善。


![](https://img2024.cnblogs.com/blog/2829682/202412/2829682-20241209135558302-1191824810.png)


### 二、查看网络访问


此时，怀疑是机器被入侵了，因此通过下面命令查看网络连接的情况。



```


|  | netstat -an |grep ESTABLISHED |
| --- | --- |


```

在查看几台机器后，发现有问题的机器都有一个外网连接，如下所示。



```


|  | tcp        0      0 10.12.15.7:39410        86.107.101.103:7643     ESTABLISHED |
| --- | --- |
|  |  |
|  | 虽然每台机器连接的外网IP地址不同，但端口号统一都是 7643，并且查询地址后发现都是国外地址。 |
|  | 由于相关的服务器并没有国外的业务，因此可以确定被病毒入侵无疑了。 |


```

### 三、查看启动项


使用下面命令查看开机启动项



```


|  | systemctl list-unit-files |grep enabled |
| --- | --- |


```

在启动项中，发现有一个名为OOlmeN2R.service 的可疑服务，怀疑就是病毒。（注：该病毒在不同机器的服务名称皆不同，随机的。但特点是乱码，有大小写或数字。）



```


|  | auditd.service                                enabled |
| --- | --- |
|  | autovt@.service                               enabled |
|  | crond.service                                 enabled |
|  | docker.service                                enabled |
|  | OOlmeN2R.service                              enabled   <------- |
|  | rhel-autorelabel.service                      enabled |
|  | rhel-configure.service                        enabled |
|  | rhel-dmesg.service                            enabled |
|  | rhel-domainname.service                       enabled |
|  | rhel-import-state.service                     enabled |
|  | rhel-loadmodules.service                      enabled |
|  | rhel-readonly.service                         enabled |
|  | rsyslog.service                               enabled |
|  | sshd.service                                  enabled |


```

通过下面命令，查看服务的启动状态以及启动文件的存放位置。



```


|  | systemctl status OOlmeN2R.service |
| --- | --- |


```

接着，找到该启动文件，并查看文件内容。



```


|  | $ cat /usr/lib/systemd/system/OOlmeN2R.service |
| --- | --- |
|  | [Unit] |
|  | Description=service |
|  | After=network.target |
|  |  |
|  | [Service] |
|  | Type=simple |
|  | ExecStart=/bin/eWqAVtbn |
|  | RemainAfterExit=yes |
|  | Restart=always |
|  | RestartSec=60s |


```

可以看到，服务在启动时调用了一个/bin/eWqAVtbn 文件，这应该是就病毒的执行文件了。


### 四、清除病毒


在发现病毒文件后，现在我们可以开始来清除病毒了。


停止病毒服务



```


|  | systemctl stop OOlmeN2R.service |
| --- | --- |
|  | systemctl disable OOlmeN2R.service |


```

删除相关病毒文件



```


|  | rm /bin/eWqAVtbn   #删除执行文件 |
| --- | --- |
|  | rm /usr/lib/systemd/system/OOlmeN2R.service  # 删除启动文件 |


```

删除完成后，重启服务器。


完成上述步骤后，再次查看该网络链接，发现该链接已消失。同时，服务器CPU使用率恢复到正常状态 ，病毒被清除了。


### 总结


该病毒有可能是挖矿类的病毒，占用机器资源进行任务，因此导致CPU使用率暴涨。同时，病毒较为狡猾，具有以下特点：


1\.隐藏自己的进程，无法通过TOP命令来发现。
2\.加入开机启动项，保证重启服务器后依然会生效。
3\.文件名随机，在不同机器上都不一样，增大了排查难度。


目前，通过本文档记录的方法，可以有效清除病毒。已知经过处理后的机器未再出现重复中毒情况。


 本博客参考[milou加速器](https://xinminxuehui.org)。转载请注明出处！
