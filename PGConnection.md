# ������� ��������� ��� ������ � PostgreSQL
� �������� 1 �������� ��� ������� ���������, ������� ������������� ���������� � ��������� ������.  
app.cpp
```c++
#include <iostream>
#include <memory>
#include <string>
#include <libpq-fe.h>

class PGConnection
{
private:

    std::string dbhost = "localhost";
    std::string dbport = "5433";
    std::string dbname = "testdb";
    std::string dbuser = "postgres";
    std::string dbpass = "1";

    std::shared_ptr<PGconn> connection;

public:
    PGConnection()
    {
        connection.reset(PQsetdbLogin(
                               dbhost.c_str(),
                               dbport.c_str(),
                               nullptr,
                               nullptr,
                               dbname.c_str(),
                               dbuser.c_str(),
                               dbpass.c_str()),
                           &PQfinish);

        if (PQstatus(connection.get()) != CONNECTION_OK)
        {
            throw std::runtime_error(PQerrorMessage(connection.get()));
        }
    }

    int exec(const char *query)
    {
        PGresult *res = PQexec(connection.get(), query);
        if (PQresultStatus(res) == PGRES_FATAL_ERROR){
            throw std::runtime_error(PQerrorMessage(connection.get()));
        }
        int result_n = PQntuples(res);
        PQclear(res);
        return result_n;
    }
};

int main()
{
    PGConnection* conn = new PGConnection();
    std::cout << conn->exec("SELECT 1+1");
    delete conn;
}
```
������� ����� PGconnection. ���������� private �������� ������: dbhost, dbport, dbname, dbuser, dbpass - ��������� ��� �������� �����������. ��� ������������ ���� ��� � ������ �����������. ��������  ������� private �������� connection - ��� ������� ����������� ����� ����������.  
� ������������ PGConnection() �� ������������� ����������. ����������, ���������� ������������� ������� PGsetdbLogin(), ������� �� �������� ��������� �����������. � ����� ��������� connection �������� ��������� ���������� ������� PGsetdbLogin() � ������� ���������� PQfinish() ��� ������������ ������.
�� ���������� �� ���������� �� ��������� ������ (������� PQstatus()) �, ���� ���������� �� ����������� �������� ����������. 

�����, � ������ PGconnection ���������� ���� ����� exec(). ���� ����� ��������� ������ (������� PQexec()) � ���������� ���������� ����� � ���������� ������� (������� PQntuples()). �������� �������� ��� ����� ������� ���������� ���������� ������� �������� �������� PQclear(res). � ������������ � libpq ������������ ������� ������������ ������ �� ��� ������� ������.  
��� �������� ������������� �������� �� ������� ������ �������� ������� PQresultStatus. � ������ �������� ��������� ��� ��������� PGRES_FATAL_ERROR � �� ������� ���������� ��������� ������� PQerrorMessage().

� �������  main() ������� ��������� ������ PQConnection � ��������� ������.
� ���������� ���������� ��������� ������� 1. ������ "SELECT 1+1" ��������� ���� ������ � ����� �������� � ��������� 2.  
�������� ��������, ��� � ����� main() �������� delete conn, ����� �������� ������� ���������� ���������� new.

# ����������� - ������������ �� ������
������ �������� ������ PGConnection ������� � ��������� ������������ ���� pg_connection.h � ��� �� �������� ��� � app.cpp.  
pg_connection.h  
```c++
#ifndef PG_CONNECTION_H
#define PG_CONNECTION_H

#include <string>
#include <mutex>
#include <libpq-fe.h>

class PGConnection
{
private:
    std::string dbhost = "localhost";
    std::string dbport = "5433";
    std::string dbname = "testdb";
    std::string dbuser = "postgres";
    std::string dbpass = "1";

    std::shared_ptr<PGconn> connection; //������ ���������� �������������� �������� PGconn

public:
    PGConnection();
    int exec(const char *query);
};

#endif //PG_CONNECTION_H
```  
���������� ������ PGConnection ��������� � ���� pg_connection.cpp � ��� �� �������� ��� � app.cpp.  
pg_connection.cpp  
```c++
#include "pg_connection.h"
#include <string>
#include <mutex>
#include <libpq-fe.h>

PGConnection::PGConnection()
{
    connection.reset(PQsetdbLogin(
                         dbhost.c_str(),
                         dbport.c_str(),
                         nullptr,
                         nullptr,
                         dbname.c_str(),
                         dbuser.c_str(),
                         dbpass.c_str()),
                     &PQfinish);

    if (PQstatus(connection.get()) != CONNECTION_OK)
    {
        throw std::runtime_error(PQerrorMessage(connection.get()));
    }
};

int PGConnection::exec(const char *query)
{
    PGresult *res = PQexec(connection.get(), query);
    if (PQresultStatus(res) == PGRES_FATAL_ERROR){
        throw std::runtime_error(PQerrorMessage(connection.get()));
    }
    int result_n = PQntuples(res);
    PQclear(res);
    return result_n;
}
```

� ����� app.cpp ������� ��������� ������.  
app.cpp  
```c++
#include "pg_connection.h"
```

����� ���� ��������� ����������������, ������� � task.json � ������ "args" ��� ���� ������ "pg_connection.cpp":  
task.json  
![task.json](img/refactor_pg_connection.png))
  
# ����������� ������ PGConnection
������ ������ ��������� ����������� �� ���� � ���������������� ����. � ������� ������� ����������� PQsetdblogin() �� ������ ������� ����������� PQconnectdbParams(). ��� ������� ��� �� ������������� ������� ��� ��� ��������� ������ ���������� �����������.

��-������ �������� ��������, ��� �������� ���������� �����������. �� ����� ������ ��������� ����������� �� ����� � ���������� � ��� ���������. �����, ��� �������� ������� �����������, �� ����� ����� ��������� �� ���� ���������. ��� ���� � ��� ���� �����������, �� ����� �� ����� ������ ���������� ���������������, ����� ��������� ��������� �����������. � ���� pg_connection.h ������� ��������� Connection_Params.  
pg_connection.h  
```c++
...
#include <libpq-fe.h>

struct Connection_Params {
private:    
    std::vector<const char* > keys_ptr;
    std::vector<const char*> values_ptr;
public:
    Connection_Params();  
    ~Connection_Params();    
    void add_key(const char* key);
    void add_value(const char* key);
    std::vector<const char*> keys();
    std::vector<const char*> values();
};

class PGConnection
...
```  
��������� Connection_Params �������� �����������, �� ��� ����������� �������.
� ���� pg_connection.cpp ������� ���������� ���� ���������.  
pg_connection.cpp  
```c++
...
Connection_Params::Connection_Params()
    : keys_ptr(std::vector<const char*>())
    , values_ptr(std::vector<const char*>())
{
}

void add(std::vector<const char*> *vec, const char* val) {
    if (val != NULL) {
        char* tmp = new char[std::strlen(val) + 1];
        std::strcpy(tmp, val);
        (*vec).push_back(tmp);
    } else {
        (*vec).push_back(nullptr);
    }
}

void Connection_Params::add_key(const char* key)
{
    add(&keys_ptr, key);
}

void Connection_Params::add_value(const char* value)
{
    add(&values_ptr, value);
}

std::vector<const char*> Connection_Params::keys()
{
    return keys_ptr;
}

std::vector<const char*> Connection_Params::values()
{
    return values_ptr;
}

Connection_Params::~Connection_Params()
{
    std::vector<const char*>::iterator itr = keys_ptr.begin();
    while (itr != keys_ptr.end()) {
        delete *itr;
        ++itr;
    }
    keys_ptr.clear();
    itr = values_ptr.begin();
    while (itr != values_ptr.end()) {
        delete *itr;
        ++itr;
    }
    values_ptr.clear();
}
```
����� ����������� Connection_Params ��������� ������ ������������� ��������: keys_ptr � values_ptr. ��� ������� �������� ��������� �� ����������� �������� char.
������ add_key() � add_value() ���������, ������� ���������� ������� � ��������� ������� add(), ���� ����� ���������� �� ������� ������, �� � ��������� �� ������ keys_ptr ��� values_ptr.
� ������� add() �� �������� ����� ��� ������ � ����, � ���������� tmp ����������� ������ �� ��� �����. ����� �������� ������������ ������ � ��� ����� � ���� � �������� ���������� tmp (�.�. ��������� �� ������) � ������.
������ ������ �������� ��������� � ������ ������������ ������ �� ������? ������-���, ������ ������ Connection_Params ����� �� ����� ���������. ������ add_key() � add_value() ����� ���������� �� ����� ������ ������� � �� �� ����� ��� ����� ����� ���� ��������� ����������, ������ �� ������� ���������� � add_key() ��� add_value(). ��������� ����� � ���� �������� ��� �� �������������� ������� ��������� Connection_Params, �� � ��������� ����� ���������� �� ������������ ����� �����.  
���������� ~Connection_Params � ������� ��������� �� ��������� �������� keys_ptr � values_ptr ����������� ������ �� ���������� ��������� ���������� � ���� � ������� ���� �������.  
��� ����� �� �������� ������� keys_ptr � values_ptr ��������� ��� �������, ����� ��������� ������ � ��� �����. �������� ������ �� ��� ������� ����� � ������� ����������� ������� keys() � values().


� ���� pg_connection.h ����� ������� ��� ������ ��� ������ ���� ��������� �� �����.  
pg_connection.h  
```c++
...
class PGConnection
{
private:
    ...
    std::shared_ptr<std::string> load_params_to_str();
    std::shared_ptr<Connection_Params> parse_params_from_str(const char* str);
public:
    ...
};
``` 
������ �� ����� � ���������� ��������� ����������� ����� � ���������� ����������.
� ����� pg_connection.cpp ������� ���������� ����������, ������� ������� ����� ���������� �� Connection_Papams. � ���� ����������� � ����� ������� ������ �� ��������� �����������. ����� ������� ��������� CONFIG_PATH - ���� � ����� ������������ (���� ������������ ����� ������� � ��������� �����, �� ����� ������ � ������ ������ �������). �� ������ ����������� �������� ����� � �������� �����. ��� ����� �� ��������, �� �������� ����� ������ �� ���������� filesystem, fstream � iostream.  
pg_connection.cpp  
```c++
...
#include <libpq-fe.h>

#include <filesystem>
#include <fstream>
#include <iostream>

const std::string CONFIG_PATH = "../config/config.json";
std::shared_ptr<Connection_Params> connection_params = nullptr;

PGConnection::PGConnection()
...
```
  
� ����� pg_connection.cpp ������� ���������� ������ load_params_to_str(). ����� �� ��������� ���������� �� ���� ������� std::filesystem::exists(). ����� ������� ����� std::fstream config_file � ������ �� ���� ������ params_line � ��������� ������ � ���� �������� ������ params_str.  
pg_connection.cpp  
```c++
...
int PGConnection::exec(const char *query)
{
    ...
};

std::shared_ptr<std::string> PGConnection::load_params_to_str()
{
    std::string params_str = "";
    std::string params_line;
    try {
        if (std::filesystem::exists(CONFIG_PATH)) {
            std::fstream config_file(CONFIG_PATH);
            while (std::getline(config_file, params_line)) {
                params_str += params_line;
            }
            config_file.close();
        } else {
            throw new std::exception("No config file. Add config file config/config.json");
        }
    } catch (std::exception e) {
        std::cout << e.what();
    };
    return std::make_shared<std::string>(params_str);
}
```  
  
����� load_params_to_str() ������ ������ ����, �� ������ ���� ������������� ������ � json. ��� ����� ������������� ����������� [rapidjson](https://github.com/Tencent/rapidjson/). ��������� ����������� �� ���� ���������. ������� ����� �� ����� ������ � ����� �������� "_include" (������������ ������� ����� ����� ������ ���� ������� ������ �����). ������������� ����������� � �������� ����� "include/rapidjson" � ����� "_include".  
��������� ���� c_cpp_properties.json � ����� .vscode. ���� ��� ������ �����, �� �������� ctrl+shift+P � �������� "c/c++: Edit Configurations (Json)". � ������ "includePath" �������� ������ "${workspaceFolder}/../_include/**".  
c_cpp_properties.json  
![c_cpp_properties.json](img/add_includePath.png)

� ����� pg_connection.cpp ��������� ������ �� ���������� rapidjson � ��������� ���������� ������ parse_params_from_str(), �������� �������� ������, �������������� �� ����������������� �����. ���� �����, ��� ���, ��������� ������ � ���������� ����� ��������� �� Connection_Params.  
pg_connection.cpp  
```c++
...
#include <rapidjson/document.h>
...
std::shared_ptr<std::string> PGConnection::load_params_to_str()
{
    ...
}

std::shared_ptr<Connection_Params> PGConnection::parse_params_from_str(const char* str)
{
    std::shared_ptr<Connection_Params> conn_params = nullptr;
    rapidjson::Document d;
    d.Parse(str);
    if (d.IsObject() && d.HasMember("connection_params")) {
        rapidjson::Value& d_params = d["connection_params"];
        if (d_params.IsObject()) {
            conn_params = std::make_shared<Connection_Params>();
            rapidjson::Value::ConstMemberIterator iter = d_params.MemberBegin();
            while (iter != d_params.MemberEnd()) {
                conn_params->add_key(iter->name.GetString());
                conn_params->add_value(iter->value.IsNull() ? "" : iter->value.GetString());
                ++iter;
            }
            conn_params->add_key(NULL);
        }
        else{
            printf("� ���������������� ����� ������� ����� ������ \"connection_params\"");
        }
    }
    else{
        printf("� ���������������� ����� ����������� ������ \"connection_params\"");
    }
    return conn_params;
}

```
����� �� ������� ������ rapidjson::Document d, ������� � ��������� ������������ ������. ������ �������� ���������, ��� ������ � ���������������� ����� �������� �������� � �������� ���� "connection_params". ����� �� ������ ���� "connection_params" � ���������� d_params �, ����� �������� ��� d_params - ��� ������, � ������� ��������� ���������� ��� ����� ������� d_params.  
� ������� ������� ��������� iter->name.GetString() � iter->value.GetString() �������� �������������� �������� ��������� � ��� ��������, � �������� � �������������� ������� ��������� Connection_Params � ������� ������� conn_params->add_key() � conn_params->add_value().  
� ����� � ������ ������ ��������� �������� NULL � ������� ������ conn_params->add_key(). ��� ����� ��� ������� PQconnectdbParams(), ��� ������ ������ ������, ���� �� �������� NULL. ��� �������� � [������������ � ���� �������](https://postgrespro.ru/docs/postgresql/9.6/libpq-connect).
  
������ ��� ����� ������� ����� ����� [�����]().

