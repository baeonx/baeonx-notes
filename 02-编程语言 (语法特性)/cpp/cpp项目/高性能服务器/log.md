# log篇的解析

# log 的架构

```
┌─────────────────────────────────────────────────────────────┐
│                       应用程序                               │
└────────────────────┬────────────────────────────────────────┘
                     │ 创建LogEvent
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  LogEventWrap (RAII)                                        │
│  • 构造函数：创建LogEvent                                     │
│  • 析构函数：自动调用Logger::log()                             │
└────────────────────┬────────────────────────────────────────┘
                     │ 调用log()
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  Logger (日志器)                                             │
│  • 管理日志级别过滤                                           │
│  • 持有Formatter和Appender列表                               │
│  • 遍历所有Appender写入日志                                   │
└───────┬──────────────────────────────┬──────────────────────┘
        │ 使用                          │ 使用
        ▼                              ▼
┌──────────────────┐           ┌──────────────────┐
│ LogFormatter     │           │ LogAppender      │
│ • 解析格式模板     │           │ • 控制输出目标     │
│ • 格式化日志内容   │           │ • 持有Formatter   │
└──────────────────┘           └────────┬─────────┘
                                       │ 继承
                          ┌────────────┴────────────┐
                          ▼                         ▼
              ┌──────────────────┐        ┌──────────────────┐
              │StdoutLogAppender │        │FileLogAppender   │
              │• 输出到控制台      │        │• 输出到文件        │
              └──────────────────┘        └──────────────────┘
```



## 1 LogEvent

1. **logevent**定义了最基本的事件类型，以及基本的事件信息

2. 流式存储使得**logEvent**可以写入字符串

[stringstream&妙用](./补充说明/流式写法.md)

[const](./补充说明/const.md)

```cpp

class LogEvent {
public:
    typedef std::shared_ptr<LogEvent> ptr;
    
    LogEvent(std::shared_ptr<Logger> logger, LogLevel::Level level
            ,const char* file, int32_t line, uint32_t elapse
            ,uint32_t thread_id, uint32_t fiber_id, uint64_t time
            ,const std::string& thread_name);
    
    const char* getFile() const { return m_file; }
    int32_t getLine() const { return m_line; }
    uint32_t getElapse() const { return m_elapse; }
    uint32_t getThreadId() const { return m_threadId; }
    uint64_t getTime() const { return m_time; }
    std::string getContent() const { return m_ss.str(); }
    std::stringstream& getSS() { return m_ss; }
    LogLevel::Level getLevel() const { return m_level;}
     std::shared_ptr<Logger> getLogger() const { return m_logger; }
    
private:
    const char* m_file;      // 文件名
    int32_t m_line;          // 行号
    uint32_t m_elapse;       // 耗时
    uint32_t m_threadId;     // 线程ID
    uint64_t m_time;         // 时间戳
    std::stringstream m_ss;  // 日志内容流
    std::shared_ptr<Logger> m_logger;  // 所属日志器
    LogLevel::Level m_level; // 日志等级
};



// 示例1：创建LogEvent的基本用法
void basicExample() {
    // 获取当前线程信息
    uint32_t thread_id = std::hash<std::thread::id>{}(std::this_thread::get_id());
    uint32_t fiber_id = 0;  // 协程ID，简单示例用0
    uint64_t time = time(nullptr);
    std::string thread_name = "main_thread";
    
    // 创建LogEvent对象
    LogEvent::ptr event(new LogEvent(
        logger,                     // 所属日志器
        LogLevel::INFO,             // 日志级别
        __FILE__,                   // 文件名
        __LINE__,                   // 行号
        0,                          // 耗时(ms)
        thread_id,                  // 线程ID
        fiber_id,                   // 协程ID
        time,                       // 时间戳
        thread_name                 // 线程名
    ));
    
    // 向日志流中写入内容
    // !!! stringstream&妙用
    event->getSS() << "用户登录成功，用户ID: " << 10001;
    
    // 获取日志内容
    std::cout << "文件: " << event->getFile() << std::endl;
    std::cout << "行号: " << event->getLine() << std::endl;
    std::cout << "线程ID: " << event->getThreadId() << std::endl;
    std::cout << "日志级别: " << event->getLevel() << std::endl;
    std::cout << "日志内容: " << event->getContent() << std::endl;
}
```





## 2 loggerLevel

[X-Macro技术](./补充说明/X-Macro技术.md)

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


// 将int转化为string
const char* LogLevel::ToString(Level level) {
    switch(level) {
        // 使用X-marco技术
        #define X(name) case name: return #name;
            X(Level::DEBUG)
            X(Level::INFO)
            X(Level::WARN)
            X(Level::ERROR)
            X(Level::FATAL)
        #undef X
            default:
                return "UNKNOW";
    }
}

// 将string转化为Level枚举类型即数字类型
LogLevel::Level LogLevel::FromString(const std::string& str) {
    // 转换为大写
    std::string upper_str = str;
    for(char& c : upper_str) {
        c = std::toupper(c);
    }

    #define X(name, str) if(upper_str == str) return Level::name;
        X(DEBUG, "DEBUG") 
        X(INFO,  "INFO")  
        X(WARN,  "WARN")  
        X(ERROR, "ERROR") 
        X(FATAL, "FATAL") 
        X(UNKNOW, "UNKNOW")
    #undef X
    
    // if(upper_str == "DEBUG") return Level::DEBUG;
    // if(upper_str == "INFO") return Level::INFO;
    // if(upper_str == "WARN") return Level::WARN;
    // if(upper_str == "ERROR") return Level::ERROR;
    // if(upper_str == "FATAL") return Level::FATAL;
    return Level::UNKNOW;
}



// 测试用例
int main() {
    std::cout << "\n========== 实用示例：日志级别过滤 ==========\n" << std::endl;
    // 设置当前日志级别
    LogLevel::Level current_level = LogLevel::INFO;
    
    //将枚举类型转换为字符串类型
    std::cout << "当前日志级别: " << LogLevel::ToString(current_level) << std::endl;
    
    // 模拟日志消息
    struct LogMessage {
        LogLevel::Level level;
        const char* message;
    };
    
    std::vector<LogMessage> messages = {
        {LogLevel::DEBUG, "这是调试信息"},
        {LogLevel::INFO, "用户登录成功"},
        {LogLevel::WARN, "磁盘空间不足"},
        {LogLevel::ERROR, "数据库连接失败"},
        {LogLevel::FATAL, "系统崩溃"}
    };
    
    std::cout << "\n根据日志级别过滤输出:" << std::endl;
    for (const auto& msg : messages) {
        // 只输出级别 >= 当前级别的日志
        if (msg.level >= current_level) {
            std::cout << "  [" << LogLevel::ToString(msg.level) << "] " 
                      << msg.message << std::endl;
        }
    }
    
    // 演示从配置字符串设置级别
    std::string config_level = "WARN";
    LogLevel::Level new_level = LogLevel::FromString(config_level);
    std::cout << "\n从配置读取级别: \"" << config_level << "\" -> " 
              << LogLevel::ToString(new_level) << std::endl;
}
```





## 3 Logger

1. 每个**logEvent**都要绑定一个日志器

```cpp
// Logger类 
class Logger {
public:
    typedef std::shared_ptr<Logger> ptr;
    
    // 为日志器命名
    Logger(const std::string& name = "root")
        : m_name(name)
        , m_level(LogLevel::DEBUG) {
    }
    
    // 核心日志方法，打印日志，
    // log()
    void log(LogLevel::Level level, LogEvent::ptr event) {
        if(level >= m_level) {
            std::cout << "[" << LogLevel::ToString(level) << "]" 
                      << "[" << m_name << "]"
                      << event->getContent() 
                      << std::endl;
        }
    }
    
    // 便捷方法，
    void debug(LogEvent::ptr event) {
        log(LogLevel::DEBUG, event);
    }
    
    void info(LogEvent::ptr event) {
        log(LogLevel::INFO, event);
    }
    
    void warn(LogEvent::ptr event) {
        log(LogLevel::WARN, event);
    }
    
    void error(LogEvent::ptr event) {
        log(LogLevel::ERROR, event);
    }
    
    void fatal(LogEvent::ptr event) {
        log(LogLevel::FATAL, event);
    }
    
    // Setter/Getter
    void setLevel(LogLevel::Level level) { m_level = level; }
    LogLevel::Level getLevel() const { return m_level; }
    const std::string& getName() const { return m_name; }
    
private:
    std::string m_name;        // 日志器名称
    LogLevel::Level m_level;    // 日志级别
};


```



## 4 LogEventWrap 的作用和配合机制

1. **logEventWarp**实现过程中的小细节

[巧用匿名对象析构函数进行流式输出](./补充说明/巧用匿名对象析构函数进行流式输出.md)



```cpp
// 没有 LogEventWrap 时的写法：
// 需要手动记录日志
LogEvent::ptr event(new LogEvent(logger, LogLevel::INFO, __FILE__, __LINE__, ...));
event->getSS() << "用户登录成功" << "，用户名：" << username;
logger->log(LogLevel::INFO, event);  // 手动调用

// 或者更糟：忘记调用log()
LogEvent::ptr event(new LogEvent(...));
event->getSS() << "这是一条日志";
// 忘记调用logger->log()，日志没打印出来，日志丢失！
```



```cpp
// LogEventWrap类
class LogEventWrap {
public:
    LogEventWrap(LogEvent::ptr e)
        : m_event(e) {
    }
    
    ~LogEventWrap();
    
    LogEvent::ptr getEvent() const { return m_event; }
    
    // 
    std::stringstream& getSS() {
        return m_event->getSS();
    }
    
private:
    LogEvent::ptr m_event;
};
```



```cpp

LogEventWrap wrap(LogEvent::ptr(new LogEvent(logger, LogLevel::INFO, ...)));
wrap.getSS() << "用户登录" << " ID: " << 1001;
// 这里通过getSS()获取stringstream，不断追加内容

```



## 5 logEvent_logger_logEventWrap结合

```cpp
#include <iostream>
#include <memory>
#include <thread>
#include <chrono>
#include <vector>
#include <iomanip>

namespace Daeon {

// 简单的Logger实现
class Logger {
public:
    typedef std::shared_ptr<Logger> ptr;
    std::string m_name;
    
    Logger(const std::string& name = "root") : m_name(name) {}
    
    void log(LogLevel::Level level, std::shared_ptr<LogEvent> event) {
        std::cout << "[" << m_name << "][" 
                  << LogLevel::ToString(level) << "] "
                  << event->getContent() 
                  << " (" << event->getFile() << ":" << event->getLine() << ")"
                  << std::endl;
    }
};

// LogEventWrap类的简单实现（如果头文件中没有定义）
class LogEventWrap {
public:
    LogEventWrap(std::shared_ptr<LogEvent> event) : m_event(event) {}
    ~LogEventWrap() {
        m_event->getLogger()->log(m_event->getLevel(), m_event);
    }
    
    std::stringstream& getSS() { return m_event->getSS(); }
    
private:
    std::shared_ptr<LogEvent> m_event;
};

} // namespace Daeon

// 辅助宏，方便创建日志事件
#define LOG_EVENT(logger, level) \
    Daeon::LogEventWrap(std::make_shared<Daeon::LogEvent>( \
        logger, level, __FILE__, __LINE__, \
        0, std::hash<std::thread::id>{}(std::this_thread::get_id()), \
        0, time(nullptr), "main_thread"))

int main() {
    using namespace Daeon;
    
    std::cout << "=========================================" << std::endl;
    std::cout << "    LogLevel, LogEvent, LogEventWrap 测试" << std::endl;
    std::cout << "=========================================" << std::endl;
    
    // 创建logger
    auto logger = std::make_shared<Logger>("test_logger");
    
    // 获取线程信息
    uint32_t thread_id = std::hash<std::thread::id>{}(std::this_thread::get_id());
    uint32_t fiber_id = 0;
    uint64_t now = time(nullptr);
    std::string thread_name = "main";
    
    std::cout << "\n========== 1. LogLevel 测试 ==========" << std::endl;
    
    // 测试ToString
    std::cout << "\n--- ToString测试 ---" << std::endl;
    std::vector<LogLevel::Level> levels = {
        LogLevel::DEBUG, LogLevel::INFO, LogLevel::WARN, 
        LogLevel::ERROR, LogLevel::FATAL, LogLevel::UNKNOW
    };
    
    for(auto level : levels) {
        std::cout << "枚举值 " << std::setw(2) << static_cast<int>(level) 
                  << " -> 字符串: " << LogLevel::ToString(level) << std::endl;
    }
    
    // 测试FromString
    std::cout << "\n--- FromString测试 ---" << std::endl;
    std::vector<std::string> test_strings = {
        "DEBUG", "info", "Warn", "ERROR", "fatal", "UNKNOW", "invalid"
    };
    
    for(const auto& str : test_strings) {
        LogLevel::Level level = LogLevel::FromString(str);
        std::cout << "输入: \"" << std::setw(8) << str << "\" -> 级别: " 
                  << LogLevel::ToString(level) << std::endl;
    }
    
    
    
    
    
    std::cout << "\n========== 2. LogEvent 直接使用测试 ==========" << std::endl;
    
    // 创建几个LogEvent实例
    auto event1 = std::make_shared<LogEvent>(
        logger, LogLevel::DEBUG, __FILE__, __LINE__, 10,
        thread_id, fiber_id, now, thread_name
    );
    event1->getSS() << "用户登录尝试，用户名: admin";
    
    auto event2 = std::make_shared<LogEvent>(
        logger, LogLevel::INFO, __FILE__, __LINE__, 25,
        thread_id, fiber_id, now, thread_name
    );
    event2->getSS() << "用户登录成功，用户ID: " << 10001 << ", 角色: administrator";
    
    auto event3 = std::make_shared<LogEvent>(
        logger, LogLevel::WARN, __FILE__, __LINE__, 150,
        thread_id, fiber_id, now, thread_name
    );
    event3->getSS() << "数据库查询缓慢，耗时: " << 150 << "ms";
    
    auto event4 = std::make_shared<LogEvent>(
        logger, LogLevel::ERROR, __FILE__, __LINE__, 0,
        thread_id, fiber_id, now, thread_name
    );
    event4->getSS() << "文件打开失败: " << "config.ini (No such file)";
    
    // 手动记录日志
    std::cout << "手动记录4条日志:" << std::endl;
    logger->log(event1->getLevel(), event1);
    logger->log(event2->getLevel(), event2);
    logger->log(event3->getLevel(), event3);
    logger->log(event4->getLevel(), event4);
    
    std::cout << "\n--- LogEvent属性访问测试 ---" << std::endl;
    std::cout << "事件1 - 文件: " << event1->getFile() << std::endl;
    std::cout << "事件1 - 行号: " << event1->getLine() << std::endl;
    std::cout << "事件1 - 耗时: " << event1->getElapse() << "ms" << std::endl;
    std::cout << "事件1 - 线程ID: " << event1->getThreadId() << std::endl;
    std::cout << "事件1 - 时间戳: " << event1->getTime() << std::endl;
    std::cout << "事件1 - 级别: " << LogLevel::ToString(event1->getLevel()) << std::endl;
    std::cout << "事件1 - 内容: " << event1->getContent() << std::endl;
    
    
    
    
    std::cout << "\n========== 3. LogEventWrap 自动日志测试 ==========" << std::endl;
    
    std::cout << "开始作用域1 (DEBUG级别)..." << std::endl;
    {
        LogEventWrap wrap(std::make_shared<LogEvent>(
            logger, LogLevel::DEBUG, __FILE__, __LINE__, 5,
            thread_id, fiber_id, time(nullptr), thread_name
        ));
        wrap.getSS() << "进入函数 processData()，参数size=" << 1024;
        std::cout << "  作用域1内: 日志已创建，但还未输出" << std::endl;
        // 离开作用域时自动调用析构，输出日志
    }
    std::cout << "离开作用域1，日志已自动输出" << std::endl;
    
    std::cout << "\n开始作用域2 (ERROR级别)..." << std::endl;
    {
        LogEventWrap wrap(std::make_shared<LogEvent>(
            logger, LogLevel::ERROR, __FILE__, __LINE__, 0,
            thread_id, fiber_id, time(nullptr), thread_name
        ));
        wrap.getSS() << "网络连接失败，错误码: " << -1 << ", 重试次数: " << 3;
        std::cout << "  作用域2内: 准备抛出异常..." << std::endl;
        // 即使发生异常，离开作用域时析构函数也会被调用
    }
    std::cout << "离开作用域2，错误日志已自动输出" << std::endl;
    
    
    
    
    
    std::cout << "\n========== 4. 使用宏简化日志创建 ==========" << std::endl;
    
    std::cout << "使用LOG_EVENT宏创建日志:" << std::endl;
    {
        auto wrap = LOG_EVENT(logger, LogLevel::INFO);
        wrap.getSS() << "这是一条使用宏创建的日志";
    }
    
    {
        auto wrap = LOG_EVENT(logger, LogLevel::WARN);
        wrap.getSS() << "警告：内存使用率超过80%，当前: " << 85 << "%";
    }
    
    
    
    
    std::cout << "\n========== 5. 综合应用示例 ==========" << std::endl;
    
    // 模拟一个函数调用链
    auto func1 = [&](int x) {
        LogEventWrap wrap(LOG_EVENT(logger, LogLevel::DEBUG));
        wrap.getSS() << "进入func1，x=" << x;
        
        if(x > 100) {
            LogEventWrap warn_wrap(LOG_EVENT(logger, LogLevel::WARN));
            warn_wrap.getSS() << "func1: 参数x=" << x << "过大，可能影响性能";
        }
        
        // 模拟处理
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    };
    
    auto func2 = [&](const std::string& name) {
        LogEventWrap wrap(LOG_EVENT(logger, LogLevel::INFO));
        wrap.getSS() << "处理用户: " << name;
        
        func1(name.length());
        
        if(name.empty()) {
            LogEventWrap error_wrap(LOG_EVENT(logger, LogLevel::ERROR));
            error_wrap.getSS() << "用户名为空";
        }
    };
    
    std::cout << "执行函数调用链测试:" << std::endl;
    func2("Alice");
    std::cout << "---" << std::endl;
    func2("Bob");
    std::cout << "---" << std::endl;
    func2("");  // 触发错误
    
    
    
    
    
    std::cout << "\n========== 6. 日志级别过滤测试 ==========" << std::endl;
    
    // 模拟日志级别过滤器
    LogLevel::Level min_level = LogLevel::WARN;  // 只记录WARN及以上级别
    
    std::cout << "当前最低日志级别: " << LogLevel::ToString(min_level) << std::endl;
    std::cout << "尝试记录不同级别的日志:" << std::endl;
    
    std::vector<LogLevel::Level> test_levels = {
        LogLevel::DEBUG, LogLevel::INFO, LogLevel::WARN, 
        LogLevel::ERROR, LogLevel::FATAL
    };
    
    for(auto level : test_levels) {
        auto event = std::make_shared<LogEvent>(
            logger, level, __FILE__, __LINE__, 0,
            thread_id, fiber_id, time(nullptr), thread_name
        );
        event->getSS() << "这是一条" << LogLevel::ToString(level) << "级别的日志";
        
        if(level >= min_level) {
            std::cout << "  ✓ 记录: ";
            logger->log(level, event);
        } else {
            std::cout << "  ✗ 过滤: " << LogLevel::ToString(level) << "级别日志被忽略" << std::endl;
        }
    }
    
    std::cout << "\n=========================================" << std::endl;
    std::cout << "                测试完成" << std::endl;
    std::cout << "=========================================" << std::endl;
    
    return 0;
}
```

























