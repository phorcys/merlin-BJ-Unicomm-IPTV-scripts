merlin-BJ-Unicomm-IPTV-scripts
==================

修改自  [twelve17/merlin-meo-scripts](https://github.com/twelve17/merlin-meo-scripts)


北京联通单网线IPTV+桥接+第三方APP 梅林固件脚本

用于实现单网线连接桥接光猫，并实现IPTV和IPTV第三方app上网共存

本脚本在  R6300V2 固件版本`380.59_X6.6.1`测试通过.

## 物理连接

* 光猫LAN1接路由器WAN
* 其他电脑IPTV机顶盒随意连接任意LAN/WIFI

## 原理

* 北京联通IPTV机顶盒通过 特定vlan，访问 联通内部网络（外部ip，内部网络，具体区段参见 configs/custom/_net_config 的IPTV_NETS
* 光猫上IPTV有一个 IPOE通道，运行在 iptv的 vlan上，脚本通过光猫将此通道桥接到路由器wan口（vlan id 3961），dhcp获得ip地址（内网，10.208.x.x)
* IPTV机顶盒专用内网dns解析域名，通过iptables将机顶盒dns请求劫持到路由器53端口以支持第三方app(通过机顶盒mac地址判定）
* 对IPTV专用的  epg.iptv域名，指定转发到联通内网dns解析（210.13.31.253,210.13.31.254)
* 对于IPTV专用网段，配置静态路由，指定所有对该网段访问走 IPTV内网（vlan3964）
* 重新配置 igmpproxy和snooper，所有其他项目成功配置后，杀掉这两个进程，以新配置重新拉起

## 前提条件

* 单网线连接光猫LAN1 <-> 路由器Wan
* 光猫支持 VLAN绑定（大部分都支持，不过需要 管理员/root 权限)
* 光猫internet和iptv链接都设置为桥接模式，记录internet和 iptv的vlanid （我这里使用  3961,3964)
* 光猫 路由-> Vlan绑定  设置 光猫LAN1为 VLAN绑定模式 ，映射关系为:    3961/3961;3964/3964
* 路由器梅林固件，本地网络  IPTV  里，设置 “选择 ISP 设置档” 为手动模式，互联网VID填入  internet的vlanid（例如 3961）
   并开启 "启动组播路由 (IGMP Proxy)" 和 "开启高效组播转发 (IGMP Snooping )"两项
* 路由器启用 jffs

## 安装

* 将configs scripts两个目录放入路由器 /jffs 目录下
* 更改scripts和 configs的权限为755   chmod -r 755 /jffs/configs /jffs/scripts

## 配置  configs/custom/_net_config
* LAN_NET 本地局域网网段 默认  192.168.1.0/24
* VLAN_IFACE IPTV vlan设备 默认 vlan3964  ,根据iptv  vlanid 改动
* VLAN_VENDOR_CLASS IPTV机顶盒DHCP VENDOR_CLASS 默认为 EC6108v9 的，如有必要根据需要改动
* IPTV_BOX_MACS IPTV 机顶盒MAC地址 支持多个，空格分离，根据需要自己修改
 
## 配置  script/service-start

* 根据路由器型号和internet iptv vlanid 情况，配置各vlan对应 网口

## 配置 ss

* 如果路由器上没有ss支持， 注释掉 wan-start 和 nat-start中的对应行（文件尾部）


## 测试

* 全部配置完成以后可以重启路由器测试（脚本启动需要时间）
* robocfg show 观察各vlan是否正确划分
* ifconfig 观察 pppoe是否正确拨号成功, iptv vlan3964是否正确分配获得ip
* cat /tmp/syslog.log 观察各脚本是否正确执行
* ps 观察 igmpproxy和 snooper是否以新参数被正确拉起
* 连接电脑和机顶盒，检查是否可以正确使用


## 其他

* 如果需要无线网络支持 IPTV，配置 configs/custom/igmpproxy.conf, 打开无线网络的支持， 2.4g无线为 eth1，5g为eth2
* 无线IPTV严重影响普通无线使用，建议用一个旧的11n路由器，使用和主路由器不同频道，当作AP接到路由器lan口，专门供IPTV机顶盒访问
* IPTV网段的10.208.x.1网关经常ping不通，可以用 210.13.0.146/177 作为测试目标，或者 用IPTV机顶盒的网络测试功能
* 如需诊断，建议安装bootstrap，opkg安装 tcpdump以便分析
