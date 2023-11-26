### Задание
● Добавить в Vagrantfile еще дисков.
● Собрать RAID-массив уровня 0/5/10 на выбор.
● Прописать собранный массив в конфиг, чтобы он собирался при загрузке
● Сломать/починить RAID.
● Создать GPT-таблицу и 5 разделов поверх массива, смонтировать их в
системе.

Копируем нужный Vagrantfile из репозитория преподователя, добавляем в него ещё один диск:

```:sata5 => {
:dfile => './sata5.vdi', # Путь, к файлу диска
:size => 250, # Размер диска в мегабайтах
:port => 5 # Номер порта, к которому будет подключен диск
}```

Запускаем виртуальную машину, подключаемся к ней и приступаем к выполнению ДЗ: `vagrant up`, `vagrant ssh`

Смотрим, какие блочные устройства у нас есть: `sudo lshw -short | grep disk`
![Скрин](https://github.com/FeeLinS9/lesson3/blob/master/picture1.png)

Обнуляем суперблоки `sudo mdadm --zero-superblock --force /dev/sd{b,c,d,e,f}`

Создаём новый RAID-массив следующей командой: `sudo mdadm --create --verbose /dev/md0 -l 6 -n 5 /dev/sd{b,c,d,e,f}`

Создадим файл mdadm.conf с описанием нашего массива.
```mkdir /etc/mdadm
echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> \
/etc/mdadm/mdadm.conf```
![Скрин](https://github.com/FeeLinS9/lesson3/blob/master/picture2.png)

Ломаем RAID `sudo mdadm /dev/md0 --fail /dev/sde`
Смотрим: `cat /proc/mdstat`
![Скрин](https://github.com/FeeLinS9/lesson3/blob/master/picture3.png)

Удалим “сломанный” диск из массива `sudo mdadm /dev/md0 --remove /dev/sde`

Представим, что мы вставили новый диск в сервер и теперь нам нужно добавить его
в RAID. `sudo mdadm /dev/md0 --add /dev/sde`

### Создаём GPT-таблицу и 5 разделов, монтируем их в систему.

Создаем таблицу разделов GPT `sudo parted -s /dev/md0 mklabel gpt`

Создаём разделы на массиве:
```sudo parted /dev/md0 mkpart primary ext4 0% 20%
sudo parted /dev/md0 mkpart primary ext4 20% 40%
sudo parted /dev/md0 mkpart primary ext4 40% 60%
sudo parted /dev/md0 mkpart primary ext4 60% 80%
sudo parted /dev/md0 mkpart primary ext4 80% 100%```

Создаём на этих разделах файловые системы `for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done`
Монтируем их по каталогам:
```mkdir -p /raid/part{1,2,3,4,5}
for i in $(seq 1 5); do sudo mount /dev/md0p$i /raid/part$i; done```

Проверяем результат `df -h`
![Скрин](https://github.com/FeeLinS9/lesson3/blob/master/picture4.png)