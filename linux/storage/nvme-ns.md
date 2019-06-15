# NVME-Cli

## Namespace Rescan

利用nvme cli指令可以完成识别namespace功能。但仅仅在kernel 4.9版本上才支持。部分linux LTS发行版kernel过低，不支持。但可以通过升级kernel来解决。参考链接[NVMe: Allow user initiated rescan](https://git.isee.biz/linux-kernel/linux-imx/commit/9ec3bb2f994bda9c8817856fdcbfaebe8f62fbd3)

```bash
nvme ns-rescan /dev/nvme0
```

参考[nvme-cli issues #177](https://github.com/linux-nvme/nvme-cli/issues/177)通过reset-controller和rescan-controller同样可以完成ns的动态识别。但是reset会暂停当前admin和IO。有一定的IO连续性影响。如果调用syscall，可以通过NVME_IOCTL_RESCAN or NVME_IOCTL_RESET。如果支持subsystem，也可以使用NVME_IOCTL_SUBSYS_RESET。

### 参考NVM-Express-1.3c

> 7.3.2 Controller Level Reset
>
> There are five primary Controller Level Reset mechanisms:
>
> - NVM Subsystem Reset;
> - Conventional Reset (PCI Express Hot, Warm, or Cold reset);
> - PCI Express transaction layer Data Link Down status;
> - Function Level Reset (PCI reset); and
> - Controller Reset (CC.EN transitions from ‘1’ to ‘0’).
>
> When any of the above resets occur, the following actions are performed:
>
> - The controller stops processing any outstanding Admin or I/O commands;
> - All I/O Submission Queues are deleted;
> - All I/O Completion Queues are deleted;
> - The controller is brought to an Idle state. When this is complete, CSTS.RDY is cleared to ‘0’; and
> - The Admin Queue registers (AQA, ASQ, or ACQ) are not reset as part of a controller reset. All
other controller registers defined in section 3 and internal controller state are reset.

### Reset/Rescan controller

```bash
echo 1 > /sys/class/nvme/<nvmectl>/rescan_controller
echo 1 > /sys/class/nvme/<nvmectl>/reset_controller
```

## Query Namespace Identity ID

```bash
nvme id-ns /dev/nvme0 -n 0x1
```

