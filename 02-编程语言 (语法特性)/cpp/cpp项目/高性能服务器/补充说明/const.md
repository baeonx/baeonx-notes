# const

## 1 前置const vs 后置const的区别

### 1.1 **后置const**（你的代码中的情况）

```cpp
LogLevel::Level getLevel() const { return m_level; }
```

- **修饰的是成员函数**
- 表示这个成员函数**不会修改对象的状态**
- 承诺不会修改类的成员变量（`m_level`）
- 常对象只能调用后置const的成员函数



### 1.2 **前置const**

```cpp
const LogLevel::Level getLevel() { return m_level; }
```

- **修饰的是返回值**
- 表示返回的值是常量，不能被修改
- 与成员函数是否修改对象无关



### 1.3 具体例子

```cpp
class LogLevel {
private:
    Level m_level;
    
public:
    // 情况1：后置const - 函数本身是const的
    Level getLevel() const { 
        // m_level = ...;  // 错误！不能修改成员变量
        return m_level;    // 正确：只读取
    }
    
    // 情况2：前置const - 返回值是const的
    const Level getLevel() { 
        return m_level;    // 可以修改成员变量
    }
    
    // 情况3：同时使用
    const Level getLevel() const {
        return m_level;    // 返回值const + 函数const
    }
};
```

