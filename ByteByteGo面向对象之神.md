# 02 A Framework for the OOD Interview

## Step 1：直觉与破题（The Intuition）

> **给一场“很容易跑偏的开放式聊天”装一条“导航路线”。**
> 这样你和面试官不会一会儿聊收费规则、一会儿聊数据库、一会儿又回到车位类型，像迷路一样绕圈；你会一直“往前推进”，而且让面试官看得懂你在干嘛。

**反直觉点**：很多人以为面试是“谁写得快谁赢”，但其实更像“谁把戏排得清楚谁赢”。代码只是你排戏的一个结果，不是起点。

------

### 你现在要记住的一句话

> **框架 = 导演的排戏流程：先搞清楚要拍什么 → 找出关键角色 → 安排他们怎么配合 → 最后挑毛病补漏洞。**

## Step 2：逻辑推演（The Logic Chain）

问题不在“你会不会写代码”，而在“你能不能把混乱变成结构”。
面试官看到的是：你很努力，但**不稳定、不可控、缺少工程思维**。

所以：**框架存在的意义**不是让你“更快画图”，而是让你**更少返工、更少跑偏、更容易展示你的取舍能力**。

------

### How：它的内部运行逻辑是怎样的？（链式推导，不是列点）

我们把“开放式题”看成一团雾。你要做的，是把雾一步步压缩成可落地的结构：

**第一步：先把雾的边界画出来（Requirements Gathering）**
因为雾最大的问题是“边界不清”。
你用提问把范围钉住：要做什么、不做什么、优先级是什么、限制是什么。
一旦边界明确，你后面每个设计决定都有依据——不再是凭感觉。

**第二步：边界明确后，必须找到“最核心的一条故事线”（Identify Core Objects）**
人脑处理复杂系统靠“主线”。
你挑一个主用例（例如：车进来 → 找位 → 发票 → 离开结算），走一遍流程。
在走流程时，名词自然冒出来（车、车位、票、停车场），动词自然冒出来（分配、占用、释放、计费）。
这一步的作用是：把抽象需求压缩成“对象 + 交互”，让系统从雾变成草图。

**第三步：有了草图，才能谈“工程化的结构”（Design Class Diagram & Code）**
此时你不是凭空设计类，而是在回答：
“这些对象各自负责什么？怎么协作？数据放哪？扩展点在哪？”
你会做权衡：继承 vs 组合、List vs Set、策略模式是否值得、哪些先简化。
然后把它翻译成 UML / code skeleton / working code（取决于交付物要求）。
这一步的本质是：把草图变成“能实现、能解释、能扩展”的结构。

**第四步：结构出来后，面试官通常会开始“找茬”——你要把找茬变成加分（Deep Dive）**
因为真实工程就是：边界条件、异常、可扩展、性能、并发、安全、测试。
你不是被动挨打，而是主动展示：
“我知道哪里可能出问题，我能解释取舍，我能按时间做增量增强。”
这一步的意义是：让面试官相信你不仅会搭框架，还能把系统带到生产可用的方向。

## Step 3：结构化拆解（MECE 结构）

下面把 **OOD 面试框架**拆成互不重叠、完全覆盖的 4 块（你只要背这个就不容易跑偏）：

### MECE：四步到底各“产出”什么？

1. **需求对齐（Requirements）**
   产出：**范围边界 + 优先级 + 假设 + 约束 + 交付物形式**
   （一句话：先把题目“钉死”，避免后面返工）
2. **核心对象（Core Objects）**
   产出：**主用例流程 + 核心名词(类) + 核心动词(方法) + 关键数据**
   （一句话：从“故事线”里抽出“角色与动作”）
3. **类图与代码骨架（Design + Skeleton/Code）**
   产出：**类职责划分 + 关系(组合/继承/依赖) + API/方法签名 + 数据结构选择**
   （一句话：把草图变成工程结构）
4. **深挖与取舍（Deep Dive）**
   产出：**边界情况处理 + 扩展点/模式 + 性能/并发/一致性 + 测试思路**
   （一句话：把“追问”变成“亮点”）

------

## Step 4：记忆编码（The Encoding）

### 最容易被误解的地方（坑点清单）

下面这些是 OOD 面试里最常见、最致命的坑（你只要躲开就赢一半）：

**坑 A：把 Step 3 当成 Step 1**

- 一上来就画类/写代码，看起来很“勤奋”，但风险最大：需求一变就全盘崩。

**坑 B：对象职责“糊成一坨”**

- 例如 ParkingLot 里既做分配又算钱又验证票又存车辆……
- 面试官会觉得你缺乏分层与单一职责（SRP）。

**坑 C：过度设计（为未来十年造航母）**

- 一上来就策略模式 + 工厂模式 + 观察者 + 线程池……
- OOD 面试最看重的是：**恰到好处的扩展点**，不是模式背诵。

**坑 D：沉默写代码**

- OOD “旅程=评分点”。你不说思路，面试官只能猜你会不会。

**坑 E：只讲设计不管数据结构**

- 例如该用 `Map<SpotId, Spot>` 却用 `List` 每次线性扫；
- 面试官会觉得你缺少工程性能意识。

# 03 OOP Fundamental

下面我把你这份 OOP 笔记，改成**10岁也能懂、又能拿去 SDE/OOD 面试直接背的版本**。（中文讲解，英文术语保留）

------

## OOP 四大基石（面试必背 1 句话版）

记忆口诀：**封-抽-继-多**（Encapsulation / Abstraction / Inheritance / Polymorphism）

- **Encapsulation（封装）**：把数据和操作数据的方法装进“盒子”，外面只能走“门”（public methods），里面细节不给乱碰（private fields）。
- **Abstraction（抽象）**：只给你“该怎么用”（what），不告诉你“里面怎么做”（how），像遥控器只给按钮不让你看电路。
- **Inheritance（继承）**：子类“是一个”父类（is-a），能复用父类代码，但会更绑定（tight coupling）。
- **Polymorphism（多态）**：同一个接口/父类引用，放不同子类对象，调用同名方法时表现不一样（运行时决定）。

------

## 0. 先把基础词汇搞清楚（面试常问）

- **class**：对象的“图纸/模板”（比如 Person）
- **object / instance**：按图纸造出来的“真实东西”（比如 new Person("Tom", 10)）

------

## 1) Encapsulation（封装）——“盒子 + 门”

### 1.1 10岁版理解

你有一个**玩具盒**（对象）：

- 玩具（数据：name、age）放在盒子里面
- 盒子上有按钮/开关（方法：getAge、setAge）
- 你不能直接把手伸进去乱改（private），必须按按钮（public）

### 1.2 面试版一句话定义（背这句）

**Encapsulation = 把数据和操作数据的方法绑在一起，并用 private/public 控制访问，让对象自己保护自己的状态（invariant）。**

> invariant = “永远必须成立的规则”，比如 age 不能为负数。

### 1.3 你代码里 Person 的“封装价值”到底是什么？

- `age` 是 **private**：外界不能 `person.age = -5`
- `setAge()` 里做检查：**把规则锁在门口**，保证对象一直合法

### 1.4 什么时候用封装（最关键的 3 个）

- **保证数据合法**：例如余额不能为负、库存不能小于 0
- **隔离变化**：内部字段以后改名/改结构，外面不用全改
- **减少耦合**：外面只依赖“门”（方法），不依赖内部实现

### 1.5 常见坑（面试会加分）

- **过度封装**：每个字段都无脑 getter/setter → 变成“裸数据结构”，对象没真正保护规则
  ✅ 更好的做法：暴露“行为”而不是暴露“所有字段”
- **暴露可变内部对象**：比如直接 return 内部 list，会被外界改坏
  ✅ 返回不可变视图 / 拷贝（根据语言特性）

------

## 2) Abstraction（抽象）——“只告诉你怎么用，不告诉你怎么做”

### 2.1 10岁版理解

遥控器的“音量 + / -”：

- 你只管按按钮（调用方法）
- 不需要知道电视里面怎么放大电流、怎么驱动喇叭（实现细节）

### 2.2 面试版一句话定义（背这句）

**Abstraction = 只暴露稳定的高层能力（what），隐藏复杂细节（how），让使用者更简单、更不容易用错。**

### 2.3 Java 里怎么做抽象？

- **interface**：只定义“必须有什么能力”（方法签名）
- **abstract class**：可以“部分实现 + 部分留空”（抽象方法）

你的例子里：

- `Shape.area()`：只规定“形状必须能算面积”，但怎么算由子类决定
- `Drawable.draw()`：规定“能画”，具体怎么画由实现类决定

### 2.4 什么时候用抽象（最关键的 3 个）

- **你有多种实现**：Circle/Rectangle/Triangle 都要 area()
- **你希望以后能加新类型**：加 Triangle 不改旧代码（面试关键词：Open-Closed）
- **你想让调用方不用懂细节**：调用方只知道 `shape.area()`

### 2.5 常见坑

- **抽象泄漏（leaky abstraction）**：接口看似简单，但使用者还是得知道内部细节才能用对
- **过度抽象**：层数太多、接口太碎 → 读代码像迷宫

------

## 3) Abstraction vs Encapsulation（最容易混，面试高频）

用一句“对比记忆”：

- **Encapsulation（封装）**：重点是**“不让你乱碰我的数据”**（保护对象内部状态）
- **Abstraction（抽象）**：重点是**“我只给你你需要的功能按钮”**（隐藏复杂度）

更直观一句：

- 封装：**关上盒子**（private + 规则）
- 抽象：**只给遥控器**（interface/abstract class）

------

## 4) Inheritance（继承）——“is-a 家族关系”

### 4.1 10岁版理解

狗（Dog）是动物（Animal）的一种：

- 狗自动拥有动物的能力（比如 eat）
- 狗还能加自己的能力（bark）

### 4.2 面试版一句话定义（背这句）

**Inheritance = 建立 is-a 关系，子类复用父类状态/行为，并能扩展或重写（override）。**

### 4.3 什么时候用继承（3 条硬标准）

- 真的满足 **is-a**，而不是“像”
- 父类行为对所有子类都合理且稳定（不会经常改）
- 子类可以被当成父类使用也不出问题（面试加分词：Liskov Substitution）

### 4.4 继承的 3 个大缺点（你笔记里说的更“面试化”版本）

- **tight coupling**：父类一改，子类可能全炸（fragile base class）
- **继承了不该继承的东西**：比如 Animal 有 fly()，企鹅怎么办？
- **关系固定，难以组合**：RobotDog 想 bark 但不想 eat，很尴尬

------

## 5) Composition（组合）——“has-a 装插件”（面试偏爱）

### 5.1 10岁版理解

手机不是“天生会导航”，而是**装一个导航 App**。
你想换导航？换 App 就行。

### 5.2 面试版一句话（背这句）

**Composition = has-a：把可变能力做成组件/策略（strategy），主类通过“持有它”获得能力，比继承更灵活、更低耦合。**

你 BarkBehavior 的例子就是典型 **Strategy pattern** 思想：

- Dog/RobotDog 不靠继承“强绑定”
- 而是各自“装一个会叫的插件”

### 5.3 面试黄金规则（直接背）

**Prefer composition over inheritance**：
当行为可能变化/组合/替换时，优先用组合。

------

## 6) Polymorphism（多态）——“同一个按钮，不同表现”

### 6.1 10岁版理解

同一个“播放 play”按钮：

- 播放音乐：走音频逻辑
- 播放视频：走视频逻辑
  你只按同一个按钮（统一接口），系统自己决定怎么播（不同实现）。

### 6.2 面试版一句话定义（背这句）

**Polymorphism = 同一接口/父类引用指向不同子类对象，调用同一方法名时，实际执行哪一个实现由对象真实类型决定（runtime dispatch）。**

### 6.3 两种多态（面试必问）

#### A) Compile-time（静态）多态：**method overloading**

- 同名方法，参数不同
- 编译器根据参数决定调用哪个

#### B) Runtime（动态）多态：**method overriding**

- 子类重写父类方法
- 运行时根据真实对象类型决定调用哪个（dynamic dispatch）

下面把你这份 **SOLID** 笔记改成：**10岁能懂 + 面试能背 + 一开口就显得你会设计** 的版本。（中文解释，英文术语保留）

------

## SOLID 一句话总纲（面试开场白）

## 0）最快背法：每个字母 = 1 个“怕什么”

- **S（SRP）**：怕一个类“太杂”，一改牵一大片
- **O（OCP）**：怕加功能要改旧代码，改了容易炸
- **L（LSP）**：怕子类“装作是父类但不守规矩”，替换就出错
- **I（ISP）**：怕接口“太胖”，逼别人实现用不到的方法
- **D（DIP）**：怕高层逻辑直接依赖细节，换实现就得重写

------

## 1) SRP — Single Responsibility Principle（单一职责）

### 10岁版：一个人干一件事

让 **Employee** 只当“员工资料”，别让它又算工资又打印报表。
不然：工资规则变了你要改它；报表格式变了你也要改它——同一个类有**两个理由**要改。

### 面试背句

**SRP：一个类只有一个“变化原因”（one reason to change）。**

### 你例子里的问题（面试会这么说）

`Employee` 同时负责：

1. 计算年薪（business logic）
2. 生成报表（presentation/reporting）
   => 两种变化混在一起，维护痛苦。

### 正确做法（最常见答案）

- `Employee`：只存数据 + 计算年薪
- `PayrollReportGenerator`：只负责报表输出

### 面试加分点（最容易被追问）

SRP 的“职责”不是随便切：

- 不是“每个方法一个类”那么极端
- 正确切法：按**变化来源**切（工资规则 vs 报表格式）

------

## 2) OCP — Open/Closed Principle（开闭原则）

### 10岁版：插新积木，不砸旧积木

你要支持新形状（Circle/Triangle），不应该每次都去改 `AreaCalculator` 的老代码。

### 面试背句

**OCP：对扩展开放（add new code），对修改关闭（don’t change tested old code）。**

### 你例子里的问题

`AreaCalculator` 只认识 `Rectangle`。
以后要支持 Circle → 你得改 `AreaCalculator` → 老代码风险增加。

**Violation of OCP**

Here’s an example of a class that violates OCP by requiring changes to support new shapes:

```java
class Rectangle {
    private double width;
    private double height;

    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }

    public double calculateArea() {
        return width * height;
    }
}

class AreaCalculator {
    public double calculateArea(Rectangle rectangle) {
        return rectangle.calculateArea();
    }
}
```

### 标准改法（面试最爱）

![image-20260207204113476](C:\Learning Notes\Java\面向对象设计\img\image-20260207204113476.png)

抽象出 `Shape`（interface/abstract class）：

- `Shape.calculateArea()`
- `Rectangle/Circle/Triangle` 各自实现
  `AreaCalculator` 只依赖 `Shape`，以后加新形状：**加类，不改老类**。

### 面试金句

**“用 polymorphism 实现 OCP。”**
（用父类/接口统一处理，新增类型只需新增实现）

------

## 3) LSP — Liskov Substitution Principle（里氏替换）

### 10岁版：说你是“鸟”，就得像鸟一样能被当鸟用

如果代码拿到一个 `Bird` 就会调用 `fly()`，那你把 `Ostrich` 当 `Bird` 传进去却直接抛异常，就等于“假装是 Bird”。

### 面试背句

**LSP：子类对象必须能替换父类对象，且程序仍然正确（subtype must be substitutable）。**

### 你例子里的核心错误（面试表达）

父类 `Bird` 的“合同/承诺（contract）”是：调用 `fly()` 能飞。
子类 `Ostrich` 却：调用 `fly()` 抛异常。
=> 违反合同，替换性破坏。

### 标准修复（两种常见答案）

**方案 A（你笔记里那个）：把能力改得更通用**

- `Bird.move()`：所有鸟都会移动
- `Sparrow.move()` = 飞
- `Ostrich.move()` = 跑
  这样任何 Bird 都能 move，不会炸。

**方案 B（更面试化、更强）：把“飞”做成单独能力**

- `interface Flyable { fly(); }`
- `Sparrow implements Flyable`
- `Ostrich 不实现 Flyable`
  调用方如果要飞，就只接收 Flyable。

### LSP 进阶三条（面试官喜欢）

子类重写时必须守住父类合同：

- **Precondition 不能更严格**（不能要求更多输入条件）
- **Postcondition 不能更弱**（不能减少保证）
- **Invariants 必须保持**（不能破坏父类永远成立的规则）

------

## 4) ISP — Interface Segregation Principle（接口隔离）

### 10岁版：不要给机器人发“吃饭作业”

`Worker` 里有 `eat()` `sleep()`，Robot 明明不需要，却被强迫实现，只能抛异常，太别扭。

#### Violation of ISP

Here’s an example of a design that violates ISP by including methods that not all implementing classes need:

```java
interface Worker {
    void work();

    void eat();

    void sleep();
}

class Robot implements Worker {
    public void work() {
        System.out.println("Performing tasks like welding.");
    }

    public void eat() {
        throw new UnsupportedOperationException("Robots don't eat.");
    }

    public void sleep() {
        throw new UnsupportedOperationException("Robots don't sleep.");
    }
}

class Human implements Worker {
    public void work() {
        System.out.println("Performing tasks like coding.");
    }

    public void eat() {
        System.out.println("Eating a meal.");
    }

    public void sleep() {
        System.out.println("Sleeping for rest.");
    }
}
```

### 面试背句

**ISP：客户端不应该被迫依赖它不需要的方法（no fat interface）。**

### 标准修复（面试标准答案）

![image-20260207203920556](C:\Learning Notes\Java\面向对象设计\img\image-20260207203920556.png)

把胖接口拆成小接口：

- `Workable.work()`
- `Eatable.eat()`
- `Sleepable.sleep()`

Robot：只实现 Workable
Human：实现 Workable + Eatable + Sleepable

### 面试一句话总结

**“接口按使用者拆，而不是按实现者偷懒合并。”**

------

## 5) DIP — Dependency Inversion Principle（依赖倒置）

**Violation of DIP**

Here’s an example of a design that violates DIP by having a high-level class depend directly on a low-level class:

```java
class LightBulb {
    public void turnOn() {
        System.out.println("LightBulb is on.");
    }

    public void turnOff() {
        System.out.println("LightBulb is off.");
    }
}

class Switch {
    private LightBulb bulb;

    public Switch(LightBulb bulb) {
        this.bulb = bulb;
    }

    public void operate() {
        bulb.turnOn();
    }
}
```

### 10岁版：开关不该只控制“灯泡”，应该控制“任何可开关的东西”

你现在的 `Switch` 直接拿 `LightBulb`，以后换成 `Fan` 你就得改 Switch。

### 面试背句（最标准）

**DIP：高层模块不依赖低层模块，二者都依赖抽象（abstractions）。**

- 高层：Switch（业务逻辑/控制逻辑）
- 低层：LightBulb（具体设备实现）
- 抽象：`Switchable`

![image-20260207203756974](C:\Learning Notes\Java\面向对象设计\img\image-20260207203756974.png)

### 标准修复（面试最爱）

- `interface Switchable { turnOn(); turnOff(); }`
- `Switch` 依赖 `Switchable`
- `LightBulb implements Switchable`
  以后加 `Fan implements Switchable`：**Switch 不用改**

### DIP 常和什么一起出现？（面试加分）

- **Dependency Injection（依赖注入）**：通过构造函数/参数把具体实现“塞进去”，Switch 不自己 new。
- 记忆：**DIP 是原则，DI 是实现手段**。

------

## 6）SOLID 一页“面试速背卡”（直接背）

- **SRP**：一个类一个变化原因（工资逻辑别和报表混）
- **OCP**：加功能用新增类，不改旧代码（Shape + polymorphism）
- **LSP**：子类必须守父类合同，能无痛替换（Ostrich 不该继承 fly 合同）
- **ISP**：接口别胖，按使用者拆（Robot 不要 eat/sleep）
- **DIP**：高层依赖抽象，不依赖细节（Switch → Switchable，不绑 LightBulb）

------

## 7）最常见追问，你怎么答（给你现成句子）

### Q1：SOLID 里你最常用哪几个？

答：**SRP + OCP + DIP** 最常用。SRP 让代码好维护，OCP 让系统好扩展，DIP 让组件可替换、可测试。

### Q2：SRP 和 OCP 有啥关系？

答：SRP 先把职责拆清楚，OCP 才能更容易对某个点扩展而不改旧代码；职责越清晰，扩展点越自然。

### Q3：LSP 违反会有什么真实后果？

答：调用方以为拿到父类就能按父类合同调用，结果子类抛异常/行为不一致 → 线上 bug，且难排查。

------

### **1️⃣ Singleton（单例）** ⭐⭐⭐⭐⭐

**出现率：最高**

**为什么爱考**

- 最简单，但**能考并发 / 线程安全**
- Java 后端面试必问

**你必须会**

- 懒汉 vs 饿汉
- `double-checked locking`
- `volatile` 的作用
- enum 单例（加分）

👉 **一句话**：面试官不考你“知道不知道”，考你**写不写得对**

**A. 饿汉式（最简单，线程安全）**

```java
public final class HungrySingleton {
    private static final HungrySingleton INSTANCE = new HungrySingleton();

    private HungrySingleton() {} // 防止 new

    public static HungrySingleton getInstance() {
        return INSTANCE;
    }
}
```

**B. 懒汉式（线程不安全，面试用来引出问题）**

```java
public final class LazySingletonBad {
    private static LazySingletonBad instance;

    private LazySingletonBad() {}

    public static LazySingletonBad getInstance() {
        if (instance == null) {
            instance = new LazySingletonBad(); // 多线程下会创建多个
        }
        return instance;
    }
}
```

------

### **2️⃣ Factory / Factory Method（工厂）** ⭐⭐⭐⭐⭐

**出现率：极高**

**为什么爱考**

- 完美贴合 **解耦 + 面向接口编程**
- Spring、后端系统到处都是

**你必须会**

- Simple Factory vs Factory Method
- 为什么不用 `new`
- 扩展性（新增类不改原逻辑）

👉 **一句话**：**new 写多了 = 设计不行**

**A. Simple Factory（简单工厂：集中一个地方创建）**

> **缺点**：新增类型要改工厂（违反开闭原则），但面试常问对比。

```java
interface Payment {
    void pay(int cents);
}

class AliPay implements Payment {
    public void pay(int cents) { System.out.println("AliPay: " + cents); }
}
class WeChatPay implements Payment {
    public void pay(int cents) { System.out.println("WeChat: " + cents); }
}

class SimplePaymentFactory {
    public static Payment create(String type) {
        return switch (type) {
            case "ALI" -> new AliPay();
            case "WX"  -> new WeChatPay();
            default -> throw new IllegalArgumentException("unknown type: " + type);
        };
    }
}

// usage:
class Demo1 {
    public static void main(String[] args) {
        Payment p = SimplePaymentFactory.create("ALI");
        p.pay(100);
    }
}
```

**B. Factory Method（工厂方法：每种产品一个工厂，扩展不用改原逻辑）**

```java
interface Payment {
    void pay(int cents);
}

class AliPay implements Payment {
    public void pay(int cents) { System.out.println("AliPay: " + cents); }
}
class WeChatPay implements Payment {
    public void pay(int cents) { System.out.println("WeChat: " + cents); }
}

interface PaymentFactory {
    Payment create();
}

class AliPayFactory implements PaymentFactory {
    public Payment create() { return new AliPay(); }
}
class WeChatPayFactory implements PaymentFactory {
    public Payment create() { return new WeChatPay(); }
}

// usage:
class Demo2 {
    public static void main(String[] args) {
        PaymentFactory factory = new WeChatPayFactory();
        factory.create().pay(200);
    }
}
```

**“为什么不用 new”一句话**：调用方只依赖接口 `Payment`，创建细节交给工厂，**解耦** + **易扩展/易替换/易测试**。

------

### **3️⃣ Strategy（策略）** ⭐⭐⭐⭐☆

**出现率：高**

**为什么爱考**

- 和 **if-else 地狱**强对比
- 非常贴近真实业务（支付、折扣、排序）

**你必须会**

- 接口 + 多实现类
- 如何在运行时切换策略
- 和 Factory 搭配用（常见）

👉 **一句话**：**干掉 if-else 的第一神器**

同样用支付，但这里核心是：**运行时切换策略，干掉 if-else**。

```java
interface DiscountStrategy {
    int apply(int originCents);
}

class NoDiscount implements DiscountStrategy {
    public int apply(int originCents) { return originCents; }
}
class PercentageOff implements DiscountStrategy {
    private final int percent; // 例如 20 表示 20% off
    public PercentageOff(int percent) { this.percent = percent; }
    public int apply(int originCents) {
        return originCents * (100 - percent) / 100;
    }
}
class MinusOff implements DiscountStrategy {
    private final int minusCents;
    public MinusOff(int minusCents) { this.minusCents = minusCents; }
    public int apply(int originCents) {
        return Math.max(0, originCents - minusCents);
    }
}

// 上下文（使用策略的地方）
class CheckoutService {
    private DiscountStrategy strategy;

    public CheckoutService(DiscountStrategy strategy) {
        this.strategy = strategy;
    }
    public void setStrategy(DiscountStrategy strategy) { // 运行时切换
        this.strategy = strategy;
    }

    public int finalPrice(int originCents) {
        return strategy.apply(originCents);
    }
}

// usage:
class Demo3 {
    public static void main(String[] args) {
        CheckoutService s = new CheckoutService(new NoDiscount());
        System.out.println(s.finalPrice(1000)); // 1000

        s.setStrategy(new PercentageOff(20));
        System.out.println(s.finalPrice(1000)); // 800

        s.setStrategy(new MinusOff(300));
        System.out.println(s.finalPrice(1000)); // 700
    }
}
```

**常见搭配：Factory + Strategy**
 用工厂根据配置/枚举创建策略实例，然后塞进 `CheckoutService`。

------

### **4️⃣ Observer（观察者）** ⭐⭐⭐⭐☆

**推模型（Subject 主动把数据推给 Observer）**

```java
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

interface Observer {
    void onEvent(String msg); // 推：直接给消息
}

class EmailObserver implements Observer {
    public void onEvent(String msg) { System.out.println("Email got: " + msg); }
}
class SmsObserver implements Observer {
    public void onEvent(String msg) { System.out.println("SMS got: " + msg); }
}

class Subject {
    private final List<Observer> observers = new CopyOnWriteArrayList<>();

    public void subscribe(Observer o) { observers.add(o); }
    public void unsubscribe(Observer o) { observers.remove(o); }

    public void publish(String msg) {
        for (Observer o : observers) o.onEvent(msg);
    }
}

// usage:
class Demo4 {
    public static void main(String[] args) {
        Subject subject = new Subject();
        subject.subscribe(new EmailObserver());
        subject.subscribe(new SmsObserver());

        subject.publish("Order created");
    }
}
```

**拉模型（Observer 只收到“有变化了”，自己去 Subject 拉数据）**

```java
interface PullObserver {
    void onChanged();
}

class StockSubject {
    private int price;
    private final java.util.List<PullObserver> observers = new java.util.ArrayList<>();

    public void subscribe(PullObserver o) { observers.add(o); }
    public int getPrice() { return price; }

    public void setPrice(int price) {
        this.price = price;
        for (PullObserver o : observers) o.onChanged();
    }
}

class Trader implements PullObserver {
    private final StockSubject subject;
    public Trader(StockSubject subject) { this.subject = subject; }
    public void onChanged() {
        System.out.println("Trader sees price=" + subject.getPrice()); // 拉
    }
}
```

**出现率：中高**

**为什么爱考**

- 考察 **事件驱动思想**
- 前后端 / 分布式都常见

**你必须会**

- Subject / Observer 关系
- 推模型 vs 拉模型
- 解耦通知机制

👉 **一句话**：**发布-订阅 = 面试官最爱的词**

------

### **5️⃣ Decorator（装饰器）** ⭐⭐⭐☆

核心：**组合包一层，运行时叠加功能，不改原类**。

```java
interface Coffee {
    int cost();
    String desc();
}

class BasicCoffee implements Coffee {
    public int cost() { return 10; }
    public String desc() { return "Coffee"; }
}

// 装饰器基类（关键：持有 Coffee）
abstract class CoffeeDecorator implements Coffee {
    protected final Coffee inner;
    protected CoffeeDecorator(Coffee inner) { this.inner = inner; }
}

class Milk extends CoffeeDecorator {
    public Milk(Coffee inner) { super(inner); }
    public int cost() { return inner.cost() + 2; }
    public String desc() { return inner.desc() + " + Milk"; }
}

class Sugar extends CoffeeDecorator {
    public Sugar(Coffee inner) { super(inner); }
    public int cost() { return inner.cost() + 1; }
    public String desc() { return inner.desc() + " + Sugar"; }
}

// usage:
class Demo5 {
    public static void main(String[] args) {
        Coffee c = new BasicCoffee();
        c = new Milk(c);
        c = new Sugar(c);

        System.out.println(c.desc()); // Coffee + Milk + Sugar
        System.out.println(c.cost()); // 13
    }
}
```

**一句话**：Decorator = Wrapper。Java IO（InputStream / BufferedInputStream）就是经典装饰器。

**出现率：中**

**为什么爱考**

- 能考你是否理解 **组合 > 继承**
- Java IO 经典例子

**你必须会**

- 和继承的区别
- 运行时叠加功能
- Wrapper 思想

👉 **一句话**：**功能叠加但不改原类**

------

### **6️⃣ Adapter（适配器）** ⭐⭐⭐☆

**出现率：中**

**为什么爱考**

- 非常贴近现实工程（老系统 / 第三方库）
- 不难，但能看出工程经验

**你必须会**

- 接口不兼容怎么办
- 类适配器 vs 对象适配器

👉 **一句话**：**“我包一层就能用了”**

**A. 对象适配器（更常用：组合）**

```java
// 你想要的接口
interface Charger {
    int output5V();
}

// 旧/第三方类：输出 220V
class OldPower {
    public int output220V() { return 220; }
}

// 适配器：把 220V 适配成 5V
class PowerAdapter implements Charger {
    private final OldPower oldPower;
    public PowerAdapter(OldPower oldPower) { this.oldPower = oldPower; }

    public int output5V() {
        int v = oldPower.output220V();
        return v / 44; // 简化：220 -> 5
    }
}

// usage:
class Demo6 {
    public static void main(String[] args) {
        Charger charger = new PowerAdapter(new OldPower());
        System.out.println(charger.output5V()); // 5
    }
}
```

# 04 Design a Parking Lot

