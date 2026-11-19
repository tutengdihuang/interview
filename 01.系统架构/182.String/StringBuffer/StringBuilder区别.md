# String/StringBuffer/StringBuilder 区别

> **场景定位**：Java 基础技术要点  
> **核心设计要点**：不可变性、线程安全、执行效率、内存优化、GC 压力、使用场景

---

## 🎯 一句话定位

**String/StringBuffer/StringBuilder 区别是什么？**
```
三个都是字符串处理类，但特性完全不同：
- String：不可变，每次修改创建新对象
- StringBuffer：可变，线程安全，性能中等
- StringBuilder：可变，线程不安全，性能最高

用错场景会导致：
- 内存浪费（String 频繁拼接）
- 性能问题（StringBuffer 单线程用）
- 线程安全问题（StringBuilder 多线程用）
```

**生活化比喻**：
- **String**：像石板刻字，每次修改都要换块新石板（不可变）
- **StringBuffer**：像白板写字，可以擦除重写，但一次只能一个人用（线程安全）
- **StringBuilder**：像白板写字，可以擦除重写，多人可以同时用（线程不安全）

**核心特点**：字符串拼接不是想怎么用就怎么用，而是按场景选型、兼顾效率与安全。

---

## 💎 核心观点（5 分钟速览版）

> **快速概览**

### 核心观点（3 个维度）

**1. 不可变性区分**
```
String：
- 底层 final char[] 数组
- 每次修改都创建新对象
- 不可变，线程安全

StringBuffer/StringBuilder：
- 底层 char[] 数组（无 final）
- 直接在原对象修改
- 可变，支持动态扩展

生活化比喻：
String = 石板（刻上去就改不了）
StringBuffer/StringBuilder = 白板（可以擦除重写）
```

**2. 线程安全对比**
```
StringBuffer：
- 方法加 synchronized 锁
- 线程安全
- 多线程共享变量用

StringBuilder：
- 无锁
- 线程不安全
- 单线程优先用

生活化比喻：
StringBuffer = 白板 + 锁（一次只能一个人用）
StringBuilder = 白板（多人可以同时用，但要小心冲突）
```

**3. 执行效率排序**
```
性能对比：
StringBuilder > StringBuffer > String

原因：
- String：每次修改创建新对象，产生大量临时对象
- StringBuffer：加锁保证线程安全，性能中等
- StringBuilder：无锁，性能最高

使用场景：
- 少量赋值：String
- 高频拼接、循环拼接：StringBuilder
- 多线程共享变量：StringBuffer

生活化比喻：
String = 每次换石板（慢）
StringBuffer = 白板 + 锁（中等）
StringBuilder = 白板（快）
```

---

## 🚀 一句话精华

**String 严禁循环内拼接，高频拼接用 StringBuilder，多线程共享用 StringBuffer，少量赋值用 String。**

---

## 详细章节

### 一、问题本质：为什么不能随便用字符串？

#### 1.1 String 的不可变性

```java
// String 的底层实现
public final class String {
    private final char value[];  // final 修饰，不可变
    
    // 每次修改都会创建新对象
    public String concat(String str) {
        char buf[] = new char[count + str.count];
        // ... 复制字符到 buf
        return new String(buf);  // 返回新对象
    }
}

// ❌ 错误示范：循环内使用 String 拼接
String result = "";
for (int i = 0; i < 10000; i++) {
    result = result + i;  // 每次都会创建新对象
}
// 问题：
// - 创建了 10000 个临时 String 对象
// - 内存浪费严重
// - GC 压力大

// ✅ 正确做法：使用 StringBuilder
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i);  // 在原对象上修改
}
String result = sb.toString();
```

#### 1.2 性能对比

```java
// 测试代码
public class StringPerformanceTest {
    
    public static void main(String[] args) {
        int count = 100000;
        
        // 1. String 拼接
        long start = System.currentTimeMillis();
        String result = "";
        for (int i = 0; i < count; i++) {
            result += i;
        }
        System.out.println("String: " + (System.currentTimeMillis() - start) + "ms");
        
        // 2. StringBuffer 拼接
        start = System.currentTimeMillis();
        StringBuffer sb1 = new StringBuffer();
        for (int i = 0; i < count; i++) {
            sb1.append(i);
        }
        System.out.println("StringBuffer: " + (System.currentTimeMillis() - start) + "ms");
        
        // 3. StringBuilder 拼接
        start = System.currentTimeMillis();
        StringBuilder sb2 = new StringBuilder();
        for (int i = 0; i < count; i++) {
            sb2.append(i);
        }
        System.out.println("StringBuilder: " + (System.currentTimeMillis() - start) + "ms");
    }
}

// 典型结果（count = 100000）：
// String: 5000ms+
// StringBuffer: 10ms
// StringBuilder: 5ms
```

---

### 二、三者详细对比

#### 2.1 String：不可变字符串

**底层实现**：

```java
public final class String {
    private final char value[];  // final 修饰，不可变
    private int count;
    private int hash;  // 缓存哈希值
    
    // 构造方法
    public String(String original) {
        this.value = original.value;
        this.count = original.count;
        this.hash = original.hash;
    }
    
    // 拼接操作（创建新对象）
    public String concat(String str) {
        if (str.count == 0) {
            return this;
        }
        char buf[] = new char[count + str.count];
        getChars(0, count, buf, 0);
        str.getChars(0, str.count, buf, count);
        return new String(buf, 0, count + str.count);  // 返回新对象
    }
    
    // 截取操作（创建新对象）
    public String substring(int beginIndex, int endIndex) {
        // ...
        return new String(buf, 0, subCount);  // 返回新对象
    }
}
```

**特点**：

```
1. 不可变性：
   - final char[] 数组
   - 每次修改创建新对象
   - 线程安全（不可变对象天然线程安全）

2. 字符串常量池：
   - 相同内容的字符串共享同一个对象
   - 节省内存空间
   
   String s1 = "hello";
   String s2 = "hello";
   System.out.println(s1 == s2);  // true（同一个对象）

3. 适用场景：
   - 少量赋值
   - 字符串常量
   - 不需要频繁修改的场景
```

**使用示例**：

```java
// ✅ 适用场景：少量赋值
String name = "张三";
String age = "25";

// ✅ 适用场景：字符串常量
public static final String APP_NAME = "MyApp";

// ❌ 不适用：频繁拼接
String result = "";
for (int i = 0; i < 10000; i++) {
    result += i;  // 性能极差
}
```

---

#### 2.2 StringBuffer：线程安全的可变字符串

**底层实现**：

```java
public final class StringBuffer extends AbstractStringBuilder {
    // 继承 AbstractStringBuilder
    // char[] value;  // 非 final，可变
    
    // 线程安全：方法加 synchronized 锁
    @Override
    public synchronized StringBuffer append(String str) {
        super.append(str);  // 调用父类方法
        return this;
    }
    
    @Override
    public synchronized StringBuffer insert(int offset, String str) {
        super.insert(offset, str);
        return this;
    }
    
    @Override
    public synchronized StringBuffer delete(int start, int end) {
        super.delete(start, end);
        return this;
    }
}

// 父类 AbstractStringBuilder
abstract class AbstractStringBuilder {
    char[] value;  // 非 final，可变
    int count;
    
    public AbstractStringBuilder append(String str) {
        int len = str.length();
        ensureCapacityInternal(count + len);
        str.getChars(0, len, value, count);  // 直接在原数组修改
        count += len;
        return this;
    }
}
```

**特点**：

```
1. 可变性：
   - char[] 数组（非 final）
   - 直接在原对象修改
   - 支持动态扩展

2. 线程安全：
   - 方法加 synchronized 锁
   - 多线程共享变量安全
   - 性能中等（加锁开销）

3. 适用场景：
   - 多线程环境
   - 共享变量拼接
   - 需要线程安全保证
```

**使用示例**：

```java
// ✅ 适用场景：多线程共享变量
public class MultiThreadExample {
    private StringBuffer sharedBuffer = new StringBuffer();
    
    public void appendData(String data) {
        // 多线程安全
        sharedBuffer.append(data);
    }
}

// ❌ 不适用：单线程场景（性能浪费）
public void singleThreadMethod() {
    StringBuffer sb = new StringBuffer();
    for (int i = 0; i < 10000; i++) {
        sb.append(i);  // 单线程用 StringBuffer，浪费性能
    }
    // 应该用 StringBuilder
}
```

---

#### 2.3 StringBuilder：高性能的可变字符串

**底层实现**：

```java
public final class StringBuilder extends AbstractStringBuilder {
    // 继承 AbstractStringBuilder
    // char[] value;  // 非 final，可变
    
    // 无锁，线程不安全
    @Override
    public StringBuilder append(String str) {
        super.append(str);  // 调用父类方法，无锁
        return this;
    }
    
    @Override
    public StringBuilder insert(int offset, String str) {
        super.insert(offset, str);
        return this;
    }
}
```

**特点**：

```
1. 可变性：
   - char[] 数组（非 final）
   - 直接在原对象修改
   - 支持动态扩展

2. 线程不安全：
   - 无锁
   - 多线程共享变量不安全
   - 性能最高（无锁开销）

3. 适用场景：
   - 单线程环境
   - 高频拼接
   - 循环拼接
   - 性能敏感场景
```

**使用示例**：

```java
// ✅ 适用场景：单线程高频拼接
public void buildSql() {
    StringBuilder sb = new StringBuilder();
    sb.append("SELECT * FROM users WHERE ");
    sb.append("age > ").append(18);
    sb.append(" AND status = ").append(1);
    return sb.toString();
}

// ✅ 适用场景：循环拼接
public String joinList(List<String> list) {
    StringBuilder sb = new StringBuilder();
    for (String item : list) {
        sb.append(item).append(",");
    }
    return sb.toString();
}

// ❌ 不适用：多线程共享变量
public class BadExample {
    private StringBuilder sharedBuilder = new StringBuilder();
    
    public void appendData(String data) {
        // 多线程不安全！
        sharedBuilder.append(data);
    }
}
```

---

### 三、完整对比表

| 特性 | String | StringBuffer | StringBuilder |
|------|--------|--------------|---------------|
| **可变性** | 不可变（final） | 可变 | 可变 |
| **线程安全** | 安全（不可变） | 安全（synchronized） | 不安全（无锁） |
| **性能** | 低（每次创建新对象） | 中（加锁开销） | 高（无锁） |
| **适用场景** | 少量赋值、常量 | 多线程共享变量 | 单线程高频拼接 |
| **内存效率** | 差（大量临时对象） | 中 | 优 |
| **底层实现** | final char[] | char[] + synchronized | char[] |

---

### 四、使用场景总结

```
场景一：少量赋值
✅ String name = "张三";

场景二：字符串常量
✅ public static final String APP_NAME = "MyApp";

场景三：单线程高频拼接
✅ StringBuilder sb = new StringBuilder();
   for (int i = 0; i < 10000; i++) {
       sb.append(i);
   }

场景四：循环拼接
✅ StringBuilder sb = new StringBuilder();
   for (String item : list) {
       sb.append(item).append(",");
   }

场景五：多线程共享变量
✅ StringBuffer sb = new StringBuffer();
   // 多线程安全

场景六：SQL 拼接
✅ StringBuilder sb = new StringBuilder();
   sb.append("SELECT * FROM users WHERE ");
   sb.append("age > ").append(18);
```

---

### 五、常见技术场景

#### 5.1 为什么 String 设计成不可变？

```
1. 字符串常量池需要：
   - 相同内容共享同一个对象
   - 如果可变，一个修改影响所有引用

2. 线程安全：
   - 不可变对象天然线程安全
   - 无需同步，性能好

3. 哈希值缓存：
   - String 常用作 HashMap 的 key
   - 哈希值只需计算一次，缓存起来
   - 如果可变，哈希值变化导致无法查找

4. 安全性：
   - 字符串广泛用作参数（网络连接、数据库连接、文件路径）
   - 不可变防止被恶意修改
```

#### 5.2 String s = new String("hello") 创建了几个对象？

```
答案：1 个或 2 个

情况一：字符串常量池已有 "hello"
- 常量池不创建新对象
- 堆中创建 1 个对象（s 指向）
- 共 1 个对象

情况二：字符串常量池没有 "hello"
- 常量池创建 1 个对象（"hello"）
- 堆中创建 1 个对象（s 指向）
- 共 2 个对象
```

#### 5.3 StringBuilder 如何保证线程安全？

```
StringBuilder 本身不保证线程安全。

多线程场景需要：
1. 使用 StringBuffer（推荐）
2. 手动加锁
   StringBuilder sb = new StringBuilder();
   synchronized (sb) {
       sb.append("data");
   }
3. 使用 ThreadLocal（每个线程独立实例）
   ThreadLocal<StringBuilder> tl = ThreadLocal.withInitial(StringBuilder::new);
```

---

技术深入探讨

### 考点 1：String、StringBuffer、StringBuilder 的区别？

**方案参考**（60 分）：
```
三者的主要区别：

1. 可变性：
   - String 不可变（final char[]）
   - StringBuffer、StringBuilder 可变（char[]）

2. 线程安全：
   - String 线程安全（不可变）
   - StringBuffer 线程安全（synchronized）
   - StringBuilder 线程不安全（无锁）

3. 性能：
   - StringBuilder > StringBuffer > String

4. 使用场景：
   - 少量赋值用 String
   - 多线程用 StringBuffer
   - 单线程高频拼接用 StringBuilder
```

**加分回答**（90 分）：
```
三者的核心区别和底层原理：

1. 底层实现：
   String：
   - final char[] 数组
   - 每次修改创建新对象
   - 不可变，线程安全
   
   StringBuffer/StringBuilder：
   - 都继承 AbstractStringBuilder
   - char[] 数组（非 final）
   - 直接在原对象修改
   
   区别：
   - StringBuffer 方法加 synchronized 锁
   - StringBuilder 无锁

2. 性能对比：
   StringBuilder > StringBuffer > String
   
   String 性能差的原因：
   - 每次修改创建新对象
   - 产生大量临时对象
   - GC 压力大
   
   StringBuilder 性能高的原因：
   - 无锁开销
   - 直接在原对象修改
   - 内存效率高

3. 使用场景：
   - 少量赋值：String
   - 字符串常量：String
   - 单线程高频拼接：StringBuilder
   - 循环拼接：StringBuilder
   - 多线程共享变量：StringBuffer

4. 注意事项：
   - 严禁循环内使用 String 拼接
   - 单线程优先用 StringBuilder
   - 多线程用 StringBuffer

核心思路：
字符串拼接不是想怎么用就怎么用，
而是按场景选型、兼顾效率与安全。
```

---

### 考点 2：为什么 String 设计成不可变？

**方案参考**（60 分）：
```
String 设计成不可变的原因：

1. 字符串常量池需要：
   - 相同内容共享同一个对象
   - 如果可变，一个修改影响所有引用

2. 线程安全：
   - 不可变对象天然线程安全
   - 无需同步

3. 哈希值缓存：
   - String 常用作 HashMap 的 key
   - 哈希值只需计算一次

4. 安全性：
   - 字符串广泛用作参数
   - 不可变防止被恶意修改
```

**加分回答**（90 分）：
```
String 不可变设计的深层原因：

1. 字符串常量池（String Pool）：
   - JVM 维护字符串常量池
   - 相同内容共享同一个对象
   - 节省内存空间
   
   String s1 = "hello";
   String s2 = "hello";
   // s1 和 s2 指向同一个对象
   
   如果 String 可变：
   s1 修改会影响 s2，逻辑混乱

2. 线程安全：
   - 不可变对象天然线程安全
   - 无需同步，性能好
   - 多个线程可以安全访问
   
   多线程场景：
   String s = "hello";
   // 多个线程读取，无需加锁

3. 哈希值缓存：
   - String 的 hashCode 常用作 HashMap 的 key
   - 哈希值只需计算一次，缓存起来
   - 如果可变，哈希值变化导致无法查找
   
   Map<String, Object> map = new HashMap<>();
   map.put("hello", value);
   // "hello" 的哈希值缓存，快速查找
   
   如果 String 可变：
   修改后哈希值变化，无法从 map 中找到

4. 安全性：
   - 字符串广泛用作参数
   - 网络连接、数据库连接、文件路径
   - 不可变防止被恶意修改
   
   如果 String 可变：
   恶意代码修改字符串参数，导致安全漏洞

核心思路：
String 的不可变设计是深思熟虑的，
兼顾了性能、安全、易用性。
```

---

### 考点 3：循环内为什么不能用 String 拼接？

**方案参考**（60 分）：
```
循环内用 String 拼接会导致：

1. 每次拼接创建新对象
   String result = "";
   for (int i = 0; i < 10000; i++) {
       result += i;  // 每次创建新对象
   }
   // 创建了 10000 个临时对象

2. 内存浪费严重
   - 临时对象占用内存
   - 老对象等待 GC 回收

3. GC 压力大
   - 大量临时对象触发 GC
   - 影响性能

正确做法：使用 StringBuilder
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i);  // 在原对象修改
}
```

**加分回答**（90 分）：
```
循环内用 String 拼接的性能问题：

1. 底层原理：
   String result = "";
   for (int i = 0; i < 10000; i++) {
       result += i;
   }
   
   编译后等价于：
   String result = "";
   for (int i = 0; i < 10000; i++) {
       StringBuilder temp = new StringBuilder(result);
       temp.append(i);
       result = temp.toString();  // 每次创建新对象
   }
   
   问题：
   - 每次循环都创建 StringBuilder 和 String 对象
   - 10000 次循环，创建 20000 个对象
   - 内存浪费严重

2. 性能对比：
   // String 拼接
   long start = System.currentTimeMillis();
   String result = "";
   for (int i = 0; i < 100000; i++) {
       result += i;
   }
   // 耗时：5000ms+
   
   // StringBuilder 拼接
   start = System.currentTimeMillis();
   StringBuilder sb = new StringBuilder();
   for (int i = 0; i < 100000; i++) {
       sb.append(i);
   }
   // 耗时：5ms
   
   性能差距：1000 倍！

3. 内存问题：
   String 拼接：
   - 每次创建临时对象
   - 老对象等待 GC 回收
   - GC 频繁触发，影响性能
   
   StringBuilder 拼接：
   - 在原对象修改
   - 只有一个对象
   - 内存效率高

4. 正确做法：
   // ✅ 单线程：StringBuilder
   StringBuilder sb = new StringBuilder();
   for (int i = 0; i < 10000; i++) {
       sb.append(i);
   }
   String result = sb.toString();
   
   // ✅ 多线程：StringBuffer
   StringBuffer sb = new StringBuffer();
   for (int i = 0; i < 10000; i++) {
       sb.append(i);
   }
   String result = sb.toString();

核心思路：
严禁循环内使用 String 拼接，
高频拼接用 StringBuilder，
多线程用 StringBuffer。
```

---

## 总结

### 一句话总结

**String 严禁循环内拼接，高频拼接用 StringBuilder，多线程共享用 StringBuffer，少量赋值用 String。**

### 核心记忆点

```
1. 三者区别：
   - String：不可变，线程安全，性能低
   - StringBuffer：可变，线程安全，性能中
   - StringBuilder：可变，线程不安全，性能高

2. 使用场景：
   - 少量赋值：String
   - 字符串常量：String
   - 单线程高频拼接：StringBuilder
   - 循环拼接：StringBuilder
   - 多线程共享变量：StringBuffer

3. 关键要点：
   - 严禁循环内使用 String 拼接
   - 单线程优先用 StringBuilder
   - 多线程用 StringBuffer

4. 底层原理：
   - String：final char[]，每次创建新对象
   - StringBuffer：char[] + synchronized
   - StringBuilder：char[]，无锁

5. 技术要点：
   - 先说三者区别（可变性、线程安全、性能）
   - 再说使用场景
   - 最后说底层原理（加分项）
```

---

## 参考链接

- **视频链接**：https://www.douyin.com/video/7628947499952196891
- **豆包对话**：https://www.doubao.com/chat/38448185980392194

---

**标签**：#后端开发 #Java技术 #String #StringBuffer #StringBuilder #字符串 #性能优化 #线程安全
