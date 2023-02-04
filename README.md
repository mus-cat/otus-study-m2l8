# Работа с загрузчиком
## Задание выполнялось с использованием VirtualBox 6.1.40. 

## Получение доступа с систиеме без пароля
### Экспременты проводились на ВМ, использовался vagrant с box-образом generic/debian10 (debian 10.13)
### Результаты следующие:
- Метод с передачей параметра `init=/bin/sh` ядру - сработал.

- Метод с передачей параметра `rd.break` ядру - сработал, но при условии, что в системе установлен пакет **dracut** (и пересобранного initrd).

- Метод с передачей параметра(ов) `rw init=/sysroot/bin/sh` - сработал, но так же требует наличия пакета **dracut** (и пересобранного initrd).

## Добавить модуль в initrd
### Экспременты проводились на ВМ, использовался vagrant с box-образом generic/debian10 (debian 10.13)
- модуль добавлен с помощью пакета **dracut**, все делалось в соответствии с методичкой. 
!["list dracut modules"](https://github.com/mus-cat/otus-study-m2l8/blob/main/08.mkinitrd.png) 
!["Draw pidgin on boot"](https://github.com/mus-cat/otus-study-m2l8/blob/main/09.pidgin.png)

## Переименовываем в LVM VG, на которой создан LV корневой ФС 
### Экспременты проводились на ВМ c Debian 11.6. Исходно система была установлена на VG с именем VG_Oldname. 
- Было в начале.  
!["View mount"](https://github.com/mus-cat/otus-study-m2l8/blob/main/10.Initstate-MountLsblk.png)
!["VIew fstab"](https://github.com/mus-cat/otus-study-m2l8/blob/main/11.Initstate-lvsFstab.png)

- Перименовали `VG_Oldname` в `VG_newname`.
```
vgrename VG_Oldname VG_newname
```
!["VG rename"](https://github.com/mus-cat/otus-study-m2l8/blob/main/12.changeVGName.png)

- Внесли изменения только в файлы **/etc/fstab** и **/boot/grub/grub.cfg**.
```
cat /etc/fstab | sed 's/VG_Oldname/VG_newname/' > tmp && mv tmp /etc/fstab
cat /boot/grub/grub.cfg | sed 's/VG_Oldname/VG_newname/' > tmp && mv tmp /boot/grub.grub.cfg
```
!["File change"](https://github.com/mus-cat/otus-study-m2l8/blob/main/13.plusMount.png)  
Перезагрузили и долго ждали загрузки, тем не менее система загрузилась.  
!["reboot result"](https://github.com/mus-cat/otus-study-m2l8/blob/main/14.rebootResult.png)  

- Дополнительно обновили **initrd**.
```
update-initramfs -u -k $(uname -r)
reboot
```
Загрузка сиситемы ускорилась

## Переносим boot раздел на LVM.
### Экспременты проводились на ВМ c Debian 11.6. Система установлена на VG с именем VG_newname, /boot исходно размещается на /dev/sda1

- Исходное состояние  
!["Initial state"](https://github.com/mus-cat/otus-study-m2l8/blob/main/15.initBootState.png)  

- В свободном пространстве VG создаем место под **boot** (соответствующий LV)  и создаем там ФС
```
lvcraete -l 100%free -n boot VG_newname
mkfs.ext4 /dev/VG_newname_boot
```

- Перенсим содержимое папки **boot** в новое место. Вносим изменения в **/etc/fstab**. Исходный раздел где размещался **boot** (**/dev/sda1**) забиваем 0.
```
mount /dev/VG_newname_boot /mnt
cp -r /boot/* /mnt
umount /mnt
mount /dev/VG_newname_boot /boot
sed -E 's/(UUID.+)(\boot.+)/\/dev\/mapper\/VG_newname-boot\t\2' /etc/fstab > tmp && mv tmp /etc/fstab
dd if=/dev/zero of=/dev/sda1 bs=10M
```
!["Zerorring"](https://github.com/mus-cat/otus-study-m2l8/blob/main/16.zerroringOldSda1.png)

3. Устанавливаем **grub** с модулем **lvm** и создаем новый конфиг для **grub** (**/boot/grub/grub.cfg**).  Перезагружаемся.
```
grub-mkconfig -o new.cfg && cp new.cfg /boot/grub/grub.cfg
grub-install --modules=lvm /dev/sda
reboot
```
!["Final State"](https://github.com/mus-cat/otus-study-m2l8/blob/main/17.finalState.png)
