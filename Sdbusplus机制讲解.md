# Sdbusplus机制讲解：一次方法调用在里面到底怎么转

> **这篇文章写给谁**：和《Sdbusplus业务详解》同一批读者——写过 openUBMC、7 年没碰 C++ 的 Java 程序员。业务背景（这库是干嘛的、服务怎么搭）请看那篇，本篇只回答"内部怎么转"。
>
> **本文回答什么问题**：`bus.call()` 到底怎么走到 sd-bus？`message` 的签名字符串是怎么算出来的？`object_t` 的 vtable 怎么搭、方法怎么被回调？`async::context` 的双线程怎么握手？match rule 信号怎么进来？错误怎么从 C 的 `sd_bus_error` 变成 C++ 异常？
>
> **讲法说明**：【先讲】真实例子和完整代码链，再【再点】机制概念。每个论断带 `相对路径/文件:行号`，可直接核对。C++ 生疏处用 Java 类比。术语用代码原名，不生造译名，不写环境搭建。
>
> **路径说明**：相对 `sdbusplus` 仓库根目录。`include/` 头文件、`src/` 实现、`tools/` 是 YAML→C++ 生成器、`example/` 是例子。

---

## 0. 先讲一个真实例子：list-users 那 5 行内部转了什么

把业务篇场景一的 5 行摆在这里，逐层剥开：

```cpp
// example/list-users.cpp:15-23
auto b = bus::new_default_system();
auto m = b.new_method_call("org.freedesktop.login1", "/org/freedesktop/login1",
                           "org.freedesktop.login1.Manager", "ListUsers");
auto reply = b.call(m);
using return_type = std::vector<std::tuple<uint32_t, std::string, object_path>>;
auto users = reply.unpack<return_type>();
```

【先讲】这 5 行看着简单，内部跨了 5 层：**① 建连接 → ② 造消息 → ③ 同步调用 → ④ 编译期签名解包 → ⑤ 析构自动回收**。逐层展开：

**① 建连接：C++ 工厂 → sd_bus**
- `bus::new_default_system()`（`include/sdbusplus/bus.hpp:587`）→ `bus_t::new_default_system()`（`src/bus.cpp:48-57`）→ `sd_bus_default_system(&b)`（`src/bus.cpp:51`）。
- 返回的 `bus_t` 内部持有 `details::bus`（即 `unique_ptr<sd_bus, BusDeleter>`，`include/sdbusplus/bus.hpp:72-73`、`include/sdbusplus/bus.hpp:558-559`）。`bus_t(busp_t, std::false_type)` 构造函数（`src/bus.cpp:130-142`）直接接管裸指针，并在构造时调一次 `get_unique_name()`——注释解释是为了确保 bus 已经初始化到足够程度，避免后面发 ObjectManager 信号时 sd-bus 内部 assert（`src/bus.cpp:133-140`）。

**② 造消息：四元组 → sd_bus_message**
- `new_method_call()`（`include/sdbusplus/bus.hpp:290-302`）调 `sd_bus_message_new_method_call`（`include/sdbusplus/bus.hpp:294`），把返回的裸 `sd_bus_message*` 交给 `message_t(m, intf, std::false_type())`——这个构造函数接管所有权（`include/sdbusplus/message.hpp:92-94`）。

**③ 同步调用：call → sd_bus_call**
- `b.call(m)`（`include/sdbusplus/bus.hpp:343-355`）调 `sd_bus_call(_bus.get(), m.get(), timeout, &error, &reply)`（`include/sdbusplus/bus.hpp:348`）。注意它先建一个 `sd_bus_error error = SD_BUS_ERROR_NULL`（`include/sdbusplus/bus.hpp:345`），调用返回负值时走 `exception::throw_dbus_error(...)`（`include/sdbusplus/bus.hpp:351`）。

**④ 解包：类型 → 编译期签名 → sd_bus_message_read_basic**
- `unpack<return_type>()`（`include/sdbusplus/message.hpp:162-189`）单类型走 `read(r)`（`include/sdbusplus/message.hpp:179-181`）。`read`（`include/sdbusplus/message.hpp:150-155`）转发给 `sdbusplus::message::read`，最终由类型系统生成的签名驱动底层 read。类型到 D-Bus type id 的映射表在 `include/sdbusplus/message/types.hpp:180-230`。

**⑤ 析构：作用域结束自动回收**
- `b` 出作用域 → `bus_t` 析构 → `unique_ptr` 的 deleter 调 `sd_bus_unref`（`include/sdbusplus/bus.hpp:29-43`）。`m`/`reply` 同理，`MsgDeleter` 调 `sd_bus_message_unref`（`include/sdbusplus/message.hpp:42-51`）。README 特意点出"message 和 bus 对象离开作用域自动释放"（`README.md:65-70`）。

### Java 类比块

| C++ 机制 | 对应到 Java 的心智 |
| --- | --- |
| `bus_t` 持 `unique_ptr<sd_bus, BusDeleter>` | try-with-resources 自动关的 `Connection`；deleter 就是 `close()` |
| `message_t` 的三种所有权构造函数（`include/sdbusplus/message.hpp:81-100`） | 引用 vs. 拷贝：`ref` 构造是多个引用指向同一对象，`std::false_type` 构造是 `new` 出来的独占对象 |
| `call()` 把 METHOD_ERROR 变异常（`include/sdbusplus/bus.hpp:351`） | RPC 框架把远端异常包装成本地异常抛出 |
| 编译期 `type_id<T>`（`include/sdbusplus/message/types.hpp:40`） | 不是 Java 泛型（运行期擦除+反射），是编译期展开、零运行时开销的模板特化 |
| 析构自动 unref | GC 的 finalize/Cleaner？不——RAII 是**确定性**的，离开作用域立刻发生，比 GC 可靠 |

> 【再点】核心认知：sdbusplus 是"薄封装"。`bus.hpp/message.hpp` 里的方法几乎都能找到同名 `sd_bus_*` 函数；README 明确说库"通常编译出来就是你本来要写的那些 sd-bus API 调用"（`README.md:42-46`）。所以读不懂 C++ 包装时，直接去查对应 sd-bus 函数的 manpage。

---

## 1. 先看构建关系：库、示例、测试各是什么

【先讲】从 `meson.build` 看这个项目到底产出哪些东西：

| 产出 | 定义位置 | 说明 |
| --- | --- | --- |
| `libsdbusplus` 库 | `meson.build:70-78` | 核心运行时，源码列表在 `meson.build:40-54`（`bus.cpp`、`message/native_types.cpp`、`server/interface.cpp`、`async/*.cpp` 等 13 个） |
| 生成的 C++ 绑定 | `include/gen/meson.build:19-34` | 由 YAML 经 sdbus++ 生成，`generated_sources` / `generated_headers` 在 `meson.build:62-68` 加入 |
| `sdbusplus_dep` 依赖对象 | `meson.build:93-98` | 把头文件、生成的绑定、`libsdbusplus`、boost/libsystemd/nlohmann_json/stdexec 打包，给下游用 |
| 示例可执行 | `example/meson.build:1-68` | `list-users`、`asio-example`、`coroutine-example`、`register-property`、`get-all-properties`、三个 calculator 程序 |
| 测试 | `test/` 子目录（`meson.build:107-109`） | `test/bus/*`、`test/message/*`、`test/async/*`、`test/server/*` 等 |
| sdbus++ 生成器 | `tools/`（`meson.build:60`） | Python 脚本 + Mako 模板（`tools/sdbusplus/templates/*.mako`） |

【再点】机制：meson 在顶层先 `subdir('tools')` 加载生成器（`meson.build:60`），再 `subdir('include/gen')` 用生成器处理 YAML（`meson.build:68`），然后 `library()` 把手写源码 + 生成源码编成库（`meson.build:70-73`），最后示例和测试通过 `sdbusplus_dep` 拿到全部依赖（`meson.build:93-98`、`example/meson.build:1`）。**"生成的绑定"和"手写的运行时"是两条线，最后汇到同一个库**——这就是 sdbusplus 的完整拼图。生成器的本体在 `tools/sdbus++`（入口脚本）和 `tools/sdbusplus/templates/`（模板）。

---

## 2. 核心类一：bus_t —— 连接怎么建、怎么释放、怎么扩展

### 2.1 所有权：unique_ptr + 自定义 deleter

【先讲】`bus_t` 自己不直接管裸指针，而是用 `details::bus = std::unique_ptr<sd_bus, BusDeleter>`（`include/sdbusplus/bus.hpp:72-73`）。`BusDeleter::operator()` 通过一个函数指针调用 `(m_interface->*deleter)(ptr)`（`include/sdbusplus/bus.hpp:35-38`），默认 deleter 是 `&SdBusInterface::sd_bus_unref`（`include/sdbusplus/bus.hpp:41-42`）。

`set_should_close(true)` 会把 deleter 换成 `sd_bus_flush_close_unref`（`include/sdbusplus/bus.hpp:152-164`）——区别在于是否 flush 并真正关闭连接。`new_bus()` / `new_system()` / `new_user()` 这些"新建"工厂会调它（`src/bus.cpp:69`、`src/bus.cpp:97`、`src/bus.cpp:83`），而 `new_default*()` 不会，因为默认连接是 shared 的、不该被某个持有者关掉。

【再点】机制：为什么要这么绕？因为 sd-bus 的引用计数是 C 风格的手动 `ref/unref`，C++ 用 unique_ptr + deleter 把它变成 RAII。`bus_t` 禁掉拷贝、只允许移动（`include/sdbusplus/bus.hpp:130-136`），就是为了保证同一份所有权不被复制。

### 2.2 为什么有一层 SdBusInterface

【先讲】`bus_t` 里所有对底层的调用都走 `_intf->sd_bus_xxx(...)`，`_intf` 是 `SdBusInterface*`（`include/sdbusplus/bus.hpp:558`）。`SdBusInterface` 是一个纯虚类，声明了所有 `sd_bus_*` 函数（`include/sdbusplus/sdbus.hpp:18-179`），真实实现 `SdBusImpl` 直接 `::sd_bus_xxx`（`include/sdbusplus/sdbus.hpp:181-606`，例如 `sd_bus_call` 在 `include/sdbusplus/sdbus.hpp:218-222`），全局单例 `sdbus_impl`（`include/sdbusplus/sdbus.hpp:608`）。

【再点】机制：这是一层**依赖注入 / 接口抽象**，目的是可测试——测试里可以塞一个 mock 实现（仓库提供 `include/sdbusplus/test/sdbus_mock.hpp`），不用真的连 D-Bus daemon。业务代码完全感知不到这层，因为默认构造都传 `&sdbus_impl`。**Java 类比**：≈ 把.static 工具调用包成接口 `SdBusApi`，生产注入 `SystemdSdBusApi`、测试注入 `MockSdBusApi`，是标准的"面向接口编程"。

### 2.3 slot：异步操作的"句柄"

`slot` 是另一类 RAII 包装，持有 `sd_bus_slot*`，析构调 `sd_bus_slot_unref`（`include/sdbusplus/slot.hpp:14-27`、`include/sdbusplus/slot.hpp:36-72`）。凡是"注册之后要能取消"的东西——match、object manager、vtable、异步调用——都返回一个 slot：
- ObjectManager：`include/sdbusplus/server/manager.hpp:27-32` 返回 `slot`
- vtable 注册：`src/server/interface.cpp:12-24` 的 `makeObjVtable` 返回 `slot`
- match：`src/bus/match.cpp:10-20` 的 `makeMatch` 返回 `slot`
- 异步调用：`include/sdbusplus/message.hpp:448-487` 的 `call_async` 返回 `slot`

【再点】slot 保证了"注册即生命周期"：对象析构，注册自动撤销，不会泄漏回调。

---

## 3. 核心类二：message_t —— 消息怎么收发、签名怎么算

### 3.1 三种所有权构造

`message_t`（`message::message` 的别名，`include/sdbusplus/message.hpp:533`）有几个关键构造：
- `message(msgp_t m, SdBusInterface* intf)`（`include/sdbusplus/message.hpp:81-83`）：调 `sd_bus_message_ref` —— **增加引用，共享所有权**。
- `explicit message(msgp_t m = nullptr)`（`include/sdbusplus/message.hpp:90`）：同上，用默认实现。
- `message(msgp_t m, SdBusInterface*, std::false_type)`（`include/sdbusplus/message.hpp:92-94`）：**直接接管，不增加引用**。
- 拷贝构造（`include/sdbusplus/message.hpp:106-108`）：`sd_bus_message_ref(other)`，多一个引用。

【再点】用 `std::false_type` 这个 tag 区分"接管"和"引用"两种语义，是 C++ 里常见的 tag dispatch；Java 没有这种区分（GC 帮你管引用计数）。

### 3.2 append / read / unpack：编译期签名从哪来

`append(Args&&...)`（`include/sdbusplus/message.hpp:138-143`）转发到 `message::append`；`read(Args&&...)`（`include/sdbusplus/message.hpp:150-155`）转发到 `message::read`。真正的核心是类型系统：

```cpp
// include/sdbusplus/message/types.hpp:40（声明）
constexpr auto type_id();
// include/sdbusplus/message/types.hpp:180-230（部分特化）
template <> struct type_id<bool>       : tuple_type_id<SD_BUS_TYPE_BOOLEAN> {};
template <> struct type_id<uint64_t>   : tuple_type_id<SD_BUS_TYPE_UINT64>  {};
template <> struct type_id<std::string>: tuple_type_id<SD_BUS_TYPE_STRING>  {};
template <> struct type_id<object_path>: tuple_type_id<SD_BUS_TYPE_OBJECT_PATH> {};
```

【先讲】`type_id<T>` 是一组模板特化，把 C++ 类型映射成 D-Bus 的 type id 序列。`tuple_type_id` 把这些字符拼成签名（`include/sdbusplus/message/types.hpp:126-141`）。`unpack<Args...>()` 在编译期就知道了签名，运行期只需按这个签名读字节。
【再点】机制：D-Bus 不支持 `int8_t`（`include/sdbusplus/message/types.hpp:187` 注释）和 `float`（`include/sdbusplus/message/types.hpp:206` 注释）——类型表里压根没有对应特化，写错会编译失败。这正是"编译期类型安全"的具体体现：错误在编译期暴露，而不是运行期抛异常。**Java 类比**：≈ 编译期就做完了 Jackson 的类型注册和 signature 计算，运行期零反射。

### 3.3 服务端回复与错误：new_method_return / new_method_error

服务端拿到一个 `message_t`（方法调用）后，要构造回复：
- `new_method_return()`（`include/sdbusplus/message.hpp:348-358`）：调 `sd_bus_message_new_method_return`，构造一个 method-return 消息。
- `new_method_error(const exception&)`（`include/sdbusplus/message.hpp:365-379`）：先 `e.set_error(_intf, &error)` 把 C++ 异常转成 `sd_bus_error`，再调 `sd_bus_message_new_method_error`。
- `new_method_errno(int, const sd_bus_error*)`（`include/sdbusplus/message.hpp:387-398`）：直接按 errno 造错误。
- `method_return()`（`include/sdbusplus/message.hpp:401-409`）：调 `sd_bus_send` 把消息发出去。

这套 API 在 `asio::object_server` 的回调里被大量使用，例如 `callback_method_instance::operator()` 里先 `m.new_method_return()`、写入返回值、再 `ret.method_return()`（`include/sdbusplus/asio/object_server.hpp:211-223`）。

### 3.4 异步调用：call_async 与那个 C 回调

```cpp
// include/sdbusplus/message.hpp:448-487（节选）
template <typename Cb>
[[nodiscard]] slot call_async(Cb&& cb, std::optional<SdBusDuration> timeout = ...)
{
    using CbT = std::remove_cv_t<std::remove_reference_t<Cb>>;
    int r = _intf->sd_bus_call_async(nullptr, &slotp, get(),
                                     details::call_async_cb<CbT>, nullptr, timeout_us);
    ...
    _intf->sd_bus_slot_set_userdata(get_slotp(ret), new CbT(std::forward<Cb>(cb)));
    return ret;
}
```

【先讲】把你的 lambda（`CbT`）`new` 出来挂到 slot 的 userdata 上，然后把一个静态模板函数 `call_async_cb<CbT>` 注册给 sd-bus；回复到了，sd-bus 调 `call_async_cb`，它从 userdata 里取出 lambda、把回复 `message_t` 传进去执行（`include/sdbusplus/message.hpp:508-527`）。lambda 的销毁挂在 slot 的 destroy callback（`include/sdbusplus/message.hpp:476-485`）。
【再点】机制：这是典型的"C 回调 → C++ 可调用对象"桥接——C 只能传 `void*`，所以 C++ 这边用 userdata 带类型信息、用模板函数在回调里恢复类型。注意 `call_async_cb` 内部 `catch (...) { std::terminate(); }`（`include/sdbusplus/message.hpp:522-525`）：**回调里抛异常会直接终止进程**，所以异步回调内必须自己处理错误。**Java 类比**：≈ 给一个只接受 `void*` 的 C API 写 JNI 桥，userdata 就是你的对象句柄。

### 3.5 消息的自省：从 message_t 里读元信息

【先讲】`message_t` 提供一组 getter，把 sd-bus 的消息元信息暴露出来：`get_signature()`（`include/sdbusplus/message.hpp:199-202`）、`get_path()`（`include/sdbusplus/message.hpp:208-211`）、`get_interface()`（`include/sdbusplus/message.hpp:217-220`）、`get_member()`（`include/sdbusplus/message.hpp:226-229`）、`get_destination()`（`include/sdbusplus/message.hpp:235-238`）、`get_sender()`（`include/sdbusplus/message.hpp:244-247`）。还有判断类：`is_method_error()`（`include/sdbusplus/message.hpp:253-256`）、`get_errno()`（`include/sdbusplus/message.hpp:262-265`）、`get_error()`（`include/sdbusplus/message.hpp:271-274`）、`get_type()`（`include/sdbusplus/message.hpp:280-289`）、`is_method_call()`（`include/sdbusplus/message.hpp:328-332`）、`is_signal()`（`include/sdbusplus/message.hpp:339-342`）。

【再点】机制：这些方法用于在**通用消息处理代码**里分辨"这条消息是什么"，比如 `example/asio-example.cpp:313` 用 `ret.is_method_error()` 判断异步调用是否失败。服务端分派时也会用 `is_method_call(interface, method)` 判断要不要处理。`get_signature()` 返回的是 D-Bus 类型串——和编译期 `type_id<T>` 算出来的是同一个东西，一个是"声明时"、一个是"运行时"。

---

## 4. 核心类三：object_t 与 vtable —— 方法怎么被注册和调用

### 4.1 vtable 是 sd-bus 的"方法表"

【先讲】`vtable_t` 就是 `sd_bus_vtable` 的别名（`include/sdbusplus/vtable.hpp:10-11`）。`include/sdbusplus/vtable.hpp` 提供一组 constexpr 工厂来拼这张表：`start()`（`include/sdbusplus/vtable.hpp:124-127`）、`end()`（`include/sdbusplus/vtable.hpp:129-132`）、`method_o()`（`include/sdbusplus/vtable.hpp:141-148`）、`property_o()`（`include/sdbusplus/vtable.hpp:171-184`）、`signal()`（`include/sdbusplus/vtable.hpp:150-154`）。标志位如 `property_::const_`、`property_::emits_change`（`include/sdbusplus/vtable.hpp:115-122`）。

【再点】机制：sd-bus 靠这张表知道"某个接口的哪些方法/属性/信号存在、分别对应哪个 C 回调"。它是 C 风格的静态数组，sdbusplus 把它包成 C++ 工厂函数，但本质没变。

### 4.2 interface_t：把 vtable 注册到总线

`server::interface_t`（`include/sdbusplus/server/interface.hpp:35`）是对 `sd_bus_add_object_vtable` 的 RAII 包装：
- 构造函数（`src/server/interface.cpp:26-33`）调 `makeObjVtable` → `sd_bus_add_object_vtable`（`src/server/interface.cpp:18`），成功返回 `slot`。
- 析构函数（`src/server/interface.cpp:41-44`）调 `emit_removed()`，发接口移除信号。
- `property_changed(const char*)`（`src/server/interface.cpp:46-54`）调 `sd_bus_emit_properties_changed_strv`（`src/server/interface.cpp:52`）发属性变化信号。

### 4.3 object_t 的组合与信号

`object_t<Args...>`（`include/sdbusplus/server/object.hpp:38-42`）用多重继承把多个接口组合成一个 D-Bus 对象（注释 `include/sdbusplus/server/object.hpp:26-37`）。`compose` 模板递归展开继承链（`include/sdbusplus/server/object.hpp:185-212`）。构造时根据 `action` 决定发哪种 added 信号（`include/sdbusplus/server/object.hpp:57-68`、`check_action()` `include/sdbusplus/server/object.hpp:125-151`）；析构时发 `sd_bus_emit_object_removed`（`include/sdbusplus/server/object.hpp:92-100`）。

【再点】机制：这就是生成器生成的 `*server.hpp` 里"业务类继承 `object_t<生成的接口>`"的那一层——生成器负责"接口长什么样"，`object_t` 负责"怎么把接口挂到对象路径上并管理生命周期"。生成的类构造函数把 `interface` + `_vtable` + `this` 传给 `interface_t`（模板见 `tools/sdbusplus/templates/interface.server.hpp.mako:41-45`）。

### 4.4 两条注册路径：生成式 vs. 动态式

- **生成式（Calculator server）**：vtable 在编译期由生成器写死在生成的 `.cpp` 里，业务类通过 `override` 虚函数提供实现（例子 `example/calculator-server.cpp:20`、`example/calculator-server.cpp:29`、`example/calculator-server.cpp:42`）。
- **动态式（asio object_server）**：运行时用 `register_method` / `register_property_r` / `register_signal` 收集回调，再在 `initialize()` 里拼出 vtable（`include/sdbusplus/asio/object_server.hpp:740-803`）：`start()` → 各 property → 各 method → 各 signal → `end()`（`include/sdbusplus/asio/object_server.hpp:747-786`），然后构造 `interface_t`（`include/sdbusplus/asio/object_server.hpp:789-792`）。

【再点】机制：两者最终都落到 `sd_bus_add_object_vtable`——区别只是"表在编译期生成"还是"运行期拼装"。**Java 类比**：生成式 ≈ 编译期生成的注解处理器代码（如 Dagger）；动态式 ≈ 运行期反射注册（如 Spring 扫描）。

### 4.5 从 YAML 到 vtable：生成器的最后一公里

【先讲】以方法为例，生成器模板 `tools/sdbusplus/templates/method.server.vtable.cpp.mako:1-10` 把每个方法写成一条 `vtable::method("方法名", 参数签名数组, 返回签名数组, _callback_方法名, flags)`。参数签名 `_param_Xxx` 和返回签名 `_return_Xxx` 也是生成的常量数组，内容就是 `type_id<T>` 算出来的字符序列。属性同理，`tools/sdbusplus/templates/property.server.vtable.cpp.mako:4-10` 根据 YAML 里有没有 `const`/`readonly` flag，决定生成只读的 `property` 还是可写的 `property`，默认 flags 是 `vtable::property_::emits_change`（`tools/sdbusplus/templates/property.server.vtable.cpp.mako:7-8`）。

【再点】机制：所以"YAML → C++"这条链是：YAML 描述方法名和类型 → 模板查类型映射生成签名数组和 vtable 条目 → `interface_t` 把这张表注册给 sd-bus → 方法被调到时进入生成的 `_callback_方法名` → 它再调你 `override` 的虚函数。每一层都能在仓库里找到对应文件：YAML 在 `include/yaml/`，模板在 `tools/sdbusplus/templates/`，注册在 `src/server/interface.cpp:18`，回调样板在 `tools/sdbusplus/templates/interface.server.hpp.mako:139-147`（属性的 `_callback_get_` / `_callback_set_` 声明）。

---

## 5. 核心类四：properties —— 读写属性内部怎么走

### 5.1 服务端的 get/set 回调

以 `asio::dbus_interface` 为例。`register_property_rw`（`include/sdbusplus/asio/object_server.hpp:449-482`）把一个属性注册成三样东西：
- `callback_get_instance`（`include/sdbusplus/asio/object_server.hpp:302-320`）：get 时调用你的 lambda，把值 append 到回复消息。
- `callback_set_message_instance`（`include/sdbusplus/asio/object_server.hpp:322-350`）：从消息里 unpack 新值，调你的 set lambda；返回 `SetPropertyReturnValue` 枚举（`fail` / `valueUpdated` / `sameValueUpdated`，定义 `include/sdbusplus/asio/object_server.hpp:34-39`）。
- `callback_set_value_instance`（`include/sdbusplus/asio/object_server.hpp:352-379`）：给服务端本地 `set_property` 用的按值版本。

底层 C 回调 `get_handler` / `set_handler`（`include/sdbusplus/asio/object_server.hpp:629`、`include/sdbusplus/asio/object_server.hpp:657`）把 `sd_bus_error*` 传出来：你的 lambda 抛 `sdbusplus::exception_t` 时，`e.set_error(error)` 把它转成 D-Bus 错误（`include/sdbusplus/asio/object_server.hpp:644-647`、`include/sdbusplus/asio/object_server.hpp:684-687`）。

值真的变了才发信号：`set_handler` 里只有 `status != sameValueUpdated` 才 `signal_property`（`include/sdbusplus/asio/object_server.hpp:673-676`）。`signal_property()`（`include/sdbusplus/asio/object_server.hpp:810-818`）最终调 `interface_->property_changed(name)` → `sd_bus_emit_properties_changed_strv`（`src/server/interface.cpp:52`）。

### 5.2 客户端的 Get/Set/GetAll

`asio::getProperty<T>`（`include/sdbusplus/asio/property.hpp:42-72`）本质是 `bus.async_method_call(..., "org.freedesktop.DBus.Properties", "Get", interface, propertyName)`（`include/sdbusplus/asio/property.hpp:70-71`）；`setProperty`（`include/sdbusplus/asio/property.hpp:107`）对应 `Set`；`getAllProperties`（`include/sdbusplus/asio/property.hpp:11`、`include/sdbusplus/asio/property.hpp:29`）对应 `GetAll`。

拿到的是 `std::vector<std::pair<std::string, std::variant<...>>>`，再用 `sdbusplus::unpackProperties` / `unpackPropertiesNoThrow` 按名字解类型。解不出来抛 `UnpackPropertyError`（`include/sdbusplus/exception.hpp:307-308`），错误原因枚举 `UnpackErrorReason{missingProperty, wrongType}`（`include/sdbusplus/exception.hpp:19-23`）。

### 5.3 unpackProperties 怎么按名字取值

【先讲】`include/sdbusplus/unpack_properties.hpp` 里，`findProperty`（`include/sdbusplus/unpack_properties.hpp:23`）按名字找，`readProperty`（`include/sdbusplus/unpack_properties.hpp:33`）做类型检查：类型不匹配报 `wrongType`（`include/sdbusplus/unpack_properties.hpp:56`、`include/sdbusplus/unpack_properties.hpp:72`、`include/sdbusplus/unpack_properties.hpp:84`），名字找不到报 `missingProperty`（`include/sdbusplus/unpack_properties.hpp:92`）。`unpackProperties`（`include/sdbusplus/unpack_properties.hpp:143`）内部调 `unpackPropertiesCommon`（`include/sdbusplus/unpack_properties.hpp:124`），默认是抛异常版；`unpackPropertiesNoThrow`（`include/sdbusplus/unpack_properties.hpp:158`）传一个错误回调进去，不抛异常而是把错误报给回调。

【再点】机制：`GetAll` 返回的是 `variant`，编译期不知道每个属性具体什么类型，所以只能运行期检查——这和 `unpack<T>()` 的编译期强类型是两个互补的入口：知道确切类型用 `getProperty<T>`，批量且类型不定用 `getAllProperties` + `unpackProperties`。**Java 类比**：前者 ≈ 强类型反序列化，后者 ≈ 先拿到 `Map<String, Object>` 再逐个 `instanceof` 校验。

---

## 6. 异步：async::context 的双线程与 task

### 6.1 context 的两个线程

【先讲】`async::context`（`include/sdbusplus/async/context.hpp:43`）的注释直接说明：它有 caller 和 worker 两个线程（`include/sdbusplus/async/context.hpp:33-41`）——caller 线程（通常是 `main` 调 `run()` 的线程）**唯一**调 `sd_bus_wait`；worker 线程执行所有 `sd_bus_process` 和异步任务，两者用 condition variable 握手（`include/sdbusplus/async/context.hpp:38-41`）。

实现对应：
- `context::run()`（`src/async/context.cpp:123-143`）：调 `caller_run()`，然后等 wait/process 停止、等内部任务完成、`loop.finish()`、join worker。
- `caller_run()`（`src/async/context.cpp:227-263`）：启动 worker 线程（`src/async/context.cpp:242`），然后循环 `wait_process_completion::wait_once(*this)`（`src/async/context.cpp:254`）。
- `worker_run()`（`src/async/context.cpp:164-173`）：spawn watchdog、`wait_process_completion::loop`，然后 `loop.run()` 跑 stdexec 的 run_loop。

【再点】机制：为什么要拆两个线程？因为 `sd_bus_wait` 会阻塞，如果它和 `sd_bus_process`/任务执行在同一线程，处理消息时就没法同时等新消息。拆开后 caller 专职"等事件"，worker 专职"干活"，通过 condition variable 传递。这是 sdbusplus 里最复杂的一块并发设计。

**Java 类比**：≈ 把 Netty 的 selector 线程（等着）和 worker 线程池（干活）分开，中间用任务队列衔接。

### 6.2 task / spawn / proxy

- `sdbusplus::async::task<T>` 是 `exec::task<T>` 的别名（`include/sdbusplus/async/task.hpp:11`）——C++20 协程的返回类型，`co_await` / `co_return` 由编译器生成状态机。
- `context::spawn(Snd&&)`（`include/sdbusplus/async/context.hpp:62-71`）接收 Sender 或 `task<void>`（非模板重载 `include/sdbusplus/async/context.hpp:80`），交给 `pending_tasks` scope，并起一个 watcher。
- `async::proxy`（`include/sdbusplus/async/proxy.hpp:51-53`）是客户端代理，用 `service()` / `path()` / `interface()`（`include/sdbusplus/async/proxy.hpp:80-94`）链式设置三元组，均返回**新对象**（不可变风格）。它支持 constexpr 构造（`include/sdbusplus/async/proxy.hpp:33-41` 的例子），需要离开作用域时用 `preserve()` 转成 `std::string` 备份（`include/sdbusplus/async/proxy.hpp:96-111`）。

【再点】`proxy` 和生成器生成的 `client` 类型是两套客户端写法；`example/calculator-client.cpp:8-12` 用生成的 `Calculator` client，`example/coroutine-example.cpp:11-14` 用通用 `proxy`。

### 6.2.1 proxy 的一次调用到底怎么发

【先讲】看 `proxy::call`（`include/sdbusplus/async/proxy.hpp:125-144`）就能把前面几节串起来：它先用 `ctx.get_bus().new_method_call(c_str(s), c_str(p), c_str(i), method)` 造消息（`include/sdbusplus/async/proxy.hpp:129-130`，回到第②步），有参数就 `msg.append(...)`（`include/sdbusplus/async/proxy.hpp:131-134`），然后用 `callback(...)` 把 `sd_bus_call_async(bus, nullptr, msg.get(), cb, data, 0)` 包成 Sender（`include/sdbusplus/async/proxy.hpp:138-142`，回到第③步的异步版），最后 `execution::then` 接一个 `m.unpack<Rs...>()`（`include/sdbusplus/async/proxy.hpp:143`，回到第④步）。

`get_property<T>`（`include/sdbusplus/async/proxy.hpp:155-165`）更巧：它造一个指向标准 Properties 接口的 proxy（`dbus_prop_intf`，`include/sdbusplus/async/proxy.hpp:160`），对 `"Get"` 方法调用 `call<result_t>`，参数传接口名和属性名（`include/sdbusplus/async/proxy.hpp:162-163`），再 `then` 取出 variant 里的 T（`include/sdbusplus/async/proxy.hpp:164`）。【再点】机制：**属性读写在通用 proxy 里也被还原成了对 `org.freedesktop.DBus.Properties` 的方法调用**，和 `asio::getProperty` 是同一个底层协议，只是包成了协程 Sender。

### 6.3 asio 版的异步

`asio::connection`（`include/sdbusplus/asio/connection.hpp:54`）继承 `bus_t`，把 D-Bus fd 接到 `io_context`：
- `async_send(m, callback, timeout)`（`include/sdbusplus/asio/connection.hpp:91-96`）用 `boost::asio::async_initiate` 发起。
- `async_method_call(handler, service, path, iface, method, args...)`（`include/sdbusplus/asio/connection.hpp:210-219`）先 `new_method_call` + `append`，再 `async_send`；回复到了用 `unpack`（`include/sdbusplus/asio/connection.hpp:109-145`）按 handler 的签名自动解包。
- `yield_method_call`（`include/sdbusplus/asio/connection.hpp:257-291`）配合 `boost::asio::yield_context`，把异步写成同步风格。

**Java 类比**：`async_method_call(handler)` ≈ `CompletableFuture.whenComplete`；`yield_method_call` ≈ Kotlin 协程用 `await()` 把 Future 写成顺序代码。

---

## 7. match rule 与信号：过滤表达式和两种消费模型

### 7.1 match rule 就是字符串

【先讲】`include/sdbusplus/bus/match.hpp:51-187` 的 `match_rules` 命名空间里全是拼字符串的小函数：`type::signal()`（`include/sdbusplus/bus/match.hpp:59`）、`sender()`（`include/sdbusplus/bus/match.hpp:78`）、`interface()`（`include/sdbusplus/bus/match.hpp:82`）、`member()`（`include/sdbusplus/bus/match.hpp:86`）、`path()`（`include/sdbusplus/bus/match.hpp:90`）、`argN()`（`include/sdbusplus/bus/match.hpp:102`），以及组合式 `nameOwnerChanged()`（`include/sdbusplus/bus/match.hpp:122-125`）、`propertiesChanged(p, i)`（`include/sdbusplus/bus/match.hpp:157-164`）、`interfacesAdded()`（`include/sdbusplus/bus/match.hpp:127-130`）。比如：

```cpp
// include/sdbusplus/bus/match.hpp:122-125
constexpr auto nameOwnerChanged() noexcept
{
    return "type='signal',sender='org.freedesktop.DBus',member='NameOwnerChanged',"s;
}
```

【再点】机制：match rule 是 D-Bus 规范定义的过滤语法，注册给 daemon 后由 daemon 侧过滤，只把匹配的信号投递给你，省去客户端自己筛查。

### 7.2 两种消费模型

- **经典回调式 `bus::match_t`**（`include/sdbusplus/bus/match.hpp:16-48`）：给一个 `callback_t = std::function<void(message_t&)>`（`include/sdbusplus/bus/match.hpp:38`）；信号来了，C 函数 `matchCallback` 转成 C++ 调用（`src/bus/match.cpp:28-36`）。注册走 `sd_bus_add_match`（`src/bus/match.cpp:14`）。
- **协程队列式 `async::match`**（`include/sdbusplus/async/match.hpp:25-74`）：内部 `std::queue<message_t>` + 一个等待者指针（`include/sdbusplus/async/match.hpp:58-62`）。信号先入队（`src/async/match.cpp:71-78`），有等待者就唤醒（`src/async/match.cpp:80-103`）。`next<T...>()` 返回一个 Sender（`include/sdbusplus/async/match.hpp:150-155`），`co_await` 后解包成指定类型。**限制**：只支持一个等待任务（注释 `include/sdbusplus/async/match.hpp:41-42`、`include/sdbusplus/async/match.hpp:49-50`），多人同时 `co_await` 会在 `match_completion::start()` 里抛 `logic_error` 并 `std::terminate`（`src/async/match.cpp:52-66`）。

【再点】**Java 类比**：回调式 ≈ `MessageListener.onMessage()`；队列式 ≈ 从 `BlockingQueue` 里 `take()`。注意队列式那个"单消费者"限制——Java 的 `LinkedBlockingQueue` 可以被多个线程 take，这里不行。

### 7.3 信号的发射

【先讲】收信号讲完了，发射端怎么发？有三类：
- **自定义信号**：先用 `bus.new_signal(path, interface, member)` 造消息（`include/sdbusplus/bus.hpp:319-330`），`append` 参数，再 `signal_send()`（`include/sdbusplus/message.hpp:411-415`，内部就是 `method_return()` 调 `sd_bus_send`）。生成式接口里是 `cleared(v)` 这样的调用（例子 `example/calculator-server.cpp:46`，信号定义在 `include/yaml/net/poettering/Calculator.interface.yaml:69-77`）。
- **属性变化信号**：`interface::property_changed()`（`src/server/interface.cpp:46-54`）调 `sd_bus_emit_properties_changed_strv`。
- **对象/接口增减信号**：`object_t` 构造时 `sd_bus_emit_object_added`（`include/sdbusplus/server/object.hpp:103-112`），析构时 `sd_bus_emit_object_removed`（`include/sdbusplus/server/object.hpp:92-100`）；也有 `bus_t::emit_interfaces_added`（`src/bus.cpp:7-13`）这类底层封装。

`asio::dbus_interface` 还提供 `register_signal(name)`（`include/sdbusplus/asio/object_server.hpp:574-591`）声明信号签名、`new_signal(member)`（`include/sdbusplus/asio/object_server.hpp:731-738`）构造信号消息、`signal_property(name)`（`include/sdbusplus/asio/object_server.hpp:810-818`）发属性变化。

【再点】机制：信号本质就是一条 `type='signal'` 的 D-Bus 消息，没有返回值、不需要回复。match rule 里的 `'type=signal'`（比如 `include/sdbusplus/bus/match.hpp:59`）正是按这个类型字段过滤的——收和发在协议层是对称的。

---

## 8. 错误处理：从 sd_bus_error 到 C++ 异常

### 8.1 异常体系

`include/sdbusplus/exception.hpp` 定义了一棵继承树：
- `exception : std::exception`（`include/sdbusplus/exception.hpp:54`），纯虚 `name()` / `description()` / `get_errno()` / `set_error()`（`include/sdbusplus/exception.hpp:56-61`）。
- `generated_exception`（`include/sdbusplus/exception.hpp:72-75`）：sdbus++ 生成的错误基类。
- `internal_exception`（`include/sdbusplus/exception.hpp:176`）：sdbusplus 自己的内部错误基类；`SdBusError`（`include/sdbusplus/exception.hpp:243-278`）是它的实现，代表底层 sd-bus 调用失败。
- `generated_event`（`include/sdbusplus/exception.hpp:140-173`）：生成器按 `*.events.yaml` 生成的事件/错误，携带 name、description、errno、severity。

别名 `exception_t = exception::exception`（`include/sdbusplus/exception.hpp:324`）。

### 8.2 C 错误 → C++ 异常

`throw_dbus_error(method, sd_bus_error*)`（`include/sdbusplus/exception.hpp:286-288`，实现 `src/exception.cpp:173`）负责把 `sd_bus_error` 转成合适的 C++ 异常并抛出。`bus_t::call()`（`include/sdbusplus/bus.hpp:351`）和 `message::call()`（`include/sdbusplus/message.hpp:435`）在调用失败时就走它。

反方向，`exception::set_error(sd_bus_error*)`（`src/exception.cpp:18-23`）把 C++ 异常写进 `sd_bus_error`，供服务端回调返回给 sd-bus。服务端回调里 `catch (const sdbusplus::exception_t& e) { return e.set_error(error); }` 是固定套路（如 `include/sdbusplus/asio/object_server.hpp:714-717`）。

### 8.3 transaction id（一个容易忽略的细节）

`include/sdbusplus/server/README.md:3-17` 描述：transaction id 是对 bus name 和 message cookie 做哈希得到的唯一标识，创建错误响应时生成，存在异常对象里（`include/sdbusplus/server/README.md:13-14`），作为 journal 日志的 metadata（`include/sdbusplus/server/README.md:19-21`）。这样一次 D-Bus 操作产生的所有日志能靠同一个 id 串起来——对排查 OpenBMC 问题很实用。

【再点】机制：错误不只是"抛个异常"，还携带了链路追踪信息。这是 sdbusplus 和 OpenBMC 的日志体系（phosphor-logging）衔接的地方。

---

## 9. 特殊要点与坑（速查）

1. **`new_default*` vs `new_*`**：`new_default_system()` 拿的是 sd-bus 存在 thread-local storage 里的**共享**连接（`include/sdbusplus/bus.hpp:84-104` 注释），`new_system()` 才是新建独立连接。写测试需要同时有 server 和 client 连接时，必须用 `new_bus` 系列，否则共享一个连接会出问题（`include/sdbusplus/bus.hpp:97-103` 注释）。
2. **`set_should_close`**：默认 deleter 是 `sd_bus_unref`，`new_bus`/`new_system`/`new_user` 会设成 `sd_bus_flush_close_unref`（`src/bus.cpp:69`、`src/bus.cpp:97`、`src/bus.cpp:83`）。别对共享的默认连接调 `set_should_close(true)`。
3. **`message::get()` 裸指针**：`include/sdbusplus/message.hpp:491-498` 的注释明确警告——暴露裸 `sd_bus_message*` 可能引发竞态、RAII 析构问题，"Here be dragons"。非必要不要用。
4. **message 拷贝 = 增加引用**：拷贝构造调 `sd_bus_message_ref`（`include/sdbusplus/message.hpp:106-108`），多一个引用计数，不是深拷贝。
5. **异步回调里别抛异常**：`call_async_cb` 里 `catch (...) { std::terminate(); }`（`include/sdbusplus/message.hpp:522-525`），回调抛异常直接终止进程。
6. **match 只支持单等待者**：`async::match` 两个任务同时 `co_await next()` 会 terminate（`src/async/match.cpp:52-66`）。
7. **D-Bus 没有 int8/float**：`include/sdbusplus/message/types.hpp:187`、`include/sdbusplus/message/types.hpp:206`，写进消息会编译失败。
8. **slot 决定注册生命周期**：match / vtable / object manager / async call 都靠 `slot` 析构自动撤销（`include/sdbusplus/slot.hpp:36-72`），把 slot 存成成员变量是常见模式（如 `match` 的 `_slot`，`include/sdbusplus/async/match.hpp:58`）。
9. **SdBusInterface 是为测试准备的**：所有底层调用都经它（`include/sdbusplus/sdbus.hpp:18-179`），生产用 `sdbus_impl`（`include/sdbusplus/sdbus.hpp:608`），测试可注入 mock。

---

## 小结

把内部机制串成一句话：**sdbusplus 用 `unique_ptr`/`slot` 做 RAII 生命周期管理，用 `SdBusInterface` 做可测试的底层抽象，用 `type_id` 模板在编译期算 D-Bus 签名，用 vtable 把 C++ 方法/属性注册成 sd-bus 回调，用 `async::context` 的双线程 + C++20 协程把异步包装成同步写法，用 match rule 字符串 + 队列/回调两种方式收信号，用 `sd_bus_error` ↔ C++ 异常双向转换统一错误处理**。业务篇里看到的每一个简单 API，背后都是这套机制在支撑。
