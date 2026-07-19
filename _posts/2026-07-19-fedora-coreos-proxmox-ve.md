---
layout: post
title: "Fedora CoreOS 在 Proxmox VE 上点火与家庭娱乐组件部署"
date: 2026-07-19 20:35:00 +0800
tags: [Proxmox VE, Fedora CoreOS, Home Lab]
lang: zh
---

成功在Proxmox VE上点火Fedora CoreOS并能正常使用家庭娱乐组件，记录一下大致过程和踩坑点。

## 使用官方的Proxmox镜像启动

官方给Proxmox制作了相应的镜像，这样就不需要下载完整的ISO然后为如何把`ignition`传进去而焦头烂额了，不过这就需要在运行PVE的主机上操作，而这些命令有的看上去不是很好理解。

整体的逻辑其实很简单：

1. 把官方的`qcow2`文件导入到PVE的磁盘里。这是个即开即用磁盘镜像，默认创建10GB磁盘，内置`cloud-init`组件用来接受`ignition`数据。
2. 设置`cloud-init`，会在VM上添加一个`cloud-init`的CD-ROM。把`ignition`文件传递到`cloud-init`的`vendor-data`部分（因为`user-data``meta-data`都是有`cloud-init`的要求的，`ignition`很显然不是正常的`cloud-init`）
2. 启动后自动点火完成配置，可以开始使用。

具体的命令讲解如下，具体细节可[参见官网](https://docs.fedoraproject.org/en-US/fedora-coreos/provisioning-proxmoxve/)。

第一步：在Proxmox主机上创建存储区（Storage）并设定该存储区的属性。

此处是在`/var/coreos`创建，用来存储`images``snippets`（`ignition`文件属于后者，需要将其放在`snippets`沐目录下）。

```bash
mkdir -p /var/coreos
pvesm add dir coreos --path /var/coreos --content images,snippets
```

第二步：创建VM机器。这个没啥好说的。

```bash
qm create ${VM_ID} --name ${NAME} --cores ${CPU} --memory ${MEMORY} \
    --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci
```

第三步：导入磁碟镜像。可以看到用了`import-from`选项，表明是基于下载的磁碟镜像建立一个新磁碟。默认创建一个10GB的硬盘。后续还跟着扩大磁盘的选项，注意这个是“扩大”而不是设置磁盘大小，而且必须带单位。`+10G`就是增加10GB。

```bash
qm set ${VM_ID} --scsi0 "${STORAGE}:0,import-from=/var/coreos/images/${QCOW}"
qm resize ${VM_ID} scsi0 +${DISK_SIZE}
```

第四步：设置`cloud-init`。`cloud-init`是放在一个CD-ROM设备中的。在对应的存储区中创建一个`cloudinit`的镜像，挂在IDE控制器上。随后指定`cloud-init`的`vendor-data`字段，从`snippets`目录中的`ign`读取。

```bash
qm set ${VM_ID} --ide2 ${STORAGE}:cloudinit
qm set ${VM_ID} --cicustom vendor=coreos:snippets/${IGN}
```

其余：设置主启动设备、以及防止在启动时自动更新，同时也包括启用串口使得交互界面更加清晰。

```bash
qm set ${VM_ID} --boot order=scsi0
qm set ${VM_ID} --serial0 socket --vga serial0 //这一步其实不是很建议添加，因为可能会造成一些兼容性问题
qm set ${VM_ID} --ciupgrade 0
```

## 接入GPU

因为是“家庭娱乐服务器”，GPU直通是必不可少的一环。

添加设备很简单，在“硬件”栏目中添加PCI设备即可。选择“Raw Device”，不要勾选“Primary GPU”。

其他的可以随便选，因为"All Function"也只有一个Function，而这个GPU并不用来展示开机界面，因此“ROM-bar”可以不选。

![添加PCI设备](<../images/Screenshot 2026-07-18 222145.png>)

我是在i440x机器下运行的，并且没有任何问题，[网络上一般建议换成q35](https://gist.github.com/patrix87/5cf66223549f7291351e83b5e09fbaae?permalink_comment_id=6188569)但是我发现换成q35后就没法启动了，具体的可以自己动尝试。

此外，如果发现一虚拟机开机内存就占满了，这也是正常现象。无论PVE显示内存使用情况如何，被分配到的内存都会被死死占用，Balloning在此处是没有效果的。在虚拟机中内存使用是正常的。

## 持久化数据的方案

即使是CoreOS，因为不是纯粹的业务处理节点，因此是需要一个方案来存储持久化数据的。

一开始我使用的是NFS挂载的方式，但是这有一个致命问题：SQL在NFS上有严重的性能问题，结果就是我的servarr系列软件一个都打不开，当然也不会报错，因为读写没有问题一直在初始化数据，但是性能很差所以久久无法完成。

很显然不能什么都依赖NFS，兼容性问题太多了，NFS应该用来读写一些不可声明的非结构数据，而不是AppData类的数据，更不能是缓存和对读写要求极高的数据。

我也想过要不使用S3来做远程备份？不过这也太麻烦了。我只是需要一个可以持久化保存数据的容器就行了，暂时想不到这么多。

因此只能用Proxmox最基础的用法，创建一个新的磁盘。

这里有一个误区需要澄清，那就是Proxmox虽然什么东西都要绑定一个VM_ID或者CT_ID，但其实除了命名上有点联系外其余没有任何联系。删除的时候倒是会有影响，但是只需要解除Reference、删除时不勾选“Destroy unreferenced disks owned by guest”就可以毫发无损。即使创建的虚拟机的VMID和磁碟的VMID对不上也是可以插进控制器的。

以下是操作的步骤：

```bash
pvesm alloc <storage> <vmid> <filename> <size_in_bytes>
```

`<storage>`就是存储区（一般是`local-lvm`）。`<filename>`值得讲一下，它的命名方式必须是`vm-<vmid>-xxx`，例如`vm-114-homo514`。`<size_in_bytes>`可以带单位，例如`10G`。

整体意思就是“给`<vmid>`虚拟机在存储区`<storage>`创建一个新的大小为`<size_in_bytes>`的硬盘，名字叫`<filename>`”。

上一步完成后只是创建了磁盘，并没有让它插进机器。将磁碟插入VM的新`scsi`控制器中：

```bash
qm set <VMID> -scsi1 <storage>:<filename>
```

此处的`<storage>`和`<filename>`成为了识别磁碟的唯一标识。此外，`scsi`后必须紧跟一个非零数字，因为它是一个独立的硬盘，插在新的控制器上。

此后，可以直接使用`ignition`中定义的内容设置挂载点、格式化等等，因为它是外来设备且要求持久化存储，因此不要在点火文件里定义格式化，直接挂载就可以了。