# BIRD 与 BGP 的新手开场

*版本：1.0-20200627.1*

本文以 [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 License](https://creativecommons.org/licenses/by-nc-sa/4.0/) 进行授权。

本文中使用的 BIRD 版本为撰写此文时的最新版，BIRD 2.0.7。目前 BIRD 1.x 和 2.x 同时在维护，区别是 1.x 中 IPv4 和 IPv6 协议是分开的（bird 和 bird6），而 2.x 将两部分代码合并在了一起且引入了更多功能。但是 1.x 本身也是落后技术终将被淘汰，所以我推荐使用 BIRD 2.x。两个版本的语法并无差别，无非就是 2.x 在和路由表相关的操作的时候，需要指定特别的协议，如 `ipv4; ipv6;`。

本文追求的是为有心学习的人提供一个更容易达到的起点。如果只是为了最简单的装逼或者说这篇文章完全看不懂，建议还是别看下去了，可能[蔡老师的 RouterOS 版本](https://www.91yunbbs.com/discussion/641)更适合你。

本文因为是 BIRD 与 BGP 的快速起步，因此不会介绍 BGP 协议，也只会介绍实践部分中用得到的 BIRD 配置，并稍微扩展介绍一些其他 BIRD 的功能，更多的内容则建议在阅读完本文后试着去阅读[官方文档](https://bird.network.cz/?get_doc&f=bird.html&v=20)，以更深入地理解。

BIRD 官方也提供了一些样例，可以去 [BIRD 的 GitLab Wiki](https://gitlab.labs.nic.cz/labs/bird/-/wikis/Examples) 查看。

## 目录

### 正文

1. [什么是 BIRD](#1-什么是-bird)
2. [BIRD 的基本语法](#2-bird-的基本语法)
    1. [杂项](#杂项)
    2. [变量与常量](#变量与常量)
    3. [操作符](#操作符)
    4. [分支](#分支)
3. [过滤器和函数](#3-过滤！过滤！)
    1. [过滤器 `filter`](#过滤器-filter)
    2. [函数 `function`](#函数-function)
    3. [过滤器写什么呢](#过滤器写什么呢)
    4. [路由属性](#路由属性)
4. [协议实例](#4-协议实例)
    1. [频道 `channel`](#频道-channel)
    2. [协议 `protocol`](#协议-protocol)
    3. [协议模板 `template`](#协议模板-template)
5. [杂七杂八的东西](#5-杂七杂八的东西)
    1. [更多配置](#更多配置)
    2. [控制台](#控制台)
6. [实践中学习](#6-实践中学习)
7. [结语](#7-结语)

### 附录

1. [BIRD 全局选路规则](#附录-1-bird-全局选路规则)
2. [BIRD BGP 协议选路规则](#附录-2-bird-bgp-协议选路规则)
3. [BIRD 的包含（`~`）操作符](#附录-3-bird-的包含操作符)
4. [常用的各种配置](#附录-4-常用的各种配置)
    1. [公网上约定俗成的最小前缀](#公网上约定俗成的最小前缀)
    2. [Bogon Prefixes and ASNs](#bogon-prefixes-and-asns)
    3. [与 Vultr 的 BGP session](#与-vultr-的-bgp-session)
    4. [与 HE TunnelBroker 的 BGP session](#与-he-tunnelbroker-的-bgp-session)

----------

## 1 什么是 BIRD

BIRD 是一个实现多种动态路由协议（如 OSPF、BGP、RIP 等）的软件，相似的软件有 FRR（Quagga）、MRT。在 BGP 玩家常用的软件中，BIRD 因其内存占用小、管理方便大受欢迎。

在 BIRD 中，有一个存储了从各个协议收到的路由（包括静态路由）的 RIB（Route Information Base，路由表），经过选路规则的挑选，产生最优的一条路由，这条最优的路由会被写入 FIB（Forward Information Base，转发表）中，导出给系统（如 Linux 内核）中的转发表，这样发到该路由器的流量就能被继续转发了。

*注：在高阶玩法中可以使用多个 RIB，在协议中可以指定其使用哪个 RIB，在 BIRD 中也可以指定 RIB 对应的是内核中的哪个 FIB。为了简洁明白，本文只使用 BIRD 的默认 RIB，和内核的默认 FIB。*


## 2 BIRD 的基本语法

和 Juniper、Cisco 等路由器，或 FRR（Quagga）等路由软件不同，写 BIRD 的配置文件就像是在写程序，如果你是个程序员，那么上手应该会很快。正因如此，它也有着和常见编程语言所类似的语法。下面则是一些基础语法。

### 杂项

用 `/* */` 包起来的内容是注释，`#` 至其所在行行末的内容也是注释。

分号 `;` 标示着一个选项或语句的结束，而花括号 `{ }` 内则是多个选项或语句。

在 BIRD 的配置文件中，有协议实例（`protocol <proto> <name> {}`），过滤器（`filter <name> [local_variables] {}`），函数（`function <name> [local_variables] {}`）可以定义，这些将在下文各部分选择性挑重点介绍。

`print` 用来输出内容，这些会输出在 BIRD 的日志文件中，在使用 systemd 的系统中，可以使用 `journalctl -xeu bird.service` 查看。

### 变量与常量

变量名、常量名、协议实例名、过滤器名、函数名等，都遵循着这样的规则：必须以下划线或字母开头，名称内也只能有字母、数字、下划线。比如 `Soha233`、`_my_filter`、`bgp_4842` 都是合法的名字。当然在 BIRD 中有例外，如果一个名字用单引号括起来，那么我们还可以用冒号、横线、点，比如 `'2.333:what-a-strange-name'`，只不过不推荐这么用就是了。

使用 `define` 定义常量，如 `define LOCAL_AS = 65550`。

BIRD 中可以针对很多变量类型定义集合。集合用一对方括号定义，如 `[1, 2, 3, 4]`。集合可以用范围来快速生成，比如 `[1, 2, 10..13]` 就会生成为 `[1, 2, 10, 11, 12, 13]`。范围的写法还可以用在社区属性中，如 `[(64512, 100..200)]`，在社区属性中还可以用通配符，如 `[(1, 2, *)]`。

前缀的集合中的范围写法较为复杂，`[prefix{low, high}]`，`prefix` 是一个用于匹配的前缀，`low` 和 `high` 两个值限制了它的 CIDR 长度。`[192.168.1.0/24{16,30}]` 表示的是包含或被包含于 192.168.1.0/24 且 CIDR 在 16-30 之间的前缀，例如 `192.168.0.0/20` 和 `192.168.1.0/29` 均属于这个集合，而 `192.168.233.0/24` 不属于。这样子的写法可能过于麻烦，所以 BIRD 中也使用加号和减号提供了两种便捷的写法，如 `[2001:db8:10::/44+, 2001:db8:2333::/48-]` 则等价于 `[2001:db8:10::/44{44,128}, 2001:db8:2333::/48{0,48}]`。

在 BIRD 中还有一类特殊的变量类型，他们都是列表，`bgppath`（AS Path，路由的 `bgp_path` 属性）、`clist`（BGP Community 列表，路由的 `bgp_community` 属性）、`eclist`（BGP Extended Community 列表，路由的 `bgp_ext_community` 属性）、`lclist`（BGP Large Community 列表，路由的 `bgp_large_community` 属性）都是这类变量，他们的操作有非常特殊的用法。下面的代码展示了 `bgppath` 的用法。`clist/eclist/lclist` 与之类似，但是它们只能使用其中的 `empty`、`len`、`add`、`delete`、`filter`。

```
function foo()
bgppath P;
bgppath P2; {
    print "path 中第一个元素是", P.first, "，最后一个元素是", P.last;
    # 第一个元素可以认为是邻居的 ASN，最后一个元素是宣告这条路由的 ASN
    # 这两个在 P 中没有元素的时候是 0
    print "path 的长度是", P.len;
    if P.empty then {
        print "path 为空";
    }
    P.prepend(233); # 在 path 的第一个位置插入元素
    P.delete(233); # 删除 path 中所有等于 233 的元素
    P.delete([64512..65535]); # 删除 path 中所有属于集合 [64512..65535] 的元素
    P.filter([64512..65535]); # 只在 path 中留下集合 [64512..65535] 中出现的元素

    # 如果不想改变 P，可以使用下面这样的方法将操作后的结果存入 P2
    P2 = delete(P, 233)
    P2 = filter(P, [64512..65535])
}
```

变量只能定义在函数或过滤器的最开头（左花括号外面），关于变量类型的更详细信息，请移步[官方文档相关部分](https://bird.network.cz/?get_doc&v=20&f=bird.html#ss5.2)。

### 操作符

在 BIRD 中有很多常见的操作符。如 `+`、`-`、`*`、`/`、`()` 这些基本的算数操作符，有等于 `a = b`、不等于 `a != b`、大于 `a > b`、大于等于 `a >= b`、小于 `a < b`、小于等于 `a <= b` 这些比较符，有与 `&&`、或 `||`、非 `!` 这三种逻辑操作符。还有 `~` 和 `!~` 这两种判断包含或者不包含的操作符。包含操作符的用法写在附录 3 中。

### 分支

过滤器和函数中的语句都是顺序执行的。同时支持 `case` 和 `if` 两种分支语句。在 BIRD 中是不支持循环的。

`if` 的写法如下：

```
if 6939 ~ bgp_path then {   # 只要 AS Path 中有 6939
    bgp_local_pref = 233;   # 就将这条路由的 Local Preference 调为 233
} else {
    bgp_local_pref = 2333;  # 否则设为 2333
}
```

`case` 的写法如下：

```
case arg1 {
    2: print "two"; print "I can do more commands without {}";
    #  ^ case 不需要花括号就能在一个分支中写下更多语句。
    3..5: print "three to five";
    else: print "something else";
}
```

在 BIRD 中，if 和 case 的写法均与常见编程语言略有不同。


## 3 过滤！过滤！

**为什么讲完基本语法就开始讲过滤，因为过滤是一个非常重要的内容！请记住，永远不要对过滤不上心，不漏路由（指把不应该发出去的路由发出去）是最重要的！**

首先要知道的是，当你敲下 `export` 的时候，操作的对象是**整个**路由表，只要是路由表里有的（不管是 static 协议、BGP 协议，还是 OSPF 协议）都是会被发出去的，所以过滤做的事情就是挑选路由表中应该发出去的表，把不应该发出去的表挡在自己的路由器内。`import` 同理，只不过是从外面挑选什么路由应该被导入到我们的路由器上。

### 过滤器 `filter`

我们先来看一个过滤器的例子。

```
filter sample_import
int set reject_origin_asn; {
# ^ 定义一个整数集
    reject_origin_asn = [64512..64519, 65500];
    # ^ 这里的变量定义没啥意义，只是给大家了解可以定义局部变量

    if net_len_too_long() then reject; # 调用函数 net_len_too_long，如果为真就 reject
    if bgp_path.last ~ reject_origin_asn then {
        reject; # 不接收上面定义的 reject_origin_asn 里面任何一个 ASN 宣告的路由
    }
    accept;
}
```

过滤器是用在导入和导出路由的时候的。过滤器会对每一条路由执行一次，同时该路由的所有属性都可以直接在过滤器中被使用（如例子中的 `bgp_path`），当前路由的可编辑属性（如 `bgp_local_pref`、`bgp_path`）也可以在这里被编辑。对于可编辑属性，如果是导入时候的过滤，那么修改后的属性会随着路由存入 BIRD 内的路由表；如果是导出时的过滤，那么修改后的属性只会传播给导出对象，BIRD 路由表中的属性不会被修改。顺便一提，属性还需要是可传播的，这样才能传播给导出对象，导出时的修改才有意义。

`reject` 语句和 `accept` 语句决定了这条路由是被拒绝还是被接受。如果导入路由的时候拒绝或接受，那么路由会被丢弃、或存入路由表；如果导出的时候拒绝或接受，那么这条路由不会被导出或会被导出。只要 `reject` 或者 `accept` 被触发，过滤器就会立刻退出，不执行之后的任何代码。

过滤器可以在协议实例的 `import filter` 或 `export filter` 选项中使用，如果一个过滤器只在一个地方用，我们也可以使用“匿名过滤器”的形式，如下面的例子所示。

```
protocol bgp {
    /* 略去一些内容 */
    ipv6 {
        import filter sample_import; # 使用名为 sample_import 的过滤器
        export filter {
            if net ~ [2001:db8::/32{40,48}] then accept;
            reject;
        }; # 一个匿名的过滤器，只导出 2001:db8::/32 下面 CIDR 为 40 到 48 的前缀
    };
}
```

### 函数 `function`

下面是一个函数的例子。

```
function net_len_too_long(int hello)
# 这里定义的参数 hello 没啥意义，只是让大家知道可以定义参数，不定义括号内留空即可
int world; {
# ^ 这里的变量定义没啥意义，只是让大家知道可以定义局部变量
    case net.type {
        NET_IP4: return net.len > 24; # IPv4 CIDR 大于 /24 为太长
        NET_IP6: return net.len > 48; # IPv6 CIDR 大于 /48 为太长
        else: print "net_len_too_long: unexpected net.type ", net.type, " ", net; return false;
    }
}
```

函数可以减少在过滤器之间的冗余代码，可以在过滤器中被调用。和过滤器中一样，当前路由的所有属性都可以直接在函数中被使用（如例子中的 `net`），可编辑属性可以在这里被编辑。`return` 可以返回任何值或不返回值，省略 `return` 也是可以的。`return` 后该函数即退出，不会执行之后的任何代码。这个值可以在过滤器中使用，如上面过滤器中的例子，就调用了 `net_len_too_long()` 来判断 CIDR 是否太长。

除了在过滤器中引用，在协议实例中也可以使用匿名过滤器来调用函数，例子是 `export where net_len_too_long();`。此时函数返回值必须为 `bool` 类型，即 `true` 和 `false`。

### 过滤器写什么呢

写过滤器的时候我们首先需要知道，我们要怎么发路由。在自己网络的内部，我们想怎么发都行，就算是直接写 `export all;` 也可以。但是和其他网络之间的 BGP session，我们就必须写好过滤器，只把正确的路由发送出去。

下面介绍几个常见的场景：

- 向上游（transit）导出

  只导出自己和下游（如果有）的路由
- 向对等伙伴（peer）导出

  只导出自己和下游（如果有）的路由
- 向下游（customer）导出

  导出自己、上游、对等伙伴的路由

  *这种情况中，自己就是下游的上游（transit）*

### 路由属性

上文提到，当前路由的所有属性都可以直接在过滤器和函数中被使用，那么有什么属性比较常用呢？

- `net`，类型为 `prefix`，就是“当前路由”
- `preference`，类型为 `int`
- `proto`，类型为 `string`，是协议实例的名字
- `source`，它的取值是这几个系统内置的常量 `RTS_DUMMY, RTS_STATIC, RTS_INHERIT, RTS_DEVICE, RTS_STATIC_DEVICE, RTS_REDIRECT, RTS_RIP, RTS_OSPF, RTS_OSPF_IA, RTS_OSPF_EXT1, RTS_OSPF_EXT2, RTS_BGP, RTS_PIPE, RTS_BABEL`，代表路由来源的协议类型
- 还有一些常用的属性（如 `bgp_path`）是协议定义的，请看协议相关的内容

如果想看更多的请看[官方文档的相关内容](https://bird.network.cz/?get_doc&v=20&f=bird.html#ss5.5)。


## 4 协议实例

一个协议实例（protocol instance）定义了使用什么协议、什么参数进行路由交换。BIRD 所有支持的协议和相关的参数，都可以在[官方文档相关内容](https://bird.network.cz/?get_doc&v=20&f=bird-6.html)中找到。

定义一个协议实例使用这样的语法：

```
protocol <协议> [实例名] {
    /* 参数们 */
    <频道> {
        /* 频道参数 */
    }
}
```

实例名是可以省略的，当然为了自己方便还是写上名字好。那么频道是什么呢？

### 频道 `channel`

简单的理解的话，一个频道就是一个网络协议，常用的频道为 `ipv4` 和 `ipv6`。当然 BIRD 支持的远不止这些。我们通常像这样来写：

```
protocol some_proto {
    /* 略 */    
    ipv6 {
        export filter some_filter1;
        import filter some_filter2;
        export limit 100 disable; # 导出的路由数量多于 100 后自动停止该协议，避免传出过多路由（往往这个时候是漏路由了）
        import limit 100 restart; # 导入的路由数量多于 100 后自动重启该协议，限制对端传入的路由数
    };
}
```

一般的，频道默认 `export none;` 和 `import all;`，所以当你用在 static 等协议中，可以不明确写出 filter。

频道的具体参数请参照[官方文档](https://bird.network.cz/?get_doc&v=20&f=bird-3.html#ss3.4)。

### 协议 `protocol`

这里简单介绍 BGP 协议、static 协议（静态路由）、direct 协议的常用参数，很多用不到的内容都被省略了，如果需要了解更多请查阅官方文档的 [BGP 协议部分](https://bird.network.cz/?get_doc&v=20&f=bird.html#ss6.3)、[static 协议部分](https://bird.network.cz/?get_doc&v=20&f=bird-6.html#ss6.14)和 [direct 协议部分](https://bird.network.cz/?get_doc&v=20&f=bird-6.html#ss6.5)。

#### static 协议

一个 static 协议中可以描述数条路由。常用的配置如下：

```
protocol static {
    ipv6; # 启用 ipv6 channel，否则不会收集 IPv6 路由

    route 2001:db8:100a::/48 reject;
    # 定义一条路由 2001:db8:100a::/48 为 reject/unreachable
    route 2001:db8:100b::/48 via "eth0";
    # 定义一条路由 2001:db8:100b::/48 的下一跳为 eth0
    route 2001:db8:100c::/48 via 2001:db8:eeee::1;
    # 定义一条路由 2001:db8:100c::/48 的下一跳为 2001:db8:eeee::1
}
```

#### direct 协议

direct 协议用来直接从内核的网络设备上获取地址和路由，并将其导入到 BIRD 的路由表中。

```
protocol direct {
    ipv4; # 启用 ipv4 channel，否则不会收集 IPv4 路由
    ipv6; # 启用 ipv6 channel，同上

    interface "eth*", "tun*";
    # 如果不写这个参数，那么 BIRD 会默认使用所有网络设备
    # 参数后面用逗号分隔不同的匹配字符
    # “*”表示通配符
    # 更详细的参数参照官方文档
    # https://bird.network.cz/?get_doc&v=20&f=bird-3.html#proto-iface
}
```

#### BGP 协议

```
protocol bgp {
    local as 65550;
    # 指定自己的 ASN 为 65550

    source address 2001:db8:ffff::6:5550;
    # 指定 BIRD 发起 BGP 会话的源地址

    neighbor 2001:db8:ffff::6:4501 as 64501;
    # 指定对端的 ASN 为 64501，IP 为 2001:db8:ffff::6:4501
    # 如果 ASN 和 local as 相同，那么 BIRD 会自动认为这是一个 iBGP，否则是 eBGP
    # i 表示 internal（内部），e 表示 external（外部）

    direct;
    # eBGP 默认启用可以不写
    # iBGP 如果是直接连接的可以写这个来避免 multihop 被指定

    multihop 2;
    # 如果自己端到对端不是直接连接的，需要指定 multihop 的值为到达对端所需要经过的跃点数
    # eBGP 默认不启用，而是 direct
    # iBGP 默认启用
    # 例如：
    # 在 Vultr 上配置的时候，对端并不是直接连接的，而是需要经过一个网关，那么这时候需要指定 multihop 为 2
    # 如果是 HE 或者别的服务商提供的隧道，一般都是直接连接的，那么这时候就不需要这个参数了
    # 具体的跃点数使用 traceroute 追踪对端即可

    password "Aa1";
    # 如果和对端约定了密码，在这里配置约定好的密码，否则不用写

    graceful restart on;
    # 配置 BGP 的 graceful restart
    # 如果对端因为网络抖动或暂时崩溃而暂时下线，会导致所有传入路由瞬间消失
    # 为了避免这种情况下数据转发中断，才有 graceful restart
    # 建议打开

    # rr client;
    # 自己作为路由反射器（Route Reflector，RR），对端为被反射的路由器
    # 此项请在阅读路由反射器相关的文献资料后使用

    <频道> {};
}
```

另外提一笔 iBGP，iBGP 是一个自治系统内部的 BGP 会话。为了建设全球性的网络，我们可能会选择通过 iBGP 的方式交换各个路由器收到的路由，以获得最佳的上网体验。在 BGP 中，为了避免产生环路，从 iBGP 收到的路由表不会传播给别的 iBGP，这就要求网络内所有 iBGP 路由之间必须采用全连接的形式（即每两台路由器都要起 iBGP 会话）。

一般 BGP Player 的网络一般不会这么大，4 台路由器也就是 6 个会话，并不是多么麻烦的事情。但是在较大的网络，比如 10 台路由器就要配置 45 个会话，这样配置和维护起来就非常吃力了。为了避免产生额外的连接成本，可以使用路由反射器（Route Reflector，RR）。另外一种情况是，如果某些路由器不方便加入全连接，也可以使用 RR 的形式。因为大部分玩家并不会有如此大型的网络，此处只是提到 RR，具体内容还请有兴趣的自行检索资料学习。

在附录 4 中介绍了一些常见情况下的协议实例配置样例。

BGP 协议中的过滤器可以使用这些属性：

- `bgp_path` AS Path
- `bgp_local_pref` Local Preference
- `bgp_med` Multiple Exit Discriminator
- `bgp_community` BGP Community
- `bgp_large_community` BGP Large Community
- 等

这么看起来，只要配置多个实例就很容易产生很多重复的代码，例如 BGP 实例中的 `local as` 参数等，我们这时候就可以用协议模板减少重复。

### 协议模板 `template`

在协议模板中定义的参数将被协议实例继承，如果模板中的参数在实例中再一次被定义，那么模板中的参数将被覆盖。

在下面这个例子中我们就定义了这样一个协议模板 `tpl_ibgp`，在 `ibgp_123` 实例中用 `from tpl_ibgp` 继承模板。然后覆盖了 IPv6 channel 中的 import 参数。

```
template bgp tpl_ibgp {
    graceful restart on;
    local as 65550;
    med metric;
    direct; # 如果 iBGP 是直接连接的（比如使用 GRE 隧道直接连接）就需要写这个，否则需要针对 session 指定 multihop
    ipv4 {
        next hop self;
        gateway direct;
        import all;
        export all;
    };
    ipv6 {
        next hop self;
        gateway direct;
        import all;
        export all;
    };
}
protocol bgp ibgp_123 from tpl_bgp {
    interface "tun1";
    neighbor fe80::1 as 65550;
    ipv6 {
        import none;
    }
}
```

## 5 杂七杂八的东西

BIRD 配置还有一些必要的内容，所以放在这里一并列出。

### 更多配置

```
# 启用日志并记录到 syslog，或者文件
log syslog all;
# log "/var/log/bird.log" all;

# 路由器识别号，32 位整数
# 一般是全球唯一的，所以建议使用自己的公网 IPv4 地址
router id 198.51.100.1;

# device 协议必须有，否则 BIRD 不会自动从内核获取比如网络接口的信息，direct 协议和寻找下一跳的时候就挂了
protocol device {}


# kernel 协议用于到处路由表到内核，这里列出了常见的配置，详细的请看官方文档
# https://bird.network.cz/?get_doc&v=20&f=bird.html#ss6.6
# IPv4 的 kernel 协议，用于导出 IPv4 路由表到内核用于数据转发
protocol kernel {
    ipv4 { export all; };
}

# IPv6 的 kernel 协议，用于导出 IPv6 路由表到内核用于数据转发
protocol kernel {
    ipv6 { export all; };
}
```

### 控制台

BIRD 提供了一个控制台工具，它叫做 `birdc`。

```
# birdc
BIRD 2.0.7 ready.
bird> ?
quit                                           Quit the client
exit                                           Exit the client
help                                           Description of the help system
show ...                                       Show status information
dump ...                                       Dump debugging information
eval <expr>                                    Evaluate an expression
echo ...                                       Control echoing of log messages
disable (<protocol> | "<pattern>" | all) [message]  Disable protocol
enable (<protocol> | "<pattern>" | all) [message]  Enable protocol
restart (<protocol> | "<pattern>" | all) [message]  Restart protocol
reload <protocol> | "<pattern>" | all          Reload protocol
debug ...                                      Control protocol debugging via BIRD logs
mrtdump ...                                    Control protocol debugging via MRTdump files
restrict                                       Restrict current CLI session to safe commands
configure ...                                  Reload configuration
down                                           Shut the daemon down
graceful restart                               Shut the daemon down for graceful restart
bird>
```

`configure` 可以在修改配置之后让 BIRD 热重载配置文件，避免重启整个 BIRD。

`show protocols` 可以用来查询协议实例，它默认会显示所有的协议实例，后面加上 `all` 则会显示出所有细节信息。如果后面再加上协议实例的名字或者用来匹配的模式，那么只会显示具体的一个或几个协议实例的信息，就像 `show protocols all "bgp_*"` 会显示出所有名字开头是“bgp_”的协议实例的详细信息。

`show route` 可以用来查询 BIRD 中的路由表，默认显示的都是简略信息，后面加上 `all` 就会显示出所有细节信息。它可以在后面加上 `for <前缀或 IP>` 来查询所有包含这个前缀或 IP 的路由，当然如果不加 `for` 就是完全匹配了。使用 `export <协议实例名>` 可以查看在指定的协议实例中导出的路由。还可以使用 `where <语句>` 来查找使 `where` 后面语句为真的路由。

`birdc` 的命令在不引起歧义的情况下可以不完整写出，比如 `s p` 等价于 `show protocols`。其他功能和命令大家可以自行摸索。

当然直接在 `birdc` 后面追加命令，也是可以执行的。像下面这样。

```
# birdc show proto
BIRD 2.0.7 ready.
Name       Proto      Table      State  Since         Info
device1    Device     ---        up     2020-06-18
kernel1    Kernel     test4      up     2020-06-18
static1    Static     test4      up     2020-06-18
ospf1      OSPF       test4      up     2020-06-18    Alone
```


## 6 实践中学习

### 场景

我们维护一个自治系统 AS65550，它拥有一个 IPv6 前缀 `2001:db8:1000::/44`、三台路由器。

这个自治系统将要在一台路由器（Router ID 10.0.0.1）上与 AS64501、AS64502 互联。AS64501 是上游（transit），AS64502 是一个对等伙伴（peer）。

这台路由器上通过 GRE 隧道连接到了路由器 A，隧道的名字是 `tun-a`。当前路由的 IPv6 地址为 `2001:db8:eeee:a::1/64`，路由器 A 的为 `2001:db8:eeee:a::a/64`。

这台路由器上通过 GRE 隧道连接到了路由器 B，隧道的名字是 `tun-b`。当前路由的 IPv6 地址为 `2001:db8:eeee:b::1/64`，路由器 B 的为 `2001:db8:eeee:b::b/64`。

我们的自治系统和 AS64501 在同一子网，我们的地址是 `2001:db8:ffff:1::6:5550/64`，对端是 `2001:db8:ffff:1::6:4501/64`。

我们的自治系统和 AS64502 在同一子网，我们的地址是 `2001:db8:ffff:2::6:5550/64`，对端是 `2001:db8:ffff:2::6:4502/64`。

AS64501 有如下的 BGP Community 规则：
- 如果我们宣告的路由有 `(0, a)` 的 Community，那么不会被宣告到 ASN 为 a 的路由器

AS64501 有如下的 BGP Large Community 规则：
- AS64501 从 ASN=a 处收来的路由，会包含 `(64501, 1, a)` 的 Large Community

我们想要达到以下目标：

1. eBGP 过滤器只宣告 `2001:db8:1000::/44` 下 CIDR 长度在 44-46 之间的 static 协议路由
1. 宣告 `2001:db8:1000::/44`
1. 配置上游 AS64501 的 BGP session
1. 不让上游 AS64501 将我们的路由发到 AS65510（即添加 Community `(0, 65510)` 到导出的路由）
1. 使用 community 不接受上游 AS64501 发来的 AS65511 的路由（即拒绝所有 Large Community 包括 `(64501, 1, 65511)` 的路由）
1. 配置对等 AS64502 的 BGP session
1. 将 AS64502 收到的路由表的 Local Preference 均设为 1000
1. 使用静态路由将 `2001:db8:100a::/48` 指向路由器 A
1. 与路由器 B 配置 iBGP session，并交换所有路由表

### 配置

**下面的配置只是代表笔者的配置习惯。**

```
log syslog all;

router id 10.0.0.1;
define LOCAL_ASN = 65550;

protocol device {}

protocol kernel {
    ipv6 { export all; };
}

# 公网上约定俗成的最小前缀长度是 24（IPv4）和 48（IPv6），所以要在导出的时候过滤
function net_len_too_long(){
    case net.type {
        NET_IP4: return net.len > 24; # IPv4 CIDR 大于 /24 为太长
        NET_IP6: return net.len > 48; # IPv6 CIDR 大于 /48 为太长
        else: print "net_len_too_long: unexpected net.type ", net.type, " ", net; return false;
    }
}

# 目标 1
function bgp_export() {
    if net_len_too_long() then return false;
    if source != RTS_STATIC then return false;
    if net !~ [2001:db8:1000::/44{44,46}] then return false;
    return true;
}

# 目标 2
protocol static {
    ipv6;
    route 2001:db8:1000::/44 reject;
}

# 目标 8
protocol static {
    ipv6;
    route 2001:db8:100a::/48 via "tun-b";
}

# 使用模板减少重复代码
template bgp tpl_bgp {
    graceful restart on;
    local as LOCAL_ASN;
    ipv6 {
        import where !net_len_too_long();
        export where bgp_export();
    };
}
template bgp tpl_ibgp from tpl_bgp {
    med metric;
    direct;
    ipv6 {
        next hop self;
        gateway direct;
        import all;
        export all;
    };
}

# 目标 3
protocol bgp bgp_as64501 from tpl_bgp {
    source address 2001:db8:ffff:1::6:5550;
    neighbor 2001:db8:ffff:1::6:4501 as 64501;
    ipv6 {
        import filter {
            if net_len_too_long() then reject;
            if (64501, 1, 65511) ~ bgp_large_community then reject; # 目标 5
            accept;
        };
        export filter {
            if !bgp_export() then reject;
            bgp_community.add((0, 65510)); # 目标 4
            accept;
        }
    };
}

# 目标 6
protocol bgp bgp_as64501 from tpl_bgp {
    source address 2001:db8:ffff:2::6:5550;
    neighbor 2001:db8:ffff:2::6:4502 as 64502;
    ipv6 {
        import filter {
            if net_len_too_long() then reject;
            bgp_local_pref = 1000; # 目标 7
            accept;
        };
        # export 使用了 template 中的默认值
    };
}

# 目标 9
protocol bgp ibgp_b from tpl_ibgp {
    source address 2001:db8:eeee:b::1;
    neighbor 2001:db8:eeee:b::b as LOCAL_ASN;
}
```


## 7 结语

希望大家能在阅读之后，对如何使用 BIRD 配置 BGP 有了点概念，并能做出正确的配置。当然更希望大家能不止步于此，多学习计算机网络相关的基础知识，或阅读 BIRD 官方文档学到更全面的配置。

特别感谢 foobar 院的 twd2 对本文的审读与修改，感谢 foobar 院的 Martian、快乐 BGP 群的 alanyhq 试读本文并提供意见，感谢快乐 BGP 群的 ZX 对本文内容的完善与修正。

本文写成较快，虽然也有多人试读、审校，难免会有遗漏，相关英文术语的翻译也会有不合适的地方，如有问题、意见或者建议，请在 [issue](https://github.com/moesoha/bird-bgp-kickstart/issues) 中提出。

*（如果有需要申请 ASN 资源的请联系我的邮箱 `soha@lohu.info`。）*

----------


## 附录 1 BIRD 全局选路规则

按照从上到下的顺序执行。如果某条的值相同，那么执行下一条。这里描述的是 BIRD 实现的选路规则，其他路由软件可能不同。

1. 路由的 preference 高者优先
1. 协议实例的 preference 高者优先
1. 如果路由来源是相同协议，如 BGP 和 BGP 比较，参照该协议的选路规则（如附录 2 的 BIRD BGP 协议选路规则）
1. 如果路由来源是不同协议，如 BGP 和 OSPF 比较，这个是未定义行为，最优的选择方式不一定

## 附录 2 BIRD BGP 协议选路规则

按照从上到下的顺序执行。如果某条的值相同，那么执行下一条。这里描述的是 BIRD 实现的选路规则，其他路由软件可能不同。

1. Local Preference（`bgp_local_pref`） 高者优先
1. AS Path（`bgp_path`） 短者优先
1. Origin 属性（`bgp_origin`）中，IGP（`ORIGIN_IGP`）优先于 EGP（`ORIGIN_EGP`）优先于 incomplete（`ORIGIN_INCOMPLETE`）
1. MED（`bgp_med`，Multiple Exit Discriminator）值小者优先
1. 从 eBGP 收到的路由优先于 iBGP 收到的路由
1. 到边界路由器的内部距离小者优先
1. 宣告该路由的 Router ID 小者优先

## 附录 3 BIRD 的包含操作符

BIRD 的包含 `~` 和不包含 `!~` 操作符能用于下表中所示的类型。`~` 或者 `!~` 左边的东西叫做左操作数（是的，它不一定是数字，也可以是其它类型），右边的东西叫做右操作数。

| 左操作数类型          | 右操作数类型  | 样例                                            | 说明 |
|:-------------------:|:------------:|:-----------------------------------------------:|:----:|
| `bgppath`           | `bgpmask`    | `bgp_path ~ [= * 64512 64513 * =]`              | bgp_path 符合右边描述的模式则为真
| `int`               | `bgppath`    | `64512 ~ bgp_path`                              | 左边被包含于右边则为真
| `pair`/`quad`/`ip4` | `clist`      | `(123, 456) ~ bgp_community`                    | 左边被包含于右边则为真，`pair`、`quad`、`ip4` 都是 32 位的，所以它们等价
| `ec`                | `eclist`     | `(rt, 10, 3) ~ bgp_ext_community`               | 左边被包含于右边则为真
| `lc`                | `lclist`     | `(4842, 0, 0) ~ bgp_large_community`            | 左边被包含于右边则为真
| `string`            | `string`     | `proto ~ "bgp_*"`                               | 左边的字符串符合右边的模式（类 shell）就为真
| `ip`                | `prefix`     | `1.1.1.1 ~ 1.0.0.0/8`                           | IP 被包含于某前缀则为真
| `prefix`            | `prefix`     | `1.1.1.0/24 ~ 1.0.0.0/8{16,24}`                 | 左边的前缀被包含于右边的前缀则为真
| `prefix`            | `prefix set` | `1.1.1.0/24 ~ [1.0.0.0/24, 1.1.0.0/22]`         | 左边的前缀被包含于右边任意一个前缀则为真
| `path`              | `int set`    | `bgp_path ~ [233, 2333, 64512]`                 | 左右两边交集非空即为真
| `clist`             | `pair set`   | `bgp_community ~ [(0, 6939), (1, 2333)]`        | 左右两边交集非空即为真
| `eclist`            | `ec set`     | `bgp_ext_community ~ [(rt, 1, 30), (ro, 2, *)]` | 左右两边交集非空即为真
| `lclist`            | `lc set`     | `bgp_large_community ~ [(1, 2, 100..233)]`      | 左右两边交集非空即为真


## 附录 4 常用的各种配置

### 公网上约定俗成的最小前缀

公网上约定俗成的最小前缀长度是 24（IPv4）和 48（IPv6）。

```
function net_len_too_long(){
    case net.type {
        NET_IP4: return net.len > 24;
        NET_IP6: return net.len > 48;
        else: print "net_len_too_long: unexpected net.type ", net.type, " ", net; return false;
    }
}
```

### Bogon Prefixes and ASNs

有一些特殊用途的、私有的、被保留的 IP 地址块或者 ASN 是不适宜出现在公网的，所以这里列出了所有“bogon”的 IP 前缀和 ASN，供大家写过滤器的时候使用。

```
define BOGON_ASNS = [
    0,                      # RFC 7607
    23456,                  # RFC 4893 AS_TRANS
    64496..64511,           # RFC 5398 and documentation/example ASNs
    64512..65534,           # RFC 6996 Private ASNs
    65535,                  # RFC 7300 Last 16 bit ASN
    65536..65551,           # RFC 5398 and documentation/example ASNs
    65552..131071,          # RFC IANA reserved ASNs
    4200000000..4294967294, # RFC 6996 Private ASNs
    4294967295              # RFC 7300 Last 32 bit ASN
];
define BOGON_PREFIXES_V4 = [
    0.0.0.0/8+,             # RFC 1122 'this' network
    10.0.0.0/8+,            # RFC 1918 private space
    100.64.0.0/10+,         # RFC 6598 Carrier grade nat space
    127.0.0.0/8+,           # RFC 1122 localhost
    169.254.0.0/16+,        # RFC 3927 link local
    172.16.0.0/12+,         # RFC 1918 private space 
    192.0.2.0/24+,          # RFC 5737 TEST-NET-1
    192.88.99.0/24+,        # RFC 7526 6to4 anycast relay
    192.168.0.0/16+,        # RFC 1918 private space
    198.18.0.0/15+,         # RFC 2544 benchmarking
    198.51.100.0/24+,       # RFC 5737 TEST-NET-2
    203.0.113.0/24+,        # RFC 5737 TEST-NET-3
    224.0.0.0/4+,           # multicast
    240.0.0.0/4+            # reserved
];
define BOGON_PREFIXES_V6 = [
    ::/8+,                  # RFC 4291 IPv4-compatible, loopback, et al 
    0100::/64+,             # RFC 6666 Discard-Only
    2001::/32{33,128},      # RFC 4380 Teredo, no more specific
    2001:2::/48+,           # RFC 5180 BMWG
    2001:10::/28+,          # RFC 4843 ORCHID
    2001:db8::/32+,         # RFC 3849 documentation
    2002::/16{17,128},      # RFC 7526 6to4 anycast relay, no more specific
    3ffe::/16+,             # RFC 3701 old 6bone
    fc00::/7+,              # RFC 4193 unique local unicast
    fe80::/10+,             # RFC 4291 link local unicast
    fec0::/10+,             # RFC 3879 old site local unicast
    ff00::/8+               # RFC 4291 multicast
];

function is_bogon_prefix() {
    case net.type {
        NET_IP4: return net ~ BOGON_PREFIXES_V4;
        NET_IP6: return net ~ BOGON_PREFIXES_V6;
        else: print "is_bogon_prefix: unexpected net.type ", net.type, " ", net; return false;
    }
}

function is_bogon_asn() {
    if bgp_path ~ BOGON_ASNS then return true;
    return false;
}
```

### 与 Vultr 的 BGP session

```
protocol bgp bgp_vultr_6 from tpl_bgp {
    source address xxx; # 请修改
    neighbor 2001:19f0:ffff::1 as 64515; # 请核对
    multihop 2;
    password "2333"; # 请修改

    ipv6 { /* 请自行写好过滤器 */ };
}
protocol static {
    ipv6;
    # 告诉 BIRD，Vultr 路由器的下一跳在哪里
    route 2001:19f0:ffff::1/128 via fe80::xxxx%ens3;
    # 这个 via 自己找，一般看内核的 v6 路由表就能找到
    # ens3 是 Vultr 的网卡，%ens3 的意思是 ens3 网卡上的 link-local 地址
}

# 这个函数可以删除 Vultr 发来的表中各个路由 AS Path 前面的私有 ASN，并替换为 20473
function preprocess_vultr_bgp_path() {
    bgp_path.delete(BOGON_ASNS);
    # ^ 目的是删除 Vultr 传过来的私有 ASN，为了实现简单，
    #   这里删除 AS Path 里面所有私有 ASN
    #  （BIRD 不能只删除特定位置的，不过私有 ASN 本身不应该出现在公网所以直接全删了也没关系）
    if bgp_path.first != 20473 then bgp_path.prepend(20473);
}
```

### 与 HE TunnelBroker 的 BGP session

```
protocol bgp bgp_he {
    graceful restart on;
    local as 64512; # 请修改
    source address 2001:470:aa:aa::2; # 请修改
    neighbor 2001:470:aa:aa::1 as 6939; # 请修改

    ipv6 { /* 请自行写好过滤器 */ };
}
```
