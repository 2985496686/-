# docker 核心原理

## 构建容器

### 1. 使用pivot_root 实现root文件系统的隔离

我们使用``namespace mount`` 创建了一个与主机文件系统隔离的容器进程，但是此时容器内的目录挂载是继承了主机目录挂载的。我们首先要将root 文件系统挂载到一个新的目录中。



通过``mount``无法实现``rootfs``的切换，因为我们无法使所有的进程停止使用当前``rootfs`` 。当我们使用``umount`` 卸载当前``rootfs``挂载后，进程已经无法正常提供``mount``操作支持我们挂载新的``rootfs`` 。但是可以采用另一个系统调用:``pivot_root``



**pivot_root**

```C
int pivot_root(const char *new_root, const char *put_old);
```



该函数首先将根挂载点移动到``put_old`` 目录下，然后``new_root`` 设置为新的根挂载点 。根挂载节点就会发生如下转移关系: ``/ -> put_old -> new_root`` 。第二次转移后就可以将``put_old umount``  掉。



man手册对该函数的说明：

> pivot_root()函数允许调用者切换到一个新的根文件系统，并将旧的根挂载点放置在new_root下的一个位置，从而可以随后卸载它。（它将所有具有旧根目录或当前工作目录的进程移动到新根，使得旧根挂载点可以更容易地卸载，从而释放用户。）

该函数使用较为复杂，需要注意下面几点：

• new_root和put_old必须是目录。

• new_root和put_old不能与当前根目录位于同一挂载上。

• put_old必须位于或位于new_root下；也就是说，将指向put_old的路径名后面添加一些非负数的"/.."后缀，应该得到与new_root相同的目录。

• new_root必须是一个挂载点的路径，但不能是"/"。一个尚未是挂载点的路径可以通过将该路径绑定到自身来转换为挂载点。

• new_root的父挂载的传播类型和当前根目录的父挂载的传播类型不能为MS_SHARED；类似地，如果put_old是一个现有的挂载点，则其传播类型不能为MS_SHARED。这些限制确保pivot_root()永远不会将任何更改传播到另一个挂载命名空间。

• 当前根目录必须是一个挂载点。





**go 代码实现**



```go
func pivotRoot(newroot string) error {
    
	putold := filepath.Join(newroot, "/.pivot_root")
	// 创建 rootfs/.pivot_root 存储old_root
	if err := os.Mkdir(putold, 0777); err != nil {
		return err
	}
	// 将newroot变为挂载点
	if err := syscall.Mount(
		newroot,
		newroot,
		"",
		syscall.MS_SLAVE|syscall.MS_BIND|syscall.MS_REC,
		"",
	); err != nil {
		return fmt.Errorf("Mount rootfs to itself error: %v", err)
	}
	// pivot_root 
	if err := syscall.PivotRoot(newroot, putold); err != nil {
		return fmt.Errorf("pivot_root :%v", err)
	}

	// 修改当前的工作目录
	if err := os.Chdir("/"); err != nil {
		return fmt.Errorf("chdir / %v", err)
	}
	putold = filepath.Join("/", ".pivot_root")
	if err := syscall.Unmount(putold, syscall.MNT_DETACH); err != nil {
		return fmt.Errorf("umount pivot_root dir %v", err)
	}
	return os.Remove(putold)
}

```





上面代码直接运行会报错：``invalid argument``



查看man手册可能报错原因:

>- ``EINVAL：new_root``不是一个挂载点。
>
>- ``EINVAL：put_old``不在或下面new_root。
>
>- ``EINVAL``：当前根目录不是挂载点（因为之前的``chroot(2)``）。
>
>- ``EINVAL``：当前根在``rootfs``（初始``ramfs``）挂载上；请参见NOTES。
>
>- ``EINVAL：new_root``处的挂载点或该挂载点的父挂载点具有MS_SHARED传播类型。
>
>- ``EINVAL：put_old``是一个挂载点，并且具有MS_SHARED传播类型。



错误定位到第五条。因为``Shared subtrees`` 机制，子``namespace`` 会继承主机的挂载点，挂载点的类型默认都是``shared`` ，可以参考文章: 

[Linux mount (第二部分 - Shared subtrees)](https://segmentfault.com/a/1190000006899213?utm_source=sf-similar-article)

在原先代码上增加如下代码：

```go
// 将namespace下的所有挂载点改为私有挂载点
if err := syscall.Mount("","/","",syscall.MS_PRIVATE|syscall.MS_REC,"",); err != nil {
	return fmt.Errorf("mount / private failed: %v", err)
}
```







**参考文章**

[Linux mount(第二部分 - Share subtrees)](https://segmentfault.com/a/1190000006899213?utm_source=sf-similar-article)

[C语言pivot_root 的Invalid argument错误解决方案](https://blog.csdn.net/qq_37857224/article/details/125421976)

[man手册](https://man7.org/linux/man-pages/man2/pivot_root.2.html)

[命名空间Go实现 - Mount](https://bingbig.github.io/topics/container/namespaces_in_go_mount.html#pivot-root)

[Linux mount 命令进阶](https://www.cnblogs.com/sparkdev/p/9045563.html)

<<自己动手写docker>>