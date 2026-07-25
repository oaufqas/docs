## Установка ansible в отдельной среде (изолированное окружение python)
#### Это хорошая практика, чтобы не мешать другим проектам и библиотекам

```bash
sudo apt update
# Устанавливаем зависимости
sudo apt install -y software-properties-common
# Добавляем официальный PPA
sudo add-apt-repository --yes --update ppa:ansible/ansible
# Устанавливаем Ansible
sudo apt install -y ansible


# Другой вариант через pip
python -m venv ansible_env

source ansible_env/bin/activate # Выйти из среды - deactivate

pip install ansible
```

## [[notes/1_linux/networking#SSH|Создание ключей]]

