# �������� ��������� ���������������
� ������ exec() ������� 1 ������������ ����������� ������� PQexec(). ��� ������� ����� ���������������� ������ ��������� �� ��������� �� ����������. � ���� � ��� ����� ������ �������� ��� �������. ������ ������������ �������� ���������� ����� - ������������ ���������������.  
���������...  
�������� ����� ```PGPool_Threads```, ������� � ������ ������ �������� ��������������� ���������� � ��������� ������ � ������� PostgreSQL.
� ����� ������ ������� �������� ���� ```pg_pool_threads.hpp```. ��� ������������ 
�� ����� ������������ ����� Threads, ������������� [�����.](./Threads.md). ���� ����� ��������� � ������������ ����� � �����, ������������� �� ����� ������ � ����� ��������, ������� ���� � ������������� ����� ���������:  
```c++
#include "../threads/threads.hpp"
```  
����� ����� ������������ ����� ```PGConnection```. ���� � ������������� ����� ���������:
```c++
#include "pg_connection.h"
```  
��� ��� �� ����� ������������ �������, �� � ��������� ����� ����� � ������������ �����: ```pg_pool_threads.hpp```. ������� ���� ���� � ����� ������ �������.   
��� ����� ����� ����������� ������ Threads:
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
�� ��� ��������� ����� ```Test_Threads``` ��������� ������ ```Threads```  ����� ������ ����� � ������ ```Threads```.  
������ �������� ����� ```PGPool_Threads```, ������� ����� ����� ��������. ���� ������ ������ ```Threads``` �������� ��� ���������, �� ������ ����� ������ ����� ��������� ���� �������� ```...Args```. �������� ```R``` ������� ������ ```Threads``` �� ������� ����� ������ ```std::string```.
```c++
...

template <typename ...Args>
class PGPool_Threads: public Threads<std::string, Args...>

...
```
����� �� ������� �������� �������� ```conns```, ������� �������� �������� ����� ���������� �� ������� ������  ```PGConnection```. �.�. ��� �������� - ����� ����������� � �� PostgreSQL.  
����� ��������� ��� ```Queue_Fn```. ��� ��� ������� ```std::string(Args...)```, ������� ���������� ������, � ��������� � �������� ���������� ```Args...```.
```c++
...

private:
    std::vector<std::shared_ptr<PGConnection>> conns;

public:
    typedef std::function<std::string(Args...)> Queue_Fn; 

...
```
����������� �������������� �������� ```conns``` ������ �������� ���������� �� ������� ������ ```PGConnection```. � ����� �������� ����������� �������� ������. �������� �������� �� ����� ������������ ```Threads<std::string, Args...>(num_threads)```. ��� ��� ����� ```Threads``` ������ �� �������� ��� ���������. ������ �������� ��������� �����, ������ �������� ���������� ���������, ����� �� ����� ��������� ������ ����� ������ ```PGPool_Threads```. � ���� ������������ �� ������� ������� ������ ```PGConnection``` � �������� ����� ��������� �� ��� � ������ ```conns```.
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
�� ������� ��� ���� ������������� �����������. ��� ������������ ���, ����� ��������� �����, �� ����� ������������ � � �������� ����������. � ������, ����� �� ��������� ����������� � ���� ������ PostgreSQL �� ����� ������������ �������� ������� ```conn```. �����, �������� ������������ ������ ����� ���������� �� ���������� � PostgreSQL. �������������� �������� ```conns``` � ��������� ����������� �������� ������ ```Threads```, �������� �������� ����� ������� ```_conns```. �.�. ���������� ������� ����� ����� ���������� ����������� � PostgreSQL.
```c++
    PGPool_Threads(std::vector<std::shared_ptr<PGConnection>> _conns)
        : conns(_conns)
        , Threads<std::string, Args...>(_conns.size())
    {}
```

����� ������������ �����-������� ```wrapper_loop()```. ������ ����� ������ ����� ��� � ������� ������ �������� ����� ```loop()``` � �������� ��� �������, ������-������� (� ������ ������ ```Threads```����� ������-������� �� ���������� ```nullptr```) � ������ ���������� ������ ```Queue_Fn```.  
�� ���������� ���� ����� ����� �������� ������-�������, ������� ����� ���������� ```conn```, ����� �������� ������������ ������ ��������. � � ����� ���������� �������� ����� ```exec()```.  
������ ```exec()``` �������� ������ - ��������� ������ ������, ���������� �� ������� �����.  
����������, ������-������� - ���� �������� ����� ������ ����������. �������� ������ ����� ������ ��������� ������� � ������� PostgreSQL. ����� ������� - ��� ��������, ������� ���������� ������ �� ������� �����.
```c++
    void wrapper_loop(std::shared_ptr<TQueue<Queue_Fn>> queue, int num_thread, Args... args) override{
        std::shared_ptr<PGConnection> conn = conns[num_thread];

        Threads<std::string, Args...>::loop(queue, [conn, num_thread, args...](Queue_Fn task) {
            // std::cout << std::string(num_thread, '\t') << num_thread << std::endl;
            conn->exec(task(args...).c_str());
        }, args...);
    }
```
# ��������� ����� PGPool_Threads
�� ��������� ���������� ������ ```run()``` ������ ```PGPool_Threads```. ��� ����� ��������� ������� ����� ���������� ```run()``` - ��� ������ ����� ������. ����� ��� ����� ��������� ���������� ������� ������� ```test_task()``` � ���������� ������� ������ ```exec()``` ���������� ```PGConnection```. ������� ��� ���������� � 10 �������.
��� ������������ ���������� ���������� doctest � fakeit. ��� �� ���������� �������� [�����](./life_hack/add_doctest.md).  
���, � ������ ����� ������������ ```std::chrono_literals```, ������� ���������� ����������:  
```c++
#include <chrono>
```
� ����� pg_pool_threads.hpp ����� �����.  
pg_pool_threads.hpp  
```c++
/* --------------------------------------------------- */
/*                       TESTS                         */
/* --------------------------------------------------- */

TEST_SUITE("���� ������ PGPool_Threads")
{
    namespace test_pg_pool_threads {
        int num_conns = 3; // ���������� ����������.
        int num_tasks = 10; // ���������� �������.
        //----������� ������� ������� �������-������� � ���� �������-�������
        int static count = 0;
        std::mutex static count_mutex;

        std::string test_task() {
            using namespace std::chrono_literals;
            std::this_thread::sleep_for(10ms);
            std::lock_guard<std::mutex> guard_mutex(count_mutex);
            count += 1;
            return "";
        }

        TEST_CASE("���������� ������� � ������ �������") {
            //----������� �������� ���������� � �������� �� � ������
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

            //----������� ������ ������ PGPool_Threads � ������� TQueue
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
�� ���������� ```TEST_SUITE```, � ������ ���� ```TEST_CASE```.  
```doctest``` ������������� ���, ��� ������ ```TEST_SUITE``` ���������� ��������� �������, � ��� ����� �����������, � ����� ����. �.�. ��������� �����, ��� ��� ������� ```main()```. ������� �������� ����� �������� ���������� �������� � ```TEST_CASE```. � ������ ������� � ```TEST_CASE``` ���������� ������ �������. ������� ����������� ����� ������� �������� ������� ```TEST_CASE```. � ������������ � ```doctest``` ��� ����������� �� �������.  
  
������ ```TEST_SUITE``` ��������� ������������ ���� ```namespace test_pg_pool_threads```, ��� ��� ����� ����������� ����� ����������, ������������ � ������ ������. � ������, �������� �������� ������������� ������������ ���� � ������.  
������ ```TEST_SUITE``` ������� ������� ������� �������-������� � ���� �������-�������. 
```c++
    std::string test_task() {
        using namespace std::chrono_literals;
        std::this_thread::sleep_for(10ms);
        std::lock_guard<std::mutex> guard_mutex(count_mutex);
        count += 1;
        return "";
    }

```   
����� ������������� ����� �� 10 ����������� ��� ���� ����� ���� ����� �� ����� ��������� ��� �������.  � ������� ```count_mutex``` �������� ```count``` (��. [�����](http://scrutator.me/post/2012/04/04/parallel-world-p1.aspx)).
  
������ ```TEST_CASE``` ������� ������� � ��������� �����.  
��� ��� ����� ```PGPool_Thrads``` ���������� ���������� � PostgreSQL, �� ��� ����� ����������� � �������� ���� ���������� � �����. �� �� �� ����� ��������� �������� ���������� � ������. ���������� ���������� ```fakeit``` (������������ [�����](https://github.com/eranpeer/FakeIt/wiki/Quickstart#invocation-matching)). ������� ��� �������� ������� ```PGConnection``` � ��������� � ��� ����� ```exec()``` ������ ```PGConnection```.
```c++
        fakeit::Mock<PGConnection> conn1;
        fakeit::Fake(Method(conn1, exec));
        conns[0] = std::shared_ptr<PGConnection>(&conn1.get(), [](...) {});
```
��������� ������ ���� �������� ����� ��������� �� �������� ���������� � ������ ```conns``` (���� ����������� ������������� ����� ���������� � ���������� ```fakeit``` - ��. [�����](https://github.com/eranpeer/FakeIt/issues/60)). ����� �� �� ���������� ���� ������ ������, ��� ����� ```exec()``` ��������� ������� ��� ������ �� ������� ����������� (���� �����) �������� ������� ����������. �������, �� ������ �������� ������ ����� ����� ��������� ������ ```exec()``` ��� ��������� ����.
  
� ��������� ������� ������� ������ ������ ```PGPool_Threads``` � ������� ```TQueue```. ��� �������� ������� ```PGPool_Threads``` � ����������� �������� ������ �������� ���������� ```conns```. ��� ����� ��� �������� ������� ��� ���� ����������� ������ ```PGPool_Threads```, ������ ��� ��� �������������� �������������� ����� ������ ���������� � PostgreSQL ����������� ������ ������������ � ���� ������ �������. ���, ������-��, ������� ����, �� ��� ������ ������������.  
```c++
        PGPool_Threads pg_pool_threads(conns);
        typedef std::function<std::string(void)> String_Fn;
        std::shared_ptr<TQueue<String_Fn>> queue = std::make_shared<TQueue<String_Fn>>(num_tasks, test_task);
```
� �����, ����������, ���� �����.
```c++
        CHECK(queue->queue_size() == num_tasks);
        pg_pool_threads.run(queue);
        CHECK(count == num_tasks);
        CHECK(queue->queue_size() == 0);
        fakeit::Verify(Method(conn1,exec)).AtLeast(1);
        fakeit::Verify(Method(conn2,exec)).AtLeast(1);
        fakeit::Verify(Method(conn3,exec)).AtLeast(1);
```
��������� ������ ������� ����� �������� ```run()```.  
��������� ```run()```.  
�������� ���������� ������� ```test_task()```.  
��������� ������ �������.  
��������� ����� ������ ```exec()``` � �������� ��������. ����� ������ ���� ������ ����-�� ���� ���. ������ ��������� �� �������, ��� ��� ������ ������� ��������������.