| Команда |	Описание                | Пример
|---------|-------------------------|--------------------------
| df	  | Использование дисков    | df -h (человекочитаемый)
| du	  | Размер директорий	    | du -sh * (суммарно)
| mount	  | Монтирование	        | mount /dev/sda1 /mnt
| umount  | Размонтирование	        | umount /mnt
| fdisk	  | Работа с разделами      | fdisk -l (список)
| lsblk	  | Список блочных устройств| lsblk
| blkid	  | UUID устройств	        | blkid
| fsck    | Проверка ФС	            | fsck /dev/sda1
| sync	  | Сброс кэша на диск	    | sync



### Сжатие файлов:


tar -czf archive.tar.gz dir/     # Создать tar.gz
tar -xzf archive.tar.gz          # Распаковать tar.gz

tar -cjf archive.tar.bz2 dir/    # Создать tar.bz2
tar -xjf archive.tar.bz2          # Распаковать tar.bz2

zip -r archive.zip dir/          # Создать zip
unzip archive.zip                # Распаковать zip

gzip file                        # Сжать файл
gunzip file.gz                   # Распаковать gz