### Процессы: `fork`, семейство `exec`, `clone`, `wait`

### `fork()` — создать копию процесса

**Сигнатура:** `pid_t fork(void);`

**Что делает:** Создаёт новый процесс — точную копию вызывающего. Копируется адресное пространство (по механизму Copy-on-Write), таблица файловых дескрипторов, обработчики сигналов. Оба процесса продолжают выполнение с точки вызова `fork`.

**Возвращает:**
- **Родителю:** PID (положительное число) созданного ребёнка.
- **Ребёнку:** `0`.
- **При ошибке:** `-1`, `errno` устанавливается (`EAGAIN` — лимит процессов, `ENOMEM` — нет памяти).

**Ключевая особенность:** после `fork` **оба** процесса выполняют один и тот же код. Различить их можно только по возвращаемому значению.

```c
pid_t pid = fork();
if (pid < 0) {
    perror("fork");
} else if (pid == 0) {
    // Это ребёнок
} else {
    // Это родитель, pid — PID ребёнка
}
```

**Цикл не нужен** — `fork` либо создаёт процесс, либо нет.

### Семейство `exec` — заменить образ процесса

**Общая идея:** `exec` **не создаёт новый процесс**. Он заменяет текущий образ (код, данные, стек) на новую программу. PID остаётся тем же. Если `exec` успешен, он **никогда не возвращается** — управление переходит в точку входа новой программы.

**`execve` — единственный настоящий системный вызов.** Все остальные — обёртки libc над ним.

**Сигнатура `execve` (низкоуровневый):**

```c
int execve(const char *pathname, char *const argv[], char *const envp[]);
```

- `pathname` — полный путь к исполняемому файлу.
- `argv` — массив аргументов, заканчивающийся `NULL`. `argv[0]` — имя программы.
- `envp` — массив переменных окружения, заканчивающийся `NULL`.

**Возвращает:** только при ошибке `-1` (при успехе не возвращается вообще).

**Семейство обёрток (суффиксы):**

|Функция|Формат аргументов|Поиск в `PATH`|Своё окружение|
|---|---|---|---|
|`execl`|список (variadic)|Нет|Нет (наследует)|
|`execv`|массив `argv[]`|Нет|Нет|
|`execlp`|список|**Да**|Нет|
|`execvp`|массив|**Да**|Нет|
|`execle`|список|Нет|**Да** (`envp`)|
|`execvpe`|массив|**Да**|**Да**|

- **`l`** (list) — аргументы передаются как отдельные параметры, список заканчивается `(char *)NULL`.
- **`v`** (vector) — аргументы передаются массивом.
- **`p`** (path) — искать файл в `PATH`, если нет `/` в имени.
- **`e`** (environment) — явно передать `envp`.

```c
// Пример: запустить ls -l
execlp("ls", "ls", "-l", (char *)NULL);
// То же через массив
char *argv[] = {"ls", "-l", NULL};
execvp("ls", argv);
```

**Ошибки:** `ENOENT` (файл не найден), `EACCES` (нет прав на выполнение), `ENOEXEC` (неверный формат).

```cpp
pid_t pid = fork();
if (pid == 0) {
    // 1. Умное формирование аргументов для защиты от пустых флагов
    char* process_args[3];
    process_args[0] = (char*)command.c_str();
    
    if (flags.empty()) {
        process_args[1] = NULL; // Если флагов нет — сразу завершаем массив
    } else {
        process_args[1] = (char*)flags.c_str();
        process_args[2] = NULL;
    }
    
    execvp(command.c_str(), process_args);
```

Зачем нужно преобразование `(char*)` перед `c_str()`?

Метод `command.c_str()` в C++ возвращает указатель типа **`const char*`** (данные только для чтения). Однако функция `execvp` требует на вход массив элементов типа **`char*`** (обычные указатели без `const`).

**Почему в Си-функциях `exec` нет слова `const` для аргументов?**  
Исторически в стандарте Си (POSIX) массив аргументов объявлен как `char *const argv[]`. Это означает, что _сам массив_ является константным (нельзя менять указатели местами), но строки, на которые он смотрит, константными не являются. Из-за этого компилятор C++ строго запрещает напрямую передавать `const char*` туда, где ожидается `char*` (нарушение контракта безопасности).

Явное приведение типов **`(char*)command.c_str()`** — это принудительное «отключение» константности (аналог `const_cast`). Вы говорите компилятору: _«Я знаю, что делаю. Закрой глаза на const, просто передай этот адрес памяти в execvp»_.

_(Примечание: Сама утилита `exec` никогда не меняет ваши строки аргументов, поэтому такое приведение здесь на 100% безопасно)._

### `clone()` — низкоуровневое создание процесса/потока


**Сигнатура (обёртка glibc):**

```c
int clone(int (*fn)(void *), void *stack, int flags, void *arg, ...);
```

**Что делает:** Создаёт новую задачу, но с **точным контролем** над тем, что разделяется между родителем и ребёнком. `fork` — это частный случай `clone` с определёнными флагами. Функция POSIX `pthread_create()` внутри тоже использует `clone` с `CLONE_VM | CLONE_FILES | CLONE_THREAD` и другими.

**Флаги (ключевые):**

|Флаг|Что разделяется|
|---|---|
|`CLONE_VM`|Адресное пространство (память)|
|`CLONE_FILES`|Таблица файловых дескрипторов|
|`CLONE_FS`|Информация о файловой системе (корень, cwd)|
|`CLONE_SIGHAND`|Обработчики сигналов|
|`CLONE_THREAD`|Считать это потоком того же процесса (один TGID)|
|`CLONE_NEWPID`|Новый PID namespace|
|`CLONE_NEWNET`|Новый сетевой namespace|

**Возвращает:** в родителе — PID ребёнка, в ребёнке — `0` (как `fork`), либо `-1` при ошибке.

**Отличие от `fork`:** в `clone` ребёнок начинает выполнение с функции `fn(arg)`, а не с точки вызова. Это позволяет избежать копирования стека и сразу запустить нужный код.

```c
void *child_stack = malloc(1024 * 1024);
int pid = clone(child_fn, child_stack + 1024*1024, CLONE_VM | SIGCHLD, NULL);
```

### `wait` и `waitpid` — дождаться завершения ребёнка

**Сигнатуры:**

```c
pid_t wait(int *wstatus);
pid_t waitpid(pid_t pid, int *wstatus, int options);
```

**Что делает:** Блокируется, пока один из дочерних процессов не завершится. Возвращает PID завершившегося ребёнка и записывает его статус в `wstatus`. Без этого вызова завершившийся ребёнок остаётся **зомби** (запись в таблице процессов, занимающая ресурсы).

**`waitpid` — более гибкий:**
- `pid > 0` — ждать конкретного ребёнка.
- `pid == -1` — ждать любого.
- `pid == 0` — ждать любого ребёнка из той же группы процессов.
- `options`: `WNOHANG` (не блокироваться, вернуть 0 если никто не завершился), `WUNTRACED`, `WCONTINUED`.

**Возвращает:** PID ребёнка, `0` при `WNOHANG` и отсутствии завершённых, `-1` при ошибке.

**Цикл нужен**, если детей несколько:

```c
while (waitpid(-1, &status, 0) > 0) {
    // обработать каждого завершившегося
}
```

### `exit` vs `_exit`

- `exit(status)` — библиотечная функция: вызывает `atexit`-обработчики, сбрасывает буферы stdio, закрывает потоки, затем вызывает `_exit`.
- `_exit(status)` — системный вызов: немедленно завершает процесс **без** сброса буферов и вызова обработчиков.

В дочернем процессе после `fork` обычно вызывают `_exit`, чтобы не сбросить буферы родителя дважды.

---

### Файлы: `open`, `read`, `write`, `close`, `dup2`, `fcntl`, `lseek`

### `open()` — открыть файл

**Сигнатура:**

```c
int open(const char *pathname, int flags, mode_t mode);
```

**Флаги:**

- `O_RDONLY`, `O_WRONLY`, `O_RDWR` — режим доступа (обязательно один).
- `O_CREAT` — создать, если нет (тогда нужен `mode`).
- `O_TRUNC` — обрезать до нуля.
- `O_APPEND` — писать в конец.
- `O_NONBLOCK` — неблокирующий режим.
- `O_CLOEXEC` — закрыть при `exec`.

**`mode`** (например, `0644`) используется только с `O_CREAT`. Реальные права = `mode & ~umask`.
**Возвращает:** файловый дескриптор (неотрицательное число) или `-1` при ошибке.
**Ошибки:** `ENOENT`, `EACCES`, `EEXIST` (с `O_CREAT|O_EXCL`).

### `read()` — прочитать из fd

**Сигнатура:**

```c
ssize_t read(int fd, void *buf, size_t count);
```

**Что делает:** Читает до `count` байт из текущей позиции файла (или из сокета/пайпа) в `buf`.

**Возвращает:**
- **`> 0`** — прочитано `n` байт.
- **`0`** — EOF (для файла) или соединение закрыто (для сокета/пайпа).
- **`-1`** — ошибка. `errno`: `EINTR` (прерван сигналом), `EAGAIN`/`EWOULDBLOCK` (неблокирующий режим, данных нет), `EBADF`, `EFAULT`.

**Может вернуть меньше `count`** — это нормально. Причины: сигнал прервал, буфер сокета меньше, файл короче. **Цикл для полного чтения нужен**, если вы знаете точный размер:

```c
ssize_t readall(int fd, void *buf, size_t count) {
    char *p = buf;
    size_t total = 0;
    while (total < count) {
        ssize_t n = read(fd, p + total, count - total);
        if (n == 0) return total;         // EOF
        if (n < 0) {
            if (errno == EINTR) continue;
            return -1;
        }
        total += n;
    }
    return total;
}
```

### `write()` — записать в fd

**Сигнатура:**

```c
ssize_t write(int fd, const void *buf, size_t count);
```

**Что делает:** Записывает до `count` байт из `buf`. Для обычных файлов обычно пишет всё сразу; для сокетов и пайпов **может записать меньше** (буфер отправки полон).

**Возвращает:** количество записанных байт, `-1` при ошибке. `EINTR`, `EAGAIN`, `EPIPE` (соединение закрыто — **плюс `SIGPIPE`**), `ENOSPC` (диск полон).

**Цикл для полной записи нужен всегда:**

```c
ssize_t writeall(int fd, const void *buf, size_t count) {
    const char *p = buf;
    size_t total = 0;
    while (total < count) {
        ssize_t n = write(fd, p + total, count - total);
        if (n < 0) {
            if (errno == EINTR) continue;
            return -1;
        }
        total += n;
    }
    return total;
}
```

### `close()` — закрыть fd

**Сигнатура:** `int close(int fd);`

**Что делает:** Уменьшает счётчик ссылок на открытый файл. Когда он достигает нуля — освобождает ресурсы, для сокетов отправляет FIN. **Не гарантирует**, что данные дошли до диска — для этого нужен `fsync`.

**Возвращает:** `0` или `-1`. `EINTR` — **не повторяйте `close` вслепую**, fd мог быть освобождён.

### `dup()` и `dup2()` — дублирование fd

**Сигнатуры:**

```c
int dup(int oldfd);
int dup2(int oldfd, int newfd);
int dup3(int oldfd, int newfd, int flags);
```

**Что делает:** Создаёт новый дескриптор, ссылающийся на **тот же открытый файл**. Общий file offset, общие флаги статуса файла.
- `dup` берёт **наименьший свободный** номер.
- `dup2` использует **заданный** `newfd`. Если он уже открыт — **молча закрывает** его. Атомарно — это важно, потому что `close` + `dup` не атомарны.

**Классическое применение:** перенаправление stdout/err в сокет или пайп.

```c
dup2(sockfd, STDOUT_FILENO);  // теперь printf идёт в сокет
dup2(sockfd, STDERR_FILENO);
close(sockfd);                 // оригинал больше не нужен
execvp(...);                  // новая программа унаследует fd 1 и 2 -> сокет
```

**Возвращает:** новый fd, `-1` при ошибке.

Пример с сервером по типу ssh:

Когда дочерний процесс делает `dup2(client_fd, 1)`, он **физически связывает дескриптор `stdout` (1) с сетевым сокетом**.

Затем вызывается `execvp`. Запущенная утилита (например, `ls` или `ping`) внутри своего Си-кода делает обычный вызов `write(1, ...)` или `printf(...)`. Но из-за `dup2` ядро Linux направляет эти байты **не на экран сервера, а напрямую в сетевую карту — клиенту!**

### `fcntl()` — управление fd

**Сигнатура:** `int fcntl(int fd, int cmd, ...);`

**Основные команды:**
- `F_GETFL` — получить флаги (например, `O_NONBLOCK`).
- `F_SETFL` — установить флаги.
- `F_GETFD`/`F_SETFD` — флаги дескриптора (`FD_CLOEXEC`).
- `F_DUPFD`/`F_DUPFD_CLOEXEC` — дублирование.

**Установка неблокирующего режима:**

```c
int flags = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);
```

### `lseek()` — переместить позицию в файле

**Сигнатура:** `off_t lseek(int fd, off_t offset, int whence);`

`whence`: `SEEK_SET`, `SEEK_CUR`, `SEEK_END`. Для сокетов и пайпов не работает (`ESPIPE`).

---

### [[CPP-Sockets|Сокеты]]: `socket`, `bind`, `listen`, `accept`, `connect`, `send`, `recv`, `shutdown`

### `socket()` — создать сокет

**Сигнатура:**

```c
int socket(int domain, int type, int protocol);
```

- `domain`: `AF_INET`, `AF_INET6`, `AF_UNIX`.
- `type`: `SOCK_STREAM` (TCP), `SOCK_DGRAM` (UDP), `SOCK_RAW`.
- `protocol`: обычно `0`.

**Возвращает:** fd или `-1`.

### `bind()` — привязать к адресу

**Сигнатура:**

```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

`sockaddr` — обобщённый тип; реально передаётся `sockaddr_in` (IPv4) или `sockaddr_in6` (IPv6). Приведение через `reinterpret_cast<sockaddr*>(&addr)`.

**Для сервера:** `sin_addr.s_addr = htonl(INADDR_ANY)` (все интерфейсы), `sin_port = htons(PORT)`.

**Возвращает:** `0` или `-1` (`EADDRINUSE`, `EACCES`).

### `listen()` — перевести в режим приёма

**Сигнатура:** `int listen(int sockfd, int backlog);`

`backlog` — размер очереди ожидающих соединений. `SOMAXCONN` — системный максимум.

**Возвращает:** `0` или `-1`.

### `accept()` — принять соединение

**Сигнатура:**

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
int accept4(int sockfd, struct sockaddr *addr, socklen_t *addrlen, int flags);
```

**Что делает:** Извлекает первое соединение из очереди, создаёт **новый** сокет для общения с этим клиентом. Слушающий сокет остаётся открытым.

**Возвращает:** новый fd или `-1`. `EAGAIN`/`EWOULDBLOCK` — если неблокирующий и нет соединений. `EINTR` — повторить.

`accept4` с `SOCK_NONBLOCK` и/или `SOCK_CLOEXEC` — атомарно создаёт неблокирующий сокет.

**Цикл нужен**, если хотите принять несколько соединений подряд:

```c
while (1) {
    int cfd = accept4(sfd, &addr, &len, SOCK_NONBLOCK);
    if (cfd < 0) {
        if (errno == EAGAIN) break;  // больше нет
        if (errno == EINTR) continue;
        perror("accept"); break;
    }
    // обработать cfd
}
```

### `connect()` — подключиться к серверу

**Сигнатура:**

```c
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

Для TCP выполняет трёхстороннее рукопожатие. Для неблокирующего сокета вернёт `-1` с `EINPROGRESS`, и надо ждать `POLLOUT` через `poll`/`epoll`.

**Возвращает:** `0` или `-1`.

### `send()` — отправить данные

**Сигнатура:**

```c
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
```

**Возвращает:** количество байт, записанных в буфер сокета, или `-1`.

**Ключевое:** **может записать меньше `len`**. Причины: буфер сокета полон (`EAGAIN` в неблокирующем режиме), `EINTR`. **Цикл обязателен:**

```c
ssize_t sendall(int fd, const void *buf, size_t len, int flags) {
    const char *p = buf;
    size_t total = 0;
    while (total < len) {
        ssize_t n = send(fd, p + total, len - total, flags | MSG_NOSIGNAL);
        if (n < 0) {
            if (errno == EINTR) continue;
            return -1;
        }
        if (n == 0) return total;
        total += n;
    }
    return total;
}
```

`MSG_NOSIGNAL` предотвращает `SIGPIPE` при разрыве соединения. Альтернатива: `signal(SIGPIPE, SIG_IGN)`.

**Ошибки:** `EPIPE` (удалённая сторона закрыла), `ECONNRESET`, `EAGAIN` (буфер полон, неблокирующий).

### `recv()` — принять данные

**Сигнатура:**

```c
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
```

**Возвращает:**

- **`> 0`** — прочитано `n` байт.
- **`0`** — удалённая сторона закрыла соединение (EOF). **Не ошибка.**
- **`-1`** — `EINTR` (повторить), `EAGAIN` (данных нет в неблокирующем режиме), другие — ошибка.

**Может вернуть меньше `len`** — TCP это поток байт. Границы сообщений не сохраняются. **Цикл для чтения полного сообщения нужен**, если знаете его длину:

```c
ssize_t recvall(int fd, void *buf, size_t len, int flags) {
    char *p = buf;
    size_t total = 0;
    while (total < len) {
        ssize_t n = recv(fd, p + total, len - total, flags);
        if (n == 0) return total;         // EOF
        if (n < 0) {
            if (errno == EINTR) continue;
            return -1;
        }
        total += n;
    }
    return total;
}
```

### `shutdown()` — закрыть одну сторону соединения

**Сигнатура:** `int shutdown(int sockfd, int how);`

`how`: `SHUT_RD`, `SHUT_WR`, `SHUT_RDWR`.

**Отличие от `close`:** `shutdown(SHUT_WR)` немедленно отправляет FIN, но fd остаётся открытым для чтения. Позволяет корректно завершить отправку и дождаться ответа.

### `sendto` / `recvfrom` — для UDP


```c
ssize_t sendto(int fd, const void *buf, size_t len, int flags,
               const struct sockaddr *dest, socklen_t addrlen);
ssize_t recvfrom(int fd, void *buf, size_t len, int flags,
                 struct sockaddr *src, socklen_t *addrlen);
```

UDP сохраняет границы датаграмм: один `sendto` = одна `recvfrom` (или потеря). Частичной отправки не бывает — либо вся датаграмма, либо ошибка (`EMSGSIZE`).

### `getaddrinfo()` — резолв имён

```c
int getaddrinfo(const char *node, const char *service,
                const struct addrinfo *hints, struct addrinfo **res);
void freeaddrinfo(struct addrinfo *res);
```

Рекомендуемый способ преобразования имени хоста/порта в адрес. Работает с IPv4 и IPv6. **Всегда вызывайте `freeaddrinfo`** после использования.

### `inet_pton()` / `inet_ntop()` — IP строка ↔ бинарный

```c
int inet_pton(int af, const char *src, void *dst);
const char *inet_ntop(int af, const void *src, char *dst, socklen_t size);
```

`inet_pton`: `"127.0.0.1"` -> 4 байта в `sin_addr.s_addr`. `inet_ntop`: обратно.

### `setsockopt()` / `getsockopt()`

```c
int setsockopt(int fd, int level, int optname, const void *optval, socklen_t optlen);
```

Ключевые опции: `SO_REUSEADDR` (обязательно для сервера), `SO_REUSEPORT`, `TCP_NODELAY`, `SO_RCVTIMEO`/`SO_SNDTIMEO`, `SO_KEEPALIVE`.

---

### Мультиплексирование: `select`, `poll`, `epoll`

### `select()`

```c
int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);
```

- `nfds` = максимальный fd + 1.
- `fd_set` — битовая маска до `FD_SETSIZE` (обычно 1024).
- **`O(N)`** — ядро сканирует все fd.
- **Модифицирует** `fd_set` на месте — нужно пересоздавать каждый вызов.

**Возвращает:** количество готовых fd, `0` при таймауте, `-1` при ошибке.

### `poll()`

```c
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

```c
struct pollfd {
    int   fd;
    short events;   // что интересует
    short revents;  // что произошло (заполняет ядро)
};
```

Нет лимита 1024. Всё ещё **`O(N)`**.

### `epoll` — `O(1)` на событие

```c
int epfd = epoll_create1(0);
```

**Регистрация:**

```c
struct epoll_event ev;
ev.events = EPOLLIN;
ev.data.fd = fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);
```

**Ожидание:**

```c
struct epoll_event events[MAX];
int n = epoll_wait(epfd, events, MAX, timeout);
```

**Level-Triggered (LT):** пока данные есть, событие повторяется. По умолчанию.

**Edge-Triggered (ET):** `EPOLLET` — событие только при изменении состояния. **Обязательно** читать до `EAGAIN` в цикле, иначе потеряете данные.

```c
// ET: читать пока не EAGAIN
while (1) {
    ssize_t n = recv(fd, buf, sizeof(buf), 0);
    if (n > 0) { process(buf, n); }
    else if (n == 0) { close(fd); break; }
    else {
        if (errno == EAGAIN || errno == EWOULDBLOCK) break;  // всё
        if (errno == EINTR) continue;
        break;
    }
}
```

---

### Пайпы и другие IPC

### `pipe()` — неименованный канал

```c
int pipe(int pipefd[2]);
```

`pipefd[0]` — чтение, `pipefd[1]` — запись. Односторонний. Для двустороннего — два пайпа или `socketpair`.

**Возвращает:** `0` или `-1`.

### `socketpair()` — двусторонний канал

```c
int socketpair(int domain, int type, int protocol, int sv[2]);
```

`domain = AF_UNIX`, `type = SOCK_STREAM` или `SOCK_DGRAM`. Два связанных сокета.

### `eventfd`, `timerfd`, `signalfd`

- `eventfd` — счётчик для уведомлений между потоками/процессами.
- `timerfd` — таймер как fd, интегрируется в epoll.
- `signalfd` — сигналы как fd, читаются через `read`.

### `inotify` — слежение за файлами

```c
int fd = inotify_init1(IN_NONBLOCK);
int wd = inotify_add_watch(fd, "/path", IN_MODIFY | IN_CREATE);
// читать struct inotify_event через read(fd, ...)
```

---

### Прочее..

### `mmap` / `munmap` — отображение файлов в память

```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
int munmap(void *addr, size_t length);
```

`prot`: `PROT_READ`, `PROT_WRITE`, `PROT_EXEC`. 
`flags`: `MAP_SHARED`, `MAP_PRIVATE`, `MAP_ANONYMOUS`.

### `stat` / `fstat` — информация о файле

```c
int stat(const char *path, struct stat *buf);
int fstat(int fd, struct stat *buf);
```

Заполняет `struct stat`: размер, права, владелец, время.

### `access` — проверить права

```c
int access(const char *path, int mode);
```

`mode`: `R_OK`, `W_OK`, `X_OK`, `F_OK`. **Небезопасен** для проверки перед открытием (TOCTOU). Лучше просто попытаться открыть.

### `fcntl` — не только флаги

Также используется для:
- Блокировок файлов (`F_SETLK`, `F_SETLKW`).
- Дублирования (`F_DUPFD`).
- Управления асинхронным I/O (`F_SETOWN`, `F_SETFL` с `O_ASYNC`).

### `ioctl` — управление устройством

```c
int ioctl(int fd, unsigned long request, ...);
```

Для сокетов, терминалов, устройств. Например, `ioctl(sockfd, FIONREAD, &n)` — сколько байт доступно для чтения.

### `kill` / `raise` / `signal` / `sigaction`

```c
int kill(pid_t pid, int sig);
int raise(int sig);
sighandler_t signal(int signum, sighandler_t handler);
int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);
```

`sigaction` — предпочтительный способ установки обработчиков. `signal` — упрощённый, поведение зависит от реализации.

### `getpid` / `getppid` / `getuid` / `getgid`

```c
pid_t getpid(void);    // PID текущего процесса
pid_t getppid(void);   // PID родителя
uid_t getuid(void);    // реальный UID
gid_t getgid(void);    // реальный GID
```

### `sched_yield` и `nanosleep`

```c
int sched_yield(void);                     // уступить CPU
int nanosleep(const struct timespec *req, struct timespec *rem);
```

### `syscall()` — прямой вызов

```c
long syscall(long number, ...);
```

Позволяет вызвать системный вызов, для которого нет обёртки libc. Номера — в `<sys/syscall.h>` (`SYS_read`, `SYS_write`, ...).

---

## Сводная таблица: где нужен цикл

|Вызов|Цикл нужен?|Почему|
|---|---|---|
|`read`|**Да**, если знаете размер|Может вернуть меньше|
|`write`|**Да**|Может записать меньше|
|`send`|**Да**|Может записать меньше|
|`recv`|**Да**, если знаете размер|Может вернуть меньше|
|`recvfrom` (UDP)|Нет|Одна датаграмма целиком|
|`sendto` (UDP)|Нет|Либо вся, либо ошибка|
|`accept`|**Да**, если nonblocking|`EAGAIN` = больше нет|
|`epoll_wait`|Нет (вызов в цикле)|Возвращает пачку событий|
|`fork`|Нет|Один вызов = один ребёнок|
|`exec*`|Нет (не возвращается)|При успехе не возвращает|
|`close`|Нет|Один вызов|
|`dup2`|Нет|Один вызов|

---

## Общие правила обработки системных вызовов

1. **Всегда проверяйте возвращаемое значение.** `-1` — ошибка, кроме случаев, где `0` или другое значение имеет смысл (например, `read == 0` — EOF).
    
2. **`EINTR` — не ошибка, а сигнал.** Повторяйте вызов. Это особенно важно при работе с `SIGINT`, `SIGCHLD`, `SIGWINCH`.
    
3. **`EAGAIN`/`EWOULDBLOCK` — не ошибка.** Для неблокирующих fd означает «попробуй позже».
    
4. **Для сокетов используйте `MSG_NOSIGNAL`** или игнорируйте `SIGPIPE`, иначе процесс умрёт при разрыве.
    
5. **Для сервера всегда `SO_REUSEADDR`**, иначе `bind` упадёт после рестарта из-за `TIME_WAIT`.
    
6. **Не забывайте `close`.** Каждый открытый fd — ресурс. Утечка дескрипторов — классическая проблема долгоживущих серверов.
    
7. **RAII в C++.** Оборачивайте fd, сокеты, потоки в классы с деструктором, чтобы не забывать освобождать.