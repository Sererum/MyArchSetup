# Установка Arch (Dual boot)

- Вставить флешку и загрузить линукс, тут все стандартно

## Подключение WiFi
- Можно просто подключиться по проводу с телефона используя в настройках точки доступа, включив **USB tethering**, или интернет провода

Или же подключаемся через терминал:
```bash
iwctl

device list # Получаем имя устройства
station *устройство* scan
station *устройство* get-networks
station *устройство* connect *SSID или имя wifi*
quit

ping ya.ru # Проверяем подключение
```
## Синхронизация времени
```bash
timedatectl set-ntp true # Синхронизация времени
timedatectl status # Проверка времени 
```

## Разметка жесткого диска
Сначала смотрим за тем, какие у нас есть разделы, и как они называются 
```bash
fdisk -l
```
Один из вариантов того, что мы видим (в разделах 4 и 5 уже установлен какой-то линукс, нам его надо будет стереть не попорти 1-3 разделы, в которых у нас Windows):
```bash
Device           ...   Type
/dev/nvme0n1p1         EFI System
/dev/nvme0n1p2         Microsoft reserved
/dev/nvme0n1p3         Microsoft basic data
/dev/nvme0n1p4         Linux swap
/dev/nvme0n1p5         Linux filesystem
```
Затем заходим в саму утилиту fdisk и размечаем диски так, как нам надо
```bash
fdisk /dev/nvme0n1 # Размечаем наш корневой раздел, который мы увидели выше

d 5 # Удаляем 5-ый раздел, который нам больше не нужен
d 4 # Аналогично удаляем 4-ый раздел
```
В итоге у нас должны остаться только EFI и Microsoft:
```bash
Device           ...   Type
/dev/nvme0n1p1         EFI System
/dev/nvme0n1p2         Microsoft reserved
/dev/nvme0n1p3         Microsoft basic data
```
Теперь заново создаем раздел SWAP
```bash
n # Создаем раздел
*Enter*
*Enter*
+2G
y
t # Помечаем как SWAP
4
19
```

Затем создаем основной раздел Arch (используя все свободное место)
```bash
n
*Enter*
*Enter*
*Enter*
y
```
Проверяем, что все правильно, и перезаписываем диски
```bash
p # Печатаем результат
w # Перезаписываем
```
Окончательная проверка, что все правильно:
```bash
fdisk -l
```
После разметки необходимо создать файловые системы:
```bash
mkswap /dev/nvme0n1p4 # Создаем подкачку
swapon /dev/nvme0n1p4 # Включаем подкачку
mkfs.ext4 /dev/nvme0n1p5 # Создаем файловую систему на основном разделе
```
И наконец монтируем нашу файловую систему к /mnt
```bash
mount /dev/nvme0n1p5 /mnt
```
В итоге мы должны иметь вот такой вывод при вводе команды lsblk:
```bash
NAME          ...  TYPE MOUNTPOINTS
loop0              loop /run/archiso/airootfs
sda                disk
--sda1             part
nvme0n1            disk
--nvme0n1p1        part
--nvme0n1p2        part
--nvme0n1p3        part
--nvme0n1p4        part [SWAP]
--nvme0n1p5        part /mnt

```
## Установка базовых пакетов
Скачиваем все базовые пакеты для Arch
```bash
pacstrap /mnt base linux linux-firmware base-devel
```
Долго ждем...
Генерируем fstab -- ОЧЕНЬ ВАЖНО
```bash
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab # Проверяем что все ок
```
## Заходим в систему
Сначала делаем chroot:
```bash
arch-chroot /mnt
```
И все, мы в системе, и пригласительная строка
```bash
root@archiso ~ #
```
Меняется на 
```bash
[root@archiso /]#
```
## Настраиваем систему
Настраиваем регион и время
```bash
ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime
hwclock --systohc
```
Устанавливаем vim, чтобы редактировать файлы
```bash
pacman -S vim 
```
Редактируем локали
```bash
vim /etc/locale.gen
```
Нужно убрать комментарий с двух строк, которые можно нати через поиск, а затем выйти и сохранить
```bash
en_US.UTF-8 UTF-8
ru_RU.UTF-8 UTF-8
:wq
```
Затем генерируем локали
```bash
locale-gen
```
После этой команды должны появиться две наши выбранные локали
Создаем файл с именем компьютера через vim
```bash
vim /etc/hostname # Этого файла пока не существует
```
В этом файле БЕЗ ПРОБЕЛОВ, без ничего вводим имя компьютера и сохраняем файл
 Редактируем наши хосты
```bash
vim /etc/hosts # Этот файл есть, его надо просто отредактировать
```
Открывшийся файл в своем итоговом варианте должен иметь вид:
```bash
# ...
# ...

127.0.0.1    localhost # Здесь 4 ПРОБЕЛА
::1          localhost # Здесь РОВНЕНЬКО
127.0.1.1    [имяПК].localdomain    [имяПК] # Здесь 4 и 4 пробела
```
Сохраняем файл и начинаем создавать пароли и пользователей
```bash
passwd # Чтобы создать пароль для root
useradd -m (имя пользователя) # Создаем нашего пользователя
passwd (имя пользователя) # Чтобы задать пароль для пользователя
```
Теперь добавим пользователя в группы:
```bash
usermod -aG группа1,группа2,... (имя пользователя)
```
Существуют следующие группы:
- wheel - ОБЯЗАТЕЛЬНАЯ, иначе у пользователя не будет доступа к root-правам
- audio - (рекомандуется) для работы с audio 
- video - (рекомандуется) для полного доступа к видеокарте 
- storage - (рекомандуется) для доступа к монтируемым жестким дискам 
- optical - для доступа к CD-ROM или DVD-ROM
- scanner - для доступа к принтерам и сканерам
- vboxusers - для тех, кто устанавливает Arch на виртуалку

В итоге должно получиться что-то типо такого
```bash
usermod -aG wheel,audio,video,storage,scanner sererum
```
Устанавливаем sudo
```bash
pacman -S sudo
```
Редактируем настройки группы wheel
```bash
EDITOR=vim visudo
```
Переходим в конец файла, и расскоментируем строчку
```bash
%wheel ALL=(ALL:ALL) ALL # ТОЛЬКО ЭТУ
```
Сохраняем файл 
Устанавливаем интернетный менеджер
```bash
pacman -S networkmanager
``` 
Теперь включаем этот менеджер
```bash
systemctl enable NetworkManager
```
Если все хорошо, то будут созданы symlink-и

## Редактирование refind
Если до этого refind не был установлен, то сначала скачаем его:
```bash
pacman -S refind gdisk
refind-install
```
Если refind был установлен до этого, то просто монтируем его к /mnt/boot/efi
```bash
mkdir -p /boot/efi
mount /dev/nvme0n1p1 /boot/efi
```
Теперь редактируем refind.conf
```bash
vim /boot/efi/EFI/refind/refind.conf
```
Нужно найти строчку "Arch Linux"
Ниже в options надо заменить строчку
```bash
options  "root=PARTUUID=... rw add_efi_memmap"
```
на (раздел, в котором находится наша директория Arch)
```bash
options  "root=PARTUUID=/dev/nvme0n1p5 rw add_efi_memmap" 
```
Далее надо поправить еще один файл:
```bash
vim /boot/efi/refind-linux.conf # Или где-то в похожей локации
```
Внутри этого файла нам нужно удалить все строчки связанные с загрузкой образа линукс, т.е. удалить (обычно) первые две строки, в которых есть строчки archisolabel=...

## Редактирование grub
Сначала скачиваем все необходимые зависимости:
```bash
pacman -S grub efibootmgr dosfstools mtools os-prober
```
Затем монтируем необходимые разделы:
```bash
mount /dev/nvme0n1p1 /boot/efi
mkdir -p /mnt/windows
mount /dev/nvme0n1p3 /mnt/windows
```
Затем устанавливаем непосредственно grub:
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
```
Затем редактируем файл grub
```bash
vim /etc/default/grub
```
В этом файле нам надо поменять:
```bash
GRUB_TIMEOUT=5 -> GRUB_TIMEOUT=15
GRUB_DISABLE_OS_PROBER=false - раскоментируем
```
После этого обновляем конфиг grub:
```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

## Перезагружаемся
Сначала выходим из arch-chroot и извлекаем разделы, которые мы монтировали
```bash
exit 
umount /mnt/boot/efi -l
umount /mnt/windows -l # Если пользовались grub
umount /mnt -l
reboot
```

## Перезагружаемся
Подключаемся к интернету, чтобы продолжить настройку системы:
```bash
nmcli device wifi list # Список сетей wifi
nmcli device wifi connect "Имя_Сети" password "Пароль" # Подключение к wifi
```
