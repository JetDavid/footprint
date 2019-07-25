# pseudo-terminals in container

## Concept

A pseudoterminal (sometimes abbreviated "pty") is a pair of virtual character devices that provide a bidirectional communication channel.

- 一端为master，一端为slave.

![Overview of a pseudo-terminal](http://rachid.koucha.free.fr/tech_corner/pty_pdip_figure_2.jpg)

- 其中一端的输入都会发送至另一端的输出
- 应用场景：such as network login services (ssh(1), rlogin(1), telnet(1)), terminal emulators, script(1), screen(1), and expect(1).

![Description of a telnet session](http://rachid.koucha.free.fr/tech_corner/pty_pdip_figure_4.jpg)

- 存在2种为终端标准: BSD and System V. SUSv1 standardized a pseudoterminal API based on the System V API, and this API should be employed in all new programs that use pseudoterminals. linux2种方式均兼容。自2.6.4版本kernel之后，BSD-Style废弃，但可以在kernel中配置。现在更多应用采用的事UNIX 98方式（System V）

> 本节参考：
>
> - [pty(7) - Linux man page](https://linux.die.net/man/7/pty)
> - [Using pseudo-terminals (pty) to control interactive programs](http://rachid.koucha.free.fr/tech_corner/pty_pdip.html)

> 扩展阅读:
>
> [The TTY demystified](http://www.linusakesson.net/programming/tty/index.php)

## API for pseudo-termimals

- *posix_openpt()*: This call creates the master side of the pty. It opens the device /dev/ptmx to get the file descriptor belonging to the master side.
- *grantpt()*: The file descriptor got from posix_openpt() is passed to grantpt() to change the access rights on the slave side: the user identifier of the device is set to the user identifier of the calling process. The group is set to an unspecified value (e.g. "tty") and the access rights are set to crx--w----.
- *unlockpt()*: After grantpt(), the file descriptor is passed to unlockpt() to unlock the slave side.
- *ptsname()*: In order to be able to open the slave side, we need to get its file name through ptsname().

### Limitation of grantpt()

grantpt()在GLIBC中实现需要fork子进程对新建pty设备change ownership，并对子进程waitpid进行进程回收，在其中grantpt()实现上获取了sigchld信号. 因此grantpt的调用方不能抓取sigchld信号，否则会失败。可以在调用grantpt()前disable sigchld handler，在调用后恢复。

```c
/* Change the ownership and access permission of the slave pseudo
   terminal associated with the master pseudo terminal specified
   by FD.  */
int
grantpt (int fd)
{
[...]
  /* We have to use the helper program.  */
 helper:;

  pid_t pid = __fork ();
  if (pid == -1)
    goto cleanup;
  else if (pid == 0)
    {
      /* Disable core dumps.  */
      struct rlimit rl = { 0, 0 };
      __setrlimit (RLIMIT_CORE, &rl);

      /* We pass the master pseudo terminal as file descriptor PTY_FILENO.  */
      if (fd != PTY_FILENO)
        if (__dup2 (fd, PTY_FILENO) < 0)
          _exit (FAIL_EBADF);

#ifdef CLOSE_ALL_FDS
      CLOSE_ALL_FDS ();
#endif

      execle (_PATH_PT_CHOWN, basename (_PATH_PT_CHOWN), NULL, NULL);
      _exit (FAIL_EXEC);
    }
  else
    {
      int w;

      if (__waitpid (pid, &w, 0) == -1)
        goto cleanup;
[...]
```

## Pseudo-terminal used in container

- 在容器内部，程序是需要依托文件系统运行的
- 不单单是image基于的文件系统，还包括：proc, tmpfs, devpts, mqueue, sys.
- pseudo-terminal的运行依赖devpts文件系统。
- 在容器构建时，runtime需要将必要的文件系统mount到容器namespace中。并且这些mount会随着容器的生命周期结束而自动umount（kernel会完成这个事情）。

### Mount devpts filesystem

- 每一个针对devpts的挂载点都是不同的
- 每一个挂载点都来自于/dev/pts/ptmx (permission 0000)
- 挂载使用ptmx，需要做相应软连接或bind mount。 ptmxmode = 0666或者chmod = 0666
- pty创建数量限制：
  - kernel.pty.max = 4096 - global limit.
  - kernel.pty.reserve = 1024 - reserved for filesystems mounted from the initial mount namespace.
  - kernel.pty.nr - current count of ptys.
  - 数量限制可以通过mount option max=\<count\>来配置, 3.4以上 kernel sysctl可以控制kernel.pty.reserve来限制。 低于3.4 kernel使用kernel.pty.max来限制

#### Mount args in Container Specification

- PATH: /dev/pts
- TYPE: devpts
- FLAG: MS_NOEXEC,MS_NOSUID
- newinstance,ptmxmode=0666,mode=620,gid=5

已知问题： 如果在mount devpts到namespace中时，没有对pty ownership授予gid=5。会引发当新建pty伪终端，grantpt()失败，提示Failed to change pseudo terminal's permission: Operation not permitted

> 本节参考：
>
> - [kernel/filesystem - devpts](https://www.kernel.org/doc/Documentation/filesystems/devpts.txt)
> - [Container Specification - v1](https://github.com/opencontainers/runc/blob/master/libcontainer/SPEC.md)
> - [Failed to create a PTY. Operation not permitted on CentOS7 #462](https://github.com/dw/mitogen/issues/462)

> 扩展阅读:
>
> - [Containers, pseudo TTYs, and backward compatibility](https://lwn.net/Articles/688809/)
