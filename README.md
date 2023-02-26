# android-server-best-practice

- chroot: Root(Magisk) + Busybox + LinuxDeploy + BatteryChargeLimit
- proot: Termux + Termux:Api + Termux:Boot

## Termux

```shell
termux-change-repo # 切换镜像到USTC科大
termux-wake-unlock # 防冻结
termux-setup-storage # 申请存储权限

# 支持ssh和自动充电
pkg install vim cronie openssh termux-api
echo 'termux-battery-status' > check_battery.sh # 查看电量 需先安装termux-api-apk 并允许应用自启动
# adb 控制充电
pkg install android-tools
# vim auto_battery.sh
addr=$(ifconfig | grep 192.168 | awk '{print $2}')
adb start-server
adb connect $addr
var=`adb -s $addr shell dumpsys battery | grep level | cut -d ":" -f 2`
echo =========
echo battery-level $var $(date "+%Y-%m-%d %H:%M:%S")
if [ $var -le 65 ]; then
    adb -s $addr shell dumpsys battery reset
    echo "battery start charging"
elif [ $var -ge 75 ]; then
    adb -s $addr shell dumpsys battery unplug
    echo "battery end charge"
fi
adb disconnect $addr
adb kill-server
# cron
crontab -e
*/30 * * * * sh /data/data/com.termux/files/home/auto_battery.sh >> /data/data/com.termux/files/home/auto_battery.log
# Termux:Boot 启动项（手机启动时） # echo 'sshd' > ~/.bashrc
mkdir ~/.termux/boot/
echo '#!/data/data/com.termux/files/usr/bin/sh' > ~/.termux/boot/start-sshd
echo 'termux-wake-lock' >> ~/.termux/boot/start-sshd
echo 'crond' >> ~/.termux/boot/start-sshd
echo 'sshd' >> ~/.termux/boot/start-sshd

# environment
pkg install openjdk-17 # java -version
pkg install python # python --version

# linux
pkg install proot-distro # 管理 Termux 内的 Linux 发行版
proot-distro install debian
# login debian
cp /etc/apt/sources.list /etc/apt/sources.list_bak # 备份软件源
sed -i 's/deb.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list # 换源USTC科大
sed -i 's|security.debian.org/debian-security|mirrors.ustc.edu.cn/debian-security|g' /etc/apt/sources.list
apt update
apt upgrade
apt install sudo vim
# new user
useradd -m sysadmin
passwd sysadmin # draw W on T
visudo # add 'sysadmin ...' under root # su - sysadmin \n sudo whoami # 验证
# Ctrl+D logout
echo 'proot-distro login debian --user sysadmin' > ~/login_debian.sh # 登录 debian
# vim /etc/rc.local
# nohup xxx > ~/log/xxx.log &
```
