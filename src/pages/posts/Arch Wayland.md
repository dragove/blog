---
title: Arch Linux安装记录(UEFI + sway(wayland) + iwd + pipewire)
date: 2022-04-04
updated: 2022-04-05
layout: '../../layouts/Layout.astro'
---

## 安装前准备

1. U盘一个
2. [Arch Linux 镜像](https://archlinux.org/download/)
3. Ventoy(多镜像U盘制作工具)

## 制作安装U盘

1. 插入U盘到电脑，使用`sudo fdisk -l`查看u盘对应在文件系统中的文件(例如/dev/sdb)
2. 输入命令 `sudo ventoy -i /dev/sdb -g -s`，请确保备份了U盘内的内容
3. 挂载U盘 `sudo mount /dev/sdb1 /mnt`
4. 复制 Arch Linux 的 ISO 文件到U盘根目录`sudo cp ~/Downloads/archlinux.iso /mnt`
5. 使用 `md5sum /mnt/archlinux.iso` 校验复制后的 ISO 文件的md5，确保复制出来的文件的完整性
5. 备份当前系统的文件到U盘

## BIOS 设置

1. 关闭并重启电脑，按特定按键(不同厂家的电脑不一样)进入BIOS 界面
2. 部分电脑需要关闭 `security boot` 才能安装 Arch Linux，磁盘模式不要选择 `raid`，可能存在缺失 `raid` 驱动的问题
3. 在 `boot` 标签下选择启动优先级，把U盘放在第一位(如果你同时拥有usb2.0和3.0的接口，建议使用2.0接口，部分电脑在启动阶段只识别2.0接口的设备)
4. 如果步骤3中未能找到U盘，请尝试更换U盘(博主学疏才浅，不清楚原因)
5. 保存 BIOS 设置并重启，进入 ventoy 界面
6. ventoy 界面中选择 Arch Linux 的 ISO 回车进入，如果出现错误，请再次核对复制进去的文件的md5是否符合官方提供的值

## 开始安装

> 建议同时参考[官方的安装手册](https://wiki.archlinux.org/title/installation_guide)

### 初始化live CD状态

1. 输入 `systemctl stop reflector` 关闭 reflector，该软件有时会导致一些问题。
2. 输入 `rfkill unblock wifi` 来启用 wifi 设备(部分机型需要该命令)

### 连接到无线网络(有线网络一般可以忽略此步骤)

1. 输入 `iwctl` 进入到 iwd 的交互式命令行，该命令行可以用于连接 wifi，在此命令行下可以通过 `Tab` 按键补全
2. 输入 `station wlan0 show` 可以查看无线连接状态
3. 输入 `station wlan0 scan` 打开无线网络搜索，输入 `station wlan0 get-networks` 查看可用无线网络列表，例如我的无线网络名称为 dragove
4. 输入 `station wlan0 connect dragove` 连接到自己的无线网络，输入密码(密码输入过程不可见)回车连接
5. 输入 `exit` 或者按 `ctrl+c` 退出 iwd 交互命令行
6. 输入 `ping www.baidu.com` 可以查看网络是否连接成功

### 更新时间与分区

```bash
timedatectl set-ntp true # 更新系统时间为网络时间
fdisk -l                 # 或者使用 lsblk 查看分区状态
parted /dev/sdx          # 对sdx进行分区，如果是固态硬盘，设备名可能是 /dev/nvme?n?
mktable                  # 修改分区类型
gpt                      # 分区类型设置为 gpt
quit                     # 退出 parted

# 使用 cfdisk 分区(也可以使用其他分区工具)，建议至少分出根目录和efi分区，
# 可以考虑建立swap分区以及home分区
cfdisk /dev/sdx          
fdisk -l                 # 查看分区情况

# 假设我的设备为 /dev/sda 且有三个分区，分别用于根目录、efi 和 swap
mkfs.ext4 /dev/sda3      # 将 /dev/sda3设为ext4文件系统
mkfs.vfat /dev/sda1      # 将 efi 分区设置为fat文件系统
mkswap /dev/sda2         # 使用 swap 分区
swapon
```

### 挂载与安装

```bash
mount /dev/sda3 /mnt
mkdir /mnt/efi
mount /dev/sda1 /mnt/efi

# 修改镜像文件，搜索 China 关键字找到中国镜像源并复制到文件开头，优先使用中国镜像源
vim /etc/pacman.d/mirrorlist
# 开始安装，使用 zen 内核可以避免后期部分需求更换内核
pacstrap /mnt base base-devel linux-zen linux-zen-headers linux-firmware
# 安装部分常用软件
pacstrap /mnt iwd neovim
# 生成 fstab
genfstab -U /mnt >> /mnt/etc/fstab
# 检查 fstab
cat /mnt/etc/fstab
```

### 新系统初始化

```bash
arch-chroot /mnt # 切换到新系统内
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime # 设置上海时区
hwclock --systohc # 将系统时间写入硬件

# 编辑 locale 文件，将 en_US.UTF-8 和 zh_CN.UTF-8 前的注释符号'#'删掉
nvim /etc/locale.gen
# 生成 locale
locale-gen
# 配置系统 LANG 变量，大部分软件依赖该变量识别语言
echo 'LANG=en_US.UTF-8'  > /etc/locale.conf # 可以按照需求配置成中文，但是不建议使用中文

# 设置主机名字，按照自己喜好取名，建议使用纯英文，例如叫 arch
echo 'arch' > /etc/hostname
# 编辑 hosts
nvim /etc/hosts
### hosts 内容参考
127.0.0.1   localhost
::1         localhost
# use your own hostname
127.0.1.1   arch
### hosts 内容参考

# 安装微码，请根据自己的cpu选择
pacman -S intel-ucode # intel cpu
pacman -S amd-ucode   # amd cpu

# 安装系统引导程序
pacman -S grub efibootmgr
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg

# 设置root密码
passwd root

# 退出liveCD并重启
exit
umount -R /mnt
reboot
# 请在关机后拔出U盘
```

## 安装后的配置

```bash
# 启动无线网络支持
nvim /etc/iwd/main.conf
### 启用iwd内置dhcp客户端
[General]
EnableNetworkConfiguration=true
### 启用iwd内置dhcp客户端
systemctl enable iwd
systemctl enable systemd-resolved # DNS 支持
systemctl start systemd-resolved
systemctl start iwd
# 完成以上步骤后可以使用 iwctl 来连接网络了

# 安装 zsh 和配置文件 grml-zsh-config
pacman -S zsh grml-zsh-config

# 添加非 root 用户(建议添加，AUR不支持 root 用户安装，root 用户操作也不安全)
useradd -m -G wheel -s /bin/zsh dove # 使用 wheel 组可以用于后面的sudoers
passwd dove # 修改密码
EDITOR=nvim visudo # 修改sudoers文件
# 找到下面这行，取消注释，NOPASSWD 是可选的，取决于你是否在使用 sudo 命令时要输入密码
# %wheel ALL=(ALL) ALL NOPASSWD: ALL

# 之后就可以切换到普通用户来执行命令了
```

## 用户界面安装

```bash
# 安装一些常用字体
sudo pacman -S adobe-source-han-serif-cn-fonts wqy-zenhei
sudo pacman -S noto-fonts-cjk noto-fonts-emoji noto-fonts-extra
# 安装 sway 终端等软件，选择 kitty 是因为 kitty 是 wayland native 的
# sway-im 是 sway 支持输入法的版本，可以在 archlinuxcn 源或者 AUR 中找到
sudo pacman -S sway-im kitty waybar wl-clipboard \
    xorg-xwayland qt5-wayland glfw-wayland

# 安装音频服务
sudo pacman -S wireplumber pipewire-pulse pipewire-jack pamixer
```

使用 `xprop` 软件可以检测窗口是否运行在 Xwayland 上，
命令行运行xprop之后点击任何窗口，如果可以点击并且 `xprop` 有输出内容，则该窗口运行在X上

```bash
# 创建并编辑 sway 配置文件
cd ~/.config
mkdir sway
cd sway
cp /etc/sway/config .
nvim config
# 修改一下term变量到自己的终端

# 启动 sway
exec sway
# 如果安装了 nvidia 驱动，会导致 sway 报错，
# 如果是双显卡用户，可以使用以下命令
exec sway --unsupported-gpu
```

到这里，就已经成功进入到 sway 的界面里了，可以通过修改 .zprofile 来修改 zsh 启动配置

```bash
# zprofile
export QT_QPA_PLATFORM=wayland
export SDL_VIDEODRIVER=wayland
# 设置火狐浏览器使用 wayland
export MOZ_ENABLE_WAYLAND=1
export _JAVA_AWT_WM_NONREPARENTING=1
export EDITOR=nvim
export VISUAL=nvim

# 登陆后自动启动 sway
if [[ ! $DISPLAY && $XDG_VTNR -eq 1 ]]; then
  exec sway --unsupported-gpu
fi
```

此时我们还需要一个程序启动器，推荐一个叫做 `sirula` 的程序启动器，简洁快速。安装后在 sway 配置文件中修改 menu 变量即可使用

### Chromium 和 electron

```
cd ~/.config
nvim chromium-flags.conf
### 输入这个内容后chromium就会使用wayland
--enable-features=UseOzonePlatform
--ozone-platform=wayland
### 完成输入
# 对于electron也是同样的内容，文件名不同
nvim electron-flags.conf
# 如果程序使用的不是最新的electron，则需要额外的配置文件
# 例如 electron17-flags.conf
nvim electron17-flags.conf

# 如果遇到electron应用启动失败的问题，可以先移除这些文件使用 xwayland 运行
```

### Emacs

Emacs 本身是个 GTK 应用，在 `emacs 28` 这个版本里，emacs 支持了 native 编译选项，可以在 archlinuxcn 里使用最新的 emacs

```bash
sudo pacman -S emacs-native-comp-git
```

### 自动登陆

```bash
sudo mkdir /etc/systemd/system/getty@tty1.service.d/
sudo nvim /etc/systemd/system/getty@tty1.service.d/autologin.conf
### 输入以下内容
[Service]
ExecStart=
ExecStart=-/sbin/agetty -o '-p -f -- \\u' --noclear --autologin username - $TERM
### 输入内容结束
```

### 博主的配置以及参考内容

- [鸽子的配置](https://github.com/dragove/dotfiles)
- [Arch Linux Studio 安装手册](https://archlinuxstudio.github.io/ArchLinuxTutorial)
- [fosskers 的wayland教程](https://www.fosskers.ca/en/blog/wayland)
- 感谢 Arch Linux CN 群里的小伙伴的帮助


