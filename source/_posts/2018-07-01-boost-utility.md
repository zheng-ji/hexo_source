---
layout: post
title: boost-utility
date: 2018-07-01 13:50:03
categories: Program
---

`BOOST` 库的很多标准已经被纳入 C++11, 这也说明了 BOOST 库设计得到标准的重视。工作中需要用到 BOOST 的特性来加速开发。以下举例常用到的 utility

* BOOST_SCOPE_EXIT

BOOST_SCOPE_EXIT（）里面可以传入多个参数, 其作用相当于回调函数， 在作用域结束之后程序会自动调用 BOOST_SCOPE_EXIT到BOOST_SCOPE_EXIT_END之间的代码。这个在异常捕捉，或者日志打印的时候会非常有用，节约代码。

举个例子

```c++
void func (int ans) 
{
    bool flag = false;
    BOOST_SCOPE_EXIT(&flag) {
        if (!flag)
            std::cout << "false" <<std::endl;
        else
            std::cout << "true" << std::endl;
    } BOOST_SCOPE_EXIT_END
    flag = ((ans == 1)? true:false);
}
```

* boost::split

文本处理，切割字符串的利器

```
#include <boost/algorithm/string/split.hpp>
int main( int argc, char** argv )
{
    string source = "how to live in the world!";
    vector<string> destination;
    boost::split( destination, source, boost::is_any_of( " ,!" ), boost::token_compress_on );
    vector<string>::iterator it ;
    for (it= destination.begin(); it != destination.end(); ++ it)
        cout << *it << endl;
    return 0;
}
```

* boost::regex

正则匹配

```
boost::regex_match(str, boost::regex("^1[3|4|5|8][0-9]{9}$")) 
```

* BOOST 读写锁

```
#include "boost/thread/shared_mutex.hpp"
boost::shared_mutex m_rwlock;
boost::shared_lock<boost::shared_mutex> guard(m_rwlock);读锁
boost::unique_lock<boost::shared_mutex> guard(m_rwlock);写锁

#include "boost/thread/locks.hpp"
#include "boost/thread/shared_mutex.hpp"
using namespace std;

//线程安全
class NameMap{

    private:
        boost::shared_mutex m_rwlock;
        map<uint32_t, string> m_map;
    public:
        bool find(uint32_t uin, string & nickname) {
            boost::shared_lock<boost::shared_mutex> guard(m_rwlock);
            auto it = m_map.find(uin);
            if(it == m_map.end()){
                return false;
            }   
            nickname = it->second;
            return true;
        }   

        void insert(uint32_t uin, const string & nickname) {
            boost::unique_lock<boost::shared_mutex> guard(m_rwlock);
            m_map[uin]=nickname;
        }   

        size_t size(){
            boost::shared_lock<boost::shared_mutex> guard(m_rwlock);
            return m_map.size();
        }   
};
```



