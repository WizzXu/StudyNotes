
##### 1.查看当前系统内核版本
```
 uname -sr
```

##### 2.在 Ubuntu 16.04 中升级内核
要升级 Ubuntu 16.04 的内核，打开 http://kernel.ubuntu.com/~kernel-ppa/mainline/ 并选择列表中需要的版本（发布此文时最新内核是 4.13.8）。

接下来，根据你的系统架构下载 .deb 文件：

对于 64 位系统：

```
wget http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.13.8/linux-headers-4.13.8-041308_4.13.8-041308.201710180430_all.deb
wget http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.13.8/linux-headers-4.13.8-041308-generic_4.13.8-041308.201710180430_amd64.deb
wget http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.13.8/linux-image-4.13.8-041308-generic_4.13.8-041308.201710180430_amd64.deb
```

##### 3.安装内核
```
sudo dpkg -i *.deb
```

##### 4.重启查看当前内核版本，看最新内核是否被使用。


