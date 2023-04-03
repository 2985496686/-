
#   
# 传统虚拟机和docker的区别  
  
- 虚拟机是在物理资源层面实现的隔离，相对于虚拟机，Docker是你APP层面实现的隔离，并且省去了虚拟机操作系统（Guest OS）），从而节省了一部分的系统资源；  
  
- Docker守护进程可以直接与主操作系统进行通信，为各个Docker容器分配资源；它还可以将容器与主操作系统隔离，并将各个容器互相隔离。  
  
- 虚拟机启动需要数分钟，而Docker容器可以在数毫秒内启动。由于没有臃肿的从操作系统，Docker可以节省大量的磁盘空间以及其他系统资源。    
- 虚拟机与容器docker的区别，在于**vm多了一层guest OS，虚拟机的Hypervisor会对硬件资源也进行虚拟化，而容器Docker会直接使用宿主机的硬件资源**。  因为虚拟机对物理环境操作系统也进行了虚拟，所以它隔离的更加彻底。而docker只是通过namespce使容器与主机隔离，容器与容器隔离，但仍然使是直接使用主机的操作系统资源。  
  
  
- **容器只是运行在宿主机上的一种特殊的进程，那么多个容器之间使用的就还是同一个宿主机的操作系统内核。** 如果想要运行不同操作系统的环境容器，docker做不到。  
  
# docker隔离性  
  
1. **文件系统隔离：** 每个Docker容器都有自己的文件系统根目录，这意味着一个容器中的文件对其他容器不可见。这样就可以避免不同容器之间的文件冲突和干扰。  
  
2. **网络隔离：** 每个Docker容器都有自己的网络命名空间，因此每个容器可以具有不同的网络配置。这样就可以防止容器之间互相干扰，并且容器可以更好地控制容器访问外部网络的方式。  
  
3. **进程隔离：** 每个Docker容器都有自己的进程命名空间，这意味着一个容器中的进程对其他容器不可见。这样就可以防止容器之间的进程互相干扰，也可以更好地控制进程资源的使用。  
  
4. **用户隔离：** 每个Docker容器都有自己的用户命名空间，这意味着一个容器中的用户对其他容器不可见。这样就可以防止容器之间的用户干扰，从而提高容器的安全性。  
  
# run命令执行  
  
首先会在本地查找运行的镜像，找到镜像，以该镜像为模板生成容器实例运行，未找到该对象，会去dockerhub远程仓库查找该镜像，找到了会下载到本地，并以该镜像为模板生产容器实例运行，未找到报错。   
# docker常用命令  
  
```shell  
docker image      # 罗列本地镜像  TAG：镜像的版本号  
  
docker search  name     #从dockerhub仓库查找某一镜像  
  
docker pull name     #从dockerhub 拉去某一镜像  
  
docker system df     #查看docker数据卷的使用情况  
  
docker rmi name/id   #删除某一个镜像   
docker run [OPTIONS] IMAGE [COMMAND] [ARG...] //启动docker  
// options 说明  
//--name=容器名字   为容器指定一个启动名字  
//-d  后台运行容器，并返回容器id  
//-i  以交互模式运行容器，与-t配合使用  
//-t   为容器重新分配一个为输入终端  
//-p   小写p，指定端口号映射  
//-P   大写P，随机端口映射  
//启动交互式容器后，输入exit退出并关闭容器，ctrl+p+q退出终端，但不关闭  
  
  
docker rm 容器id //删已经关闭的容器  
docker stop id/name  //停止容器  
docker start/restart  id/name  //开启/重启 容器  
docker kill id/name  //强制停止容器  
  
docker logs id  //查看某一容器的日志  
docker top id  //查看某一容器运行的进程  
docker inspect id //查看某一容器内部运行细节  
  
  
docker exec -it id /bin/bash  //在容器中打开新的终端，并且可以启动新进程，用exit是退出当前终端，不会导致容器关闭  
  
docker attach  id//直接进入容器启动命令的终端，exit会让容器关闭  
  
docker cp id:容器上的文件路径 主机路径 //将容器上的文件拷贝到主机  
```  
  
## docker导入导出  
  
```shell  
docker export id > name.tar  //导出  
  
docker import name.tar :TAG  //导入  
```  
  
**遇到的坑**  
  
我尝试使用export导出一个redis容器，再通过import导入，但是出现了问题，导入的镜像无法启动。  
  
大概原因：export 导出（import 导入）是根据容器拿到的镜像，再导入时会丢失镜像所有的历史记录和元数据信息（即仅保存容器当时的快照状态），所以无法进行回滚操作。  
  
正确做法：我们可以将存有数据的redis容器commit成一个新的镜像，再通过save将镜像保存为tar文件，然后通过load加载tar文件，获得新镜像，使用该镜像启动的容器会保留原有数据。  
  
  
  
# docker 镜像加载原理  
  
## 分层文件系统  
docker 在下载镜像时是很明显的分层下载的，因为docker使用的是分层文件系统。  
  
**UnionFS（联合文件系统）**  
- docker的镜像是一层一层叠加形成的，我们可以在基础镜像上添加自己的配置后，打包成新的镜像，此时这个新镜像就挂载了原先镜像的文件目录，但有在此基础上新增了一些内容。  
- docker的镜像文件是不可写的，容器是可写的。  
  
## 镜像加载
**docker镜像加载**  
  
Docker的镜像实际上由一层一层的文件系统组成，这种层级的文件系统UnionFS（联合文件系统）。  
  
分为两个部分：  
  
-   `bootfs（boot file system）`：主要包含bootloader和kernel（Linux内核），bootloader主要是引导加载kernel，Linux刚启动时会加载bootfs文件系统，而在Docker镜像的最底层也是bootfs这一层，这与我们典型的Linux/Unix系统是一样的，包含boot加载器和内核。当boot加载完成之后，整个内核就都在内存中了，此时内存的使用权已由bootfs转交给内核，此时系统也会卸载bootfs。  
  
    即：系统启动时需要的引导加载，这个过程会需要一定时间。就是黑屏到开机之间的这么一个过程。电脑、虚拟机、Docker容器启动都需要的过程。在说回镜像，所以这一部分，无论是什么镜像都是公用的。  
  
  
-   `rootfs（root file system）`：rootfs在bootfs之上。包含的就是典型Linux系统中的`/dev`，`/proc`，`/bin`，`/etc`等标准目录和文件。rootfs就是各种不同的操作系统发行版，比如Ubuntu，Centos等等。  
  
    即：镜像启动之后的一个小的底层系统，这就是我们之前所说的，容器就是一个小的虚拟机环境，比如Ubuntu，Centos等，这个小的虚拟机环境就相当于rootfs。  
  
  
**`bootfs`和`rootfs`关系如下图：**  
  
![输入图片说明](https://raw.githubusercontent.com/2985496686/-/master/imgs/docker/2JnFSEnSqqP6V4YJ.png)


# docker容器数据卷

大致概念：映射本机目录与容器目录，实现两个数据的互通，防止重要数据的丢失。 

卷是文件目录的意思，通过设置容器数据卷，容器就会绕过联合文件系统，将数据存储到指定的文件目录中

**数据卷挂载命令**
在docker运行时加上-v参数 进行 文件的映射，-v参数如下：
```
-v   主机目录: 容器目录 ：参数
```

参数可以是 rw 或者 ro，-v也可以有多组

- rw 表示容器可读可写，也是默认参数
- ro 表示容器只能读


数据卷挂载后，若容器崩了，数据不会丢失。
**容器数据卷的继承**

在启动容器时，可以通过数据卷的继承来使用其他容器的目录映射关系。
```shell
docker run -it --volumnes-from 父类  镜像
```

继承后，父类数据卷容器挂掉，不会影响继承数据卷的新容器。


# 常用软件的下载
 
## mysql
1. 启动时需要设置密码。
```shell
-e MYSQL_ROOT_PASSWORD
```
3. 需要进行数据卷的映射。
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTE4MjQwOTMyMywyMzIwMTE0NTMsLTE3OD
E2MDczMTRdfQ==
-->