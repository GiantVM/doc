# GiantVM Booting (Single machine, TCP)
## Change HostOS kernel to Linux-DSM
### 1. Preparation

```shell
    sudo apt-get install build-essential openssl libncurses5-dev libssl-dev 
    sudo apt-get install zlibc minizip libidn11-dev libidn11 bison flex
    
    git clone https://github.com/xianliang66/Linux-DSM.git
    cd Linux-DSM

```
Switch the MACRO definition from `USE_KRDMA_NETWORK` to `USE_KTCP_NETWORK` in arch/x86/include/asm/kvm_host.h.
### 2. Enable DSM support in kernel config
```shell
    make menuconfig
```

`Virtualization` --> `KVM distributed software memory support` --> press 'Y' to include the option

`Save` -->  `Exit`

### 3. Compile the kernel
```shell
    make -jN
```
`N` is the number of threads you'd like to run parallelly.

### 4. Install the kernel
```shell
    sudo make modules_install
    sudo make install
```

### 5. Update grub
```shell
    sudo update-grub
```

### 6. reboot
```shell
    reboot
```
Select the newly installed kernel, which should be `4.9.76+`.

## QEMU
### 1. Preparation

```shell
    sudo apt-get install python pkg-config libglib2.0-dev zlib1g-dev libpixman-1-dev libfdt-dev qemu-system-x86
    
    git clone https://github.com/xianliang66/QEMU.git
    cd QEMU
    
```

### 2. Configuration
```shell
    ./configure --target-list=x86_64-softmmu --enable-kvm
```

### 3. Compilation
```shell
    make -jN
```

### 4. Create hard disk image
```shell
    wget http://ftp.sjtu.edu.cn/ubuntu-cd/16.04.7/ubuntu-16.04.7-server-amd64.iso
    cd QEMU
    ./qemu-img create -f qcow2 ubuntu-server.img 10G
    `qemu-system-x86_64` -m 1024 ubuntu-server.img -cdrom ../ubuntu-16.04.7-server-amd64.iso -enable-kvm
```

## Run GiantVM on a single machine
We simulate two machines by running two QEMU processes on a single machine.

First we install vncviewer to monitor the guest.
```shell
   wget https://www.realvnc.com/download/file/viewer.files/VNC-Viewer-6.19.325-Linux-x64.deb
   sudo dpkg -i VNC-Viewer-6.19.325-Linux-x64.deb
```

Terminal 1:
```shell
    sudo x86_64-softmmu/qemu-system-x86_64 --nographic -hda ubuntu-server.img -cpu host -machine kernel-irqchip=off -smp 2 -m 2048  --enable-kvm -serial mon:stdio -local-cpu 1,start=0,iplist="127.0.0.1 127.0.0.1" -vnc :0
```
Terminal 2:
```shell
    sudo x86_64-softmmu/qemu-system-x86_64 --nographic -hda ubuntu-server.img -cpu host -machine kernel-irqchip=off -smp 2 -m 2048  --enable-kvm -serial mon:stdio -local-cpu 1,start=1,iplist="127.0.0.1 127.0.0.1"
```
Terminal 3:
```shell
    vncviewer :0
```

### Network Configuration
By default, QEMU provides an emulated, user mode networking for VMs, which is slow compared with physical network devices.


We could configure a network bridge between the Host and the VM.

#### On Host:

- Check network devices
    ```
    ifconfig
    ```
##### Change all the "eth0" below to the name of the network device of your machine.

- Create a bridge
    ```
	brctl addbr br0
	```
    
- Clear IP of `eth0`
    ```
    ip addr flush dev eth0
    ```

- Add `eth0` to bridge
	```
    brctl addif br0 eth0
    ```
    
- Create tap interface
	```
    tunctl -t tap0 -u root
    ```

- Add `tap0` to bridge
	```
    brctl addif br0 tap0
    ```
    
- Make sure everything is up
	```
    ifconfig eth0 up
    ifconfig tap0 up
    ifconfig br0 up
    ```

- Check if properly bridged
	```
    brctl show
    ```

- Assign ip to `br0`
	```
	dhclient -v br0
    ```

Reference: https://gist.github.com/extremecoders-re/e8fd8a67a515fee0c873dcafc81d811c

#### Run GiantVM with bridged network
Terminal 1:
```shell
    sudo x86_64-softmmu/qemu-system-x86_64 --nographic -hda ubuntu-server.img -cpu host -machine kernel-irqchip=off -smp 2 -m 2048  --enable-kvm -serial mon:stdio -local-cpu 1,start=0,iplist="127.0.0.1 127.0.0.1" -netdev tap,id=mynet0,ifname=tap0,script=no,downscript=no -device e1000,netdev=mynet0,mac=52:55:00:d1:55:01 -vnc :0
```
Terminal 2:
```shell
    sudo x86_64-softmmu/qemu-system-x86_64 --nographic -hda ubuntu-server.img -cpu host -machine kernel-irqchip=off -smp 2 -m 2048  --enable-kvm -serial mon:stdio -local-cpu 1,start=1,iplist="127.0.0.1 127.0.0.1"
```
Terminal 3:
```shell
    vncviewer :0
```

## Author
Weiye Chen
