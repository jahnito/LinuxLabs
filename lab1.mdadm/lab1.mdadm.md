# lab1. mdadm

## 1. **RAID0**

Цель лабарторной работы, создать массив 0 уровня из двух дисков.
Добавляем 2 диска в виртуальную машину одинакого объема.Запускаем виртуальную машину.

### Проверка наличия новых дисков в системе.

Через утилиту **lsblk**

```jahn@debian:~$ lsblk 
NAME           MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda              8:0    0    8G  0 disk  
└─sda1           8:1    0    8G  0 part  
  └─md0          9:0    0    8G  0 raid1 
    ├─vg0-boot 253:0    0  488M  0 lvm   /boot
    ├─vg0-swap 253:1    0  976M  0 lvm   [SWAP]
    └─vg0-root 253:2    0  6.6G  0 lvm   /
sdb              8:16   0    8G  0 disk  
└─sdb1           8:17   0    8G  0 part  
  └─md0          9:0    0    8G  0 raid1 
    ├─vg0-boot 253:0    0  488M  0 lvm   /boot
    ├─vg0-swap 253:1    0  976M  0 lvm   [SWAP]
    └─vg0-root 253:2    0  6.6G  0 lvm   /
nvme0n1        259:0    0    2G  0 disk  
nvme0n2        259:1    0    2G  0 disk  
```

Через утилиту **fdisk**

```
jahn@debian:~$ sudo fdisk -l
[sudo] password for jahn: 
Disk /dev/nvme0n1: 2 GiB, 2147483648 bytes, 4194304 sectors
Disk model: ORCL-VBOX-NVME-VER12                    
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/nvme0n2: 2 GiB, 2147483648 bytes, 4194304 sectors
Disk model: ORCL-VBOX-NVME-VER12                    
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sda: 8 GiB, 8589934592 bytes, 16777216 sectors
Disk model: VBOX HARDDISK   
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xeaa4e05f

Device     Boot Start      End  Sectors Size Id Type
/dev/sda1  *     2048 16775167 16773120   8G fd Linux raid autodetect
...
```
В приведенном примере отображаются два не размеченных nvme диска.

### Подготовка дисков

Производим зануление суперблоков на дисках

```
jahn@debian:~$ sudo mdadm --zero-superblock --force /dev/nvme0n{1,2}
mdadm: Unrecognised md component device - /dev/nvme0n1
mdadm: Unrecognised md component device - /dev/nvme0n2
```

такой вывод сообщает об отсутствии ранее созданных массивов, далее удаляем старые метаданные и сигнатуры

```
sudo wipefs --all --force /dev/nvme0n{1,2}
```

### Создание массива

Необходимые параметры mdadm:

--create /dev/md1    # Инструкция которая говорит что необходимо создать массив с номером, сокращенная опция -C
--level={$l}         # Параметр определяющий уровень массива 0, 1, 5, 10, сокращенная опция -l
--raid-devices=2     # Количество устройств для объеденения в массив,  сокращенная опция -n
--verbose            # Вывод доп. информации при выполнении операции,  сокращенная опция -v

Дополнительные опции доступны для просмотра в **man mdadm**

```
jahn@debian:~$ sudo mdadm --create /dev/md1 --verbose --level=0 --raid-devices=2 /dev/nvme0n1 /dev/nvme0n2

mdadm: chunk size defaults to 512K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md1 started.
```

Т.к. в приведенной системе система изначально установлена на массив md0, новый массив получило имя md1. После создания проверяем наличие массива.

```
jahn@debian:~$ lsblk 
NAME           MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda              8:0    0    8G  0 disk  
└─sda1           8:1    0    8G  0 part  
  └─md0          9:0    0    8G  0 raid1 
    ├─vg0-boot 253:0    0  488M  0 lvm   /boot
    ├─vg0-swap 253:1    0  976M  0 lvm   [SWAP]
    └─vg0-root 253:2    0  6.6G  0 lvm   /
sdb              8:16   0    8G  0 disk  
└─sdb1           8:17   0    8G  0 part  
  └─md0          9:0    0    8G  0 raid1 
    ├─vg0-boot 253:0    0  488M  0 lvm   /boot
    ├─vg0-swap 253:1    0  976M  0 lvm   [SWAP]
    └─vg0-root 253:2    0  6.6G  0 lvm   /
nvme0n1        259:0    0    2G  0 disk  
└─md1            9:1    0    4G  0 raid0 
nvme0n2        259:1    0    2G  0 disk  
└─md1            9:1    0    4G  0 raid0 
```

Ёмкость массива стала 4Gb. Смотрим состояние массива.

```
jahn@debian:~$ cat /proc/mdstat 
Personalities : [raid1] [linear] [multipath] [raid0] [raid6] [raid5] [raid4] [raid10] 
md1 : active raid0 nvme0n2[1] nvme0n1[0]
      4188160 blocks super 1.2 512k chunks
      
md0 : active raid1 sda1[0] sdb1[1]
      8381440 blocks super 1.2 [2/2] [UU]
      
unused devices: <none>
```

Смотрим детальную информацию о массиве

```
jahn@debian:~$ sudo mdadm --detail /dev/md1
/dev/md1:
           Version : 1.2
     Creation Time : Fri Dec 22 02:22:36 2023
        Raid Level : raid0
        Array Size : 4188160 (3.99 GiB 4.29 GB)
      Raid Devices : 2
     Total Devices : 2
       Persistence : Superblock is persistent

       Update Time : Fri Dec 22 02:22:36 2023
             State : clean 
    Active Devices : 2
   Working Devices : 2
    Failed Devices : 0
     Spare Devices : 0

            Layout : -unknown-
        Chunk Size : 512K

Consistency Policy : none

              Name : debian:1  (local to host debian)
              UUID : ef13166f:d3da15ed:74b9fe53:65a9ee44
            Events : 0

    Number   Major   Minor   RaidDevice State
       0     259        0        0      active sync   /dev/nvme0n1
       1     259        1        1      active sync   /dev/nvme0n2
```

+ Version — версия метаданных.
+ Creation Time — дата в время создания массива.
+ Raid Level — уровень RAID.
+ Array Size — объем дискового пространства для RAID.
+ Used Dev Size — используемый объем для устройств. Для каждого уровня будет индивидуальный расчет: RAID1 — равен половине общего размера дисков, RAID5 — равен размеру, используемому для контроля четности.
+ Raid Devices — количество используемых устройств для RAID.
+ Total Devices — количество добавленных в RAID устройств.
+ Update Time — дата и время последнего изменения массива.
+ State — текущее состояние. clean — все в порядке.
+ Active Devices — количество работающих в массиве устройств.
+ Working Devices — количество добавленных в массив устройств в рабочем состоянии.
+ Failed Devices — количество сбойных устройств.
+ Spare Devices — количество запасных устройств.
+ Consistency Policy — политика согласованности активного массива (при неожиданном сбое). По умолчанию используется resync — полная ресинхронизация после восстановления. Также могут быть bitmap, journal, ppl.
+ Name — имя компьютера.
+ UUID — идентификатор для массива.
+ Events — количество событий обновления.
+ Chunk Size (для RAID5) — размер блока в килобайтах, который пишется на разные диски.


### Сохранение конфигурации массива

Так как по молчанию конфигурация массива никуда не сохраняется, потребуется её сохранить в конфигурационный файл mdadm.conf
Переходим в режим суперпользователя, т.к. потребуется полный доступ к командам доступным администратору и сохранению вывода в конфиггурацию системы

```
jahn@debian:~$ sudo -i
root@debian:~# mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
```

Для того чтобы имена массивов сохранялись при загрузке системы, обновляем образ начальной закрузки.

```
jahn@debian:~$ sudo update-initramfs -u 
update-initramfs: Generating /boot/initrd.img-6.1.0-12-amd64
```

### Разметка и создание файловой системы

Помечаем массив как GPT
Создаем 1 раздел, на весь объём массива.

```
sudo fdisk /dev/md1 - через интерактивное меню выполняем необходимые действия
```

```
jahn@debian:~$ lsblk 
NAME           MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda              8:0    0    8G  0 disk  
└─sda1           8:1    0    8G  0 part  
  └─md0          9:0    0    8G  0 raid1 
    ├─vg0-boot 253:0    0  488M  0 lvm   /boot
    ├─vg0-swap 253:1    0  976M  0 lvm   [SWAP]
    └─vg0-root 253:2    0  6.6G  0 lvm   /
sdb              8:16   0    8G  0 disk  
└─sdb1           8:17   0    8G  0 part  
  └─md0          9:0    0    8G  0 raid1 
    ├─vg0-boot 253:0    0  488M  0 lvm   /boot
    ├─vg0-swap 253:1    0  976M  0 lvm   [SWAP]
    └─vg0-root 253:2    0  6.6G  0 lvm   /
nvme0n1        259:0    0    2G  0 disk  
└─md1            9:1    0    4G  0 raid0 
  └─md1p1      259:3    0    4G  0 part  
nvme0n2        259:1    0    2G  0 disk  
└─md1            9:1    0    4G  0 raid0 
  └─md1p1      259:3    0    4G  0 part  
```

Раздел создан, создаём файловую систему

```
jahn@debian:~$ sudo mkfs.ext4 /dev/md1p1 
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 1046528 4k blocks and 261632 inodes
Filesystem UUID: 1920e290-3307-4fbb-b86d-bce539f2eca7
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```

Монтируем раздел в каталог mnt 

```
jahn@debian:~$ sudo mount /dev/md1p1 /mnt/
jahn@debian:~$ mount
....
/dev/md1p1 on /mnt type ext4 (rw,relatime,stripe=256)
```

Записываем тестовый файл на диск 

```
jahn@debian:~$ sudo dd if=/dev/urandom of=/mnt/test_data bs=4M count=512 status=progress 
2046820352 bytes (2.0 GB, 1.9 GiB) copied, 16 s, 128 MB/s
512+0 records in
512+0 records out
2147483648 bytes (2.1 GB, 2.0 GiB) copied, 16.6377 s, 129 MB/s
```

```
jahn@debian:~$ ls -lh /mnt/
total 2.1G
drwx------ 2 root root  16K Dec 22 02:56 lost+found
-rw-r--r-- 1 root root 2.0G Dec 22 03:00 test_data
```

### Удаляем массив

Останавливаем массив

```
jahn@debian:~$ sudo mdadm --stop /dev/md1
[sudo] password for jahn: 
mdadm: stopped /dev/md1
```

Очищаем суперблоки и обновляем конфигурацию массива в образе
```
sudo mdadm --zero-superblock /dev/nvme0n{1,2}
sudo update-initramfs -u
```

## 2. **RAID1**

### Для организации RAID1 будем использовать диски, ранее использованные для создания массива RAID0

```
# Просмотр блочных устройств
lsblk
# Затираем суперблоки и метаданные
sudo mdadm --zero-superblock --force /dev/nvme0n{1,2}
sudo wipefs --all --force /dev/nvme0n{1,2}
# Создаём массив уровня RAID1
sudo mdadm --create /dev/md1 --verbose --level=1 --raid-devices=2 /dev/nvme0n1 /dev/nvme0n2
# Просматриваем информацию о массиве
cat /proc/mdstat 
sudo mdadm --detail /dev/md1
# Формируем файл конфигурации массива
sudo -i
mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
sudo update-initramfs -u
# Создаём раздел и монтируем
sudo fdisk /dev/md1
sudo mkfs.ext4 /dev/md1p1
sudo mount /dev/md1p1 /mnt/
```

### Производим тестовую запись в раздел и разрушение/сборку массива

```
# Производим запись файла
sudo dd if=/dev/urandom of=/mnt/test_data bs=1M count=1800 status=progress
ls /mnt/ -lh
# Проверяем что массив работает
sudo mdadm --detail /dev/md1
# Помечаем один из дисков сбойным
sudo mdadm /dev/md1 --fail /dev/nvme0n1
# Проверяем состояние массива
sudo mdadm --detail /dev/md1
```
Находим следующую информацию State : clean, **degraded**

Перезаписываем файл, чистим сбойный диск, удаляем из массива

```
# Создаём файл
sudo dd if=/dev/urandom of=/mnt/test_data bs=1M count=18000 status=progress
# Выводим диск из массива
sudo mdadm /dev/md1 --remove /dev/nvme0n1
# Затираем суперблок и метаданные
sudo mdadm --zero-superblock --force /dev/nvme0n1
sudo wipefs --all --force /dev/nvme0n1
```

Добавляем диск обратно
```
sudo mdadm /dev/md1 --add /dev/nvme0n1
# Отслеживаем процес восстановления массива
cat /proc/mdstat
```
Какой может быть вывод. Также для удобства можно отслеживать через утилиту watch
```
jahn@debian:~$ cat /proc/mdstat 
Personalities : [raid1] [linear] [multipath] [raid0] [raid6] [raid5] [raid4] [raid10] 
md1 : active raid1 nvme0n1[2] nvme0n2[1]
      2094080 blocks super 1.2 [2/1] [_U]
      [====>................]  recovery = 23.8% (500416/2094080) finish=0.2min speed=125104K/sec
```

### Удаляем массив после выполнения работы
```
# Отмонтируем диск
sudo umount /dev/md1
# Удалим массив
sudo mdadm -S /dev/md1
# Чистим диски
sudo mdadm --zero-superblock /dev/nvme0n{1,2}
# Удаляем информацию из конфигурации и обновляем initramfs
sudo update-initramfs -u
```

## 2. **RAID5**

Для создания RAID5 требуется не менее 3 дисков, поэтому добавляем в систему еще пару дисков и приступаем к его с созданию.
Для отслеживания производительности массива во врем его восстановления, установим мултиплексов tmux

```
sudo apt install tmux
```

### Начинаем работу по созданию массива
```
lsblk
sudo mdadm --zero-superblock --force /dev/nvme0n{1,2,3,4}
sudo wipefs --all --force /dev/nvme0n{1,2,3,4}
sudo mdadm --create /dev/md1 --verbose --level=5 --raid-devices=4 /dev/nvme0n{1,2,3,4}
cat /proc/mdstat
sudo mdadm --detail /dev/md1
```

Через мультиплексор tmux открываем 2 терминала

```
tmux
ctrl + b -> shift + '   # создаём второе окно
ctrl + b -> up_arrow    #
```

Создаём файловую систему и монтируем

```
ctrl + b -> down_arrow
sudo mkfs.ext3 /dev/md1
sudo mount /dev/md1 /mnt/
# Запускаем бесконечный цикл записи в файл, начинаем отслеживать скорость записи в массив
while true; do dd if=/dev/urandom of=/mnt/rand bs=1M count=1000 status=progress; done
sudo mdadm /dev/md1 --fail /dev/nvme0n2
sudo mdadm /dev/md1 --remove /dev/nvme0n2
sudo wipefs --all --force /dev/nvme0n2
```

Добавляем диск обратно, отслеживаем процес восстановления массива и анализируем скорость записи в этот массив

```
sudo mdadm /dev/md1 --add /dev/nvme0n2 && watch cat /proc/mdstat # 

```

В ходе восстановления скорость записи на массив может снизиться в связи с синхронизацией всех дисков.




Источники:
http://xgu.ru/wiki/mdadm
https://www.dmosk.ru/miniinstruktions.php?mini=mdadm