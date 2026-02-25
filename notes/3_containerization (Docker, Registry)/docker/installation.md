# Установка bash скрипта, который устанавливает docker + plugin compose
curl -fsSl https://get.docker.com | sudo bash

# Создать группу docker (если её ещё нет)
sudo groupadd docker

# Добавить текущего пользователя в группу docker
sudo usermod -aG docker $USER

# Активировать изменения в текущей сессии
newgrp docker