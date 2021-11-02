# Linux 服务器内文件同步



**背景**： 需要在Linux 上建立两个文件夹，test1 和 test2, test1 作文文件存储目录，test2 作为test1 的备份存储目录，需要 test2 能够实时同步test1 的内容。



依赖组件：

- **inotify-tool**
- **shell**



## inotify-tool

​        Inotify一种强大的、细粒度的、异步文件系统监控机制，它满足各种各样的文件监控需要，可以监控文件系统的访问属性、读写属性、权限属性、删除创建、移动等操作，也就是可以监控文件发生的一切变化。inotify-tools是一个C库和一组命令行的工作提供Linux下inotify的简单接口。inotify-tools安装后会得到inotifywait和inotifywatch这两条命令：



### 一、安装

1.从内核和目录里面查看是否支持inotify,2.6.13以上版本内核都会支持

```shell
[root~] uname -r
3.10.0-693.2.2.el7.x86_64
```

2.检查是否有安装inotify 如果没有就安装

```
rpm -qa inotify-tools
yum install inotify-tools -y
```

### 二、 参数详解

安装完成后会生成两个命令
/usr/bin/inotifywait
/usr/bin/inotifywatch

- inotifywait命令可以用来收集有关文件访问信息，Linux发行版一般没有包括这个命令，需要安装inotify-tools，这个命令还需要将inotify支持编译入Linux内核，好在大多数Linux发行版都在内核中启用了inotify。
- inotifywatch命令用于收集关于被监视的文件系统的统计数据，包括每个 inotify 事件发生多少次



**inotifywait命令参数：**

- -m是要持续监视变化。
- -r使用递归形式监视目录。
- -q减少冗余信息，只打印出需要的信息。
- -e指定要监视的事件列表。
- –timefmt是指定时间的输出格式。
- FMT: # --timefmt ‘%y-%m-%d %H:%M’
- –format指定文件变化的详细信息。
  FMT: # --format ‘%T %f %e’
- –outfile将事件输出到指定文件，而不输出到屏幕
- -d|–daemon以守护进程方式后台运行(除了在后台运行外，与-m选项一样)



**可监听事件**

| 事件   | 描述                     |
| ------ | ------------------------ |
| access | 访问，读取文件           |
| modify | 修改，文件内容被修改     |
| attrib | 属性，文件元数据被修改   |
| move   | 移动，对文件进行移动操作 |
| create | 创建，生成新文件         |
| open   | 打开，对文件进行打开操作 |
| close  | 关闭，对文件进行关闭操作 |
| delete | 删除，文件被删除         |

三、使用示例
监听/tmp目录内所有文件和目录的"增删改"操作

```
/usr/bin/inotifywait -mrq -e 'create,delete,close_write,attrib,moved_to' --timefmt '%Y-%m-%d %H:%M' --format '%T %f %e' /tmp/

2018-05-21 19:53 xiaoke.txt CREATE
2018-05-21 19:53 xiaoke.txt ATTRIB
2018-05-21 19:53 xiaoke.txt CLOSE_WRITE,CLOSE
```



### 脚本代码

```shell
#/bin/bash
src=/test1/
dst=/test2
inotifywait -mrq -e modify,delete,create,attrib $src | while read file
do
  rsync -azHXA --delete $src $dst &> /dev/null
done
```

