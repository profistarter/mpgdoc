# ����������� Threads. ������� ������� � ��������� �����.
������������ ����� Treads �� �������� ������� ������� � ��� �����, ��� ��������� �����. ��� ��������, ����� ����������� ��������������� ��� ��������� ������ ������ �������� ������, ���� "����" ������� ������������� - ��� �������� ������� � ���� ���������, ������������� ������ ��� ������� �������. ������� ������ �����, ��� ����� Threads - ��� ������� ����� ��� �������� ������ PGPool-Treads, ������� ����� ��������� ��� ���������� � ��������� �������������� ��������� ������� ������� (��������� ������� � �������� PostgreSQL).  
����� ����, �� ��������� ������� ����� ��� ����������� ��������� ������� �������, ��� ����� ����� �������������� �������. � ���� ��, ������� ������� ����������. �� ������� ���� ������� ��������� �����������, �� ��� �����.  
������ �� ���������������, �������� ������������� �������� ������� �� ������ Threads.  

������� ����� ����� queue. ��������! ������������ ����������� ��������� ����� ������ ������ �� ����� ������ � ������� ��������. �.�. ����� queue � ����� threads ����� ���������� � ����� �����, ���������� ���� �������. � ���� ����� ������� ���� queue.hpp � ��������� � ���� ����� ������ ������ � ��������. 
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
�������� ����������� ������� � ������� ��� ��������� � �������� ������ Threads.  
����� ������� ���������: ������� ������� ���������� ��� ������� ```std::function<...>```.  
�� �������� ��� ������������: ������� ������� ������ �������, ������ ������� ������� �� ������ �������, ���� ```std::vector<Queue_Fn>```, ������ ����������� ������� ������� ��������� ������� ���������� �����.  
����� ����, �� �������� ��� ```Predicator``` (��. �������� ����).
```c++
public:
    typedef std::function<void(Queue_Fn)> Predicator;
```
���, �������� ��� ������ ```for_each()```, ```push()```.  
����� ```for_each()``` ���������� ������ ��������� � �������� �������-��������� ������ ������� �������. �������-�������� ����� ��� ```Predicator```.  ������ ��������� ���������� [��������� ���������](https://metanit.com/cpp/tutorial/7.3.php).  
����� ```push()``` ��������� ������� � ����� �������.
  
  
������ ������� ����� threads, ���� threads.hpp � ������ ���������.  
������ ��������:
```c++
    std::mutex queue_mutex;
    std::vector<Queue_Fn> queue;
```
������ ������:
```c++
    Queue_Fn get_task(bool* queue_end);
    void add_queue(std::vector<Queue_Fn> _queue_async);
    unsigned int queue_size();
```
  
������� ��������� ������� ```TQueue```:
```c++
#include "../queue/queue.hpp"
```
������� ����� ```run()```, ������ �� ����� ���������� � ���� ����� ������� ```TQueue```:
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
����� � ����� ```run()``` �������� ����� ��������� ```std::shared_ptr``` �� ������� ```TQueue<Queue_Fn>```. � ���� ������ ```run()``` ��������� ����� ������. ������ ����� ���������� ����� ```wrapper_loop()```, �� ��� � ����� ���������� ```queue```.
  
��� �� ����� �������� ��������� � ������ ```wrapper_loop()``` � ```loop()```. 
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
� ���� ������ ```loop()``` �������� ����� ```get_task()```, ������ �� ����� ��������� ��� ```queue->get_task(&queue_end)```.
  

�������� ��������� �����.  
������
```c++
    std::vector<Threads<void>::Queue_Fn> queue (4, count_task);
```
������� �� ������
```c++
    typedef std::function<void(void)> Void_Fn;
    std::shared_ptr<TQueue<Void_Fn>> queue = std::make_shared<TQueue<Void_Fn>>(4, count_task);
```
�.�. ����� ������� ��� �� ������, � ����� ��������� �� ������� ```TQueue``` ���� ```std::function<void(void)>```. ���������� ������������� ����������� ����� ���.  
  
���� ���� ����� ���������:
```c++
    TEST_CASE("���������� ������� � ������ �������") {
        CHECK(queue->queue_size() == 4);

        threads.run(queue);
        CHECK(count == 4);
        CHECK(queue->queue_size() == 0);
    }
```
  
� ������ ���������� ```Test_Treads``` �������� ����� ```wrapper_loop()``` �� �������� � ������� �������.
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
  
������
```c++
    std::vector<Test_Threads::Test_Fn> queue2(num_tasks, counts_task);
```
������� �� ������
```c++  
    std::shared_ptr<TQueue<Test_Threads::Test_Fn>> queue2 = std::make_shared<TQueue<Test_Threads::Test_Fn>>(num_tasks, counts_task);
```
�.�. ����� ������� ��� �� ������, � ����� ��������� �� ������� ```TQueue``` ���� ```std::function<void(void)>```. ���������� ������������� ����������� ����� ���.  
  
���� ���� ����� ���������:
```c++
    TEST_CASE("������������� ���������� ������� � ������ �������") {
        CHECK(queue2->queue_size() == num_tasks);

        test_threads.run(queue2, num_tasks);
        for (int i = 0; i < counts.size(); ++i){
            CHECK(counts[i] > 0);
        }
        CHECK(queue2->queue_size() == 0);
    }
```

