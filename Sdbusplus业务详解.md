# Sdbusplus业务详解：从"查一下有几个用户在线"看懂 OpenBMC 服务之间怎么对话

> **这篇文章写给谁**：写过 openUBMC、7 年没碰 C++ 的 Java 程序员。你熟悉的"服务 A 调服务 B 的接口"在这里叫 D-Bus method call，本文就是把这个老心智搬到 OpenBMC 的 C++ 世界。
>
> **本文回答什么问题**：sdbusplus 到底是个啥？一个 D-Bus 调用从发起、到编解码、到对方执行、再返回，完整链路是什么？它和 phosphor-dbus-interfaces、各守护进程（phosphor-*）是怎么分工的？
>
> **讲法说明**：每个技术点都【先讲】真实场景 / 真实代码，再【再点】概念。行内 `相对路径/文件:行号` 是源码锚点，可直接翻代码核对。C++ 生疏的地方用 Java 类比（RAII≈try-with-resources、智能指针≈显式所有权的引用、模板≠泛型、异步回调≈Netty Future）。术语一律用代码里的原名，不翻译。
>
> **路径说明**：以下路径都相对 `sdbusplus` 仓库根目录；`include/` 是头文件，`src/` 是实现，`example/` 是可编译例子，`tools/` 是 YAML→C++ 的代码生成器。本文只讲业务怎么用，不碰环境搭建。

---

## 0. 一句话说清业务

**sdbusplus 就是 OpenBMC 里各个服务之间说 D-Bus 的"共同语言 / 翻译层"**：它把 systemd 的 C 语言 `sd-bus` API 包成现代 C++，让你用几行类型安全的代码就能"发个消息、调个方法、读个属性、监听个信号"，而不用手写 `sd_bus_message_append(...)` 那套 C 样板。

把 openUBMC 的心智搬过来：

| openUBMC 里你熟悉的东西 | sdbusplus 里对应的东西 | 源码锚点 |
| --- | --- | --- |
| 每个 BMC 服务对外暴露"接口 + 方法 + 属性 + 信号" | 一样，只是接口定义写在 `*.interface.yaml` 里 | `include/yaml/net/poettering/Calculator.interface.yaml:5` |
| 用服务名 + 对象路径 + 接口名定位一个远程对象 | `bus.new_method_call(service, path, interface, method)` | `include/sdbusplus/bus.hpp:290` |
| 框架帮你做序列化 / 反序列化 | C++ 模板在**编译期**根据类型生成 D-Bus 签名字符串 | `include/sdbusplus/message/types.hpp:40` |
| 客户端调服务端方法，走组件 / RPC | 走 `sd_bus_call`，由 dbus-daemon 转发 | `include/sdbusplus/bus.hpp:343` |
| 监听某个服务上线 / 属性变化 | 注册 match rule，信号来了回调 / `co_await` | `include/sdbusplus/bus/match.hpp:51` |
| 用 busctl 看对象树 | 服务端装 ObjectManager | `include/sdbusplus/server/manager.hpp:27` |

仓库自己怎么定位这件事，README 写得很直白：sdbusplus 包含两部分——一个 C++ 库 libsdbusplus，一个代码生成工具 sdbus++（`README.md:3-8`）；这个库"构建在 sd-bus 之上，尽量轻量，通常编译出来就是你本来要写的那些 sd-bus API 调用，同时提供编译期类型安全和内存泄漏防护"（`README.md:42-46`）。

更关键的一句在 `README.md:72-77`：库刻意模仿 sd-bus 的命名——`sd_bus_call` 变成 `sdbusplus::bus::call`、`sd_bus_get_unique_name` 变成 `sdbusplus::bus::get_unique_name`、`sd_bus_message_get_signature` 变成 `sdbusplus::message::get_signature`。**所以在 sdbusplus 里看到的几乎每个方法，背后都有一个同名的 sd-bus C 函数**。这是全文最重要的阅读技巧：C++ 这层是"薄封装"，真正干活的是 sd-bus。

> 【再点】概念：D-Bus 是 Linux 上进程间通信（IPC）的一套总线；systemd 提供了 C 实现 `sd-bus`，sdbusplus 是它的 C++ 包装。OpenBMC 生态里，**每个 phosphor-* 守护进程都是 D-Bus 上的一个"服务"，通过对象路径暴露接口**，这就是你在 openUBMC 里已经熟悉的那套东西换了个语言实现。
>
> D-Bus 寻址靠四元组：**service（服务名）→ object path（对象路径）→ interface（接口）→ member（方法或信号名）**。这四样东西在 `new_method_call` 的四个参数里一一对应（`include/sdbusplus/bus.hpp:290-291`）。记住它，后面所有场景都是这个四元组的排列组合。

---

## 1. 真实场景一：客户端查"系统里有几个用户登录"

先看最完整、最短的一条链路——仓库自带的 `example/list-users.cpp`（在 `example/meson.build:1` 注册为可执行程序）。它调用 systemd login1 服务的 `ListUsers` 方法，拿到在线用户列表。

### 1.1 真实代码

```cpp
// example/list-users.cpp:11-31
int main()
{
    using namespace sdbusplus;

    auto b = bus::new_default_system();                    // ① 建连接
    auto m =
        b.new_method_call("org.freedesktop.login1", "/org/freedesktop/login1",
                          "org.freedesktop.login1.Manager", "ListUsers");     // ② 造消息
    auto reply = b.call(m);                                // ③ 发出去，拿回包

    using return_type =
        std::vector<std::tuple<uint32_t, std::string, object_path>>;
    auto users = reply.unpack<return_type>();              // ④ 解包

    for (auto& user : users)
    {
        std::cout << std::get<std::string>(user) << "\n";  // 打印用户名
    }
    return 0;
}
```

注意，`README.md:50-63` 里有几乎一样的代码片段——仓库把它当作"库长什么样"的头号示例。也就是说：**这 30 行就是 sdbusplus 想卖给你的全部体验**。

### 1.2 完整链路拆解（① ② ③ ④）

**① 建连接 `bus::new_default_system()`**
- `example/list-users.cpp:15` 调用的工厂函数定义在 `include/sdbusplus/bus.hpp:587`，转到 `bus_t::new_default_system()`，实现在 `src/bus.cpp:48-57`：底层调 `sd_bus_default_system(&b)`，失败就抛 `exception::SdBusError`。
- 【先讲】这行代码就是"我要连上系统总线"。【再点】`bus_t` 是连接的持有者，内部一个 `unique_ptr<sd_bus, BusDeleter>`（`include/sdbusplus/bus.hpp:558-559`），析构时自动 unref。
- **Java 类比**：把它想成 `Connection conn = dataSource.getConnection()` 或一个 Netty 的 `Channel`——拿到它才能收发。`BusDeleter`（`include/sdbusplus/bus.hpp:29-43`）在析构时调用 `sd_bus_unref`，就是 Java 里 try-with-resources 帮你关连接的那部分，只不过 C++ 用 RAII 自动做，不需要你写 `finally`。

**② 造消息 `new_method_call(service, path, interface, method)`**
- `example/list-users.cpp:16-18` 传入四个字符串：目标服务 `org.freedesktop.login1`、对象路径 `/org/freedesktop/login1`、接口 `org.freedesktop.login1.Manager`、方法 `ListUsers`。
- 实现在 `include/sdbusplus/bus.hpp:290-302`：调 `sd_bus_message_new_method_call`，成功后把裸 `sd_bus_message*` 包成 `message_t` 返回。有个重载接受 `object_path` 类型（`include/sdbusplus/bus.hpp:304-309`），内部还是转成 `.str.c_str()`。
- 【先讲】这一步就是"填一张信封"：发给谁、发到哪个对象、用哪个接口、调哪个方法。【再点】这四个字段是 D-Bus 寻址的全部要素，和你在 openUBMC 里定位一个服务接口是同一个思路。
- **还要填参数**：如果方法要入参，用 `message.append(args...)`（`include/sdbusplus/message.hpp:138-143`），它转发到 `message::append`，由模板推导签名。真实例子见 `example/asio-example.cpp:109`（`method.append(netFn, lun, cmd, commandData, options)`）。
- **Java 类比**：`message_t` 像一个请求 DTO / `HttpRequest`，`new_method_call` 是构造函数，`append` 相当于 `setBody`/`addParameter`。

**③ 发出去并等回复 `call(m)`**
- `example/list-users.cpp:19`，实现在 `include/sdbusplus/bus.hpp:343-355`：调 `sd_bus_call`，返回一个新的 `message_t` 作为应答。
- 注意 `include/sdbusplus/bus.hpp:332-337` 的注释特别说明：**这个方法抛出的错误既包括底层 D-Bus 错误，也包括对方返回的 METHOD_ERROR，所以你不需要再检查 `is_method_error()`**。对比 `example/asio-example.cpp:311-318` 的异步版本还得手动判断 `ret.is_method_error()`——同步版把错误处理"吞"进了异常。
- 【先讲】这一步是"把信封寄出去，并阻塞等回信"。【再点】`sd_bus_call` 是同步阻塞调用，由 dbus-daemon 根据服务名找到 login1 进程，把消息投递过去。
- **Java 类比**：一次同步 RPC（Feign / Dubbo 调用），`call()` 抛异常 ≈ RPC 框架把远端异常包装成本地异常抛出。

**④ 解包 `reply.unpack<return_type>()`**
- `example/list-users.cpp:21-23` 声明返回类型 `std::vector<std::tuple<uint32_t, std::string, object_path>>`，`unpack` 在 `include/sdbusplus/message.hpp:162-189`：0 个类型返回 void，1 个类型直接返回该类型，多个类型返回 `std::tuple`。
- `unpack` 内部调 `read`（`include/sdbusplus/message.hpp:150-155`），再转发给 `sdbusplus::message::read`，最终由编译期算出的签名驱动 `sd_bus_message_read_basic`（`include/sdbusplus/sdbus.hpp:496`）。
- 【先讲】这一步是"按约定的格式拆开回信"。【再点】签名（signature）是 D-Bus 的类型描述串，比如 `a(us o)` 表示"数组，元素是无符号整数+字符串+对象路径"。sdbusplus 的厉害之处：**你只写 C++ 类型，签名字符串在编译期由模板算出来**（`include/sdbusplus/message/types.hpp:40`），不用手写 `a(uso)`。
- **Java 类比**：`unpack<T>()` ≈ Jackson 的 `readValue(json, TypeReference<T>)`。但 Java 泛型是运行期擦除、靠反射；C++ 模板是编译期展开、无运行时开销——这就是"模板≠泛型"。

### 1.3 这条链路暴露的三个机制

1. **RAII ≈ try-with-resources**：`bus_t` 和 `message_t` 析构时自动释放底层对象（`include/sdbusplus/bus.hpp:73`、`include/sdbusplus/message.hpp:50-51`），不用手动 `free`。`message_t` 甚至可拷贝——拷贝构造会 `sd_bus_message_ref` 增加引用计数（`include/sdbusplus/message.hpp:106-108`），类似 Java 里多个引用指向同一对象。
2. **编译期类型映射**：类型表在 `include/sdbusplus/message/types.hpp:180-230`，例如 `std::string → SD_BUS_TYPE_STRING`（`include/sdbusplus/message/types.hpp:220`）、`object_path → SD_BUS_TYPE_OBJECT_PATH`（`include/sdbusplus/message/types.hpp:226`）、`uint64_t → SD_BUS_TYPE_UINT64`（`include/sdbusplus/message/types.hpp:201`）。注释还写明 D-Bus 不支持 `int8_t` 和 `float`（`include/sdbusplus/message/types.hpp:187`、`include/sdbusplus/message/types.hpp:206`）。
3. **薄封装**：每个 C++ 方法基本对应一个 `sd_bus_*` C 函数（对照 `README.md:72-77`）。查不懂时去翻 systemd 的 manpage——这是读这个库最省力的办法。

---

## 2. 真实场景二：写一个 Calculator 服务，注册方法、属性、信号

场景一是"调别人"，场景二是"自己当服务端"。仓库自带的 `example/calculator-server.cpp`（在 `example/meson.build:46-52` 注册）完整演示了写一个 D-Bus 服务要做的事。

### 2.1 先看接口定义（YAML 是源头）

服务不是凭空写的，接口先定义在 YAML 里。Calculator 的接口在 `include/yaml/net/poettering/Calculator.interface.yaml`：

```yaml
# include/yaml/net/poettering/Calculator.interface.yaml:5-48（节选，省略描述文字）
methods:
    - name: Multiply                  # 方法 Multiply
      parameters:
          - name: x
            type: int64
          - name: y
            type: int64
            default: 1
      returns:
          - name: z
            type: int64
    - name: Divide                    # 方法 Divide
      errors:
          - self.DivisionByZero       # 会抛 DivisionByZero 错误
    - name: Clear                     # 方法 Clear
      flags:
          - unprivileged
          - no_reply
properties:                           # 属性
    - name: LastResult
      type: int64
      default: 0
    - name: Status
      type: enum[self.State]
      flags:
          - const
    - name: Owner
      type: string
signals:                              # 信号
    - name: Cleared
      properties:
          - type: int64
paths:
    - instance: /net/poettering/calculator
service_names:
    - default: net.poettering.Calculator
```

【先讲】这个 YAML 就是"服务的说明书"：有哪些方法、哪些属性、哪些信号、挂在哪个路径、服务名叫什么。【再点】概念：sdbus++ 代码生成器读这个 YAML，生成 C++ 基类；你写业务类继承它、实现对应方法即可。这就是 README 说的"用一个小 YAML 定义接口，工具生成实现接口的 C++ 类，类里每个方法和属性都有虚函数供你重写"（`README.md:83-92`）。

错误定义单独写在 `include/yaml/net/poettering/Calculator.events.yaml:3-14`：`DivisionByZero`（errno `EDOM`，`include/yaml/net/poettering/Calculator.events.yaml:4-8`）和 `PermissionDenied`（errno `EPERM`，`include/yaml/net/poettering/Calculator.events.yaml:10-14`）。【再点】概念：README `README.md:94-97` 说明有两种 YAML——interface 用来生成服务端/客户端接口，error 用来生成能自动转成 D-Bus 错误响应的 C++ 异常。

生成器怎么把这些变成代码？看模板最直观：`tools/sdbusplus/templates/interface.common.hpp.mako:27-29` 把接口名生成 `static constexpr auto interface = "..."`，`path` 和 `service_names` 生成 `static constexpr` 常量（`tools/sdbusplus/templates/interface.common.hpp.mako:109-114`）——这正是例子里 `Calculator::instance_path` 和 `Calculator::default_service` 的来源；`tools/sdbusplus/templates/interface.server.hpp.mako:17-19` 生成服务基类，`tools/sdbusplus/templates/interface.server.hpp.mako:41-45` 生成"把 vtable 和 this 注册进去"的构造函数。

### 2.2 真实代码：业务类 + main

```cpp
// example/calculator-server.cpp:8-49（节选）
using Calculator_inherit =
    sdbusplus::server::object_t<sdbusplus::server::net::poettering::Calculator>;

struct Calculator : Calculator_inherit
{
    Calculator(sdbusplus::bus_t& bus, const char* path) :
        Calculator_inherit(bus, path)
    {}

    int64_t multiply(int64_t x, int64_t y) override          // 实现 Multiply
    {
        return lastResult(x * y);
    }

    int64_t divide(int64_t x, int64_t y) override            // 实现 Divide
    {
        using sdbusplus::error::net::poettering::Calculator::DivisionByZero;
        if (y == 0)
        {
            status(State::Error);
            throw DivisionByZero();                          // 抛生成的异常
        }
        return lastResult(x / y);
    }

    void clear() override                                   // 实现 Clear
    {
        auto v = lastResult();
        lastResult(0);
        cleared(v);                                          // 发 Cleared 信号
    }
};
```

```cpp
// example/calculator-server.cpp:51-66
int main()
{
    auto b = sdbusplus::bus::new_default();                      // ① 建连接
    sdbusplus::server::manager_t m{b, Calculator::instance_path}; // ② 装 ObjectManager
    b.request_name(Calculator::default_service);                // ③ 占服务名
    Calculator c1{b, Calculator::instance_path};                // ④ 造对象（注册到总线）
    b.process_loop();                                           // ⑤ 事件循环
}
```

### 2.3 完整链路拆解（① ② ③ ④ ⑤）

**① 建连接 `bus::new_default()`**
- `example/calculator-server.cpp:55`。和场景一的 `new_default_system()` 不同，`new_default()` 根据当前用户选总线：root 用 system bus，普通用户用 session bus（`include/sdbusplus/bus.hpp:84-104` 的注释，`src/bus.cpp:24-33`）。OpenBMC 守护进程以 root 跑，实际上就是 system bus。

**② 装 ObjectManager `manager_t m{b, path}`**
- `example/calculator-server.cpp:56`。`manager_t` 是 `manager::manager` 的别名（`include/sdbusplus/server/manager.hpp:57`），构造函数在 `include/sdbusplus/server/manager.hpp:46-48`，底层调 `sd_bus_add_object_manager`（`include/sdbusplus/server/manager.hpp:27-32`）。
- 【先讲】这一步让外部客户端能遍历"这个服务下面有哪些对象、哪些接口"。【再点】概念：ObjectManager 是 D-Bus 标准接口，`busctl tree` 之类的工具就靠它列出对象树。你在 openUBMC 里用 busctl 看对象树，背后的对象就是这么注册上去的。

**③ 占服务名 `request_name(Calculator::default_service)`**
- `example/calculator-server.cpp:59`。`default_service` 是生成器根据 YAML 的 `service_names`（`include/yaml/net/poettering/Calculator.interface.yaml:94-96`）生成的常量，值就是 `net.poettering.Calculator`（生成逻辑见 `tools/sdbusplus/templates/interface.common.hpp.mako:109-114`）。
- 实现在 `include/sdbusplus/bus.hpp:270-279`：调 `sd_bus_request_name`，带 `SD_BUS_NAME_ALLOW_REPLACEMENT | SD_BUS_NAME_REPLACE_EXISTING` 标志（`include/sdbusplus/bus.hpp:274`）。
- 【先讲】这一步是"向总线宣布：net.poettering.Calculator 这个名字归我了"。【再点】概念：服务名必须唯一，客户端才能通过它找到你。

**④ 造对象 `Calculator c1{b, path}`**
- `example/calculator-server.cpp:62`。`Calculator` 继承 `object_t<...Calculator>`（`example/calculator-server.cpp:8-9`），构造时调用 `object_t` 的构造函数（`include/sdbusplus/server/object.hpp:76-85`），它把接口和 vtable 挂到对象路径上。
- `object_t` 是 `object::object<Args...>` 的别名（`include/sdbusplus/server/object.hpp:156-158`）。它的设计是用多重继承把多个 D-Bus 接口组合成一个对象（`include/sdbusplus/server/object.hpp:26-37` 注释）；构造函数接受一个 `action` 参数控制发哪种 added 信号（`include/sdbusplus/server/object.hpp:57-68`），默认 `emit_object_added`（`include/sdbusplus/server/object.hpp:77`）。
- 【先讲】这一步是"我真的在 `/net/poettering/calculator` 上创建了一个对象，外界现在能调它的 Multiply/Divide/Clear、读写 LastResult、收到 Cleared 信号"。【再点】概念：`object_t` 用模板组合"接口"和"对象信号"，构造时通过 `add_object_vtable` 注册方法表，析构时自动发 `sd_bus_emit_object_removed`（`include/sdbusplus/server/object.hpp:92-100`）。
- **vtable（方法表）**是整个服务端的枢纽。它是一张 `sd_bus_vtable` 数组，由 `vtable::start()` / `method_o()` / `property_o()` / `signal()` / `end()` 这些工厂函数拼出来（声明在 `include/sdbusplus/vtable.hpp:13-101`）。`asio::dbus_interface::initialize()` 里能看到完整的拼装过程：`vtable_.emplace_back(vtable::start())` → 逐个 property → 逐个 method → 逐个 signal → `end()`（`include/sdbusplus/asio/object_server.hpp:747-787`），最后 `server::interface_t` 拿这张表注册（`include/sdbusplus/asio/object_server.hpp:789-792`）。
- **Java 类比**：`object_t` ≈ 用 `@RestController` 注册的一个 bean；vtable ≈ Spring MVC 的 handler mapping——请求来了，框架按方法名找到你写的 `override` 函数并调用。

**⑤ 事件循环 `b.process_loop()`**
- `example/calculator-server.cpp:65`。实现是死循环 `process_discard()` + `wait()`（`include/sdbusplus/bus.hpp:257-264`）：先处理已排队的消息，再阻塞等新消息。
- 【先讲】这一步是"服务不退出，一直等着别人来调"。【再点】概念：D-Bus 服务必须跑一个事件循环来收发消息，这是所有 phosphor-* 守护进程的共同骨架。
- **Java 类比**：≈ Netty 的 `eventLoop.run()` 或 `while(true){ take(); handle(); }` 的消息循环。

### 2.4 方法里的两个细节

- **抛异常变错误响应**：`divide()` 里 `y==0` 时 `throw DivisionByZero()`（`example/calculator-server.cpp:35`）。这个异常类由生成器根据 `include/yaml/net/poettering/Calculator.events.yaml:4-8` 生成，最终变成 D-Bus METHOD_ERROR 回复给调用方。客户端 `co_await` 时会收到异常（见 `example/coroutine-example.cpp:62-70` 的 catch 例子）。
- **属性赋值连带发信号**：`multiply()` 里 `lastResult(x*y)`（`example/calculator-server.cpp:22`）不只是改内存值：生成器对没标 `const`/`readonly` 的属性默认加 `vtable::property_::emits_change` 标志（`tools/sdbusplus/templates/property.server.vtable.cpp.mako:7-8`），所以生成的 setter 会顺带发出 `PropertiesChanged` 信号。同理 `clear()` 里 `cleared(v)`（`example/calculator-server.cpp:46`）发的是接口自定义的 `Cleared` 信号（定义在 `include/yaml/net/poettering/Calculator.interface.yaml:69-77`）。

---

## 3. 真实场景三：现代写法——协程客户端和异步服务端

前两个场景用的是"经典" API（同步 `call` + `process_loop`）。仓库还提供两套更现代的写法，OpenBMC 新代码里很常见。

### 3.1 协程客户端（`calculator-client.cpp`，注册于 `example/meson.build:62-68`）

```cpp
// example/calculator-client.cpp:6-26（节选）
auto startup(sdbusplus::async::context& ctx) -> sdbusplus::async::task<>
{
    using Calculator = sdbusplus::client::net::poettering::Calculator<>;
    auto c = Calculator(ctx)
                 .service(Calculator::default_service)
                 .path(Calculator::instance_path);

    auto _ = co_await c.multiply(7, 6);   // 像写同步代码一样发异步调用
    std::cout << "Should be 42: " << _ << std::endl;
}
// example/calculator-client.cpp:72-82
int main()
{
    sdbusplus::async::context ctx;
    ctx.spawn(startup(ctx));
    ctx.run();
    return 0;
}
```

【先讲】`co_await c.multiply(7,6)` 看起来像同步调用，实际是异步的：调用发出后协程挂起，回复到了再继续。后面的 `last_result()` / `last_result(1234)` / `properties()` 同样是 `co_await`（`example/calculator-client.cpp:30`、`example/calculator-client.cpp:47`、`example/calculator-client.cpp:64`）——读属性、写属性、批量读属性都用统一的 await 风格。
【再点】概念：`sdbusplus::async::context` 是运行循环（run-loop），职责见 `include/sdbusplus/async/context.hpp:25-42`；内部有 caller 和 worker 两个线程，caller 做 `sd_bus_wait`、worker 做 `sd_bus_process`（`include/sdbusplus/async/context.hpp:33-41`）。`co_await` 由 C++20 协程实现，`sdbusplus::async::task` 是 `exec::task` 的别名（`include/sdbusplus/async/task.hpp:11`）。

**Java 类比**：`co_await` ≈ Kotlin 的 `suspend` / Java 里对 `CompletableFuture` 的阻塞式取结果，但 C++ 协程是编译器状态机、无额外线程开销；`ctx.spawn(...)` ≈ 提交给 `ExecutorService`，`ctx.run()` ≈ 启动事件循环。

### 3.2 异步服务端（`calculator-aserver.cpp`，注册于 `example/meson.build:54-60`）

```cpp
// example/calculator-aserver.cpp:6-20（节选）
class Calculator :
    public sdbusplus::aserver::net::poettering::Calculator<Calculator>
{
  public:
    auto method_call(multiply_t, auto x, auto y)
    {
        auto r = x * y;
        last_result(r);
        return r;
    }
    auto method_call(divide_t, auto x, auto y)
        -> sdbusplus::async::task<divide_t::return_type>
    {
        if (y == 0) { throw DivisionByZero(); }
        co_return x / y;
    }
    bool set_property(owner_t, auto owner)              // 属性 setter 也可重写
    {
        std::swap(properties.owner, owner);
        return true;
    }
};
```

【先讲】方法不再用 `override` 虚函数，而是写成 `method_call(multiply_t, ...)` 这样的重载：第一个参数是生成的"方法标签"类型（`multiply_t`），后面才是入参；属性用 `get_property(owner_t)` / `set_property(owner_t, ...)`（`example/calculator-aserver.cpp:45`、`example/calculator-aserver.cpp:51`）。
【再点】概念：这是 CRTP（奇异递归模板模式）+ tag dispatch：生成器为每个方法生成一个 tag 类型，服务端用重载区分；CRTP 体现在 `Calculator<Calculator>` 这种"把自己当模板参数传进去"的写法（`example/calculator-aserver.cpp:7`）。相比经典版的虚函数，这套是编译期绑定。

### 3.3 asio 集成：把 D-Bus 装进 Boost.Asio 的事件循环

`example/asio-example.cpp`（注册于 `example/meson.build:18-30`）展示了最贴近 OpenBMC 实际代码的一层——`sdbusplus::asio`。

```cpp
// example/asio-example.cpp:195-206（节选）
int server()
{
    boost::asio::io_context io;
    auto conn = std::make_shared<sdbusplus::asio::connection>(io);
    conn->request_name("xyz.openbmc_project.asio-test");
    auto server = sdbusplus::asio::object_server(conn);
    std::shared_ptr<sdbusplus::asio::dbus_interface> iface =
        server.add_interface("/xyz/openbmc_project/test",
                             "xyz.openbmc_project.test");
    // 注册属性、方法……
    iface->initialize();
    io.run();
}
```

【先讲】这里的主角不再是 `bus_t`，而是 `sdbusplus::asio::connection`——它**继承** `sdbusplus::bus_t`（`include/sdbusplus/asio/connection.hpp:54`），把 D-Bus 的文件描述符塞进 Boost.Asio 的 `io_context` 统一调度（`include/sdbusplus/asio/connection.hpp:58-65` 构造函数里创建 `stream_descriptor` 和 `steady_timer`）。
【再点】概念：这样 D-Bus 消息、定时器、socket 都在同一个事件循环里跑，不用自己写 `process_loop`。`asio::object_server::add_interface`（`include/sdbusplus/asio/object_server.hpp:855-861`）动态创建接口，再 `register_property_r` / `register_property_rw` / `register_method`（`include/sdbusplus/asio/object_server.hpp:408`、`include/sdbusplus/asio/object_server.hpp:451`、`include/sdbusplus/asio/object_server.hpp:593`）注册，最后 `initialize()`（`include/sdbusplus/asio/object_server.hpp:740-803`）构造 vtable 并挂到总线。析构时 `dbus_interface` 会自动发 `InterfacesRemoved`（`include/sdbusplus/asio/object_server.hpp:402-406`）。

**Java 类比**：`boost::asio::io_context` ≈ Netty 的 `EventLoopGroup`；`sdbusplus::asio::connection` ≈ 把 D-Bus 适配成 Netty `Channel` 的适配器；`async_method_call(handler)`（`include/sdbusplus/asio/connection.hpp:210-219`）≈ 传回调的异步 RPC，等价于 `CompletableFuture.whenComplete(...)`；`yield_method_call`（`include/sdbusplus/asio/connection.hpp:257-291`）配合 `boost::asio::yield_context` ≈ 用协程把异步写成同步风格。

---

## 4. 真实场景四：读写属性（Properties.Get / Set / GetAll）

属性是 OpenBMC 用得最多的东西——传感器读数、电源状态、固件版本全是属性。看服务端怎么注册、客户端怎么读写。

### 4.1 服务端注册属性

```cpp
// example/register-property.cpp:23-37（节选）
demo_ = objServer_.add_interface(demoObjectPath, demoInterfaceName);

demo_->register_property_r<std::string>(
    propertyGrettingName, sdbusplus::vtable::property_::const_,
    [this](const auto&) { return greetings_; });             // 只读属性

demo_->register_property_rw<std::string>(
    propertyGoodbyesName, sdbusplus::vtable::property_::emits_change,
    [this](const auto& newPropertyValue, const auto&) {
        goodbyes_ = newPropertyValue;
        return true;                                        // set 回调
    },
    [this](const auto&) { return goodbyes_; });              // get 回调

demo_->initialize();
```

【先讲】`register_property_r` 注册只读属性，`register_property_rw` 注册读写属性，各自传一个 get lambda、set lambda；flag 控制行为：`const_`（`include/sdbusplus/vtable.hpp:117`）表示不变，`emits_change`（`include/sdbusplus/vtable.hpp:118`）表示变化时要发信号。
【再点】概念：这些 lambda 被包进 `property_callback`（`include/sdbusplus/asio/object_server.hpp:43-64`），`initialize()` 时生成 `property_o` vtable 条目（`include/sdbusplus/asio/object_server.hpp:751-767`）。底层真正被调用的是 `get_handler` / `set_handler`（`include/sdbusplus/asio/object_server.hpp:629`、`include/sdbusplus/asio/object_server.hpp:657`），它们把异常转成 `sd_bus_error`（`include/sdbusplus/asio/object_server.hpp:644-647`、`include/sdbusplus/asio/object_server.hpp:684-687`）。
**Java 类比**：`register_property_rw` ≈ 给一个 POJO 字段注册 getter/setter + `@Getter/@Setter`，外加一个"值变了要发事件"的监听器。

### 4.2 客户端读写属性

```cpp
// example/register-property.cpp:80-90（节选）
sdbusplus::asio::getProperty<std::string>(
    bus_, demoServiceName, demoObjectPath, demoInterfaceName,
    propertyGrettingName,
    [this](boost::system::error_code ec, std::string value) {
        if (ec) { getFailed(); return; }
        std::cout << "Greetings value is: " << value << "\n";
    });

// example/register-property.cpp:124-126（节选）
sdbusplus::asio::setProperty(
    bus_, demoServiceName, demoObjectPath, demoInterfaceName,
    propertyGoodbyesName, "Bye bye",
    [this](const boost::system::error_code& ec) { /* ... */ });
```

`getProperty<T>` 的实现（`include/sdbusplus/asio/property.hpp:42-72`）本质就是发一个标准方法调用：`bus.async_method_call(..., service, path, "org.freedesktop.DBus.Properties", "Get", interface, propertyName)`（`include/sdbusplus/asio/property.hpp:70-71`）；`setProperty` 对应 `Set`（`include/sdbusplus/asio/property.hpp:107`）。【先讲】所以"读写属性"在 D-Bus 层根本不是特殊操作，就是调用标准接口 `org.freedesktop.DBus.Properties` 的 `Get`/`Set` 方法。【再点】这也解释了为什么 `getProperty` 的 handler 签名里带 `boost::system::error_code`——它走的还是那套异步方法调用。
**Java 类比**：≈ 调一个 REST 资源 `GET/POST /properties/{name}`，只不过这里是 D-Bus 内置的标准接口。

批量读用 `getAllProperties`（`example/get-all-properties.cpp:85-135`），拿到一批 `pair<string, variant>`；再用 `sdbusplus::unpackPropertiesNoThrow`（`example/get-all-properties.cpp:99-105`）或 `unpackProperties`（`example/get-all-properties.cpp:156-159`）按名字解出具体类型。类型不匹配时抛 `UnpackPropertyError`（`example/get-all-properties.cpp:129-133`，异常定义在 `include/sdbusplus/exception.hpp:307-308`）。

---

## 5. 真实场景五：监听服务上线 / 下线（match rule + 信号）

最后一个场景是 OpenBMC 里极常见的需求："等某个服务出现再调它"、"某个服务挂了我要知道"。仓库的 `example/coroutine-example.cpp:72-92` 演示了监听 `NameOwnerChanged` 信号。

### 5.1 真实代码

```cpp
// example/coroutine-example.cpp:72-92（节选）
namespace rules = sdbusplus::match_rules;
auto match = sdbusplus::async::match(ctx, rules::nameOwnerChanged());

for (size_t i = 0; i < 4; ++i)
{
    auto nextResult =
        co_await match.next<std::string, std::string, std::string>();
    auto [service, old_name, new_name] = std::move(nextResult);

    if (!new_name.empty())
        std::cout << new_name << " owns " << service << std::endl;
    else
        std::cout << service << " released" << std::endl;
}
```

【先讲】这两步是"我要订阅一类信号，然后一条一条等"：`async::match(ctx, pattern)` 注册监听，`co_await match.next<T...>()` 等到下一条并解包成三个字符串。【再点】概念：`nameOwnerChanged()` 是预置的 match rule 构造函数，返回字符串 `"type='signal',sender='org.freedesktop.DBus',member='NameOwnerChanged',"`（`include/sdbusplus/bus/match.hpp:122-125`）；match rule 就是 D-Bus 的订阅过滤表达式。

### 5.2 match rule 是什么

`include/sdbusplus/bus/match.hpp:51-187` 的 `match_rules` 命名空间提供了一堆拼字符串的小函数：`type::signal()`（`include/sdbusplus/bus/match.hpp:59`）、`sender()`（`include/sdbusplus/bus/match.hpp:78`）、`interface()`（`include/sdbusplus/bus/match.hpp:82`）、`member()`（`include/sdbusplus/bus/match.hpp:86`）、`path()`（`include/sdbusplus/bus/match.hpp:90`）、`argN()`（`include/sdbusplus/bus/match.hpp:102`），以及组合好的 `propertiesChanged()`（`include/sdbusplus/bus/match.hpp:157`）、`interfacesAdded()`（`include/sdbusplus/bus/match.hpp:127`）等。
【先讲】它们只是把 D-Bus 规范里的过滤语法拼成字符串，比如 `propertiesChanged(path, iface)` 拼出 `type='signal',path='...',member='PropertiesChanged',interface='org.freedesktop.DBus.Properties',arg0='...'`（`include/sdbusplus/bus/match.hpp:157-164`）。【再点】概念：注册后由 sd-bus 在 daemon 侧按这个表达式过滤，只把匹配的信号发给你，省得客户端自己筛。

**Java 类比**：match rule ≈消息队列的 topic / 订阅表达式；`async::match` ≈ 一个 `MessageListener`，`co_await match.next()` ≈ 从 `BlockingQueue` 里 `take()` 一条消息。

### 5.3 信号是怎么送进来的

底层有两条路，都在本仓库里：

- **经典版 `bus::match_t`**（`include/sdbusplus/bus/match.hpp:16-48`）：注册时给一个 `callback_t`（`std::function<void(message_t&)`，`include/sdbusplus/bus/match.hpp:38`），信号来了 C 回调 `matchCallback` 转成 C++ 调用（`src/bus/match.cpp:28-36`），构造时调 `sd_bus_add_match`（`src/bus/match.cpp:10-20`）。
- **协程版 `async::match`**（`include/sdbusplus/async/match.hpp:25-74`）：内部维护一个 `std::queue<message_t>` 和"当前等待者"（`include/sdbusplus/async/match.hpp:58-62`）；信号进来先入队（`src/async/match.cpp:71-78`），有等待者就唤醒（`src/async/match.cpp:80-103`）。注意注释明确说明"只支持一个等待任务"（`include/sdbusplus/async/match.hpp:41-42`、`include/sdbusplus/async/match.hpp:49-50`）。

【先讲】经典版是"来一条调一次你的函数"，协程版是"来一条放进队列，等的人自己取"。【再点】这是两种消费模型：回调驱动 vs. 拉取驱动。

---

## 6. 结尾：它和 phosphor-dbus-interfaces / 各守护进程的分工

把三个角色摆在一起就清楚了：

| 角色 | 负责什么 | 在本文代码里的位置 |
| --- | --- | --- |
| **phosphor-dbus-interfaces** | 存放**接口定义 YAML**（哪些方法/属性/信号），是接口定义的源头 | 本文用仓库自带的示例 YAML `include/yaml/net/poettering/Calculator.interface.yaml` 演示；真实 OpenBMC 里对应的就是 phosphor-dbus-interfaces 仓库 |
| **sdbusplus（本仓库）** | ① `sdbus++` 生成器把 YAML 变成 C++ 基类（`tools/sdbusplus/templates/interface.server.hpp.mako:17`）；② libsdbusplus 提供连接/消息/对象/属性/异步/信号监听的 C++ 运行时封装（`meson.build:40-54` 列出的源文件） | 全文所有 `include/sdbusplus/*` 和 `src/*` |
| **各守护进程（ipmid、sensors 等）** | 用生成的基类 + sdbusplus 运行时写业务逻辑，跑事件循环，对外提供 D-Bus 服务 | 本文的 `example/calculator-server.cpp`、`calculator-aserver.cpp`、`asio-example.cpp` 就是它们的缩小版 |

一句话分工：**phosphor-dbus-interfaces 定义"说什么"，sdbusplus 提供"怎么说"的工具和管道，各守护进程决定"具体说什么内容"**。客户端（`calculator-client.cpp`、`list-users.cpp`、`register-property.cpp`）则通过 `bus.call` / `async::proxy` / `asio::connection` 去调这些服务。

从构建关系也能印证：`libsdbusplus` 由 `meson.build:70-78` 产出，链接 `libsystemd`、nlohmann_json、stdexec（`meson.build:75`）；所有例子通过 `sdbusplus_dep`（`meson.build:93-98`）拿到头文件、生成的绑定和链接；generator 本身在 `tools/`（`meson.build:60`）。也就是说，**守护进程只要依赖 `sdbusplus_dep`，就同时拿到"运行时库 + 生成的接口绑定"**，这正是 sdbusplus 把"翻译层"做完整的地方。

想要更深入"内部怎么转"——vtable 怎么搭、`call_async` 的回调怎么被 C 代码调用、match rule 字符串怎么拼、`SdBusInterface` 这层抽象为什么存在、异步 context 的双线程怎么握手——请看姊妹篇《Sdbusplus机制讲解》。
