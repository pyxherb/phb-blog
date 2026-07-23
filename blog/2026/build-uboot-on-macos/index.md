# MacOS编译U-Boot

## 第一步：克隆U-Boot仓库

此步略过，直接`git clone <U-Boot仓库链接> --depth=1`即可，然后`cd <U-Boot目录>`设置当前路径到U-Boot路径。

## 第二步：通过homebrew安装必要的依赖

```sh
brew tap messense/macos-cross-toolchains
brew trust messense/macos-cross-toolchains
brew install aarch64-unknown-linux-gnu
brew install bash # 注意：MacOS自带的bash版本过旧会导致shopt -s lastpipe无法使用
```

## 第三步：修改Makefile避免错误

定位到Makefile的这一行：

```Makefile
undefine MK_ARCH
```

然后将其注释掉即可。

## 第四步：配置U-Boot

```sh
make qemu_arm64_defconfig
```

## 第五步：安装OpenSSL并复制到U-Boot目录解决依赖问题

```sh
brew install openssl
# 注意：如果安装后在下一步编译没有出现错误则不需要复制到U-Boot目录
cp -R /opt/homebrew/Cellar/openssl@3/<OpenSSL软件包版本>/include/openssl include
```

## 第六步：编译U-Boot

```sh
export PATH="/opt/homebrew/bin:/opt/homebrew/sbin:$PATH" # 新版MacOS+新版homebrew不会覆盖默认bash，要正常编译必须临时覆盖环境变量！
make CROSS_COMPILE=aarch64-linux-gnu- # 此时可能会让你配置，注意不要选择启用汇编版的memset和memcpy函数实现
```

至此，U-Boot编译完成。

## 参考

https://blog.csdn.net/weixin_44221345/article/details/134785081
