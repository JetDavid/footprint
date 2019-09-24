# overlayFS for docker

## 1. What is the overlayFS

> OverlayFS is a type of union file system. It allows the user to overlay one file system on top of another. Changes are recorded in the upper file system, while the lower file system remains unmodified. This allows multiple users to share a file-system image, such as a container or a DVD-ROM, where the base image is on read-only media. Refer to the kernel file Documentation/filesystems/overlayfs.txt for additional information.

## 2. Recommand overlayFS for docker graph driver

From docker docs:

- overlay2 FS is the best, but several prerequisites were needed.
- overlay2 FS with kernel 4.0 or higher or CentOS(RHEL) using version 3.10.0-514 and above.
- Use overlay2 rather than overlay(not supported in docker-ee, and inefficient use of inode)
- d_type=true must be enabled when using xfs to be the backing filesystem for overlayFS. Otherwise there will be a fatal error occurred.

> OverlayFS is a modern union filesystem that is similar to AUFS, but faster and with a simpler implementation. Docker provides two storage drivers for OverlayFS: the original overlay, and the newer and more stable overlay2.
>
> If you use OverlayFS, use the overlay2 driver rather than the overlay driver, because it is more efficient in terms of inode utilization. To use the new driver, you need version 4.0 or higher of the Linux kernel, or RHEL or CentOS using version 3.10.0-514 and above.

### 2.1 Recommandations for recent release Linux distribution

From docker docs:

Linux distribution | Recommended storage drivers | Alternative drivers
------------------|-------------------------------|---------------------
Docker Engine - Community on Ubuntu | overlay2 or aufs (for Ubuntu 14.04 running on kernel 3.13) | overlay¹, devicemapper², zfs, vfs
Docker Engine - Community on Debian | overlay2 (Debian Stretch), aufs or devicemapper (older versions) | overlay¹, vfs
Docker Engine - Community on CentOS | overlay2 | overlay¹, devicemapper², zfs, vfs
Docker Engine - Community on Fedora | overlay2 | overlay¹, devicemapper², zfs, vfs

### 2.2 Recommandations for backing filesystems

From docker docs:

Storage driver | Supported backing filesystems
---------------|------------------------------
overlay2, overlay | xfs with ftype=1, ext4
aufs | xfs, ext4
devicemapper | direct-lvm
btrfs | btrfs
zfs | zfs
vfs | any filesystem

## 3. Why is d_type of backing filesystem necessary to overlayFS

From linux docs:

> overlayFS: The lower filesystem can be any filesystem supported by Linux and does not need to be writable.  The lower filesystem can even be another overlayfs.  The upper filesystem will normally be writable and if it is it must support the creation of trusted.* extended attributes, and must provide valid d_type in readdir responses, so NFS is not suitable.

What is the d_type:

> d_type is the term used in Linux kernel that stands for “directory entry type”. Directory entry is a data structure that Linux kernel used to describe the information of a directory on the filesystem. d_type is a field in that data structure which represents the type of an “file” the directory entry points to. It could a directory, a regular file, some special file such as a pipe, a char device, a socket file etc.
>
> d_type information was added to Linux kernel version 2.6.4. Since then Linux filesystems started to implement it over time. However still some filesystem don’t implement yet, some implement it in a optional way, i.e. it could be enabled or disabled depends on how the user creates the filesystem.

## 4. Prerequirement and Restrictions

From redhat docs:

- OverlayFS仅仅作为COW，对于持久化数据不建议。最好使用特别的volume来保存数据。
- OverlayFS配置而言，docker的默认行为即可：Only default Docker configuration can be used; that is, one level of overlay, one lowerdir, and both lower and upper levels are on the same file system.
- Only XFS is currently supported for use as a **lower layer** file system.
- OverlayFS的kernel ABI和userspace行为，目前还不稳定。在未来会改进.
- OverlayFS提供的是有限的POSIX std标准。应用使用后需要多做测试
- 对于xfs需要在创建时指定-n ftype=1或安装系统时--mkfsoptions=-n ftype=1来支持overlayFS
- 当在物理服务器上运行docker时，必须enable SELinux并开启SELinux的enforcing mode，而在容器里必须disable SELinux

From docker docs:

overlayFS 对POSIX standard支持不够充分，docker官方指出了具体接口：

> open(2): OverlayFS only implements a subset of the POSIX standards. This can result in certain OverlayFS operations breaking POSIX standards. One such operation is the copy-up operation. Suppose that your application calls fd1=open("foo", O_RDONLY) and then fd2=open("foo", O_RDWR). In this case, your application expects fd1 and fd2 to refer to the same file. However, due to a copy-up operation that occurs after the second calling to open(2), the descriptors refer to different files. The fd1 continues to reference the file in the image (lowerdir) and the fd2 references the file in the container (upperdir). A workaround for this is to touch the files which causes the copy-up operation to happen. All subsequent open(2) operations regardless of read-only or read-write access mode reference the file in the container (upperdir).
>
> yum is known to be affected unless the yum-plugin-ovl package is installed. If the yum-plugin-ovl package is not available in your distribution such as RHEL/CentOS prior to 6.8 or 7.2, you may need to run touch /var/lib/rpm/* before running yum install. This package implements the touch workaround referenced above for yum.
>
> rename(2): OverlayFS does not fully support the rename(2) system call. Your application needs to detect its failure and fall back to a “copy and unlink” strategy.

总而言之：

1. 当有2个操作对同一个文件做open调用，RONLY和RW需要特别注意，建议先touch文件，再执行
2. 当rename文件时，需要特别注意copy unlink strategy的错误结果。言外之意，再做rename操作时，需要关注错误，推测可能需要重试或全拷贝文件来解决。
3. 同样的问题在AUFS上也存在。

## 5. Differences of Other Graph drivers

驱动 | 联合文件系统 | 写时复制 | 内核 | SELinux | 适合生产环境 | 速度 | 存储空间占用
-----|-----------|---------|-----|---------|-------------|-----|------------
AUFS | 是 | 文件级别 | 不支持 | 支持 | 推荐 | 快 | 小
Device Mapper | 不是 | 块级别 | 2.6.9 | 支持 | 有限推荐 | 慢 | 较大
Btrfs | 不是 | 块级别 | 2.6.29 | 不支持 | 有限推荐 | 较快 | 较小
OverlayFS | 是 | 文件级别 | 3.18 | 不支持 | 不推荐 | 快 | 小
ZFS | 不是 | 块级别 | 不支持 | 不支持 | 不推荐 | 较快 | 较小
VFS | 不是 | 不支持 | 2.4 | 支持 | 不推荐 | 很慢 | 大

![image](http://dockone.io/uploads/article/20190702/747be895d53add6ea9ddf868f95ff8ec.jpg)

## 6. Conclusion

overlay2 FS是docker官方推荐，但有诸多限制：

- 需要注意在更高版本的kernel上使用overlayFS. kernel >= 4.0 or CentOS(RHEL) kernel >= 3.10.0-514
- 需要注意XFS必须带有d_type=true来使用，ext4已经默认配置
- 需要注意overlayFS 对 POSIX standard支持不够充分。
- 建议在container host上enforce SELinux功能。但目前经验来看，不开启未发生问题。
- 建议XFS作为lower layer来使用。该部分需要测试
- 建议关键数据不要再COW layer（各种graph driver）上试图保存，而是单独使用额外数据卷来做持久化。

## Reference

- [About storage drivers](https://docs.docker.com/storage/storagedriver/)
- [What is d_type and why Docker overlayfs need it](https://blog.csdn.net/liukuan73/article/details/77986139)
- [Overlay Filesystem](https://github.com/torvalds/linux/blob/master/Documentation/filesystems/overlayfs.txt)
- [REDHAT -- CHAPTER 21. FILE SYSTEMS](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/7.2_release_notes/technology-preview-file_systems)
- [Centos 7 fails to remove files or directories which are created on build of the image using overlay or overlay2. #27358](https://github.com/moby/moby/issues/27358)
- [Docker五种存储驱动原理及应用场景和性能测试对比](http://dockone.io/article/1513)
- [Use the OverlayFS storage driver](https://docs.docker.com/storage/storagedriver/overlayfs-driver/)
- [Docker storage drivers](https://docs.docker.com/storage/storagedriver/select-storage-driver/)
