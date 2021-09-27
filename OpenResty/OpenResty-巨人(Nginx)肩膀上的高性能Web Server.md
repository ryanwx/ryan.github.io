## OpenResty-巨人(Nginx)肩膀上的高性能Web Server

- ### 项目背景

OpenResty 最早是雅虎中国的一个公司项目，项目发起人是章亦春，大家唤他为春哥。最初是为了对外提供 OpenAPI，后来陆续应用于雅虎中国的搜索业务，随着春哥加入淘宝量子统计部门，项目被重写，应用重点放在了 Web 产品上，开发的重点则是性能优化。

- ### 项目介绍

OpenResty 是基于 Nginx 开发的一个Web Server 框架，可以认为它是Nginx的扩展，项目采用Nginx + lua 的开发形式。这是一个纯 AJAX 的应用，所有操作同步非阻塞，且以 Nginx 为基础，采用 lua 开发，再搭配 LuaJIT 热编译，因此在拥有极高的性能的同时也兼顾了开发的简易。

- ### 项目特点

基于 Nginx 扩展开发高性能的 web 服务器，所有操作同步非阻塞，使用 Lua 动态脚本语言开发，在保证高性能的同时，对开发是友好的（*Life is short, you need* *Python* *OpenResty*），除了以上尤其明显的优势，同时也有一些问题，如：第三方应用对 OpenResty 的支持不够，适用场景有限，社区活跃度不够等。

- ### 适用场景

  - Nginx 重度使用用户，Nginx 现有的功能无法满足，需要对 Nginx 做扩展
  - 众多云供应商，对 Web Server 有较高的定制化需求。
  - 中小企业开发高并发的中小型 Web 应用

- ### OpenResty重点模块介绍

  - #### LuaJIT

    - **什么是 LuaJIT**

LuaJIT 是OpenResty 默认捆绑的 Lua 解释器，Lua 代码开始被 LuaJIT 解释运行，运行过程中 LuaJIT 记录 Lua 函数，文件等的调用信息，当达到它设置的阈值时，LuaJIT 将会对这些 Lua 执行编译工作，这被称为 OpenResty 的热装载功能。这是 OpenResty 在嵌入 Lua 脚本语言后仍能保持高性能的原因之一。

- **LuaJIT 热编译功能的配置**

OpenResty 默认将热编译功能开启，并且当我们将热编译功能关闭后重启/重载 nginx 时，将会抛出警告信息，提示我们关闭热编译功能将伤害 OpenResty 的性能。

OpenResty 热编译配置：

```
lua_code_cache on|off;
```

关闭热编译后重载 nginx，看到警告信息（通常我们在开发环境设置为 off）

```
PS D:\tools\WebService\OpenResty> nginx -s reload
nginx: [alert] lua_code_cache is off; this will hurt performance in ./conf/nginx.conf:38
```

热编译配置下，项目所有 Lua 代码修改后，必须让 Nginx 重载，否则修改的 Lua 代码不会生效。

```
PS D:\tools\WebService\OpenResty> nginx -s reload
```

- ##### **LuaJIT 编译的局限性**

并非所有的 Lua 代码都能被 LuaJIT 编译，LuaJIT 不能编译所有的 Lua 代码，所以我们在开发 OpenResty 项目时，需要特别注意 LuaJIT 的编译对象，对于不能编译的方法/函数，不要使用，好在 OpenResty 生态中，涌现了不少的第三方库 resty 库（这在后续中会详细介绍 resty 库），这些库的代码能够被 LuaJIT 编译，符合 OpenResty 的同步非阻塞的设计，可以作为那些不能被 LuaJIT 编译的官方库的替换。

- #### cosocket

Cosocket 可能是OpenResty 世界中技术、实用价值最高部分。它在提升程序处理性能同时，让程序员们能够以低廉的成本，优雅的姿势编写出比传统的 socket 效率高好几倍的程序。

> cosocket = coroutine + socket
>
> coroutine：协同程序（简称：协程） socket：网络套接字

* cosocket 原理图解

在 Lua 世界中调用任何一个有关 cosocket 网络函数内部关键调用如图所示：

![img](https://ciloa.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDJkMTlkYWY5ZTZmZWU4Y2JiYjQ3MDE5ZTRmNWE2OTFfVUZTWjdObXl3YzN4NzMyZXVOYzQ5MUg2Qm96M1NBY0NfVG9rZW46Ym94Y254emg4UXhIMUNLbTBtVTdVS09FUTlnXzE2MzI3MjM4MTg6MTYzMjcyNzQxOF9WNA)

从该图中我们可以看到，用户的 Lua 脚本每触发一个网络操作，都会有协程的 yield 以及 resume，因为请求的 Lua 脚本实际上都运行在独享协程之上，可以在任何需要的时候暂停自己（yield），也可以在任何需要的时候被唤醒（resume）

暂停自己，把网络事件注册到 Nginx 监听列表中，并把运行权限交给 Nginx。当有 Nginx 注册网络事件达到触发条件时，唤醒对应的协程继续处理

显然，cosocket 是依赖 Lua 协程 + Nginx 事件通知两个重要特性拼的

由上图也可看出，cosocket 是同步非阻塞的。

- ##### cosocket 在OpenResty 中的应用

  官方默认绑定包有众多是基于 cosocket 的，如：

  - [ngx_stream_lua_module](https://github.com/openresty/stream-lua-nginx-module#readme) Nginx "stream" 子系统的官方模块版本（通用的下游 TCP 对话）
  - [lua-resty-memcached](https://github.com/openresty/lua-resty-memcached) 基于 ngx_lua cosocket 的库。
  - [lua-resty-redis](https://github.com/openresty/lua-resty-redis) 基于 ngx_lua cosocket 的库。
  - [lua-resty-mysql](https://github.com/openresty/lua-resty-mysql) 基于 ngx_lua cosocket 的库。
  - [lua-resty-upload](https://github.com/openresty/lua-resty-upload) 基于 ngx_lua cosocket 的库。
  - [lua-resty-dns](https://github.com/openresty/lua-resty-dns) 基于 ngx_lua cosocket 的库。
  - [lua-resty-websocket](https://github.com/openresty/lua-resty-websocket) 提供 WebSocket 的客户端、服务端，基于 ngx_lua cosocket 的库。

- #### LuaNginxModule

  ​		OpenResty 扩展了 Nginx 的模块，这些模块就被称为 LuaNginxModule，比如我们最常使用的 content_by_lua，又比如添加权限控制的 access_by_lua，正是基于这些优秀的第三方模块，才得以让 OpenResty 成为一个功能强大的 Web Server。

  - ##### 执行阶段

    ​		对于这些被 OpenResty 扩展的模块，每个模块有其处理顺序，这些模块执行顺序，即是 OpenResty 的执行阶段，OpenResty 遵照既定的执行阶段处理每一个 LuaNginxModule。每个请求的处理流程都遵照这个顺序。

    ​		![](C:\Users\ryan.zhu.WUXI\Downloads\飞书20210927-143923.png)

- 不同阶段变量的传递与共享

试想我们的 Web Server 应用中，必然需要做一些鉴权的操作，鉴权的逻辑我们可以另写一个 lua 文件，并利用 access_by_lua 模块，在OpenResty 执行 content_by_lua 之前执行，执行过程我们也许会通过客户端上送的信息验证用户身份，取得用户的身份信息，这些身份信息，在后续 content_byt_lua 模块处理中极有可能也是需要的。

对此，我们可以使用 ngx.ctx 变量作为存储，ngx.ctx 是全局变量，OpenResty 中全局变量是所有模块共享的（OpenResty 开发中，通常我们不主动声明全局变量，仅使用 OpenResty 核心提供的全局变量。代码开发中所有的变量声明都应该加上 local 声明为局部变量）， 生命周期是整个请求周期。

- #### 第三方 resty 库

resty 库是第三方基于 OpenResty 开发的优秀的开源库。

- ##### 加载第三方 resty 库

是OpenResty Nginx 配置中，可以配置 lua_package_path 来告知 Nginx 第三方 lua 包所在的目录，Nginx 会自动加载该目录下的所有 lua 包。

- ##### 关键 resty 库介绍

  - [lua-resty-core](https://github.com/openresty/lua-resty-core)

    ​	OpenResty 的核心库，安装的 OpenResty 中默认加载。

  - [lua-resty-websocket](https://github.com/openresty/lua-resty-websocket)

    ​	Web 应用长连接开发使用

  - [lua-resty-redis](https://github.com/openresty/lua-resty-redis)

    ​	redis 连接与操作类库

  - [lua-resty-mysql](https://github.com/openresty/lua-resty-mysql)

    ​	Mysql 连接与操作类库

  - [lua-resty-waf](https://github.com/p0pr0ck5/lua-resty-waf)

    ​	防火墙应用

  - [lua-resty-mongo](https://github.com/Olivine-Labs/resty-mongol/tree/master/src)

    ​	MongoDB 连接与操作类库

  - [lua-resty-lrucache](https://github.com/openresty/lua-resty-lrucache)

    ​	LRU 算法缓存数据

- #### OpenResty 数据共享 Shared dict

  ​	Shared dict 是OpenResty作为数据共享的模块，可以实现多个 Nginx worker 共享数据，数据存储在内存中。

  - ##### 配置

    syntax：lua_shared_dict <name> <size>

    ```nginx
    http {
        lua_shared_dict dogs 10m; 
        ...
    }
    ```

  - ##### 优势

    ​	配置简单，不需要额外安装其它应用，或做额外的配置，可直接使用，且使用简单、直接。

  - ##### 劣势

    ​	shared.DICT 不是队列，不能将其应用到 生产者/消费者模式 中，对比完善的缓存应用 redis，性能、功能的多样性上没有优势，不建议作为项目缓存应用使用。

- #### 包罗万象的变量 ngx

  ​	ngx 变量就好像我们在开发中使用的 util 包，OpenResty 将许多核心的函数、系统变量都存放在了 ngx 变量中，如：

  - ngx.ctx

    ​	ngx_lua 模块上下文，变量作用域属于单个 location，当 location 中有多个 lua 模块时，任一模块中的 ngx.ctx 指向同一个地址（于是依照执行阶段，上一阶段对 ngx.ctx 做的变更，能够在下一阶段执行的模块中正确体现）。

    ​	如：

    ```nginx
    location /test_ctx {
        rewrite_by_lua_block {
            ngx.ctx.foo = 660
        }
        access_by_lua_block {
            ngx.ctx.test = 6
            ngx.ctx.foo = ngx.ctx.foo + ngx.ctx.test
        }
        content_by_lua_block {
            ngx.ctx.result = ngx.ctx.foo
            ngx.say(ngx.ctx.result)
        }
    }
    ```

  - ngx.say

    ​	应用于服务端相应的数据输出。

  - ngx.re

    ​	OpenResty 的正则匹配模块，考虑所有代码需要被 `luaJIT`热编译，因此正则匹配我们往往使用 ngx.re 而非 lua 语言自带的正则匹配

  - ngx.shared.DICT

    ​	Shared DICT 模块

  - ngx.worker

    ​	Nginx worker 信息

  众多 OpenResty 的核心库都绑定到了 ngx 变量中，关于 ngx 变量中的API的更多说明可参考：[lua_nginx_API](https://openresty-reference.readthedocs.io/en/latest/Lua_Nginx_API)

- #### FFI

  FFI 库是 LuaJIT 中最重要的一个扩展库，它允许从纯 Lua 代码调用外部 C 函数，使用 C 数据结构，`JIT` 编译器在 C 数据结构上所产生的代码，等同于一个 C 编译器应该生产的代码。

  - ##### 编译器动态库路径

    ​	编译器在查找动态库所在的路径的时候会在`LD_LIBRARY_PATH` 这个环境变量中的所有路径进行查找，因此生成的 C 的动态链接库最好存放在这个目录下集中管理，这样才能被编译器找到。

  - ##### 调用示例	

    ```lua
    local ffi = require "ffi"
    local myffi = ffi.load('myffi')
    
    ffi.cdef[[
    int add(int x, int y);   /* don't forget to declare */
    ]]
    
    local res = myffi.add(1, 2)
    ```

    - 第一步 require 是必须的，加载 ffi 模块
    - 第二步 则是加载动态链接库，这一步中，ffi.load 方法的第一个参数 myffi 也可以是动态库的完整路径，当它是一个路径时，编译器直接查找该路径，否则，编译将到默认的`LD_LIBRARY_PATH` 路径下查找该名称的动态库。
    - 第三步 ffi.cdef 是定义 C 函数，我们在完成加载动态链接库之后，需要在 Lua 代码中定义该方法才能真正完成加载到 Lua 代码中
    - 第四步 执行定义好的 add 方法

    此外，我们也可以将需要加载的动态链接库作为一个系统函数，那么加载时需要使用全局加载，即在 load 时，加入第二个参数，并传参 true，以下是示例：

    ```lua
    local ffi = require "ffi"
    ffi.load('myffi', true)
    
    ffi.cdef[[
    int add(int x, int y);   /* don't forget to declare */
    ]]
    
    local res = ffi.C.add(1, 2)
    ```

参考资料：

- [OpenResty 官网](https://openresty.org/en/)
- [章亦春访谈实录](https://www.oschina.net/question/28_60461)
- [agentzh 的 Nginx 教程（章亦春编写的优秀 Nginx 教程）](http://openresty.org/download/agentzh-nginx-tutorials-zhcn.html#00-Foreword01)
- [OpenResty 最佳实践](https://www.kancloud.cn/allanyu/openresty-best-practices/82569)
- [Kong 官网（基于OpenResty 开发的网关应用）](https://konghq.com/)