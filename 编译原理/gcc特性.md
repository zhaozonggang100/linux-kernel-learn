# 1、特殊符号

### 1、#

字符串转换

demo：

```
#define FUNC(a) #a   // 将任意a代表的字符（长度不限）转换为字符串

FUNC(123)---> 123
FUNC(123abc)--->123abc
```

### 2、##

连接

demo：

```
#define FUNC(A,B) A##B // 将任意A和B连接组成字符串

FUNC(123,456)--->123456
FUNC(123,abc)--->123abc
```

