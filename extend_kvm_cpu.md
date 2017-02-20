## 让KVM突破限制，支持512个vCPU

需要修改qemu和kvm。

1. qemu-system-x86_64: unsupported number of maxcpus

    include/sysemu/sysemu.h

    ```c
    #define MAX_CPUMASK_BITS 288
    ```

    =>

    ```c
    #define MAX_CPUMASK_BITS 512
    ```

2. qemu-system-x86_64: Number of SMP CPUs requested (500) exceeds max CPUs supported by machine 'pc-i440fx-2.8' (255)

    ```
    m->max_cpus = 288;
    ```

    =>

    ```
    m->max_cpus = 512;
    ```


3. Warning: Number of SMP cpus requested (500) exceeds the recommended cpus supported by KVM (240)
Number of SMP cpus requested (500) exceeds the maximum cpus supported by KVM (288)

    改kvm-all.c

    ```c
    #define KVM_CAP_MAX_VCPUS 66       /* returns max vcpus per vm */
    static int kvm_max_vcpus(KVMState *s)
    {
        int ret = kvm_check_extension(s, KVM_CAP_MAX_VCPUS);
        return (ret) ? ret : kvm_recommended_vcpus(s);
    }

    int kvm_check_extension(KVMState *s, unsigned int extension)
    {
        int ret;

        ret = kvm_ioctl(s, KVM_CHECK_EXTENSION, extension);
        if (ret < 0) {
            ret = 0;
        }

        return ret;
    }

    static int kvm_init(MachineState *ms)
    {
        ...
        soft_vcpus_limit = kvm_recommended_vcpus(s);
        hard_vcpus_limit = kvm_max_vcpus(s);

        while (nc->name) {
            if (nc->num > soft_vcpus_limit) {
                fprintf(stderr,
                        "Warning: Number of %s cpus requested (%d) exceeds "
                        "the recommended cpus supported by KVM (%d)\n",
                        nc->name, nc->num, soft_vcpus_limit);

                if (nc->num > hard_vcpus_limit) {
                    fprintf(stderr, "Number of %s cpus requested (%d) exceeds "
                            "the maximum cpus supported by KVM (%d)\n",
                            nc->name, nc->num, hard_vcpus_limit);
                    exit(1);
                }
            }
            nc++;
        }
    }
    ```

    KVM接口返回的vcpu数量限制。本质上还是要改kvm。

    arch/x86/include/asm/kvm_host.h

    ```c
    #define KVM_MAX_VCPUS 288
    #define KVM_SOFT_MAX_VCPUS 240
    ```

    =>

    ```c
    #define KVM_MAX_VCPUS 512
    #define KVM_SOFT_MAX_VCPUS 512
    ```


4. qemu-system-x86_64: current -smp configuration requires Extended Interrupt Mode enabled. You can add an IOMMU using: -device intel-iommu,intremap=on,eim=on

    启动时加上参数 -device intel-iommu,intremap=on,eim=on

5. qemu-system-x86_64: -device intel-iommu,intremap=on,eim=on: Intel Interrupt Remapping cannot work with kernel-irqchip=on, please use 'split|off'.

    sudo /home/binss/Desktop/qemu/bin/debug/native/x86_64-softmmu/qemu-system-x86_64 -m 512 -hda myvm.img -boot c -vnc :1 -enable-kvm -smp 500 -machine q35,kernel-irqchip=split -device intel-iommu,intremap=on,eim=on

    可参考：https://lists.gnu.org/archive/html/qemu-devel/2016-07/msg02930.html


6. 突破guest OS的CPU限制

    修改nr_cpu，当前为

    ```
    grep NR_CPUS /boot/config-`uname -r`
    256
    ```

    这里可以手动修改config然后重新编译kernel，或者换用更新的kernel，比如4.4.0-62。


