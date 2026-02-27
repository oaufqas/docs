| Команда	 | Описание	                  | Пример                         |
|------------|----------------------------|--------------------------------|
| ip	     | Настройка сети	          | ip addr show (IP адреса)       |
| ifconfig	 | Устаревшая	              | ifconfig -a                    |
| ping	     | Проверка доступности	      | ping -c 4 google.com           |
| netstat	 | Сетевые соединения	      | netstat -tulpn                 |
| ss	     | Современная замена netstat |	ss -tulpn                      |
| curl	     | HTTP запросы	              | curl -I http://example.com     |
| wget	     | Скачивание	              | wget https://example.com/file  | 
| traceroute | Маршрут	                  | traceroute google.com          |
| nslookup	 | DNS запросы	              | nslookup google.com            |
| dig	     | Расширенный DNS	          | dig google.com                 |
| nc	     | Сетевой Swiss Army Knife	  | nc -zv host port               |
| iptables	 | Файрвол	                  | iptables -L -n                 |
| ufw	     | Простой файрвол	          | ufw status                     |
| arp        | Просмотр ARP-таблицы       | arp -a      arp -scan          |

### SSH:

ssh user@host                    # Подключиться
ssh -p 2222 user@host            # На другой порт
ssh -i key.pem user@host         # С ключом
ssh-copy-id user@host             # Скопировать ключ

# Туннелирование
ssh -L 8080:localhost:80 user@host  # Проброс порта
ssh -D 1080 user@host                # SOCKS прокси

# Копирование через SSH
scp file user@host:/path/        # Скопировать файл с хоста на целевой сервер
scp -r dir user@host:/path/      # Скопировать папку с хоста на целевой сервер
scp user@host:/path/file path/   # Скопировать файл с целевого сервера на хост
scp -r user@host:/path/ path/   # Скопировать папку с целевого сервера на хост
rsync -av dir/ user@host:/path/  # Синхронизация





### ----------------------- Файерволы в Linux -----------------------


            ┌─────────────────┐
            │   iptables      │  ← классика (сложно, но мощно)
            ├─────────────────┤
            │   nftables      │  ← новый стандарт (пришел на смену)
            ├─────────────────┤
            │   ufw           │  ← надстройка над iptables (Ubuntu)
            ├─────────────────┤
            │   firewalld     │  ← надстройка над nftables (RHEL/CentOS)
            └─────────────────┘

## Iptables (глубокое погружение)
Iptables — это утилита для настройки правил фильтрации пакетов. Самое сложное в нем — понять архитектуру.

**Цепочки (Chains) и Таблицы (Tables)**, нужно представить, что пакет — это посылка, которая проходит через сортировочный центр с несколькими этапами:
- Таблицы — это разные сортировочные столы:
- filter — основной стол (разрешить/заблокировать)
- nat — стол для подмены адресов
- mangle — стол для изменения пакетов
- raw — специальные настройки

Цепочки — это этапы, которые проходит пакет:

                     ┌───────────┐
    ──> Пакет входит │ PREROUTING│ (nat/mangle/raw)
                     └───────────┘
                        │
                        ▼
                  Принимать решение?
                        │
           ┌────────────┴─────────────┐
           │                          │                                                   
           ▼                          ▼
    ┌───────────┐               ┌───────────┐
    │ FORWARD   │               │  INPUT    │  ← для пакетов САМОМУ СЕРВЕРУ
    │ (транзит) │               └───────────┘
    └───────────┘                     │
           │                          ▼
           │                    ┌───────────┐
           │                    │  OUTPUT   │  ← от сервера наружу
           │                    └───────────┘
           │                          │
           └────────────┬─────────────┘
                        ▼
                  ┌───────────┐
                  │POSTROUTING│ (nat/mangle)  ──> Пакет уходит
                  └───────────┘

INPUT — пакеты, идущие к твоему серверу (кто-то стучится)

OUTPUT — пакеты, идущие от твоего сервера (ты стучишься наружу)

FORWARD — пакеты, которые проходят транзитом (сервер как роутер)



### Базовые команды iptables

# Посмотреть все правила
iptables -L -v
iptables -t nat -L -v    # Посмотреть таблицу NAT

# Политики по умолчанию (что делать, если нет правила)
iptables -P INPUT DROP    # По умолчанию блокировать входящие
iptables -P OUTPUT ACCEPT # Разрешать исходящие

# Разрешить SSH (чтобы не заблокировать себя)
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Разрешить уже установленные соединения
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Заблокировать конкретный IP
iptables -A INPUT -s 192.168.1.100 -j DROP

# Разрешить только свой офисный IP на порт 443
iptables -A INPUT -p tcp --dport 443 -s 95.25.15.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j DROP

# Сохранить правила (чтобы не сбросились после перезагрузки)
iptables-save > /etc/iptables/rules.v4


### Основное руководство по использованию iptables/nftables будет в ../2_networks/iptables.md и ../2_networks/nftables.md соответственно