# Установка bash скрипта, который устанавливает docker + plugin compose
curl -fsSl https://get.docker.com | sudo bash

# Создать группу docker (если её ещё нет)
sudo groupadd docker

# Добавить текущего пользователя в группу docker
sudo usermod -aG docker $USER

# Активировать изменения в текущей сессии
newgrp docker

Чтобы работать с docker на windows нужно устанавливать docker desktop и wsl (Подсистема Linux для Windows), по сравнению с работой на linux, на windows он будет потреблять намного больше ресурсов, чтобы создать скрытую виртуальную машину Linux. Также чтобы он работал, приложение docker desktop всегда должно быть запущено, (задачей или процессом не важно)