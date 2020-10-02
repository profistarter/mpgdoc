# ”скор€ем использу€ многопоточность
[«десь](./PGConnection.md) мы разработали простое приложение запросов к базе данных PostgreSQL.  
¬ методе exec() используетс€ блокирующа€ функци€ PQexec() из библиотеки libpq. Ёта функци€ будет приостанавливать работу программы до окончани€ ее выполнени€. ј если у нас будет тыс€ча запросов или миллион. ћногие существующие средства предлагают выход - использовать многопоточность.  
ѕопробуем...  
ћы будем использовать класс Threads, разработанный [здесь.](./Threads.md). Ётот класс находитс€ в заголовочном файле в папке, расположенной на одном уровне с нашим проектом, поэтому путь к заголовочному файлу следующий:  
```../threads/threads.hpp```  
“ак как мы будем использовать шаблоны, то и описывать класс будем в заголовочном файле: ```pg_connection_pool.hpp```  
Ќаш класс будет наследником Threads:
```c++
#include <mutex>
#include <thread>
#include <vector>
#include "../threads/threads.hpp"
#include "pg_connection.h"

template <typename ...Args>
class Pool_Thread: public Threads<std::string, Args...>
{
private:
    std::vector<std::shared_ptr<PGConnection>> conns;

public:
    Pool_Thread(int num_threads = std::thread::hardware_concurrency())
        : conns(std::vector<std::shared_ptr<PGConnection>>(num_threads))
        , Threads<std::string, Args...>(num_threads)
    {
        for (int i = 0; i < num_threads; i++) {
            conns[i] = std::make_shared<PGConnection>();
        }
    }
    void wrapper_loop(int num_thread, Args... args) override{
        std::shared_ptr<PGConnection> conn = conns[num_thread];

        Threads<std::string, Args...>::loop([conn, num_thread, args...](Queue_Fn task) {
            // std::cout << std::string(num_thread, '\t') << num_thread << std::endl;
            int res = conn.get()->exec(task(args...).c_str());
        }, args...);
    }
};
```