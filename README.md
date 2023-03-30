# Загрузка Linux
Для выполнения этого действия требуется установить приложением git:
`git clone https://github.com/altyn-kenzhebaev/boot-hw7.git`
В текущей директории появится папка с именем репозитория. В данном случае hw-1. Ознакомимся с содержимым:
```
cd boot-hw7
ls -l
README.md
Vagrantfile
```
Здесь:
- README.md - файл с данным руководством
- Vagrantfile - файл описывающий виртуальную инфраструктуру для `Vagrant`
Запускаем ВМ:
```
vagrant up
```
### Попасть в систему без пароля:
#### Способ 1
- Заходим в загрузчик до старта системы (перед тем, как начнется загрузка нужно успеть нажать на клавиушу e)
- Затем находим строку `linux16` и убираем из нее все значение `console`, а частности: `console=tty0 console=ttyS0,115200n8` и добавляем `rd.break`, далее нажимаем `ctrl+x`
- Произойдет загрузка системы в аварийном режиме, далее выполняем команду перемонтирования корня для чтения и записи - `mount -o remount,rw /sysroot`, далее `chroot /sysroot`
- И вот мы в системе без пароля

#### Способ 2
- Заходим в загрузчик до старта системы (перед тем, как начнется загрузка нужно успеть нажать на клавиушу e)
- Затем находим строку `linux16` и убираем из нее все значение `console`, а частности: `console=tty0 console=ttyS0,115200n8` и добавляем `init=/bin/bash`, далее нажимаем `ctrl+x`
- Произойдет загрузка системы в аварийном режиме, далее выполняем команду перемонтирования корня для чтения и записи - `mount -o remount,rw /sysroot`, далее `chroot /sysroot`
- И вот мы в системе без пароля

### Установить систему с LVM, после чего переименовать VG:
Меняем название `VG vgrename VolGroup00 OtusRoot`
Далее будем править файлы, чтобы в дальнейшем мы могли загрузиться, первый файл это ```/etc/fstab```, заходим в него и меняем старое название на новое:
```bash
vi /etc/fstab
vi /etc/default/grub
vi /boot/grub2/grub.cfg
```
Далее пересоздаем initrd image, выполняем 
```bash
mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)
```
Перезагружаем сервер, теперь у нас система загружается с новым названием VG. Также можно менять названия логических томов.

### Добавить модуль в initrd:
Скрипты модулей хранятся в каталоге /usr/lib/dracut/modules.d/. Для того чтобы добавить свой модуль создаем там папку с именем 01test, 
```bash
mkdir /usr/lib/dracut/modules.d/01test
vi /usr/lib/dracut/modules.d/01test/module-setup.sh
vi /usr/lib/dracut/modules.d/01test/test.sh
chmod +x /usr/lib/dracut/modules.d/01test/*
```
`module-setup.sh` вписываем:
```
#!/bin/bash

check() { # Функция, которая указывает что модуль должен быть включен по умолчанию
    return 0
}

depends() { # Выводит все зависимости от которых зависит наш модуль
    return 0
}

install() {
    inst_hook cleanup 00 "${moddir}/test.sh" # Запускает скрипт
}
```
В файле `test.sh`:
```
#!/bin/bash

cat <<'msgend'
Hello! You are in dracut module!
 ___________________
< I'm dracut module >
 -------------------
   \
    \
        .--.
       |o_o |
       |:_/ |
      //   \ \
     (|     | )
    /'\_   _/`\
    \___)=(___/
msgend
sleep 10
echo " continuing...."
```
- Далее выполняем команду ```dracut -f -v```
- Перезагружаемся и видим пингвина

### Задание со (*). Сконфигурировать систему без отдельного раздела с /boot, а только с LVM:
Размечаем, создаем новый том LVM и форматируем:
```
parted -s /dev/sdb mklabel msdos
parted /dev/sdb mkpart primary xfs 0% 100%
pvcreate /dev/sdb1 --bootloaderareasize 1M
vgcreate vg01 /dev/sdb1
lvcreate -n root -l 100%FREE vg01
mkfs.xfs /dev/mapper/vg01-root
```
Переводим ОС на новый диск, обновляем grub:
```
yum -y install xfsdump
mount /dev/mapper/vg01-root /mnt
xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
rsync -avx /boot/ /mnt/boot/
for i in /proc/ /sys/ /dev/ /run/ ; do mount --bind $i /mnt/$i; done
chroot /mnt/root
yum update grub2 -y
```
меняем на наш новый lvm и комментируем /boot:
```
vi /etc/fstab
```
Правим grub (значение rd.lvm.lv на `rd.lvm.lv=vg01/root`):
```
vi /etc/default/grub 
grub2-mkconfig -o /boot/grub2/grub.cfg
grub2-install /dev/sdb
```
Выключаем SELINUX, правим в значение `Disabled`:
```
vi /etc/selinux/config
```
Выходим и перезагружаемся с нового диска командой `reboot`, не забываем при загрузке ВМ выбрать нужный нам диск, кнопка F12

GRUB:
```
### BEGIN /etc/grub.d/10_linux ###
menuentry 'CentOS Linux (3.10.0-862.2.3.el7.x86_64) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-862.2.3.el7.x86_64-advanced-1852efec-0768-40cd-a577-d23f4b6cd32b' {
	load_video
	set gfxpayload=keep
	insmod gzio
	insmod part_msdos
	insmod lvm
	insmod xfs
	set root='lvmid/mHBb3o-AiIH-vsCv-3Kx3-g1nf-EWHu-pUnBct/rRyovA-e2RX-woE6-OIKJ-W8i8-TIUQ-4cJHNt'
	if [ x$feature_platform_search_hint = xy ]; then
	  search --no-floppy --fs-uuid --set=root --hint='lvmid/mHBb3o-AiIH-vsCv-3Kx3-g1nf-EWHu-pUnBct/rRyovA-e2RX-woE6-OIKJ-W8i8-TIUQ-4cJHNt'  1852efec-0768-40cd-a577-d23f4b6cd32b
	else
	  search --no-floppy --fs-uuid --set=root 1852efec-0768-40cd-a577-d23f4b6cd32b
	fi
	linux16 /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64 root=/dev/mapper/vg01-root ro no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto rd.lvm.lv=vg01/root rd.lvm.lv=vg01/root rhgb quiet 
	initrd16 /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
}
if [ "x$default" = 'CentOS Linux (3.10.0-862.2.3.el7.x86_64) 7 (Core)' ]; then default='Advanced options for CentOS Linux>CentOS Linux (3.10.0-862.2.3.el7.x86_64) 7 (Core)'; fi;
### END /etc/grub.d/10_linux ###
```
Вывод дисков:
```
[root@repo ~]# df -h
Filesystem             Size  Used Avail Use% Mounted on
/dev/mapper/vg01-root   40G  847M   40G   3% /
devtmpfs               235M     0  235M   0% /dev
tmpfs                  244M     0  244M   0% /dev/shm
tmpfs                  244M  4.5M  240M   2% /run
tmpfs                  244M     0  244M   0% /sys/fs/cgroup
tmpfs                   49M     0   49M   0% /run/user/1000
[root@repo ~]# lsblk 
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:1    0 37.5G  0 lvm  
  └─VolGroup00-LogVol01 253:2    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0 39.9G  0 disk 
└─sdb1                    8:17   0 39.9G  0 part 
  └─vg01-root           253:0    0 39.9G  0 lvm  /
[root@repo ~]#
```
