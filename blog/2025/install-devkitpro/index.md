# 记一次在Debian上安装devkitPro

众所周知，由于某些原因devkitPro大部分情况下会因为无法连接而无法顺利安装，本文记录在Debian上安装devkitPro的方法。

首先，需要先设置wget，具体需要修改`/etc/wgetrc`，找到类似如下的行：

```conf
# https_proxy = http://xxxxxxxxx:nnnn/
# http_proxy = http://xxxxxxxxx:nnnn/
# ftp_proxy = http://xxxxxxxxx:nnnn/
```

取消注释改为自己可以连通devkitPro官方服务器的服务器地址即可。

紧接着需要跟着[devkitPro官方的指引](https://github.com/devkitPro/pacman/releases)执行下列命令

```sh
wget https://apt.devkitpro.org/install-devkitpro-pacman
chmod +x ./install-devkitpro-pacman
sudo ./install-devkitpro-pacman
```

在执行脚本中最后一步`apt-get install devkitpro-pacman`时很可能会出现无法更新pacman软件源的问题，此时需要去设置devkitPro的pacman。

找到`/opt/devkitpro/pacman`可以看到如下目录结构：

```text
bin  etc  include  lib  share  var
```

找到`./etc/pacman.conf`，将：

```conf
# XferCommand = /usr/bin/wget --passive-ftp -c -O %o %u
```

这一行取消注释，紧接着在shell中执行`dkp-pacman -Syu`即可。

## dkp-libs错误

在更新时有可能会出现类似下面的错误：

```text
error: dkp-libs: signature from "Dave Murphy (WinterMute) <davem@devkitpro.org>" is invalid
```

找到`./var/lib/pacman/`删除，接着找到`./etc/pacman.conf`中：

```conf
[dkp-libs]
Server = https://pkg.devkitpro.org/packages
```

注释掉然后执行`dkp-pacman -Sy`，紧接着在shell中执行

```sh
wget https://pkg.devkitpro.org/devkitpro-keyring.pkg.tar.xz
sudo dkp-pacman -U devkitpro-keyring.pkg.tar.xz
```

之后将之前`./etc/pacman.conf`中注释掉的行取消注释，然后再`dkp-pacman -Sy`就行。

## 参考

https://devkitpro.org/viewtopic.php?f=15&t=9234


------------------------------

2025年首发于CSDN