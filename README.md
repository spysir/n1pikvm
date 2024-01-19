# N1 安装 PiKVM
需要的物料有 n1 一个 ￥90，双公头 USB 数据线 ￥5，MS2109 芯片的采集卡 ￥35。
系统安装完成后，双公头USB线插靠近HDMI口那个USB口，MS2109采集卡插最边上的那个USB口（不靠近HDMI口那个），所有操作均在root账号下进行
## 1.安装Armbian
首先需要给 N1 安装 Armbian，这里推荐[amlogic-s9xxx-armbian](https://github.com/ophub/amlogic-s9xxx-armbian)，
已经降级过的N1，请使用【N1-ADB U盘启动】工具来开启N1从U盘启动，然后就可以输入armbian-install安装系统到EMMC【U盘要先刷入Armbian系统】
## 2.开启 otg 功能
安装完系统第一步肯定是更新
```
apt-get update
```
使用 dtc 工具，编译 n1 设备树文件(dtb) 为可修改的 dts 文件。
```
dtc -I dtb -O dts -o n1.dts /boot/dtb/amlogic/meson-gxl-s905d-phicomm-n1.dtb
```
在生成的 dts 文件中搜索（所生成的n1.dts文件在root目录下）
```
dr_mode = "host";
```
把 host 模式修改为 peripheral 模式，开启 n1 的 otg 功能并保存更改
```
peripheral
```
编译回设备树二进制
```
dtc -I dts -O dtb -o n1.dtb n1.dts
```
移动回 boot 目录
```
mv n1.dtb /boot/dtb/amlogic/
```
编辑 uboot 文件
```
nano /boot/uEnv.txt
```
修改原先的设备树为我们开启了 otg 功能的设备树并保存更改
```
FDT=/dtb/amlogic/n1.dtb
```
## 3.安装[fruity-pikvm](https://github.com/jacobbar/fruity-pikvm)
下载 fruity-pikvm 源码
```
git clone https://github.com/spysir/fruity-pikvm
```
执行 fruity-pikvm 安装脚本
```
cd fruity-pikvm && ./install.sh
```
重启 n1
```
reboot
```
## 4.配置 PiKVM

添加用户 请将fox修改为你的用户名并输入密码
```
kvmd-htpasswd set fox
```
删除默认用户admin
```
kvmd-htpasswd del admin
```
列出所有用户
```
kvmd-htpasswd list
```
重启服务，使修改立刻生效
```
systemctl restart kvmd kvmd-nginx
```
pikvm 访问地址是你 n1 的 ip 地址，默认的账户与密码皆是 admin

删除 /etc/kvmd/override.yaml 中所有内容，修改为以下内容，开启 pikvm 的 Wake on lan 功能，其中 mac 地址填写你被控电脑的 mac 地址，IP 可填可不填，不填时需要删除或者注释该行。注意缩进，否则服务无法启动
```
kvmd:  
    msd:  
        type: disabled
    gpio:    
        drivers:    
            wol_1:    
                type: wol    
                mac: ff:ff:ff:ff:ff:ff
                ip: 10.0.0.114
        scheme:    
            wol_1:    
                driver: wol_1    
                pin: 0    
                mode: output    
                switch: false    
        view:    
            header:      
                title: 网络唤醒
            table:    
                - ["#PC", "wol_1|唤醒"]    
```
重启 pikvm 服务
```
systemctl restart kvmd kvmd-nginx
```
此时你的 pikvm 已经可以正常工作，但是没有虚拟光驱功能，由于 n1 默认只有 8g emmc 存储，并且只有两个USB接口，推荐你购买或者自行扩容为 32G emmc 或者插入一张 sd 卡，然后修改 /etc/kvmd/override.yaml 中 msd 的 type 为 enabled 分割出一个新的分区并挂载到 /var/lib/kvmd/msd 挂载后重启即可使用虚拟光驱。
## 接口说明
靠近 N1 HDMI 接口的 USB 连接双公头 USB 连接线，远离 HDMI 的接口的 USB 连接 MS2109 采集卡。
## 一些总结
fruity-pikvm 的安装脚本几乎可以在任何设备上安装 pikvm，比如随身 WiFi 但可能无法使用虚拟光驱，并且随身 WiFi 还需要加上一块 USB 扩展板，好处是随身 WiFi 它可以使用 4G 上网，使用 frp 等工具穿透，可以在你远程把家里网络搞坏的时候连接修复，并且一块 18650 电池就可以当 ups 续航很久，大小也比 n1 小得多可以塞入机箱内部，安装的设备内存推荐至少大于 512m，内存的大小与控制的流畅度相关，如果设备无法 otg 还可以使用 Arduino + [open-ip-kvm](https://github.com/Nihiue/open-ip-kvm) 方案，更多玩法还等着你发掘。
