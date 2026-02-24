------- git config - настройка Git -------


# Установить имя пользователя
git config --global user.name "Ваше Имя"

# Установить email
git config --global user.email "ваш@email.com"

# Просмотреть настройки
git config --list






------- git init - создать новый репозиторий -------

# Создать репозиторий в текущей папке
git init

# Создать репозиторий с веткой main (вместо master)
git init -b main






------- git status - проверить статус файлов -------

# Показать измененные, новые и удаленные файлы
git status

# Короткий формат
git status -s





------- git add - добавить файлы в отслеживание -------

# Добавить ВСЕ файлы
git add .

# Добавить конкретный файл
git add index.js

# Добавить все .js файлы
git add *.js

# Интерактивное добавление
git add -p





------- git commit - сохранить изменения -------

# Создать коммит с сообщением
git commit -m "Добавил корзину покупок"

# Добавить ВСЕ изменения и сделать коммит (осторожно!)
git commit -am "Быстрый коммит"

# Исправить последний коммит
git commit --amend





------- git log - просмотр истории коммитов -------

# Полная история
git log

# Краткий формат
git log --oneline

# История с графиком веток
git log --oneline --graph --all

# Показать изменения в файлах
git log -p





------- git diff - посмотреть изменения -------

# Показать недобавленные изменения
git diff

# Показать добавленные изменения
git diff --staged

# Показать изменения в конкретном коммите
git diff abc123





------- Работа с ветками -------
git branch - управление ветками


# Показать все ветки
git branch

# Создать новую ветку
git branch feature-payment

# Удалить ветку
git branch -d feature-payment

# Принудительно переименовать ветку
git branch -M main





------- git checkout - переключение между ветками -------

# Переключиться на ветку
git checkout main

# Создать и переключиться на новую ветку
git checkout -b feature-search

# Отменить изменения в файле
git checkout -- index.js





------- git switch - современная альтернатива checkout -------
# Переключиться на ветку
git switch main

# Создать и переключиться
git switch -c feature-auth





------- git merge - объединение веток -------

# Объединить feature в main
git switch main
git merge feature-payment





------- Работа с удаленным репозиторием -------
git remote - управление удаленными репозиториями


# Показать удаленные репозитории
git remote -v

# Добавить удаленный репозиторий
git remote add origin https://github.com/user/repo.git

# Удалить удаленный репозиторий
git remote remove origin





------- git push - отправить изменения на GitHub -------

# Отправить ветку в удаленный репозиторий
git push -u origin main

# Отправить все ветки
git push --all origin

# Принудительная отправка (осторожно!)
git push -f origin main





------- git pull - получить изменения с GitHub -------

# Получить изменения и объединить
git pull origin main

# Только получить изменения (без объединения)
git fetch origin




------- git clone - скопировать репозиторий -------

# Склонировать репозиторий
git clone https://github.com/user/repo.git

# Склонировать в конкретную папку
git clone https://github.com/user/repo.git my-project




------- git reset - отмена изменений -------

# Отменить добавление файлов (перед коммитом)
git reset

# Отменить последний коммит (сохранить изменения)
git reset --soft HEAD~1

# Полностью отменить последний коммит
git reset --hard HEAD~1






------- git stash - временное сохранение изменений -------

# Сохранить текущие изменения
git stash

# Посмотреть список сохранений
git stash list

# Вернуть сохраненные изменения
git stash pop

# Сохранить с именем
git stash save "Работа над корзиной"






------- git tag - работа с тегами -------

# Создать тег
git tag v1.0.0

# Создать тег с сообщением
git tag -a v1.0.0 -m "Релиз версии 1.0.0"

# Отправить теги на сервер
git push --tags




------- git clean - удалить неотслеживаемые файлы -------

# Показать что будет удалено
git clean -n

# Удалить неотслеживаемые файлы
git clean -f

# Удалить неотслеживаемые папки и файлы
git clean -fd







------- git rm - удалить файлы из репозитория -------

# Удалить файл
git rm old-file.js

# Удалить папку
git rm -r old-folder



[alias]
    co = checkout
    ci = commit
    st = status
    br = branch
    hist = log --pretty=format:\"%h %ad | %s%d [%an]\" --graph --date=short
    type = cat-file -t
    dump = cat-file -p


# Статус файлов в Git
U  new-file.js      # Серый - неотслеживаемый
M  modified.js      # Желтый - измененный  
A  staged-file.js   # Зеленый - в staging
D  deleted-file.js  # Красный - удаленный