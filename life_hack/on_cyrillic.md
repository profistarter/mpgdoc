---
layout: default
title: Кириллица в консоли
parent: Лайфхаки
nav_order: 2
---
# Кириллица в консоли
{: .no_toc }  
Используя консоль в windows, мы сталкиваемся с проблемой отображения русского языка.  
Теория [здесь](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/).  
И [здесь](http://www.unn.ru/pages/e-library/vestnik/99999999_West_2011_3%282%29/47.pdf).
  
Итак, имеем две составляющие, которые нужно соединить:  
- кодовая страница текста;  
- кодовая страница, которую понимает консоль и устройство вывода (может файл).  

Если нужно кросплатформенное решение, то пойдем по пути преобразования кодировки текста. Сам код программы и конфигурационных файлов будем хранить в utf-8, но так как консоль windows не понимает его, то будем преобразовывать кодировку текста в кодировку, понятную консоли windows (в linux все проще ничего преобразовывать не надо).  
  
Если программа будет использоваться только в windows, то можно и код программы и конфигурационных файлов хранить в кодировке, понятной консоли windows.
## Содержание
{: .no_toc }  
1. TOC
{:toc}
# Исполизуем функции преобразования кодовой страницы
## Возможные преобразования кодовой страницы
Если создаем кроссплатформенное приложение, то нужно код программы и конфигурационцих файлов хранить в utf-8.  
Если файл в другой кодировке (например, ```windows1251```), то его необходимо пересохранить в кодировке ```UTF-8```.  
В нижней панели VSCode можно увидеть метку ```UTF-8``` (или другая - в которой сохранен файл). Нажимаем на это. Откроется всплывающее окно. Нажимаем *Save with encoding*. Теперь можно выбрать новую кодировку для этого файла (например, ```UTF-8```).  
Чтобы программа правильно отображала русский текст в консоли или правильно сохраняла в файл, нужно кодировку этого текста преобразовывать. Например, когда вы пишите в программе:  
```c++
    std::cout << "Русский текст." << std::endl;
```
В windows программа выведет "кроказябры", так как сам код программы сохранен в кодировке utf-8. Чтобы этого избежать преобразовываем кодовую страницу текста в кодовую страницу консоли с помощью функции ```cpt()```(см. ниже):  
```c++
    std::cout << cpt("Русский текст.", CP_UTF8, CP_OEMCP)  << std::endl;
    //или
    SetConsoleOutputCP(1251);
    std::cout << cpt("Русский текст.")  << std::endl; // CP_UTF8, CP_ACP - по умолчанию
```  
Еще, если, например, нужно записать текст, введенный в консоли windows в файл, то используем обратное преобразование (помним, что консоль понимает одну кодировку - обычно cp866, - а файл записывается в другой кодировке - y нас utf-8).
```c++
    std::string str;
    std::cin >> str;
    std::string str_utf8 = cpt(str.c_str(), CP_OEMCP, CP_UTF8);
    std::cout << is_valid_utf8(str_utf8.c_str()) << std::endl;
```
Описание функции ```cpt()``` см. ниже.  
## Создаем модуль utils
В папке наших проектов на одном уровне с проектом ```mpg``` создаем папку ```utils```. В папке ```utils``` создаем файлы ```utils.h``` и ```utils.cpp```. В файл заголовков помещаем определения функций преобразования кодировки:  
utils.h
```c++
#include <windows.h>
#include <string>

std::string cpt(const char *str, unsigned int srcCP = CP_UTF8, unsigned int dstCP = CP_ACP);
bool is_valid_utf8(const char * string);
```
Эти функции взяты [отсюда](https://ru.stackoverflow.com/questions/783946/Конвертировать-в-кодировку-utf8).  
```cpt()``` - фукция преобразования кодировки;  
```is_valid_utf8()``` - функция проверки на кодировку utf-8.  
  
В функцию  ```cpt()``` мы передаем три аргумента:
- ```const char *str``` - строка, которую хотим преобразовать;
- ``` unsigned int srcCP``` - номер кодовой страницы строки. По умолчанию ```CP_UTF8``` - эта константа определена в ```<windows.h>```. В подавляющем большинстве случаев будем использовать именно ее.
- ```unsigned int dstCP``` - номер кодовой страницы, в которую необходимо преобразовать строку. По умолчанию ```CP_ACP``` - эта константа определена в ```<windows.h>```.  
```CP_ACP``` - системная кодовая страница ANSI Windows по умолчанию (в большинстве случаев windows1251).  
Мы также можем использовать ```CP_OEMCP``` - текущая системная кодовая страница OEM (в большинстве случаев cp866).  
Подробнее о константах кодовых страниц можно посмотреть в описании функции [MultiByteToWideChar] (https://docs.microsoft.com/en-us/windows/win32/api/stringapiset/nf-stringapiset-multibytetowidechar).  
Что такое ANSI и OEM см. [теорию](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/).
  
В файл реализации помещаем реализацию этих фукций:  
utils.cpp
```c++
#include <windows.h>
#include <string>
 
//Взято отсюда: https://ru.stackoverflow.com/questions/783946/Конвертировать-в-кодировку-utf8

//Функция преобразования кодировки строки. По умолчанию из utf-8 в текущую OEM (обычно cp866).
std::string cpt(const char *str, unsigned int srcCP = CP_UTF8, unsigned int dstCP = CP_ACP){
	std::string res;	
	int result_u, result_c;

	result_u = MultiByteToWideChar(srcCP,	0,	str, -1, 0,	0);
	if (!result_u)	return 0;

	wchar_t *ures = new wchar_t[result_u];
	if(!MultiByteToWideChar(srcCP, 0, str, -1, ures, result_u))	{
		delete[] ures;
		return 0;
	}
	result_c = WideCharToMultiByte(dstCP, 0, ures, -1, 0, 0,	0, 0);
	if(!result_c)	{
		delete [] ures;
		return 0;
	}
	char *cres = new char[result_c];
	if(!WideCharToMultiByte(dstCP, 0, ures, -1, cres, result_c, 0, 0))	{
		delete[] cres;
		return 0;
	}
	delete[] ures;
	res.append(cres);
	delete[] cres;
	return res;
}

//Взято отсюда: https://ru.stackoverflow.com/questions/783946/Конвертировать-в-кодировку-utf8

//Функция проверки кодировки строки на utf-8.
bool is_valid_utf8(const char * string){
    if(!string){return true;}
    const unsigned char * bytes = (const unsigned char *)string;
    unsigned int cp;
    int num;
    while(*bytes != 0x00){
        if((*bytes & 0x80) == 0x00){
            // U+0000 to U+007F 
            cp = (*bytes & 0x7F);
            num = 1;
        }
        else if((*bytes & 0xE0) == 0xC0){
            // U+0080 to U+07FF 
            cp = (*bytes & 0x1F);
            num = 2;
        }
        else if((*bytes & 0xF0) == 0xE0){
            // U+0800 to U+FFFF 
            cp = (*bytes & 0x0F);
            num = 3;
        }
        else if((*bytes & 0xF8) == 0xF0){
            // U+10000 to U+10FFFF 
            cp = (*bytes & 0x07);
            num = 4;
        }
        else{return false;}
        bytes += 1;
        for(int i = 1; i < num; ++i){
            if((*bytes & 0xC0) != 0x80){return false;}
            cp = (cp << 6) | (*bytes & 0x3F);
            bytes += 1;
        }
        if( (cp > 0x10FFFF) ||
            ((cp <= 0x007F) && (num != 1)) || 
            ((cp >= 0xD800) && (cp <= 0xDFFF)) ||
            ((cp >= 0x0080) && (cp <= 0x07FF)  && (num != 2)) ||
            ((cp >= 0x0800) && (cp <= 0xFFFF)  && (num != 3)) ||
            ((cp >= 0x10000)&& (cp <= 0x1FFFFF)&& (num != 4)) ){return false;}
    }
    return true;
}
```
# Изменяем  кодовую страницу консоли и программы
Если мы решили писать программу только для windows, то меняем кодировку наших файлов на кодировку, понятную консоли windows. Для этого сначала нужно настроить кодировку файлов по умолчанию в VSCode.  
  
Для этого нажимаем Ctrl + Shift + P и набираем ```Preferences: Open settings (JSON)```. Откроется файл конфигурации VSCode.  
Вставляем стоку ```"files.encoding": "windows1251"``` или ```"files.encoding": "cp866"```.  
Если выбираем ```cp866```, то больше ничего делать не надо (консоль windows работает по умолчанию в кодировке ```cp866```), но это прошлый век.  
  
Если файл в другой кодировке (например, ```UTF-8```), то его необходимо пересохранить в нужной кодировке.  
В нижней панели VSCode можно увидеть метку ```UTF-8``` (или другая - в которой сохранен файл). Нажимаем на это. Откроется всплывающее окно. Нажимаем *Save with encoding*. Теперь можно выбрать новую кодировку для этого файла (например, ```windows1251```).  
  
Для того чтобы поменять кодировку в консоли для отображения русских символов, в главный файл программы добавим следующие строки (подробнее [здесь](https://code-live.ru/post/cpp-cyrillic-manual/)):
```c++
#include <windows.h>

int main(int argc, char** argv) {
    SetConsoleCP(1251);// Ввод кириллицы с консоли.
    SetConsoleOutputCP(1251);// Вывод кириллицы на консоль.

    //Вывод кириллицы из текста программы на консоль.
    //При использовании предыдущих функций не влияет, 
    //за исключением разделитель дробной части числа, формат даты, времени и пр.
    //setlocale(LC_CTYPE, ".1251"); 
    ...
```
К сожалению, на ```utf-8``` не можем поменять.
  
