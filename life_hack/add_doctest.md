# ���������� ���������� ������������ doctest � FakeIt
## ��������� ���������� doctest
��� ������������ ��������� ����� ������������ ���������� doctest. ��������� zip ����� � ����� https://github.com/onqtam/doctest. ������������� � �������� ���� � ������ doctest/doctest.h � ����� _include, ������������� �� ����� ������ � ����� ��������.  
## ��������� ���������� FakeIt
��� �������� �������� � ��������� �������� ����� ������������ ���������� FakeIt. ��������� zip ����� � ����� https://github.com/eranpeer/FakeIt. ������������� � �������� ���� single_header/standalone/fakeit.hpp � ����� _include/fakeit. ����� _include ����������a �� ����� ������ � ����� ��������. 

## ������� ���� ������� � ��������� ������������
���� ����� ������������� ��� ����������, �������� ������� ���� � ��������  ```main()```, �� ������� ��� ���� ���� � �������� ```main()``` � ������� ��� test.cpp.
���� ����� ������������� ��� ����������, �� � ����� ����������, ������� ����� ����������� ��������� ���� test.cpp.
```c++
#define DOCTEST_CONFIG_IMPLEMENT
#include <windows.h>
#include "doctest/doctest.h"
#include "threads.hpp"

int main(int argc, char** argv) {
    SetConsoleCP(1251);
    SetConsoleOutputCP(1251);
    setlocale(LC_CTYPE, ".1251");

    doctest::Context context;
    context.applyCommandLine(argc, argv);
    context.setOption("no-breaks", true); // don't break in the debugger when assertions fail

    int res = context.run(); 
    if(context.shouldExit()) // important - query flags (and --exit) rely on the user doing this
        return res;          
    int client_stuff_return_code = 0;   
    return res + client_stuff_return_code; // the result from doctest is propagated here as well
}
```
���������� ```#define DOCTEST_CONFIG_IMPLEMENT``` �������� ���������� doctest, ��� �� ������� ���� ������� main(). ���������� ������������� ����������� ������������ ������������� ����������� ������� main() (��. ������������).  
���� ������� main() �����, ����� �������� ���������� ���������� ��������� � �������: �������� ������ ����� ������ �� ������� (��������� [�����](./on_cyrillic.md)).  
```#include <windows.h>``` ����� ��� ������ ��������� � �������.  
```#include "doctest/doctest.h"``` - ���������� doctest.  
```#include "threads.hpp"``` - ���� ����������, ������� �� ���������.  
  
```c++
    SetConsoleCP(1251);
    SetConsoleOutputCP(1251);
    setlocale(LC_CTYPE, ".1251");
```
���� �������� ��������� ���������� ��������� � �������.  
  
����� ���������� ��� ������� main() - ��� ���������������� ���������� doctest. �� ����� ��� �� ������������.

## ���������� ������
� ����� ����������� ���������� (��������, threads.hpp) �������� ���������� doctest:  
threads.hpp  
```c++
#include "doctest/doctest.h"

...
```
  
� ����� ������������ ����� ��������� �����:  
threads.hpp  
```c++
...

TEST_SUITE("����"){
    short k = 1;
    TEST_CASE("��������� ...") {
        CHECK(k == 1);
    }
}
```
  
## ��������� �����������
���������� VSCode � ��������� ����������� ������� [�����](./preparation.md).  
��������� task.json � launch.json. launch.json ����������� ��� ��� ������� ������.  
� task.json � ���������� ����������� ```"args"``` ���������:  
```"test.cpp"``` - ������������� ����,
```"${workspaceFolder}\\test.exe"``` - �������� ���� (�������� ����������).
```json
{
    "version": "2.0.0",
    "tasks": [
        {
            ...

            "args": [
                ...,
                "${workspaceFolder}\\test.exe",
                "test.cpp"
            ]

            ...
        }
    ]
}
```
���� � ������� ��� ���� task.json, �� ��������� ��� ���� ������, ����� ������ ����������� � ������ ������� ```"label"``` - ��� �������� ������. ������� �� ```"tests"```. ����� ������ ������� ```"args"``` ��� ������� ����.
```json
{
    "version": "2.0.0",
    "tasks": [
        {
            ...
        },
        {
            ...
            "label":"tests",
            ...

        }
    ]
}
```
���� � ������� ��� ���� launch.json, �� ��������� ��� ���� ������������, ����� ������ ����������� � ������ ������� ```"name"```, ```"program"``` � ```"preLaunchTask"```. ��� ����� ����� ���������� �����.   
```"preLaunchTask"``` - ��� �������, ��� �������� �������� ������, ����������� ����� ��������.
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            ...
        },
        {
            ...
            "name":"tests",
            "program": "test.exe",
            ...
            "preLaunchTask": "tests"
        }
    ]
}
```
��� ���� ����� �������������� �����, ������� ```Ctrl + Shift + B```. ������ ���������� ��� ������. �������� ������ ```tests```. 
���! ����������� � ��������� test.exe.
