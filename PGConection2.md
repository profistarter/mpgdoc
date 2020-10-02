# Ускоряем используя многопоточность
В методе exec() листинг 1 используется блокирующая функция PQexec(). Эта функция будет приостанавливать работу программы до окончания ее выполнения. А если у нас будет тысяча запросов или миллион. Многие существующие средства предлагают выход - использовать многопоточность.  
Попробуем...  
Создадим класс ```PGPool_Threads```, который в каждом потоке выбирает соответствующее соединение и выполняет запрос к серверу PostgreSQL.
В папке нашего проекта создадим файл ```pg_pool_threads.hpp```. Это заголовочный 
Мы будем использовать класс Threads, разработанный [здесь.](./Threads.md). Этот класс находится в заголовочном файле в папке, расположенной на одном уровне с нашим проектом, поэтому путь к заголовочному файлу следующий:  
```c++
#include "../threads/threads.hpp"
```  
Также будем использовать класс ```PGConnection```. Путь к заголовочному файлу следующий:
```c++
#include "pg_connection.h"
```  
Так как мы будем использовать шаблоны, то и описывать класс будем в заголовочном файле: ```pg_pool_threads.hpp```. Создаем этот файл в папке нашего проекта.   
Наш класс будет наследником класса Threads:
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
Теперь создадим класс ```PGPool_Threads```, который также будет шаблоном. Если шаблон класса ```Threads``` принимал два параметра, то шаблон этого класса будет принимать один параметр ```...Args```. Параметр ```R``` шаблона класса ```Threads``` мы зададим сразу равным ```std::string```.
```c++
...

template <typename ...Args>
class PGPool_Threads: public Threads<std::string, Args...>

...
```
Далее мы создаем закрытое свойство ```conns```, которое является вектором умных указателей на объекты класса  ```PGConnection```. Т.е. это свойство - набор подключений к БД PostgreSQL.  
Также определим тип ```Queue_Fn```. Это тип функции ```std::string(Args...)```, которая возвращает строку, а принимает в качестве аргументов ```Args...```.
```c++
...

private:
    std::vector<std::shared_ptr<PGConnection>> conns;

public:
    typedef std::function<std::string(Args...)> Queue_Fn; 

...
```
Конструктор инициализирует свойство ```conns``` пустым вектором указателей на объекты класса ```PGConnection```. А также вызывает конструктор базового класса. Обращаем внимание на вызов конструктора ```Threads<std::string, Args...>(num_threads)```. Так как класс ```Threads``` шаблон мы передаем ему параметры. Первый параметр определим сразу, второй параметр компилятор определит, когда мы будем создавать объект этого класса ```PGPool_Threads```. В теле конструктора вы создаем объекты класса ```PGConnection``` и помещаем умные указатели на них в вектор ```conns```.
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
Мы создали еще один перегруженный конструктор. Это понадобилось нам, чтобы запустить тесты, но может понадобиться и в реальном приложении. В тестах, чтобы не создавать подключение к базе данных PostgreSQL мы будем использовать фейковые объекты ```conn```. Здесь, передаем конструктору вектор умных указателей на соединение с PostgreSQL. Инициализируем свойство ```conns``` и вызвываем конструктор базового класса ```Threads```, которому передаем длину вектора ```_conns```. Т.е. количество потоков будет равно количеству подключений к PostgreSQL.
```c++
    PGPool_Threads(std::vector<std::shared_ptr<PGConnection>> _conns)
        : conns(_conns)
        , Threads<std::string, Args...>(_conns.size())
    {}
```

Также переписываем метод-обертку ```wrapper_loop()```. Внутри этого методе также как в базовом классе вызываем метод ```loop()``` и передаем ему очередь, лямбда-функцию (в классе предке ```Threads```вмето лямбда-функции мы передавали ```nullptr```) и список аргументов задачи ```Queue_Fn```.  
Мы переписали этот метод чтобы добавить лямбда-функцию, которая берет соединение ```conn```, номер которого соответсвует номеру процесса. И у этого соединения вызываем метод ```exec()```.  
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
Мы тестируем выполнение метода ```run()``` класса ```PGPool_Threads```. Нам нужно проверить очередь после выполнения ```run()``` - онв должнв стать пустой. Также нам нужно проверить количество вызовов заданий ```test_task()``` и количество вызовов метода ```exec()``` соединения ```PGConnection```. Возьмем три соединения и 10 заданий.
Для тестирования используем библиотеки doctest и fakeit. Как их подключить смотрите [здесь](./life_hack/add_doctest.md).  
Еще, в тестах будем использовать ```std::chrono_literals```, поэтому подключаем библиотеку:  
```c++
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
Здесь останавливаем поток на 10 миллисекунд для того чтобы один поток не успел выполнить все задания.  С помощью ```count_mutex``` защищаем ```count``` (см. [здесь](http://scrutator.me/post/2012/04/04/parallel-world-p1.aspx)).
  
Внутри ```TEST_CASE``` создаем объекты и выполняем тесты.  
Так как класс ```PGPool_Thrads``` использует соединения к PostgreSQL, то нам нужно позаботится о создании этих соединений в тесте. Мы же не будем создавать реальные соединения в тестах. Используем библиотеку ```fakeit``` (документация [здесь](https://github.com/eranpeer/FakeIt/wiki/Quickstart#invocation-matching)). Создаем три фейковых объекта ```PGConnection``` и имитируем в них метод ```exec()``` класса ```PGConnection```.
```c++
        fakeit::Mock<PGConnection> conn1;
        fakeit::Fake(Method(conn1, exec));
        conns[0] = std::shared_ptr<PGConnection>(&conn1.get(), [](...) {});
```
Последняя строка кода помещает умный указатель на фейковое соединение в вектор ```conns``` (есть особенность использования умных указателей в библиотеке ```fakeit``` - см. [здесь](https://github.com/eranpeer/FakeIt/issues/60)). Здесь мы не используем цикл только потому, что метод ```exec()``` фейкового объекта при выходе из области определения (тела цикла) странным образом обнуляется. Поэтому, мы должны учитвать веремя жизни этого фейкового метода ```exec()``` при написании кода.
  
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