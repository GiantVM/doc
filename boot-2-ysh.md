# GiantVm 操作过程 -ysh

主机环境：Windows 11 64位 core i9 12900p

此次使用VMWare Workstation pro 16进行操作



## 1.环境配置

#### step1 系统准备

> win+r 打开cmd
>
> sysinfo
>
> 查看是否开启了Hyper-V，如果开启应将其关闭
>
> 关闭方式
>
> - 【启用或关闭Windows功能】-> 将虚拟机平台和windows虚拟机监控程序平台关闭
>   ![image-20220930105100032](C:\Users\Ysh78\AppData\Roaming\Typora\typora-user-images\image-20220930105100032.png)
> - 如果看到了Hyper-V的选项框，将其取消勾选
> - 然后重启生效修改

   ### step2 VMware 设置

> VMware 中启动Ubuntu16.04
> 配置为
> 内核： Linux4.15.0-112，磁盘分配>40G,在CPU设置中启用嵌套虚拟化

   ### step3  下载必要的包
> ```shell
> sudo apt-get install build-essential openssl libncurses5-dev libssl-dev
> 
> sudo apt-get install zlibc minizip libidn11-dev libidn11 bison flex
> ```
>
> 

   ### step4 获得Linux-DSM
> ```shell
> git clone https://github.com/GiantVM/Linux-DSM.git
> ```
>
> 

   ### step5
> ```shell
> cd Linux-DSM
> ```
>
> 

   ### step6 Enable DSM support
> ```shell
> make menuconfig
> ```
>
>    `Virtualization` --> `KVM distributed software memory support` --> `press 'Y' to include the option`
>    `Save` --> `Exit`

   ### step7 Compile the Kernel (make)
   make -jN   
   [N 是]
   wait for about three hour(or more)
   `
   之前的失败经历：
   Environment: win11 wsl2 Ubuntu16.04 LinuxKernel version 5.10
   output :
   makefile:976: recipe for target 'vmlinux' failed 
   `

   ### step 8 install the Kernel
   > ```shell
   > sudo make modules_install
   sudo make install
   ```

   

### step 9 update the grub

   > [在我的尝试中，这时候应当先打开grub这个文件]
   > [gedit 比较方便看，用vi也可以]

   ```shell
   sudo gedit /etc/default/grub
   ```

   [然后将GRUB_HIDDEN_TIMEOUT 这个属性置为0，不然之后重启的时候没时间换系统]
   这自己操作
   [然后是核心操作 ]

   ```shell
   sudo update-grub
   ```

   [之后重启]

   ```shell
   reboot
   ```

   [重启后看到下面界面，按照图片选择]
   [之后等待，启动后，在shell里输入]

   ```shell
   uname -a 
   ```

   [可以看到版本为ubuntu 4.9.76+]



## 2.QEMU
### step1 Prepartion
> sudo apt-get install python pkg-config libglib2.0-dev zlib1g-dev libpixman-1-dev libfdt-dev
> git clone https://github.com/GiantVM/QEMU.git
> cd QEMU

### step2 Configuration
> ```shell
> ./configure --target-list=x86_64-softmmu --enable-kvm
> ```
>
> 

### step3 Compilation
> ```shell
> make -jN
> ```
>
> 

### step4 Create hard disk image
> ```shell
> cd ..
> wget http://ftp.sjtu.edu.cn/ubuntu-cd/16.04.7/ubuntu-16.04.7-server-amd64.iso
> ```
>
> [如果找不到，可以直接输入http://ftp.sjtu.edu.cn/ubuntu-cd/16.04.7，在里面找到Ubuntu-16.x-server.iso ，然后wget]
>
> [这里就是得用apt下一个qemu，选择不下,用底下x86-64_softmmu/ 底下的qemu-system-x86_64会卡死]
>
> ```shell
> /qemu-img create -f qcow2 ubuntu-server.img 10G
> sudo apt-get install qemu
> qemu-system-x86_64 -m 1024 ubuntu-server.img -cdrom ../ubuntu-16.04.7-server-amd64.iso -enable-kvm
> ```
>
> [上面会跳出系统设置，基本设置一下用户名和密码，然后会问是否要装载GRUB，选择yes，其他无所谓]

## 3.Run Giant VM on a single machine

### First we install vncviewer to monitor the guest.

```shell
   wget https://www.realvnc.com/download/file/viewer.files/VNC-Viewer-6.19.325-Linux-x64.deb
   sudo dpkg -i VNC-Viewer-6.19.325-Linux-x64.deb
```

如果下面报错说内存不够，把虚拟机关掉，多分配给它一点内存（>8G)

>  ### terminal 1 : 
>
>  ```shell
>  cd QEMU/
>  
>  sudo x86_64-softmmu/qemu-system-x86_64 --nographic -hda ubuntu-server.img -cpu host -machine kernel-irqchip=off -smp 4 -m 4096  --enable-kvm -serial mon:stdio -local-cpu 2,start=0,iplist="127.0.0.1 127.0.0.1" -vnc :0
>  ```
>
>  

> ### terminal 2：
>
> ```shell
> cd QEMU/
> sudo x86_64-softmmu/qemu-system-x86_64 --nographic -hda ubuntu-server.img -cpu host -machine kernel-irqchip=off -smp 4 -m 2048  --enable-kvm -serial mon:stdio -local-cpu 2,start=2,iplist="127.0.0.1 127.0.0.1"
> ```

> ### terminal 3:[启动 vncviewer]
>
> 冒号后面的0和前面 terminal1 后面-vnc 后面的数字对应
>
> ```shell
> vncviewer :0
> ```
>
> 如果启动之后看到 nobootable device ，可能需要检查2 QEMU 最后一步的 qemu-system-x86_64 那段是不是正常
>
> 

