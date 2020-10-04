```c++
#include <vector>
#include <memory>
#include <iostream>

struct Connection_Params {
    std::vector<const char*> keys;
    std::vector<const char*> values;
    ~Connection_Params(){        
        keys.clear();
        values.clear();
        std::cout << "delete connection_params" << std::endl;
    }
};

std::vector<char> create_vector(){
    std::vector<char> v;
    return v;
}

std::shared_ptr<Connection_Params> create_shared_connection_params(){
    std::vector<const char*> k { "1111", "kbhjk" };
    std::vector<const char*> v { "2222", "asdfasd" };

    std::shared_ptr<Connection_Params> c = std::make_shared<Connection_Params>();
    c.get()->keys = k;
    c.get()->values = v;
    return c;
}

Connection_Params create_connection_params()
{
    std::vector<const char*> k { "1111", "kbhjk" };
    std::vector<const char*> v { "2222", "asdfasd" };

    Connection_Params c;
    c.keys = k;
    c.values = v;
    return c;
}

int main(){
    std::cout << "create_shared_connection_params" << std::endl;
    std::shared_ptr<Connection_Params> sh_c = create_shared_connection_params();
    std::cout << sh_c.get()->keys[0] << " " << sh_c.get()->values[0] << std::endl;
    std::cout << "count connection_params:" << sh_c.use_count() << std::endl;

    // std::cout << "create_connection_params" << std::endl;
    // Connection_Papams c = create_connection_params();
    // std::cout << c.keys[0] << " " << c.values[0] << std::endl;
}
```