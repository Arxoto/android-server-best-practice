
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
cat > ~/.termux/boot/start-sshd << EOF
#!/data/data/com.termux/files/usr/bin/sh
termux-wake-lock
crond
sshd
proot-distro login ubuntu
EOF

# environment
pkg install openjdk-17 # java -version
pkg install python # python --version

# linux
pkg install proot-distro # 管理 Termux 内的 Linux 发行版
proot-distro install debian
# login debian
cp /etc/apt/sources.list /etc/apt/sources.list.backup # 备份软件源
sed -i 's/deb.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list # 换源USTC科大
sed -i 's|security.debian.org/debian-security|mirrors.ustc.edu.cn/debian-security|g' /etc/apt/sources.list
# or
deb https://mirrors.ustc.edu.cn/debian/ testing main contrib non-free
# deb-src https://mirrors.ustc.edu.cn/debian/ testing main contrib non-free
deb https://mirrors.ustc.edu.cn/debian/ testing-updates main contrib non-free
# deb-src https://mirrors.ustc.edu.cn/debian/ testing-updates main contrib non-free
deb https://mirrors.ustc.edu.cn/debian-security/ stable-security main non-free contrib
# deb-src https://mirrors.ustc.edu.cn/debian-security/ stable-security main non-free contrib

apt update
apt upgrade
apt install sudo vim
# new user
useradd -m sysadmin
passwd sysadmin # draw W on T
visudo # add 'sysadmin ...' under root # su - sysadmin \n sudo whoami # 验证
# Ctrl+D logout
echo 'proot-distro login debian --user sysadmin' > ~/login_debian_sysadmin.sh # 登录 debian
echo 'proot-distro login ubuntu' > ~/login_ubuntu_root.sh # 登录 ubuntu
# vim /etc/rc.local
# nohup xxx > /var/log/wom/xxx.log 2>&1 &

# debian environment
apt install git # git --version
apt install gcc g++ make # ... -v # installed by default
apt install rustc # curl https://sh.rustup.rs -sSf | sh # rustc --version
apt install golang-go # go version
apt install openjdk-17 # java -version # installed by default
apt install nodejs npm # node -v && npm -v
apt install python # python --version

# build
cd /opt/
git config --global http.proxy "http://192.168.3.88:10811"
git config --global https.proxy "http://192.168.3.88:10811"
git config --global --get http.proxy
git config --global --unset http.proxy
git config --global --unset https.proxy

# piping-server
git clone https://github.com/nwtgck/piping-server-rust.git
cd piping-server-rust
cargo build --release
./target/release/piping-server --http-port=8888
nohup /opt/piping-server-rust/target/release/piping-server --http-port=8888 > /var/log/wom/piping-server.log 2>&1 &
kill -9 $(ps -ef | grep piping-server | grep -v grep | awk '{print $2}')

# piping-ui-web
git clone https://github.com/nwtgck/piping-ui-web.git
cd piping-ui-web
npm ci
PIPING_SERVER_URLS='["http://192.168.3.5:8888"]' npm run build
PIPING_SERVER_URLS='["http://localhost:8888"]' npm run build
serve -s dist # npm install -g serve
nohup serve -s /opt/piping-ui-web/dist/ > /var/log/wom/piping-ui-web.log 2>&1 &
kill -9 $(ps -ef | grep piping-ui-web | grep -v grep | awk '{print $2}')
http://192.168.3.5:3000

python -m http.server 9000
cd snapdrop
node server # snapdrop server
serve -s client # snapdrop client
# 需要解决局域网ssl证书的问题
# 思路 每个终端导入证书

# ================================================================================
# 首先使用以下命令生成一个证书密钥对 key.pem 和 cert.pem，它将有效期约10年（准确地说是3650天）
openssl req -newkey rsa:2048 -new -nodes -x509 -days 3650 -keyout key.pem -out cert.pem
openssl x509 -inform pem -in cert.pem -outform der -out cert.cer
# Country Name (2 letter code) [AU]:cn
# State or Province Name (full name) [Some-State]:zhejiang
# Locality Name (eg, city) []:hangzhou
# Organization Name (eg, company) [Internet Widgits Pty Ltd]:wom
# Organizational Unit Name (eg, section) []:dev
# Common Name (e.g. server FQDN or YOUR name) []:jm
# Email Address []:myphone@meizu.com
# ================================================================================
# 然后便可以起服务了 下面两个命令都可以，后者会自动打开默认浏览器运行页面
http-server -S -C cert.pem # npm install --global http-server

# 日志压缩
cat > /etc/logrotate.d/wom << EOF
/var/log/wom/wom-server.log {
  rotate 12
  weekly
  compress
  missingok
  notifempty
  minsize 200M
}
EOF

# 出现了springboot被kill的情况 网上说是没有swap 默认是有的 临时编写监控进程
cat > loop_check.sh << EOF
while true
do
echo =====================; date; free -m; ps -ef | grep wom-server | grep -v grep;
sleep 1;
done
EOF
nohup sh loop_check.sh > /var/log/wom/loop_check.log 2>&1 &
```
