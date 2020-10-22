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

## Содержание
{: .no_toc }  
1. TOC
{:toc}

# Настроим VSCode
## mpg/.vscode/task.json
После копирования папки с проектами в линукс нужно запустить компиляцию нашего кода. Для этого добавим задачу в файл ```task.json```.  До сих пор есть две задачи ```app``` и ```tests```. Добавим еще две ```app linux``` и ```tests linux```:  
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
                "${workspaceFolder}\\app",
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
                "${workspaceFolder}\\test",
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
- ```"${workspaceFolder}\\app"``` - выходное имя файла скомпилированной программы. VSCode в линукс работет немного по другому и после компиляции выходной файл появляется не в папке ```mpg```, а в корневой папке наших проектов. Т.е. в линукс ```${workspaceFolder}``` работает правильно, только выходной файл почему-то называется ```mpgapp```;
- ```"-I", "/usr/include/postgresql"``` - в отличие от windows заголовочные файлы лежат где положено можно посмотреть подсказку [здесь](https://www.linux.org.ru/forum/general/11736854);
- далее идут папки наших папок заголовочных файлов;
- ```"-L", "/usr/lib/x86_64-linux-gnu",``` - библиотеки тоже лежат где положено;
- ```"-lpq", "-lpthread"``` - имена используемых подключаемых библиотек. Здесь видно, что мы дополнительно подключили библиотеку ```lpthread``` для использования потоков.  
  
## mpg/.vscode/launch.json
Добавим в файл ```launch.json``` еще две конфигурации для отладки.  
```json
    "configurations": [
        ...,
        {
            "name": "app linux",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}\\..\\mpgapp",
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
            "program": "${workspaceFolder}\\..\\mpgtest",
            "args": [],
            "stopAtEntry": true,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "preLaunchTask": "tests linux"
        }
    ]

```
Здесь все понятно, кроме, пожалуй вот этой строки ```"${workspaceFolder}\\..\\mpgapp"```. Как уже говорилось в предыдущем разделе? файл почему-то компилируется в корень папки с проектами с названием ```mpgapp```.  
  
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
Заключим в конструкцию: ```#ifdef _WIN32 ... #endif``` объявленин заголовочного файла ```#include <windows.h>```. Подробнее [здесь](https://coderoad.ru/6649936/C-компиляция-на-Windows-и-Linux-переключатель-ifdef).  
Объявим переменные ```CP_UTF8``` и ```CP_ACP```. Их объявляем только для линукс (```#ifdef __linux__```), так как для windows они определены в ```<windows.h>```.  
Чтобы не исправлять код во всей программе мы оставим функции ```utils::cpt()``` и ```utils::is_valis_utf8()```, только для линукс реадизуем из по-другому.  
Ну и, конструкция ниже обязательна:  
```c++
#ifndef UTILS_H
#define UTILS_H
...
#endif // UTILS_H
```
Компилятор ```cl.exe``` более умный, наверно. Компилировал без этой конструкции. Компилятор ```g++``` без этой конструкции пытается несколько раз засунуть в программу этот код, если заголовок используется с нескольких местах программы.
  
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
Функцию ```utils::cpt()``` исправим на следующую:  
```c++
    std::string cpt(const char *str, unsigned int srcCP, unsigned int dstCP){
```
Здесь мы убрали значения по умолчанию для аргументов ```srcCP``` и ```dstCP```, так как компилятор ```g++``` требует устанавливать значения по умолчанию в заголовочном файле, если таковой есть.  
  
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
Здесь также одно исправление. Как уже упоминалось, компилятор ```g++``` требует чтобы значения по умолчанию аргументов функций указывалось в определении функции, а не в реализации.  
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
## mpg/pg_connection.cpp
Здесь неверно написана работа с ```exceptions```.  
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
    throw "No config file. Add config file config/config.json";
```
  
## mpg/pg_pool_async.hpp
Добавим:  
```c++
#include <cstring>// linux for strerror()
```
Этот модуль нужен для работы функции ```strerror()``` в линукс.

## mpg/app.cpp
```c++
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
Так как ```SetConsoleOutputCP(1251);``` работает только в windows, заключим эту функцию в следущую конструкцию: ```#ifdef _WIN32  #endif```. Подробнее [здесь](https://coderoad.ru/6649936/C-компиляция-на-Windows-и-Linux-переключатель-ifdef).

Кроме того, компилятор ```g++``` под линукс более требовательный. Например выдает ошибку если не включить возврат из функции ```return 0```.  
  
## mpg/test.cpp
Заключим в конструкцию: ```#ifdef _WIN32 ... #endif``` следующий код:
- ```#include <windows.h>``` - хотя, этот заголовочный файл можно вообще убрать.
- ```SetConsoleOutputCP(1251);```