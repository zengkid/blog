+++
title = 'PVE安装设置RouterOS指南'
date = 2023-05-21T12:52:29+08:00
draft = false
+++

### PVE下安装routeros
* 设置磁盘镜像可保存在Local中
  > 登录PVE的web管理界面
  > DataCenter -> local(pve) -> Storage, 选中local双击打开编辑对话框
  > 在Content列表中选中Disk
  
    ![image](https://github.com/zengkid/blog/assets/3382739/bc225d27-2863-4e11-977b-306d993b8daf)
  > 保存该设置

* 通过SSH登录PVE，账号需要有创建VM的权限
* 编写自动安装脚本, 此脚本来源于官方
  > vi routeros-install.sh, 然后将以下内容贴上去
  ```
  #!/bin/bash

  #vars
  version="nil"
  vmID="nil"

  echo "############## Start of Script ##############

  ## Checking if temp dir is available..."
  if [ -d /root/temp ] 
  then
      echo "-- Directory exists!"
  else
      echo "-- Creating temp dir!"
      mkdir /root/temp
  fi
  # Ask user for version
  echo "## Preparing for image download and VM creation!"
  read -p "Please input CHR version to deploy (6.38.2, 6.40.1, etc):" version
  # Check if image is available and download if needed
  if [ -f /root/temp/chr-$version.img ] 
  then
      echo "-- CHR image is available."
  else
      echo "-- Downloading CHR $version image file."
      cd  /root/temp
      echo "---------------------------------------------------------------------------"
      wget https://download.mikrotik.com/routeros/$version/chr-$version.img.zip
      unzip chr-$version.img.zip
      echo "---------------------------------------------------------------------------"
  fi
  # List already existing VM's and ask for vmID
  echo "== Printing list of VM's on this hypervisor!"
  qm list
  echo ""
  read -p "Please Enter free vm ID to use:" vmID
  echo ""
  # Create storage dir for VM if needed.
  if [ -d /var/lib/vz/images/$vmID ] 
  then
      echo "-- VM Directory exists! Ideally try another vm ID!"
      read -p "Please Enter free vm ID to use:" vmID
  else
      echo "-- Creating VM image dir!"
      mkdir /var/lib/vz/images/$vmID
  fi
  # Creating qcow2 image for CHR.
  echo "-- Converting image to qcow2 format "
  qemu-img convert \
      -f raw \
      -O qcow2 \
      /root/temp/chr-$version.img \
      /var/lib/vz/images/$vmID/vm-$vmID-disk-1.qcow2
  # Creating VM
  echo "-- Creating new CHR VM"
  qm create $vmID \
    --name chr-$version \
    --net0 virtio,bridge=vmbr0 \
    --bootdisk virtio0 \
    --ostype l26 \
    --memory 256 \
    --onboot no \
    --sockets 1 \
    --cores 1 \
    --virtio0 local:$vmID/vm-$vmID-disk-1.qcow2
  echo "############## End of Script ##############"
  ```
  > 按ESC后输入 `:x` 保存退出
  > 添加运行权限 `chmod +x routeros-install.sh`
* 执行安装脚本  
  > ./routeros-install.sh, 然后安装脚本会提示你要安装哪个版本的routeros，目前最新的是7.9,所以可以输入7.9
  
    ![image](https://github.com/zengkid/blog/assets/3382739/f5c9d1aa-1da4-472d-a41d-897b191a0bb4)
  > 然后提示需要创建的虚拟机编号，输入一个不在列表中的虚拟机ID `110`
  
    ![image](https://github.com/zengkid/blog/assets/3382739/6fde058e-b81e-4a30-8d44-4109516f30e5)
  > 根据提示会结束安装
       
     ![image](https://github.com/zengkid/blog/assets/3382739/93f49e32-32d7-4103-9000-44b8613dc37a)
  
 * 修改虚拟机配置
   > 安装后在PVE看到新创建的routeros虚拟机，不过内存和硬盘都比较小,而且只有一块网卡
    
      ![image](https://github.com/zengkid/blog/assets/3382739/b6fd2e84-1fa8-40c8-83d3-44c42d436d3c)

   > 修改内存和硬盘大小 `vi /etc/pve/qemu-server/110.conf`，根据个人喜好修改内存和硬盘大小
    
      ![image](https://github.com/zengkid/blog/assets/3382739/d0f14aca-3113-4033-bdd0-91f4331ea3d7)

   > 添加新网卡
     
     Hardware -> Add -> Network Device

     ![image](https://github.com/zengkid/blog/assets/3382739/21246dd1-7cc5-4af8-8738-5939096266aa)
     
     选择桥接网卡

     ![image](https://github.com/zengkid/blog/assets/3382739/45812de3-3f4b-4482-883c-0c893fb4bf5e)
     
      **注意：由于我在PVE的第一网卡为管理口了，所以只能设置为LAN口，所以我在配置routeros时候，桥接口调整了以下，net0的桥接口为vmbr1，而net1的桥接口为vmbr0** 最终配置如下图

     ![image](https://github.com/zengkid/blog/assets/3382739/e89d5141-3290-4e5d-af22-4bc8a731aeef)
    
* 安装教程到此为止，下一步进入routeros的设置教程   

#### routeros初始设置
* 连接routeros
  1. 下载[winbox](ttps://mikrotik.com/download)
  2. 打开winbox,可以在网上邻居看到routeros的连接，因为还没有配置任何ip地址，所以看到的ip地址是 `0.0.0.0`或者是 `fe80::e80c:1aff:fe83:a97b%19`选择其中一个作为连接，然后在Login输入默认账号 `admin`,密码为空

        ![image](https://github.com/zengkid/blog/assets/3382739/7fc95d33-9fa1-49a9-9fc2-dddff93fb661)
  3. 进入winbox后提示修改密码

        ![image](https://github.com/zengkid/blog/assets/3382739/3c81535b-0137-41f4-ac55-a2351d23d7d7)
  4. 输入新密码后点击`Change Now`

        ![image](https://github.com/zengkid/blog/assets/3382739/b39606da-2f61-4864-92d3-d6e053bbed87)
* routeros快速设置
  1. 点击左侧菜单的Quick Set

        ![image](https://github.com/zengkid/blog/assets/3382739/c256c522-bfb5-4bd2-8261-8bf16018afd8)
  2. `Configuartion` -> `Mode` 选择 `Router`打开快速设置窗口
  3. `Internet` -> `Address Acquisition` 选择 `PPPoE`
  4. 在`PPoE User`和`PPoe Password`d分别输入拨号账号和密码
  5. Local Network设置
     1. IP Address输入内网IP，此IP将作为内网的网关,例如192.168.8.1
     2. Netmask ->内网IP的掩码,如255.255.255.0
     3. 将`Bridge All LAN Ports`，`DHCP Server`和`NAT`都打上勾
     4. DHCP Server Range可以输入客户端分配的IP地址池，可以输入`192.168.8.100-192.168.8.200`
     5. System -> Router Identity修改软路由名称，可以保留不变
     6. 点击`apply`确认
     7. 然后就可以观察到拨号的PPPoE status为连接状态，此时要注意DHCP Server Range有没有被重置回来或者不是我们之前填写的IP段，如果有问题的话请修改回来
  6. DNS设置，IP -> DNS打开DNS设置窗口，在Servers输入114.114.114.114和1.1.1.1或者其他你熟悉的DNS服务器，将Allow remote Requests打勾

      ![image](https://github.com/zengkid/blog/assets/3382739/34b52ba9-8863-4316-bade-7303746782b3)
      ![image](https://github.com/zengkid/blog/assets/3382739/4bd172da-b42e-4bd6-9a30-e90dda79f137)

  7.最基本的上网设置已经完成，其他客户端现在可以连接上routeros上网了  

* 高级设置
  1. 端口转发,远程桌面3389示例
  ```
  /ip firewall nat
  add chain=dstnat protocol=tcp dst-port=3389 in-interface-list=WAN  action=dst-nat to-address=192.168.8.8 to-port=3389 
  ```
   routeros的端口映射后，在外网能正常访问，但是在内网通过公网ip或者域名访问时候就连接不上了。不过我们在内网一般不会直接通过外网IP访问内网的服务，这是我们直接通过routeros的DNS static映射一个域名服务给具体的设备。
   例如我们要访问内网群晖的服务一般是https://qunhui.mydomain.com:5000,做完5000端口转发后，然后我们打开DNS的设置, IP -> DNS -> Static打开静态ip域名映射
   Name输入qunhui.mydomain.com, Address输入群晖的内网IP地址，点击OK确认。
   这样设置后内外网都可以通过域名访问群晖了，不过如果转发端口dst-port和to-port不一样的话，就需要根据内外网来访问了。
   
  2. 以下防火墙命令开发ping命令响应和开放9291,80和22端口，如果不想开放这些端口可以不加
  ```
  /ip firewall filter
  add chain=input connection-state=established,related action=accept comment="accept established,related";
  add chain=input connection-state=invalid action=drop;
  add chain=input in-interface-list=WAN protocol=icmp action=accept comment="allow ICMP";
  add chain=input in-interface-list=WAN protocol=tcp port=8291 action=accept comment="allow Winbox";
  add chain=input in-interface-list=WAN protocol=tcp port=80 action=accept comment="allow webfig";
  add chain=input in-interface-list=WAN protocol=tcp port=22 action=accept comment="allow SSH";
  add chain=input in-interface-list=WAN action=drop comment="block everything else";
  ```
  不过开放22端口的后果很严重，日志不断地接收到远程尝试登录，所以最好不要开放22端口，如果实在又远程ssh登录的需求，可以尝试修改SSH的端口号,例如修改为2200
  ```
    /ip service set ssh port=2200
    /ip/firewall/filter add chain=input in-interface-list=WAN protocol=tcp port=2200 action=accept comment="allow SSH";
  ```

  3. 新增用户并且删除默认的`admin`用户
    ```
    /user add name=myadmin password=mypassword group=full
    /user remove admin
    ```
    
网上提供防火墙的例子,感觉不错。
 ```
  # RouterOS 7.9
  #
  # model = RB4011iGS+5HacQ2HnD
  /ip firewall address-list
  add address=123.45.67.89 comment=Giga3 list=vps
  add address=23.237.231.0/24 comment=SatTV list=vps
  /ip firewall filter
  add action=fasttrack-connection chain=forward comment="defconf: fasttrack" \
      connection-state=established,related hw-offload=yes
  add action=accept chain=forward comment=\
      "defconf: accept established,related, untracked" connection-state=\
      established,related,new,untracked
  add action=accept chain=input comment=\
      "defconf: accept established,related,untracked" connection-state=\
      established,related,untracked
  add action=accept chain=forward comment="Allow all from local" \
      in-interface-list=!WAN
  add action=accept chain=input comment="Accept all from local " \
      in-interface-list=LAN
  add action=accept chain=input comment="RB4011 accept SSH" dst-port=22,80,443 \
      in-interface-list=WAN protocol=tcp
  add action=accept chain=input comment="local WG virtual**" dst-port=12312 \
      in-interface=pppoe-out1 protocol=udp
  add action=accept chain=input comment="accept OSPF" in-interface-list=WAN \
      protocol=ospf
  add action=accept chain=input comment="defconf: accept ICMP" protocol=icmp
  add action=accept chain=input comment=\
      "defconf: accept to local loopback (for CAPsMAN)" dst-address=127.0.0.1
  add action=drop chain=input comment="defconf: drop invalid" connection-state=\
      invalid
  add action=drop chain=input comment="defconf: drop all not coming from LAN" \
      in-interface-list=!LAN
  add action=accept chain=forward comment="defconf: accept in ipsec policy" \
      ipsec-policy=in,ipsec
  add action=accept chain=forward comment="defconf: accept out ipsec policy" \
      ipsec-policy=out,ipsec
  add action=reject chain=forward comment="Guest can't access main " \
      connection-state=invalid,new dst-address=192.168.88.0/24 reject-with=\
      icmp-network-unreachable src-address=172.16.1.0/24
  add action=drop chain=forward comment="defconf: drop invalid" \
      connection-state=invalid log=yes log-prefix=fwd
  add action=drop chain=forward comment=\
      "defconf: drop all from WAN not DSTNATed" connection-nat-state=!dstnat \
      connection-state=new in-interface-list=WAN
  /ip firewall mangle
  add action=change-mss chain=forward new-mss=clamp-to-pmtu out-interface-list=\
      WAN passthrough=yes protocol=tcp tcp-flags=syn
  add action=mark-routing chain=prerouting comment="Fitst 32 hosts no virtual**" \
      dst-address=!192.168.0.0/16 new-routing-mark=DirectWAN passthrough=no \
      src-address=192.168.88.0/27
  add action=mark-routing chain=prerouting comment=\
      "atombox to WAN only use PPPoE" dst-address=!192.168.0.0/16 \
      new-routing-mark=DirectWAN passthrough=no src-mac-address=\
      12:34:56:B2:33:80
  add action=mark-routing chain=prerouting comment="All VPS go PPPoE" \
      dst-address-list=vps new-routing-mark=DirectWAN passthrough=yes
  add action=mark-routing chain=prerouting comment="Guest vlan " \
      new-routing-mark=DirectWAN passthrough=no src-address=172.16.1.0/24
  /ip firewall nat
  add action=src-nat chain=srcnat comment=SRC out-interface=pppoe-out1 \
      to-addresses=101.102.103.104
  add action=src-nat chain=srcnat out-interface=wg1 to-addresses=10.28.6.20
  add action=src-nat chain=srcnat out-interface=wg2 to-addresses=10.28.7.20
  add action=src-nat chain=srcnat comment=SRC log=yes out-interface=pppoe-out1 \
      src-address=172.16.0.0/16 to-addresses=101.102.103.104
  add action=src-nat chain=srcnat comment="VLAN IoT" out-interface=vlan_iot \
      to-addresses=172.16.10.1
  add action=masquerade chain=srcnat comment="Fiber Modem" out-interface=ether1
  /ip firewall service-port
  set ftp disabled=yes
  set tftp disabled=yes
  set h323 disabled=yes
  set p p t p disabled=yes
  set rtsp disabled=no
  #
  # IPV6
  #
  /ipv6 firewall address-list
  add address=::/128 comment="defconf: unspecified address" list=bad_ipv6
  add address=::1/128 comment="defconf: lo" list=bad_ipv6
  add address=fec0::/10 comment="defconf: site-local" list=bad_ipv6
  add address=::ffff:0.0.0.0/96 comment="defconf: ipv4-mapped" list=bad_ipv6
  add address=::/96 comment="defconf: ipv4 compat" list=bad_ipv6
  add address=100::/64 comment="defconf: discard only " list=bad_ipv6
  add address=2001:db8::/32 comment="defconf: documentation" list=bad_ipv6
  add address=2001:10::/28 comment="defconf: ORCHID" list=bad_ipv6
  add address=3ffe::/16 comment="defconf: 6bone" list=bad_ipv6
  /ipv6 firewall filter
  add action=accept chain=input comment=\
      "defconf: accept established,related,untracked" connection-state=\
      established,related,untracked
  add action=accept chain=forward comment=\
      "defconf: accept established,related,untracked" connection-state=\
      established,related,untracked
  add action=accept chain=forward comment="Allow Local " in-interface-list=!WAN
  add action=accept chain=forward comment=Ping protocol=icmpv6
  add action=accept chain=input comment="accept anything from LAN" \
      in-interface-list=!WAN
  add action=accept chain=input comment="RB4011 accept SSH" dst-port=22,80,443 \
      in-interface-list=WAN protocol=tcp
  add action=accept chain=forward comment="allow SSH,WWW,HTTPS,etc" dst-port=\
      22,80,443,993 in-interface-list=WAN protocol=tcp
  add action=accept chain=input comment="Local Wireguard" dst-port=12312 \
      in-interface=pppoe-out1 protocol=udp
  add action=accept chain=input comment="for OSPF" protocol=ospf
  add action=accept chain=input comment="defconf: accept ICMPv6" protocol=\
      icmpv6
  add action=accept chain=input comment="defconf: accept UDP traceroute" port=\
      33434-33534 protocol=udp
  add action=accept chain=input comment=\
      "defconf: accept DHCPv6-Client prefix delegation." dst-port=546 protocol=\
      udp src-address=fe80::/10
  add action=accept chain=input comment="defconf: accept IKE" dst-port=500,4500 \
      protocol=udp
  add action=accept chain=input comment="defconf: accept ipsec AH" protocol=\
      ipsec-ah
  add action=accept chain=input comment="defconf: accept ipsec ESP" protocol=\
      ipsec-esp
  add action=accept chain=input comment=\
      "defconf: accept all that matches ipsec policy" ipsec-policy=in,ipsec
  add action=drop chain=input comment="defconf: drop invalid" connection-state=\
      invalid
  add action=drop chain=forward comment=\
      "defconf: drop packets with bad src ipv6" src-address-list=bad_ipv6
  add action=drop chain=forward comment=\
      "defconf: drop packets with bad dst ipv6" dst-address-list=bad_ipv6
  add action=drop chain=forward comment="defconf: rfc4890 drop hop-limit=1" \
      hop-limit=equal:1 protocol=icmpv6
  add action=drop chain=forward comment="defconf: drop invalid" \
      connection-state=invalid
  add action=accept chain=forward comment="defconf: accept HIP" protocol=139
  add action=drop chain=forward comment=\
      "defconf: drop everything else not coming from LAN" in-interface-list=\
      !LAN
  /ipv6 firewall mangle
  add action=change-mss chain=forward comment="fix MTU, make HTTPS happy" \
      new-mss=clamp-to-pmtu passthrough=yes protocol=tcp tcp-flags=syn
  add action=mark-routing chain=prerouting comment=ATOMBOX dst-address=::/0 \
      in-interface-list=LAN log=yes new-routing-mark=DirectWAN passthrough=no \
      src-mac-address=12:34:56:B2:33:80
  /ipv6 firewall nat
  add action=src-nat chain=srcnat out-interface=wg1 src-address=!fe00::/8 \
      to-address=fd80:88:2::20/128
  add action=src-nat chain=srcnat out-interface=wg2 src-address=!fe00::/8 \
      to-address=fd80:88:66:3::20/128
  add action=masquerade chain=srcnat out-interface=pppoe-out1 src-address=\
      fd80::/16
```

