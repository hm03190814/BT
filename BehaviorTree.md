# BehaviorTree

- 官方文档 https://www.behaviortree.dev/docs/intro 

# 基本概念

## 简介

### 基本概念

与有限状态机不同，行为树是由层次节点组成的树，它控制着"任务"的执行流程。

- 一个叫做**tick**的信号被发送到树的根并在树中传播，直到到达叶节点。
- 任何接收到**tick**信号的**TreeNode**(树节点)都会执行它的回调。 这个回调必须返回以下结果之一
  - **SUCCESS** (成功)
  - **FAILURE** (失败)
  - **RUNNING**  (运行中)

- RUNNING 表示该操作需要更多时间才能返回有效结果。
- 如果一个 TreeNode 有一个或多个子节点，则它有责任传播tick()；每个节点类型对于是否、何时以及多少次子节点被勾选有不同的规则。
- __LeafNodes(叶节点)__，是没有任何子节点的 TreeNodes，是实际的命令，即行为树与系统其余部分交互的节点 。操作节点(__Action nodes__)是最常见的叶节点类型。

### 节点类型

- TreeNode
  - DecoratorNode (装饰器节点)
  - ControlNode
    - Sequence (序列)
    - Fallback (回退)
  - LeafNode
    - ConditionNode (会在一次tick后立马返回成功或失败的状态信息)
    - ActionNode (可以跨越多个tick执行，直到到达它的结束状态)
      - synchronous (同步节点)
      - asynchronous (异步节点)

| 树节点类型    | 子节点个数 | 概念                                                         |
| ------------- | ---------- | ------------------------------------------------------------ |
| ControlNode   | 1~N        | 根据逻辑传递tick信号到子节点并根据子节点执行状态返回其本身状态 |
| DecoratorNode | 1          | 此外，可以改变子节点的结果或者改变子节点执行的次数           |
| ConditionNode | 0          | 不应该改变系统。不可以返回RUNNING。                          |
| ActionNode    | 0          | 这是一个执行实际任务的节点                                   |

#### ActionNodes

`ActionNodes`分为同步和异步节点。前者自动执行并阻塞树，直到返回`SUCCESS`或`FAILURE`。异步操作可能会返回`RUNNING`以传达操作仍在执行中，我们需要再次勾选它们，直到最终返回`SUCCESS`或`FAILURE`。

#### ControlNode_Sequence(序列)

现有三种sequence node框架：__sequence__、__sequenceStar__、__ReactiveSequence__。

- 触发第一个子节点之前，节点的状态变为RUNNING。
- 如果一个子节点返回FAILURE，则不再勾选子节点，并且Sequence返回FAILURE。
- 如果所有子节点都返回SUCCESS，所有子节点停止，并且Sequence也返回SUCCESS。

__不同点__ 

| 节点类型         | 子节点failure | 子节点running |
| ---------------- | ------------- | ------------- |
| Sequence         | Restart       | Tick again    |
| ReactiveSequence | Restart       | Restart       |
| SequenceStar     | Tick again    | Tick again    |

“__Restart__”指整个序列从第一个子节点重新开始；“__Tick again__”指下次触发时，从当前子节点触发。先前返回SUCCESS的子节点，不再触发。

#### ControlNode_Fallback(回退)

回退节点有两种类型：__Fallback__和__ReactiveFallback__ 

__相同点__

- 触发第一个子节点之前，节点的状态变为running。
- 子节点返回failure，则触发下一个子节点。
- 若最后一个子节点返回failure，所有子节点停止，并返回failure。
- 若任一子节点返回success，停止所有子节点，该节点返回success。

__不同点__ 

__Fallback__ 在下一次被执行tick时，同一个子节点会再次调用。以前已经返回失败的子节点，则不会再被调用。

__ReactiveFallback__ 则每一次都将重投开始执行所有子节点。


#### DecoratorsNode(装饰器节点)

一个装饰器节点只有一个子节点。它可以有以下几类：

| 节点             | 效果                                                         |
| ---------------- | ------------------------------------------------------------ |
| InverterNode     | 反转子节点结果。但当子节点返回RUNNING时，同样返回RUNNING。   |
| ForceSuccessNode | 子节点返回RUNNING时，同样返回RUNNING；否则一律返回SUCCESS。  |
| ForceFailureNode | 子节点返回RUNNING时，同样返回RUNNING；否则一律返回FAILURE。  |
| RepeatNode       | 只要子节点返回SUCCESS，就一直调用子节点，最多N次，其中N是作为输入端口传递的。如果子节点返回FAILURE，则中断循环，并返回FAILURE。如果子节点返回RUNNING，也返回RUNNING。 |
| RetryNode        | 只要子节点返回FAILURE，就一直调用子节点，最多N次，其中N是作为输入端口传递的。如果子节点返回SUCCESS，则中断该循环，并返回SUCCESS。如果子节点返回RUNNING，也返回RUNNING。 |
| KeepRunningUntilFailure | 该节点总是返回FAILURE (FAILURE in child)或RUNNING (SUCCESS或RUNNING in child)。 |
| Delay | 在指定的时间过后勾选子项。延迟被指定为输入端口的`delay_msec` 。如果子节点返回RUNNING，则该节点也返回RUNNING，并将在Delay节点的下一次勾选中勾选子节点。否则，返回子节点的状态。 |
| RunOnce | 只想执行子进程一次时，则使用本节点。 如果子线程是异步的，它将一直打钩，直到返回SUCCESS或FAILURE。在第一次执行后，可以将输入端口`then_skip` 的值设置为:{【TRUE(默认值)，该节点将在将来被跳过。】或【FALSE，永远同步返回子进程返回的状态。】} |


## 重要概念

### tick()回调函数

```c++
// 可以将最简单的回调封装在BT操作中
NodeStatus HelloTick()
{
  std::cout << "Hello World\n"; 
  return NodeStatus::SUCCESS;
}

// 允许library创建调用HelloTick()的操作
factory.registerSimpleAction("Hello", std::bind(HelloTick));

// (factory 可以创建 Hello 节点的多个实例)
```

### 端口与黑板

- **Ports**(端口))是节点之间用来交换信息的一种机制。
- __Blackboard__(黑板)是一个简单的键/值存储，由树的所有节点共享。
- 端口通过黑板的相同键来连接。
- 一个节点的端口数量、名称和类型必须在编译时知道(C++); 端口之间的连接是在部署时完成的(XML)。

## XML模式

```xml
 <root BTCPP_format="4">
     <BehaviorTree ID="MainTree">
        <Sequence name="root_sequence">
            <SaySomething   name="action_hello" message="Hello"/>
            <OpenGripper    name="open_gripper"/>
            <ApproachObject name="approach_object"/>
            <CloseGripper   name="close_gripper"/>
        </Sequence>
     </BehaviorTree>
 </root>
```

- 树的第一个标签是 `<root>` ，它应包含一或多个 `<BehaviorTree>` 标签。
- 标签 `<root>` 应包含 `[BTCPP_format]` 属性。
- 标签 `<BehaviorTree>` 应有 `[ID]` 属性。
- 每个TreeNode由单个标记表示。特别是:
  - 标记的名称是用于在工厂中注册TreeNode的ID。
  - 属性 `[name]` 指的是实例的名称，并且是可选的。
- 在子节点数量方面 :
  - `ControlNodes` 包含1 ~ N个子节点。
  - `DecoratorNodes` 和 `Subtrees` 只包含1个子树。
  - `ActionNodes` 和 `ConditionNodes` 没有子节点。

### Ports重新映射和指向黑板条目的指针

输入/输出端口可以通过Blackboard的条目名重新映射。

```xml
<!-- eg -->
<root BTCPP_format="4" >
     <BehaviorTree ID="MainTree">
        <Sequence name="root_sequence">
            <SaySomething message="Hello"/> 
            <SaySomething message="{my_message}"/> 
        </Sequence>
     </BehaviorTree>
 </root>
<!-- 序列的第一个子节点输出“Hello” -->
<!-- 第二个子节点读和写黑板中名为“my_message”的条目中包含的值 -->
```

### 紧凑与显式表示

我们称前者为__紧凑语法__，后者__显式语法__。 

```XML
 <SaySomething               name="action_hello" message="Hello World"/>
 <Action ID="SaySomething"   name="action_hello" message="Hello World"/>
```

```XML
<!-- eg -->
<!-- 紧凑语法 -->
 <root BTCPP_format="4" >
     <BehaviorTree ID="MainTree">
        <Sequence name="root_sequence">
            <CheckBattery   name="check_battery"/>
            <OpenGripper    name="open_gripper"/>
            <ApproachObject name="approach_object"/>
            <CloseGripper   name="close_gripper"/>
        </Sequence>
     </BehaviorTree>
 </root>
<!-- 显式语法 -->
<root BTCPP_format="4" >
     <BehaviorTree ID="MainTree">
        <Sequence name="root_sequence">
           <Action ID="SaySomething"   name="action_hello" message="Hello"/>
           <Action ID="OpenGripper"    name="open_gripper"/>
           <Action ID="ApproachObject" name="approach_object"/>
           <Action ID="CloseGripper"   name="close_gripper"/>
        </Sequence>
     </BehaviorTree>
 </root>
```

紧凑语法更方便且易于编写，但提供的关于TreeNode模型的信息太少。像__Groot__这样的工具都需要显式语法或附加信息。可以使用 `<TreeNodeModel>` 标签添加此信息。为了使树的精简版本与Groot兼容，XML 必须修改如下:

```XML
 <root BTCPP_format="4" >
     <BehaviorTree ID="MainTree">
        <Sequence name="root_sequence">
           <SaySomething   name="action_hello" message="Hello"/>
           <OpenGripper    name="open_gripper"/>
           <ApproachObject name="approach_object"/>
           <CloseGripper   name="close_gripper"/>
        </Sequence>
    </BehaviorTree>
    
    <!-- (BT不需要以下内容, 但Groot需要) -->     
    <TreeNodeModel>
        <Action ID="SaySomething">
            <input_port name="message" type="std::string" />
        </Action>
        <Action ID="OpenGripper"/>
        <Action ID="ApproachObject"/>
        <Action ID="CloseGripper"/>      
    </TreeNodeModel>
 </root>
```

### 子树

可以在子树中添加另外一个子树，以避免“复制粘贴”同一棵树到多个位置，以减少复杂性。

假设我们想要将一些动作封装到行为树 AAA_Sub_Tree 中 (属性[name]是可选的，为简单起见省略)。

```XML
 <root BTCPP_format="4" >
 
     <BehaviorTree ID="MainTree">
        <Sequence>
           <Action  ID="SaySomething"  message="Hello World"/>
           <SubTree ID="AAA_Sub_Tree  "/>
        </Sequence>
     </BehaviorTree>
     
     <BehaviorTree ID="AAA_Sub_Tree  ">
        <Sequence>
           <Action ID="OpenGripper"/>
           <Action ID="ApproachObject"/>
           <Action ID="CloseGripper"/>
        </Sequence>
     </BehaviorTree>  
 </root>

```

### 包括外部文件

您可以以类似于C中的`#include `的方式包含外部文件。 我们可以使用标签轻松地做到这一点:

```XML
<include path="路径">
```

对于前面的例子，我们可以将两个行为树拆分为两个文件:

```XML
 <!-- file maintree.xml -->
 <root BTCPP_format="4" >
     <include path="grasp.xml"/>
     <BehaviorTree ID="MainTree">
        <Sequence>
           <Action  ID="SaySomething"  message="Hello World"/>
           <SubTree ID="AAA_Sub_Tree"/>
        </Sequence>
     </BehaviorTree>
  </root>
```

```xml
 <!-- file grasp.xml -->
 <root BTCPP_format="4" >
     <BehaviorTree ID="AAA_Sub_Tree">
        <Sequence>
           <Action ID="OpenGripper"/>
           <Action ID="ApproachObject"/>
           <Action ID="CloseGripper"/>
        </Sequence>
     </BehaviorTree>  
 </root>
```

__ROS用户须知: __如果你想在ROS包中找到一个文件， 你可以使用这个语法:

```XML
<include ros_pkg="name_package"  path="路径/文件名.xml"/>
```

# 基础教程

## 01_你的第一个行为树

### 创建TreeNode

方法一、创建TreeNode的默认(也是推荐的)方式是通过继承。

```cpp
// 继承SyncActionNode(同步节点)创建同步动作结点
class TreeNodename_1 : public BT::SyncActionNode 
{
public:
  TreeNodename_1(const std::string& name) : BT::SyncActionNode(name, {})
  {}
    
  // 方法tick()是实际操作发生的地方。它必须始终返回一个NodeStatus，即 RUNNING、SUCCESS 或 FAILURE。
  BT::NodeStatus tick() override // 必须重载tick函数
  {
    // TreeNode的任何实例都有一个 name 。这个标识符意味着是人类可读，它不需要是唯一的。
    std::cout << "TreeNodename: " << this->name() << std::endl;
    run(); // run()中写要实现的功能。
    return BT::NodeStatus::SUCCESS; // 返回NodeStatus
  }
};
```

方法二、此外也可以用函数指针创建一个TreeNode,函数满足如下形式

```cpp
BT::NodeStatus myFunction(BT::TreeNode& self) 
```

```cpp
// eg
using namespace BT;

// 可以是返回NodeStatus的简单函数
BT::NodeStatus func_1()
{
  run(); // run()中写要实现的功能。
  return BT::NodeStatus::SUCCESS;
}

// 也可以是返回NodeStatus的类方法
class TreeNodename_2
{
public:
  TreeNodename(): a(true) {}
  BT::NodeStatus action() 
  {
    run(); // run()中写要实现的功能。
    return BT::NodeStatus::SUCCESS;
  }
private:
  bool a; // 相关变量
};
```

### CPP文件编写

```cpp
#include "behaviortree_cpp/bt_factory.h"// 行为树头文件
#include "dummy_nodes.h"// TreeNode头文件
int main()
{
  // 创建工厂
  BehaviorTreeFactory factory;
  // 建议通过继承来创建节点
  factory.registerNodeType<TreeNodename_1>("the_X"); 
  // 使用函数指针注册
  factory.registerSimpleCondition("the_Y", [&](TreeNode&) { return func_1(); });
  // 也可以使用类方法创建
  TreeNodename_2 gripper;
  factory.registerSimpleAction("the_Z", [&](TreeNode&){ return gripper.action(); } );
  // 注意：当对象“树”超出范围时，所有树节点都将被销毁
  auto tree = factory.createTreeFromFile("./my_tree.xml");// 树在部署时创建（即在运行时创建，但在开始时仅创建一次）
  // 执行行为树
  tree.tickRoot();
  return 0;
}
```

### XML文件编写

我们必须首先将自定义 TreeNode 注册到 中 BehaviorTreeFactory ，然后从文件或文本加载 XML。

XML 中使用的标识符必须与用于注册 TreeNode 的标识符一致。

```
控制结点（Sequence，Fallback，Parallel，Decorator）在xml创建
执行结点（Action，Condition）在工厂中注册，在xml中调用
```

```xml
 <root>
     <BehaviorTree ID="MainTree"> <!--创建行为树-->
        <Sequence name="root_sequence"> <!--序列节点-->
            <the_X   name="随便起名就行_x"/>
            <the_Y   name="随便起名就行_y"/>
            <the_Z   name="随便起名就行_z"/>
        </Sequence>
     </BehaviorTree>
 </root>
```

## 02_黑板和窗口

- __"黑板"__是一个简单的键/值存储，由树的所有节点共享。
- 黑板上的一个 "条目 "是一个键/值对。
- __输入端口__可以读取黑板上的一个条目，而__输出端口__可以写入一个条目。

**注意**：当自定义 TreeNode 有输入和/或输出端口时，必须在静态方法中声明这些端口：

```cpp
static MyCustomNode::PortsList providedPorts();
```

### 输入端口

> 一个有效的输入有两种方式：
>
> 一个静态字符串，Node将读取并解析它。
>
> 指向黑板上的一个条目的 "指针"，由一个键识别。

- XML语法

```xml
<!--名为message的端口接收静态字符串"hello world"-->
<ActionNode name="first"   message="hello world" /> 
<!--名为message的端口使用条目"greetings"查找黑板上的值-->
<ActionNode name="second" message="{greetings}" /> 
<!--注意：条目"greetings"的值可以（并且可能会）在运行时改变。-->
```

创建TreeNode（输入）

```cpp
class TreeNodename_in : public SyncActionNode
{
public:
  // 如果节点有端口，则必须使用此构造函数签名 
  TreeNodename_in(const std::string& name, const NodeConfig& config)
    : SyncActionNode(name, config)
  { }

  // 有输入或输出端口时，必须在静态方法中声明
  static PortsList providedPorts()
  {
    // InputPort存入Blackboard
    return { InputPort<std::string>("message") }; // message可选，要与xml中保持一致
  }

  // 重写虚函数 tick()    
  NodeStatus tick() override
  {
    Expected<std::string> msg = getInput<std::string>("message");
    // 检查可选项是否有效。如果没有，抛出错误      
    if (!msg){
      throw BT::RuntimeError("missing required input [message]: ", msg.error() );
    }
    // 使用方法value()提取有效消息
    std::cout << "Robot says: " << msg.value() << std::endl;
    return NodeStatus::SUCCESS;
  }
};
```

可以使用模板方法`TreeNode::getInput<T>(key)`来读取端口消息的输入。这个方法可能因为多种原因而失败。由用户来检查返回值的有效性并决定如何处理。

- 返回`NodeStatus::FAILURE`？
- 抛出一个异常？
- 使用一个不同的默认值？

**重要事项：**我们建议在`tick()`中调用`getInput()`函数，而不是在类的构造函数中。

**重要事项：**C++代码应该期望输入的实际值在运行时发生变化，为此，它应该定期更新。

### 输出端口

只有当另一个节点已经在同一条目中写入了“某物”时，指向黑板上的条目输入端口才有效。

```xml
<ActionNode   text="{key}"/>
```

TreeNodename_out使用输出端口将字符串写入条目。

```c++
class TreeNodename_out : public SyncActionNode
{
public:
  // 如果节点有端口，则必须使用此构造函数
  TreeNodename_out(const std::string& name, const NodeConfig& config)
    : SyncActionNode(name, config)
  { }
    
  // 有输入或输出端口时，必须在静态方法中声明
  static PortsList providedPorts()
  {
    // OutputPort从Blackboard中读取
    return { OutputPort<std::string>("text") }; //text可选，要与xml中保持一致
  }

  // 此操作将值写入端口“text”
  NodeStatus tick() override
  {
    // 输出可能每次都发生变化
    std::string op = "The answer is 42"; // 也可以写入别的类型，如自定义类型
    setOutput("text", op);
    run(); // run()中写要实现的功能。
    return NodeStatus::SUCCESS;
  }
};
```

大多数情况下出于调试目的，可以使用名为`Script`的内置操作将静态值写入条目`the_answer` 

```xml
<Script code=" the_answer:='The answer is 42' " />
```

这些端口可以相互连接，因为它们的类型相同，即`std::string`。如果尝试连接不同类型的端口，方法`factory.createTreeFromFile`将抛出异常。

### eg

在这个例子中，包含3个动作节点的序列被执行。

- 动作节点1从一个静态字符串中读取输入`message`。
- 动作节点2在黑板的`the_answer`条目中写了一些东西。
- 动作节点3从黑板的`the_answer`的条目中读取输入`message`。

```xml
<root BTCPP_format="4" >
    <BehaviorTree ID="MainTree">
       <Sequence name="root_sequence">
           <TreeNodename_in     message="hello" />
           <TreeNodename_out    text="{the_answer}"/>
           <TreeNodename_in     message="{the_answer}" />
       </Sequence>
    </BehaviorTree>
</root>
```

```cpp
#include "behaviortree_cpp/bt_factory.h"
// 包含自定义节点的定义的文件
#include "dummy_nodes.h"
using namespace DummyNodes;

int main()
{  
  BehaviorTreeFactory factory;
  factory.registerNodeType<TreeNodename_in>("TreeNodename_in");
  factory.registerNodeType<TreeNodename_out>("TreeNodename_out");

  auto tree = factory.createTreeFromFile("./my_tree.xml");
  tree.tickWhileRunning();
  return 0;
}

/*  Expected output:
  Robot says: hello
  Robot says: The answer is 42
*/
```

我们使用相同的键（`the_answer`）将输出端口与输入端口 连接"起来；换句话说，它们指向黑板的同一个条目。这些端口可以相互连接，因为它们的类型是相同的，即 `std::string`。如果试图连接不同类型的端口，方法 `factory.createTreeFromFile` 会抛出异常。

## 03_含有通用类型的端口

**BT**支持将字符串自动转换为常见类型，如`int`, `long`, `double`, `bool`,`NodeStatus`等，也可以轻松支持__用户自定义类型__。例如:

```c++
struct Position2D { 
  double x;
  double y; 
};
```

### 解析字符串

为了让XML加载器能够从字符串中实例化`Position2D`，需提供`BT::convertFromString<Position2D>(StringView)`的模板类实现。

```c++
// 将字符串转换为Position2D的模板专用化
namespace BT
{
    // StringView是std::string_view的 C++11 版本。可以传递一个std::string或一个const char*。
    template <> inline Position2D convertFromString(StringView str)
    {
        // 用分号将两个数字分开
        auto parts = splitString(str, ';');
        if (parts.size() != 2){
            throw RuntimeError("invalid input)");
        }else{
            Position2D output;
            // 当我们将输入分解成单独的数字时，可以重用特例"convertFromString()"
            output.x = convertFromString<double>(parts[0]);
            output.y = convertFromString<double>(parts[1]);
            return output;
        }
    }
} // end namespace BT
```

### eg

创建两个自定义动作，一个将写入一个端口，另一个将从一个端口读取。

```c++
// 写入端口 "goal"
class CalculateGoal: public SyncActionNode
{
  public:
    CalculateGoal(const std::string& name, const NodeConfig& config): SyncActionNode(name,config)
    {}

    static PortsList providedPorts()
    {
      return { OutputPort<Position2D>("goal") };
    }

    NodeStatus tick() override
    {
      Position2D mygoal = {1.1, 2.3};
      setOutput<Position2D>("goal", mygoal);
      return NodeStatus::SUCCESS;
    }
};

// 读取端口
class PrintTarget: public SyncActionNode
{
  public:
    PrintTarget(const std::string& name, const NodeConfig& config): SyncActionNode(name,config)
    {}

    static PortsList providedPorts()
    {
      // (可选)端口可以具有人类可读的描述
      const char*  description = "Simply print the goal on console...";
      return { InputPort<Position2D>("target", description) };
    }
      
    NodeStatus tick() override
    {
      auto res = getInput<Position2D>("target");
      if( !res ){
        throw RuntimeError("error reading port [target]:", res.error());
      }
      Position2D target = res.value();
      printf("Target positions: [ %.1f, %.1f ]\n", target.x, target.y );
      return NodeStatus::SUCCESS;
    }
};
```

树是一个由4个动作节点组成的序列节点。

- 使用动作`CalculateGoal`在条目`GoalPosition`中存储一个`Position2D`的值。
- 调用`PrintTarget`。输入端口`target`将从黑板条目`GoalPosition`中读取。
- 使用内置的动作脚本，将字符串`"-1;3 "`分配给条目`OtherGoal`。从字符串到`Position2D`的转换将被自动完成。
- 再次调用`PrintTarget`。输入端口`target`将从黑板条目`OtherGoal`中读取。

```c++
// 创建行为树的另一种方法
//将行为树的xml存入到static const char*中
static const char* xml_text = R"(
 <root BTCPP_format="4" >
     <BehaviorTree ID="MainTree">
        <Sequence name="root">
            <CalculateGoal goal="{GoalPosition}" />
            <PrintTarget   target="{GoalPosition}" />
            <Script        code=" OtherGoal:='-1;3' " />
            <PrintTarget   target="{OtherGoal}" />
        </Sequence>
     </BehaviorTree>
 </root>
 )";

int main()
{
  BT::BehaviorTreeFactory factory;
  factory.registerNodeType<CalculateGoal>("CalculateGoal");
  factory.registerNodeType<PrintTarget>("PrintTarget");    
    
  // 注意：FromText 和 FromFile 区别
  auto tree = factory.createTreeFromText(xml_text); // 从static const char*中创建行为树
  auto tree = factory.createTreeFromFile("src/point/config/main_tree.xml"); // 从xml文件中创建行为树
 
  tree.tickRoot();
  //tree.tickRootWhileRunning();// 异步操作 保证树在持续进行直到全部返回成功
  return 0;
}

/* Expected output:
    Target positions: [ 1.1, 2.3 ]
    Converting string: "-1;3"
    Target positions: [ -1.0, 3.0 ]
*/
```

## 04_反应性行为

### 异步

一个__异步动作__(__Asynchronous Action__)有以下要求：

- 它不应该在方法中阻塞`tick()`太多时间。应尽快返回执行流程。
- 勾选后，可能会返回 RUNNING，而不是 SUCCESS 或 FAILURE。
- `halt()`调用该方法时可以尽快停止。

### 状态同步动作节点（StatefulAsyncAction）

StatefulAsyncAction是实现异步动作节点的首选方式。

当你的代码包含**请求-回复模式**时，即当动作向另一个进程发送异步请求，并定期检查是否收到回复时，它特别有用。

根据该答复，它可能返回SUCCESS或FAILURE。

如果你不与外部进程通信，而是进行一些需要很长时间的计算，你可能想把它分成小的 "块"，或者你可能想把该计算转移到另一个线程（见[AsyncThreadedAction](https://link.zhihu.com/?target=https%3A//www.behaviortree.dev/docs/tutorial-advanced/asynchronous_nodes)教程）。

__`StatefulAsyncAction`__的派生类必须包括以下虚构方法，而不是`tick()`。

- **NodeStatus onStart()**：当节点处于IDLE状态时调用。它可能立即成功或失败或RUNNING。在后一种情况下，下次收到tick时，将执行onRunning方法。
- **NodeStatus onRunning()**: 当节点处于RUNNING状态时调用。返回新的状态。
- **void onHalted()**：当这个节点被树上的另一个节点中止时被调用。

让我们创建一个名为`MoveBaseAction`的虚拟节点。

```c++
struct Pose2D
{
    double x, y, theta;
};

namespace chr = std::chrono;

class MoveBaseAction : public BT::StatefulActionNode
{
  public:
    // 如果节点有端口，则必须使用此构造函数
    MoveBaseAction(const std::string& name, const BT::NodeConfig& config)
      : StatefulActionNode(name, config)
    {}
    // 有输入或输出端口时，必须在静态方法中声明
    static BT::PortsList providedPorts()
    {
        return{ BT::InputPort<Pose2D>("goal") };
    }

    // 该函数在开始时调用一次
    BT::NodeStatus onStart() override;

    // 如果onStart()返回RUNNING，将继续调用该方法，直到它返回与RUNNING不同的东西
    BT::NodeStatus onRunning() override;

    // 如果操作被另一个节点中止，则要执行的回调
    void onHalted() override;

  private:
    Pose2D _goal;
    chr::system_clock::time_point _completion_time;
};

//-------------------------
// 类外实现
BT::NodeStatus MoveBaseAction::onStart()
{
  if ( !getInput<Pose2D>("goal", _goal))
  {
    throw BT::RuntimeError("missing required input [goal]");
  }
  printf("[ MoveBase: SEND REQUEST ]. goal: x=%f y=%f theta=%f\n", _goal.x, _goal.y, _goal.theta);

  // 我们使用此计数器来模拟某个动作 假设要完成的时间量（200毫秒）
  _completion_time = chr::system_clock::now() + chr::milliseconds(220);

  return BT::NodeStatus::RUNNING;
}

BT::NodeStatus MoveBaseAction::onRunning()
{
  // 假设我们正在检查是否已收到回复，你不想在这个函数内阻塞太多时间。
  std::this_thread::sleep_for(chr::milliseconds(10)); // 阻塞以判断是否完成 

  // 假设在一段时间后，我们已经完成了操作
  if(chr::system_clock::now() >= _completion_time)
  {
    std::cout << "[ MoveBase: FINISHED ]" << std::endl;
    return BT::NodeStatus::SUCCESS; // 完成操作后返回SUCCESS
  }
  return BT::NodeStatus::RUNNING; // 未完成返回RUNNING
}

void MoveBaseAction::onHalted()
{
  printf("[ MoveBase: ABORTED ]");
}
```

### Sequence VS ReactiveSequence

- Sequence，子进程只会顺序执行一次，当子进程返回RUNNING时，其他进程会被卡住。

- ReactiveSequence，当子进程返回 RUNNING 时，序列将重新启动。

- >详细请看[知乎-BT-反应性行为](https://zhuanlan.zhihu.com/p/581128960)和[官网-BT-反应性行为](https://www.behaviortree.dev/docs/tutorial-basics/tutorial_04_sequence/#sequence-vs-reactivesequence)  

### 事件驱动树?

注意：应该首选`Tree::sleep()`方法，因为当 "有变化 "时，它可以被树中的一个Node打断。

当`TreeNode::emitStateChanged()`方法被调用时，`Tree::sleep()`将被中断。

## 05_使用子树

- __XML中__使用节点**SubTree**实现包含其他子树功能。

```xml
<root BTCPP_format="4">

    <BehaviorTree ID="MainTree">
        <Sequence>
            <!-- 回退节点 -->
            <Fallback>
                <!--Inverter节点是一个Decorator(装饰节点)，它反转其子节点返回的结果-->
                <Inverter>
                    <T0/>
                </Inverter>
                <SubTree ID="DoorClosed"/> <!--使用SubTree包含子树-->
            </Fallback>
            <T1/>
        </Sequence>
    </BehaviorTree>

    <BehaviorTree ID="DoorClosed"> <!--子树-->
        <Fallback>
            <T2/>
            <RetryUntilSuccessful num_attempts="5">
                <T3/>
            </RetryUntilSuccessful>
            <T4/>
        </Fallback>
    </BehaviorTree>
    
</root>
```

- __CPP代码__ (含registerNodes补充、Fallback补充等知识)

> 如果门开着， `T1` 。如果门是关着的，试试`T2`，或者试试`T3`，最多5次，最后试试`T4`。如果`DoorClosed`子树中至少有一个动作成功了，那么`T1`。

```c++
class CrossDoor
{
public:
    void registerNodes(BT::BehaviorTreeFactory& factory);
    BT::NodeStatus T0();
    BT::NodeStatus T1();
    BT::NodeStatus T3();
    BT::NodeStatus T2();
    BT::NodeStatus T4();
private:
    bool _door_open   = false;
    bool _door_locked = true;
    int _pick_attempts = 0;
};

// 使用registerNodes可以同时创建多个子节点，为使用者提供便利
void CrossDoor::registerNodes(BT::BehaviorTreeFactory &factory)
{
  factory.registerSimpleCondition("T0", std::bind(&CrossDoor::a_T0, this));
  factory.registerSimpleAction("T1", std::bind(&CrossDoor::a_T1, this));
  factory.registerSimpleAction("T2", std::bind(&CrossDoor::a_T2, this));
  factory.registerSimpleAction("T3", std::bind(&CrossDoor::a_T3, this));
  factory.registerSimpleCondition("T4", std::bind(&CrossDoor::a_T4, this));
}

int main()
{
  BehaviorTreeFactory factory;
  CrossDoor cross_door;
  // 使用registerNodes创建子节点
  cross_door.registerNodes(factory);

  // 在本例中，单个XML包含多个＜BehaviorTree＞
  // 要确定哪一个是“主要的”，我们应该先注册xml，然后使用其ID分配特定的树
  factory.registerBehaviorTreeFromText(xml_text);
  auto tree = factory.createTree("MainTree");

  // 打印树的helper函数
  printTreeRecursively(tree.rootNode());

  tree.tickWhileRunning();
  return 0;
}
```

## 06_端口重映射

为了避免在非常大的树中发生名称冲突，任何树和子树都使用不同的Blackboard实例。因此，我们需要将树的端口和子树的端口显式连接起来。此重新映射完全在 XML 定义中完成。

### eg_xml

```xml
<root BTCPP_format="4">

    <BehaviorTree ID="MainTree">
        <Sequence>
            <Script code=" move_goal='1;2;3' " />
            <!--target == move_goal,result == move_result 相当于指向了同一个地址-->
            <SubTree ID="MoveRobot" target="{move_goal}" 
                                    result="{move_result}" />
            <SaySomething message="{move_result}"/>
        </Sequence>
    </BehaviorTree>

    <BehaviorTree ID="MoveRobot">  <!--子树 MoveRobot-->
        <Fallback>
            <Sequence>
                <MoveBase  goal="{target}"/>
                <Script code=" result:='goal reached' " />
            </Sequence>
            <ForceFailure>
                <Script code=" result:='error' " />
            </ForceFailure>
        </Fallback>
    </BehaviorTree>

</root>

<!--我们有一个 MainTree ，它包含一个子树 MoveRobot-->
<!--我们要将MoveRobot子树中的端口与MainTree中的其他端口 "连接"（即 "重新映射"）-->
```

### eg_c++

这里没有什么可做的。我们使用`debugMessage`方法来检查黑板的值。

```c++
int main()
{
  BT::BehaviorTreeFactory factory;

  factory.registerNodeType<SaySomething>("SaySomething");
  factory.registerNodeType<MoveBaseAction>("MoveBase");

  factory.registerBehaviorTreeFromText(xml_text);
  auto tree = factory.createTree("MainTree");

  //一直ticking直到最后
  tree.tickWhileRunning();

  // 让我们可视化一些关于黑板当前状态的信息。
  std::cout << "\n------ First BB ------" << std::endl;
  tree.subtrees[0]->blackboard->debugMessage();
  std::cout << "\n------ Second BB------" << std::endl;
  tree.subtrees[1]->blackboard->debugMessage();

  return 0;
}

/*
------ First BB ------
move_result (std::string)
move_goal (Pose2D)

------ Second BB------
[result] remapped to port of parent tree [move_result]
[target] remapped to port of parent tree [move_goal]

*/
```

## 07_如何使用多个XML文件

- 有如下两个`.xml`文件

```xml
<!--subtree_A.xml-->
<root>
    <BehaviorTree ID="SubTreeA">
        <SaySomething message="Executing Sub_A" />
    </BehaviorTree>
</root>
```

```xml
<!--subtree_B.xml-->
<root>
    <BehaviorTree ID="SubTreeB">
        <SaySomething message="Executing Sub_B" />
    </BehaviorTree>
</root>
```

### 手动加载多文件（推荐）

文件**main_tree.xml**，它包括其他两个文件。

```xml
<root>
    <BehaviorTree ID="MainTree">
        <Sequence>
            <SaySomething message="starting MainTree" />
            <SubTree ID="SubTreeA" />
            <SubTree ID="SubTreeB" />
        </Sequence>
    </BehaviorTree>
<root>
```

手动加载多个文件

```c++
int main()
{
  BT::BehaviorTreeFactory factory;
  factory.registerNodeType<DummyNodes::SaySomething>("SaySomething");

  // 在文件夹中查找所有XML文件并注册文件。
  // 我们将使用 std::filesystem::directory_iterator
  std::string search_directory = "./";

  using std::filesystem::directory_iterator;
  for (auto const& entry : directory_iterator(search_directory)) 
  {
    if( entry.path().extension() == ".xml")
    {
      factory.registerBehaviorTreeFromFile(entry.path().string());
    }
  }
  // 在我们的具体情况下是 :
  // factory.registerBehaviorTreeFromFile("./main_tree.xml");
  // factory.registerBehaviorTreeFromFile("./subtree_A.xml");
  // factory.registerBehaviorTreeFromFile("./subtree_B.xml");

  // 您可创建MainTree，它会自动添加子树。
  std::cout << "----- MainTree tick ----" << std::endl;
  auto main_tree = factory.createTree("MainTree");
  main_tree.tickWhileRunning();

  // 或者只创建一个子树
  std::cout << "----- SubA tick ----" << std::endl;
  auto subA_tree = factory.createTree("SubTreeA");
  subA_tree.tickWhileRunning();

  return 0;
}
/* Expected output:

Registered BehaviorTrees:
 - MainTree
 - SubTreeA
 - SubTreeB
----- MainTree tick ----
Robot says: starting MainTree
Robot says: Executing Sub_A
Robot says: Executing Sub_B
----- SubA tick ----
Robot says: Executing Sub_A
```

### 通过`include`添加多个文件

如果你喜欢把要包含的树的信息移到XML本身，你可以按下面的方式修改main_tree.xml。

```xml
<root BTCPP_format="4">
    <include path="./subtree_A.xml" />
    <include path="./subtree_B.xml" />
    <BehaviorTree ID="MainTree">
        <Sequence>
            <SaySomething message="starting MainTree" />
            <SubTree ID="SubTreeA" />
            <SubTree ID="SubTreeB" />
        </Sequence>
    </BehaviorTree>
<root>
```

路径是相对于main_tree.xml的。现在我们可以像往常一样创建树。

```cpp
factory.createTreeFromFile("main_tree.xml")
```

## 08_向节点传递参数

到目前为止，我们所展示的所有的例子中，我们都“强制性”的提供了如下结构的构造函数

```cpp
MyCustomNode(const std::string& name, const NodeConfig& config);
```

在某些情况下，需要将额外的参数、参数、指针、引用等传递给我们类的构造函数。

即使从理论上讲，这些参数**可以**使用输入端口传递，但如果出现以下情况，这将是错误的方法：

- 这些参数在*部署时*（构建树时）是已知的。
- *这些参数在运行时*不会改变。
- 不需要从 XML 设置参数。

如果满足所有这些条件，则强烈建议不要使用端口或黑板。

### 构造函数添加参数（推荐）

考虑下面这个名为**Action_A**的自定义节点。我们想传递两个额外的参数；它们可以是任意复杂的对象，不限于内置类型。

```c++
// Action_A的构造函数与默认构造函数不同。
class Action_A: public SyncActionNode {
public:
    // 传递给构造函数的其他参数
    Action_A(const std::string& name, const NodeConfiguration& config,
             int arg_int, std::string arg_str):
        SyncActionNode(name, config),
        _arg1(arg_int),
        _arg2(arg_str) {}

    // 此示例不需要任何端口
    static PortsList providedPorts() { return {}; }

    // tick（）可以访问私有成员
    NodeStatus tick() override;

private:
    int _arg1;
    std::string _arg2;
};
```

注册该节点并传递已知参数非常简单，如下所示:

```cpp
BT::BehaviorTreeFactory factory;
factory.registerNodeType<Action_A>("Action_A", 42, "hello world");
// 如果希望指定参数模板，则:
// factory.registerNodeType<Action_A, int, std::string>("Action_A", 42, "hello world");
```

```c++
BT::BehaviorTreeFactory factory;

// 节点生成器是一个函数，用于创建std:：unique_ptr＜TreeNode＞。 使用lambdas或std:：bind，我们可以很容易地“注入”额外的参数。
BT::NodeBuilder builder_A =
   [](const std::string& name, const NodeConfiguration& config)
{
    return std::make_unique<Action_A>(name, config, 42, "hello world");
};

// //BehaviorTreeFactory:：registerBuilder是一种更通用的注册自定义节点的方法。
factory.registerBuilder<Action_A>("Action_A", builder_A);
```

### 使用初始化

如果出于任何原因，你需要向一个Node类型的各个实例传递不同的值，你可能要考虑另一种模式。

```c++
class Action_B: public SyncActionNode
{
public:
    // 构造函数看起来和平常一样
    Action_B(const std::string& name, const NodeConfig& config):
        SyncActionNode(name, config) {}

    // 我们希望在第一个tick()之前调用此方法一次
    void initialize(int arg_int, const std::string& arg_str)
    {
        _arg1 = arg_int;
        _arg2 = arg_str;
    }

    // 此示例不要任何端口
    static PortsList providedPorts() { return {}; }

    // tick() 可以访问私有成员
    NodeStatus tick() override;

private:
    int _arg1;
    std::string _arg2;
};
```

我们注册和初始化Action_B的方式是不同的:

```cpp
BT::BehaviorTreeFactory factory;

// 初始化
factory.registerNodeType<Action_B>("Action_B");

// 创建整棵树. 此时Action_B的实例还未初始化
auto tree = factory.createTreeFromText(xml_text);

// visitor 将初始化实例
auto visitor = [](TreeNode* node)
{
  if (auto action_B_node = dynamic_cast<Action_B*>(node))
  {
    action_B_node->initialize(69, "interesting_value");
  }
};

// 将 visitor 应用于树的所有节点
tree.applyVisitor(visitor);
```

## 09_脚本语言

脚本语言允许用户快速读取/写入黑板的变量(即条目)。节点脚本支持的类型是数字（整数和实数）、字符串和注册的 ENUMS。

### 赋值运算符 

运算符:= 和 =都可以用于赋值操作

运算符**“:=”**和**“=”**之间的区别在于，前者可能会在黑板上创建一个新条目（如果该条目不存在），而后者如果黑板不包含该条目，则会抛出异常。

```
param_A := 42
param_B = 3.14
message = 'hello world'
```

还可以使用分号来在单个脚本中添加多个命令。

```text
A:= 42; B:=24
```

### 算数运算符

```
param_A := 7
param_B := 5
param_B *= 2
param_C := (param_A * 3) + param_B
//  param_B 的结果值为10， param_C 的结果值为31。
```

| 操作员 | 分配运算符 | 描述 |
| ------ | ---------- | ---- |
| +      | +=         | 加   |
| -      | -=         | 减   |
| *      | *=         | 乘   |
| /      | /=         | 除   |

注意：加法运算符是唯一适用于`string(用于连接两个字符串)`类型的运算符。

### 位运算符

仅当值可以转换为整数时，这些运算符才起作用。将它们与字符串或实数一起使用将导致异常。

```
value:= 0x7F
val_A:= value & 0x0F
val_B:= value | 0xF0
//  val_A 的值为0x0F(或15); val_B 为0xFF(或255)。
```

| 二元运算符 | 描述     |
| ---------- | -------- |
| \|         | 按位或   |
| &          | 按位与   |
| ^          | 按位异或 |

| 一元运算符 | 描述 |
| ---------- | ---- |
| ～         | 否定 |

### 逻辑和比较运算符

```
val_A := true
val_B := 5 > 3
val_C := (val_A == val_B)
val_D := (val_A && val_B) || !val_C
```

| 运算符     | 描述                        |
| ---------- | --------------------------- |
| true/false | 布尔值。分别可转换为 1 和 0 |
| &&         | 逻辑与                      |
| \| \|      | 逻辑或                      |
| ！         | 否定                        |
| ==         | 平等                        |
| !=         | 不等式                      |
| <          | 较少的                      |
| <=         | 小于等于                    |
| >          | 更大                        |
| >=         | 大于等于                    |

### 三元运算符 ? ：

```
val_B = (val_A > 1) ? 42 : 24
```

### eg

```xml
<!--本例中使用节点脚本来设置变量，并观察是否可以将它们作为SaySomething中的输入端口进行访问。-->
<root BTCPP_format="4">
  <BehaviorTree>
    <Sequence>
      <Script code=" msg:='hello world' " />
      <Script code=" A:=THE_ANSWER; B:=3.14; color:=RED " />
        <Precondition if="A>B && color != BLUE" else="FAILURE">
          <Sequence>
            <SaySomething message="{A}"/>
            <SaySomething message="{B}"/>
            <SaySomething message="{msg}"/>
            <SaySomething message="{color}"/>
        </Sequence>
      </Precondition>
    </Sequence>
  </BehaviorTree>
</root>

<!--我们期望下面的黑板条目包含:-->
<!--msg:字符串"hello world"-->
<!--A:与别名THE_ANSWER对应的整数值-->
<!--B:真实值3.14-->
<!--C:与 enum RED对应的整数值-->

<!--因此，预期的输出是:-->
<!--Robot says: 42.000000-->
<!--Robot says: 3.140000-->
<!--Robot says: hello world-->
<!--Robot says: 1.000000-->
```

```c++
enum Color
{
  RED = 1,
  BLUE = 2,
  GREEN = 3
};

int main()
{
  BehaviorTreeFactory factory;
  factory.registerNodeType<DummyNodes::SaySomething>("SaySomething");

  // 我们可以将枚举添加到脚本语言中
  factory.registerScriptingEnums<Color>();
  // 给标签"THE_ANSWER"赋值
  factory.registerScriptingEnum("THE_ANSWER", 42);

  auto tree = factory.createTreeFromText(xml_text);
  tree.tickWhileRunning();
  return 0;
}
```

## 10_记录器和观察器

### logger(记录器)

BT.CPP 提供了一种在运行时向树添加**记录器的方法，通常是在创建树之后、开始勾选它之前。**

“记录器”是一个类，每次 TreeNode 更改其状态时都会调用回调；它是所谓[观察者模式](https://en.wikipedia.org/wiki/Observer_pattern)的非侵入式实现。

更具体地说，将调用的回调是：

```cpp
  virtual void callback(
    BT::Duration timestamp, // 转换发生时
    const TreeNode& node,   // 更改状态的节点
    NodeStatus prev_status, // 以前的状态
    NodeStatus status);     // 新状态
```

### TreeObserver(观察器)

`TreeObserver`是一个简单的记录器实现，它收集树的每个节点的以下统计信息

```c++
struct NodeStatistics
  {
    NodeStatus last_result; // 最后一个有效结果 (SUCCESS or FAILURE)
    NodeStatus current_status; // 上次状态。可以是任何状态，包括IDLE(空闲)或SKIPPED(跳过)
    unsigned transitions_count; // 计数状态转换，不包括转换到IDLE
    unsigned success_count; // 计数到SUCCESS的转换次数
    unsigned failure_count; // 计数到FAILURE的转换次数
    unsigned skip_count; // 计数到SKIPPED的转换次数
    Duration last_timestamp; // 上次转换的时间戳
  };
```

需要头文件

```c++
#include "behaviortree_cpp/loggers/bt_observer.h" 
```

### 如何唯一标识一个节点

由于观察者允许我们收集特定节点的统计信息，因此我们需要一种方法来唯一标识该节点，可以使用两种机制：

- 这是与树的[深度优先遍历](https://en.wikipedia.org/wiki/Depth-first_search)`TreeNode::UID()`相对应的唯一数字。
- 旨在`TreeNode::fullPath()`成为特定节点的唯一但人为可读的标识符。

我们使用术语“路径”，因为典型的字符串值可能如下所示：

```xml
 first_subtree/nested_subtree/node_name
```

换句话说，路径包含有关子树层次结构中节点位置的信息。

“node_name”可以是在 XML 中分配的名称属性，也可以是使用节点注册后跟“::”和 UID 自动分配的。

### eg_xml

```xml
<root BTCPP_format="4">
  <BehaviorTree ID="MainTree">
    <Sequence>
     <Fallback>
       <AlwaysFailure name="failing_action"/>
       <SubTree ID="SubTreeA" name="mysub"/>
     </Fallback>
     <AlwaysSuccess name="last_action"/>
    </Sequence>
  </BehaviorTree>

  <BehaviorTree ID="SubTreeA">
    <Sequence>
      <AlwaysSuccess name="action_subA"/>
      <SubTree ID="SubTreeB" name="sub_nested"/>
      <SubTree ID="SubTreeB" />
    </Sequence>
  </BehaviorTree>

  <BehaviorTree ID="SubTreeB">
    <AlwaysSuccess name="action_subB"/>
  </BehaviorTree>
</root>
```

您可能注意到有些节点具有`name`属性，但是有些没有。

`UID->fullPath`的对应列表如下：

```
1 -> Sequence::1
2 -> Fallback::2
3 -> failing_action
4 -> mysub
5 -> mysub/Sequence::5
6 -> mysub/action_subA
7 -> mysub/sub_nested
8 -> mysub/sub_nested/action_subB
9 -> mysub/SubTreeB::9
10 -> mysub/SubTreeB::9/action_subB
11 -> last_action
```

### eg_c++

将会做如下操作

- 递归打印树的结构。
- 将 `TreeObserver` 附加到树。
- 输出 `UID / fullPath` 对。
- 收集名为`last_action`的特定节点的统计信息。
- 显示观察者收集的所有统计数据。

```c++
#include "ros/ros.h"
#include "behaviortree_cpp/bt_factory.h"
#include "behaviortree_cpp/loggers/bt_observer.h" 

using namespace BT;

int main()
{
  BT::BehaviorTreeFactory factory;

  factory.registerBehaviorTreeFromText(xml_text); // 具体路径
  auto tree = factory.createTree("MainTree");

  // 辅助打印函数
  BT::printTreeRecursively(tree.rootNode());

  // 创建观察者 观察者的目的是保存一些关于次数的统计数据
  BT::TreeObserver observer(tree);

  // 打印唯一ID和相应的人为可读路径，路径也应是唯一的。
  std::map<uint16_t, std::string> ordered_UID_to_path;
  for(const auto& [name, uid]: observer.pathToUID()) {
    ordered_UID_to_path[uid] = name;
  }

  for(const auto& [uid, name]: ordered_UID_to_path) {
    std::cout << uid << " -> " << name << std::endl;
  }

  tree.tickWhileRunning();

  // 您可以使用完整路径或UID访问特定统计信息
  const auto& last_action_stats = observer.getStatistics("last_action");
  assert(last_action_stats.transitions_count > 0);

  std::cout << "----------------" << std::endl;
  // 打印所有统计信息
  for(const auto& [uid, name]: ordered_UID_to_path) {
    const auto& stats = observer.getStatistics(uid);
    std::cout << "[" << name
              << "] \tT/S/F:  " << stats.transitions_count
              << "/" << stats.success_count
              << "/" << stats.failure_count
              << std::endl;
  }

  return 0;
}
```

## 11_连接到groot2

[tutorial_11_groot2 | BehaviorTree.CPP](https://www.behaviortree.dev/docs/tutorial-basics/tutorial_11_groot2) 

[Groot | BehaviorTree.CPP](https://www.behaviortree.dev/groot)

# 高级教程

## 12_默认端口值

定义端口时，添加默认值可能会很方便，即如果未在 XML 中指定，则端口默认的值。

### 默认输入端口

```c++
  static PortsList providedPorts()
  {
    return { 
      BT::InputPort<Point2D>("input"), //没有默认值，并且必须在 XML 中提供值或黑板条目。
      BT::InputPort<Point2D>("pointA", Point2D{1, 2}, "..."), //默认值1，2
      BT::InputPort<Point2D>("pointB", "3,4",         "..."), //默认值3，4
      //等效BT::InputPort<Point2D>("pointB", Point2D{3, 4}, "...");
      BT::InputPort<Point2D>("pointC", "{point}",     "..."),
      BT::InputPort<Point2D>("pointD", "{=}",         "...") 
      //等效BT::InputPort<Point2D>("pointD", "{pointD}", "...");
    };
  }
```

### 默认输出端口

输出端口更加有限，只能指向黑板条目。当两个名称相同时，您仍然可以使用“{=}”。

```c++
  static PortsList providedPorts()
  {
    return { 
      BT::OutputPort<Point2D>("result", "{target}", "..."); // 默认情况下指向BB条目｛target｝
    };
  }
```

### 在xml中定义默认值

```xml
<SubTree ID="MoveRobot" target="{move_goal}"  frame="world" result="{error_code}" />
<!--我们不想每次都复制并粘贴这三个 XML 属性，除非它们的值不同target,frame,result-->
<!--为了避免这种情况，我们可以在<TreeNodesModel>中定义它们的默认值。-->
<TreeNodesModel>
  <SubTree ID="MoveRobot">
    <input_port  name="target"  default="{move_goal}"/>
    <input_port  name="frame"   default="world"/>
    <output_port name="result"  default="{error_code}"/>
  </SubTree>
</TreeNodesModel>
```

## 13_通过引用访问端口

在某些情况下，可能需要使用**引用语义**，即直接访问存储在 Blackboard 中的对象。当对象是以下情况时，这一点尤其重要：

- 复杂的数据结构
- 复制成本高昂
- 不可复制。

### 方法1_黑板条目作为共享指针

假设我们有这样一个简单的BT:

```xml
 <root BTCPP_format="4" >
    <BehaviorTree ID="SegmentCup">
       <Sequence>
           <AcquirePointCloud  cloud="{pointcloud}"/> 
           <SegmentObject  obj_name="cup" cloud="{pointcloud}" obj_pose="{pose}"/>
       </Sequence>
    </BehaviorTree>
</root>
<!--AcquirePointCloud将写入黑板条目 pointcloud 。-->
<!--SegmentObject将从该条目中读取。-->
```

在本例中，建议使用的端口类型为:

```c++
PortsList AcquirePointCloud::providedPorts()
{
    return { OutputPort<std::shared_ptr<Pointcloud>>("cloud") };
}

PortsList SegmentObject::providedPorts()
{
    return { InputPort<std::string>("obj_name"),
            // 使用共享指针通过引用去访问
             InputPort<std::shared_ptr<Pointcloud>>("cloud"),
             OutputPort<Pose3D>("obj_pose") };
}
```

方法 `getInput` 和 `setOutput` 可以照常使用，并且仍然具有值语义。 但是由于被复制的对象是 `shared_ptr` ，我们实际上访问的是Pointcloud实例的引用。

### 方法二_线程安全的castPtr（推荐）

使用该方法时最值得注意的问题shared_ptr它的线程是不是安全的。

如果自定义异步 Node 有自己的线程，则实际对象可能会同时被其他线程访问。

为了防止这个问题，我们提供了一个包含锁定机制的不同 API。

```c++
PortsList AcquirePointCloud::providedPorts()
{
    return { OutputPort<Pointcloud>("cloud") };
}

PortsList SegmentObject::providedPorts()
{
    return { InputPort<std::string>("obj_name"),
             InputPort<Pointcloud>("cloud"), // 在创建端口时，无需将其包装在std::shared_ptr中
             OutputPort<Pose3D>("obj_pose") };
}
```

通过指针/引用访问Pointcloud的实例:

```c++
//在下面的范围内，只要“any_locked”存在，就有一个互斥对象保护
if(auto any_locked = getLockedPortContent("cloud"))
{
  if(any_locked->empty())
  {
   //黑板上的条目还没有初始化。您可以通过以下操作对其进行初始化：
    any_locked.assign(my_initial_pointcloud);
  }
  else if(Pointcloud* cloud_ptr = any_locked->castPtr<Pointcloud>())
  {
    //成功转换为Pointcloud*（原始类型）。使用cloud_ptr修改点云实例
  }
}
```

## 14_子树模型和自动映射

在[教程6](https://www.behaviortree.dev/docs/tutorial-basics/tutorial_06_subtree_ports)中介绍了子树重映射。当在多个位置使用相同的SubTree时，我们会发现自己需要复制和粘贴相同的长XML标记。例如: `<SubTree ID="MoveRobot" target="{move_goal}"  frame="world" result="{error_code}" />`。我们不想每次都复制粘贴三个XML属性 `target` 、 `frame` 和 `result` ， 除非它们的值不同。为了避免这种情况，我们可以在 `<TreeNodesModel>` 中定义它们的默认值。

```xml
  <TreeNodesModel>
    <SubTree ID="MoveRobot">
      <input_port  name="target"  default="{move_goal}"/>
      <input_port  name="frame"   default="world"/>
      <output_port name="result"  default="{error_code}"/>
    </SubTree>
  </TreeNodesModel>
```

如果在XML中指定，重新映射的黑板条目的值将被覆盖。 在下面的例子中重写了“frame”的值，其他保持默认值。

```xml
<SubTree ID="MoveRobot" frame="map" />
```

### 子树自动重映射

当子树和父树中的条目名称相同时**，**可以使用该属性`_autoremap`。

```xml
<SubTree ID="MoveRobot" target="{target}"  frame="{frame}" result="{result}" />
<!--可以等效替换为-->
<SubTree ID="MoveRobot" _autoremap="true" />
<!--我们仍然可以覆盖特定值，并自动重新映射其他值-->
<SubTree ID="MoveRobot" _autoremap="true" frame="world" />
```

**注意**：该属性`_autoremap="true"`将自动重新映射子树中的**所有**条目，**除非**它们的名称以下划线（字符“_”）开头。这可能是将子树中的条目标记为“私有”的便捷方法。

## 15_模拟和节点替换

在实现集成和单元测试时， 我们希望有一种机制能让我们快速地将特定Node或整个Node类替换为 “测试”版本。从 4.1 版本开始，我们引入了一种称为“替换规则”的新机制，它使这个过程变得更容易。它由类中的附加方法组成，BehaviorTreeFactory这些方法应在注册节点之后和实例化实际树之前调用。

例如，给定XML:

```xml
<SaySomething name="talk" message="hello world"/>
```

我们可以用另一个叫做TestMessage的结点来代替这个结点。相应的替换是用命令完成的:

```cpp
factory.addSubstitutionRule("talk", "TestMessage");
```

第一个参数包含将与 `TreeNode::fullPath` 匹配的通配符字符串。有关fullPath的详细信息，请查看之前的[教程10](https://www.behaviortree.dev/docs/tutorial-basics/tutorial_10_observer). 

### Test节点

 `TestNode` 是一个Action，可以配置为:

- 返回一个特定的状态，SUCCESS或FAILURE。

- 是同步的还是异步的;在后一种情况下，a timeout should be specified(应设定一个超时时间)。
- 后置条件脚本，通常用于模拟OutputPort。

### 完整的实例

在这个例子中，我们将看到如何:

- 用替换法则将一个节点替换为另一个节点。
- 如何使用内置的 `TestNode` 。
- 通配符匹配的例子。
- 如何在运行时使用JSON文件传递这些规则。

__xml__ 

```xml
<root BTCPP_format="4">
  <BehaviorTree ID="MainTree">
    <Sequence>
      <SaySomething name="talk" message="hello world"/>
        <Fallback>
          <AlwaysFailure name="failing_action"/>
          <SubTree ID="MySub" name="mysub"/>
        </Fallback>
        <SaySomething message="before last_action"/>
        <Script code="msg:='after last_action'"/>
        <AlwaysSuccess name="last_action"/>
        <SaySomething message="{msg}"/>
    </Sequence>
  </BehaviorTree>

  <BehaviorTree ID="MySub">
    <Sequence>
      <AlwaysSuccess name="action_subA"/>
      <AlwaysSuccess name="action_subB"/>
    </Sequence>
  </BehaviorTree>
</root>
```

__c++__ 

```c++
int main(int argc, char** argv)
{
  BT::BehaviorTreeFactory factory;
  factory.registerNodeType<SaySomething>("SaySomething");

  // 我们使用lambdas和registerSimpleAction来创建一个“伪”节点，我们想将其替换为给定的节点。
  factory.registerSimpleAction("DummyAction", [](BT::TreeNode& self){
    std::cout << "DummyAction substituting: "<< self.name() << std::endl;
    return BT::NodeStatus::SUCCESS;
  });

  // 旨在替代SaySomething的操作。它将尝试使用输入端口“message”。
  factory.registerSimpleAction("TestSaySomething", [](BT::TreeNode& self){
    auto msg = self.getInput<std::string>("message");
    if (!msg)
    {
      throw BT::RuntimeError( "missing required input [message]: ", msg.error() );
    }
    std::cout << "TestSaySomething: " << msg.value() << std::endl;
    return BT::NodeStatus::SUCCESS;
  });

  // 将“no_sub”作为第一个参数传递以避免添加规则
  bool skip_substitution = (argc == 2) && std::string(argv[1]) == "no_sub";

  if(!skip_substitution)
  {
    // 我们可以使用JSON文件来配置替换规则或者手动操作
    bool const USE_JSON = true;

    if(USE_JSON) {
      factory.loadSubstitutionRuleFromJSON(json_text);
    } else { 
      // 用TestAction替换与此通配符模式匹配的节点
      factory.addSubstitutionRule("mysub/action_*", "TestAction");
	  // 将名称为[talk]的节点替换为TestSaySomething
      factory.addSubstitutionRule("talk", "TestSaySomething");
      // 此配置将传递给TestNode
      BT::TestNodeConfig test_config;
      // 以异步方式转换节点并等待2000毫秒
      test_config.async_delay = std::chrono::milliseconds(2000);
      // 完成后执行此后置条件
      test_config.post_script = "msg ='message SUBSTITUED'";
      // 将名称为[last_action]的节点替换为TestNode，使用test_config配置
      factory.addSubstitutionRule("last_action", test_config);
    }
  }
  factory.registerBehaviorTreeFromText(xml_text);

  // 在树的构建阶段替换，规则将用于实例化测试节点，而不原创的。
  auto tree = factory.createTree("MainTree");
  tree.tickWhileRunning();

  return 0;
}
```

__Json__ 

JSON文件，相当于 `USE_JSON == false` 时执行的分支为:

```json
{
  "TestNodeConfigs": {
    "MyTest": {
      "async_delay": 2000,
      "return_status": "SUCCESS",
      "post_script": "msg ='message SUBSTITUED'"
    }
  },

  "SubstitutionRules": {
    "mysub/action_*": "TestAction",
    "talk": "TestSaySomething",
    "last_action": "MyTest"
  }
}
```

- __TestNodeConfigs__，其中设置了一个或多个TestNode的参数和名称。
- **SubstitutionRules**，其中指定了实际规则。

## 16_global blackboard

正如在前面的教程中所描述的，BT坚持使用“限定范围的黑板”来隔离每个子树，因为它们是编程语言中的独立函数或例程。但在某些情况下，需要拥有一个“全局”黑板，它可以从每个子树直接访问，不需要重新映射。这是有道理的:

- 如[08_向节点传递参数](https://www.behaviortree.dev/docs/tutorial-basics/tutorial_08_additional_args)所述，不能共享的单例对象和全局对象。
- 机器人的全局状态。
- 在行为树之外写入/读取的数据，即在执行tick的主循环中。

### 黑板的层次结构

我们可以像这样实现一个外部的“全局黑板”:

```cpp
auto global_bb = BT::Blackboard::create();
auto maintree_bb = BT::Blackboard::create(global_bb);
auto tree = factory.createTree("MainTree", maintree_bb);
```

实例 `global_bb` 存在于行为树的“外部”，如果对象树被破坏，它将持续存在。此外，可以使用 `set` 和 `get` 方法轻松访问它。

### 如何通过树访问顶层黑板

所谓“顶层黑板”，我们指的是位于根或层次结构的黑板。在上面的代码中， `global_bb` 为顶层黑板。从BT.CPP 4.6版开始，可以通过在条目的名称中添加前缀 `@` 访问顶层黑板，无需重新映射。例如:

```xml
<PrintNumber val="{@value}" />
```

端口val将搜索顶层黑板中的 `value` 条目，而不是本地条目。

### 完整的示例

```xml
  <BehaviorTree ID="MainTree">
    <Sequence>
      <PrintNumber name="main_print" val="{@value}" />
      <SubTree ID="MySub"/>
    </Sequence>
  </BehaviorTree>

  <BehaviorTree ID="MySub">
    <Sequence>
      <PrintNumber name="sub_print" val="{@value}" />
      <Script code="@value_sqr := @value * @value" />
    </Sequence>
  </BehaviorTree>
```

```c++
class PrintNumber : public BT::SyncActionNode
{
public:
  PrintNumber(const std::string& name, const BT::NodeConfig& config)
    : BT::SyncActionNode(name, config)
  {}
  
  static BT::PortsList providedPorts()
  {
    return { BT::InputPort<int>("val") };
  }

  NodeStatus tick() override
  {
    const int val = getInput<int>("val").value();
    std::cout << "[" << name() << "] val: " << val << std::endl;
    return NodeStatus::SUCCESS;
  }
};

int main()
{
  BehaviorTreeFactory factory;
  factory.registerNodeType<PrintNumber>("PrintNumber");
  factory.registerBehaviorTreeFromText(xml_main);

  // 没人能过获得此黑板的所有权
  auto global_bb = BT::Blackboard::create();
  // "MainTree" 将获取 maintree_bb
  auto maintree_bb = BT::Blackboard::create(global_bb);
  auto tree = factory.createTree("MainTree", maintree_bb);

  // 我们可以直接与 global_bb 进行交互
  for(int i = 1; i <= 3; i++)
  {
    // 写入条目 "value"
    global_bb->set("value", i);
    // tick the tree
    tree.tickOnce();
    // 读取条目"value_sqr"
    auto value_sqr = global_bb->get<int>("value_sqr");
    // print 
    std::cout << "[While loop] value: " << i << " value_sqr: " << value_sqr << "\n\n";
  }
  return 0;
}

// 输出
[main_print] val: 1
[sub_print] val: 1
[While loop] value: 1 value_sqr: 1

[main_print] val: 2
[sub_print] val: 2
[While loop] value: 2 value_sqr: 4

[main_print] val: 3
[sub_print] val: 3
[While loop] value: 3 value_sqr: 9
```

Notes: 注:

- 前缀`@`在输入/输出端口或脚本语言中都可以使用。
- 在子树中不需要重新映射。
- 当在主循环中直接访问黑板时，不需要前缀`@`。

# 指南

## 前置条件和后置条件

BT.CPP 4.x 引入了前置条件和后置条件的概念，即可以在节点的实际tick()之前或之后运行的脚本。

所有节点都支持前置条件和后置条件，无需对 C++ 代码进行任何修改。

**注意**：脚本编写的目标不是**编写**复杂的代码，而只是为了提高树的可读性并减少在非常简单的用例中对自定义 C++ 节点的需求。

### 前置条件

| Name           | Description                                                  |
| -------------- | :----------------------------------------------------------- |
| **_skipIf**    | 如果条件为真，则跳过该节点的执行                             |
| **_failureIf** | 如果条件为真，则跳过并返回 FAILURE                           |
| **_successIf** | 如果条件为真，则跳过并返回 SUCCESS                           |
| **_while**     | 与 _skipIf 相同，但如果条件变为 false，也可能会中断正在运行的节点。 |

```xml
<!--以前的写法-->
<Fallback>
    <Inverter>
        <IsDoorClosed/>
    </Inverter>
    <OpenDoor/>
</Fallback>
<!--新的实现-->
<!--如果不使用自定义的<IsDoorClosed/>，而是把布尔值存到条目door_closed中，则可以重写为-->
<OpenDoor _skipIf="!door_closed"/>
```

### 后置条件

| Name           | Description                                   |
| -------------- | --------------------------------------------- |
| **_onSuccess** | 如果节点返回 SUCCESS，则执行此脚本            |
| **_onFailure** | 如果节点返回 FAILURE，则执行此脚本            |
| **_post**      | 如果节点返回 SUCCESS 或 FAILURE，则执行此脚本 |
| **_onHalted**  | 如果正在运行的节点停止则执行脚本              |

```xml
<!--以前的写法-->
<Fallback>
    <Sequence>
        <MoveBase  goal="{target}"/>
        <SetBlackboard output_key="result" value="0" />
    </Sequence>
    <ForceFailure>
        <SetBlackboard output_key="result" value="-1" />
    </ForceFailure>
</Fallback>
<!--新的实现-->
<!--使用后置条件会简单得多。此外，新语法支持enums。-->
<MoveBase goal="{target}" 
          _onSuccess="result:=OK"
          _onFailure="result:=ERROR"/>
```

## 异步操作

### 并发vs并行

__并发__是指两个或多个任务可以在重叠的时间段内启动、运行和完成。 这并不一定意味着它们会同时运行。__并行__是指任务在不同的线程中同时运行，例如在多核处理器上。

__BT__并发地执行所有节点。换句话说:

- Tree执行引擎是单线程的。
- 所有 `tick()` 方法顺序执行。
- 如果任何 `tick()` 方法阻塞，则整个执行流程将被阻塞。

一个需要很长时间执行的Action应该尽快返回RUNNING状态，这就可以通过“并发”和异步执行来实现反应式行为。

这告诉树执行器操作已经开始，需要更多时间才能返回状态SUCCESS或FAILURE。 我们需要再次勾选该节点，以了解状态是否发生了变化(轮询)。

异步节点可以将此长时间执行委托给另一个进程 (使用进程间通信)或另一个线程。

### 异步vs同步

一般来说，异步节点是一个:

- 当勾选时，可能返回RUNNING而不是SUCCESS或FAILURE。
- 可以在调用方法 `halt()` 时尽可能快地停止。通常，方法halt()必须由开发人员实现。

当您的树执行返回RUNNING的异步操作时，该状态通常向后传播，并且整个树被认为处于RUNNING状态。

```c++
// 在此eg中，“ActionE”是异步且正在运行的;当一个节点是RUNNING，通常它的父节点也返回RUNNING。
// 让我们来看一个简单的“睡眠节点”(StatefulActionNode)。

using namespace std::chrono;

// 使用StatefulActionNode作为基类的异步节点示例
class SleepNode : public BT::StatefulActionNode
{
  public:
    SleepNode(const std::string& name, const BT::NodeConfig& config)
      : BT::StatefulActionNode(name, config)
    {}

    static BT::PortsList providedPorts()
    {
      // 我们想要睡眠的毫秒数
      return{ BT::InputPort<int>("msec") };
    }

    NodeStatus onStart() override
    {
      int msec = 0;
      getInput("msec", msec);

      if( msec <= 0 ) {
        return NodeStatus::SUCCESS;
      } else {
        // 一旦到达最后期限，将返回SUCCESS。
        deadline_ = system_clock::now() + milliseconds(msec);
        return NodeStatus::RUNNING;
      }
    }

    /// 该方法由处于RUNNING状态的操作调用。
    NodeStatus onRunning() override
    {
      if ( system_clock::now() >= deadline_ ) {
        return NodeStatus::SUCCESS;
      } else {
        return NodeStatus::RUNNING;
      }
    }

    void onHalted() override
    {
      std::cout << "SleepNode interrupted" << std::endl;
    }

  private:
    system_clock::time_point deadline_;
};
// 第一次勾选SleepNode时，执行 onStart()方法。如果睡眠时间为0，立即返回SUCCESS，否则将返回RUNNING。
// 若返回否则将返回RUNNING，将继续在树的循环中tick并调用方法onRunning()，该方法可能再次返回RUNNING或最终返回SUCCESS。
// 另一个节点可能触发 halt() 信号。在这种情况下，将调用 onHalted() 方法。
```

### 避免阻塞

实现 `SleepNode` 的错误方法是这样的:

```cpp
// 这是Node的同步版本。可能不是我们想要的。
class BadSleepNode : public BT::ActionNodeBase
{
  public:
    BadSleepNode(const std::string& name, const BT::NodeConfig& config)
      : BT::ActionNodeBase(name, config)
    {}

    static BT::PortsList providedPorts()
    {
      return{ BT::InputPort<int>("msec") };
    }

    NodeStatus tick() override
    {  
      int msec = 0;
      getInput("msec", msec);
      // 此操作将冻结整个树
      std::this_thread::sleep_for( milliseconds(msec) );
      return NodeStatus::SUCCESS;
     }

    void halt() override
    {
      // 没人可以调用这个方法，因为树被冻结了。即使这个方法可以执行，也不可能
      // interrupt std:：this_thread:：sleep_for（）
    }
};
```

### 多线程问题

在早期库版本中生成一个新线程看起来是构建异步操作的一个很好的解决方案。但是这是不合适的，因为：

- 以线程安全的方式访问黑板比较困难(稍后会详细介绍)。
- 人们认为这将神奇地使动作“异步”，但他们总是忘记在`halt()`方法被调用时停止线程。

因此不鼓励用户使用 `BT::ThreadedAction` 作为 基类。

```c++
// 让我们再来看看SleepNode。它运用子线程，但是在停止运行是会产生一些问题
class BadSleepNode : public BT::ThreadedAction
{
  public:
    BadSleepNode(const std::string& name, const BT::NodeConfig& config)
      : BT::ActionNodeBase(name, config)
    {}

    static BT::PortsList providedPorts()
    {
      return{ BT::InputPort<int>("msec") };
    }

    NodeStatus tick() override
    {  
	  // 此代码在其自己的线程中运行，因此树仍在运行。这看起来不错，但线程仍然无法中止
      int msec = 0;
      getInput("msec", msec);
      std::this_thread::sleep_for( std::chrono::milliseconds(msec) );
      return NodeStatus::SUCCESS;
    }
    // halt()方法无法终止子线程
};
```

正确的版本应该为:

```c++
// 在此处我会创建自己的线程
class ThreadedSleepNode : public BT::ThreadedAction
{
  public:
    ThreadedSleepNode(const std::string& name, const BT::NodeConfig& config)
      : BT::ActionNodeBase(name, config)
    {}

    static BT::PortsList providedPorts()
    {
      return{ BT::InputPort<int>("msec") };
    }

    NodeStatus tick() override
    {  
      // 此代码在其自己的线程中运行，因此树仍在运行。
      int msec = 0;
      getInput("msec", msec);

      using namespace std::chrono;
      const auto deadline = system_clock::now() + milliseconds(msec);

      // 定期检查isHaltRequested()
      // 只睡一小段时间(1毫秒)
      while( !isHaltRequested() && system_clock::now() < deadline )
      {
        std::this_thread::sleep_for( std::chrono::milliseconds(1) );
      }
      return NodeStatus::SUCCESS;
    }

    // halt()将isHaltRequested()设置为true，并停止子线程中的while()循环
};
```

# 集成ROS

[Integration with ROS2 | BehaviorTree.CPP](https://www.behaviortree.dev/docs/ros2_integration)

# 从V3迁移到V4

## 修复C++

想快速修复 C++ 代码的编译（**即使鼓励重构**），请添加：

```cpp
namespace BT 
{
  using NodeConfiguration = NodeConfig;
  using AsyncActionNode = ThreadedAction;
  using Optional = Expected;
}
```

## xml

您应该将该属性添加到XML 的`BTCPP_format`\<root >标记中：

```xml
<root> <!--以前-->
<root BTCPP_format="4"> <!--现在-->
```

## 脚本语言升级

```xml
<!--以前-->
<SetBlackboard output_key="port_A" value="42" />
<SetBlackboard output_key="port_B" value="69" />
<BlackboardCheckInt value_A="{port_A}" value_B="{port_B}" 
                    return_on_mismatch="FAILURE">
    <MyAction/>
</BlackboardCheckInt>

<!--现在-->
<Script code="port_A:=42; port_B:=69" />
<MyAction _failureIf="port_A!=port_B"/>
```

## 在while循环中Ticking

```cpp
// 简化代码，常见与3.8
while(status != NodeStatus::SUCCESS || status == NodeStatus::FAILURE) 
{
  status tree.tickRoot();
  std::this_thread::sleep_for(sleep_ms);
}
//行为树的“轮询”模型有时会受到批评。睡眠对于避免“繁忙循环”是必要的，但可能会引入一些延迟。
//为了提高行为树的反应性，我们引入了方法
Tree::sleep(std::chrono::milliseconds timeout)
//如果树中的任何节点调用该方法，则可以中断该睡眠的特定实现。这允许循环立即重新勾选树。TreeNode::emitWakeUpSignal
//该方法Tree::tickRoot()已从公共 API 中删除，新推荐的方法是：
// 使用BT::sleep并等待SUCCESS或FAILURE
while(!BT::isStatusCompleted(status)) 
{
  status = tree.tickOnce();
  tree.sleep(sleep_ms);
}
//---- or, even better ------
status = tree.tickWhileRunning(sleep_ms); 
```

`Tree::tickWhileRunning`是新的默认值，它有自己的内部循环；第一个参数是循环内睡眠的超时。

或者，您可以使用以下方法：

- `Tree::tickExactlyOnce()`：相当于 3.8+ 中的旧行为
- `Tree::tickOnce()`大致相当于`tickWhileRunning(0ms)`.它可能会勾选不止一次。

## ControlNodes和 Decorators必须支持 NodeStatus:SKIPPED

当 Node 返回**SKIPPED**时，它通知其父节点（ControlNode 或 Decorator）它尚未执行。

当您实现自己的自定义**叶节点**时，您不应返回**SKIPPED**。此状态是为先决条件保留的。

另一方面，必须修改**ControlNode 和 Decorators以支持这种新状态。**

## 异步控制节点

当 `Sequence`(或`Fallback`) 仅具有同步子级时，整个序列变得“原子”。

换句话说，当“synch_sequence”开始时，就不可能`AbortCondition`停止它。

为了解决这个问题，我们添加了两个新节点，`AsyncSequence`和`AsyncFallback`。

`AsyncSequence`使用时，在执行**每个**同步子级之后、移动到下一个同级之前，将返回**RUNNING 。** 
