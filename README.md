# **OTUS - Домашнее Задание #2: работа с mdadm**

**Цель:** получить знания о RAID массивах и их отличиях, научиться получать информацию о дисковой подсистеме на любом сервере с ОС Linux. Научиться собирать программный RAID и восстановить его после сбоя.

## **Задание**

- добавить в Vagrantfile еще дисков
- собрать R0/R5/R10 на выбор
- прописать собранный рейд в конф, чтобы рейд собирался при загрузке
- сломать/починить raid
- создать GPT раздел и 5 партиций


## **Ход выполнения**

**1. Добавление в Vagrantfile дополнительного диска 5 в секцию  :disks => {...}**
```
:sata5 => {
         :dfile => './sata5.vdi',
         :size => 250, # Megabytes
         :port => 5
 }
```

**2. Сборка RAID-5**
```
sudo lsblk
mdadm --zero-superblock --force /dev/sd{b,c,d,e,f}
mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd
```
...посмотрим, что получилось:
```[root@otuslinux vagrant]# mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Wed Dec 11 06:51:05 2019
        Raid Level : raid5
        Array Size : 761856 (744.00 MiB 780.14 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 4
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Wed Dec 11 17:17:09 2019
             State : clean 
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : 11fc047e:f0bfa094:9040e3fd:8023b5a6
            Events : 61

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       5       8       48        2      active sync   /dev/sdd
       4       8       64        3      active sync   /dev/sde
[root@otuslinux vagrant]# cat /proc/mdstat 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sde[4] sdd[2] sdb[0] sdc[1]
      761856 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/4] [UUUU]
 ```
 
 
**3. Прописание собранного RAID-5 в конф, чтобы собирался при загрузке:**
```
mkdir /etc/mdadm
mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' | tee /etc/mdadm/mdadm.conf
```


**4. Сломать / восстановить RAID**

- ломаем:
```
mdadm /dev/md0 --fail /dev/sdd
```
```
[root@otuslinux vagrant]# cat /proc/mdstat 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sde[4] sdd[2](F) sdb[0] sdc[1]
      761856 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/3] [UU_U]
```
- восстанавливаем:
```
mdadm /dev/md0 --remove /dev/sdd
mdadm /dev/md0 --add /dev/sdd
```
- смотрим:
```
cat /proc/mdstat
mdadm -D /dev/md0
```


**5. Создаем GPT раздел и 5 партиций

```
parted -s /dev/md0 mklabel gpt
parted /dev/md0 mkpart primary ext4 0% 15%
parted /dev/md0 mkpart primary ext4 15% 30%
parted /dev/md0 mkpart primary ext4 30% 45%
parted /dev/md0 mkpart primary ext4 45% 60%
parted /dev/md0 mkpart primary ext4 60% 75%
```
- оставляем часть пространства не размеченным на диске на всякий случай)

```
sudo mkfs.ext4 /dev/md0p1
sudo mkfs.ext4 /dev/md0p2
sudo mkfs.ext4 /dev/md0p3
sudo mkfs.ext4 /dev/md0p4
sudo mkfs.ext4 /dev/md0p5

```
                                           
