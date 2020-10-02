# Рефакторинг Threads. Вынесем очередь в отдельный класс.
Разрабатывая класс Treads мы включили очередь заданий в сам класс, как составную часть. Это делалось, чтобы максимально инкапсулировать это поведение внутрь нашего базового класса, ведь "наша" очередь специфическая - она содержит функции и есть поведение, специфическое именно для очереди функций. Забегая вперед скажу, что класс Threads - это базовый класс для создания класса PGPool-Treads, который будет создавать пул соединений и выполнять многопотоковую обработку очереди заданий (выполнять запросы к сервверу PostgreSQL).  
Кроме того, мы планируем создать класс для асинхронной обработки очереди заданий, там также будет использоваться очередь. К тому же, очередь заданий расширится. Мы добавим туда функции обработки результатов, но это позже.  
Исходя из вышеизложенного, возникла необходимость выделить очередь из класса Threads.  

Создаем новую папку queue. ВНИМАНИЕ! НАстоятельно рекомендуем создавать папку нового модуля на одном уровне с другими модклями. Т.е. папка queue и папка threads будут находиться в одной папке, содержащей наши проекты. В этой папке создаем файл queue.hpp и переносим в этот класс методы работы с очередью. 
queue.hpp
```c++
#ifndef QUEUE_H
#define QUEUE_H
#include <mutex>
#include <vector>
#include <functional>


/* --------------------------------------------------- */
/*                       HEADER                        */
/* --------------------------------------------------- */
template <typename Queue_Fn>
class TQueue {
public:
    typedef std::function<void(Queue_Fn)> Predicator;

private:
    std::mutex queue_mutex;
    std::vector<Queue_Fn> queue;

public:
    TQueue();
    TQueue(std::vector<Queue_Fn> _queue);
    TQueue(int size, Queue_Fn task);
    void add_queue(std::vector<Queue_Fn> _queue_async);
    unsigned int queue_size();
    Queue_Fn get_task(bool* queue_end);
    void for_each(Predicator predicator);
    void push(Queue_Fn  fn);
};

/* --------------------------------------------------- */
/*                   IMPLEMENTATION                    */
/* --------------------------------------------------- */

template <typename Queue_Fn>
TQueue<Queue_Fn>::TQueue()
    : queue(std::vector<Queue_Fn>()) {}

template <typename Queue_Fn>
TQueue<Queue_Fn>::TQueue(std::vector<Queue_Fn> _queue)
    : queue(_queue) {}

template <typename Queue_Fn>
TQueue<Queue_Fn>::TQueue(int size, Queue_Fn task)
    : queue(std::vector<Queue_Fn>(size, task)) {}    

template <typename Queue_Fn>
void TQueue<Queue_Fn>::add_queue(std::vector<Queue_Fn> _queue)
{
    std::lock_guard<std::mutex> guard_mutex(queue_mutex);
    queue.insert(queue.end(), _queue.begin(), _queue.end());
}

template <typename Queue_Fn>
unsigned int TQueue<Queue_Fn>::queue_size()
{
    std::lock_guard<std::mutex> guard_mutex(queue_mutex);
    return queue.size();
}

template <typename Queue_Fn>
Queue_Fn TQueue<Queue_Fn>::get_task(bool *queue_end) {
    Queue_Fn task = nullptr;
    {
        std::lock_guard<std::mutex> guard_mutex(queue_mutex);
        if (queue.size() != 0) {
            task = queue.front();
            queue.erase(queue.begin());
        } else {
            *queue_end = true;
        }
    }
    return task;
}

template <typename Queue_Fn>
void TQueue<Queue_Fn>::for_each(Predicator predicator){
    std::vector<Queue_Fn>::iterator iter = queue.begin();
    while(iter!=queue.end()) 
    {
        if(predicator)
            predicator(*iter);
        ++iter;
    }
}

template <typename Queue_Fn>
void TQueue<Queue_Fn>::push(Queue_Fn fn){
    queue.push_back(fn);
}

#endif //QUEUE_H
```
Описание большинства свойств и методов уже приведено в описании класса Threads.  
Здесь отметим следующее: Шаблону очереди передается тип функции ```std::function<...>```.  
Мы добавили три конструктора: перввый создает пустую очередь, второй создает очередь на основе вектора, типа ```std::vector<Queue_Fn>```, третий конструктор создает очередь заданного размера одинаковых задач.  
Кроме того, мы добавили тип ```Predicator``` (см. описание ниже).
```c++
public:
    typedef std::function<void(Queue_Fn)> Predicator;
```
Еще, добавили два метода ```for_each()```, ```push()```.  
Метод ```for_each()``` перебирает вектор элементов и передает функции-предикату каждый элемент очереди. Функция-предикат имеет тип ```Predicator```.  Вектор элементов перебираем [используя итераторы](https://metanit.com/cpp/tutorial/7.3.php).  
Метод ```push()``` добавляет элемент в конец очереди.
  
  
Терерь откроем папку threads, файл threads.hpp и внесем изменения.  
Удалим свойства:
```c++
    std::mutex queue_mutex;
    std::vector<Queue_Fn> queue;
```
Удалим методы:
```c++
    Queue_Fn get_task(bool* queue_end);
    void add_queue(std::vector<Queue_Fn> _queue_async);
    unsigned int queue_size();
```
  
Включим заголовок очереди ```TQueue```:
```c++
#include "../queue/queue.hpp"
```
Изменим метод ```run()```, теперь мы будем передавать в этот метод очередь ```TQueue```:
```c++
...
template <typename R, typename ...Args>
class Threads {
    ...

    void run(std::shared_ptr<TQueue<Queue_Fn>> queue, Args... args);

    ...
}

template <typename R, typename ...Args>
void Threads<R, Args...>::run(std::shared_ptr<TQueue<Queue_Fn>> queue, Args... args)
{
    ...

    while(iter != threads.end()){
        *iter = std::thread(&Threads::wrapper_loop,
            std::ref(*this), queue, i++, args...);
        ++iter;
    }

    ...
}
```
Здесь в метод ```run()``` передаем умный указатель ```std::shared_ptr``` на очередь ```TQueue<Queue_Fn>```. В теле метода ```run()``` изменится вызов потока. Потоку также передается метод ```wrapper_loop()```, но еще с одним аргументом ```queue```.
  
Тот же самый аргумент добавляем в методы ```wrapper_loop()``` и ```loop()```. 
```c++
...

template <typename R, typename ...Args>
class Threads {

...

protected:
    virtual void wrapper_loop(std::shared_ptr<TQueue<Queue_Fn>> queue, int num_thread, Args... args){
        loop(queue, nullptr, args...);
    }
    void loop(std::shared_ptr<TQueue<Queue_Fn>> queue, Loop_Fn _fn, Args... args);
};

...

template <typename R, typename ...Args>
void Threads<R, Args...>::loop(std::shared_ptr<TQueue<Queue_Fn>> queue, Loop_Fn _fn, Args... args)
{
    ...
    while (!queue_end) {
        Queue_Fn task = queue->get_task(&queue_end);

    ...
}

```
В теле метода ```loop()``` изменяем вызов ```get_task()```, теперь он будет выглядеть так ```queue->get_task(&queue_end)```.
  

Осталось поправить тесты.  
Строку
```c++
    std::vector<Threads<void>::Queue_Fn> queue (4, count_task);
```
заменим на строки
```c++
    typedef std::function<void(void)> Void_Fn;
    std::shared_ptr<TQueue<Void_Fn>> queue = std::make_shared<TQueue<Void_Fn>>(4, count_task);
```
Т.е. здесь создаем уже не вертор, а умный указатель на очередь ```TQueue``` типа ```std::function<void(void)>```. Используем перегруженный конструктор номер три.  
  
Тест кейс будет следующим:
```c++
    TEST_CASE("Количество вызовов и размер очереди") {
        CHECK(queue->queue_size() == 4);

        threads.run(queue);
        CHECK(count == 4);
        CHECK(queue->queue_size() == 0);
    }
```
  
В классе наследнике ```Test_Treads``` исправим метод ```wrapper_loop()``` по аналогии с базовым классом.
```c++
    class Test_Threads: public Threads<int, int> {
    public:
        typedef Threads<int, int>::Queue_Fn Test_Fn;

        ...

        void wrapper_loop(std::shared_ptr<Queue<Test_Fn>> queue, int num_thread, int arg) override{
            Threads<int, int>::loop(queue, [num_thread](Queue_Fn task) {

                ...

            }, num_thread);
        }
    };
```
  
Строку
```c++
    std::vector<Test_Threads::Test_Fn> queue2(num_tasks, counts_task);
```
заменим на строку
```c++  
    std::shared_ptr<TQueue<Test_Threads::Test_Fn>> queue2 = std::make_shared<TQueue<Test_Threads::Test_Fn>>(num_tasks, counts_task);
```
Т.е. здесь создаем уже не вертор, а умный указатель на очередь ```TQueue``` типа ```std::function<void(void)>```. Используем перегруженный конструктор номер три.  
  
Тест кейс будет следующим:
```c++
    TEST_CASE("Равномерность выполнения потоков и размер очереди") {
        CHECK(queue2->queue_size() == num_tasks);

        test_threads.run(queue2, num_tasks);
        for (int i = 0; i < counts.size(); ++i){
            CHECK(counts[i] > 0);
        }
        CHECK(queue2->queue_size() == 0);
    }
```

