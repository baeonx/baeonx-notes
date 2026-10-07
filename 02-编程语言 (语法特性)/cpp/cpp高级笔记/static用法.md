# static用法

## 1 类的方法调用

```cpp
class LogLevel {
public:
    enum Level {
        UNKNOW = 0,
        DEBUG = 1,
        INFO = 2,
        WARN = 3,
        ERROR = 4,
        FATAL = 5
    };
    
    static const char* ToString(Level level);
    static Level FromString(const std::string& str);
};


// 在此处调用ToString不用实例化LogLevel
std::cout << "DEBUG -> " << LogLevel::ToString(LogLevel::DEBUG) << std::endl;
```





































