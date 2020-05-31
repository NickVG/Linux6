# Linux6
# ДЗ по теме Загрузка Системы


**Попасть в систему без пароля несколькими способами**

*Способ 1. init=/bin/bash*
с помощью опции init мы задаём какая команда выполнится первой при загрузке системы

	mount -o remount,rw /
	passwd root
	touch /.autorelabel

*Способ 2. rd.break*
Использование rd.break прерывает процесс загрузки перед тем как управление переходит от initramfs к systemd
	
	mount -o remount,rw /sysroot
	chroot /sysroot
        passwd root
        touch /.autorelabel

*Способ 3. rw init=/sysroot/bin/bash*
заменяем ro на rw init=/sysroot/bin/bash - монитруем файловую систему сразу в режиме на запись. Остальное аналогично  примерy 1.

Вместо touch ./autorelabel наверное можно выключить selinux в /etc/selinux/config (SELINUX=permissive), а после загрузки поменять контекст на /etc/shadow и /etc/passwd

**Установить систему с LVM, после чего переименовать VG **
Результаты выполнения ***cat /proc/cmdline, vgs и lvs*** для проверки того, что работа производтся на сервере с поддержкой lvm

	[root@lvm vagrant]# vgs
	  VG         #PV #LV #SN Attr   VSize   VFree
	  VolGroup00   1   2   0 wz--n- <38.97g    0 
	[root@lvm vagrant]# lvs
	  LV       VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
	  LogVol00 VolGroup00 -wi-ao---- <37.47g                                                    
	  LogVol01 VolGroup00 -wi-ao----   1.50g                                                    
	[root@lvm vagrant]# cat /proc/cmdline 
	BOOT_IMAGE=/vmlinuz-3.10.0-862.2.3.el7.x86_64 root=/dev/mapper/VolGroup00-LogVol00 ro no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto rd.lvm.lv=VolGroup00/LogVol00 rd.lvm.lv=VolGroup00/LogVol01 rhgb quiet

Переименуем VG

	[root@centos Linux6]# vgrename VolGroup00 OtusRoot
	  Volume group "VolGroup00" successfully renamed to "OtusRoot"
	[root@lvm vagrant]# lsblk
	NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
	sda                       8:0    0   40G  0 disk 
	├─sda1                    8:1    0    1M  0 part 
	├─sda2                    8:2    0    1G  0 part /boot
	└─sda3                    8:3    0   39G  0 part 
	  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
	  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]

Далее необходимо заменить в: /etc/fstab, /etc/default/grub, /boot/grub2/grub.cfg. старое название на новое.

Пересоздаем initrd image, чтобý он знал новое название Volume Group

	[root@lvm vagrant]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)

Проверяем, что получилось после перезагрузки
	[root@lvm vagrant]# vgs
	  VG       #PV #LV #SN Attr   VSize   VFree
	  OtusRoot   1   2   0 wz--n- <38.97g    0 
	[root@lvm vagrant]# lvs
	  LV       VG       Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
	  LogVol00 OtusRoot -wi-ao---- <37.47g                                                    
	  LogVol01 OtusRoot -wi-ao----   1.50g


***Добавить модуль в initrd***

Для того чтобы добавить свой модуль создаем папку с именем 01test:

	mkdir /usr/lib/dracut/modules.d/01test

В созданной папке создаём два скрипта:
module-setup.sh - станавливает модуль и визывает скрипт test.sh


	#!/bin/bash

	#проверка неких условий
	check() {
	    return 0
	}

	#проверка зависимостей	
	depends() {
	    return 0
	}

	#указание к какому хуку прицепить модуль и что при этом сделать(здесь последний хук загрузки initrd и выполнить test.sh из папки модуля)
	install() {
	    inst_hook cleanup 00 "${moddir}/test.sh"
	}

test.sh - вызываемый скрипт, в нём у нас рисуется пингвинчик

	#!/bin/bash

	exec 0<>/dev/console 1<>/dev/console 2<>/dev/console
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

Пересобираем образ initrd

	dracut -f -v

Проверяем, что модуль загружен в образ:

	lsinitrd -m /boot/initramfs-$(uname -r).img | grep test

После чего можно пойти двумя путями для проверки:
- Перезагрузиться и руками вцключить опции rghb и quiet и увидеть вывод
- Либо отредактировать grub.cfg убрав пути опции

При перезагрузке появляется картинка пингвина.

