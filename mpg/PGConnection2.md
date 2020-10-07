---
layout: default
title: Часть 2. Используем многопоточность
parent: Разработка mpg
nav_order: 2
---
# Часть 2. Используем многопоточность
{: .no_toc }  
В [части 1](./PGConnection.md) мы разработали простое приложение запросов к базе данных PostgreSQL.  
В методе ```exec()``` мы использовали блокирующую функцию ```PQexec()```. Эта функция приостанавливает работу программы до окончания ее выполнения. А если у нас будет тысяча запросов или миллион. Многие существующие средства предлагают выход - использовать многопоточность.  
Для продолжения чтения лучше ознакомиться с Частями [1](../threads/Threads.md) и [2](../threads/Threads2.md) раздела Управление потоками.  
Рабочий проект этого раздела можно найти [здесь](https://github.com/profistarter/mpg/tree/05c23431a13393e8a8eeeaee1aa2d994663a9baa) (ссылка на коммит).
## Содержание
{: .no_toc }  
1. TOC
{:toc}

# Создаем класс PGPool_Threads - пул соединений к PostgreSQL
Создадим специальный класс ```PGPool_Threads```, который будет создавать потоки и передавать им соединения ```PGConnection```. Для передачи запросов соединениям будем использовать очередь запросов (заданий). 
  
В папке нашего проекта ```mpg``` создадим файл ```pg_pool_threads.hpp```.
Мы будем использовать класс ```Threads```, разработанный [здесь.](../threads/Threads.md). Этот класс находится в заголовочном файле в папке ```threads```, расположенной на одном уровне с нашим проектом, поэтому путь к заголовочному файлу следующий:  
```c++
#include "../threads/threads.hpp"
```  
Также будем использовать класс ```PGConnection```. Путь к заголовочному файлу следующий:
```c++
#include "pg_connection.h"
```  
Так как мы будем использовать шаблоны, то и описывать класс будем в заголовочном файле: ```pg_pool_threads.hpp```. Дело в том, что нам не удалось разделить шаблон класса на два файла .h и .cpp.  Создаем файл .hpp в папке нашего проекта.   
Наш класс будет наследником класса ```Threads```:  
pg_pool_threads.hpp
```c++
#ifndef PG_POOL_THREADS_H
#define PG_POOL_THREADS_H
#include <mutex>
#include <thread>
#include <vector>
#include "../threads/threads.hpp"
#include "pg_connection.h"

template <typename ...Args>
class PGPool_Threads: public Threads<std::string, Args...>
{
private:
    std::vector<std::shared_ptr<PGConnection>> conns;

public:
    typedef std::function<std::string(Args...)> Queue_Fn; 

    PGPool_Threads(int num_threads = std::thread::hardware_concurrency())
        : conns(std::vector<std::shared_ptr<PGConnection>>(num_threads))
        , Threads<std::string, Args...>(num_threads)
    {
        for (int i = 0; i < num_threads; i++) {
            conns[i] = std::make_shared<PGConnection>();
        }
    }

    PGPool_Threads(std::vector<std::shared_ptr<PGConnection>> _conns)
        : conns(_conns)
        , Threads<std::string, Args...>(_conns.size())
    {}

    void wrapper_loop(std::shared_ptr<TQueue<Queue_Fn>> queue, int num_thread, Args... args) override{
        std::shared_ptr<PGConnection> conn = conns[num_thread];

        Threads<std::string, Args...>::loop(queue, [conn, num_thread, args...](Queue_Fn task) {
            // std::cout << std::string(num_thread, '\t') << num_thread << std::endl;
            conn->exec(task(args...).c_str());
        }, args...);
    }
};

#endif // PG_POOL_THREADS_H
```
Мы уже создавали класс ```Test_Threads``` наследник класса ```Threads```  когда писали тесты к классу ```Threads```.  
Класс ```PGPool_Threads```, также будет шаблоном. Если шаблон класса ```Threads``` принимал два параметра, то шаблон этого класса будет принимать один параметр ```...Args```. Параметр ```R``` шаблона класса ```Threads``` мы зададим сразу равным ```std::string```.
```c++
...

template <typename ...Args>
class PGPool_Threads: public Threads<std::string, Args...>

...
```
Далее мы создаем закрытое свойство ```conns```, которое является вектором умных указателей на объекты класса  ```PGConnection```. Т.е. это свойство - набор подключений к PostgreSQL.  
Также определим тип ```Queue_Fn```. Это тип функции ```std::string(Args...)```, которая возвращает строку, а принимает в качестве аргументов ```Args...```.
```c++
...

private:
    std::vector<std::shared_ptr<PGConnection>> conns;

public:
    typedef std::function<std::string(Args...)> Queue_Fn; 

...
```
Конструктор инициализирует свойство ```conns``` пустым вектором указателей на объекты класса ```PGConnection```. А также вызывает конструктор базового класса. Обращаем внимание на вызов конструктора ```Threads<std::string, Args...>(num_threads)```. Так как класс ```Threads``` шаблон мы передаем ему параметры. Первый параметр определим сразу, второй параметр компилятор определит, когда мы будем создавать объект этого класса ```PGPool_Threads```. В теле конструктора мы создаем объекты класса ```PGConnection``` и помещаем умные указатели на них в вектор ```conns```.    
```c++
    PGPool_Threads(int num_threads = std::thread::hardware_concurrency())
        : conns(std::vector<std::shared_ptr<PGConnection>>(num_threads))
        , Threads<std::string, Args...>(num_threads)
    {
        for (int i = 0; i < num_threads; i++) {
            conns[i] = std::make_shared<PGConnection>();
        }
    }
```
Далее, мы создали еще один перегруженный конструктор. Это понадобилось нам, чтобы запустить тесты, но может понадобиться и в реальном приложении. В тестах, чтобы не создавать подключение к базе данных PostgreSQL мы будем использовать фейковые объекты ```conn```. Здесь, передаем конструктору вектор умных указателей на соединение с PostgreSQL. Инициализируем свойство ```conns``` и вызвываем конструктор базового класса ```Threads```, которому передаем длину вектора ```_conns```. Т.е. количество потоков будет равно количеству подключений к PostgreSQL.
```c++
    PGPool_Threads(std::vector<std::shared_ptr<PGConnection>> _conns)
        : conns(_conns)
        , Threads<std::string, Args...>(_conns.size())
    {}
```

Также переписываем метод-обертку ```wrapper_loop()```. Внутри этого методе также как в базовом классе вызываем метод ```loop()``` и передаем ему очередь, лямбда-функцию (в классе предке ```Threads```вмето лямбда-функции мы передавали ```nullptr```) и список аргументов задачи ```Queue_Fn```.  
Мы переписали этот метод чтобы добавить лямбда-функцию, которая берет соединение ```conn```, номер которого соответствует номеру процесса. И у этого соединения вызываем метод ```exec()```.  
Методу ```exec()``` передаем строку - результат работы задачи, полученной из очереди задач.  
Собственно, лямбда-функция - цель создания этого класса наследника. Основная задача этого класса выполнять запросы к серверу PostgreSQL. Текст запроса - это значение, которое возвращает задача из очереди задач.
```c++
    void wrapper_loop(std::shared_ptr<TQueue<Queue_Fn>> queue, int num_thread, Args... args) override{
        std::shared_ptr<PGConnection> conn = conns[num_thread];

        Threads<std::string, Args...>::loop(queue, [conn, num_thread, args...](Queue_Fn task) {
            // std::cout << std::string(num_thread, '\t') << num_thread << std::endl;
            conn->exec(task(args...).c_str());
        }, args...);
    }
```
# Тестируем класс PGPool_Threads
Для тестирования используем библиотеки doctest и fakeit. Если они еще не подключена смотрите [здесь](../life_hack/add_doctest.md).  
Мы тестируем выполнение метода ```run()``` класса ```PGPool_Threads```. Нам нужно проверить очередь после выполнения ```run()``` - она должна стать пустой. Также нам нужно проверить количество вызовов заданий ```test_task()``` и количество вызовов метода ```exec()``` соединения ```PGConnection```. Возьмем три соединения и 10 заданий.  
Еще, в тестах будем использовать ```std::chrono_literals```, поэтому подключаем еще библиотеку:  
```c++
#include "fakeit/fakeit.hpp"
#include "doctest/doctest.h"
#include <chrono>
```
В файле pg_pool_threads.hpp пишем тесты.  
pg_pool_threads.hpp  
```c++
/* --------------------------------------------------- */
/*                       TESTS                         */
/* --------------------------------------------------- */

TEST_SUITE("Тест класса PGPool_Threads")
{
    namespace test_pg_pool_threads {
        int num_conns = 3; // Количество соединений.
        int num_tasks = 10; // Количество заданий.
        //----Создаем счетчик вызовов функции-задания и саму функцию-задание
        int static count = 0;
        std::mutex static count_mutex;

        std::string test_task() {
            using namespace std::chrono_literals;
            std::this_thread::sleep_for(10ms);
            std::lock_guard<std::mutex> guard_mutex(count_mutex);
            count += 1;
            return "";
        }

        TEST_CASE("Количество вызовов и размер очереди") {
            //----Создаем фейковые соединения и помещаем их в вектор
            std::vector<std::shared_ptr<PGConnection>> conns(num_conns);
            fakeit::Mock<PGConnection> conn1;
            fakeit::Fake(Method(conn1, exec));
            //https://github.com/eranpeer/FakeIt/issues/60
            conns[0] = std::shared_ptr<PGConnection>(&conn1.get(), [](...) {});
            
            fakeit::Mock<PGConnection> conn2;
            fakeit::Fake(Method(conn2, exec));
            conns[1] = std::shared_ptr<PGConnection>(&conn2.get(), [](...) {});

            fakeit::Mock<PGConnection> conn3;
            fakeit::Fake(Method(conn3, exec));
            conns[2] = std::shared_ptr<PGConnection>(&conn3.get(), [](...) {});

            //----Создаем объект класса PGPool_Threads и очередь TQueue
            PGPool_Threads pg_pool_threads(conns);
            typedef std::function<std::string(void)> String_Fn;
            std::shared_ptr<TQueue<String_Fn>> queue = std::make_shared<TQueue<String_Fn>>(num_tasks, test_task);


            CHECK(queue->queue_size() == num_tasks);
            pg_pool_threads.run(queue);
            CHECK(count == num_tasks);
            CHECK(queue->queue_size() == 0);
            fakeit::Verify(Method(conn1,exec)).AtLeast(1);
            fakeit::Verify(Method(conn2,exec)).AtLeast(1);
            fakeit::Verify(Method(conn3,exec)).AtLeast(1);
        }

    }
}
``` 
Мы используем ```TEST_SUITE```, а внутри него ```TEST_CASE```.  
```doctest``` спроектирован так, что внутри ```TEST_SUITE``` невозможно запустить функцию, в том числе конструктор, а также цикл. Т.е. поведение такое, как вне функции ```main()```. Поэтому создание любых объектов необходимо помещать в ```TEST_CASE```. С другой стороны в ```TEST_CASE``` невозможно задать функцию. Поэтому определение любой функции помещаем снаружи ```TEST_CASE```. В документации к ```doctest``` эти особенности не указаны.  
  
Внутри ```TEST_SUITE``` указываем пространство имен ```namespace test_pg_pool_threads```, так как будут встречаться имена переменных, используемых в других тестах. И вообще, полезная практика использования пространства имен в тестах.  
Внутри ```TEST_SUITE``` создаем счетчик вызовов функции-задания и саму функцию-задание. 
```c++
    std::string test_task() {
        using namespace std::chrono_literals;
        std::this_thread::sleep_for(10ms);
        std::lock_guard<std::mutex> guard_mutex(count_mutex);
        count += 1;
        return "";
    }

```   
Здесь останавливаем поток на 10 миллисекунд для того чтобы один поток не успел выполнить все задания.  С помощью ```count_mutex``` защищаем ```count``` (о мьютексах см. [здесь](http://scrutator.me/post/2012/04/04/parallel-world-p1.aspx)).
  
Внутри ```TEST_CASE``` создаем объекты и выполняем тесты.  
Так как класс ```PGPool_Thrads``` использует соединения к PostgreSQL, то нам нужно позаботится о создании этих соединений в тесте. Мы же не будем создавать реальные соединения в тестах. Используем библиотеку ```fakeit``` (документация [здесь](https://github.com/eranpeer/FakeIt/wiki/Quickstart)). Создаем три фейковых объекта ```PGConnection``` и имитируем в них метод ```exec()``` класса ```PGConnection```.  
Ниже показан пример создания одного фейкового объекта.  
```c++
        fakeit::Mock<PGConnection> conn1;
        fakeit::Fake(Method(conn1, exec));
        conns[0] = std::shared_ptr<PGConnection>(&conn1.get(), [](...) {});
```
Последняя строка кода помещает умный указатель на фейковое соединение в вектор ```conns``` (есть особенность использования умных указателей в библиотеке ```fakeit``` - см. [здесь](https://github.com/eranpeer/FakeIt/issues/60)). Здесь мы не используем цикл только потому, что метод ```exec()``` фейкового объекта при выходе из области определения (тела цикла) странным образом обнуляется. Поэтому, мы должны учитывать веремя жизни этого фейкового метода ```exec()``` при написании кода.
  
В последнюю очередь создаем объект класса ```PGPool_Threads``` и очередь ```TQueue```. При создании объекта ```PGPool_Threads``` в конструктор передаем вектор фейковых соединений ```conns```. Для этого нам пришлось создать еще один конструктор класса ```PGPool_Threads```, потому что при первоначальном проектировании этого класса соединения к PostgreSQL создавались внутри конструктора и были скрыты снаружи. Это, вообще-то, хорошая идея, но для тестов неподходящая.  
```c++
        PGPool_Threads pg_pool_threads(conns);
        typedef std::function<std::string(void)> String_Fn;
        std::shared_ptr<TQueue<String_Fn>> queue = std::make_shared<TQueue<String_Fn>>(num_tasks, test_task);
```
В конце, собственно, сами тесты.
```c++
        CHECK(queue->queue_size() == num_tasks);
        pg_pool_threads.run(queue);
        CHECK(count == num_tasks);
        CHECK(queue->queue_size() == 0);
        fakeit::Verify(Method(conn1,exec)).AtLeast(1);
        fakeit::Verify(Method(conn2,exec)).AtLeast(1);
        fakeit::Verify(Method(conn3,exec)).AtLeast(1);
```
Проверяем размер очереди перед запуском ```run()```.  
Запускаем ```run()```.  
Поверяем количество вызовов ```test_task()```.  
Проверяем размер очереди.  
Проверяем вызов метода ```exec()``` у фейковых объектов. Метод должен быть вызван хотя-бы один раз. Точнее проверить не удастся, так как работа потоков непредсказуема.  
  
Еще один нюанс: библиотека фейковых объектов ```fakeit``` под капотом создает наследника класса, объекты которого делаем фейковыми. Для успешного создания фейкового метода, он (метод) должен быть виртуальным. Откроем файл ```pg_connection.h``` укажем, что метод ```exec()``` является виртуальным.  
```c++
class PGConnection {
    ...
    
    virtual int exec(const char *query);
};
```
  
Для компиляции и запуска тестов используем файл ```test.cpp```, а также добавьте еще одну задачу в ```tasks.json``` и конфигурацию в ```launch.json``` (см. [здесь](../life_hack/add_doctest.md)).

# Проверяем скорость - изменяем main()
В предыдущих пунктых мы создали класс ```PGPool_Threads``` в файле ```pg_pool_threads.hpp```.  
Теперь изменим файл самой программы ```app.cpp```. Напишем код с нуля т.е. будем шаг за шагом добавлять код в файл ```app.cpp``` с пояснениями.
```c++
#define DOCTEST_CONFIG_DISABLE
#include "doctest/doctest.h"
```
Эти две строчки кода дают команду не включать код тестов в компилируемую программу. Для этого есть файл ```test.cpp``` (см. раздел [Подключение библиотеки тестирования](../life_hack/preparation.md)) .
```c++
#include <windows.h>
#include <vector>
#include <functional>
#include <iostream>
#include "pg_pool_threads.hpp"
#include "../utils/queue.hpp"
```
Эти заголовки подключают дополнительные библиотеки, в том числе те, которые мы написали сами. Файл ```queue.hpp``` лежит в папке, которая находится на одном уровне с папкой нашего проекта (см. [здесь](../threads/Threads2.md)).  
```c++
//https://ravesli.com/urok-129-tajming-koda-vremya-vypolneniya-programmy/
class Timer {
private:
    using clock_t = std::chrono::high_resolution_clock;
    using second_t = std::chrono::duration<double, std::ratio<1>>;

    std::chrono::time_point<clock_t> m_beg;

public:
    Timer() 
        : m_beg(clock_t::now()) {}

    void reset() {
        m_beg = clock_t::now();
    }

    double elapsed() const {
        return std::chrono::duration_cast<second_t>(clock_t::now() - m_beg).count();
    }
};
```
Это класс ```Timer```, который необходим для вычисления времени выполнения заданий в очереди (см. далее). Реализация этого класса взята [отсюда](https://ravesli.com/urok-129-tajming-koda-vremya-vypolneniya-programmy/).  
```c++
enum class Weight {
    little = 1,
    medium = 1000,
    large = 2000
};
```
```Weight``` - это перечисление. Мы ввели его, чтобы проверить быстродействие на табляцах SQL разного веса (weight). Номер определяет количество десятков сток в таблице.
```c++
std::string str_weght(const Weight &weight) {
    std::string weight_str;
    switch (weight) {
        case Weight::little: weight_str.append("small");
            break;  
        case Weight::medium: weight_str.append("medium");
            break;
        case Weight::large: weight_str.append("large");
            break;            
    }
    return weight_str;
}
```
Функция ```str_weght()```  возвращает строку соответствующую весу таблицы. Например, выбрав вес ```Weight::little``` мы будем работать с таблицей ```small_table```.
```c++
std::string myFunc(const Weight &weight) {
    std::string str = str_weght(weight);
    return "SELECT SUM(a_field) + SUM(b_field) FROM " + str + "_table";
}
```
Функция ```myFunc()``` - это, собственно, наша задача, котороую поместим в очередь заданий. Мы передаем функции вес таблицы. Наша задача возвращает строку SQL запроса. Запрос суммирует все значения столбцов ```a_field``` и ```b_field``` из таблицы соответсвующей весу: ```small_table``` или ```medium_table``` или ```large_table```.  
```c++
typedef std::function<std::string(Weight)> String_Fn;
std::shared_ptr<TQueue<String_Fn>> get_queue(const int &num){
    std::shared_ptr<TQueue<String_Fn>> queue = std::make_shared<TQueue<String_Fn>>();
    int i = 0;
    while (i < num) {
        queue->push(myFunc);
        ++i;
    }
    return queue;
}
```
Функция ```get_queue()``` создает очередь ```TQueue```. Мы передаем функции количество заданий и функция в цикле помещает задания ```myFunc``` в очередь. Возвращаем умный указатель на очередь.  
Перед реализацией функции мы определили новый тип, описывающий тип заданий ```std::string(Weight)```, т.е. тип функции ```myFunc```.
```c++
void prepare(const Weight &weight){
    std::string weight_str = str_weght(weight);
    PGConnection conn;
    conn.exec(std::string("DROP TABLE IF EXISTS " + weight_str + "_table").c_str());
    conn.exec(std::string("CREATE TABLE IF NOT EXISTS " + weight_str + "_table (       \
                    a_field bigint,  \
                    b_field bigint   \
                )").c_str());
    for (int i = 0; i < static_cast<int>(weight); i++) {
        conn.exec(std::string("INSERT INTO " + weight_str + "_table (a_field, b_field) VALUES   \
                    (1000000, 1000000),    \
                    (2000000, 2000000),    \
                    (3000000, 3000000),    \
                    (1000000, 1000000),    \
                    (2000000, 2000000),    \
                    (3000000, 3000000),    \
                    (1000000, 1000000),    \
                    (2000000, 2000000),    \
                    (3000000, 3000000),    \
                    (1000000, 1000000);    \
                ").c_str());
    }
}
```
Функция ```prepare()``` подготавливает соответствующую таблицу в зависимости от передаваемого в нее веса (```weight```).  
Здесь мы создаем объект класса ```PGConnection conn``` .
Первый вызов метода ```conn.exec()```  удаляет соответствующую таблицу если таковая есть. Это необходимо для чистоты эксперимента - на случай если мы изменим эту таблицу в результате тестов.  
Второй вызов метода ```conn.exec()``` создаем эту таблицу заново.  
Третий вызов метода ```conn.exec()``` заполняем эту таблицу необходимыми для теста данными, в зависимости от веса таблицы. Функция ```static_cast<int>(weight)``` возвращает целочисленное значение элемента перечисления, которое и определяет количество десятков строк в таблице.
```c++
void sync(const int &num_query, const Weight &weight){
    PGConnection conn;
    std::shared_ptr<TQueue<String_Fn>> queue = get_queue(num_query);
    Timer t1;
    queue->for_each([&conn, weight](String_Fn task) {
        conn.exec(task(weight).c_str());
    });
    std::cout << " " << t1.elapsed() << " \t |";
}
```
Функция ```sync()``` запускает последовательное выполнение заданий из очереди.  
Здесь мы создаем объект класса ```PGConnection conn``` . Этот объект с помощью метода ```exec()``` будет выполнять запросы к PostgreSQL. Создаем очередь с помощью функции ```get_queue()```. Для перебора всех заданий в очереди используется метод класса ```TQueue::for_each()```. Этому методу передается функция-предикатор, которая выполняет задание, получает строку запроса и передает эту стоку запроса в метод ```conn.exec()```. Как видно, заданию ```task()``` передается вес (```weight```), чтобы получить запрос с именем соответствующей таблицы.  
Внутри ```TQueue::for_each()``` перебирются все элементы очереди, и каждый элемент (задание) передается функции предикатору.  
Последняя строка функции ```sync()``` формирует строку, где выводится время, затраченное на выполнение всех заданий из очереди.
```c++
void treads(const int &num_query, const Weight &weight, const int &num_threads){
    std::shared_ptr<TQueue<String_Fn>> queue = get_queue(num_query);
    PGPool_Threads<Weight> pool_threads{num_threads};
    Timer t2;
    pool_threads.run(queue, weight);
    std::cout << " " << t2.elapsed() << " \t |";
}
```
Функция ```threads()``` запускает параллельное выполнение заданий из очереди.  
Здесь мы  создаем очередь с помощью функции ```get_queue()```.  
Создаем объект класса ```PGPool_Threads```. Заметьте, так как ```PGPool_Threads``` является шаблоном класса, то при создании объекта мы передаем параметры шаблона (см. радел PGPool_Threads выше), в данном случае ```PGPool_Threads<Weight>```. Анализируя эту строку кода компилятор создает конкретный класс с учетом этого параметра.  
Мы передаем в конструктор класса ```PGPool_Threads<Weight>``` количество потоков ```num_threads```, которое получает функция ```threads()```. Внутри конструктора (скрыто от нас) создаются соединения к PostgreSQL - объекты класса ```PGConnection```.  
Потом с помощью метода ```PGPool_Threads<Weight>::run()``` мы запускаем выполнение многопоточной обработки заданий очереди. В метод ```run()``` мы передаем очередь (```queue```) и вес таблицы (```weight```).  
```weight``` - это параметр Parameter Pack - ```Args...``` описанный в классе (см. выше):
```c++
//Этот код не надо вставлять в app.cpp
template <typename ...Args>
class PGPool_Threads: public Threads<std::string, Args...>
```
Последняя строка функции ```threads()``` формирует строку, где выводится время, затраченное на выполнение всех заданий из очереди.
```c++
int main()
{
    SetConsoleOutputCP(1251);
    
    //Подготовка
    prepare(Weight::little);
    prepare(Weight::medium);
    prepare(Weight::large);

    const int num_query = 100;

    //==============Последовательно

    std::cout << "| \t\t\t\t | small \t | medium \t | large \t |" << std::endl;
    std::cout << "| sync \t\t\t\t |";
    sync(num_query, Weight::little);
    sync(num_query, Weight::medium);
    sync(num_query, Weight::large);
    std::cout << std::endl;

    ///=============Параллельно
    std::cout << "| -------------------------------------------------------------------------" << std::endl;
    for (int i = 1; i < 6; ++i){
        std::cout << "| parallel " << i << " thread \t\t |";
        treads(num_query, Weight::little, i);
        treads(num_query, Weight::medium, i);
        treads(num_query, Weight::large, i);
        std::cout << std::endl;
    }
}
```
В функции ```main()```:  
- Функция ```SetConsoleOutputCP(1251)``` позволяет правильно отображать кириллицу в консоли. Это нам понадобится для отображеня ошибок подключения к PostgreSQL.  
- Трижды вызываем функцию ```prepare()``` - тем самым формируем три таблицы SQL с соответсвующим весом: ```small_table``` или ```medium_table``` или ```large_table```.  
- Трижды вызываем функцию ```sync()``` для запуска последовательной обработки заданий очереди для каждой таблици с соответствующим весом. Время выполнения каждой функции ```sync()``` запишется в строку. Каждое знаечение в соответствующем столбце:  
```| small   | medium    | large     |```.  
- Выполняем цикл соответствующий количеству потоков. Мы проверяем время выполнения для разного количества потоков от одного до пяти.  
В цикле трижды  вызываем функцию ```threads()``` для запуска параллельной обработки заданий очереди для каждой таблици с соответсвующим весом. Время выполнения каждой функции ```threads()``` запишется в строку. Каждое знаечение в соответсаующем столбце:  
```| small   | medium    | large     |```.

У нас получился следующий результат:
```
|                                | small         | medium        | large         |
| sync                           | 0.0273079     | 0.40432       | 0.732363      |
| -------------------------------------------------------------------------
| parallel 1 thread              | 0.0271957     | 0.3628        | 0.706151      |
| parallel 2 thread              | 0.016277      | 0.237222      | 0.356096      |
| parallel 3 thread              | 0.0114329     | 0.133317      | 0.253607      |
| parallel 4 thread              | 0.0102801     | 0.107307      | 0.203747      |
| parallel 5 thread              | 0.0108458     | 0.117218      | 0.20426       |
```
на компьютере со следующими параметрами:
```
Базовая скорость:       2,10 ГГц
Сокетов:                1
Ядра:                   4
Логических процессоров: 8
```
На этом же компьютере был запущен PostgreSQL.  
  
В целом, результат ожидаемый, просто, интересна зависимость производительности от количества потоков. Использование более четырех потоков на нашем компьютере смысла не имеет. Еще одно нетривиальное наблюдение - производительность растет одинаково для запросов разной длительности (таблиц разного веса). 
  
Более интересный результат мы получим, когда добавим асинхронную обработку заданий в очереди.




