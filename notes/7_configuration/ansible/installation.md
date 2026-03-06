## Установка ansible в отдельной среде (изолированное окружение python)
#### Это хорошая практика, чтобы не мешать другим проектам и библиотекам

```bash
python -m venv ansible_env

source ansible_env/bin/activate # Выйти из среды - deactivate

pip install ansible
```

## [[networking-firewall#SSH|Создание ключей]]

