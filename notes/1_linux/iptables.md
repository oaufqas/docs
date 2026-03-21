##### IpTables - административный инструмент для настройки правил фильтрации пакетов (файервол в linux). Который управляет netfilter'ом Самое сложное в нем - понять архитектуру

**Netfilter** — это мощный фреймворк (набор инструментов), встроенный в ядро Linux, который обеспечивает фильтрацию сетевого трафика, преобразование сетевых адресов **([[ip-addressing#NAT-адресация (Network Address Translation)|NAT]])** и управление пакетами. Он работает как межсетевой экран (фаервол), обрабатывая входящие, исходящие и транзитные пакеты через цепочки правил, часто управляемые утилитой **iptables**.

Как правило, безопасность системы настраивается из идеологии: запрещено все, что не разрешено.

Правила `iptables` слетают после перезагрузки, потому что ==они хранятся только в оперативной памяти (RAM) и применяются "на лету"==.

**Как закрепить правила (сделать их постоянными):**

 **Сохранение правил (Debian/Ubuntu):**  

```bash
sudo apt-get install iptables-persistent -y
sudo systemctl enable netfilter-persistent

# Сохранение изменений
sudo netfilter-persistent save
# или ручной метод:
sudo sh -c "iptables-save > /etc/iptables/rules.v4"

# Загрузить в ядро, при ручном варианте
sudo sh -c "iptables-restore < /etc/iptables/rules.v4"
```

 **Сохранение правил (RHEL/CentOS/Fedora):**  

```bash
sudo dnf install iptables-services -y
sudo systemctl start/enable iptables

service iptables save
# или ручной метод:
iptables-save > /etc/sysconfig/iptables

# Загрузить правила в ядро
sudo iptables-restore < /etc/sysconfig/iptables
```

После этих действий правила будут автоматически загружаться из файлов `/etc/iptables/rules.v4` или `/etc/sysconfig/iptables` при старте системы.

---


![[Pasted image 20260317201851.png]]

**Цепочки (Chains) и Таблицы (Tables)**, нужно представить, что пакет — это посылка, которая проходит через сортировочный центр с несколькими этапами:

##### Таблицы — это разные сортировочные столы:
- `filter` — основной стол (разрешить/заблокировать) (таблица по умолчанию)
- `nat` — стол для подмены адресов
- `mangle` — стол для изменения пакетов
- `raw` — специальные настройки

##### Цепочки — это этапы, которые проходит пакет. Набор правил называется цепочкой, потому что правила читаются сверху вниз до первого совпадения, примеры цепочек:

`INPUT` — пакеты, идущие к твоему серверу (кто-то стучится)
`OUTPUT` — пакеты, идущие от твоего сервера (ты стучишься наружу)
`FORWARD` — пакеты, которые проходят транзитом (сервер как роутер)
`PREROUTING` — самый первый этап входа пакета в систему
`POSTROUTING` — этап отправки пакета после `FORWARD`

![[Pasted image 20260317203232.png]]

##### Политика (Policy) работы цепочки - это действия по умолчанию, если правил для пакета не было найдено, пример смены правил:

```bash
# Полный пример для веб-сервера
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp -m multiport --dports 22,80,443 -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/second -j ACCEPT
iptables -A INPUT -j LOG --log-prefix "IPTABLES-DROP: "
```

#### Синтаксис работы с утилитой iptables выглядит так:

![[Pasted image 20260317123344.png]]


### Основные флаги iptables:

```bash
#### Посмотреть все правила таблицы NAT
iptables -t nat -L -nv
-v   # подробный вывод
-vv  # еще подробнее
-n   # вывод в числах
-t   # таблица, по умолчанию filter
-x   # точные числа, без округления
--line-numbers   # Показывать номера строк
-S   # вывод правил в формате команд


iptables -A INPUT -p tcp --dport 443 -s 95.25.15.0/24 -j ACCEPT
#### Основные флаги действия
-A   # (Append) Добавить правило в конец цепочки
-I   # (Insert) Вставить правило в начало (можно с номером)
-D   # (Delete) Удалить правило (по номеру или тексту)
-R   # (Replace) Заменить правило по номеру
-L   # (List) Показать все правила в цепочке
-F   # (Flush) Удалить все правила из цепочки/таблицы
-Z   # (Zero) Обнулить счётчики пакетов/байт
-N   # (New chain) Создать новую пользовательскую цепочку
-X   # (Delete chain) Удалить пустую пользовательскую цепочку
-P   # (Policy) Установить политику по умолчанию для цепочки
-E   # (Rename chain) Переименовать пользовательскую цепочку


#### Базовые условия
-s   # (source) ip подключающегося устройства
-p   # Протокол (tcp, icmp)
-d   # (destination) ip устройства назначения
-i   # Входящий интерфейс (lo, eth0)
-o   # Исходящий интерфейс (wlan0)
-f   # Фрагментированные пакеты

--sport   # Source port (порт источника)
--dport   # Destination port (порт назначения)
--tcp-flags   # Проверка TCP-флагов
--syn         # SYN-пакеты (сокращение от --tcp-flags SYN,RST,ACK SYN)
--icmp-type   # Тип ICMP


#### Модули (-m)
-m state      # Проверка состояния соединения -m state --state NEW,ESTABLISHED
-m conntrack  # Современная проверка состояния -m conntrack --ctstate NEW
-m multiport  # Несколько портов -m multiport --dports 80,443,8080
-m limit      # Ограничение частоты -m limit --limit 10/minute
-m recent     # Список недавних соединений (для защиты) -m recent --set
-m mac        # MAC-адрес источника -m mac --mac-source 00:11:22:33:44:55
-m time       # Ограничение по времени -m time --timestart 09:00 --timestop 18:00
-m comment    # Комментарий к правилу -m comment --comment "Разрешить SSH"
-m iprange    # Диапазон IP-адресов -m iprange --src-range ip.a-192.168.1.200
-m length     # Длина пакета -m length --length 0:500
-m ttl        # TTL пакета -m ttl --ttl-gt 64


#### Цели (-j)
ACCEPT # Разрешить пакет
DROP # Просто удаление пакета, никаких признаков работы сервера с нашей стороны.
REJECT # Это ICMP ответ запрещенным подключениям, например о том, что подключение запрещено. Явное обозначение работы файервола.
LOG # Записать в лог (не прерывает обработку)
RETURN # Выйти из пользовательской цепочки
DNAT # Изменить адрес назначения (в nat таблице)
SNAT # Изменить адрес источника (в nat таблице)
MASQUERADE # Динамический SNAT для PPPoE/DHCP
REDIRECT # Перенаправить на локальный порт
MARK # Поставить метку на пакет
```

### Частоиспользуемые команды iptables:


```bash
# Политики по умолчанию (что делать, если нет правила)
iptables -P INPUT DROP    # По умолчанию блокировать входящие
iptables -P OUTPUT ACCEPT # Разрешать исходящие
iptables -A INPUT -i lo -j ACCEPT # Разрешить loopback

# Разрешить SSH (чтобы не заблокировать себя)
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Ограничить число подключений (защита от брутфорса)
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --set
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --update --seconds 60 --hitcount 4 -j DROP

# Разрешить только свой офисный IP на порт 443
iptables -A INPUT -p tcp --dport 443 -s 95.25.15.0/24 -j ACCEPT
iptables -A INPUT -p tcp -m multiport --dport 80, 443, 8080 -j DROP

# Разрешить уже установленные соединения
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Установить в начало правило, запрещающее определенному ip вход
iptables -I INPUT -s 209.175.153.23 -j DROP


#### NAT


# Маскарадинг (выход в интернет с локальных IP)
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Проброс порта 8080 снаружи на внутренний сервер 192.168.1.10:80
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.10:80

# Проброс с учётом обратного трафика
iptables -A FORWARD -p tcp -d 192.168.1.10 --dport 80 -j ACCEPT


#### Лимитирование


# Ограничить ICMP ping
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/second -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP

# Ограничить новые соединения на порт 80
iptables -A INPUT -p tcp --dport 80 -m conntrack --ctstate NEW -m limit --limit 50/minute -j ACCEPT


#### Удаление правил


# Удалить по номеру (сначала посмотреть с --line-numbers)
iptables -L INPUT --line-numbers
iptables -D INPUT 5

# Удалить конкретное правило (скопировать из iptables -S)
iptables -D INPUT -p tcp -m tcp --dport 22 -j ACCEPT

# Очистить всё (осторожно!)
iptables -F
iptables -t nat -F
iptables -t mangle -F
iptables -X

```
