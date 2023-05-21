### PVE下安装routeros
* 设置Local保存Disk
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
       
     ![image](https://github.com/zengkid/blog/assets/3382739/700cedaf-f362-41d4-9960-ddc16007980a)
  
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
     
      **注意：由于我在PVE的第一网卡为管理口了，所以只能设置为LAN口，所以我在配置routeros时候，桥接口调整了以下，net0的桥接口为vmbr1，而net1的桥接口为vmbr0zui
      最终配置如下图
     ![image](https://github.com/zengkid/blog/assets/3382739/e89d5141-3290-4e5d-af22-4bc8a731aeef)
    
* 安装教程到此为止，下一步进入routeros的设置教程   

