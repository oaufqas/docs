### Класс и объект

**Класс** — это пользовательский тип: набор данных (поля) + функций (методы), работающих с этими данными. **Объект** — экземпляр класса.

```cpp
class Point {
    double x_;   // поле (данные)
    double y_;
public:
    double length() const {   // метод
        return std::sqrt(x_*x_ + y_*y_);
    }
};
Point p;   // объект
```

`struct` и `class` в C++ — одно и то же, отличие только в доступе по умолчанию: у `struct` — `public`, у `class` — `private`.

```cpp
struct A { int x; };   // x публичное
class  B { int x; };   // x приватное
```

### Инкапсуляция: модификаторы доступа

Три уровня:

|Модификатор|Кто видит|
|---|---|
|`public`|Все|
|`protected`|Сам класс и наследники|
|`private`|Только сам класс|


```cpp
class BankAccount {
private:
    double balance_ = 0;
protected:
    void log(const std::string& msg) { /* ... */ }
public:
    double balance() const { return balance_; }
    void deposit(double amount) {
        if (amount > 0) balance_ += amount;
    }
};
```

**Зачем инкапсуляция:** клиент работает только через публичный интерфейс, внутреннее устройство можно менять. Это главная идея ООП — «чёрный ящик».

**Правило:** поля — `private`, доступ через методы. Исключение — простые структуры-данные (`struct Point { double x, y; }`).

### Конструкторы и деструктор

>Конструктор ызывается при создании объекта. Имя совпадает с именем класса, нет возвращаемого типа.

```cpp
class Point {
    double x_, y_;
public:
    Point() : x_(0), y_(0) {}               // по умолчанию
    Point(double x, double y) : x_(x), y_(y) {}  // с параметрами
};
```

**Список инициализации** (`: x_(x), y_(y)`) — обязателен для `const`-полей и ссылок, эффективнее присваивания в теле.

#### Виды конструкторов

```cpp
Point() = default;                            // компилятор сгенерирует
Point(const Point& other) = default;          // копирующий
Point(Point&& other) noexcept = default;      // перемещающий
Point(double x = 0, double y = 0);            // с параметрами по умолчанию
explicit Point(double x);                     // запрещает неявное преобразование
```

**`explicit`** — запрещает неявное преобразование `double -> Point`. Всегда ставьте, если конструктор принимает один аргумент.

```cpp
Point p = 5.0;       // ошибка, если конструктор explicit
Point p(5.0);        // ок
```


>Деструктор вызывается при уничтожении объекта. Имя — `~Имя`, без параметров.

```cpp
class File {
    FILE* f_;
public:
    File(const char* path) : f_(fopen(path, "r")) {}
    ~File() { if (f_) fclose(f_); }   // освобождение ресурса
};
```

**RAII** (Resource Acquisition Is Initialization) — главный паттерн C++: ресурс захватывается в конструкторе, освобождается в деструкторе. Работает автоматически при выходе из области видимости, при исключениях, при разрушении контейнера. Так устроены `std::string`, `std::vector`, `std::unique_ptr`, `std::lock_guard`.

### Инициализация полей

```cpp
class A {
    int x_ = 0;              // инициализация по умолчанию (C++11)
    std::string s_ = "abc";  // тоже
    int y_;                  // не инициализировано — мусор!
};
```

Совет: **инициализируйте все поля**, либо через `= значение`, либо через список инициализации.

#### `this`, `static`, `const`

Внутри метода `this` — указатель на текущий объект.

```cpp
class Point {
    double x_, y_;
public:
    Point& setX(double x) {
        this->x_ = x;     // явно, чтобы отличить от параметра
        return *this;     // вернуть себя — для цепочек
    }
};
```

`static`-члены. Принадлежат классу, а не объекту. Одно значение на все экземпляры.

```cpp
class Counter {
    static int count_;    // объявление
public:
    Counter()  { ++count_; }
    ~Counter() { --count_; }
    static int get() { return count_; }
};
int Counter::count_ = 0;  // определение (C++17: inline static int count_ = 0;)
```

Обращение: `Counter::get()`, а не `obj.get()`. **`static`-метод** не имеет `this` и не может обращаться к нестатическим полям.

`const`-методы. Метод, который не меняет объект. Внутри `this` — `const T*`.

```cpp
class Point {
    double x_, y_;
public:
    double length() const {   // const после скобок
        return std::sqrt(x_*x_ + y_*y_);
    }
    void move(double dx, double dy) {  // не const — меняет поля
        x_ += dx; y_ += dy;
    }
};
const Point p(3, 4);
p.length();   // ок
// p.move(1, 1);  // ошибка: move не const
```

**Правило:** если метод не меняет объект — пометьте `const`. Это нужно для работы с `const`-объектами и `const`-ссылками.

### `mutable`

Поле, которое можно менять даже в `const`-методе. Используется редко — для кэшей, счётчиков.

```cpp
class Cache {
    mutable int hits_ = 0;
public:
    int get(int x) const { ++hits_; return x*2; }
};
```

### Наследование

```cpp
class Animal {
public:
    void breathe() { std::cout << "breathe\n"; }
};
class Dog : public Animal {   // Dog наследует Animal
public:
    void bark() { std::cout << "woof\n"; }
};
Dog d;
d.breathe();   // унаследовано
d.bark();      // своё
```

#### Виды наследования

|Вид|Что становится с `public`/`protected` базового|
|---|---|
|`public`|остаётся `public`/`protected` (обычное наследование «является»)|
|`protected`|становится `protected`|
|`private`|становится `private` (по умолчанию для `class`)|

На практике почти всегда используют `public`.

#### Вызов конструктора базового класса

```cpp
class Animal {
protected:
    std::string name_;
public:
    Animal(std::string name) : name_(std::move(name)) {}
};
class Dog : public Animal {
public:
    Dog(std::string name) : Animal(std::move(name)) {}   // вызов базового
};
```

Базовый конструктор вызывается **первым**, потом поля наследника, потом тело конструктора наследника. Деструкторы — в обратном порядке.

#### `override` и `final` (C++11)

```cpp
class Base {
public:
    virtual void f() {}
    virtual void g() final {}   // нельзя переопределить
};
class Derived : public Base {
public:
    void f() override {}   // явно переопределяем
    // void g() override {}   // ошибка: final
};
```

Всегда ставьте `override` — компилятор поймает опечатку в имени.

### Полиморфизм

Главная идея ООП: **один интерфейс — много реализаций**. Работаешь с базовым классом, получаешь поведение наследника.
### `virtual`

Без `virtual` вызов метода определяется **типом указателя/ссылки** (статический полиморфизм). С `virtual` — **типом реального объекта** (динамический полиморфизм).

```cpp
class Animal {
public:
    virtual void speak() { std::cout << "...\n"; }
    virtual ~Animal() = default;   // ВАЖНО
};
class Dog : public Animal {
public:
    void speak() override { std::cout << "woof\n"; }
};
class Cat : public Animal {
public:
    void speak() override { std::cout << "meow\n"; }
};
void make_speak(Animal& a) { a.speak(); }   // вызывается по реальному типу
Dog d; Cat c;
make_speak(d);   // woof
make_speak(c);   // meow
```

#### Виртуальный деструктор

**Если класс имеет виртуальные методы — деструктор должен быть виртуальным.** Иначе при удалении через указатель на базу не вызовется деструктор наследника → утечка.

```cpp
class Base {
public:
    virtual ~Base() = default;
};
std::unique_ptr<Base> p = std::make_unique<Dog>();
// без virtual ~Base() деструктор Dog не вызовется — утечка
```

#### Чисто виртуальные методы и абстрактные классы

```cpp
class Shape {
public:
    virtual double area() const = 0;   // = 0 — чисто виртуальный
    virtual ~Shape() = default;
};
```

Класс с хотя бы одним чисто виртуальным методом — **абстрактный**. Создать объект нельзя, только наследовать.

```cpp
class Circle : public Shape {
    double r_;
public:
    Circle(double r) : r_(r) {}
    double area() const override { return 3.14159 * r_ * r_; }
};
// Shape s;            // ошибка: абстрактный
Shape* p = new Circle(5);  // ок
```

#### Интерфейс

В C++ нет ключевого слова `interface`. Интерфейс — это абстрактный класс **без полей** и **только с чисто виртуальными методами** и виртуальным деструктором.

```cpp
class ILogger {
public:
    virtual void log(const std::string& msg) = 0;
    virtual ~ILogger() = default;
};
```

### `override`, `final`, `virtual` — итог

- `virtual` — в **базовом** классе, разрешает переопределение.
- `override` — в **наследнике**, проверяет, что реально переопределяем.
- `final` — запрещает дальнейшее переопределение (или наследование, если на классе).

### Копирование и перемещение

#### Копирующий конструктор и оператор присваивания

```cpp
class Buffer {
    char* data_;
    size_t size_;
public:
    Buffer(const Buffer& other)                    // копирующий конструктор
        : data_(new char[other.size_]), size_(other.size_) {
        std::memcpy(data_, other.data_, size_);
    }
    Buffer& operator=(const Buffer& other) {       // оператор присваивания
        if (this != &other) {                      // самоприсваивание
            delete[] data_;
            size_ = other.size_;
            data_ = new char[size_];
            std::memcpy(data_, other.data_, size_);
        }
        return *this;
    }
    ~Buffer() { delete[] data_; }
};
```

#### Move-семантика (C++11)

Позволяет **забрать** ресурсы у временного объекта, не копируя.

```cpp
class Buffer {
    char* data_ = nullptr;
    size_t size_ = 0;
public:
    Buffer(Buffer&& other) noexcept               // перемещающий конструктор
        : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr;
        other.size_ = 0;
    }
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            delete[] data_;
            data_ = other.data_;
            size_ = other.size_;
            other.data_ = nullptr;
            other.size_ = 0;
        }
        return *this;
    }
};
```

### Правило трёх / пяти / нуля

- **Правило трёх** (до C++11): если нужен свой деструктор, копирующий конструктор или копирующее присваивание — нужны все три.
- **Правило пяти** (C++11+): плюс перемещающие конструктор и присваивание.
- **Правило нуля** (современное): **не пишите ни одного из них вручную.** Используйте `std::string`, `std::vector`, `std::unique_ptr` — они сами управляют ресурсами, и компилятор сгенерирует корректные копирование/перемещение.


```cpp
// Плохо: ручное управление
class Bad { char* data_; ~Bad() { delete[] data_; } /* ... */ };
// Хорошо: правило нуля
class Good {
    std::vector<char> data_;   // сам управляет памятью
    // ни деструктора, ни копирования, ни перемещения писать не нужно
};
```

#### Дружественные функции и классы

`friend` даёт доступ к `private`-полям извне.

```cpp
class Point {
    double x_, y_;
    friend std::ostream& operator<<(std::ostream&, const Point&);
public:
    Point(double x, double y) : x_(x), y_(y) {}
};
std::ostream& operator<<(std::ostream& os, const Point& p) {
    return os << "(" << p.x_ << ", " << p.y_ << ")";   // доступ к private
}
```

**Когда использовать:** только для операторов ввода-вывода или тесно связанных классов. В остальных случаях — геттеры/сеттеры.

#### Перегрузка операторов

Позволяет работать с объектами как со встроенными типами.

```cpp
class Vec2 {
public:
    double x, y;
    Vec2(double x, double y) : x(x), y(y) {}
    Vec2 operator+(const Vec2& other) const {    // a + b
        return Vec2(x + other.x, y + other.y);
    }
    Vec2& operator+=(const Vec2& other) {         // a += b
        x += other.x; y += other.y;
        return *this;
    }
    bool operator==(const Vec2& other) const {    // a == b
        return x == other.x && y == other.y;
    }
    double& operator[](size_t i) {                 // a[i]
        return i == 0 ? x : y;
    }
};

Vec2 a(1, 2), b(3, 4);
Vec2 c = a + b;
c += a;
```

**Что перегружать:** `==`, `!=`, `+`, `-`, `<<`, `[]`, `()`, `=`, `+=`.  
**Что не перегружать:** `&&`, `||`, `,`, `&` — почти всегда плохая идея.

Для сравнения в C++20 удобно:

```cpp
class Point {
    double x, y;
public:
    auto operator<=>(const Point&) const = default;   // генерирует все сравнения
};
```