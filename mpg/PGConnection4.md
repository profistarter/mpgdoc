---
layout: default
title: Часть 4. Переносим на линукс
parent: Разработка mpg
nav_order: 4
---
# Часть 4. Переносим на линукс
{: .no_toc }  
До сих пор мы разрабатывали нашу систему в windows. Перенесем теперь ее в linux.  
Мы использовали Ubuntu 20.04.  
Как установить VSCode на Ubuntu можно найти [здесь](https://losst.ru/kak-ustanovit-visual-studio-code-na-ubuntu)  или [здесь](https://linuxize.com/post/how-to-install-visual-studio-code-on-ubuntu-20-04/).  
Как установить ```с++``` на ubuntu смотри [здесь](https://linuxconfig.org/how-to-install-g-the-c-compiler-on-ubuntu-20-04-lts-focal-fossa-linux). Поисковик в помощь.  
Не забудем поставить модуль с++ для VSCode.  
Установка PostgreSQL на линукс [здесь](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-20-04-ru).  
PostgreSQL устанавливается на линукс с ролью postgres по умолчанию без пароля. Т.е. нужно задать пароль. 
В консоли линукс набираем:  
```
~> sudo -u postgres psql postgres
postgres=# \password postgre
```
Указываем пароль.  Для выхода наберите ```\q```. 
  
Теперь создаем базу данных:  
```
~> sudo psql -h localhost -p 5432 -U postgres
postgres=# CREATE DATABASE testdb;
CREATE DATABASE
postgres=# \c testdb
testdb=#
```
Если у вас появилось приглашение ```testdb=#```, то все в порядке. Для выхода наберите ```\q```.    
  
Рабочий проект можно найти [здесь](https://github.com/profistarter/mpg/tree/90125cb4fc948396317f013c712c3b005cd0a350).

## Содержание
{: .no_toc }  
1. TOC
{:toc}
 
# Настроим VSCode
## mpg/.vscode/task.json
После копирования папки с проектами в линукс, нужно запустить компиляцию нашего кода. Для этого добавим задачи в файл ```task.json```.  До сих пор есть две задачи ```app``` и ```tests```. Добавим еще две ```app linux``` и ```tests linux```:  
```json
    tasks: [
        ...,
        {
            "type": "shell",
            "label": "app linux",
            "command": "/usr/bin/g++",
            "args": [
                "-std=c++1z",
                "-g",
                "app.cpp",  
                "../utils/utils.cpp",           
                "pg_connection.cpp",
                "-o",
                "${workspaceFolder}/app",
                "-I", "/usr/include/postgresql",
                "-I", "${workspaceFolder}/../utils",
                "-I", "${workspaceFolder}/../_include",
                "-L", "/usr/lib/x86_64-linux-gnu",
                "-lpq", "-lpthread"
            ],
            "options": {
                "cwd": "${workspaceFolder}"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            }
        },
        {
            "type": "shell",
            "label": "tests linux",
            "command": "/usr/bin/g++",
            "args": [
                "-std=c++1z",
                "-g",
                "test.cpp",  
                "../utils/utils.cpp",           
                "pg_connection.cpp",
                "-o",
                "${workspaceFolder}/test",
                "-I", "/usr/include/postgresql",
                "-I", "${workspaceFolder}/../utils",
                "-I", "${workspaceFolder}/../_include",
                "-L", "/usr/lib/x86_64-linux-gnu",
                "-lpq", "-lpthread"
            ],
            "options": {
                "cwd": "${workspaceFolder}"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            }
        }
    ]
```
Поясним аргументы компилятора:
- ```"-std=c++1z"``` - указывает компилятору использовать стандарт с++17;
- далее идут исходные файлы нашей программы;
- ```"${workspaceFolder}/app"``` - выходное имя файла скомпилированной программы;
- ```"-I", "/usr/include/postgresql"``` - в отличие от windows заголовочные файлы лежат где положено можно посмотреть подсказку [здесь](https://www.linux.org.ru/forum/general/11736854);
- далее идут папки "наших" папок заголовочных файлов;
- ```"-L", "/usr/lib/x86_64-linux-gnu",``` - библиотеки тоже лежат где положено;
- ```"-lpq", "-lpthread"``` - имена используемых подключаемых библиотек. Здесь видно, что мы дополнительно подключили библиотеку ```lpthread``` для использования потоков.  
  
## mpg/.vscode/launch.json
Добавим в файл ```launch.json``` еще две конфигурации для отладки нашего приложения и тестов в линукс.  
```json
    "configurations": [
        ...,
        {
            "name": "app linux",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}\\app",
            "args": [],
            "stopAtEntry": true,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "preLaunchTask": "app linux"
        },
        {
            "name": "tests linux",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}\\test",
            "args": [],
            "stopAtEntry": true,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "preLaunchTask": "tests linux"
        }
    ]

```
Здесь, вроде, все понятно.  
  
## mpg/.vscode/c_cpp_properties.json
Для правильной работы ```intellisense``` необходимо настроить конфигурацию модуля с++. Без этого код будет подсвечиваться красными волнистыми линиями. Откроем файл ```c_cpp_properties.json``` и добавим конфигурацию для линукс:  
```json
    "configurations": [
        ...,
        {
            "name": "Linux",
            "includePath": [
                "${workspaceFolder}/**",
                "${default}",
                "${workspaceFolder}/../_include/**" 
            ],
            "defines": [],
            "compilerPath": "/usr/bin/gcc",
            "cStandard": "gnu18",
            "cppStandard": "gnu++17",
            "intelliSenseMode": "gcc-x64"
        }
    ],
    ...

```
  
# Исправим утилиты и дополнительные модули
## utils/utils.h
Приведем файл целиком, благо он маленький:  
```c++
#ifndef UTILS_H
#define UTILS_H

#ifdef _WIN32
#include <windows.h>
#endif
#include <string>

namespace utils {
    #ifdef __linux__
    extern unsigned int CP_UTF8;
    extern unsigned int CP_ACP;
    #endif
    std::string cpt(const char *str, unsigned int srcCP = CP_UTF8, unsigned int dstCP = CP_ACP);
    bool is_valid_utf8(const char * string);
}

#endif // UTILS_H

```
Заключим в конструкцию ```#ifdef _WIN32 ... #endif``` объявление заголовочного файла ```#include <windows.h>```. Подробнее [здесь](https://coderoad.ru/6649936/C-компиляция-на-Windows-и-Linux-переключатель-ifdef).  
Объявим переменные ```CP_UTF8``` и ```CP_ACP```. Их объявляем только для линукс (```#ifdef __linux__```), так как для windows они определены в ```<windows.h>```.  
Чтобы не исправлять код во всей программе мы оставим функции ```utils::cpt()``` и ```utils::is_valis_utf8()```, только для линукс реализуем их по-другому.  
Ну и, конструкция ниже обязательна:  
```c++
#ifndef UTILS_H
#define UTILS_H
...
#endif // UTILS_H
```
Компилятор ```cl.exe``` более умный, наверно. Компилировал без этой конструкции. Компилятор ```g++```, без этой конструкции, пытается несколько раз "засунуть" в программу код этого файла, если заголовок используется в нескольких местах программы.
  
## utils/utils.cpp
Откроем файл ```utils/utils.cpp```.  
Заключим ```#include <windows.h>``` в следущую конструкцию: 
```c++
#ifdef _WIN32 
#include <windows.h> 
#endif
```
Далее включим заголовочный файл:  
```c++
#include "utils.h"
```
Примечательно, что ```cl.exe``` в windows компилировал без него. Не знаю хорошо это или плохо???  
  
Внутри ```namespace utils``` добавим инициализацию переменнных ```CP_UTF8``` и ```CP_ACP``` (только для линукс):  
```c++
    #ifdef __linux__
    unsigned int CP_UTF8 = 0;
    unsigned int CP_ACP = 0;
    #endif
```
Заголовок функции ```utils::cpt()``` исправим на следующий:  
```c++
...
    std::string cpt(const char *str, unsigned int srcCP, unsigned int dstCP)
    {
    ...
    }
...
```
Здесь мы убрали значения по умолчанию для аргументов ```srcCP``` и ```dstCP```, так как компилятор ```g++``` требует устанавливать значения по умолчанию в объявлении функции. Объявление функции мы поместили в заголовочный файл ```utils.h```.  
  
Обе функции ```utils::cpt()``` и ```utils::is_valis_utf8()```, реализованные для windows, заключим в следующую конструкцию ```#ifdef _WIN32``` и добавим реализацию линукс:  
```c++
    #ifdef _WIN32
    ...
    #elif __linux__
    std::string cpt(const char *str, unsigned int srcCP, unsigned int dstCP){
        return str;
    }
    bool is_valid_utf8(const char * string){
        return true;
    }
    #else

    #endif

```
## utils/queue.hpp
Откроем файл ```utils/queue.hpp```.  
Здесь одно исправление. Есть, конечно, предположение, почему так, но как-то странно.  
Заменим конструкцию:  
```c++
    std::vector<Queue_Fn>::iterator iter = queue.begin();
```
следующей конструкцией:  
```c++
    typename std::vector<Queue_Fn>::iterator iter;
    iter = queue.begin();
```
  
## threads/threads.hpp
Откроем файл ```threads/threads.hpp```.   
Здесь также одно исправление. Как уже упоминалось, компилятор ```g++``` требует, чтобы значения по умолчанию аргументов функций указывалось в определении функции, а не в реализации.  
**Заменим конструкцию**:  
```c++
class Threads {
    ...
    Threads(int _num_threads);
    ...
}
```
следующей конструкцией:  
```c++
class Threads {
    ...
    Threads(int _num_threads = std::thread::hardware_concurrency());
    ...
}
```
  
**Также заменим конструкцию**:  
```c++
Threads<R, Args...>::Threads(int _num_threads = std::thread::hardware_concurrency())
```
следующей конструкцией:  
```c++
Threads<R, Args...>::Threads(int _num_threads)
```
  
# Исправим программу mpg
## mpg/pg_connection.h
Добавим модуль  ```#include <memory>```. Мы используем ```shared_ptr```. Странно, что в windows компилировалось.  
```c++
#include <memory>
```
  
## mpg/pg_connection.cpp
Здесь неверно написана работа с ```exceptions```. Позже мы пересмотрим работу с ```exceptions``` в этом классе.
Включим модуль ```exceptions```:  
```c++
#include <exception>
```
  
Заменим конструкцию:  
```c++
    throw new std::exception("No config file. Add config file config/config.json");
```
следующей конструкцией:  
```c++
    throw std::runtime_error("No config file. Add config file config/config.json");
    
```
И изменим ```catch``` секцию: 
```c++
    ...
    
    catch(std::exception& e) {
        throw;
    }
```
  
## mpg/pg_pool_async.hpp
Добавим:  
```c++
#include <cstring>// linux for strerror()
```
Этот модуль нужен для работы функции ```strerror()``` в линукс.
  
В функции ```select()``` исправьте первый аргумент:  
```c++
        if (select(nfds+1, &input_mask, &output_mask, NULL, &timeout) < 0)
        {
            ...
        }
```
В windows он не учитывался. В линукс важно, чтобы значение было на единицу больше значения самого большого дескриптора.  См. [здесь](http://ru.manpages.org/select/2).

## mpg/app.cpp
```c++
#ifdef _WIN32  
#include <windows.h>
#endif
...

int main()
{
    #ifdef _WIN32
    SetConsoleOutputCP(1251);
    #endif

    ...
    return 0;
}
```  
Заголовочный файл ```#include <windows.h>``` мы заключили в конструкцию: ```#ifdef _WIN32  #endif```, хотя он, похоже, вообще не нужен. Без него работает в windows.

Так как ```SetConsoleOutputCP(1251);``` работает только в windows, заключим эту функцию в следущую конструкцию: ```#ifdef _WIN32  #endif```. Подробнее [здесь](https://coderoad.ru/6649936/C-компиляция-на-Windows-и-Linux-переключатель-ifdef).

Кроме того, компилятор ```g++``` под линукс более требовательный. Например выдает ошибку если не включить возврат из функции ```return 0```.  
  
## mpg/test.cpp
Заключим в конструкцию: ```#ifdef _WIN32 ... #endif``` следующий код:
- ```#include <windows.h>``` - хотя, этот заголовочный файл можно, вообще, убрать.
- ```SetConsoleOutputCP(1251);```
  
# Заключение
В папке ```threads/.vscode``` есть еще один ```task.json``` с задачей компиляции тестов для класса ```Treads```. Мы его не стали переносить в линукс, потому что эта задача больше не используется. Тесты класса  ```Threds``` запускаются из тестов нашего главного проиложения в папке ```mpg```.  
В windows в режиме отладки русская кодировка выводится неправильно. Это, скорее всего, связано с отличающейся константой ```_WIN32```, используемой при отладке. Но, пока разбираться не стали.
