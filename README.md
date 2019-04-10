# **Дисковая подсистема**

## **Homework**

- Работа с mdadm.
- Добавить в Vagrantfile еще дисков.
- Собрать R0/R5/R10 на выбор.
- Сломать/починить raid.
- Прописать собранный рейд в конф, чтобы рейд собирался при загрузке.
- Создать GPT раздел и 5 партиций.
- Vagrantfile, который сразу собирает систему с подключенным рейдом.

## 1. Работа с Vagrant

# vagrant -v
Vagrant 2.2.4

# vagrant box list
centos/7 (virtualbox, 1902.01)

# ls -a
.  ..  .gitignore  sata1.vdi  sata2.vdi  sata3.vdi  sata4.vdi  sata5.vdi  .vagrant  Vagrantfile

Создан Vagrantfile(приложен к OtusHW2 с добавленными дисками), значит можем поднимать нашу виртуальную машину. 
# vagrant up

Далее проверим поднялась ли машина:
# vagrant status
Current machine states:

otuslinux                 running (virtualbox)

Все работает корректно, можно подлючиться к ней:
# vagrant ssh
Last login: Tue Apr  9 09:37:23 2019 from 10.0.2.2
[vagrant@otuslinux ~]$ 

Видим что все прошло корректно:
[vagrant@otuslinux ~]$ lsscsi
[0:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sda 
[3:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sdb 
[4:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sdc 
[5:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sdd 
[6:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sde 
[7:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sdf 

# 2.Работа с mdadm, сборка RAID

Собираем из дисков RAID.Мы выбрали RAID 5.Опция -l какого уровня RAID создавать,опция - n указывает на кол-во устройств в RAID:
#mdadm --create --verbose /dev/md0 -l 5 -n 5 /dev/sd{b,c,d,e,f}

Смотрим состояние рэйда после сборки:
[vagrant@otuslinux ~]$ cat /proc/mdstat 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sdf[5] sde[6] sdd[2] sdc[1] sdb[0]
      1015808 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/5] [UUUUU]
      
unused devices: <none>

# 3.Работа с mdadm, поломать/починить RAID5

Ломаем:
[vagrant@otuslinux ~]$ sudo mdadm /dev/md0 --fail /dev/sde
mdadm: set /dev/sde faulty in /dev/md0

Видно что помечен теперь как попорченный:
[vagrant@otuslinux ~]$ watch cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md0 : active raid5 sdf[5] sde[6](F) sdd[2] sdc[1] sdb[0]
      1015808 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/4] [UUU_U]

Уберем его, чтобы заменить на новый:
[vagrant@otuslinux ~]$ sudo mdadm /dev/md0 --remove /dev/sde
[vagrant@otuslinux ~]$ sudo mdadm /dev/md0 --add /dev/sde
mdadm: added /dev/sde
[vagrant@otuslinux ~]$ cat /proc/mdstat 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sde[6] sdf[5] sdd[2] sdc[1] sdb[0]
      1015808 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/4] [UUU_U]
      [=================>...]  recovery = 87.7% (223488/253952) finish=0.0min speed=18624K/sec
      
unused devices: <none>

RAID снова собран успешно:
[vagrant@otuslinux ~]$ cat /proc/mdstat 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sde[6] sdf[5] sdd[2] sdc[1] sdb[0]
      1015808 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/5] [UUUUU]
      
unused devices: <none>

# 4.Прописываем собранный рейд в конф, чтобы рейд собирался при загрузке.

 Для того, чтобы быть уверенным что ОС запомнила какой RAID массив требуется создать и какие компоненты в него входят создадим файл mdadm.conf.
Сначала убедимся, что информация верна:
[vagrant@otuslinux ~]$ sudo mdadm --detail --scan --verbose
ARRAY /dev/md0 level=raid5 num-devices=5 metadata=1.2 name=otuslinux:0 UUID=b4d3da63:dc6dd4d1:493d0a01:3bded655
   devices=/dev/sdb,/dev/sdc,/dev/sdd,/dev/sde,/dev/sdf

Теперь создадим mdadm.conf:
[root@otuslinux vagrant]#echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
[root@otuslinux vagrant]#mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
Созданный файл:
[vagrant@otuslinux ~]$ cat /etc/mdadm/mdadm.conf
DEVICE partitions
ARRAY /dev/md0 level=raid5 num-devices=5 metadata=1.2 name=otuslinux:0 UUID=b4d3da63:dc6dd4d1:493d0a01:3bded655

# 5.Создание GPT раздела и 5 партиций

Создаем раздел GPT на RAID:
[root@otuslinux vagrant]#parted -s /dev/md0 mklabel gpt

Создаем партиции:
[root@otuslinux vagrant]#parted /dev/md0 mkpart primary ext4 0% 20%
[root@otuslinux vagrant]#parted /dev/md0 mkpart primary ext4 20% 40%
[root@otuslinux vagrant]#parted /dev/md0 mkpart primary ext4 40% 60%
[root@otuslinux vagrant]#parted /dev/md0 mkpart primary ext4 60% 80%
[root@otuslinux vagrant]#parted /dev/md0 mkpart primary ext4 80% 100%

Создаем на этих партициях фс: 
[root@otuslinux vagrant]#for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done

Монтируем их по каталогам:
[root@otuslinux vagrant]#mkdir -p /raid/part{1,2,3,4,5}
[root@otuslinux vagrant]#for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done

В итоге получаем следущее:
[vagrant@otuslinux ~]$ lsblk
NAME      MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda         8:0    0   40G  0 disk  
└─sda1      8:1    0   40G  0 part  /
sdb         8:16   0  250M  0 disk  
└─md0       9:0    0  992M  0 raid5 
  ├─md0p1 259:0    0  196M  0 md    
  ├─md0p2 259:1    0  198M  0 md    
  ├─md0p3 259:2    0  200M  0 md    
  ├─md0p4 259:3    0  198M  0 md    
  └─md0p5 259:4    0  196M  0 md    
sdc         8:32   0  250M  0 disk  
└─md0       9:0    0  992M  0 raid5 
  ├─md0p1 259:0    0  196M  0 md    
  ├─md0p2 259:1    0  198M  0 md    
  ├─md0p3 259:2    0  200M  0 md    
  ├─md0p4 259:3    0  198M  0 md    
  └─md0p5 259:4    0  196M  0 md    
sdd         8:48   0  250M  0 disk  
└─md0       9:0    0  992M  0 raid5 
  ├─md0p1 259:0    0  196M  0 md    
  ├─md0p2 259:1    0  198M  0 md    
  ├─md0p3 259:2    0  200M  0 md    
  ├─md0p4 259:3    0  198M  0 md    
  └─md0p5 259:4    0  196M  0 md    
sde         8:64   0  250M  0 disk  
└─md0       9:0    0  992M  0 raid5 
  ├─md0p1 259:0    0  196M  0 md    
  ├─md0p2 259:1    0  198M  0 md    
  ├─md0p3 259:2    0  200M  0 md    
  ├─md0p4 259:3    0  198M  0 md    
  └─md0p5 259:4    0  196M  0 md    
sdf         8:80   0  250M  0 disk  
└─md0       9:0    0  992M  0 raid5 
  ├─md0p1 259:0    0  196M  0 md    
  ├─md0p2 259:1    0  198M  0 md    
  ├─md0p3 259:2    0  200M  0 md    
  ├─md0p4 259:3    0  198M  0 md    
  └─md0p5 259:4    0  196M  0 md    

# 6.Vagrantfile, который сразу собирает систему с подключенным рейдом приложен в OtusHW2.

