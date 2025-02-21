# kernel_scratch_0

## Setup for running kernel_srcatch_0

#### Qemu Setup 
```shell
$ wget https://download.qemu.org/qemu-7.0.0-rc0.tar.xz
$ tar -xvf qemu-7.0.0-rc0.tar.xz
$ cd qemu-7.0.0-rc0
$ ./configure --target-list=x86_64-softmmu --enable-virtfs
$ make -j 48  
```
#### Linux Kernel Setup (incase required) 
```shell
$ git clone https://github.com/torvalds/linux.git
$ cd linux 
$ git checkout master 
$ make mrproper
$ make x86_64_defconfig
$ cat <<EOF >.config-fragment
$ CONFIG_DEBUG_INFO=y
$ CONFIG_DEBUG_KERNEL=y
$ CONFIG_GDB_SCRIPTS=y
$ CONFIG_DEBUG_INFO_NONE=n
$ EOF

$ grep CONFIG_DEBUG_INFO /home/Khadem/linux/.config

If not enabled, add it:

$ echo "CONFIG_DEBUG_INFO=y" >> /home/Khadem/linux/.config
$ echo "CONFIG_DEBUG_INFO_NONE=n" >> /home/Khadem/linux/.config
$ make olddefconfig
$ make -j$(nproc)
```
Make sure that there is no error during compliation and the kernel generate output as following 
```shell
Kernel: arch/x86/boot/bzImage is ready  (#1)
```
#### Create a minimal root file system 
```shell
$ mkdir minimal_rootfs
$ cd minimal_rootfs
$ mkdir -p {bin,sbin,etc,proc,sys,dev,tmp,lib,lib64,usr}

# Copy BusyBox for minimal tools
$ dnf install busybox 
$ which busybox 

$ cp /sbin/busybox ./bin/
$ chmod +x ./bin/busybox

$ mkdir -p ./usr/bin ./bin ./sbin ./usr/sbin
Then, re-run the BusyBox install:
$ chroot ./ /bin/busybox --install -s
$ find . | cpio -o --format=newc | gzip > ../initramfs.img
```
#### Create qemu myos.img
```shell
$ cd /home/Khadem/qemu-7.0.0-rc0/build
$ mkdir qemu/
$ qemu-img create qemu/myos.img 512M
$ mkfs.ext4 qemu/myos.img
$ mkdir -p qemu/mnt
$ mount qemu/myos.img qemu/mnt -t ext4 -o loop 
$ sudo cp -r /home/Khadem/minimal_rootfs/* /home/Khadem/qemu-7.0.0-rc0/build/qemu/mnt
$ sudo umount -f /home/Khadem/qemu-7.0.0-rc0/build/qemu/mnt

$ cat >user-data.txt <<EOF
#cloud-config
$ password: secretpassword
$ chpasswd: { expire: False }
$ ssh_pwauth: True
$ EOF
$ cloud-localds user-data.img user-data.txt
```
#### Running the kernel with qemu 
```shell
$ cd qemu-7.0.0-rc0
$ cd build 
./qemu-system-x86_64 -m 8192 \
        -kernel  /home/Khadem/linux/arch/x86/boot/bzImage \
        -initrd /home/Khadem/initramfs.img -m 4G \
       -machine pc-q35-7.0,accel=kvm,kernel-irqchip=split \
       -device intel-iommu -display curses -nographic \
       -append "root=/dev/sda rw console=ttyS0 nokaslr" \
       -drive file=/home/Khadem/qemu-7.0.0-rc0/build/qemu/myos.img,format=raw  \
       -drive file=/home/Khadem/user-data.img,format=raw \
       -nic user,model=virtio-net-pci,net=192.168.254.0/24,host=192.168.254.1,dns=192.168.254.2,dhcpstart=192.168.254.20,hostname=qemu-bullseye,hostfwd=tcp:127.0.0.1:2201-192.168.254.20:22
```
#### Running kernel_scratch_0 with qemu 
```shell
./qemu-system-x86_64 -m 8192 \
  -cdrom /home/Khadem/kernel/my-kernel.iso \
  -boot d \
  -machine pc-q35-7.0,accel=kvm,kernel-irqchip=split \
  -device intel-iommu -display curses -nographic \
  -drive file=/home/Khadem/qemu-7.0.0-rc0/build/qemu/myos.img,format=raw \
  -drive file=/home/Khadem/user-data.img,format=raw \
  -nic user,model=virtio-net-pci,net=192.168.254.0/24,host=192.168.254.1,dns=192.168.254.2,dhcpstart=192.168.254.20,hostname=qemu-bullseye,hostfwd=tcp:127.0.0.1:2201-192.168.254.20:22
```
![image](https://github.com/user-attachments/assets/426edcf7-3ee9-4553-88cd-7dfd366920af)

Reference:
https://www.linuxjournal.com/content/what-does-it-take-make-kernel-0
