---
title: 网络RAID技术DRBD
date: 2019-08-08 15:00:35
tags: Storage
categories: Storage
copyright: true
---
# DRBD简介
[官方文档](https://docs.linbit.com/docs/users-guide-8.4/)

>DRBD的全称为：`Distributed Replicated Block Device(DRBD)`分布式块设备复制，`DRBD`是由内核模块和相关脚本构成，用以构建高可用的集群。其实现方式是通过网络来镜像整个设备。可以把它看作是一种网络`RAID`。它允许用户在远程机器上建立一个本地块设备的实时镜像。

## DRBD是如何工作的
> `（DRBD Primary）`负责接收数据，把数据写到本地磁盘并发送给另一台主机`（DRBD Secondary）`。另一个主机再将数据存到自己的磁盘中。目前，`DRBD`每次只允许对一个节点进行读写访问，但这对于通常的故障切换高可用集群来说已经足够用了，有可能以后的版本支持两个节点进行读写存取。
> `DRBD`协议说明
> 1）数据一旦写入磁盘并发送到网络中就认为完成了写入操作。
> 2）收到接收确认就认为完成了写入操作。
> 3）收到写入确认就认为完成了写入操作。

## DRBD与HA的关系
>一个`DRBD`系统由两个节点构成，与`HA`集群类似，也有主节点和备用节点之分，在带有主要设备的节点上，应用程序和操作系统可以运行和访问`DRBD`设备`（/dev/drbd*）`。在主节点写入的数据通过`DRBD`设备存储到主节点的设备写入到备用节点的磁盘中。现在大部分的高可用性集群都会使用共享存储，而`DRBD`也可以作为一个共享存储设备，使用`DRBD`不需要太多的硬件的设备。因为它在`TCP/IP`网络中运行，所以，利用`DRBD`作为共享存储设备，节约很多成本，因为价格要比专用的存储网络便宜很多；其性能与稳定性方面也不错。

# DRBD工作原理
>`DRBD`是`linux`的内核的存储层中的一个分布式存储系统，可用使用`DRBD`在两台`Linux`服务器之间共享块设备，共享文件系统和数据。类似网络`RAID-1`，其工作的原理架构图如下：

![image.png](网络RAID技术DRBD/01.jpg)

# DRBD的特性
>分布式复制块设备（`DRBD`技术）是一种基于软件的，无共享，复制的存储解决方案，在服务器之间的对块设备（硬盘、分区、逻辑卷等）进行镜像。
>`DRBD`镜像数据的特性：
>1）实时性：当某个应用程序完成对数据的修改时，复制功能立即发生
>2）透明性：应用程序的数据存储在镜像块设备上是独立透明的，他们的数据在两个节点上都保存一份，因此，无论哪一台服务器宕机，都不会影响应用程序读取数据的操作，所以说是透明的。
>3）同步镜像和异步镜像：同步镜像表示当应用程序提交本地的写操作后，数据会同步写到两个节点上去；异步镜像表示当应用程序提交写操作后，只有当本地的节点上完成写操作后，另一个节点才可以完成写操作。

# DRBD的模式
>`DRBD`共有2种模式，一种是`DRBD的主从模式`，另一种是`DRBD的双主模式`
>**1）DRBD的主从模式：**
>   这种模式下，其中一个节点作为主节点，另一个节点作为从节点。其中主节点可以执行读、写操作；从节点不可以挂载文件系统，因此也不可以执行读写操作。在这种模式下，资源在任何时间只能存储在主节点上。这种模式可用在任何的文件系统（`ext3、ext4、xfs`）等。默认这种模式下，一旦主节点发生故障，从节点需要手工将资源进行转移，且主节点变成从节点和从节点变成主节点需要手动进行切换。不能自动进行转移，因此比较麻烦。
>为了解决手动将资源和节点进行转移，可以将`DRBD`做成高可用集群的资源代理`（RA）`，这样一旦其中的一个节点宕机，资源就会自动转移到另一个节点，从而保证服务的连续性。
>**2）DRBD的双主模式:**
>这是`DRBD8.0`之后的新特性，在双主模式下，任何资源在任何特定的时间都存在两个主节点。这种模式需要一个共享的集群文件系统，利用分布式的锁机制进行管理，如`GFS`和`OCFS2`。部署双主模式时，`DRBD`可以是负载均衡的集群，这就需要从两个并发的主节点中选取一个首选的访问数据。这种模式默认是禁用的，如果要是用的话必须在配置文件中进行申明。

# DRBD的复制模式
>`DRBD`的复制功能就是将应用程序提交的数据一份保存在本地节点，一份复制传输保存在另一份节点上。但是`DRBD`需要对传输的数据进行确认以便保证另一个节点的写操作完成，就需要用到`DRBD`的复制模式，`DRBD`有三种复制模式：

**1）协议A：异步复制协议**
>一旦本地磁盘写入已经完成，数据包已在发送队列中，则写被认为是完成的。在一个节点上发生故障时，可能发生数据丢失，因为被写入到远程节点上的数据可能仍在发送队列。尽管，在故障转移节点上的数据是一致的，但没有及时更新。这通常是用于地理上分开的节点。

**2）协议B：内存同步（半同步）复制协议**
>一旦本地磁盘写入已完成且复制数据包达到了对等节点则认为写在主节点上被认为是完成的。数据丢失可能发生在参加的两个节点同时故障的情况下，因为在传输中的数据可能不会被提交到磁盘。

**3）协议C：同步复制协议**
>只有在本地和远程节点的磁盘已经确认了写操作完成，写才被认为完成。没有任何数据丢失，所以这是一个集群节点的流行模式，但`I/O`吞吐量依赖于网络带宽。一般使用`协议C`，但选择`C协议`将影响流量，从而影响网络延迟。为了数据可靠性，我们在生产环境使用时须慎重选择使用哪一种协议。

# DRBD配置文件说明
```
DRBD的主配置文件为/etc/drbd.conf；为了管理的便捷性，
目前通常会将配置文件分成多个部分，且都保存至/etc/drbd.d目录中，主配置文件中仅使用"include"指令将这些配置文件片断整合起来。通常，/etc/drbd.d目录中的配置文件
为global_common.conf和所有以.res结尾的文件。其中global_common.conf中主要定义global段和common段，而每一个.res的文件用于定义一个资源。
  
在配置文件中，global段仅能出现一次，且如果所有的配置信息都保存至同一个配置文件中而不分开为多个文件的话，global段必须位于配置文件的最开始处。目前global段中
可以定义的参数仅有minor-count, dialog-refresh, disable-ip-verification和usage-count。
  
common段则用于定义被每一个资源默认继承的参数，可以在资源定义中使用的参数都可以在common段中定义。实际应用中，common段并非必须，但建议将多个资源共享的参数定
义为common段中的参数以降低配置文件的复杂度。
  
resource段则用于定义DRBD资源，每个资源通常定义在一个单独的位于/etc/drbd.d目录中的以.res结尾的文件中。资源在定义时必须为其命名，名字可以由非空白的ASCII字符
组成。每一个资源段的定义中至少要包含两个host子段，以定义此资源关联至的节点，其它参数均可以从common段或DRBD的默认中进行继承而无须定义。
  
DRBD配置文件
[root@drbd1 ~]# cat /etc/drbd.d/global_common.conf
global {
 usage-count yes;         //是否参加DRBD使用者统计，默认是参加
 # minor-count dialog-refresh disable-ip-verification  //这里是global可以使用的参数
 #minor-count：32         //从（设备）个数，取值范围1~255，默认值为32。该选项设定了允许定义的resource个数，当要定义的resource超过了此选项的设定时，需要重新载入DRBD内核模块。
 #disable-ip-verification：no   //是否禁用ip检查
   
}
common {
 protocol C;      //指定复制协议,复制协议共有三种，为协议A，B，C，默认协议为协议C
 handlers {       //该配置段用来定义一系列处理器，用来回应特定事件。
  # These are EXAMPLE handlers only.
  # They may have severe implications,
  # like hard resetting the node under certain circumstances.
  # Be careful when chosing your poison.
  # pri-on-incon-degr "/usr/lib/DRBD/notify-pri-on-incon-degr.sh; /usr/lib/DRBD/notify-emergency-reboot.sh; echo b > /proc/sysrq-trigger ; reboot -f";
  # pri-lost-after-sb "/usr/lib/DRBD/notify-pri-lost-after-sb.sh; /usr/lib/DRBD/notify-emergency-reboot.sh; echo b > /proc/sysrq-trigger ; reboot -f";
  # local-io-error "/usr/lib/DRBD/notify-io-error.sh; /usr/lib/DRBD/notify-emergency-shutdown.sh; echo o > /proc/sysrq-trigger ; halt -f";
  # fence-peer "/usr/lib/DRBD/crm-fence-peer.sh";
  # split-brain "/usr/lib/DRBD/notify-split-brain.sh root";
  # out-of-sync "/usr/lib/DRBD/notify-out-of-sync.sh root";
  # before-resync-target "/usr/lib/DRBD/snapshot-resync-target-lvm.sh -p 15 -- -c 16k";
  # after-resync-target /usr/lib/DRBD/unsnapshot-resync-target-lvm.sh;
 }
 startup {    //#DRBD同步时使用的验证方式和密码。该配置段用来更加精细地调节DRBD属性，它作用于配置节点在启动或重启时。常用选项有：
  # wfc-timeout degr-wfc-timeout outdated-wfc-timeout wait-after-sb
  wfc-timeout：  //该选项设定一个时间值，单位是秒。在启用DRBD块时，初始化脚本DRBD会阻塞启动进程的运行，直到对等节点的出现。该选项就是用来限制这个等待时间的，默认为0，即不限制，永远等待。
  degr-wfc-timeout：  //该选项也设定一个时间值，单位为秒。也是用于限制等待时间，只是作用的情形不同：它作用于一个降级集群（即那些只剩下一个节点的集群）在重启时的等待时间。
  outdated-wfc-timeout：  //同上，也是用来设定等待时间，单位为秒。它用于设定等待过期节点的时间
 }
 disk {
  # on-io-error fencing use-bmbv no-disk-barrier no-disk-flushes   //这里是disk段内可以定义的参数
  # no-disk-drain no-md-flushes max-bio-bvecs                      //这里是disk段内可以定义的参数
  on-io-error： detach      //选项：此选项设定了一个策略，如果底层设备向上层设备报告发生I/O错误，将按照该策略进行处理。有效的策略包括：
    detach      //发生I/O错误的节点将放弃底层设备，以diskless mode继续工作。在diskless mode下，只要还有网络连接，DRBD将从secondary node读写数据，而不需要failover（故障转移）。该策略会导致一定的损失，但好处也很明显，DRBD服务不会中断。官方推荐和默认策略。
    pass_on       //把I/O错误报告给上层设备。如果错误发生在primary节点，把它报告给文件系统，由上层设备处理这些错误（例如，它会导致文件系统以只读方式重新挂载），它可能会导致DRBD停止提供服务；如果发生在secondary节点，则忽略该错误（因为secondary节点没有上层设备可以报告）。该策略曾经是默认策略，但现在已被detach所取代。
    call-local-io-error   //调用预定义的本地local-io-error脚本进行处理。该策略需要在resource（或common）配置段的handlers部分，预定义一个相应的local-io-error命令调用。该策略完全由管理员通过local-io-error命令（或脚本）调用来控制如何处理I/O错误。
  fencing：               //该选项设定一个策略来避免split brain的状况。有效的策略包括：
    dont-care：  //默认策略。不采取任何隔离措施。
    resource-only：   //在此策略下，如果一个节点处于split brain状态，它将尝试隔离对端节点的磁盘。这个操作通过调用fence-peer处理器来实现。fence-peer处理器将通过其它通信路径到达对等节点，并在这个对等节点上调用DRBDadm outdate res命令
    resource-and-stonith：   //在此策略下，如果一个节点处于split brain状态，它将停止I/O操作，并调用fence-peer处理器。处理器通过其它通信路径到达对等节点，并在这个对等节点上调用DRBDadm outdate res命令。如果无法到达对等节点，它将向对等端发送关机命令。一旦问题解决，I/O操作将重新进行。如果处理器失败，你可以使用resume-io命令来重新开始I/O操作。
    
 }
 net {        //该配置段用来精细地调节DRBD的属性，网络相关的属性。常用的选项有：
  # sndbuf-size rcvbuf-size timeout connect-int ping-int ping-timeout max-buffers     //这里是net段内可以定义的参数
  # max-epoch-size ko-count allow-two-primaries cram-hmac-alg shared-secret      //这里是net段内可以定义的参数
  # after-sb-0pri after-sb-1pri after-sb-2pri data-integrity-alg no-tcp-cork      //这里是net段内可以定义的参数
  sndbuf-size：     //该选项用来调节TCP send buffer的大小，DRBD 8.2.7以前的版本，默认值为0，意味着自动调节大小；新版本的DRBD的默认值为128KiB。高吞吐量的网络（例如专用的千兆网卡，或负载均衡中绑定的连接）中，增加到512K比较合适，或者可以更高，但是最好不要超过2M。
  timeout：        //该选项设定一个时间值，单位为0.1秒。如果搭档节点没有在此时间内发来应答包，那么就认为搭档节点已经死亡，因此将断开这次TCP/IP连接。默认值为60，即6秒。该选项的值必须小于connect-int和ping-int的值。
  connect-int：     //如果无法立即连接上远程DRBD设备，系统将断续尝试连接。该选项设定的就是两次尝试间隔时间。单位为秒，默认值为10秒。
  ping-timeout：    //该选项设定一个时间值，单位是0.1秒。如果对端节点没有在此时间内应答keep-alive包，它将被认为已经死亡。默认值是500ms。
  max-buffers：     //该选项设定一个由DRBD分配的最大请求数，单位是页面大小（PAGE_SIZE），大多数系统中，页面大小为4KB。这些buffer用来存储那些即将写入磁盘的数据。最小值为32（即128KB）。这个值大一点好。
  max-epoch-size：    //该选项设定了两次write barriers之间最大的数据块数。如果选项的值小于10，将影响系统性能。大一点好
  ko-count：     //该选项设定一个值，把该选项设定的值 乘以 timeout设定的值，得到一个数字N，如果secondary节点没有在此时间内完成单次写请求，它将从集群中被移除（即，primary node进入StandAlong模式）。取值范围0~200，默认值为0，即禁用该功能。
  allow-two-primaries：   //这个是DRBD8.0及以后版本才支持的新特性，允许一个集群中有两个primary node。该模式需要特定文件系统的支撑，目前只有OCFS2和GFS可以，传统的ext3、ext4、xfs等都不行！
  cram-hmac-alg：    //该选项可以用来指定HMAC算法来启用对端节点授权。DRBD强烈建议启用对端点授权机制。可以指定/proc/crypto文件中识别的任一算法。必须在此指定算法，以明确启用对端节点授权机制，实现数据加密传输。
  shared-secret：    //该选项用来设定在对端节点授权中使用的密码，最长64个字符。
  data-integrity-alg：   //该选项设定内核支持的一个算法，用于网络上的用户数据的一致性校验。通常的数据一致性校验，由TCP/IP头中所包含的16位校验和来进行，而该选项可以使用内核所支持的任一算法。该功能默认关闭。
 }
 syncer {       //该配置段用来更加精细地调节服务的同步进程。常用选项有
  # rate after al-extents use-rle cpu-mask verify-alg csums-alg
  rate：    //设置同步时的速率，默认为250KB。默认的单位是KB/sec，也允许使用K、M和G，如40M。注意：syncer中的速率是以bytes，而不是bits来设定的。配置文件中的这个选项设置的速率是永久性的，但可使用下列命令临时地改变rate的值：DRBDsetup /dev/DRBDN syncer -r 100M。如果想重新恢复成drbd.conf配置文件中设定的速率，执行如下命令： DRBDadm adjust resource
  verify-alg：    //该选项指定一个用于在线校验的算法，内核一般都会支持md5、sha1和crc32c校验算法。在线校验默认关闭，必须在此选项设定参数，以明确启用在线设备校验。DRBD支持在线设备校验，它以一种高效的方式对不同节点的数据进行一致性校验。在线校验会影响CPU负载和使用，但影响比较轻微。DRBD 8.2.5及以后版本支持此功能。一旦启用了该功能，你就可以使用下列命令进行一个在线校验： DRBDadm verify resource。该命令对指定的resource进行检验，如果检测到有数据块没有同步，它会标记这些块，并往内核日志中写入一条信息。这个过程不会影响正在使用该设备的程序。
                    如果检测到未同步的块，当检验结束后，你就可以如下命令重新同步它们：DRBDadm disconnect resource   or   DRBDadm connetc resource
 }
}
  
common段是用来定义共享的资源参数，以减少资源定义的重复性。common段是非必须的。resource段一般为DRBD上每一个节点来定义其资源参数的。
资源配置文件详解 
  
  
[root@drbd1 ~]# cat /etc/drbd.d/web.res
resource web {       //web为资源名称
 on ha1.xsl.com {                       //on后面为节点的名称,有几个节点就有几个on段，这里是定义节点ha1.xsl.com上的资源
  device   /dev/DRBD0;       //定义DRBD虚拟块设备，这个设备事先不要格式化。
  disk /dev/sda6;        //定义存储磁盘为/dev/sda6,该分区创建完成之后就行了，不要进行格式化操作
  address 192.168.108.199:7789;      //定义DRBD监听的地址和端口，以便和对端进行通信
  meta-disk  internal;       //该参数有2个选项：internal和externally，其中internal表示将元数据和数据存储在同一个磁盘上；而externally表示将元数据和数据分开存储，元数据被放在另一个磁盘上。
 }
 on ha2.xsl.com {        //这里是定义节点ha2.xsl.com上的资源
  device /dev/DRBD0;
  disk /dev/sda6;
  address 192.168.108.201:7789;
  meta-disk internal;
 }
}
```


# DRBD安装配置（主从模式）
**环境说明**
>这里使用所用环境为`centos7.4`操作系统
>在下面的操作步骤中`# command`的命令表示主从节点服务器都需要执行。

| IP           | 主机名            | 角色说明 |
| ------------ | ----------------- | -------- |
| 192.168.1.31 | drbd1.cluster.com | 主服务器 |
| 192.168.1.32 | drbd2.cluster.com | 从服务器 |

## 具体步骤
1）两台机器上添加`DRBD`磁盘，进行分区，不做格式化，并在本地系统创建`/data`目录，不做挂载操作
```
在drbd1服务器上操作
[root@drbd1 ~]# lsblk 
......
sdb               8:16   0   10G  0 disk
[root@drbd1 ~]# fdisk /dev/sdb
依次输入"n->p->1->回车->回车-w"

在drbd2服务器上操作
[root@drbd2 ~]# lsblk 
......
sdb               8:16   0   10G  0 disk
[root@drbd2 ~]# fdisk /dev/sdb
依次输入"n->p->1->回车->回车-w"
```

2）两台服务器之间的防火墙要允许互相访问，这里测试关闭`selinux`和防火墙(两台服务器同样操作)
```
# systemctl stop firewalld
# setenforce 0     //临时性关闭；永久关闭需修改/etc/sysconfig/selinux的SELINUX为disable
```

3）`hosts`绑定（两台服务器同样操作）
```
# vim /etc/hosts
192.168.1.31    drbd1.cluster.com
192.168.1.32    drbd2.cluster.com
```

4）两台服务器进行时间同步（两台服务器同样操作）
```
# yum -y install ntpdate
# ntpdate -u asia.pool.ntp.org
```

5）`DRBD`安装配置（两台服务器同样操作）
```
这里使用yum方式安装
# rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
# yum -y install https://www.elrepo.org/elrepo-release-7.0-4.el7.elrepo.noarch.rpm
# yum -y install kmod-drbd84 drbd86-utils 

因为在执行yum -y install kmod-drbd84 drbd86-utils时，对内核kernel进行了update，要重新启动服务器，更新后的内核才会生效。
加载模块：
# reboot
# modprobe drbd
查看模块是否已添加上
# lsmod |grep drbd
drbd                  397041  0 
libcrc32c              12644  4 xfs,drbd,nf_nat,nf_conntrack
```

6）`DRBD`配置（两台服务器上同样操作）
```
# cat /etc/drbd.conf     //主配置文件
# You can find an example in  /usr/share/doc/drbd.../drbd.conf.example

include "drbd.d/global_common.conf";
include "drbd.d/*.res";

备份配置文件
# cp /etc/drbd.d/global_common.conf{,.bak} 
# vim /etc/drbd.d/global_common.conf    //全局配置文件
global {
	usage-count no;     #是否参加DRBD使用统计，默认为yes。官方统计drbd的装机量
}
common {
    protocol C;    #使用DRBD的同步协议
	handlers {
	}
	startup {
	}
	options {
	}
	disk {
                on-io-error detach;    #配置I/O错误处理策略为分离
	}
	net {
	}
    syncer {
                rate 30M;     #设置主备节点同步时的网络速率
    }
}

# vim /etc/drbd.d/r0.res    #创建资源
resource r0 {
    on drbd1.cluster.com {
        device    /dev/drbd0;    #drbd1服务器上的DRBD虚拟块设备，事先不要格式化
        disk      /dev/sdb1;
        address   192.168.1.31:7789;    #DRBD监听的地址和端口，端口可以自定义。
        meta-disk internal;
    }

    on drbd2.cluster.com {
        device    /dev/drbd0;    #drbd2服务器上的DRBD虚拟块设备，事先不要格式化
        disk      /dev/sdb1;
        address   192.168.1.32:7789;
        meta-disk internal;
    }
}
```

7）在两台服务器上分别启用`r0`资源（两台服务器上同样操作）
```
# drbdadm create-md r0
initializing activity log
initializing bitmap (320 KB) to all zero
Writing meta data...
New drbd meta data block successfully created.

# 启动drbd(注意：需要两边服务器同时启动方能生效)
[root@drbd1 ~]# systemctl start drbd
[root@drbd2 ~]# systemctl start drbd

# 查看状态（两台机器上都执行查看）
# cat /proc/drbd
version: 8.4.11-1 (api:1/proto:86-101)
GIT-hash: 66145a308421e9c124ec391a7848ac20203bb03c build by mockbuild@, 2018-11-03 01:26:55
 0: cs:Connected ro:Secondary/Secondary ds:Inconsistent/Inconsistent C r-----
    ns:0 nr:0 dw:0 dr:0 al:8 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:10484380

#由上面两台主机的DRBD状态查看结果里的ro:Secondary/Secondary表示两台主机的状态都是备机状态，ds是磁盘状态，
```

8）将`drbd1`服务器配置为`DRBD`的主节点，进行初始化设备同步
```
[root@drbd1 ~]# drbdsetup /dev/drbd0 primary --force
[root@drbd1 ~]# cat /proc/drbd 
version: 8.4.11-1 (api:1/proto:86-101)
GIT-hash: 66145a308421e9c124ec391a7848ac20203bb03c build by mockbuild@, 2018-11-03 01:26:55
 0: cs:SyncSource ro:Primary/Secondary ds:UpToDate/Inconsistent C r-----
    ns:2418688 nr:0 dw:0 dr:2420808 al:8 bm:0 lo:0 pe:1 ua:0 ap:0 ep:1 wo:f oos:8066716
	[===>................] sync'ed: 23.1% (7876/10236)M
	finish: 0:03:08 speed: 42,792 (37,776) K/sec
###同步完成时状态如下
[root@drbd1 ~]# cat /proc/drbd     #drbd1上查看
version: 8.4.11-1 (api:1/proto:86-101)
GIT-hash: 66145a308421e9c124ec391a7848ac20203bb03c build by mockbuild@, 2018-11-03 01:26:55
 0: cs:Connected ro:Primary/Secondary ds:UpToDate/UpToDate C r-----
    ns:10484380 nr:0 dw:0 dr:10486500 al:8 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0
[root@drbd2 ~]# cat /proc/drbd     #drbd2上查看
version: 8.4.11-1 (api:1/proto:86-101)
GIT-hash: 66145a308421e9c124ec391a7848ac20203bb03c build by mockbuild@, 2018-11-03 01:26:55
 0: cs:Connected ro:Secondary/Primary ds:UpToDate/UpToDate C r-----
    ns:0 nr:10484380 dw:10484380 dr:0 al:8 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0

#说明：
ro在主从服务器上分别显示 Primary/Secondary和Secondary/Primary
ds显示UpToDate/UpToDate 表示主从配置成功
```

9）挂载`DRBD`（`drbd1`主节点服务器上操作）
```
#先格式化/dev/drbd0
[root@drbd1 ~]# mkfs.ext4 /dev/drbd0

#创建挂载目录，然后进行挂载
[root@drbd1 ~]# mkdir /data
[root@drbd1 ~]# mount /dev/drbd0 /data
[root@drbd1 ~]# df -h
文件系统                 容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root   20G  3.7G   16G   20% /
devtmpfs                 471M     0  471M    0% /dev
tmpfs                    487M     0  487M    0% /dev/shm
tmpfs                    487M  7.1M  480M    2% /run
tmpfs                    487M     0  487M    0% /sys/fs/cgroup
/dev/sda1                497M  193M  305M   39% /boot
tmpfs                     98M     0   98M    0% /run/user/0
/dev/drbd0               9.8G   37M  9.2G    1% /data

特别注意：
从节点(Secondary)上不允许对DRBD设备进行任何操作，包括只读，所有的读写操作只能在主节点(Primary)上进行。
只有当主节点(Primary)挂掉时，从节点(Secondary)才能提示为主节点(Primary)。
```

## DRBD主备故障切换测试
>模拟主节点(`drbd1.cluster.com`）发生故障，从节点（`drbd2.cluster.com`）接管并提升为主节点。

1）主节点上操作
```
# 创建测试文件并取消挂载
[root@drbd1 ~]# cd /data/
[root@drbd1 data]# touch test01 test02 test03
[root@drbd1 data]# cd 
[root@drbd1 ~]# umount /data

[root@drbd1 ~]# drbdsetup /dev/drbd0 secondary        //将主节点设置为DRBD的备节点。在实际生产环境中，直接在(Secondary）节点上提权（即设置为主节点）即可。
[root@drbd1 ~]# cat /proc/drbd 
version: 8.4.11-1 (api:1/proto:86-101)
GIT-hash: 66145a308421e9c124ec391a7848ac20203bb03c build by mockbuild@, 2018-11-03 01:26:55
 0: cs:Connected ro:Secondary/Secondary ds:UpToDate/UpToDate C r-----
    ns:10783808 nr:0 dw:299428 dr:10488701 al:82 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0

注意：这里实际生产环境若Primary主节点宕机，在Secondary状态信息中ro的值会显示为Secondary/Unknown,只需要进行DRBD提权操作即可。
```

2）备节点上操作
```
[root@drbd2 ~]# drbdsetup /dev/drbd0 primary
[root@drbd2 ~]# cat /proc/drbd 
version: 8.4.11-1 (api:1/proto:86-101)
GIT-hash: 66145a308421e9c124ec391a7848ac20203bb03c build by mockbuild@, 2018-11-03 01:26:55
 0: cs:Connected ro:Primary/Secondary ds:UpToDate/UpToDate C r-----
    ns:0 nr:10783808 dw:10783808 dr:2120 al:8 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0

#挂载验证数据
[root@drbd2 ~]# mkdir /data
[root@drbd2 ~]# mount /dev/drbd0 /data
[root@drbd2 ~]# df -h
文件系统                 容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root   20G  3.7G   16G   20% /
devtmpfs                 471M     0  471M    0% /dev
tmpfs                    487M     0  487M    0% /dev/shm
tmpfs                    487M  7.1M  480M    2% /run
tmpfs                    487M     0  487M    0% /sys/fs/cgroup
/dev/sda1                497M  193M  305M   39% /boot
tmpfs                     98M     0   98M    0% /run/user/0
/dev/drbd0               9.8G   37M  9.2G    1% /data

[root@drbd2 ~]# cd /data/
[root@drbd2 data]# ls
test01  test02  test03
```

到此，`DRBD`的主从环境的部署工作已经完成。不过上面是记录的是主备手动切换，至于保证`DRBD`主从结构的智能切换，实现高可用，还需里用到`Keepalived`或`Heartbeat`来实现了（会在`DRBD`主端挂掉的情况下，自动切换从端为主端并自动挂载`/data`分区）

# 相关配置操作
## 资源的连接状态详细介绍
如何查看资源连接状态？
```
[root@drbd1 ~]# drbdadm cstate r0    #r0为资源名称
Connected
```
资源的连接状态；一个资源可能有一下连接状态的一种
```
 StandAlone 独立的：网络配置不可用；资源还没有被连接或是被管理断开（使用 drbdadm disconnect 命令），或是由于出现认证失败或是脑裂的情况 
 Disconnecting 断开：断开只是临时状态，下一个状态是StandAlone独立的 modprobe drbd
 Unconnected 悬空：是尝试连接前的临时状态，可能下一个状态为WFconnection和WFReportParams 
 Timeout 超时：与对等节点连接超时，也是临时状态，下一个状态为Unconected悬空 
 BrokerPipe：与对等节点连接丢失，也是临时状态，下一个状态为Unconected悬空 
 NetworkFailure：与对等节点推动连接后的临时状态，下一个状态为Unconected悬空 
 ProtocolError：与对等节点推动连接后的临时状态，下一个状态为Unconected悬空 
 TearDown 拆解：临时状态，对等节点关闭，下一个状态为Unconected悬空 
 WFConnection：等待和对等节点建立网络连接 
 WFReportParams：已经建立TCP连接，本节点等待从对等节点传来的第一个网络包 
 Connected 连接：DRBD已经建立连接，数据镜像现在可用，节点处于正常状态 
 StartingSyncS：完全同步，有管理员发起的刚刚开始同步，未来可能的状态为SyncSource或PausedSyncS 
 StartingSyncT：完全同步，有管理员发起的刚刚开始同步，下一状态为WFSyncUUID 
 WFBitMapS：部分同步刚刚开始，下一步可能的状态为SyncSource或PausedSyncS 
 WFBitMapT：部分同步刚刚开始，下一步可能的状态为WFSyncUUID 
 WFSyncUUID：同步即将开始，下一步可能的状态为SyncTarget或PausedSyncT 
 SyncSource：以本节点为同步源的同步正在进行 
 SyncTarget：以本节点为同步目标的同步正在进行 
 PausedSyncS：以本地节点是一个持续同步的源，但是目前同步已经暂停，可能是因为另外一个同步正在进行或是使用命令(drbdadm pause-sync)暂停了同步 
 PausedSyncT：以本地节点为持续同步的目标，但是目前同步已经暂停，这可以是因为另外一个同步正在进行或是使用命令(drbdadm pause-sync)暂停了同步 
 VerifyS：以本地节点为验证源的线上设备验证正在执行 
 VerifyT：以本地节点为验证目标的线上设备验证正在执行 
```

## 资源角色
查看资源角色命令
```
[root@drbd1 ~]# drbdadm role r0
Secondary/Primary
[root@drbd1 ~]# cat /proc/drbd 
version: 8.4.11-1 (api:1/proto:86-101)
GIT-hash: 66145a308421e9c124ec391a7848ac20203bb03c build by mockbuild@, 2018-11-03 01:26:55
 0: cs:Connected ro:Secondary/Primary ds:UpToDate/UpToDate C r-----
    ns:10783808 nr:24 dw:299452 dr:10488701 al:82 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0

注释：
Parimary 主：资源目前为主，并且可能正在被读取或写入，如果不是双主只会出现在两个节点中的其中一个节点上 
Secondary 次：资源目前为次，正常接收对等节点的更新 
Unknown 未知：资源角色目前未知，本地的资源不会出现这种状态
```

## 硬盘数据状态
查看硬盘状态命令
```
[root@drbd1 ~]# drbdadm dstate r0
UpToDate/UpToDate
```
本地和对等节点的硬盘有可能为下列状态之一：
```
 Diskless 无盘：本地没有块设备分配给DRBD使用，这表示没有可用的设备，或者使用drbdadm命令手工分离或是底层的I/O错误导致自动分离 
 Attaching：读取无数据时候的瞬间状态 
 Failed 失败：本地块设备报告I/O错误的下一个状态，其下一个状态为Diskless无盘 
 Negotiating：在已经连接的DRBD设置进行Attach读取无数据前的瞬间状态 
 Inconsistent：数据是不一致的，在两个节点上（初始的完全同步前）这种状态出现后立即创建一个新的资源。此外，在同步期间（同步目标）在一个节点上出现这种状态 
 Outdated：数据资源是一致的，但是已经过时 
 DUnknown：当对等节点网络连接不可用时出现这种状态 
 Consistent：一个没有连接的节点数据一致，当建立连接时，它决定数据是UpToDate或是Outdated 
 UpToDate：一致的最新的数据状态，这个状态为正常状态
```