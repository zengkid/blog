### PVE下安装routeros
* 设置Local保存Disk
  > 登录PVE的web管理界面
  > DataCenter -> local(pve) -> Storage, 选中local双击打开编辑对话框
  > 在Content列表中选中Disk
  
    ![image](https://github.com/zengkid/blog/assets/3382739/bc225d27-2863-4e11-977b-306d993b8daf)
  > 保存该设置

* 通过SSH登录PVE，账号需要有创建VM的权限
* 编写自动安装脚本, 此脚本来源于官方
  > vi routeros-install.sh, 然后将一下内容贴上去
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
  
    ![image](https://github.com/zengkid/blog/assets/3382739/2906402e-de11-4627-9397-12c8a7d68469)

