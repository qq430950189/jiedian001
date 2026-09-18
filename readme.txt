you asked

message time: 2026-09-18 01:00:09

网上搜索一下，除了hy2，还有哪些组合适合非443端口的代理


chatgpt response

我会按“非 443 是否天然适配、TCP/UDP、部署复杂度、与 VLESS/REALITY 的互补性”去查现有主流方案，并优先看官方文档和项目仓库。

我查了目前还在活跃维护、且 2026 年仍有实际客户端/服务端支持的方案。(GitHub)

如果你的目的不是“再找一个 Hy2”，而是想做一个真正有区别的 非 443 备用池，我会重点看下面几类。

组合 典型外部端口 传输 非 443 适配 和 VLESS+REALITY 的差异
TUIC v5 + QUIC UDP 2053/8443/自定义 UDP/QUIC ★★★★★ UDP/QUIC
AnyTLS TCP 8443/2053/自定义 TCP+TLS ★★★★★ TCP+标准 TLS、不同协议栈
Trojan + TLS TCP 8443/2053/自定义 TCP ★★★★☆ 不用 REALITY
NaiveProxy + HTTP/2 TCP 8443/2053 TCP/H2 ★★★★☆ 真正的 HTTP/2 代理形态
Shadowsocks 2022 + ShadowTLS TCP 8443/2053 TCP+TLS ★★★★★ SS 数据层 + ShadowTLS TLS 层
Shadowsocks 2022 TCP/UDP 任意 TCP/UDP ★★★★★ 最简单，但外观最普通
VLESS + REALITY TCP 8443 等 TCP ★★★☆☆ 你现在已有

其中最值得你关注的是前五种。


1. TUIC v5 + QUIC：我认为最适合当 Hy2 的“同类但不同实现”

TUIC 本身主要设计在 QUIC 上，支持 TCP relay、UDP relay、多路复用，并且有 0-RTT、连接迁移等 QUIC 特性；sing-box 当前也原生支持 TUIC。(GitHub)

结构：

UDP:8443
     │
     ▼
   QUIC
     │
   TUIC v5
     │
 TCP / UDP Proxy

它和 Hy2 的共同点：

UDP
QUIC

但协议层不同：

Hysteria2
    ≠
TUIC

所以如果你的目标是：

TCP 主节点
+
UDP Hy2
+
UDP TUIC

确实可以形成第三种协议栈。

不过要注意：Hy2 和 TUIC 都是 QUIC/UDP 家族，所以如果某个网络整体限制 UDP/QUIC，它们可能一起受影响。TUIC 官方协议明确以 QUIC 为主要底层设计。(GitHub)

因此它适合“协议备用”，但不是最彻底的“网络栈备用”。


2. AnyTLS：非常适合你的非 443 TCP 备用

这个反而值得你认真考虑。

AnyTLS 的设计目标就是解决传统 TLS-in-TLS 类代理的指纹/嵌套问题，同时提供连接复用和分包/填充策略。参考实现当前就直接用：

0.0.0.0:8443

作为示例服务端监听地址。(GitHub)

也就是说：

TCP:8443
   ↓
TLS
   ↓
AnyTLS
   ↓
Proxy

它天然不需要把自己绑定到 443。

而且现在 sing-box、mihomo、Shadowrocket 等生态都已经有 AnyTLS 支持；当前 sing-box 生态脚本也仍把 AnyTLS 作为独立协议提供。(GitHub)

对于你的需求，我会把它理解为：

443:
VLESS + REALITY + Vision

8443:
AnyTLS

这种组合非常合理，因为：

REALITY
   ↓
Xray/REALITY

AnyTLS
   ↓
普通 TLS / AnyTLS

不是同一个协议栈。


3. Trojan + TLS：老牌，但仍然很适合非 443

Trojan 的核心就是：

TCP
+
TLS
+
认证

服务端 TLS socket 收到请求后，如果密码和 Trojan 请求格式正确就做代理；否则可以把它当成普通 HTTPS 流量转给预设的 Web 服务。官方协议文档仍明确采用这种设计。(GitHub)

所以：

TCP:8443
   ↓
TLS
   ↓
Trojan

完全成立。

它最大的价值不是性能，而是：

它和 REALITY 是两个不同的 TCP/TLS 方案。

你的备用体系就可以变成：

443
VLESS + REALITY

8443
Trojan + TLS

而不是：

443
REALITY

8443
REALITY

后者两台其实还是一个技术路线。


4. NaiveProxy + HTTP/2：很值得留一手

这个比较特殊。

NaiveProxy 的核心思路是直接使用浏览器常见的 HTTP/2/HTTP/3 网络模型来做代理，官方 README 当前仍强调通过 HTTP/2 前端代理以及 padding 来降低代理流量特征；当前实现也支持：

https://...
quic://...

两种连接形式。(GitHub)

所以可以做：

TCP:8443
    ↓
TLS
    ↓
HTTP/2
    ↓
NaiveProxy

这和：

TCP:443
    ↓
REALITY
    ↓
Vision

差异非常明显。

而且一个有趣的点是，NaiveProxy 并不要求外部端口必须是 443；其 URI 规范允许显式指定其它端口，实际项目也有人使用 8443 的 QUIC 入口。(GitHub)

它适合作为“另一种 TCP 备用”

尤其你已经有：

VLESS REALITY

那 NaiveProxy 可以提供：

真正 HTTP/2

这比再开一条：

VLESS REALITY :8443

有意义得多。


5. Shadowsocks 2022 + ShadowTLS：我觉得非常适合你这个“留一手”的思路

这里重点不是单独的 Shadowsocks，而是：

ShadowTLS
    +
Shadowsocks

ShadowTLS 本身并不是完整的代理协议，它通常与 Shadowsocks 配合使用；官方文档明确就是这么设计的。它负责外层真实 TLS 握手/伪装，Shadowsocks 负责实际加密代理。(GitHub)

结构：

TCP:8443
     ↓
 ShadowTLS
     ↓
Shadowsocks
     ↓
  Proxy

这个方案特别有意思，因为它和：

VLESS + REALITY

在原理上明显不同。

而 ShadowTLS 的服务端完全可以监听非 443，例如官方运行示例就是：

--listen 0.0.0.0:4430

同时 TLS handshake server 可以指向其它域名/443。(GitHub)

所以非常适合：

443  → VLESS REALITY
8443 → ShadowTLS + SS


如果让我给你设计一套“备用矩阵”

考虑到你已经明确有 Hy2，我反而不会再堆一大堆类似协议。

我会做：

                    主力
                     │
             ┌───────┴───────┐
             │               │
           TCP             UDP
             │               │
       VLESS REALITY       Hy2
          :443             :443
             │
             │
        TCP 备用
             │
          AnyTLS
          :8443
             │
             │
        TCP 第二备用
             │
    ShadowTLS + SS
          :2053

甚至可以变成：

1. VLESS + REALITY + Vision
   TCP/443

2. AnyTLS
   TCP/8443

3. ShadowTLS + Shadowsocks
   TCP/2053

4. Hysteria2
   UDP/443

5. TUIC v5
   UDP/8443

这样你不是简单地准备“五个节点”，而是准备了几种不同的网络形态：

TCP + REALITY
TCP + TLS
TCP + HTTP/2
TCP + TLS wrapper + SS
UDP + QUIC(Hy2)
UDP + QUIC(TUIC)


对你而言，我会特别推荐这三个非 443

第一选择：AnyTLS

TCP/8443
AnyTLS

原因：部署简单，当前生态已经比较完整，而且与 REALITY 的协议结构差异明显。AnyTLS 官方参考实现甚至直接以 8443 作为示例监听端口。(GitHub)

第二选择：ShadowTLS + Shadowsocks

TCP/2053
ShadowTLS
+
SS

它的外层 TLS 与 REALITY 是不同实现路线，而且可以把 TLS handshake server 与实际监听端口解耦。(GitHub)

第三选择：TUIC v5

UDP/8443
TUIC

如果你想要第二个 UDP/QUIC 方案，它比简单再开一个 Hy2 更有意义；但由于两者都基于 QUIC/UDP，所以不应该把它当成“UDP 被限制后的备用”。(v2.hysteria.network)


反而不太建议你优先堆的

VLESS + TLS + WS :8443
VMess + WS :8443
VLESS + gRPC :8443

它们当然能用，而且现在 sing-box 生态仍支持这些组合；但如果你的目标单纯是“给 REALITY + Hy2 留后手”，这些方案和你已有体系的新增价值没有前面几个大。当前 2026 年的 sing-box 生态仍然把 REALITY、Hy2、TUIC、Trojan、Shadowsocks、AnyTLS、NaiveProxy 等作为主流可选协议。(GitHub)

所以按你的实际需求，我会优先研究 AnyTLS :8443 和 ShadowTLS + SS :2053，而不是再搞一条 VLESS + REALITY :8443。

另外，如果你的服务器是你之前提到的那种 Alpine/OpenRC、小内存 VPS，AnyTLS、ShadowTLS+SS、TUIC 三者的资源占用和 ARM/i386/静态二进制兼容性其实差别不小；这一点可以直接决定哪个最适合你现有机器。


you asked

message time: 2026-09-18 01:02:31

我的要求是非443，我还需要对比这些组合的加密混淆抗封锁，性能效率，打击力度现状


chatgpt response

目前有一个关键结论：非 443 并不等于更抗封锁。端口主要改变暴露面；真正决定可识别性的，是 TLS/QUIC 指纹、首包长度/时序、主动探测响应，以及 IP/域名本身。近期研究尤其显示 GFW 已经能对 QUIC Initial 做大规模解密和 SNI 级过滤。

我重新按你真正关心的四个维度整理：加密、流量混淆/可识别性、性能效率、目前公开可见的针对性打击情况。而且只讨论非 443 入口，不再把“443 主力节点”混进来。

先给结论：

如果你已经有 Hysteria2，我认为非 443 最值得留的 TCP 备用是 AnyTLS 或 ShadowTLS + Shadowsocks 2022；如果允许再准备一个 UDP 备用，才考虑 TUIC。

NaiveProxy 的流量仿真能力很强，但部署和资源成本明显高一些；Trojan + TLS 很成熟，但现在更适合做“传统 TLS 备用”，而不是唯一后手。




不要把“QUIC 很快”直接等同于“抗封锁更强”：2025 年公开研究已经确认 GFW 会大规模解密 QUIC Initial 并按 SNI 过滤，而且这个 QUIC 专用封锁体系拥有独立 blocklist。(GFW Report)




一、先明确：这几个指标其实不是一回事

加密

看的是：

数据本身是否有可靠的现代密码保护

例如：

REALITY → TLS 1.3
Trojan → TLS
AnyTLS → TLS
NaiveProxy → TLS
ShadowTLS + SS2022 → TLS + AEAD2022
TUIC → QUIC/TLS
Hy2 → QUIC/TLS

所以单看“能不能加密”，现在这些都没有明显短板。

混淆

真正需要比较的是：

别人看到你的连接以后
能不能从 ClientHello / 首包 / 包长 / 时序 / ALPN / 协议行为
判断“这不是普通 HTTPS”

这恰恰是这些协议差别最大的地方。

性能

主要看：

TCP vs QUIC
用户态协议栈
TLS 层
multiplex
padding
分片
CPU
RTT / 丢包

这里不存在绝对排名。尤其 QUIC 在差网络下可能非常快，但在低端 VPS 上却比 TCP 更吃 CPU；Hysteria 官方文档自己也明确说明 QUIC 的用户态协议栈相比成熟的内核 TCP 会增加处理开销。(Hysteria 2)


二、我把非 443 方案放一起比较

方案 非443适配 加密 混淆/仿真 性能 当前公开“针对性审查”情况
AnyTLS ★★★★★ TLS ★★★★☆ ★★★★☆ 低/未知，但在快速演进
ShadowTLS + SS2022 ★★★★★ TLS + AEAD ★★★★☆ ★★★★½ 中等/已有长期研究
Trojan + TLS ★★★★★ TLS ★★★★☆ ★★★★½ 中等/老牌
NaiveProxy + H2 ★★★★☆ TLS ★★★★★ ★★★½ 长期被研究，但设计针对性强
VLESS + REALITY ★★★★★ TLS/REALITY ★★★★½ ★★★★½ 高认知，但无公开统一的实时封锁评级
TUIC v5 ★★★★★ QUIC/TLS ★★★☆☆ ★★★★★ QUIC 处于明确审查范围
Hysteria2 ★★★★★ QUIC/TLS ★★★★☆ ★★★★★ QUIC 处于明确审查范围
SS2022 单独使用 ★★★★★ AEAD2022 ★★☆☆☆ ★★★★★ 老牌、可识别性问题更明显

这里的星级是基于协议架构和公开文档得出的技术比较，不是实测“翻墙成功率”评分。尤其“当前打击力度”没有公开、持续、可重复的跨协议统一测试，所以我不会假装有一个精确的排行榜。


三、AnyTLS：现在最值得研究的非 443 TCP 备用之一

AnyTLS 的定位非常直接：

用 TLS 做外层，同时解决传统 TLS-in-TLS 代理明显的握手和流量长度特征。



官方协议设计里有：

TLS
 ↓
认证
 ↓
Session multiplex
 ↓
padding / 分包
 ↓
proxy stream

而且它支持动态 padding scheme。服务器可以在首次通信时更新 padding 方案；官方协议甚至明确提到，这个机制就是考虑到固定的默认流量特征以后可能被审查系统加入黑名单。(GitHub)

这点非常重要。

优点

TCP
TLS
可调整 padding
连接复用

因此你做：

TCP/8443
AnyTLS

非常自然。

官方参考实现本身就直接用：

0.0.0.0:8443

作为示例监听端口。(GitHub)

缺点

它毕竟是较新的协议。

官方自己就非常坦率：

默认参数不保证不会被识别。



而 sing-box 目前也已经出现了 AnyTLS 与其他协议链路组合时的性能问题案例；例如 2026 年 8 月的一项 issue 报告了 AnyTLS→Hysteria2 链路吞吐在 83–808 Mbps 之间大幅波动，不过这个问题是特定串联场景，并不能代表 AnyTLS 单独性能。(GitHub)

另外，AnyTLS 在连接完全新建时的延迟表现可能不如保持 idle session 的情况；sing-box 2025 年就有用户报告首次测速延迟偏高，而保持 min-idle-session 后恢复正常。(GitHub)

我的判断

非常适合做你所说的“留一手”。

尤其是：

443
VLESS + REALITY

8443
AnyTLS

两个 TCP 方案的实现路线已经不一样。


四、ShadowTLS + Shadowsocks 2022：很适合真正意义上的“第二套 TCP 技术栈”

这个组合是：

TCP
 ↓
TLS / ShadowTLS
 ↓
Shadowsocks 2022
 ↓
AEAD

SS2022 本身现在支持：

2022-blake3-aes-128-gcm
2022-blake3-aes-256-gcm
2022-blake3-chacha20-poly1305
2022-blake3-chacha8-poly1305

shadowsocks-rust 目前仍在积极维护，2026 年还有持续更新。(GitHub)

而 ShadowTLS 的 V3 是专门强化 TLS 外层伪装和抗劫持设计的版本；当前 releases 页面已经有 v0.2.16，继续维护 V3。(GitHub)

它的一个优势

你可以把：

TLS 外壳

和：

代理协议

解耦。

因此：

8443
ShadowTLS
+
SS2022

和：

443
VLESS
+
REALITY

是非常不同的两条线路。

性能

这一套通常非常优秀：

Rust
+
成熟 TCP
+
AEAD

对于小型 VPS 尤其友好。

而且 ShadowTLS 官方说明其 Linux/macOS 实现强调性能，新版本也继续维护。(GitHub)

风险

它不是“隐形魔法”。

ShadowTLS / SS 的组合已经存在很多年，审查研究者当然早就知道其基本工作模式。

所以它更接近：

成熟、工程化、可控的第二套 TCP 栈



而不是：

“没人知道的新协议”。




五、Trojan + TLS：稳，但“新鲜度”没那么高

Trojan 本质就是：

真实 TLS
 ↓
Trojan authentication
 ↓
Proxy

它的一个经典设计就是：

如果客户端不像 Trojan，服务端就把它当普通 HTTPS 请求。



官方协议明确写了：

正确 Trojan → proxy
错误结构/主动 probe → fallback HTTPS

并且 Trojan 的第一包会把认证结构和 payload 放在一起，以减轻简单的长度模式识别。(GitHub)

这意味着它在主动探测方面的思想仍然很成熟。

更重要的是

sing-box 当前 Trojan 文档甚至直接指出：

没有证据表明 GFW 会因为 HTTP response 判断 Trojan 服务并进行封锁；但如果你在标准 HTTP/S 端口上运行一个服务，端口本身反而会成为更大的签名。



(GitHub)

这句话其实非常符合你的问题。

也就是说：

Trojan :8443

并不是因为“8443 有魔法”，而是因为：

协议行为
+
TLS 行为
+
端口
+
服务器实际网站行为

要一起看。


六、NaiveProxy：从“仿真普通互联网流量”的角度，它很特殊

它与前面几个不太一样。

NaiveProxy 直接使用 Chromium 网络栈。

官方项目明确把它的目标写成：

Website fingerprinting → HTTP/2 multiplex
TLS fingerprinting → Chromium network stack
Active probing → application fronting
Length analysis → padding / fragmentation

(GitHub)

这一点非常关键。

它不是：

“我模拟一个 Chrome ClientHello”

而是：

“我尽量使用真实 Chromium 网络栈的行为”

这比单纯 uTLS fingerprint=chrome 的思路更深入。

而且 sing-box 现在甚至明确警告，传统 uTLS 模拟存在反复被发现的 fingerprinting 漏洞，对于 TLS 指纹抗识别，更推荐 NaiveProxy。(GitHub)

但为什么我不把它放第一？

因为成本。

它需要：

Chromium 网络栈
HTTP/2
前端代理 / Caddy / HAProxy
证书
application routing

比：

AnyTLS

和：

ShadowTLS + SS

明显复杂。

而且你强调的是：

非 443



NaiveProxy 最大的优势之一是“像正常 HTTPS 网站”。

如果你直接暴露：

example.com:8443

它仍然可以工作，但：

https://example.com:8443

显然比：

https://example.com

少了一层“普通性”。

因此 NaiveProxy 的设计优势在非 443 环境下会有所折损，尽管协议本身仍然成立。


七、VLESS + REALITY：你已经有了，它应该留在 443

REALITY 的优势仍然是：

TLS
+
uTLS client fingerprint
+
服务器身份验证
+
shortId
+
主动探测处理

而且官方配置本身就是以：

VLESS
+
XTLS Vision
+
REALITY

作为典型方案。(GitHub)

但这里有个你现在特别需要注意的问题：

REALITY ≠ “Chrome fingerprint 就不会被识别”

Xray 当前文档仍要求 REALITY 使用 fingerprint，而且 unsafe 都不允许，因为 REALITY 依赖 uTLS 修改低层 TLS 参数。(GitHub)

而 sing-box 对传统 uTLS 现在已经明确提出 fingerprinting 风险，建议 TLS 指纹抗识别使用 NaiveProxy。(GitHub)

这并不是说 REALITY 已经“不行了”，而是：

不要把“fp=chrome”理解成等同于真实 Chrome。



所以你的：

VLESS + REALITY

作为主力没问题，但我不建议你的所有备用都继续复制 REALITY 思路。


八、TUIC：性能很好，但你已经有 Hy2 后，意义下降

TUIC v5 是：

TLS
+
QUIC
+
multiplex
+
TCP relay
+
UDP relay

官方规范明确以 QUIC 作为主要底层，而且本身设计为 multiplex TLS-encrypted stream。(GitHub)

所以：

TUIC :8443/UDP

性能非常优秀。

但是你现在已经明确有：

Hysteria2

那么：

Hy2
+
TUIC

都属于：

UDP
+
QUIC

这一大类。

而这个大类当前恰恰有一个非常现实的问题：

QUIC 已经进入 GFW 的明确观察/过滤范围

2025 年 USENIX Security 的测量结果非常明确：

GFW 自 2024 年 4 月 7 日起开始对特定域名进行 QUIC 封锁；
可以大规模解密 QUIC Initial；
使用 QUIC-SNI 独立 blocklist；
2024-10-08 至 2025-01-15 的长期测量中，观察到 58,207 个不同 FQDN 曾被 QUIC-SNI 机制封锁；
研究人员因此把规避策略集成到了多个 QUIC-based circumvention tools。(GFW Report)

所以你应该把：

Hy2
TUIC

看成：

两个不同的 QUIC 实现



而不是：

两个完全不同的抗封锁体系。




九、Hysteria2 为什么仍然值得保留

Hy2 的情况其实比“普通 QUIC 代理”复杂。

它默认会：

伪装 HTTP/3

而且如果认证失败，可以表现成正常 HTTP/3 server。官方协议明确把这一点作为抗主动探测设计的一部分。(Hysteria 2)

更有意思的是它现在已经有：

Salamander
Gecko

两套额外混淆。

官方文档明确说：

Salamander：把每个 QUIC 包处理成随机字节；
Gecko：进一步对 QUIC 握手包进行随机分片 + 随机 padding。(Hysteria 2)

但是代价也非常明显：

一旦使用这些 obfuscation，就不再表现为标准 HTTP/3。



(Hysteria 2)

所以你可以把 Hy2 理解成两个状态：

Hy2 默认
= HTTP/3 camouflage

Hy2 + Salamander/Gecko
= 不再追求“像正常 HTTP/3”，而是追求“隐藏 QUIC 特征”

这两种抗封锁思路完全不一样。


十、所以真正适合你的不是“哪个协议最抗封”

而是下面这种组合

你已经确定：

Hysteria2

那我会建议你把线路分成：

                 ┌── TCP ── VLESS REALITY ── :443
                 │
主/备用体系 ─────┼── TCP ── AnyTLS ────────── :8443
                 │
                 ├── TCP ── ShadowTLS+SS2022 ─ :2053
                 │
                 └── UDP ── Hysteria2 ──────── :443

如果还想多准备一条：

UDP ── TUIC ── :8443

但我把 TUIC 放最后，不是因为 TUIC 性能差，恰恰相反，是因为：

Hy2
TUIC

都依赖 QUIC/UDP，因此你的故障相关性仍然比较高。


十一、如果只让我从“非 443”里面选两个

我会选：

AnyTLS :8443

理由：

TCP
TLS
padding
session multiplex
协议较新
非 443 原生使用

而且参考实现本身就以 8443 为示例。(GitHub)

这是一个很适合和：

VLESS + REALITY :443

并存的 TCP 备用。


ShadowTLS + SS2022 :2053

理由：

TLS 外层
+
不同的代理数据层
+
SS2022 AEAD
+
Rust 高性能实现

它和 REALITY 已经不是同一种设计。


十二、如果把“抗封锁”拆开看，我反而会这样理解

对“被动 DPI / 流量分类”

NaiveProxy
    ≈ 非常强

REALITY
    ≈ 强

AnyTLS
    ≈ 强，但持续演化中

ShadowTLS + SS2022
    ≈ 强

Trojan TLS
    ≈ 强

Hy2 / TUIC
    ≈ QUIC 层强，但 QUIC 本身已经受到重点观察

NaiveProxy 的优势来自真正 Chromium 网络栈、HTTP/2 multiplex、padding 和 frontend routing，而不是简单伪造一个 ClientHello。(GitHub)

对“主动探测”

NaiveProxy
    ≈ 很强

Trojan
    ≈ 很强

Hy2
    ≈ 很强

REALITY
    ≈ 很强

AnyTLS
    ≈ 强

ShadowTLS
    ≈ 强

Trojan 和 Hy2 都明确设计了“认证失败后表现得像正常服务”的逻辑；Hy2 官方尤其明确要求未认证请求表现成普通 HTTP/3 server。(GitHub)

对“当前针对性研究/打击”

这里反而最值得注意：

QUIC / Hy2 / TUIC
        ↑
      明确有公开的 GFW 专项研究

2025 年的实测论文是目前公开证据最强的一块。(gfw.report)

而：

AnyTLS

官方自己承认默认流量特征有可能进入黑名单，只是设计了动态 padding 更新机制。(GitHub)

至于：

REALITY / Trojan / ShadowTLS / SS2022 / NaiveProxy

我没有找到一套截至 2026 年 9 月、对这些协议进行统一、同条件、长期实测并给出“当前封锁率”的公开数据，因此不建议把网上各种“XX 已死 / XX 无敌”的帖子当成事实。


最后，按照你的“留一手”思路，我会这样排

非 443 TCP：

① AnyTLS
② ShadowTLS + SS2022
③ Trojan + TLS
④ NaiveProxy H2

非 443 UDP：

① Hysteria2
② TUIC

而你的整体节点池最有价值的地方并不是“协议数量多”，而是：

VLESS REALITY
       │
       │ TCP
       ▼
    主线路

AnyTLS
       │
       │ TCP
       ▼
   TCP备用

ShadowTLS + SS2022
       │
       │ TCP
       ▼
   第二TCP备用

Hysteria2
       │
       │ UDP/QUIC
       ▼
   UDP备用

这样真正形成了 “TCP/REALITY → TCP/TLS → TCP/SS → UDP/QUIC” 的故障隔离，而不是简单堆四个基于 QUIC/TLS 的协议。

还有一个非常重要的现实因素：非 443 端口本身不是抗封锁加分项。2025 年甚至出现过 GFW 在约 74 分钟内针对 TCP/443 注入 RST+ACK、而 22/80/8443 等端口未同步受到该异常行为影响的事件；这说明端口级策略确实可以存在，但也说明不能把某个“神奇备用端口”当成永久安全区。(Tom's Hardware)

对你这个需求，我会优先实际折腾 **AnyTLS :8443 和 ShadowTLS + SS2022 :2053**，而不是再开一个 VLESS REALITY :8443。


you asked

message time: 2026-09-18 01:11:51

看看这个呢，你研究得不够全面详细啊，虽然下面这个也是。我来研究非443端口的加密代理方案组合，重点对比加密混淆抗封锁能力、性能效率和当前打击力度。先搜索相关资料。继续深入获取非443端口具体方案和最新打击现状细节。非443端口加密代理方案组合研究与对比（基于2025-2026年公开资料与实测报告）

非443端口方案的核心动机通常是：规避443上的高强度TLS/SNI/JA3指纹检测、主动探测或临时全量RST事件（如2025年8月短暂443 unconditional RST注入，其他端口如80/8443未受影响）；或应对运营商对特定端口的QoS；或利用端口跳跃对抗“IP+端口”封锁。但非443也有劣势：真实HTTPS流量高度集中在443，非标准端口的加密流量更容易因统计特征（高熵、长连接、流量模式）被标记，且部分企业/校园网直接封锁非80/443出站。

主流可用组合（自建为主，机场类似）聚焦以下，按抗封锁优先级大致排序。所有方案都需强加密（AEAD/TLS1.3）、定期换IP/多节点、分流（国内直连），并配合客户端如Xray/sing-box/Clash Meta/Hysteria原生。Reality类强烈推荐443（借用真实大站如microsoft.com握手，非443会显著削弱伪装，因为目标站不监听其他端口）。

1. 主要方案组合

Hysteria2 + Salamander OBFS + Port Hopping（高UDP端口范围，如20000-50000）：QUIC底层，Brutal/BBR拥塞控制。服务端监听范围或单端口+iptables/nftables重定向；客户端多端口随机跳跃（hopInterval 5-30s+）。OBFS将QUIC变成高熵随机UDP。可加masquerade（伪装HTTP/3站点）。
Shadowsocks-2022/AEAD（ChaCha20/AES-GCM） + 插件（v2ray-plugin WS/TLS、simple-obfs、Outline prefix等）：任意TCP/UDP高端口或随机。插件可加TLS/WS伪装。
VLESS/VMess + WebSocket/gRPC/XHTTP + TLS（自有域名+证书）：端口如8443、2083、2087、2096、2053等（常见CDN alt）。可加uTLS模拟浏览器指纹、padding。XHTTP进一步HTTP伪装。
Trojan（或Trojan-Go） + TLS + 真实网站fallback/回落：自有域名证书，端口8443等。非认证流量回落到Nginx静态站，抗主动探测。
TUIC v5（类似Hysteria，QUIC）：高UDP端口 + 可选混淆/跳跃。
NaiveProxy（Chromium网络栈 + HTTP/2或QUIC）：自有域名+证书，端口非443可行但次优（浏览器特征更匹配443）。抗探测靠application fronting + padding。
其他：AmneziaWG（WireGuard混淆）或OpenVPN+obfs/scramble on 非标准UDP/TCP；mKCP等，抗封锁较弱。

CDN中转（CF Workers/Pages + VLESS WS）常被迫用443/80，非纯非443。

2. 对比维度（2025-2026数据汇总）

评分基于社区实测、协议分析与GFW行为报告（存活率、检测率、速度开销等，主观综合，满分相对）。环境差异大（电信最严UDP限速，联通/移动相对宽松；敏感期全面加强）。

方案组合 加密/混淆抗封锁 性能效率 打击力度现状（GFW/运营商） 优缺点与适用 推荐指数（非443场景）
Hysteria2 + Salamander + Port Hopping (UDP高端口范围) 高：OBFS随机化抗DPI/QUIC指纹；跳跃对抗单端口封/限速；masquerade抗探测。抗主动探测中等。 极高：0-RTT、无队头阻塞、Brutal吃满带宽；弱网/高丢包/游戏/4K优势明显（比TCP快30-50%+）；CPU中等偏高。 中-高：2024.4起QUIC SNI阻断 + 运营商UDP QoS（电信重）；跳跃+OBFS有效绕过多数端口级限制，但全UDP封锁时失效。IP易因大流量被盯。存活中等，需备TCP。 优：速度王、弱网强。缺：UDP依赖、生态稍弱。游戏/下载/移动首选。 ★★★★★（非443最契合）
SS-2022/AEAD + 插件 (任意高端口) 中：AEAD强加密+时间戳抗重放；插件（WS/TLS）提升伪装。裸SS高熵易被ML检测。抗探测中（静默关闭）。 高：开销极低，延迟小，CPU低；干净网络接近线速。弱网一般。 中-高：裸SS敏感期存活~45%；插件后提升但仍不如TLS伪装。高熵+异常端口易触发统计封锁。 优：轻量、易部署、兼容好。缺：无强伪装时脆弱。日常/备用/路由器。 ★★★☆
VLESS/VMess + WS/gRPC/XHTTP + TLS (8443等) 高（XHTTP更高）：TLS1.3 + uTLS指纹 + HTTP伪装；gRPC像正常RPC。抗探测靠fallback较好。非Reality时证书/SNI可被针对。 高：VLESS开销低于VMess（~15-30%提升）；XTLS-Vision接近原生；多路复用好。 中：TLS alt端口不如443“正常”；WS/gRPC被识别风险中等。XHTTP+Reality变体强但Reality非443弱。IP封锁为主。 优：灵活、生态完善（Xray）。缺：需域名证书；非443伪装折扣。全能/CDN辅助。 ★★★★
Trojan + TLS + Fallback (8443等) 极高（接近443版）：完美HTTPS模拟 + 真实证书 + 回落网站抗主动探测。流量与正常浏览难区分。 高：TLS开销小，稳定；接近VLESS。 中-低（相对）：TLS伪装强，敏感期存活高（平时94%、敏感87%、高压76%样本）；非443略降“正常度”，但仍优于高熵。 优：稳定、抗探测强。缺：需域名/证书维护。敏感期/长期稳。 ★★★★
TUIC / NaiveProxy (非443) TUIC：类似Hy2中-高。Naive：极高（真Chrome栈，指纹/ padding完美，抗探测强）。 TUIC极高（QUIC）；Naive中-高（HTTP/2稳，CPU较低，延迟稳）。 TUIC同Hy2；Naive高抗（像真浏览），但需域名，非443次优。 Naive维护复杂/客户端少；TUIC弱网好。 Naive ★★★★；TUIC ★★★★
WireGuard/Amnezia 或 OpenVPN+obfs (非标UDP/TCP) 低-中：原生特征明显；混淆后中等。 极高（WG）；OpenVPN较低。 高：易被识别/RST，即使混淆。 仅低审查或内网。 ★★

抗封锁细节：

加密混淆：TLS伪装（Trojan/VLESS+TLS/Naive）> Reality（但Reality非443大打折扣）> QUIC+OBFS（Hy2）> 高熵AEAD（SS）。主动探测靠fallback/静默/真实响应。流量分析靠padding、多路、指纹模拟（uTLS/Chrome）。非443需更强混淆，因端口本身可疑。
性能：干净链路 WG/SS/VLESS > Trojan/Hy2（开销）；弱网/丢包 Hy2/TUIC完胜（无队头、自定义CC）。CPU：SS/VLESS低，Hy2/Naive中，多用户时注意。
打击现状：GFW重点DPI（JA3/JA4、SNI、熵、时序、长度分布）+ 主动探测 + IP信誉 + 行为（大流量突发）。2024+ QUIC SNI解密阻断（可武器化）；UDP QoS普遍（尤其电信）；临时443事件显示端口策略存在但非绝对。非443可躲部分443规则，但异常端口+加密易触发更广封锁。敏感期/重大事件全面升级，多协议+IP池+短连接更关键。存活依赖运营（换IP频率）>纯协议。

3. 推荐实践与注意事项

非443优先组合：Hy2（跳跃+OBFS）作速度主力 + Trojan/VLESS+TLS（alt端口）作稳定备；或SS插件作轻量。多开端口/协议自动切换（Clash/sing-box URL-test）。
部署建议：高端口避开常见（22/80/443/3306等）；范围跳跃需防火墙放行（ufw/iptables）；域名+真实站点回落；定期测速/换dest；监控流量模式避免“代理特征”。
风险：无绝对“抗封锁”。IP被墙后换机；遵守当地法律；自建注意安全（更新核心、强密码、防火墙）。机场节点质量与响应速度往往比协议更关键。
测试建议：多ISP（电信/联通/移动）、不同时段实测；工具如iperf、真实应用（4K/游戏/会议）。

资料主要来自协议官方文档、社区实测博客（2025-2026更新）、USENIX等论文及GFW报告。环境快速变化，建议结合实时社区（如相关GitHub Discussions）验证。如果需要具体配置示例或某一组合深化，可提供更多细节。


chatgpt response

你这份版本确实比我上一轮覆盖得更广，尤其把“端口因素”和“协议因素”分开了。不过里面有几处结论需要纠正，我再把证据链补齐。

对，你贴的这份研究明显比我上一轮全面。我重新核了一遍官方文档、当前项目仓库和 2025–2026 年公开审查研究后，发现它的框架基本对，但有几处结论说得过头了，而且还漏掉了 Restls、Mieru、XHTTP 这几类值得纳入主比较表的方案。

最重要的是：“非 443”是一个独立变量，不能直接等价成“更抗封锁”。

先纠正几个关键点

1. 2025 年 8 月的 443 事件是真的，而且恰好证明“非 443 备用”有价值

2025 年 8 月 20 日，北京时间 00:34–01:48，大约 74 分钟，GFW 对 TCP/443 出现了无条件伪造 RST+ACK；测试中 22、80、8443 等端口没有出现同样的注入。这个事件确实说明：如果所有线路都压在 TCP/443，遇到端口级异常会没有退路。但报告本身也只把它记录为一次短暂异常，并没有证明 8443 永久比 443 “安全”。 (GFW Report)

所以你的“留一手”思路是对的，但应该理解成：

443 主路
+
非443 TCP 备路
+
UDP/QUIC 备路

而不是：

非443 = 更安全


2. Hysteria2 的“端口跳跃”与“混淆”是两种完全不同的机制

你贴的：

Salamander OBFS + Port Hopping



技术上可以同时考虑，但两者解决的问题完全不同。

Hysteria2 官方把 Port Hopping 定位成：

某个 UDP 端口被单独限速/封锁时，通过改变目标端口来规避“单端口”级限制。



官方还明确写了：如果整个 UDP 都被封，port hopping 就没有帮助。当前 Hysteria2 支持单端口、多端口和端口范围，并允许固定或随机跳换时间。 (Hysteria 2)

而：

Salamander
Gecko

解决的是QUIC/HTTP3 本身的流量形状。Salamander 会把 QUIC 包变成随机字节；Gecko 在此基础上进一步拆分并随机 padding QUIC 握手包。官方同时明确提醒：启用这种 obfuscation 后，它就不再是正常 HTTP/3。 (Hysteria 2)

所以：

Port Hopping
→ 对抗“端口级”限制

Salamander / Gecko
→ 对抗“QUIC/HTTP3 流量特征”

不能把两者都归到“混淆”。


3. QUIC/Hy2/TUIC 当前确实属于“重点被研究”的协议族

这一点你贴的报告是正确方向，而且我认为应该在整个比较里给予更高权重。

2025 年 USENIX Security 的公开测量发现，GFW 从 2024 年 4 月开始针对 QUIC 做基于 SNI 的封锁；其设备可以大规模解密 QUIC Initial，按照 QUIC-specific blocklist 做判断。被命中的连接随后会出现约 180 秒的 UDP 丢包/阻断状态。研究还发现，这个 blocklist 与传统 TLS/HTTP/DNS blocklist 不同。 (GFW Report)

更重要的是，研究证明：

QUIC Initial
   ↓
读取其中的 TLS ClientHello / SNI
   ↓
命中规则
   ↓
后续 UDP 包丢弃

所以现在不能再用早期的：

“QUIC 全加密，所以 DPI 很难看”



来理解 Hy2/TUIC。QUIC 已经有专门的审查路径。 (GFW Report)

这也是为什么我不会把：

Hy2
TUIC

算成两个完全独立的灾备体系。

它们的应用协议不同，但都落在：

UDP
+
QUIC
+
TLS Initial

这个大类里。


反而是你这份研究里，REALITY 那句话需要改

你写：

Reality 类强烈推荐 443；非443会显著削弱伪装，因为目标站不监听其他端口。



这个表述过度简化了 REALITY。

REALITY 客户端连的是：

你的 IP:端口

而 REALITY 服务端内部还有：

target
serverNames

两层概念。

它的核心逻辑不是：

你连 8443
↓
必须让伪装网站也监听 8443

REALITY 实现本身就是把目标连接到 Dest，同时根据 serverName 等字段处理握手。官方实现和配置示例都把 serverNames、shortIds、Dest 分开处理。 (GitHub)

所以更准确的是：

REALITY 放在 443 时，端口和外部互联网中最常见的 HTTPS 行为更加一致；放在 8443 等非标准端口后，端口本身增加了一点异常度，但并不意味着 REALITY 的 TLS/握手伪装机制“失效”。



这两句话差别很大。

甚至 Xray 自己的 RealiTLScanner 就支持显式指定扫描端口，而默认只是 443，说明 REALITY 本身并没有协议上的“必须 443”限制。 (GitHub)


真正值得重新加入你的表的，是 XHTTP

这一项你贴的研究里虽然提到了：

VLESS/VMess + XHTTP

但其实低估了它。

Xray 当前文档已经明确：

WebSocket 存在比较明显的流量特征，推荐切换到 XHTTP；gRPC 也推荐 XHTTP。 (Project X)



XHTTP 的当前设计包括：

HTTP/1.1 / HTTP/2 / HTTP/3
header padding
multiplex
上下行分离
packet-up / stream-up

并且当前实现强调通过 HTTP 中间盒/CDN 的兼容性。Xray 官方的长篇设计文档还特别描述了 HTTP header padding、连接复用以及 H2/H3 等行为。 (GitHub)

所以如果你要求的是：

非 443 + TCP + 不想再复制一份 REALITY



那么：

VLESS + XHTTP + TLS

其实是非常应该进入候选集的。

它与：

VLESS + REALITY + Vision

虽然都可能使用 VLESS，但传输层行为已经明显不同。


还有一个非常容易漏掉的：Restls

这个我上一轮也漏了。

Restls 的设计目标非常明确，就是解决：

TLS-in-TLS
固定长度
固定时序
TLS ClientHello fingerprint

它在 TLS 外层基础上增加了自己的认证和 application-data 行为控制，并且有 Restls-Script，可以主动控制前几组数据包的长度、响应次数和 padding，从而改变“代理流量”的行为模式。 (GitHub)

这和：

Trojan
AnyTLS
NaiveProxy

的思路都不完全一样。

尤其值得注意的是，Restls 作者自己明确承认：

为了更强的伪装能力，可以牺牲部分性能。



所以它非常符合你的“备用线路”思维：

主：
VLESS + REALITY

备用：
Restls + SS

但它的问题是：

生态规模
客户端覆盖
工程成熟度

都明显不如 Xray/Naive/SS/Hy2 那些大生态。


再一个你漏掉的：Mieru

这个反而值得你实际测试。

现在的 Mieru 并不是简单的“又一个 Shadowsocks”。

当前协议设计包括：

TCP / UDP
AEAD
随机 padding
分段
时间相关密钥
低熵编码模式
可调整 traffic pattern

官方协议文档甚至把：

padding 0
padding 1
padding 2
low entropy mode
fragment

都做成协议级机制。 (GitHub)

而且它到 2026-09-13 仍在发布新版本，当前公开版本是 v3.37.0。 (GitHub)

所以从“有没有人在持续维护”这个角度，它现在已经不是我以前会把它归为冷门实验项目的东西了。

但问题同样明显：

我找不到足够高质量的公开测量，可以证明 2026 年它在中国 GFW 下有怎样的协议级存活率。



所以：

技术设计值得研究

不等于：

当前中国实测抗封一定强

这点必须分开。


下面才是我现在认为比较完整的表

我把“抗封锁”拆成三个维度：

P = 被动流量识别
A = 主动探测
T = 当前公开针对性研究/打击证据

不是简单的“五星抗封”。

组合 加密 P：被动识别 A：主动探测 性能 T：公开审查证据 非443价值
Hysteria2 + Salamander TLS/QUIC + obfs 高 中 极高 高 很高
Hysteria2 + Gecko TLS/QUIC + obfs 更强但实验性 中 极高 高 很高
TUIC v5 TLS/QUIC 中 中 极高 高 高
AnyTLS TLS 高 高/取决于 fallback 高 目前公开证据少 很高
ShadowTLS + SS2022 TLS + AEAD 高 高 高 中 很高
Trojan + TLS + fallback TLS 高 很强 高 中 很高
NaiveProxy + H2 TLS/H2 很强 很强（使用 frontend 时） 中～高 中/地区性证据 高
VLESS + XHTTP + TLS TLS/H2/H3 高～很高 取决于前端 高 公开统一数据不足 很高
VLESS + REALITY REALITY/TLS 高 高 很高 认知度高 中～高
Restls + SS TLS + AEAD 很高 很强 中 公开数据少 很高
Mieru AEAD + traffic pattern 高 中～高 高 公开中国数据不足 很高
AmneziaWG WireGuard + obfs 中～高 中 极高 俄罗斯证据强，中国证据不足 高
裸 SS2022 AEAD 中/高熵特征明显 中 极高 中 高
WG/OpenVPN + 普通 obfs 强加密 中～低 中 高 较高 中

这里最重要的一列其实是最后的：

非 443 价值



因为它不是“协议强不强”，而是：

放在非 443 后，这个协议本身的优势还剩多少。




分方案重新解释一次

Hysteria2

强项

QUIC
0-RTT
多路复用
UDP
高吞吐
弱网

Hysteria 官方的设计确实非常偏性能，而且支持自定义拥塞控制/速率模型。 (Hysteria 2)

最大问题

它现在已经不能被归类为：

“GFW 看不懂的 QUIC”。



2025 年公开研究已经证明 GFW 对 QUIC Initial 做规模化解析和 SNI 筛选。 (GFW Report)

所以 Hy2 的价值更准确是：

高性能 UDP
+
QUIC camouflage / obfs
+
port hopping

而不是：

完全不受 QUIC 审查影响


TUIC

TUIC 最大价值仍然是：

QUIC
+
0-RTT
+
multiplex
+
TCP/UDP relay

官方协议本身就是围绕 QUIC 设计的。 (GitHub)

但因为：

Hy2 = QUIC
TUIC = QUIC

所以它非常适合：

“Hy2 协议坏掉时再换 TUIC”

却不适合被理解成：

“UDP 被封以后还剩 TUIC”

因为网络层故障可能两者一起中招。


AnyTLS

这个我现在会把它放得比我上一轮更高。

AnyTLS 官方自己的 FAQ 其实相当诚实：

默认参数不能保证不会被封，所以协议专门设计了可更新流量特征的机制。



而协议层核心恰好就是：

TLS
+
session multiplex
+
padding
+
length pattern control

(GitHub)

当前 sing-box 也已经把 AnyTLS 做成正式协议类型，提供完整 padding scheme 配置。 (Sing Box)

它的问题不是性能，而是：

协议太新，长期 GFW 公开测量资料还不够。



所以不能说“当前抗封最好”，只能说：

它是现在非常值得拿来做非 443 TCP 灾备的候选。




ShadowTLS + SS2022

这里我特别赞成你的思路。

因为它实际上是：

真实 TLS 外层
+
独立代理层

Shadowsocks 2022 本身采用现代 AEAD，而且有 replay protection；代价是它本身没有 forward secrecy。 (Shadowsocks)

ShadowTLS V3 又专门针对 active probing 做过协议演进。它的历史设计文档直接写明，早期版本假设审查者不主动探测，V2 后来加入了 challenge-response 和 application-data encapsulation。 (GitHub)

当前 ShadowTLS 项目仍然维护，且 Linux 版本强调高性能。 (GitHub)

所以：

VLESS + REALITY
        +
ShadowTLS + SS2022

是非常漂亮的一组“异构 TCP”。


Trojan + TLS

你贴的“极高”我会改成：

主动探测防御很成熟，被动识别并不是无敌。



Trojan 官方协议设计非常明确：

TLS
 ↓
正确 password + protocol
   → proxy

其他请求
   → fallback website

所以主动探测打到服务器时，可以得到正常 HTTPS/HTTP 服务的行为。 (GitHub)

但“TLS 看起来像 HTTPS”并不等于：

长期代理行为
=
普通浏览器

这是很多早期 Trojan 宣传里容易被忽略的地方。


NaiveProxy

这个方案的核心优势依旧很强：

Chromium network stack
HTTP/2 multiplex
TLS fingerprint = actual Chrome stack
application fronting
length padding

这些都是项目 README 明确写出来的。 (GitHub)

但你这份研究里说：

Naive：极高



我会加一个很大的条件：

frontend 存在时，优势才完整。



因为 application fronting 本身就是 Naive 的重要一环。直接暴露一个 Naive server，与“藏在普通前端后面”的模型不是同一个威胁面。 (GitHub)

而且 2025–2026 年的公开 issue 里已经可以看到针对其 padding / traffic behavior 的识别讨论，例如 2025 年有报告称其 padding 在伊朗审查环境中被识别，2026 年也出现某些网络环境下 Naive 可以连接、但部分外部站点访问异常的案例。这些是社区 issue，不是 GFW 官方测量，因此只能当作“已有针对性研究迹象”，不能直接转化成中国当前封锁结论。 (GitHub)


AmneziaWG：必须和中国 GFW 数据分开

这个其实很有意思。

Amnezia 官方在 2026 年已经发展到了 3.1，并声称 3.1 是针对 2026 年 6–7 月俄罗斯大规模封锁后，对包大小、序列、时间间隔和元数据进行进一步随机化。 (Amnezia 文档)

这说明一个重要趋势：

现代 DPI 正在从“识别某一个握手特征”向统计流量分析发展。



但是：

俄罗斯 2026
≠
中国 2026

所以我不会把“AmneziaWG 3.1 被俄罗斯逼出来了”直接写成“目前 GFW 对 WG 打击很严重”的证据。


这里我最想补充的是 Mieru + Restls

因为它们代表了两种和“套 TLS”完全不同的思路。

Mieru

偏：

AEAD
+
分段
+
随机 padding
+
低熵模式
+
traffic pattern

Restls

偏：

真实 TLS 行为
+
ClientHello 模拟
+
握手后行为模拟
+
script 控制

一个是在改变数据统计特征。

一个是在模拟 TLS 行为过程。

这其实比：

再开一个 VLESS REALITY

有价值得多。


还有个细节：你提到的 2053 / 2083 / 2087 / 2096 / 8443

这些确实不是随便乱写的“代理端口”。

Cloudflare 目前官方文档明确列出：

HTTPS:
443
2053
2083
2087
2096
8443

作为其默认代理支持端口；不过 Cloudflare 又明确说明，中国 Network 的 HTTP/HTTPS 只兼容 80/443。也就是说：

这些端口在全球 Web/CDN 生态中确实有真实存在性，但不能因为 Cloudflare 支持它们，就推导出“中国网络一定把它们当正常 HTTPS”。



(Cloudflare Docs)

这也是为什么我更赞同：

8443

作为非 443 TCP 备用的“工程默认选择”，而不是随便挑一个 30000+ 端口。


最后，我会把你这份研究改成这样的结论

第一层：按“网络栈差异”分类

TCP / TLS
├── VLESS + REALITY
├── AnyTLS
├── Trojan
├── ShadowTLS + SS2022
├── NaiveProxy
├── Restls
├── Mieru
└── VLESS + XHTTP

UDP / QUIC
├── Hysteria2
└── TUIC

UDP / WireGuard-derived
└── AmneziaWG

第二层：按“当前公开针对性审查证据”分类

证据最强的是 QUIC：

Hy2
TUIC

因为已经有专门的 2025 年 GFW 测量论文。 (GFW Report)

证据中等的是传统 TLS/代理协议：

Trojan
ShadowTLS
SS
VLESS/REALITY
Naive

因为存在长期研究和各种识别工作，但我没有找到截至 2026-09 对它们统一进行“中国当前封锁率”的公开实验数据。

证据最弱但技术上值得跟踪的是新方案：

AnyTLS
Restls
Mieru

这里“证据弱”不是“协议弱”，而是还没有足够长时间的大规模中国公开测量。


因此，按照你真正的目的——“443 已经有一条，我还想再留几条真正不同的后手”——我现在会把候选池收敛成：

TCP 主力
VLESS + REALITY

TCP 非443
AnyTLS
ShadowTLS + SS2022
VLESS + XHTTP
Trojan
NaiveProxy
Restls
Mieru

UDP
Hysteria2
TUIC

UDP/WG
AmneziaWG

其中真正有意义的不是同时开十条，而是至少做到：

REALITY
    ↓
TLS/HTTP 类备用
    ↓
另一个 TLS 外壳 + SS 类备用
    ↓
QUIC/UDP

这样才是真正的协议与网络栈故障隔离。

另外，你这份材料里出现的“SS-2022 裸协议存活约 45%、Trojan 敏感期 87%/76%/94%、Hy2 比 TCP 快 30–50%+”这类数字，我暂时不会采纳进最终研究结论。我这次搜索到的是协议文档和高质量测量，但没有找到能够证明这些具体百分比、且同时控制运营商/地区/IP/时段变量的公开原始数据；把社区样本直接写成通用指标，会让整份研究的可信度反而下降。


you asked

message time: 2026-09-18 02:31:27

这份研究整体框架扎实，纠正和补充方向正确，尤其是把“非443”当成独立变量、区分 Port Hopping 与 OBFS、强调 QUIC 已有专门审查路径、以及把 XHTTP / Restls / Mieru / AnyTLS 纳入主表，都比早期简化版更贴近 2025–2026 年公开证据。 我基于官方文档、gfw.report、USENIX Security 2025 论文、项目仓库和社区实测做了交叉核验，确认核心事实成立，同时标出仍需谨慎的地方，并给出更完整的对比与推荐收敛。

关键事实核验（有公开来源支撑）

2025年8月20日 TCP/443 无条件 RST+ACK 事件是真实的
北京时间约 00:34–01:48（约74分钟），GFW 对 TCP/443 注入伪造 RST+ACK，其他端口（22、80、8443 等）未受同样影响。触发不对称（出站 SYN 和 SYN+ACK 都可触发，入站主要靠 SYN+ACK），设备指纹与已知 GFW 组件不完全匹配，可能是新设备或已知设备异常/误配置状态。报告明确记录为短暂异常，不能推导出“8443 永久比 443 安全”。这正好证明“443 主路 + 非443 TCP 备路 + UDP/QUIC 备路”的价值，而不是“非443 = 更安全”。


QUIC SNI 审查（2024年4月起）有高质量论文支撑
USENIX Security 2025 论文（Zohaib 等）证实 GFW 可大规模解密 QUIC Initial 包、提取 TLS ClientHello/SNI，使用独立 blocklist，命中后约180秒 UDP 丢包/阻断。blocklist 与传统 TLS/HTTP/DNS 不同，且解密开销在中等负载下会降低有效性，可被武器化阻断任意 UDP。因此 Hy2 / TUIC 同属 UDP + QUIC + TLS Initial 大类，不能视为完全独立灾备；“QUIC 全加密所以 DPI 难”已过时。


Hysteria2 Port Hopping ≠ OBFS  

Port Hopping：官方定位为对抗单端口限速/封锁（客户端多端口/范围随机跳，hopInterval 默认30s，可设 min/max）。若整个 UDP 被封则无效。支持单端口、多端口、范围。  
Salamander：把 QUIC 包变成随机字节（高熵 UDP）。  
Gecko（v2.9.2+，实验性）：在 Salamander 之上，进一步把 QUIC 长头握手包拆成 2–8 个随机大小 + 随机 padding 碎片。启用后不再是正常 HTTP/3。
两者可叠加，但解决不同问题。官方明确提醒混淆后流量形状改变。


REALITY 非必须 443，但 443 一致性更好
客户端连的是“你的 IP:端口”，服务端内部有 target / serverNames / shortIds / Dest 等独立配置。伪装机制不因端口非443而“失效”，但端口本身增加异常度（真实 HTTPS 高度集中于443）。Xray 的 RealiTLScanner 支持指定端口（默认443），协议本身无“必须443”限制。REALITY 可与 RAW / XHTTP / gRPC 组合。更准确表述是：443 时端口行为与常见 HTTPS 更一致；非443 会增加一点异常度，但伪装握手机制仍有效。


XHTTP（原 SplitHTTP）值得提升权重
Xray 明确推荐从 WebSocket/gRPC 切换到 XHTTP（HTTP/1.1/2/3、header padding、multiplex、上下行分离、packet-up/stream-up 等），强调中间盒/CDN 兼容性。与 VLESS + REALITY + Vision 的传输层行为明显不同，适合“非443 + TCP + 不想再复制一份 REALITY”的场景。


Restls、Mieru、AnyTLS 设计与现状  

Restls：目标明确解决 TLS-in-TLS、固定长度/时序、ClientHello 指纹。基于 utls 伪装浏览器，服务端伪装任意 TLS 站点（支持 TLS1.2/1.3），有 Restls-Script 主动控制前几组包长度、响应次数和 padding。作者承认可为伪装牺牲性能。生态（客户端覆盖、成熟度）弱于主流，适合作为异构备用。  
Mieru（enfein，v3.37.0 仍在更新至至少2026-09）：TCP/UDP + AEAD（XChaCha20-Poly1305，时间相关密钥）、随机 padding/分段、低熵编码、可调 traffic pattern（padding 0/1/2、fragment、nonce 等）。不依赖 TLS/域名/假站，抗主动探测设计明确。但高质量中国 GFW 公开测量不足，技术设计 ≠ 实测存活率。  
AnyTLS：TLS + session multiplex + 可配置 padding scheme + length pattern control。sing-box 1.12+ 正式支持，padding 可更新以应对特征变化。官方 FAQ 诚实：默认参数不保证永不被封。新协议，长期 GFW 公开测量少，但非常适合非443 TCP 灾备候选。


Cloudflare 端口与中国 Network
官方 HTTPS 默认支持：443、2053、2083、2087、2096、8443。中国 Network 的 HTTP/HTTPS 仅兼容 80/443。这些端口在全球 Web/CDN 生态有真实存在性，但不能推导“中国网络一定把它们当正常 HTTPS”。8443 作为非443 TCP 工程默认选择合理（比随机高端口更“像”常见服务）。


其他  

ShadowTLS + SS2022：真实 TLS 外层 + 独立 AEAD 层；V3 针对主动探测演进（challenge-response、application-data encapsulation 等）。仍有维护，但2025年有检测向量报告（如 ServerFinished 长度差异），需关注。  
Trojan + TLS + fallback：主动探测防御成熟（正确 password → proxy，否则 fallback 网站），但长期代理行为 ≠ 普通浏览器流量。  
NaiveProxy：Chromium 网络栈 + H2/H3 + 真实 Chrome 指纹 + padding + application fronting。frontend 存在时优势完整；2025 年伊朗环境有 padding 识别讨论（社区 issue，非 GFW 官方测量）。  
AmneziaWG：针对俄罗斯 TSPU/DPI 演进到 3.1（包大小、序列、时间间隔、元数据进一步随机化，2026年6–7月大规模封锁后更新）。俄罗斯证据强，中国证据不足，不能直接等同。现代 DPI 正从握手特征转向统计流量分析。  
具体存活率百分比（如“SS-2022 裸约45%”“Trojan 敏感期87%/76%/94%”“Hy2 比 TCP 快30–50%+”）缺乏控制运营商/地区/IP/时段变量的公开原始数据，不宜直接写入通用结论，否则降低可信度。社区样本仅供参考。

更完整的对比表（三维抗封锁 + 非443价值）

把“抗封锁”拆成：
P = 被动流量识别（熵、长度分布、时序、指纹 JA3/JA4、QUIC Initial 等）
A = 主动探测
T = 当前公开针对性研究/打击证据（论文、gfw.report、社区高质量测量）
非443价值 = 放在非443后，协议本身优势还剩多少（端口异常度 vs 伪装/混淆能力）。

组合 加密/混淆 P：被动识别 A：主动探测 性能 T：公开审查证据 非443价值 备注
Hysteria2 + Salamander QUIC/TLS + 随机字节 高 中 极高 高（QUIC SNI 论文） 很高 跳跃对抗单端口；全UDP封则失效
Hysteria2 + Gecko 同上 + 握手分片/padding 更强（实验） 中 极高 高 很高 v2.9.2+，生态支持逐步跟进
TUIC v5 QUIC/TLS 中 中 极高 高 高 与 Hy2 同属 QUIC 大类，适合协议切换而非网络层隔离
AnyTLS TLS + padding/multiplex/pattern 高 高（fallback 依赖） 高 公开证据少（新） 很高 sing-box 正式支持，可更新特征
ShadowTLS + SS2022 真实 TLS 外层 + AEAD 高 高 高 中（有检测向量报告） 很高 异构 TCP 经典组合
Trojan + TLS + fallback TLS 高 很强 高 中 很高 主动探测成熟，被动非无敌
NaiveProxy + H2 TLS/H2（真实 Chrome 栈） 很强 很强（frontend 时） 中～高 中/地区性证据 高 frontend 完整优势；padding 有针对性讨论
VLESS + XHTTP + TLS TLS/H2/H3 + padding/mux 高～很高 取决于前端 高 公开统一数据不足 很高 推荐替代 WS/gRPC；与 REALITY 传输层不同
VLESS + REALITY REALITY/TLS 高 高 很高 认知度高 中～高 443 一致性更好，非443 不失效但异常度↑
Restls + SS TLS + Script 行为控制 很高 很强 中 公开数据少 很高 模拟 TLS 全过程 + 可定制剧本
Mieru AEAD + padding/fragment/低熵/pattern 高 中～高 高 中国公开数据不足 很高 无 TLS/域名依赖，持续维护
AmneziaWG WG + 包/序列/时间/元数据随机 中～高 中 极高 俄罗斯强，中国不足 高 统计流量分析趋势
裸 SS2022 AEAD 中/高熵明显 中 极高 中 高 轻量，需插件/外层提升
WG/OpenVPN + 普通 obfs 强加密 + 基础混淆 中～低 中 高 较高 中 特征较明显

非443价值一列最关键：它衡量的是“协议优势在非标准端口下的保留程度”，而非单纯协议强度。

按网络栈与证据强度分类（故障隔离视角）

第一层：网络栈差异（真正隔离）  

TCP / TLS
├── VLESS + REALITY（主力，优先443）
├── AnyTLS
├── Trojan + fallback
├── ShadowTLS + SS2022
├── NaiveProxy（frontend 时）
├── Restls + SS
├── Mieru
└── VLESS + XHTTP + TLS

UDP / QUIC
├── Hysteria2（+Salamander/Gecko + hopping）
└── TUIC

UDP / WireGuard-derived
└── AmneziaWG

第二层：公开针对性审查证据  

最强：QUIC（Hy2/TUIC）——有 2025 USENIX 论文。  
中等：传统 TLS/代理（Trojan、ShadowTLS、SS、VLESS/REALITY、Naive）——长期研究 + 各种识别工作，但缺截至2026-09 统一“中国当前封锁率”公开实验。  
最弱但技术值得跟踪：AnyTLS、Restls、Mieru——证据弱 ≠ 协议弱，只是大规模中国公开测量时间不够。

针对“443 已有一条，再留真正不同后手”的收敛推荐

真正有意义的不是同时开十条，而是协议 + 网络栈故障隔离：

主力（443）：VLESS + REALITY (+ Vision)

TCP 非443 备：
- AnyTLS 或 ShadowTLS + SS2022（异构 TLS 外壳）
- VLESS + XHTTP + TLS
- Trojan + fallback
- （可选更深）Restls + SS 或 Mieru

UDP 备：
- Hysteria2 + Salamander/Gecko + Port Hopping（高端口范围，如20000-50000）
- TUIC（协议切换）

WG 备（可选）：
- AmneziaWG（注意中俄差异）

实践要点：  

多协议自动切换（sing-box / mihomo / Clash Meta URL-test 或 fallback）。  
非443 优先选 8443 等 Cloudflare 常见 alt 端口，或高范围 + hopping；避开常见服务端口。  
强加密（AEAD/TLS1.3）、定期换 IP/多节点、分流（国内直连）、监控流量模式。  
无绝对抗封锁；存活依赖运营（换 IP 频率、行为正常化）> 纯协议。敏感期/重大事件全面升级。  
测试：多 ISP（电信 UDP QoS 较严）、不同时段、真实应用（4K/游戏/会议）+ iperf 等。  
合规与安全：自建注意更新核心、强认证、防火墙；遵守当地法律。

仍需谨慎/待更多数据的地方：  

具体存活率百分比缺乏控制变量的公开原始数据，暂不采纳为通用指标。  
ShadowTLS 有新检测向量报告，AmneziaWG 中国证据不足，Naive 在伊朗有 padding 讨论（非直接 GFW 结论）。  
新协议（AnyTLS/Restls/Mieru/Gecko）生态与长期实测仍在演进，建议实际部署前多 ISP 验证，并关注 GitHub Discussions / gfw.report / 相关论文更新。

这份研究已经比早期版本全面得多。如果你需要某一组合的具体配置示例（服务端/客户端 YAML/JSON）、某一协议的更深协议分析、或针对特定 ISP/场景的部署建议，提供更多细节我可以继续细化。环境变化快，建议结合实时社区和官方文档验证。


chatgpt response

你这版已经可以作为“研究底稿”了。我再做一轮只针对容易被过度概括、可能随版本变化、以及证据等级不够的核验，然后给出最终收敛表。

对，这一版已经可以作为比较成熟的研究框架了。经过我刚才又按 2026-09-18 当前项目状态核了一遍，我会保留你的总体结构，但再做几处关键修正。尤其是 AnyTLS、Mieru、XHTTP、Hysteria2 的定位，现在可以更精确一些。

先给最终判断

对于你的目标：

443 已经有 VLESS + REALITY，另外准备非 443 的“真正备用”，同时考虑加密/混淆、性能、当前公开打击情况。



我现在会把候选收敛成：

主 TCP/443
VLESS + REALITY + Vision

第一 TCP 备用
AnyTLS

第二 TCP 备用
ShadowTLS + SS2022

第三 TCP 备用
VLESS + XHTTP + TLS

UDP 备用
Hysteria2 + Port Hopping
[是否使用 Salamander / Gecko 取决于你到底想伪装 QUIC 还是彻底改变 QUIC 外形]

可选第二 UDP
TUIC

研究型备用
Mieru / Restls

其中最值得你实际长期部署的不是十个，而是：

REALITY + AnyTLS/ShadowTLS+SS2022 + Hysteria2

这已经形成了比较明显的异构灾备。


1. AnyTLS：我会把它再往上调一档

你之前的表里把它写成：

TLS + padding/multiplex/pattern



这是对的，但现在有一个很重要的细节：

AnyTLS 官方协议已经不是简单的“TLS 里面套一个代理协议”。

它明确有：

TLS
 ↓
认证
 ↓
Session
 ↓
Stream
 ↓
padding / waste

而且服务器可以主动向客户端发送：

cmdUpdatePaddingScheme

让客户端更新 padding scheme。

更重要的是，官方协议文档明确写出了它的设计意图：

如果默认 padding scheme 被识别并进入黑名单，可以通过服务器下发新的 padding scheme，让后续连接产生不同的流量特征。(GitHub)



这使它和传统的：

Trojan + TLS
VLESS + TLS

有一个很明显的区别：

它从协议设计层面就把“流量模式可更新”作为一等公民。

这对你的“长期备用”思路非常重要。

但是你表中另一点要稍微修正：

“AnyTLS → TLS ClientHello/ServerHello 处理得很好”



这个不能完全归功于 AnyTLS 协议本身。

AnyTLS 官方 FAQ 明确说，项目本身并不专门处理 TLS ClientHello/ServerHello fingerprint，这些由现有 TLS 工具控制。(GitHub)

所以应当拆成：

AnyTLS 协议：
    session / padding / stream pattern

TLS 实现：
    ClientHello / ServerHello fingerprint

这是一个很重要的技术边界。


2. AnyTLS 的“新”既是优势，也是缺点

你这份研究把：

“公开针对性研究少”



归为证据弱。

这没问题，但还可以进一步说：

对灾备来说

新协议
+
目前公开针对研究较少

实际上是一个值得保留的维度。

但是：

公开研究少
≠
当前没有识别能力

而 AnyTLS 自己也明确说默认 padding 只是示例。(GitHub)

因此我不会把它称作：

“目前最抗封锁”。



更准确是：

“非常值得作为异构 TCP 备用，而且协议设计本身为后续变更流量特征留下了机制。”



这比“新协议所以不容易被封”严谨得多。


3. Mieru：你的判断是对的，而且现在值得认真纳入

这个我刚刚重新确认了一下。

截至 2026-09-13，Mieru 已经发布：

v3.37.0

而且 9 月 15 日仍有代码提交。(GitHub)

更重要的是，它的协议确实采用：

XChaCha20-Poly1305
+
time-dependent key
+
fragmentation
+
small segments

官方协议文档明确说明，当前版本使用 XChaCha20-Poly1305，并利用系统时间参与密钥生成；原始流会被分割成较小 fragment 后再加密封装。(GitHub)

所以 Mieru 的思路其实和：

REALITY = TLS mimic
AnyTLS = TLS + dynamic padding

不完全一样。

它更接近：

AEAD
+
时间相关密钥
+
fragment
+
traffic pattern

这就是为什么我现在更愿意把它列为：

“研究型异构 TCP/UDP 后手”



而不是普通 SS 替代品。

问题依然是你说的那个：

没有足够高质量的 2025–2026 中国大规模实测数据。

所以不能给它一个很高的“中国抗封锁”结论。


4. XHTTP：这里我会再修正一个你表中的地方

你写的是：

VLESS + XHTTP + TLS



这个没问题。

但实际上 XHTTP 现在比“一个新的 HTTP transport”要复杂得多。

当前 Xray 的实现已经支持：

HTTP/1.1
HTTP/2
HTTP/3

而且内部有：

Xmux
stream-one
stream-up
packet-up
download settings
upload chunking
header payload

等机制。当前源码甚至直接根据配置决定：

TLS → H1/H2
H3 → UDP/QUIC
REALITY → H2

并支持上下行分离。(GitHub)

所以：

XHTTP

已经不是简单的：

WS → HTTP

升级版。

它实际上是在构建一个：

HTTP 原生承载层。




5. 更关键的是：XHTTP 本身也能和 REALITY 组合

这一点你研究里其实已经提到了，但值得单独强调。

Xray 当前代码路径中，XHTTP 可以获取：

TLS config
+
REALITY config

并且对于 REALITY 会默认选择 HTTP/2 路径；最新的 issue 里也有实际的：

VLESS + XHTTP + REALITY

配置和运行案例。(GitHub)

所以不要简单把：

VLESS + REALITY

和：

VLESS + XHTTP + TLS

理解成两个完全互斥的方案。

更准确的分类是：

VLESS
├── TCP + REALITY
├── XHTTP + TLS
└── XHTTP + REALITY

这又给你增加了一个维度。


6. 但如果你的目标是“真正异构”，XHTTP + REALITY 不够异构

这个非常关键。

例如：

443
VLESS + REALITY + Vision

8443
VLESS + XHTTP + REALITY

虽然传输方式不同，但依然共享：

VLESS
REALITY
TLS/REALITY
Xray

所以它属于：

同生态不同 transport



而不是：

完全异构。



因此如果你的主要目标是：

“443 REALITY 出问题以后，我要有一个明显不同的技术栈。”



那么：

REALITY
+
AnyTLS

比：

REALITY
+
XHTTP+REALITY

更有价值。


7. Hysteria2 的 Port Hopping，我现在会进一步降低“抗 DPI”定位

这一点你研究已经很接近正确答案。

官方明确说：

Port Hopping 是解决某些 ISP 对特定 UDP 端口进行封锁/限速的办法。



而不是：

DPI 无法识别 Hysteria2。



官方甚至直接写明：

如果整个 UDP 被限制，Port Hopping 没有帮助。(Hysteria 2)



所以它应该在你的表里单独记为：

Port Hopping
= port-level resilience

不是：

DPI camouflage


8. Salamander / Gecko 又是第三件事情

可以画成：

Hysteria2
│
├── Port Hopping
│   └── 解决“单端口 UDP 限速/封锁”
│
├── Salamander
│   └── 改变 QUIC 包形态
│
└── Gecko
    └── QUIC handshake fragmentation + padding

所以它们其实是三个正交维度。

这比把：

Hy2 + Salamander + hopping

整体写成“高抗封套餐”更加准确。


9. QUIC 这一大类，我会继续给它一个醒目的警告

这是目前最有证据的一项。

2025 年 USENIX Security 研究已经证明：

QUIC Initial
↓
解析 TLS ClientHello
↓
提取 SNI
↓
QUIC-specific blocklist
↓
命中后丢包

这个机制不是理论推测，是实际测量结果。

因此：

Hy2
TUIC
XHTTP/3

只要底层仍落入：

UDP + QUIC

就应该被归到同一个较高相关性的审查面。

这就是为什么你要“留一手”时：

Hy2
+
TUIC

没有：

Hy2
+
AnyTLS

那么强的故障隔离价值。


10. ShadowTLS + SS2022，我仍然认为非常值得保留

它的价值不是：

“它一定比 REALITY 更抗封。”



而是：

REALITY
     ↓
ShadowTLS + SS2022

已经至少跨过了一层明显不同的协议设计。

而且 ShadowTLS V3 针对主动探测已经经历过专门演化。

但是你新补充的：

ServerFinished 长度差异等检测向量



我同意应该放进“持续关注”项，而不是直接把它打成弱协议。

这类研究更说明：

没有任何一个“TLS wrapper”可以永久依靠一套固定的握手/长度模式。



换句话说：

ShadowTLS
并非“不可识别”

只是：

主动探测防御 + TLS 外层

这两个维度做得比较成熟。


11. NaiveProxy 的位置依然很特殊

它真正的强项不是：

TLS

而是：

Chromium network stack
+
HTTP/2
+
padding
+
application fronting

因此：

普通 Naive server

与：

Naive + frontend

实际上是两个不同等级的部署形态。

而你的“非 443”要求恰好会削弱一点“看起来像普通 HTTPS”这一优势：

example.com:443

天然更普通。

相比：

example.com:8443

多了一点端口异常性。

所以我仍然不会把 Naive 放在你的第一备用。


12. 非 443 端口应该怎么选，我也会稍微修改你原来的策略

你说：

8443 等 Cloudflare alt ports



这在“全球 Web 生态中确实存在”这个意义上成立。

但是不要进一步推导：

8443 = GFW认为正常 HTTPS

这是两回事。

我更喜欢把端口分成三类：

第一类：Web/HTTPS 生态中的常见替代端口

8443
2053
2083
2087
2096

它们确实有真实 Web/HTTPS 使用场景。

第二类：普通服务端口

22
80
3306
6379

不建议随便拿来做代理，因为服务类型语义过强。

第三类：随机高端口

20000-50000

对 Hysteria2 + Port Hopping 特别有意义。

但对：

TCP + TLS

反而可能更容易表现出：

“随机加密长连接”

的统计异常。

所以：

8443 对 TCP TLS 备用比较自然；高端口范围对 UDP Port Hopping 更有意义。



这个区分值得写进最终报告。


13. 因此你的最终架构，我会收敛为四层

第一层：主力

TCP/443
VLESS
+
REALITY
+
Vision

特点：

高性能
TLS mimic
成熟


第二层：TCP 异构备用

二选一：

TCP/8443
AnyTLS

或者：

TCP/8443
ShadowTLS
+
SS2022

这里其实我更建议你两个都留，因为资源成本很低。


第三层：HTTP 生态备用

TCP/8443
VLESS
+
XHTTP
+
标准 TLS

它和：

REALITY

的 HTTP 行为明显不同。

而且 XHTTP 当前实现已经是真正的 H1/H2/H3 transport，不是简单 WS 替换。(GitHub)


第四层：UDP

UDP
Hysteria2
+
Port Hopping

然后根据目标：

普通 QUIC camouflage

或者：

Salamander / Gecko

二选其一。


14. 这样看，你真正需要的其实不是“十几个协议”

而是这四个故障域：

                  ┌─ TCP / REALITY
                  │
                  ├─ TCP / TLS-session
                  │
                  ├─ TCP / HTTP
                  │
                  └─ UDP / QUIC

对应：

443      REALITY
8443     AnyTLS / ShadowTLS+SS
8443     XHTTP+TLS
高端口   Hy2

如果再加：

UDP      TUIC

只是把 QUIC 领域内部再增加一个实现。


我现在对这些方案的最终定位

方案 我会把它定位成
VLESS + REALITY :443 主力 TCP
AnyTLS :8443 首选异构 TCP 备用
ShadowTLS + SS2022 :8443/2053 成熟异构 TCP 备用
VLESS + XHTTP + TLS :8443 HTTP 生态备用
Trojan + TLS + fallback 传统 TLS 备用
NaiveProxy 高仿真 HTTP/TLS 研究型方案
Hysteria2 + hopping 高性能 UDP 备用
TUIC 第二 QUIC 实现
Mieru 新型 AEAD/traffic-pattern 备用
Restls TLS 行为模拟研究型备用
AmneziaWG WG/UDP 异构备用，但中国公开证据不足
裸 SS2022 轻量备用，不适合作为主要伪装路线


一个特别重要的改进

你的研究现在已经不应该再使用一个简单的：

抗封锁 ★★★★★

因为这个评分会把完全不同的事情压扁。

我建议最终正式版改成：

P = Passive identification resistance
A = Active probe resistance
R = Resilience to port/IP blocking
E = Efficiency
D = Diversity / disaster-recovery value
T = Evidence quality

然后每个协议单独记录这 6 个指标，尤其把“技术设计”和“中国实际证据”分开。

这样你最后看到的就会非常清楚：

AnyTLS
P：高
A：取决于 fallback
R：高
E：高
D：★★★★★
T：★★

Hy2
P：中～高（取决于 obfs）
A：高
R：★★★★（hopping）
E：★★★★★
D：★★★（因为 QUIC）
T：★★★★★

Mieru
P：理论上高
A：中～高
R：高
E：高
D：★★★★★
T：★～★★

这比“谁五星谁三星”有意义得多。

另外，你前面提到自己的服务器环境比较偏小型 VPS、Alpine/OpenRC 一类；在这个条件下，Mieru、AnyTLS、ShadowTLS+SS2022、XHTTP、Hy2 的实际 CPU/RAM/单连接吞吐差异会非常值得测。Mieru 当前已经提供 x86_64/ARM64 Linux 包，9 月 13 日发布的 v3.37.0 也说明它不是停滞项目。(GitHub)

如果把这一步继续做下去，最有价值的下一份表其实不是“抗封锁排行榜”，而是 AnyTLS vs ShadowTLS+SS2022 vs XHTTP vs Mieru vs Trojan 的同机实测矩阵：CPU、RAM、TLS RTT、握手次数、1/10/100 并发、单流/多流吞吐、弱网丢包下表现、以及非443端口的实际客户端覆盖率。这样才能把现在大量“设计上应该怎样”的资料，变成你真正可以用来选节点的工程数据。

**这份研究整体框架扎实，纠正和补充方向正确，尤其是把“非443”当成独立变量、区分 Port Hopping 与 OBFS、强调 QUIC 已有专门审查路径、以及把 XHTTP / Restls / Mieru / AnyTLS 纳入主表，都比早期简化版更贴近 2025–2026 年公开证据。** 我基于官方文档、gfw.report、USENIX Security 2025 论文、项目仓库和社区实测做了交叉核验，确认核心事实成立，同时标出仍需谨慎的地方，并给出更完整的对比与推荐收敛。

### 关键事实核验（有公开来源支撑）

1. **2025年8月20日 TCP/443 无条件 RST+ACK 事件是真实的**  
   北京时间约 00:34–01:48（约74分钟），GFW 对 TCP/443 注入伪造 RST+ACK，其他端口（22、80、8443 等）未受同样影响。触发不对称（出站 SYN 和 SYN+ACK 都可触发，入站主要靠 SYN+ACK），设备指纹与已知 GFW 组件不完全匹配，可能是新设备或已知设备异常/误配置状态。报告明确记录为短暂异常，**不能**推导出“8443 永久比 443 安全”。这正好证明“443 主路 + 非443 TCP 备路 + UDP/QUIC 备路”的价值，而不是“非443 = 更安全”。

2. **QUIC SNI 审查（2024年4月起）有高质量论文支撑**  
   USENIX Security 2025 论文（Zohaib 等）证实 GFW 可大规模解密 QUIC Initial 包、提取 TLS ClientHello/SNI，使用独立 blocklist，命中后约180秒 UDP 丢包/阻断。blocklist 与传统 TLS/HTTP/DNS 不同，且解密开销在中等负载下会降低有效性，可被武器化阻断任意 UDP。因此 Hy2 / TUIC 同属 **UDP + QUIC + TLS Initial** 大类，不能视为完全独立灾备；“QUIC 全加密所以 DPI 难”已过时。

3. **Hysteria2 Port Hopping ≠ OBFS**  
   - **Port Hopping**：官方定位为对抗单端口限速/封锁（客户端多端口/范围随机跳，hopInterval 默认30s，可设 min/max）。若整个 UDP 被封则无效。支持单端口、多端口、范围。  
   - **Salamander**：把 QUIC 包变成随机字节（高熵 UDP）。  
   - **Gecko**（v2.9.2+，实验性）：在 Salamander 之上，进一步把 QUIC 长头握手包拆成 2–8 个随机大小 + 随机 padding 碎片。启用后不再是正常 HTTP/3。  
   两者可叠加，但解决不同问题。官方明确提醒混淆后流量形状改变。

4. **REALITY 非必须 443，但 443 一致性更好**  
   客户端连的是“你的 IP:端口”，服务端内部有 `target` / `serverNames` / `shortIds` / `Dest` 等独立配置。伪装机制不因端口非443而“失效”，但端口本身增加异常度（真实 HTTPS 高度集中于443）。Xray 的 RealiTLScanner 支持指定端口（默认443），协议本身无“必须443”限制。REALITY 可与 RAW / XHTTP / gRPC 组合。更准确表述是：**443 时端口行为与常见 HTTPS 更一致；非443 会增加一点异常度，但伪装握手机制仍有效。**

5. **XHTTP（原 SplitHTTP）值得提升权重**  
   Xray 明确推荐从 WebSocket/gRPC 切换到 XHTTP（HTTP/1.1/2/3、header padding、multiplex、上下行分离、packet-up/stream-up 等），强调中间盒/CDN 兼容性。与 VLESS + REALITY + Vision 的传输层行为明显不同，适合“非443 + TCP + 不想再复制一份 REALITY”的场景。

6. **Restls、Mieru、AnyTLS 设计与现状**  
   - **Restls**：目标明确解决 TLS-in-TLS、固定长度/时序、ClientHello 指纹。基于 utls 伪装浏览器，服务端伪装任意 TLS 站点（支持 TLS1.2/1.3），有 Restls-Script 主动控制前几组包长度、响应次数和 padding。作者承认可为伪装牺牲性能。生态（客户端覆盖、成熟度）弱于主流，适合作为异构备用。  
   - **Mieru**（enfein，v3.37.0 仍在更新至至少2026-09）：TCP/UDP + AEAD（XChaCha20-Poly1305，时间相关密钥）、随机 padding/分段、低熵编码、可调 traffic pattern（padding 0/1/2、fragment、nonce 等）。不依赖 TLS/域名/假站，抗主动探测设计明确。但高质量中国 GFW 公开测量不足，技术设计 ≠ 实测存活率。  
   - **AnyTLS**：TLS + session multiplex + 可配置 padding scheme + length pattern control。sing-box 1.12+ 正式支持，padding 可更新以应对特征变化。官方 FAQ 诚实：默认参数不保证永不被封。新协议，长期 GFW 公开测量少，但非常适合非443 TCP 灾备候选。

7. **Cloudflare 端口与中国 Network**  
   官方 HTTPS 默认支持：443、2053、2083、2087、2096、8443。中国 Network 的 HTTP/HTTPS **仅兼容 80/443**。这些端口在全球 Web/CDN 生态有真实存在性，但不能推导“中国网络一定把它们当正常 HTTPS”。8443 作为非443 TCP 工程默认选择合理（比随机高端口更“像”常见服务）。

8. **其他**  
   - ShadowTLS + SS2022：真实 TLS 外层 + 独立 AEAD 层；V3 针对主动探测演进（challenge-response、application-data encapsulation 等）。仍有维护，但2025年有检测向量报告（如 ServerFinished 长度差异），需关注。  
   - Trojan + TLS + fallback：主动探测防御成熟（正确 password → proxy，否则 fallback 网站），但长期代理行为 ≠ 普通浏览器流量。  
   - NaiveProxy：Chromium 网络栈 + H2/H3 + 真实 Chrome 指纹 + padding + application fronting。frontend 存在时优势完整；2025 年伊朗环境有 padding 识别讨论（社区 issue，非 GFW 官方测量）。  
   - AmneziaWG：针对俄罗斯 TSPU/DPI 演进到 3.1（包大小、序列、时间间隔、元数据进一步随机化，2026年6–7月大规模封锁后更新）。俄罗斯证据强，**中国证据不足**，不能直接等同。现代 DPI 正从握手特征转向统计流量分析。  
   - 具体存活率百分比（如“SS-2022 裸约45%”“Trojan 敏感期87%/76%/94%”“Hy2 比 TCP 快30–50%+”）缺乏控制运营商/地区/IP/时段变量的公开原始数据，**不宜直接写入通用结论**，否则降低可信度。社区样本仅供参考。

### 更完整的对比表（三维抗封锁 + 非443价值）

把“抗封锁”拆成：  
**P** = 被动流量识别（熵、长度分布、时序、指纹 JA3/JA4、QUIC Initial 等）  
**A** = 主动探测  
**T** = 当前公开针对性研究/打击证据（论文、gfw.report、社区高质量测量）  
**非443价值** = 放在非443后，协议本身优势还剩多少（端口异常度 vs 伪装/混淆能力）。

| 组合 | 加密/混淆 | P：被动识别 | A：主动探测 | 性能 | T：公开审查证据 | 非443价值 | 备注 |
|------|-----------|-------------|-------------|------|-----------------|-----------|------|
| **Hysteria2 + Salamander** | QUIC/TLS + 随机字节 | 高 | 中 | 极高 | **高**（QUIC SNI 论文） | 很高 | 跳跃对抗单端口；全UDP封则失效 |
| **Hysteria2 + Gecko** | 同上 + 握手分片/padding | 更强（实验） | 中 | 极高 | **高** | 很高 | v2.9.2+，生态支持逐步跟进 |
| **TUIC v5** | QUIC/TLS | 中 | 中 | 极高 | **高** | 高 | 与 Hy2 同属 QUIC 大类，适合协议切换而非网络层隔离 |
| **AnyTLS** | TLS + padding/multiplex/pattern | 高 | 高（fallback 依赖） | 高 | 公开证据少（新） | **很高** | sing-box 正式支持，可更新特征 |
| **ShadowTLS + SS2022** | 真实 TLS 外层 + AEAD | 高 | 高 | 高 | 中（有检测向量报告） | **很高** | 异构 TCP 经典组合 |
| **Trojan + TLS + fallback** | TLS | 高 | **很强** | 高 | 中 | **很高** | 主动探测成熟，被动非无敌 |
| **NaiveProxy + H2** | TLS/H2（真实 Chrome 栈） | **很强** | **很强**（frontend 时） | 中～高 | 中/地区性证据 | 高 | frontend 完整优势；padding 有针对性讨论 |
| **VLESS + XHTTP + TLS** | TLS/H2/H3 + padding/mux | 高～很高 | 取决于前端 | 高 | 公开统一数据不足 | **很高** | 推荐替代 WS/gRPC；与 REALITY 传输层不同 |
| **VLESS + REALITY** | REALITY/TLS | 高 | 高 | 很高 | **认知度高** | 中～高 | 443 一致性更好，非443 不失效但异常度↑ |
| **Restls + SS** | TLS + Script 行为控制 | **很高** | 很强 | 中 | 公开数据少 | **很高** | 模拟 TLS 全过程 + 可定制剧本 |
| **Mieru** | AEAD + padding/fragment/低熵/pattern | 高 | 中～高 | 高 | 中国公开数据不足 | **很高** | 无 TLS/域名依赖，持续维护 |
| **AmneziaWG** | WG + 包/序列/时间/元数据随机 | 中～高 | 中 | **极高** | **俄罗斯强，中国不足** | 高 | 统计流量分析趋势 |
| **裸 SS2022** | AEAD | 中/高熵明显 | 中 | **极高** | 中 | 高 | 轻量，需插件/外层提升 |
| **WG/OpenVPN + 普通 obfs** | 强加密 + 基础混淆 | 中～低 | 中 | 高 | 较高 | 中 | 特征较明显 |

**非443价值**一列最关键：它衡量的是“协议优势在非标准端口下的保留程度”，而非单纯协议强度。

### 按网络栈与证据强度分类（故障隔离视角）

**第一层：网络栈差异（真正隔离）**  
```
TCP / TLS
├── VLESS + REALITY（主力，优先443）
├── AnyTLS
├── Trojan + fallback
├── ShadowTLS + SS2022
├── NaiveProxy（frontend 时）
├── Restls + SS
├── Mieru
└── VLESS + XHTTP + TLS

UDP / QUIC
├── Hysteria2（+Salamander/Gecko + hopping）
└── TUIC

UDP / WireGuard-derived
└── AmneziaWG
```

**第二层：公开针对性审查证据**  
- **最强**：QUIC（Hy2/TUIC）——有 2025 USENIX 论文。  
- **中等**：传统 TLS/代理（Trojan、ShadowTLS、SS、VLESS/REALITY、Naive）——长期研究 + 各种识别工作，但缺截至2026-09 统一“中国当前封锁率”公开实验。  
- **最弱但技术值得跟踪**：AnyTLS、Restls、Mieru——证据弱 ≠ 协议弱，只是大规模中国公开测量时间不够。

### 针对“443 已有一条，再留真正不同后手”的收敛推荐

真正有意义的不是同时开十条，而是**协议 + 网络栈故障隔离**：

```
主力（443）：VLESS + REALITY (+ Vision)

TCP 非443 备：
- AnyTLS 或 ShadowTLS + SS2022（异构 TLS 外壳）
- VLESS + XHTTP + TLS
- Trojan + fallback
- （可选更深）Restls + SS 或 Mieru

UDP 备：
- Hysteria2 + Salamander/Gecko + Port Hopping（高端口范围，如20000-50000）
- TUIC（协议切换）

WG 备（可选）：
- AmneziaWG（注意中俄差异）
```

**实践要点**：  
- 多协议自动切换（sing-box / mihomo / Clash Meta URL-test 或 fallback）。  
- 非443 优先选 8443 等 Cloudflare 常见 alt 端口，或高范围 + hopping；避开常见服务端口。  
- 强加密（AEAD/TLS1.3）、定期换 IP/多节点、分流（国内直连）、监控流量模式。  
- 无绝对抗封锁；存活依赖运营（换 IP 频率、行为正常化）> 纯协议。敏感期/重大事件全面升级。  
- 测试：多 ISP（电信 UDP QoS 较严）、不同时段、真实应用（4K/游戏/会议）+ iperf 等。  
- 合规与安全：自建注意更新核心、强认证、防火墙；遵守当地法律。

**仍需谨慎/待更多数据的地方**：  
- 具体存活率百分比缺乏控制变量的公开原始数据，暂不采纳为通用指标。  
- ShadowTLS 有新检测向量报告，AmneziaWG 中国证据不足，Naive 在伊朗有 padding 讨论（非直接 GFW 结论）。  
- 新协议（AnyTLS/Restls/Mieru/Gecko）生态与长期实测仍在演进，建议实际部署前多 ISP 验证，并关注 GitHub Discussions / gfw.report / 相关论文更新。

这份研究已经比早期版本全面得多。如果你需要某一组合的具体配置示例（服务端/客户端 YAML/JSON）、某一协议的更深协议分析、或针对特定 ISP/场景的部署建议，提供更多细节我可以继续细化。环境变化快，建议结合实时社区和官方文档验证。

对当前适合在非 443 端口运行的主流与前沿代理协议进行系统梳理，并结合 2025–2026 年公开的审查测量（如 USENIX Security 2025
QUIC 审查论文、GFW 端口级异常事件分析及相关项目演进）进行深度对比。

一、 核心认知：非 443 端口的本质逻辑与双刃剑效应

在设计非 443 节点前，必须明确非标准端口在网络拓扑中的利弊：

1.  为什么选择非 443 端口？

      - 规避 443 端口级策略/异常事件：例如 2025 年 8 月 GFW 针对 TCP/443 触发的短暂无条件伪造 RST+ACK
        注入事件，非标端口（如 8443、80 等）完全未受波及。
      - 对抗运营商 QoS 与端口黑洞：部分省份运营商（尤其是中国电信国际出口）对非 443/80 的 TCP 或公网 UDP 实施了差异化的 QoS
        丢包策略，或相反地对长期占用 443 的高吞吐 IP 实施精准封锁。
      - 配合端口跳跃（Port
        Hopping）：通过防火墙（iptables/nftables）将高端口范围（如 20000–50000）重定向到服务端，对抗“IP
        + 单端口”封锁与单端口流量限速。

2.  非 443 端口的天然劣势：

      - 统计异常度上升：公网真实 HTTPS 流量 >95% 集中在 443 端口。若在 34567 这样的随机高端口上运行长时间、大流量的 TLS
        握手与加密流，在 ISP 边缘统计分析（NetFlow/IPFIX）中会直接归为“高熵长连接异常”，显著提高被被动 DPI 抽检的概率。
      - 客户端网络环境出站限制：大量校园网、企业内网、酒店 Wi-Fi 的防火墙采用默认拒绝策略，仅放行 TCP 80/443 与
        UDP 53，非标准端口在此类网络下直接不可达。

3.  非 443 端口选型建议：

      - CDN / Web 替代端口（推荐用于 TCP+TLS）：8443、2053、2083、2087、2096。这些是 Cloudflare 等主流
        CDN 兼容的 HTTPS 端口，在公共日志中具有一定的真实业务背景，推荐作为 TCP 备用端口的首选。
      - 高端口范围（推荐用于 UDP 跳跃）：20000–50000。专用于 Hysteria 2 等 UDP 端口跳跃场景。
      - 业务端口（强烈避开）：22、3306、6379 等。语义过强，一旦出现非对应协议的行为，极易被主动探测系统告警。

二、 适合非 443 端口的全协议方案图谱

当前生态下适合非 443 端口的方案可划分为三大路线：

非 443 代理方案全景
├── 1. UDP / QUIC 协议族（弱网加速与动态跳跃）
│   ├── Hysteria 2 (+ Port Hopping + Salamander / Gecko)
│   └── TUIC v5
├── 2. 现代 TLS 增强与传输层革新（TCP 流量伪装）
│   ├── AnyTLS (内置动态 Padding Scheme 与 Session 复用)
│   ├── ShadowTLS v3 + Shadowsocks-2022 (外层 TLS 握手解耦)
│   ├── VLESS + XHTTP + TLS (原 SplitHTTP，支持分包与流式模式)
│   ├── Trojan + TLS + 真实 Web Fallback (成熟防主动探测)
│   ├── NaiveProxy + H2 (真实 Chromium 网络栈仿真)
│   └── Restls + SS/VLESS (可编程 TLS 交互剧本)
└── 3. 非 TLS / 独立混淆协议族（彻底脱离 Web 伪装体系）
    ├── Mieru (XChaCha20-Poly1305 + 时间密钥 + 动态分片/低熵)
    ├── AmneziaWG (AWG 3.1，WireGuard 数据包头与间隔随机化)
    └── 裸 Shadowsocks-2022 (极简 AEAD，在非 443 仅作内网/低审查备用)

三、 核心多维对比矩阵

采用六维评估模型：

  - P（被动识别防御）：对抗流量特征（熵值、包长序列、时序特征、ClientHello 指纹、QUIC Initial 解析）的能力。
  - A（主动探测防御）：面对审查系统发起的重放、格式错乱请求或主动 TLS 探针时的行为真实度。
  - E（性能与效率）：吞吐量、握手延迟、抗丢包能力，以及在小内存/低配 VPS 上的 CPU/RAM 资源开销。
  - R（抗端口封锁/QoS）：对抗单端口阻断、单端口限速的能力。
  - T（公开审查证据等级）：学术论文、gfw.report 等公开渠道确认的被针对性打击程度（证据越充分，说明已被纳入重点监控范畴）。
  - 非 443 适配度：该方案搬到非 443 端口后，自身伪装机制的有效性与合理性。

| 方案组合                                    | 协议与传输层            | P: 被动识别                    | A: 主动探测                         | E: 性能效率              | R: 抗端口限制                | T: 公开打击证据                            | 非 443 适配度 | 核心机制与定位                                   |
| :-------------------------------------- | :---------------- | :------------------------- | :------------------------------ | :------------------- | :---------------------- | :----------------------------------- | :-------- | :---------------------------------------- |
| **Hysteria 2** *(+ Salamander / Gecko)* | UDP / QUIC        | 高（Salamander）<br>极高（Gecko） | 中～高 *(伪装 H3 站点)*                | 极高 *(Brutal 拥塞控制)*   | **极高** *(Port Hopping)* | **高** *(2024–2025 QUIC SNI 解密阻断已确认)* | ★★★★★     | 弱网吞吐主力；跳跃对抗端口封锁；Gecko 解决握手特征              |
| **TUIC v5**                             | UDP / QUIC        | 中 *(原生 QUIC 特征)*           | 中 *(握手认证)*                      | 极高 *(原生多路复用)*        | 中 *(需外部跳跃)*             | **高** *(受 QUIC 审查链条波及)*              | ★★★★☆     | 原生标准 QUIC 代理，生态成熟但抗特征识别稍逊于 Hy2            |
| **AnyTLS**                              | TCP / TLS         | 高 *(动态 Padding)*           | 高 *(依赖 Fallback)*               | 高 *(Session 池化)*     | 中 *(固定端口)*              | **低** *(较新，针对性公开研究少)*                | ★★★★★     | 原生支持 8443 等端口；支持服务端动态下发 Padding 策略        |
| **ShadowTLS v3 + SS2022**               | TCP / TLS+AEAD    | 高 *(借用目标 TLS 握手)*          | 高 *(V3 挑战响应机制)*                 | 高 *(Rust 极低开销)*      | 中 *(固定端口)*              | **中** *(已有 ServerFinished 长度差研究)*    | ★★★★★     | 端口与 SNI 目标解耦；将真实 TLS 外壳与 AEAD 内层隔离        |
| **VLESS + XHTTP + TLS**                 | TCP / H1, H2, H3  | 很高 *(XMUX + Padding)*      | 高 *(应用层路由)*                     | 高 *(流式 stream-up)*   | 中 *(固定端口)*              | **中** *(Xray 新生代，长期数据积累中)*           | ★★★★★     | 替代 WS/gRPC；解决中间盒兼容性，天然适合 8443 运行          |
| **Trojan + TLS + Fallback**             | TCP / TLS         | 高 *(标准 TLS 1.3)*           | **极强** *(真实 Web 服务回落)*          | 高 *(成熟协议栈)*          | 中 *(固定端口)*              | **中** *(经典协议，长期研究基线)*                | ★★★★☆     | 最成熟的主动探测防守方案，建议配合真实证书与 Nginx/Caddy        |
| **NaiveProxy + H2/H3**                  | TCP/UDP / H2/H3   | **极强** *(Chromium 原生栈)*    | **极强** *(Application Fronting)* | 中～高 *(双层加密/内存稍大)*    | 中 *(固定端口)*              | **中** *(海外曾有 padding 讨论)*            | ★★★☆☆     | TLS 指纹最逼真；但在非 443 运行略显不自然（Chrome 较少直连高端口） |
| **VLESS + REALITY**                     | TCP / TLS 伪装      | 高 *(uTLS 模拟)*              | 高 *(目标站握手借用)*                   | 很高 *(XTLS Vision)*   | 中 *(固定端口)*              | **中～高** *(认知度极高，uTLS 存在微小盲区)*        | ★★★☆☆     | 虽可在 8443 运行且握手不失效，但借用 443 站点在逻辑上不如原版自洽    |
| **Restls + SS**                         | TCP / TLS+AEAD    | **极高** *(行为剧本控制)*          | 强 *(双向认证)*                      | 中 *(剧本延时开销)*         | 中 *(固定端口)*              | **低** *(小众生态，公开测量不足)*                | ★★★★★     | 模拟真实浏览器多阶段交互；针对 TLS-in-TLS 特征定制           |
| **Mieru (mita)**                        | TCP/UDP / 自研 AEAD | 高 *(动态分段与低熵)*              | 强 *(无握手回包静默)*                   | 高 *(Go 原生，开销低)*      | 高 *(支持端口范围/跳跃)*         | **极低** *(缺乏公开大规模对抗数据)*               | ★★★★★     | 彻底摆脱 TLS/域名依赖；采用时间窗口密钥与动态填充               |
| **AmneziaWG (AWG 3.1)**                 | UDP / WG 变体       | 中～高 *(包头/大小/时序随机)*         | 中 *(单向握手校验)*                    | 极高 *(内核/Go 线速)*      | 中～高 *(可配合端口跳跃)*         | **高 (俄罗斯) / 低 (中国)**                 | ★★★★☆     | 针对 Russia TSPU 深度演化；抗统计分析，但对深度连接追踪防御有限    |
| **Shadowsocks-2022 (裸)**                | TCP/UDP / AEAD    | 低～中 *(全随机高熵流)*             | 中 *(静默丢弃)*                      | **极高** *(超低 CPU 开销)* | 中 *(固定端口)*              | **中～高** *(高熵流量极易被机器学习特征捕获)*          | ★★★☆☆     | 不建议直接裸奔暴露在非 443，缺乏 TLS/Web 伪装保护           |

四、 核心技术深入剖析与审查打击现状核查

1. UDP / QUIC 赛道：审查现状与防护升级

  - QUIC 的审查现实（USENIX Security 2025 论文核实）： 过去“QUIC 握手加密，因此 DPI 无法识别”的观点已被证伪。GFW
    早已部署了对 QUIC Initial 包的大规模解密能力，能提取明文的 TLS ClientHello 及 SNI，并结合专用的 QUIC
    域名黑名单实施阻断（阻断触发后通常伴随约 180 秒的 UDP 单向丢包）。
  - Hysteria 2 的三层防护机制区分：
      - Port Hopping（端口跳跃）：解决的是网络传输层与 ISP QoS 问题。当单个 UDP
        端口被限速、丢包或封锁时，客户端在端口池中周期性轮换。如果整个 UDP 协议被封杀，Port Hopping 无效。
      - Salamander（XOR 随机化）：解决的是协议类型识别问题。将 QUIC 报文打碎为纯随机高熵字节流，使其脱离 QUIC
        协议解析器的识别范围。
      - Gecko（分片与填充，v2.9.2+）：解决的是长报文统计指纹问题。QUIC 握手阶段的 Long Header
        报文具有特征性的长度分布。Gecko 将握手包拆分成 2–8 个随机大小的碎片并辅以随机 Padding，彻底抹除握手报文的几何特征。

2. AnyTLS：动态流量模式控制

AnyTLS 在 2025–2026 年快速被 sing-box、mihomo 等内核采纳。其突破性设计在于：

  - 解耦 TLS-in-TLS：不仅解决 TLS 套 TLS 的多层嵌套长度泄露，还引入了应用层 Session
    Multiplex（多路复用）与废弃填充（Padding Waste）。
  - 动态更新特征（cmdUpdatePaddingScheme）：传统的 Trojan 或 VLESS，其填充逻辑是硬编码在二进制中的。AnyTLS
    允许服务端在连接建立后向客户端下发新的填充方案。如果某一填充算法被 GFW
    标记入库，运维端只需升级服务端配置即可改变全体客户端的外观流量特征，无需紧急替换协议栈。

3. XHTTP（原 SplitHTTP）：现代 HTTP 承载体系

Xray 推出的 XHTTP 是替代 WebSocket 和 gRPC 的战略级传输层：

  - 模式多样性：支持 stream-up（流式上行，避免早期分包导致的延迟放大）、stream-one（双向流）以及 packet-up（UDP
    报文承载）。
  - 中间盒亲和力：XHTTP 可完美通过各种只支持标准 HTTP/1.1、HTTP/2 或 HTTP/3 的企业反向代理和 CDN，且内置
    XMUX（多路复用控制）和 Header Padding。在非 443 端口（如 8443）上，VLESS + XHTTP +
    真实证书构成了一个极其稳固的标准 Web 流量伪装链条。

4. ShadowTLS 与 REALITY 的机制差异

  - REALITY 的端口局限：REALITY 核心在于借用公网大站的证书与私钥完成 ClientHello 验证。公网大站通常只在 443 提供
    HTTPS 服务。虽然配置 REALITY 监听 8443 并指定目标站的 443 依然可以握手成功，但这在被动监测视角下会呈现“访问目标站 8443
    端口但完成了 443 SNI 协商”的逻辑破绽。
  - ShadowTLS 的优势：ShadowTLS v3 从设计之初就将“本端监听端口”与“回源 SNI
    目标的端口”彻底解耦。服务端可以极其自然地监听 0.0.0.0:8443 或 2053，同时将未通过认证的主动探测透明代理到任意目标站的 443
    端口，非常适合作为非 443 的 TCP 备用方案。

5. Mieru：彻底脱离 TLS 体系的研究型路径

如果未来遭遇针对整个 Web/TLS 体系的极端阻断（如强制白名单 SNI、全量 TLS 握手降级干扰）：

  - Mieru 采用了无握手协议（No handshake round-trip），基于系统时间动态计算临时密钥，报文无固定特征头。
  - 提供 Low-entropy（低熵）编码模式与动态 Fragment，能够让流量在统计学上模拟压缩文件或多媒体流，规避“高熵即代理”的启发式审查模型。

五、 落地建议：“443 主力 + 非 443 异构后手”架构

为了实现真正的故障域隔离（Fault Domain
Isolation），不应在不同端口上简单复制相同的协议栈，而应确保每个节点在传输层、加密层与伪装模型上均不相同。

推荐收敛架构拓扑：

                    【用户终端】
                         │
        ┌────────────────┼────────────────┬────────────────┐
        │ [主力快速通道] │ [TCP 第一异构] │ [HTTP 生态]   │ [UDP 弱网爆发]
        ▼                ▼                ▼                ▼
     TCP / 443        TCP / 8443       TCP / 8443      UDP / 20000-50000
 ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐
 │    VLESS     │ │    AnyTLS    │ │   VLESS +    │ │   Hysteria 2     │
 │   REALITY    │ │      或      │ │    XHTTP     │ │  + Port Hopping  │
 │   + Vision   │ │ ShadowTLS+SS │ │    + TLS     │ │  + Gecko/Salama  │
 └──────────────┘ └──────────────┘ └──────────────┘ └──────────────────┘
 [真实大站伪装]   [独立TLS会话]    [标准H2/H3流式]   [QUIC分片/随机化]

各节点具体分工与落地指南：

1.  主力节点（TCP 443）：VLESS + REALITY + Vision

      - 角色：日常主力，享受 443 端口原生 HTTPS 隐蔽性与 XTLS 原生线速转发。
      - 适用场景：常规浏览、大文件下载、低延迟网页交互。

2.  TCP 第一备用（TCP 8443）：AnyTLS 或 ShadowTLS v3 + SS-2022

      - 角色：防范 443 端口级策略故障、RST 注入或 REALITY 指纹被定点清除。
      - 选型依据：若追求最新动态混淆与连接池复用，选择 AnyTLS（sing-box/mihomo 生态支持良好）；若追求极低 CPU 占用与
        Rust 极高并发稳定性（特别是低配 VPS），选择 ShadowTLS + SS-2022。

3.  HTTP 备用（TCP 8443 / 2053）：VLESS + XHTTP + 标准 TLS

      - 角色：完全遵循现代 HTTP/2 或 HTTP/3 语义的传输链路。
      - 价值：一旦常规 TCP 代理协议的二进制流行为被统计模型拦截，XHTTP 的流式上行与 Padding 机制能够以纯粹的 Web
        交互形式提供极高的存活率。需配合自有域名和真实 CA 证书。

4.  UDP 备用（UDP 20000–50000 范围）：Hysteria 2 + Port Hopping (+ Gecko)

      - 角色：高峰期网络拥塞、晚高峰晚点对线、丢包率高时的提速利器，以及 TCP 全面受阻时的 UDP 逃生通道。
      - 配置要点：
          - 服务端配置 iptables -t nat -A PREROUTING -p udp --dport 20000:50000 -j
            REDIRECT --to-ports 44300；
          - 客户端端口填入范围（如 20000-50000），启用 hopInterval: 30s；
          - 若所在 ISP 对 QUIC 特征审查严格，启用 obfs: gecko，对握手长报文做拆分 Padding。

5.  冷备后手（研究型储备）：Mieru (mita)

      - 角色：极端敏感时期，域名 DNS 污染、证书撤销、全局 SNI 阻断下的终极兜底方案。不依赖任何公网证书与 Web
        假站，独立监听非标端口（如 18888）。

部署细节与实践经验：

  - 内存受限环境（Alpine / OpenRC / 512MB VPS）：优先选用 C/Rust 编写的组件（如 ShadowTLS +
    shadowsocks-rust）或资源占用控制极佳的单个 sing-box 进程整合 AnyTLS 与 Hy2；尽量避免同时启动重量级的
    Chromium 依赖型进程（如原生 NaiveProxy）或开启过多需要大缓存的 XHTTP 多路复用实例。
  - 分流与探测规避：客户端务必配置严格的分流规则（中国大陆 IP 及常见域名直连），避免因为内网探测行为将海外非 443 节点暴露在本地 DPI
    的审计链路中。同时，对外暴露的 Web 类非标端口，务必配置合理的默认回落（Fallback），对探测请求返回合法的 HTTP 响应代码（如 400
    Bad Request 或 404 Not Found），拒绝任何未经验证的静默挂起。
    
    
    在科学防封锁与网络代理架构中，443 端口是地位最特殊、流量最集中、对抗也最激烈的战略级端口。

如果说非 443 端口的策略是“避开拥挤的主干道，在旁路小道上利用冷门端口或跳跃端口降低暴露”，那么 443 端口的策略则是**“大隐隐于市——在互联网最核心的
HTTPS/QUIC 流量洪流中实现不可区分性”**。

在 443
端口部署代理方案，需要系统性地梳理适合该端口的主流协议组合，从**加密混淆、被动流量识别（P）、主动探测防御（A）、封锁抗性（R）、性能效率（E）、架构异构多样性（D）与公开打击证据等级（T）**等维度进行深度对比与总结。

一、 443 端口的本质特征与博弈环境

在评估具体协议前，必须明确 443 端口在审查模型中的双刃剑效应：

1.  天然掩护性（Collateral Damage Shield）： 全球 80% 以上的合法 Web 浏览、API 调用、移动应用数据均运行在
    TCP/443（HTTPS）和 UDP/443（HTTP/3 / QUIC）上。审查系统对 443 端口的 IP
    级或端口级封锁需要承受极高的误杀代价（如 2025 年 8 月短暂出现的 443 全量 RST 异常仅持续约 74 分钟即被终止）。
2.  强制合规性（Zero-Tolerance for Non-TLS）： 非 443
    端口如果出现高熵未加密数据，可能仅仅被视为未知私有协议；但如果在 443
    端口传输非 TLS / 非 QUIC 格式的高熵流量（如裸 Shadowsocks、裸 WireGuard），会瞬间触发 DPI 的“443
    协议违规（Protocol Violation）”启发式封锁。443 端口要求外层必须是严格合规的 TLS 1.3 握手或 QUIC 握手。
3.  高强度主动探测（Active Probing Ground Zero）： 审查系统对境外的 443 端口建立了常态化的主动探测雷达。如果一个 443
    端口有持续大流量，但收到针对常见 Web 漏洞的请求、错误的 TLS 握手或未认证 HTTP
    请求时直接断开（RST）或超时挂起，就会被判定为“代理服务器特征”，触发封锁。
4.  TCP/UDP 双栈共存优势（Dual-Stack Binding）： 操作系统的套接字（Socket）允许在同一个 IPv4/IPv6 上，分别将
    TCP/443 和 UDP/443 绑定给不同的服务。这使得单个 VPS 可以在 443 上同时部署一套顶级 TCP 伪装方案和一套高性能
    UDP/QUIC 方案，模拟真实 Web 服务器的 HTTPS/H3 双栈行为。

二、 2026年适合 443 端口的主流代理方案全景图

根据 TLS 伪装的底层逻辑，443 端口的代理组合可划分为四大技术流派：

                    ┌── 借鸡生蛋派 (借用大站真实 TLS，免域名)
                    │    ├── VLESS + REALITY + Vision (TCP)
                    │    ├── VLESS + REALITY + XHTTP (TCP)
                    │    └── ShadowTLS v3 + Shadowsocks 2022 (TCP)
                    │
                    ├── 正规军仿生派 (自有域名证书 + 真实站点回落)
                    │    ├── NaiveProxy + Caddy / HAProxy (TCP/H2)
443 端口方案矩阵 ───┤    ├── VLESS + XHTTP + TLS + Fallback (TCP/H2)
                    │    ├── Trojan + TLS + 真实 Web Fallback (TCP)
                    │    └── AnyTLS + TLS + Padding (TCP)
                    │
                    ├── HTTP/3 与 QUIC 原生派 (UDP/443 专用)
                    │    ├── Hysteria 2 (原生 HTTP/3 伪装 + ECH)
                    │    └── TUIC v5 (QUIC 0-RTT 原生代理)
                    │
                    └── 边缘保护派 (云原生 CDN 中转)
                         └── VLESS / Trojan + XHTTP / WS + TLS (via Cloudflare/Fastly)

三、 全方案技术横向对比矩阵

评估标准延续严谨的多维指标：

  - P（被动识别防御）：指纹（JA3/JA4/ALPN）、首包特征、流控、TLS-in-TLS 双层特征清除能力。
  - A（主动探测防御）：遇到审查探测包（重放包、残缺包、探测脚本）时的自洽回落与响应能力。
  - R（抗封锁弹性）：面对端口 QoS、IP 信誉污染、临时 RST 注入的韧性。
  - E（性能与资源效率）：吞吐量、CPU/RAM 开销、系统调用优化（如 splice）、弱网抗丢包能力。
  - D（异构容灾价值）：与常规主流配置的架构差异，防止因同一特征被团灭。
  - T（公开证据等级）：实测论文、gfw.report 等公开权威研究的充分程度。

| 组合名称                               | 传输层 / 端口  | 核心伪装机制                                       | P：被动识别                 | A：主动探测            | R：抗封抗限               | E：性能效率               | D：异构价值          | T：公开证据                | 适用场景定位                     |
| :--------------------------------- | :-------- | :------------------------------------------- | :--------------------- | :---------------- | :------------------- | :------------------- | :-------------- | :-------------------- | :------------------------- |
| **VLESS + REALITY + Vision**       | TCP / 443 | uTLS模拟 + 借用大站TLS 1.3 + Vision消去内层TLS特征       | ★★★★☆                  | ★★★★★             | ★★★★☆                | ★★★★★ (splice零拷贝)    | ★★★☆☆ (当前极主流)   | ★★★★☆ (认知度极高)         | **443 端口的首选主力 TCP 线路**     |
| **VLESS + REALITY + XHTTP**        | TCP / 443 | REALITY 外层 + HTTP/2 流封装 + 报头填充/多路复用          | ★★★★★                  | ★★★★★             | ★★★★☆                | ★★★★☆ (H2封装开销)       | ★★★★☆ (传输层异构)   | ★★★☆☆ (新兴方案)          | **抗审查要求极高的 TCP 方案**        |
| **NaiveProxy + Caddy**             | TCP / 443 | 真实 Chromium 网络栈 + Caddy 前端 + 8轮双向 Padding    | ★★★★★ (绝对真Chrome)      | ★★★★★ (真正Caddy回落) | ★★★★★                | ★★★☆☆ (Chromium较吃资源) | ★★★★★ (独立协议栈)   | ★★★★☆ (长期抗审稳定)        | **最逼真、合规性最高的“防封天花板”**      |
| **VLESS + XHTTP + TLS + Fallback** | TCP / 443 | 自有证书 + 真实 Nginx/Caddy 回落 + 现代原生 HTTP 承载      | ★★★★☆                  | ★★★★★             | ★★★★☆                | ★★★★☆                | ★★★★☆           | ★★★☆☆ (WS替代方案)        | **自有域名+真实站点建站者的首选**        |
| **ShadowTLS v3 + SS2022**          | TCP / 443 | 真实大站 TLS 代理握手 + Challenge-Response 鉴权 + AEAD | ★★★★☆                  | ★★★★☆ (需防时序侧信道)   | ★★★★☆                | ★★★★½ (Rust高性能)      | ★★★★★ (非Xray体系) | ★★★★☆ (有专门检测论文)       | **高性能、异构的第二套 TCP 核心备用**    |
| **Trojan + TLS + Fallback**        | TCP / 443 | 真实 TLS + 密码鉴权 + 真实 Web 服务回落                  | ★★★☆☆ (易受TLS-in-TLS识别) | ★★★★★             | ★★★☆☆                | ★★★★½                | ★★★☆☆ (老牌方案)    | ★★★★☆ (已被深度研究)        | **低成本、传统稳定的建站附带代理**        |
| **AnyTLS + TLS**                   | TCP / 443 | 动态可升级 Padding Scheme + 会话复用                  | ★★★★☆                  | ★★★★☆ (依赖配置)      | ★★★★☆                | ★★★★☆                | ★★★★★ (全新协议)    | ★★☆☆☆ (公开长期数据少)       | **防特征封锁的实验型 TCP 备用**       |
| **Hysteria 2 (原生 HTTP/3)**         | UDP / 443 | 伪装合法 HTTP/3 站点 + ECH + Brutal/BBR 拥塞控制       | ★★★★☆ (防高熵封锁)          | ★★★★☆ (原生H3伪装)    | ★★★☆☆ (QUIC SNI封锁风险) | ★★★★★ (弱网吞吐之王)       | ★★★★★ (网络栈完全隔离) | ★★★★★ (2025 USENIX论文) | **443 端口的首席 UDP / 弱网专用通道** |
| **TUIC v5**                        | UDP / 443 | 原生标准 QUIC 实现 + 0-RTT                         | ★★★☆☆                  | ★★★☆☆             | ★★★☆☆ (QUIC SNI封锁风险) | ★★★★★                | ★★★★☆           | ★★★★☆                 | **轻量级标准 QUIC 替代方案**        |
| **VLESS + XHTTP + CDN**            | TCP / 443 | Cloudflare CDN 边缘反代 (443进，443出)              | ★★★☆☆ (CDN特征较固定)       | ★★★★★ (直接打到CDN)   | ★★★★★ (IP近乎不灭)       | ★★☆☆☆ (延迟翻倍/限速)      | ★★★★☆           | ★★★★★ (最成熟技术)         | **VPS IP 被墙后的最终“复活保底”手段**  |

四、 核心方案深度技术解析

1. VLESS + REALITY + Vision：443 端口上的“完全体形态”

  - 为什么 REALITY 在 443 是完全体，在非 443 是残血版？
      - REALITY 的原理是“借用（偷取）”境外权威大站（Dest，如
        gateway.icloud.com、www.microsoft.com、www.cloudflare.com）的真实证书和握手。这些权威大站在公网上只在 443
        端口开放 HTTPS。
      - 如果在 8443 等非标准端口部署 REALITY，审查系统只需发起一次简单的关联校验即可察觉异常：“目标 SNI 是
        microsoft.com，但请求发送到了 8443，而真正的微软服务器 8443 根本不提供该证书”。
      - 在 443 端口上，REALITY 真正做到了与真实大站的 IP
        外部行为完全一致：未经授权的主动探测会被透明转发给真实服务器，返回真实证书、协商真实的
        TLS 1.3，并给出真实的 HTTP 400/404 页面。
  - XTLS-Vision 的关键作用：
      - 当使用代理访问外部 HTTPS 网站时，外部流量本身也是 TLS（这称为 TLS-in-TLS）。普通的 TLS 代理会在外层 TLS
        记录中原封不动地打包内层 TLS ClientHello，这会在特定数据包（第 2~5 包）产生固定的长度和时序签名。
      - Vision 能够精准提取内层 TLS 的握手，动态调整外层填充（Padding），消除这一双层特征；在握手结束后直接启用 Linux 内核
        splice() 进行流传输，不仅隐蔽且吞吐极高。

2. NaiveProxy + Caddy：被动与主动防御的“工业级标杆”

  - 彻底根除指纹分歧（Zero-Fingerprint Gap）：
      - 无论是 uTLS 还是各类自建模拟器，对 Chrome 浏览器 TLS 指纹的模仿都是“黑盒模拟”，其扩展顺序、GREASE
        生成规律、HTTP/2 设置帧（Settings Frames）往往会在 Chromium 版本迭代中留下破绽。
      - NaiveProxy 的客户端直接提取了 Google Chromium 的真实网络协议栈，其产生的 TLS
        ClientHello、ALPN、TCP 窗口扩展、H2 优先级树与真实 Chrome 一模一样。
  - 8 轮双向填充（Application-Layer Padding）：
      - NaiveProxy 在建立 HTTP/2 CONNECT 隧道后，对前 8 次读写数据施加随机长度填充，彻底打碎了网页请求和 API
        通信在握手初期的固定包长分布。
  - 原生企业级站点回落：
      - 服务端作为 Caddy
        的一个插件运行。未授权的连接会直接作为普通静态文件请求或反代请求处理，审查者发起探测只会看到一个真实正常运行的英文博客、企业主页或静态
        API，且支持 probe_resistance 防撞库探测。
  - 代价：客户端必须随着 Chrome 的主版本同步更新以保持指纹新鲜；不支持内核零拷贝，CPU 占用高于 VLESS。

3. XHTTP（原 SplitHTTP）：现代 HTTP 传输的全面革新

  - 为什么全面推荐 XHTTP 替代传统的 WebSocket 和 gRPC？
      - WebSocket 的硬伤：由于其协议规范限制，TLS 握手时的 ALPN 强制为 http/1.1，随后必须经历明文文本格式的
        Upgrade: websocket 请求头交互。在 2026 年的现代互联网，绝大多数真实浏览器访问 HTTPS 网站的 ALPN 均为
        h2（HTTP/2）或 h3（HTTP/3）。443 端口上充斥着长期保持的长连接 http/1.1
        WebSocket，在审查系统的大数据统计模型中显得极其醒目。
      - gRPC 的硬伤：虽然基于 HTTP/2，但其固定带有 content-type: application/grpc 以及独特的 gRPC
        流控帧，且缺乏自然的多路径分流与回落机制。
      - XHTTP 的突破：
        1.  原生支持 HTTP/2 与 HTTP/3：协商标准现代 ALPN。
        2.  请求与响应分离（Asymmetric Transport）：支持上下行不同路径（如上传使用 chunked post，下载使用
            stream-up），规避审查系统对 HTTP 上下行流量对称性的统计分类器。
        3.  报头填充与随机化：避免了固定代理请求头模式，在兼容 CDN 的同时具备极高的反 DPI 性能。

4. UDP/443 的特殊考量：Hysteria 2 与 QUIC 的真实处境

  - UDP/443 的天然正当性：
      - 浏览器在连接支持 HTTP/3 的服务器时，会先通过 TCP/443 的 alt-svc: h3=":443" 获得通知，随后自动建立
        UDP/443 连接。
      - 因此，在 UDP/443 上运行 Hysteria 2 伪装成 HTTP/3，完全符合全球互联网的主流行为。
  - 重大风险警示：QUIC Initial 审查：
      - USENIX Security 2025 的权威论文证实：GFW 自 2024 年 4 月起部署了大规模的硬件 QUIC 解密模块。它利用
        QUIC 规范中公开的盐值（Initial Salt）实时解密客户端发出的 QUIC Initial 报文，提取其中内嵌的未加密 TLS
        ClientHello，并对里面的 SNI 执行独立于 TCP 的专用黑名单审查。一旦命中，该五元组将进入约 180 秒的持续丢包惩罚。
      - 在 443 端口使用 Salamander 混淆的潜在风险：
          - Salamander 将 QUIC 报文完全随机化为高熵数据。这在非 443 高端口（配合端口跳跃）是非常优秀的技术；但在
            UDP/443 端口上，正常世界里只有标准的 QUIC 流量。如果 DPI 在 UDP/443 上检测到完全无法解析为 QUIC
            报文格式的高熵加密数据包，反而会使其失去 HTTP/3 的保护伞，触发异常流量标记。
          - 建议：在 UDP/443 上，应优先使用 Hysteria 2 的默认 HTTP/3 伪装（Camouflage）配合合规域名及真实网站
            Masquerade，或尝试配置 ECH（Encrypted Client Hello）屏蔽明文 SNI。如果网络环境彻底阻断了
            QUIC，才考虑退守到端口跳跃+混淆模式。

五、 443 端口下的架构淘汰与替代路径

经过 2024–2026 年的攻防演化，443 端口的选型已发生清晰的技术代际更替：

1.  淘汰方案（不推荐在 443 继续新装）：
      - 裸 VMess / 裸 Shadowsocks / 裸 WireGuard on 443：高熵特征违反 443 协议标准，极易直接被黑洞封锁。
      - VMess + TLS + WebSocket：双层加密指纹明显，且 ALPN http/1.1 特征过时。
      - VLESS + gRPC + TLS：主动探测防护弱，中间件穿透及指纹抗性不如 XHTTP。
2.  当前技术代际（推荐采用）：
      - 无域名需求：VLESS + REALITY + Vision (TCP)。
      - 有域名+追求极限隐蔽：NaiveProxy + Caddy (TCP) 或 VLESS + XHTTP + TLS + Fallback
        (TCP)。
      - 弱网与低延迟加速：Hysteria 2 (HTTP/3 Masquerade on UDP/443)。
      - 异构容灾：ShadowTLS v3 + Shadowsocks-2022 (TCP)。

六、 443 端口最优工程实践方案落地指南

针对“在 443 端口构建高可用、高隐蔽、异构容灾代理”的目标，推荐以下三种架构搭配：

方案 A：单 IP 极简双栈王者架构（无需域名，主流标配）

  - TCP 443：VLESS + REALITY + Vision（Dest 指向 www.apple.com:443 或
    gateway.icloud.com:443）
  - UDP 443：Hysteria 2 + HTTP/3 伪装（Masquerade 同样配置对应站点）
  - 架构逻辑：
      - 充分利用单机单 IP 的双栈特性。TCP 走内核零拷贝 splice，拥有极低的 CPU 占用和超高吞吐，承载日常绝大多数并发与下载流量；
      - UDP 走 Hysteria 2 的 Brutal/BBR
        拥塞控制，在国际出口拥堵、高峰期丢包严重、或有实时音视频/游戏需求时自动路由，抗丢包能力极佳；
      - 客户端配置智能分流（如 sing-box / mihomo 的 url-test 或 fallback 策略），TCP 故障时无缝降级到
        UDP。

方案 B：正规化企业站点伪装架构（拥有自有域名，抗审查天花板）

  - 前端核心：Caddy 监听 TCP:443 和 UDP:443，配置正式的 Let's Encrypt 证书，部署一个真实的静态业务网站或技术博客。
  - TCP 分流：
      - 挂载 NaiveProxy 模块（Caddy forwardproxy），提供给主力工作机使用，享受纯正 Chromium
        网络栈带来的无指纹特征红利；
      - 或者后端通过反向代理将指定复杂 UUID 路径的请求转发给内网的 Xray (VLESS + XHTTP)。
  - UDP 分流：
      - Caddy 原生支持 HTTP/3，对外提供标准的 Web 页面浏览体验；内部将代理流量按需分发。
  - 防御效果：无论是 GFW 发起的深度主动探测，还是人工审查员直接通过浏览器访问 VPS 的 IP
    与域名，看到的都是一个格式完美、响应合规、拥有合法证书的现代 HTTPS/H3 站点，完全消除了作为“代理服务器”的行为差异。

方案 C：全异构立体容灾池（应对极端敏感期）

如果希望彻底避免“一处被封全盘皆输”，建议在节点池中形成跨协议栈、跨传输模式的对等隔离：

1.  主力 1（TCP/443 借用 TLS）：VLESS + REALITY + Vision（主攻日常大带宽）
2.  主力 2（UDP/443 原生 QUIC）：Hysteria 2（主攻弱网抗抖动）
3.  备用 1（TCP/443 真实仿生）：NaiveProxy（主攻审查高压期抗 DPI）
4.  备用 2（TCP/443 异构数据层）：ShadowTLS v3 + SS2022（脱离 VLESS 体系的备用 TCP 线路）
5.  保底 3（TCP/443 CDN 保护）：VLESS + XHTTP + TLS via Cloudflare（即使 VPS IP 被直接封锁，仍能通过
    CDN 边缘节点恢复低速连接用于维护系统）

总结结论

在 443 端口上部署代理，“加密”早已不是决定存活的核心指标，**“传输行为与全球正规 HTTPS/H3
流量的高度一致性”以及“面对主动探测时的自洽回落”**才是生命线所在。对于 443
端口，以 VLESS-REALITY（或 NaiveProxy）负责 TCP，以 Hysteria 2 负责 UDP 形成端口复用双栈，辅以 XHTTP 和
ShadowTLS 作为异构灾备，是兼顾隐蔽性、极限性能与抗审查韧性的最佳技术选择。


我按 **2026 年 9 月 18 日** 的状态重新梳理了一遍。这里把你的问题理解为：

> **公网服务器具备原生 IPv6，服务端直接或间接使用 443，主要面对中国大陆 GFW/DPI/主动探测，希望比较当前仍值得部署的代理协议、TLS/QUIC 传输组合、CDN/反代组合，以及它们的加密、流量伪装、抗主动探测、性能、IPv6 适配和截至目前已经暴露出的针对性打击。**

先说明一个重要结论：

**IPv6 本身并不等于“更难封”。** 近年的测量已经明确表明中国对 IPv6 流量也实施实质性的 DNS、TLS、HTTP 等审查；因此真正需要比较的是“IPv6 + 443 + 协议指纹”的组合，而不是把 IPv6 当作额外的一层混淆。([GFW Report][1])

另外，“所有”在严格意义上不可能穷举，因为大量协议有 fork、私有实现和客户端插件。下面覆盖的是 **2026 年仍有维护、生态仍能找到客户端、并且确实值得考虑的主要协议/组合**。

---

# 一、先看现在的总体格局

目前基本形成了四条路线：

| 路线            | 代表组合                                 | 传输      | 核心优势                          | 核心问题                 |
| ------------- | ------------------------------------ | ------- | ----------------------------- | -------------------- |
| **TLS 仿真型**   | VLESS + REALITY                      | TCP 443 | TLS 外观、主动探测防护、直连简单            | 已进入持续攻防阶段            |
| **Web 原生型**   | NaiveProxy + Caddy/HTTP2             | TCP 443 | 真正使用 Chromium 网络栈，HTTP/2 行为自然 | SNI/域名仍可被针对，生态较小     |
| **TLS 流量整形型** | AnyTLS / Restls / ShadowTLS + SS2022 | TCP 443 | 主动解决 TLS-in-TLS、长度和行为特征       | 生态、成熟度不如 VLESS/Naive |
| **QUIC 高性能型** | Hysteria2 / TUIC / Juicity           | UDP 443 | RTT、丢包、高并发、UDP 性能             | **QUIC 已被 GFW 专门分析** |

2025 年 USENIX Security 的实测已经确认，GFW 自 2024 年 4 月 7 日开始具备针对 QUIC 的专门审查能力，可以解密 QUIC Initial、读取其中的 SNI，并应用独立的启发式过滤逻辑；研究还专门观察到了对 Hysteria 和 MASQUE 类流量的影响。([GFW Report][2])

所以现在最大的误区之一就是：

> **“QUIC 加密了，所以 Hysteria2/TUIC 天然比 TCP 隐蔽。”**

不成立。

QUIC 的**内容加密**和 QUIC 的**传输可识别性**是两件事。

---

# 二、我建议把评价拆成 6 个维度

下面我用：

* **加密**：密码学保护本身
* **伪装**：被动 DPI 看起来像普通流量的程度
* **主动探测**：别人主动访问服务器以后，能不能判断“这里有代理”
* **性能**：延迟、吞吐、多路复用、丢包环境
* **生态**：客户端/服务端成熟度
* **2026 风险**：目前公开研究和实践暴露出来的针对性问题

这里的高/中/低是综合判断，不是某个实验室的统一评分。

---

# 三、第一梯队：VLESS + REALITY

## 1. VLESS + REALITY + TCP 443

这是目前最典型的 IPv6/443 组合。

结构可以概念化成：

```text
IPv6
  │
TCP/443
  │
TLS/REALITY
  │
VLESS
  │
代理数据
```

REALITY 本身不是一个代理协议，而是 Xray 的 TLS 仿真层。

Xray 官方文档目前明确列出 REALITY 可以和：

* RAW
* XHTTP
* gRPC

组合，而且它通过借用目标站点的 TLS 外观来伪装；官方也把它定位为目前比较强的传输安全方案。([GitHub][3])

### VLESS + REALITY + RAW + Vision

这是传统而成熟的形态。

优点非常明确：

**1. TCP 443**

因此在 IPv6 环境下就是最普通的：

```text
AAAA → TCP/443 → TLS
```

没有 UDP/QUIC 的额外特征。

**2. REALITY 不是简单的“自己签证书”**

它的核心价值就在于服务器对外表现出目标 TLS 站点的特征，而不是：

```text
自己的域名
自己的证书
自己的 TLS fingerprint
```

这种组合对于主动探测尤其有意义。Xray 的官方文档明确说明 REALITY 的目标就是使外部流量形态与正常 Web 流量一致。([GitHub][4])

**3. Vision 可以减少代理流量自身的额外特征**

Xray 官方文档称合适的 XTLS Vision flow-control 可以改善性能。([GitHub][3])

### 主要问题

REALITY 并不是“永久不可识别”。

2026 年出现了越来越多针对 REALITY/XHTTP 的现场报告。其中 2026 年 7 月的一份 Xray issue 记录了一个很具体的中国侧事件：

* TCP/443
* VLESS + REALITY + XHTTP
* 服务端仍然正常
* 海外可以正常连接
* 中国侧突然无法完成 TCP 三次握手
* 同时期服务器出现大量陌生 ClientHello / SNI
* 作者怀疑发生主动探测/IP 污染后的阻断

这是**现场报告，不等于已经被证明是 GFW 对 REALITY 的完整协议级破解**，但它说明这种组合并不是理论上“只要配了就万事大吉”。([GitHub][5])

### 综合判断

| 项目      | 判断        |
| ------- | --------- |
| IPv6    | ★★★★★     |
| TCP/443 | ★★★★★     |
| 加密      | ★★★★★     |
| 被动伪装    | ★★★★★     |
| 主动探测抗性  | ★★★★★     |
| 性能      | ★★★★★     |
| UDP     | 不适合       |
| 2026 风险 | 中高，进入持续攻防 |

**它仍然是目前 TCP/443 体系中非常重要的方案，但不应再理解为“已经彻底解决 GFW”。**

---

# 四、VLESS + XHTTP：2026 年值得重点关注，但要区别于 Vision

这是我认为目前最值得认真研究的一条路线。

XHTTP 的目标实际上不是简单“再套一层 HTTP”。

它在解决的是：

> **怎样让一个代理连接更像正常 HTTP/Web 流量，同时把上行、下行、复用和 CDN/反代兼容性做好。**

Xray 当前文档明确支持：

```text
VLESS + XHTTP + TLS
VLESS + XHTTP + REALITY
```

而且官方文档现在已经明确要求逐渐迁移到 VLESS Encryption 等新机制。([GitHub][6])

---

## 2. VLESS + XHTTP + REALITY + TCP 443

这是目前很强的一套组合，但是我不会把它简单理解成“Vision 的完全升级版”。

XHTTP 自己已经解决了：

* 多路复用
* HTTP 风格传输
* 上下行模式
* CDN/反代适配

官方社区文档也明确提到 XHTTP 默认多路复用，而且它就是为 CDN/反代场景设计的。([GitHub][7])

### 性能

XHTTP 对大量短连接、小连接场景有优势，因为多个代理 stream 可以复用一个外层连接。

但有一个有意思的区别：

Xray 社区资料显示：

> XHTTP 多路复用后，延迟可能很好，但多线程测速不一定比 Vision 好。

因此：

```text
网页 / API / SSH / 大量小连接
```

XHTTP 很有吸引力。

而：

```text
单线程大文件
极限测速
```

Vision/传统 TCP 未必会输。([GitHub][7])

### 当前风险

这里是整个研究里非常值得注意的一点。

2026 年 7 月已经出现：

> VLESS + REALITY + XHTTP 在中国方向疑似遭遇主动探测/握手阻断。

另外还有一个 2026 年的 Xray issue 报告，在 VLESS + REALITY + XHTTP 使用中出现极大量 TCP 连接和文件描述符增长，最终达到约 1 GB 级内存占用的现象。该问题是具体实现问题报告，不代表 XHTTP 本身一定存在普遍性泄漏，但说明当前实现仍在高速迭代。([GitHub][5])

所以它是：

> **技术方向先进，但是截至 2026 年 9 月仍属于“需要跟着 core 版本演进”的方案。**

---

# 五、VLESS + XHTTP + TLS + CDN

这个组合和 REALITY 直连完全不是一回事。

结构：

```text
IPv6
 ↓
CDN IPv6 edge :443
 ↓
TLS
 ↓
HTTP/XHTTP
 ↓
origin
 ↓
VLESS
```

它真正的优势是：

### 服务器 IP 不一定直接暴露

这在 IP reputation / VPS IP 被污染时非常有意义。

而且 XHTTP 官方生态本身就在围绕 CDN / reverse proxy 设计。([GitHub][7])

---

## 它的问题

CDN 不是“免费隐身”。

因为：

```text
用户
 ↓
CDN
 ↓
代理服务器
```

中间多了一层：

* CDN 能看到连接元数据
* 域名仍然存在
* SNI/HTTP 语义仍然存在
* CDN 对 WebSocket/gRPC/H2/H3 的行为可能不同
* 长连接可能被回收
* 非所有 CDN 都适合代理任意代理协议

所以它不是“比 REALITY 无条件更强”，而是：

> **把风险从“服务器本身可识别”转移到“CDN 前端 + Web 语义 + 域名”。**

---

# 六、NaiveProxy：一个完全不同的思路

如果把：

```text
VLESS + REALITY
```

理解成：

> 模拟 TLS

那么：

```text
NaiveProxy
```

更接近：

> **直接使用 Chrome/Chromium 的网络栈来做代理。**

官方 README 明确列出的目标包括：

* HTTP/2 multiplexing
* Chrome TLS fingerprint
* application fronting
* padding / fragmentation

即：

> 不是“我要伪装成 Chrome”，而是**尽可能复用 Chrome 本身的网络实现**。([GitHub][8])

---

# 七、NaiveProxy + Caddy + HTTP/2 + TCP 443

这是非常有价值的一组。

它的特点：

```text
IPv6
  ↓
TCP/443
  ↓
TLS
  ↓
HTTP/2
  ↓
Caddy forward proxy
  ↓
Naive
```

Naive 官方设计甚至直接把：

```text
Browser → Naive client
     ↓
Censor
     ↓
Frontend
     ↓
Naive server
```

作为架构模型。

而且前端服务器可以借助正常 HTTPS frontend 来抵抗主动探测。([GitHub][8])

### 这是它真正强的地方

它不像传统：

```text
TLS
 ↓
一个代理协议
```

而是：

```text
真正的 Chrome 网络协议栈
+
HTTP/2
+
正常 HTTPS frontend
```

所以在“Web 流量仿真”这件事上，它的思路和 REALITY 完全不同。

---

## Naive 的问题

第一，**SNI 还是在那里**。

2025 年已有社区报告某些中国移动网络直接针对 Naive 的 SNI 进行阻断，而同服务器的 REALITY IP 直连仍然可以使用。这个案例是社区 issue，不是系统性测量。([GitHub][9])

第二，性能并不是它唯一的优势。

Naive 官方设计强调安全性和隐蔽性，但实际 issue 中确实存在部分平台性能异常的报告。([GitHub][10])

第三，它的 HTTP/3/QUIC 路线没有必要作为主线。

Naive 自身历史文档就承认 QUIC/HTTP3 在实际环境中可能受到 carrier throttling / congestion-control fairness 影响；当前 GFW 又已经具备 QUIC 专门审查能力。([GitHub][11])

因此：

> **Naive 更值得用 HTTP/2/TCP 443，而不是为了追求“更现代”主动换 QUIC。**

---

# 八、AnyTLS：2025–2026 很值得关注

AnyTLS 是这一批新协议里一个很有意思的设计。

它解决的是一个老问题：

> **TLS 里面再塞一个代理 TLS/加密流量，会不会出现 TLS-in-TLS 特征？**

AnyTLS 的设计直接包含：

* Session multiplexing
* padding
* packet length control
* TLS transport
* reusable sessions

官方协议文档就是明确针对：

> “嵌套 TLS 握手指纹”

进行设计的。([GitHub][12])

---

## AnyTLS + TLS + TCP 443

特点：

| 项目         | 表现   |
| ---------- | ---- |
| IPv6       | 很好   |
| TCP/443    | 很好   |
| 加密         | TLS  |
| TLS-in-TLS | 专门针对 |
| 多路复用       | 强    |
| CPU        | 中    |
| 配置复杂度      | 中    |
| 生态         | 中    |
| 当前成熟度      | 中    |

而且现在：

* sing-box 已集成
* mihomo 已集成
* Shadowrocket 2.2.65+ 已实现客户端

AnyTLS 官方 README 已明确列出这些兼容实现。([GitHub][13])

---

## 但是有一个关键问题

AnyTLS 自己的 FAQ 很诚实：

> 它不能保证当前 padding 参数永远不会被墙识别。

因此 AnyTLS 的思路不是：

```text
做一个永远不变的完美协议
```

而是：

```text
把流量特征参数做成可改变
```

这其实是很重要的设计哲学。

([GitHub][14])

所以我会把它归入：

> **很有前景的新一代 TCP/TLS 方案，但目前不宜把它当成比成熟 REALITY 稳定很多的“终极替代品”。**

---

# 九、Restls：技术上非常有意思，但生态明显小

Restls 直接针对：

> **TLS-in-TLS + TLS fingerprint + TLS 后行为**

三个问题。

它使用：

* uTLS/browser fingerprint
* TLS server impersonation
* Restls-Script
* packet length shaping

尤其是 Restls-Script，可以人为定义：

```text
第 1 个包多大
第 2 个包多大
什么时候要求对端回应
padding 多大
```

用来破坏普通代理中很容易出现的：

```text
ClientHello
→
巨大的 ServerHello
→
固定大小的数据
```

这种代理指纹。([GitHub][15])

这在理论上非常漂亮。

但它的问题也很明显：

> **生态太小。**

目前主要集中在 Restls 自己以及 Clash.Meta 的相关 fork。([GitHub][16])

因此：

**研究价值很高，生产通用价值暂时低于 VLESS/Naive/AnyTLS。**

---

# 十、Shadowsocks 2022

Shadowsocks 本身仍然没有消失，而且 AEAD 2022 目前仍然是正式维护中的实现。

shadowsocks-rust 当前支持：

```text
2022-blake3-aes-128-gcm
2022-blake3-aes-256-gcm
2022-blake3-chacha20-poly1305
2022-blake3-chacha8-poly1305
```

同时也保留传统 AEAD。([GitHub][17])

---

## SS2022 + TCP 443

它最大的优势是：

* 加密成熟
* CPU 成本低
* 实现多
* 客户端丰富
* 非常简单

但如果直接：

```text
SS2022 → TCP/443
```

我不会把它放进最强的“隐身”方案里。

原因是：

> **加密不等于 Web camouflage。**

Xray 当前文档甚至明确把 Shadowsocks 与 VMess 的“协议层加密”和 TLS/REALITY 的“HTTPS 外观”区分开来，并指出前者仍可能被流量分类。([GitHub][18])

---

# 十一、Shadowsocks 2022 + ShadowTLS V3

这个组合就有意思得多：

```text
TCP/443
 ↓
ShadowTLS V3
 ↓
SS2022
```

ShadowTLS 本身不是加密代理，它的目标是：

> 给防火墙看到一个真实 TLS server。

ShadowTLS V3 又进一步增加：

* TLS traffic encapsulation
* server authentication
* client authentication
* anti replay
* anti active detection
* anti hijacking

官方协议文档明确这样定义。([GitHub][19])

当前项目仍有 release，最新公开版本已经支持 V3。([GitHub][20])

### 但是

它属于：

```text
TLS 外层
+
代理加密内层
```

所以理论上仍存在：

> nested TLS / TLS-in-TLS behavior

这种长期类别的问题。

这也是为什么：

```text
ShadowTLS
Restls
AnyTLS
REALITY
```

这些项目虽然目的相似，但设计方向并不一样。

---

# 十二、Mieru：现在反而值得重新看

以前很多人会忽略 Mieru。

但截至 **2026 年 9 月 13 日**，Mieru 已经发布到 **3.37.0**，而且 2026 年连续加入了：

* heartbeat jitter
* padding traffic pattern
* low-entropy mode
* CPU/RAM 优化

说明项目现在不是停滞状态。([GitHub][21])

Mieru 当前协议有一个非常特殊的特点：

> **它根本不使用 TLS。**

官方明确说明：

* TCP / UDP
* XChaCha20-Poly1305
* random padding
* replay protection
* IPv4 / IPv6

([GitHub][22])

---

## Mieru + TCP 443

对于 IPv6：

```text
IPv6
 ↓
TCP/443
 ↓
Mieru
```

技术上完全没问题。

它甚至可以明确配置：

```text
ONLY_IPv6
PREFER_IPv6
```

等地址策略。([GitHub][23])

### 它的真正问题

它没有：

```text
TLS
HTTP
HTTPS
Chrome
QUIC
```

这些自然协议外观。

所以：

> **它是“加密 + 流量统计/长度混淆”路线，而不是“伪装成 HTTPS”路线。**

Mieru 项目自己过去也承认，原始协议曾经遇到过中国网络封锁，需要持续改进。([GitHub][24])

2026 年又在不断加入低熵、padding、heartbeat jitter 等机制，这恰恰说明它采用的是动态对抗型思路。

---

# 十三、Hysteria 2：性能王者，但不能再把它等同于“最隐蔽”

Hysteria 2：

```text
IPv6
 ↓
UDP/443
 ↓
QUIC
 ↓
Hysteria2
```

这是完全不同的世界。

---

## 性能

Hysteria 的核心卖点：

* QUIC
* UDP
* 用户态拥塞控制
* Brutal
* 多路复用
* 丢包环境下保持高吞吐

这是它最强的地方。

所以：

```text
跨洲
高 RTT
高丢包
网络拥堵
移动网络
```

经常是它比传统 TCP 更有优势的场景。

---

# 十四、但 2024–2026 年 QUIC 的安全环境发生了根本变化

这是整个研究里最重要的一条时间线：

### 2024-04-07

GFW 开始针对 QUIC 的 SNI 进行审查。

### 2025

USENIX Security 论文确认：

> GFW 可以解密 QUIC Initial。

而不是简单地：

```text
UDP
+
看不懂
+
不管
```

它已经能够在规模化条件下解析 QUIC Initial 并使用 SNI/启发式规则过滤。([GFW Report][2])

因此：

> **“QUIC 加密”不能作为 Hysteria2 抗审查能力的主要依据。**

---

# 十五、Hysteria2 2026 年其实正在主动升级

这也很值得注意。

目前新版 Hysteria2 / sing-box 已经加入：

* Chrome QUIC fingerprint parroting
* BBR profile
* hop interval randomization
* Gecko obfuscation

官方 Hysteria 2.11/2.12 更新已经直接加入 Chrome QUIC fingerprint imitation。([GitHub][25])

sing-box 1.14 又加入：

```text
Chrome QUIC fingerprint
BBR profile
hop randomization
Gecko obfs
```

([Sing Box][26])

这说明：

> **Hysteria 开发者已经认识到“QUIC 原生指纹”本身就是问题。**

---

# 十六、Hysteria2 的两类混淆不要混为一谈

### 原生 Hysteria2

```text
QUIC
+
HTTP/3 appearance
```

官方协议说明本身就是：

> 未授权客户端看起来像普通 HTTP/3 server。

([GitHub][27])

### Salamander

把 QUIC packet 整体扰乱成随机数据。

优点：

> 可以规避针对标准 QUIC/HTTP3 的直接识别。

缺点：

> 它反而不再像正常 QUIC。

### Gecko

2026 新加入的思路：

> 在 QUIC handshake 上进一步做随机 fragment / padding。

官方文档明确描述了这一点。([GitHub][28])

所以这里存在典型矛盾：

```text
越像 QUIC
→ 越自然
→ 越容易进入 QUIC DPI

越不像 QUIC
→ 越难被 QUIC 规则命中
→ 但“随机 UDP”本身又可能成为特征
```

---

# 十七、Hysteria2 + IPv6 的一个现实问题：MTU

这一点尤其和你当前使用 IPv6 的场景有关。

2026 年 8 月的一个 Hysteria issue 就出现了：

> Chrome QUIC fingerprint + BBR + 小 MTU

导致 PMTU 更新触发 BBR panic。

之后 Hysteria 2.12 已经修复。([GitHub][29])

这说明：

**IPv6 + QUIC + 复杂 tunnel 的 MTU 路径问题值得特别重视。**

IPv6 并不会自动帮你解决 PMTU。

---

# 十八、TUIC v5

TUIC 本质上也是：

```text
UDP/443
+
QUIC
+
TLS 1.3
```

它和 Hysteria2 最大的区别是：

> 更倾向于标准 QUIC / 标准拥塞控制，而不是 Hysteria 的 Brutal。

sing-box 当前支持：

* cubic
* new_reno
* bbr

以及原生 UDP relay。([GitHub][30])

TUIC v5 还包含：

* 0-RTT TCP
* UDP fragmentation
* TLS exporter authentication
* 多用户
* heartbeat 等

([GitHub][31])

---

## 性能

它通常属于：

> **高性能、低延迟、QUIC 原生化**

路线。

相对 Hysteria2：

```text
Hysteria2
= 更激进

TUIC
= 更接近标准 QUIC
```

因此在某些运营商的 QoS 环境下，TUIC 可能反而表现得更平滑。

但是：

> **抗 GFW 的“协议隐蔽性”并不会因为它是 TUIC 就自动增强。**

它仍然继承 QUIC 这条攻击面。GFW 已经有专门 QUIC SNI 审查体系。([GFW Report][2])

---

# 十九、Juicity

Juicity 同属 QUIC 体系。

它的一个特点是协议设计比较简洁，而且协议层明确原生支持：

```text
IPv4
IPv6
DOMAIN
TCP
UDP
```

([GitHub][32])

所以从：

```text
IPv6 + UDP/443
```

角度，它是完全合格的。

但是从今天的生态来看：

```text
Hysteria2
TUIC
```

明显更主流。

Juicity 更适合作为：

> **QUIC 备用协议**

而不是核心方案。

---

# 二十、WireGuard

如果只看：

```text
性能
加密
IPv6
稳定性
```

WireGuard 非常强。

但是如果加上：

```text
中国大陆 GFW
主动探测
协议识别
```

情况会完全不同。

WireGuard 的标准握手长度和报文格式高度固定。WireGuard 官方实现中：

```text
Handshake Initiation = 148 bytes
Handshake Response = 92 bytes
```

并且消息类型固定。([GitHub][33])

sing-box 当前文档甚至直接指出：

> WireGuard 加密本身足够强，但固定 UDP 特征很容易被识别/封锁，因此不适合直接作为 GFW circumvention transport。([GitHub][18])

---

# 二十一、AmneziaWG

这才是 WireGuard 在“抗识别”方向上的真正升级。

2026 年的 AmneziaWG 已经发展到新的混淆层，包括：

* junk packets
* header protection
* packet size changes
* handshake sequence changes
* mimicry
* timing changes

官方文档明确称其目标就是降低 WireGuard 的 DPI 可识别性。([Amnezia 文档][34])

所以：

```text
WireGuard
```

和：

```text
AmneziaWG
```

绝对不能当成同一级别。

不过需要注意：

> AmneziaWG 的目标是“隐藏 WireGuard 特征”，而不是“变成正常 HTTPS”。

所以它与：

```text
REALITY
Naive
AnyTLS
```

属于不同设计哲学。

---

# 二十二、现在最值得比较的核心组合

下面这张表可以作为你真正的选型表。

| 组合                               | 外层      |  IPv6 |    伪装 |  主动探测 |    性能 | 当前主要问题              |
| -------------------------------- | ------- | ----: | ----: | ----: | ----: | ------------------- |
| **VLESS + REALITY + Vision**     | TCP 443 | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | 进入持续针对阶段            |
| **VLESS + XHTTP + REALITY**      | TCP 443 | ★★★★★ | ★★★★★ | ★★★★☆ | ★★★★☆ | 2026 年已有现场阻断/实现问题报告 |
| **VLESS + XHTTP + TLS + CDN**    | TCP 443 | ★★★★★ | ★★★★★ | ★★★★☆ | ★★★★☆ | CDN/SNI/平台依赖        |
| **NaiveProxy + HTTP/2**          | TCP 443 | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★☆ | SNI、生态规模            |
| **AnyTLS + TLS**                 | TCP 443 | ★★★★★ | ★★★★☆ | ★★★★☆ | ★★★★☆ | 新协议、生态仍成长           |
| **Restls + SS**                  | TCP 443 | ★★★★☆ | ★★★★★ | ★★★★☆ | ★★★☆☆ | 生态太小                |
| **ShadowTLS V3 + SS2022**        | TCP 443 | ★★★★★ | ★★★★☆ | ★★★★☆ | ★★★★☆ | 多层 TLS 行为           |
| **Mieru TCP**                    | TCP 443 | ★★★★★ | ★★★★☆ | ★★★★☆ | ★★★★☆ | 非 Web 型、存在历史封锁记录    |
| **Trojan + TLS**                 | TCP 443 | ★★★★★ | ★★★☆☆ | ★★★☆☆ | ★★★★★ | 太经典，长期特征研究充分        |
| **SS2022 直连**                    | TCP/443 | ★★★★★ | ★★☆☆☆ | ★★☆☆☆ | ★★★★★ | 加密强≠Web 隐蔽          |
| **Hysteria2**                    | UDP 443 | ★★★★★ | ★★★☆☆ | ★★★★☆ | ★★★★★ | QUIC 被专门审查          |
| **Hysteria2 + Gecko/Salamander** | UDP 443 | ★★★★★ | ★★★★☆ | ★★★★☆ | ★★★★☆ | 随机/非标准 UDP 也有代价     |
| **TUIC v5**                      | UDP 443 | ★★★★★ | ★★★☆☆ | ★★★☆☆ | ★★★★★ | QUIC 指纹问题           |
| **Juicity**                      | UDP 443 | ★★★★★ | ★★★☆☆ | ★★★☆☆ | ★★★★★ | 生态小                 |
| **WireGuard**                    | UDP 443 | ★★★★★ | ★☆☆☆☆ | ★☆☆☆☆ | ★★★★★ | 固定特征非常明显            |
| **AmneziaWG**                    | UDP 443 | ★★★★★ | ★★★★☆ | ★★★☆☆ | ★★★★★ | 仍不像正常 HTTPS         |

这不是单纯速度 benchmark，而是根据协议结构、官方设计文档和当前公开研究做的综合判断。QUIC 项目的隐蔽性尤其需要扣分，因为 GFW 已有专门的 QUIC Initial/SNI 审查机制。([GFW Report][2])

---

# 二十三、如果把它们按“设计哲学”重新分类，会更清楚

## A. “看起来就是 HTTPS”

### VLESS + REALITY

核心思想：

```text
我不是代理
我是另一个真实 TLS server
```

### NaiveProxy

核心思想：

```text
我就是 Chrome / HTTP2
```

### Restls

核心思想：

```text
我不仅像 TLS
连 TLS 后面的行为都尽可能模拟
```

### AnyTLS

核心思想：

```text
我用 TLS
但把 TLS-in-TLS 特征主动拆掉
```

---

# 二十四、B. “我不需要像 HTTPS，我只要让 DPI 难分类”

### Mieru

```text
高强度加密
+
padding
+
low entropy
+
heartbeat jitter
```

### SS2022

```text
高速 AEAD
+
简单实现
```

### AmneziaWG

```text
WireGuard
+
transport obfuscation
```

这类方案不依赖“伪装成某个大网站”。

---

# 二十五、C. “我的第一目标是性能”

### Hysteria2

```text
QUIC
+
Brutal / BBR
```

### TUIC

```text
QUIC
+
标准 congestion control
+
UDP
```

### WireGuard / AmneziaWG

```text
kernel/user-space VPN
+
非常低开销
```

---

# 二十六、当前 GFW 到底打到了什么程度？

这个问题必须分层看。

---

## 第一层：IP / 服务器信誉

这个最现实。

一个节点即便协议完全没有问题：

```text
IP
↓
大量代理流量
↓
主动探测
↓
被标记
↓
服务器 IP 被限制
```

协议本身并不能解决这个问题。

---

## 第二层：SNI

这是最传统的一层。

例如：

```text
my-proxy.example.com
```

如果域名/SNI 本身被处理：

```text
TLS 伪装
=
依然可能被域名侧打
```

所以：

> “TLS 加密”不能隐藏 SNI 本身。

---

## 第三层：TLS fingerprint

GFW 历史上已经被研究证明可以基于 TLS 指纹识别代理，而不是只看域名。([GFW Report][35])

因此：

```text
自己生成一套奇怪 TLS
```

不是好的长期方案。

这也是：

```text
REALITY
Naive
uTLS
Restls
```

长期存在的根本原因。

---

# 二十七、第四层：TLS-in-TLS

这是 2022 年之后整个代理协议设计出现重大转向的原因之一。

简单说：

正常浏览器：

```text
TLS
└── HTTP
```

传统代理：

```text
TLS
└── TLS
    └── HTTP
```

第二种虽然“加密得更厉害”，但统计上反而可能更容易识别。

因此 AnyTLS / Restls / ShadowTLS / REALITY 都在不同程度上尝试解决这个问题。([GitHub][36])

---

# 二十八、第五层：QUIC / UDP

这是 2024–2026 最大的变化。

过去：

```text
UDP
=
很多时候 GFW 不愿意分析
```

现在不能这么看。

2025 年的研究明确证明 GFW 已经可以：

```text
QUIC Initial
↓
解密
↓
读取 SNI
↓
启发式过滤
```

而且该系统可以被利用来对跨境 UDP 做更广泛的阻断。([GFW Report][2])

这意味着：

> **QUIC 已经从“天然有利于翻墙”变成了“高性能，但拥有专门审查面”的传输。**

---

# 二十九、443 也不能神化

2025 年 8 月 20 日曾发生过一次非常特殊的事故：

中国方向的 **TCP/443** 在大约 74 分钟内被大规模无条件注入 RST+ACK。

报告观察到：

* 443 被影响
* 22/80/8443 未出现相同现象
* 出入境均受到影响

这虽然是异常事件，不代表 GFW 平时会永久封 443，但它很好地证明：

> **“443 = 永远不会因为端口本身被打”是错误的假设。**

([GFW Report][37])

---

# 三十、IPv6 对这些协议真正意味着什么？

假设服务器：

```text
AAAA = 2001:db8::1234
TCP/443
```

那么：

### VLESS/REALITY

非常适合。

### Naive

非常适合。

### AnyTLS

非常适合。

### Mieru

官方明确支持 IPv6。([GitHub][22])

### Hysteria2/TUIC

同样适合，但：

```text
IPv6
+
UDP
+
QUIC
```

对：

* MTU
* PMTU discovery
* NAT/防火墙
* 移动运营商

更加敏感。

---

# 三十一、IPv6 还有一个特别容易踩的坑

很多人会：

```text
AAAA
↓
IPv6
↓
TCP/443
```

然后认为：

> “IPv6 直连应该很干净。”

但实际客户端如果：

```text
IPv4
AAAA
```

双栈同时存在，很多系统会采用 Happy Eyeballs。

所以最终可能出现：

```text
客户端
 ├── IPv4 → VLESS
 └── IPv6 → VLESS
```

谁先成功就走谁。

因此如果你的研究目标就是：

> **专门测试 IPv6 这条链路**

不要只在 DNS 上增加 AAAA。

还要：

```text
确认客户端实际连接的是 IPv6
确认服务器监听 [::]:443
确认防火墙允许 IPv6 TCP/443
确认回程 IPv6
确认代理上游也不会因为 IPv6 出现异常
```

---

# 三十二、Cloudflare/CDN 能怎么组合？

这里必须把协议分成两类。

## 最适合普通 CDN

### VLESS + XHTTP + TLS

非常适合。

### VLESS + WebSocket + TLS

老牌方案，生态非常成熟。

### VLESS + gRPC + TLS

HTTP/2/CDN 生态成熟。

### Naive + HTTP frontend

可以做，但不像 XHTTP 那么天然适合普通 CDN。

---

## 不适合普通 Cloudflare Orange Cloud 的

### Hysteria2

UDP/QUIC。

### TUIC

UDP/QUIC。

### Juicity

UDP/QUIC。

### WireGuard

UDP。

### AmneziaWG

UDP。

因为普通 Web CDN 的反代层主要解决的是：

```text
HTTP / HTTPS
```

不是任意 UDP。

除非使用专门的 L4/UDP proxy 产品，而那已经不是普通免费 CDN 的概念。

---

# 三十三、如果要做“多协议冗余”，现在最合理的是分散风险，而不是堆套娃

我不建议：

```text
VLESS
 ↓
TLS
 ↓
WS
 ↓
gRPC
 ↓
CDN
 ↓
ShadowTLS
```

这种“越套越安全”的思路。

很多时候反而会制造：

```text
TLS-in-TLS
HTTP-in-HTTP
奇怪长度模式
多层连接复用
```

最终形成新的指纹。

更合理的是**平行部署不同传输特征**：

```text
                 ┌─ VLESS + REALITY / TCP 443
IPv6 Server ─────┼─ NaiveProxy / HTTP2 443
                 ├─ AnyTLS / TLS 443
                 └─ Hysteria2 / QUIC 443
```

也就是：

> **不是把 4 个协议串起来，而是让 4 个协议拥有完全不同的失败模式。**

这对于长期稳定性比“叠一堆混淆层”更加合理。

---

# 三十四、我会把 2026 年的组合分成这几个层级

这不是“谁最好”的总排名，而是不同使用目标下的技术定位。

### 第一类：TCP/443 隐蔽性路线

```text
VLESS + REALITY + Vision
VLESS + XHTTP + REALITY
NaiveProxy + HTTP/2
AnyTLS + TLS
```

它们共同特点：

> TCP + TLS/Web 生态。

---

### 第二类：TCP/443 实验/特殊路线

```text
Restls + SS
ShadowTLS V3 + SS2022
Mieru TCP
```

其中：

* Restls：流量行为控制最有意思
* ShadowTLS：真实 TLS fronting 思路成熟
* Mieru：完全不同的非 TLS 设计

---

### 第三类：UDP/443 性能路线

```text
Hysteria2
Hysteria2 + Chrome Parrot
Hysteria2 + Gecko
TUIC v5
Juicity
```

核心优势：

> 性能。

核心弱点：

> QUIC 已经成为 GFW 的专项分析对象。([GFW Report][2])

---

### 第四类：VPN 路线

```text
WireGuard
AmneziaWG
```

核心优势：

> 性能/全局代理。

核心弱点：

> WireGuard 原生指纹非常明显；AmneziaWG 虽然明显改善，但仍不是“正常 HTTPS”。([Amnezia 文档][34])

---

# 三十五、目前我认为最值得持续关注的技术趋势

真正值得注意的不是某一个协议，而是下面五个方向。

## 1. TLS mimicry → browser mimicry

从：

```text
自己模拟 TLS
```

逐渐变成：

```text
直接复用 Chrome/浏览器网络栈
```

Naive 是代表。([GitHub][8])

---

## 2. 固定 padding → 可变 traffic scripting

Restls / AnyTLS 的共同方向就是：

> 不让一个代理拥有固定流量形状。

([GitHub][36])

---

## 3. 标准 QUIC → QUIC fingerprint mimicry

Hysteria 2 现在已经开始：

```text
QUIC
 ↓
Chrome QUIC fingerprint
```

甚至进一步出现：

```text
Gecko
```

这是非常明确的军备竞赛方向。([GitHub][28])

---

## 4. “协议隐身” → “IP 风险管理”

随着 DPI 越来越强，单纯研究：

```text
哪一个协议更隐蔽
```

的重要性正在下降。

真正的可用性越来越像：

```text
协议指纹
+
IP reputation
+
域名 reputation
+
SNI
+
连接行为
+
使用模式
```

---

## 5. 从单协议到异构冗余

未来更合理的架构不是：

```text
一个神协议
```

而是：

```text
TCP stealth
+
HTTP-native
+
QUIC fast path
```

三套完全不同的 transport。

---

# 三十六、如果专门针对“IPv6 + 443”做架构，我会这样理解

一个比较现代的服务器可以逻辑上提供：

```text
                  IPv6
                    │
                 TCP/443
                    │
          ┌─────────┼─────────┐
          │         │         │
       REALITY    Naive     AnyTLS
          │         │         │
        VLESS       │         │
                    │         │
               HTTP/2        │
                              │
                         TLS session
```

以及：

```text
                 IPv6
                   │
                UDP/443
                   │
          ┌────────┼────────┐
          │        │        │
       Hysteria2   TUIC   Juicity
```

这里真正有价值的地方不是“三个协议同时跑”。

而是：

> **让 TCP 路线和 UDP 路线具有不同的封锁触发条件。**

例如：

```text
TCP/443 不通
        ↓
QUIC/443 仍可能可用

QUIC/443 被 QoS
        ↓
TCP/443 继续工作
```

这是异构冗余的真正价值。

---

# 三十七、最后给你一个我认为比较准确的 2026 状态图

```text
                    2026 Proxy Landscape
                            │
          ┌─────────────────┴─────────────────┐
          │                                   │
       TCP 443                              UDP 443
          │                                   │
   ┌──────┼────────┐                   ┌──────┼───────┐
   │      │        │                   │      │       │
REALITY  Naive   AnyTLS             Hysteria TUIC   AWG
   │      │        │                   │      │
   │      │        └──────┐            │      │
   │      │               │            │      │
   │      └──── Web       Restls       QUIC   QUIC
   │              native     │          │      │
   │                         │          │      │
 TLS mimic              traffic-shaping │      │
   │                                    │      │
   └───────── 被动 DPI / 主动探测 ───────┘
```

而底下又存在一条贯穿所有协议的风险：

```text
IP reputation
     │
SNI / Domain
     │
TLS fingerprint
     │
Packet length
     │
Timing
     │
Active probing
     │
Protocol-specific DPI
```

---

# 三十八、最终总结

如果只从 **2026 年、IPv6、443、长期抗封锁、性能** 这几个条件同时考虑，现阶段最值得认真研究的不是十几个古老协议，而是下面这些方向：

**TCP/443 主线：**

```text
VLESS + REALITY + Vision
VLESS + XHTTP + REALITY
NaiveProxy + HTTP/2
AnyTLS + TLS
```

这四个分别代表：

```text
TLS mimicry
Web transport
TLS behavior shaping
```

**TCP/443 第二路线：**

```text
Restls + SS2022
ShadowTLS V3 + SS2022
Mieru TCP
```

适合当作完全不同的 traffic profile。

**UDP/443 性能路线：**

```text
Hysteria2
Hysteria2 + Chrome QUIC fingerprint
Hysteria2 + Gecko
TUIC v5
```

但是必须记住：

> **从 2024 年开始，QUIC 已经进入 GFW 的专项审查范围；它的最大优势是性能，而不是天然隐蔽。** ([GFW Report][2])

**VPN 路线：**

```text
AmneziaWG
```

比原生 WireGuard 更有抗 DPI 设计，但依然属于“混淆 VPN”，不是“伪装 HTTPS”。

---

### 一个特别重要的结论

如果你把你的目标定义得非常严格：

> **“IPv6 原生直连、TCP/443、无需 CDN、希望外观尽可能接近正常 HTTPS、性能还不能太差。”**

那么真正应该重点研究的是：

```text
VLESS + REALITY + Vision
        vs
NaiveProxy + HTTP/2
        vs
VLESS + XHTTP + REALITY
        vs
AnyTLS
        vs
Restls
```

而不是拿 Hysteria2/TUIC 和它们简单比较“谁更隐蔽”。

因为 **Hysteria2/TUIC 解决的是 transport performance 问题，而上面五个主要解决的是 TCP/TLS traffic classification 问题**。

更进一步，2026 年真正值得研究的是：

> **REALITY 与 Naive 的“真实 Web 外观”路线，和 AnyTLS/Restls 的“主动破坏 TLS-in-TLS 行为特征”路线，到底谁更能适应下一代 DPI。**

这比单纯比较“VLESS vs Hysteria2 谁快”要有意义得多。

[1]: https://gfw.report/publications/foci25/en/?utm_source=chatgpt.com "Extended Abstract: I’ll Shake Your Hand: What Happens After DNS Poisoning"
[2]: https://gfw.report/publications/usenixsecurity25/en/?utm_source=chatgpt.com "Exposing and Circumventing SNI-based QUIC Censorship of the Great Firewall of China"
[3]: https://github.com/XTLS/Xray-docs-next/blob/main/docs/en/config/transport.md?utm_source=chatgpt.com "Xray-docs-next/docs/en/config/transport.md at main · XTLS/Xray-docs-next · GitHub"
[4]: https://github.com/XTLS/Xray-docs-next/blob/main/docs/en/config/transports/reality.md?utm_source=chatgpt.com "Xray-docs-next/docs/en/config/transports/reality.md at main · XTLS/Xray-docs-next · GitHub"
[5]: https://github.com/XTLS/Xray-core/issues/6421?utm_source=chatgpt.com "VLESS + REALITY + XHTTP(443) suspected active probing / China-side TCP handshake blocking evidence · Issue #6421 · XTLS/Xray-core · GitHub"
[6]: https://github.com/XTLS/Xray-core/discussions/5778?utm_source=chatgpt.com "Warning \"The feature VLESS (with no Flow, etc.) is deprecated\" · XTLS Xray-core · Discussion #5778 · GitHub"
[7]: https://github.com/XTLS/Xray-core/discussions/4113?utm_source=chatgpt.com "XHTTP: Beyond REALITY · XTLS Xray-core · Discussion #4113 · GitHub"
[8]: https://github.com/klzgrad/naiveproxy/blob/master/README.md?utm_source=chatgpt.com "naiveproxy/README.md at master · klzgrad/naiveproxy · GitHub"
[9]: https://github.com/klzgrad/naiveproxy/issues/720?utm_source=chatgpt.com "Question,关于移动宽带的SNI阻断 · Issue #720 · klzgrad/naiveproxy · GitHub"
[10]: https://github.com/klzgrad/naiveproxy/issues/723?utm_source=chatgpt.com "naiveproxy在Linux平台下的性能问题 · Issue #723 · klzgrad/naiveproxy · GitHub"
[11]: https://github.com/klzgrad/naiveproxy/issues/175?utm_source=chatgpt.com "How to setup caddyserver http3 and naive client http3? ERR_QUIC_HANDSHAKE_FAILED · Issue #175 · klzgrad/naiveproxy"
[12]: https://github.com/anytls/anytls-go/blob/main/docs/protocol.md?utm_source=chatgpt.com "anytls-go/docs/protocol.md at main · anytls/anytls-go · GitHub"
[13]: https://github.com/anytls/anytls-go/blob/main/readme.md?utm_source=chatgpt.com "anytls-go/readme.md at main · anytls/anytls-go · GitHub"
[14]: https://github.com/anytls/anytls-go/blob/main/docs/faq.md?utm_source=chatgpt.com "anytls-go/docs/faq.md at main · anytls/anytls-go · GitHub"
[15]: https://github.com/3andne/restls?utm_source=chatgpt.com "GitHub - 3andne/restls: Restls Protocol: A Perfect Impersonation of TLS; Restls协议: 对TLS的完美伪装 · GitHub"
[16]: https://github.com/3andne/Clash.Meta?utm_source=chatgpt.com "GitHub - 3andne/Clash.Meta: A rule-based tunnel in Go. · GitHub"
[17]: https://github.com/shadowsocks/shadowsocks-rust/blob/master/Cargo.toml?utm_source=chatgpt.com "shadowsocks-rust/Cargo.toml at master · shadowsocks/shadowsocks-rust · GitHub"
[18]: https://github.com/XTLS/Xray-docs-next/blob/main/docs/config/transport.md?utm_source=chatgpt.com "Xray-docs-next/docs/config/transport.md at main · XTLS/Xray-docs-next · GitHub"
[19]: https://github.com/ihciah/shadow-tls/wiki/V3-Protocol?utm_source=chatgpt.com "V3 Protocol · ihciah/shadow-tls Wiki · GitHub"
[20]: https://github.com/ihciah/shadow-tls/releases?utm_source=chatgpt.com "Releases · ihciah/shadow-tls · GitHub"
[21]: https://github.com/enfein/mieru/releases?utm_source=chatgpt.com "Releases · enfein/mieru · GitHub"
[22]: https://github.com/enfein/mieru/blob/main/README.md?utm_source=chatgpt.com "mieru/README.md at main · enfein/mieru · GitHub"
[23]: https://github.com/enfein/mieru/blob/main/docs/server-install.md?utm_source=chatgpt.com "mieru/docs/server-install.md at main · enfein/mieru · GitHub"
[24]: https://github.com/enfein/mieru/issues/54?utm_source=chatgpt.com "mieru v2 and beyond · Issue #54 · enfein/mieru · GitHub"
[25]: https://github.com/HyNetworks/hysteria/releases?after=v0.5.5&utm_source=chatgpt.com "Releases · HyNetworks/hysteria · GitHub"
[26]: https://sing-box.sagernet.org/changelog/?utm_source=chatgpt.com "Change Log - sing-box"
[27]: https://github.com/liummingyuan/hysteria2/blob/master/PROTOCOL.md?utm_source=chatgpt.com "hysteria2/PROTOCOL.md at master · liummingyuan/hysteria2 · GitHub"
[28]: https://github.com/apernet/hysteria-website/blob/master/docs/docs/advanced/Full-Client-Config.md?utm_source=chatgpt.com "hysteria-website/docs/docs/advanced/Full-Client-Config.md at master · apernet/hysteria-website · GitHub"
[29]: https://github.com/apernet/hysteria/issues/1654?utm_source=chatgpt.com "BBR panic on PMTU update to 1262 when Chrome parroting uses InitialPacketSize 1250 · Issue #1654 · apernet/hysteria · GitHub"
[30]: https://github.com/SagerNet/sing-box/blob/testing/docs/configuration/outbound/tuic.md?utm_source=chatgpt.com "sing-box/docs/configuration/outbound/tuic.md at testing · SagerNet/sing-box · GitHub"
[31]: https://github.com/tuic-protocol/tuic/releases?utm_source=chatgpt.com "Releases · tuic-protocol/tuic · GitHub"
[32]: https://github.com/juicity/juicity/blob/main/docs/spec.md?utm_source=chatgpt.com "juicity/docs/spec.md at main · juicity/juicity · GitHub"
[33]: https://github.com/WireGuard/wireguard-go/blob/master/device/noise-protocol.go?utm_source=chatgpt.com "wireguard-go/device/noise-protocol.go at master · WireGuard/wireguard-go · GitHub"
[34]: https://docs.amnezia.org/documentation/amnezia-wg/?utm_source=chatgpt.com "AmneziaWG | Amnezia Docs"
[35]: https://gfw.report/blog/blocking_of_tls_based_circumvention_tools/zh?utm_source=chatgpt.com "中国大规模地封锁基于TLS的翻墙服务器"
[36]: https://github.com/3andne/restls/blob/main/Restls-Script%3A%20Hide%20Your%20Proxy%20Traffic%20Behavior.md?utm_source=chatgpt.com "restls/Restls-Script: Hide Your Proxy Traffic Behavior.md at main · 3andne/restls · GitHub"
[37]: https://gfw.report/blog/gfw_unconditional_rst_20250820/zh/?utm_source=chatgpt.com "2025年8月20日中国防火长城GFW对443端口实施无条件封禁的分析"


继续往下挖之后，结论比上一版更清楚了：**2026 年的核心已经不是“哪个代理协议有更强的加密”，而是“能不能让审查者无法把你的连接从正常互联网流量中可靠地区分出来”。**

而且我修正一个上一版略显乐观的地方：**REALITY、XHTTP、Naive 现在都不能简单说成“★★★★★ 主动探测抗性”**。其中 Naive 的架构在主动探测防御上仍然非常漂亮；REALITY/XHTTP 则已经出现 2026 年针对性的现场阻断报告，而独立、系统性的公开测量还不足以下结论说它们已经被“完整破解”。这一点必须严格区分。

---

# 一、先给你这轮研究最重要的结论

截至 **2026 年 9 月 18 日**，如果把“IPv6 + 443 + 抗封锁 + 性能”作为整体问题，我会把现在的技术路线理解为：

```text
                    代理传输
                       │
        ┌──────────────┼──────────────┐
        │              │              │
     TCP/TLS        HTTP-native     UDP/QUIC
        │              │              │
 REALITY / AnyTLS   Naive / XHTTP   Hysteria2 / TUIC
        │              │              │
   主动探测问题      行为仿真问题      QUIC DPI
        │              │              │
        └──────────────┼──────────────┘
                       │
                  IP / SNI / DNS
                  reputation
```

真正的下一代竞争点已经变成了：

**TLS fingerprint → packet length → timing → multiplexing → active probing → IP reputation → behavioral correlation**

也就是说：

> **“加密”已经是最容易解决的问题；“流量看起来像什么”才是越来越难的问题。**

2022 年 GFW Report 就已经记录到 Trojan、VLESS、gRPC、V2Ray TLS+WS 等 TLS 类方案遭遇大规模封锁，并认为 TLS fingerprint 可能参与识别；而 2025 年 USENIX Security 的公开测量进一步证明 GFW 已经对 QUIC 建立了专门的解析与 SNI 过滤系统。([GFW Report][1])

---

# 二、目前最重要的区别：REALITY 和 NaiveProxy 并不是同一种“伪装”

这是研究以后我认为最值得强调的一点。

## REALITY 的思路

```text
你看到：

正常 TLS Server A
```

实际：

```text
客户端
 ↓
REALITY
 ↓
VLESS
```

它解决的是：

> **TLS 连接本身应该长什么样。**

所以 REALITY 的强项主要在：

* TLS ServerHello / ClientHello 外观
* certificate / server name 仿真
* 主动探测时不要暴露“这里是代理服务器”

---

## NaiveProxy 的思路

NaiveProxy 则是：

```text
Chrome network stack
        +
HTTP/2
        +
正常 frontend
        +
padding / fragmentation
```

官方目前仍明确把它的防御目标定义为：

* website fingerprinting
* TLS parameter fingerprinting
* active probing
* length-based traffic analysis

并且强调应该持续更新，以保持与 Chrome 的签名一致。([GitHub][2])

所以它解决的是：

> **“不只是 TLS 像浏览器，而是整个 HTTPS 会话尽可能像真实 Chrome。”**

这在理论模型上，其实比：

```text
uTLS 模拟 Chrome
```

要先进。

因为真正的浏览器不是只有 ClientHello。

---

# 三、这也是为什么 sing-box 现在直接不推荐 uTLS 作为长期抗审查手段

这是一个非常值得注意的 2026 更新。

sing-box 当前文档已经明确写：

> uTLS 有重复出现的 fingerprinting vulnerabilities。

原因是：

```text
uTLS
=
复制 ClientHello
```

但：

```text
Chrome
=
BoringSSL
+
真实 TCP 行为
+
真实 TLS 行为
+
真实 record 行为
+
真实 HTTP/2
+
真实连接生命周期
```

所以只复制 ClientHello 并不足够。

sing-box 当前文档甚至直接建议：

> 要做 TLS fingerprint resistance，应该使用 NaiveProxy，而不是 uTLS。([GitHub][3])

这个判断很重要。

---

# 四、因此我现在会把“TLS 伪装”重新分成三级

## 第一代

```text
TLS
+
自己实现 ClientHello
```

例如很多早期 Trojan/VLESS/SS + TLS。

最容易出问题。

---

## 第二代

```text
TLS
+
uTLS fingerprint
```

例如：

```text
chrome
firefox
safari
```

理论上更像浏览器。

但只是：

> **指纹像。**

---

## 第三代

```text
真实浏览器网络栈
+
真实 HTTP/2
+
真实 multiplexing
+
真实 connection behavior
```

也就是 NaiveProxy 的思路。

这是目前技术上最值得关注的一条路线。

---

# 五、但是 NaiveProxy 也不是“无法封锁”

这里必须把架构优势和现实结果分开。

2025 年就有用户报告：

```text
Naive
↓
某地区突然无法使用

REALITY
↓
同地区正常
```

用户自己推测可能是基于 SNI 进行了限制。这个是社区现场报告，不是 GFW 的独立实验室证明。([GitHub][4])

所以 Naive 的优势不是：

> “GFW 看不出来。”

而是：

> **如果审查者要封，它更难只通过一个非常特殊的代理指纹完成。**

这是两回事。

---

# 六、Naive 最大的优势，其实是“前端 + 后端分离”

这个比单纯 Chrome fingerprint 更重要。

Naive 的架构：

```text
          ┌─────────────┐
Client →  │  CDN / Caddy│
          └──────┬──────┘
                 │
      正常用户 ──┤
                 │
      探测请求 ──┤→ 普通 Web
                 │
      认证代理 ──┴→ Naive
```

官方 README 明确采用：

```text
Browser
 ↓
Naive client
 ↓
censor
 ↓
Frontend
 ↓
Naive server
```

未认证请求可以被前端路由到普通网站，因此主动扫描服务器时不一定能得到“这里存在代理”的证据。([GitHub][2])

这实际上是一种：

> **application-layer active probing resistance**

而不是单纯密码学意义上的“抗探测”。

---

# 七、REALITY 的问题正在变成另一个东西：IP / endpoint reputation

2026 年已经出现了多个 REALITY 的现场报告。

例如：

* 2026-01：用户报告 REALITY 节点被封；
* 2026-03：用户报告大量 REALITY 相关 IP 同时被封，而其他协议 IP 未受影响；
* 2026-07：出现 VLESS + REALITY + XHTTP 在 TCP/443 上疑似遭遇主动探测后，中国侧握手阻断的报告。

这些都是 **GitHub 用户/开发者 issue，不是独立测量结果**，所以不能说：

> “REALITY 已经被 GFW 破解。”

但足够说明：

> **现实中已经不能把 REALITY 当成静态、永久有效的隐身层。**

([GitHub][5])

尤其 2026-07 的那条现场记录很值得关注：服务器在失败后仍然能看到来自中国客户端的 SYN，但连接无法形成正常代理会话；报告同时观察到了疑似异常探测流量。不过维护者没有把它定性为 REALITY 已被协议级破解。([GitHub][6])

---

# 八、XHTTP 的地位因此发生了变化

上一版我把：

```text
VLESS + XHTTP + REALITY
```

看得比较高。

现在重新审查以后，我仍然认为它**技术上先进**，但要分成：

### 技术设计先进

和

### 当前稳定性 / 对抗成熟度

这是两个指标。

---

# 九、XHTTP 为什么设计得漂亮？

XHTTP 本质上不是：

```text
VLESS
+
HTTP wrapper
```

而是把 HTTP 会话本身作为传输层重新设计。

当前 Xray 的配置已经支持：

```text
packet-up
stream-up
stream-one
```

还有：

* request padding
* header/query/cookie padding
* session placement
* sequence placement
* HTTP POST/GET
* SSE-like response
* gRPC-like header
* XMUX connection reuse

这些都不是传统 WS 的简单包装。([GitHub][7])

---

# 十、XHTTP 的三个模式其实对应三种流量模型

### packet-up

```text
POST
POST
POST
POST
   ↓
流式下载
```

优势：

* HTTP 兼容性强
* 更容易进入 CDN / reverse proxy

缺点：

* 上行切成多个 HTTP request
* 性能不如真正 stream

---

### stream-up

```text
一个长期 POST
        ↕
     双向流
```

优势：

* 性能更接近裸 TCP
* 上行不会被拆成大量 POST

---

### stream-one

```text
POST
 ↓
长期双向 response
```

设计非常特别。

XHTTP 开发讨论中还说明了：

* stream-up / stream-one
* packet-up
* H2 / H3 / REALITY 不同默认模式
* CDN/HTTP 服务器的不同兼容行为

([GitHub][8])

---

# 十一、所以 XHTTP 现在真正的问题不是“功能不够”

恰恰相反：

> **XHTTP 的功能已经越来越复杂。**

2026 年出现的实际问题包括：

* v26.6.27 → v26.7.11 后 XHTTP + REALITY 出现连接兼容性问题；
* `mode=auto` 曾出现兼容问题；
* session ID 在不同版本间曾出现兼容变化；
* 有 2026 年报告发现 XHTTP + REALITY 出现内存/连接泄漏；
* 另有 2026 年报告称某些 v26 客户端版本相比 v25 更容易在受审查网络中被识别。

这些仍然需要区分：

**实现 bug ≠ 审查破解。**

但对于生产部署而言，它们都意味着：

> **XHTTP 是快速演进的协议，不适合“配一次放三年”。**

([GitHub][9])

---

# 十二、还有一个非常新的变化：VLESS Encryption

这是我这次特别建议你关注的。

2026 年 Xray 已经开始推动：

```text
VLESS
+
VLESS Encryption
+
Vision
```

而不是继续把传统 Shadowsocks-style encryption 当作未来路线。

Xray 当前代码已经支持：

```text
mlkem768x25519plus
```

并使用 X25519 / ML-KEM-768 等机制，同时加入 padding / relay 等设计。([GitHub][10])

Xray 项目在 2026 年还明确开始对：

```text
VLESS without Flow
```

发出弃用警告，并推动 VLESS Encryption。([GitHub][11])

---

# 十三、但 VLESS Encryption 千万别理解成“新的混淆协议”

它更接近：

```text
VLESS
+
更现代的端到端加密 / PFS / PQ crypto
```

而不是：

```text
新的 HTTPS camouflage
```

所以：

> **VLESS Encryption 解决的是“代理内层加密结构”的未来问题，不是直接解决 GFW DPI。**

例如：

```text
VLESS Encryption
        ↓
加密很好
        ↓
但外层如果仍然是奇怪的 RAW/TCP
        ↓
DPI 依旧可以看外部行为
```

这一点在理解 Xray 新架构时非常重要。

---

# 十四、ML-KEM 还可能让“行为伪装”问题变得更复杂

Xray 的 VLESS Encryption 握手中本身会涉及：

```text
ML-KEM-768 ciphertext
```

其长度明显大于传统 X25519 公钥。

代码中可以直接看到：

```text
X25519 public key = 32 bytes

ML-KEM-768 ciphertext = 1088 bytes
```

所以：

```text
PQ handshake
```

本身会改变首包长度及分片行为。([GitHub][12])

这不是说 ML-KEM 会导致协议被封。

恰恰相反：

> **后量子密钥交换是密码学进步，但“更大、更特殊的握手”同样会改变流量统计特征。**

这就是为什么：

```text
cryptographic security
```

和：

```text
traffic analysis resistance
```

绝对不能混成一个维度。

---

# 十五、AnyTLS 现在反而变得更有意思

AnyTLS 最新公开版本已经到 **v0.0.13**。([GitHub][13])

它的核心设计还是：

```text
TLS
 ↓
AnyTLS session
 ↓
multiplex streams
```

但是和普通：

```text
TLS
 ↓
另一个 TLS
```

不一样。

AnyTLS 直接把：

* padding
* session
* stream
* heartbeat
* padding scheme update

纳入协议本身。([GitHub][14])

---

# 十六、AnyTLS 真正值得关注的东西，是“可变协议行为”

当前协议允许服务端动态下发：

```text
padding scheme
```

并且不同连接可以改变 padding 行为。([GitHub][14])

这和传统：

```text
固定包长
固定 padding
固定握手
```

已经是不同思想。

---

# 十七、但 AnyTLS 自己也明确告诉你一个事实

它的 FAQ 直接说：

> 默认 PaddingScheme 只是一个示例。

并没有声称：

> “这个 padding 就是某种不可识别模板。”

同时它也明确表示 TLS ClientHello/ServerHello fingerprint 本身不是 AnyTLS 参考实现重点。([GitHub][15])

所以：

```text
AnyTLS
=
非常好的“TLS 内部行为整形协议”
```

但不是：

```text
AnyTLS
=
Chrome impersonation
```

这两者必须分开。

---

# 十八、这也引出了 Restls 的真正价值

Restls 现在仍然很有研究意义，因为它明确承认：

> TLS-in-TLS 本身可以形成侧信道。

它直接让用户定义：

```text
第1个包多少
第2个包多少
第3个包是否等待回应
第4个包多少
```

并明确表示：

> 通过 script 可以打散代理常见的 TLS-in-TLS 行为。([GitHub][16])

---

# 十九、Restls 有一个非常重要的自我承认

它并没有声称：

> 后续所有行为都能伪装。

官方自己指出：

> script 主要解决前几个包；长时间的大流量行为本身很难完全隐藏。([GitHub][17])

这其实是非常正确的网络安全现实。

比如：

```text
普通网页

100 KB
20 KB
40 KB
断续
100 KB
2 MB
停止
```

而代理：

```text
连续下载
连续 300 Mbps
持续 20 分钟
```

即使前 5 个 packet 完全一样：

> 后面的行为还是明显不同。

所以：

**不存在一个“完美 packet padding”能消灭所有 traffic analysis。**

---

# 二十、Mieru 现在反而是一个值得长期观察的“第三路线”

它没有 TLS。

它的思路是：

```text
加密
+
随机 padding
+
low entropy
+
heartbeat jitter
+
traffic pattern
```

2026 年连续更新：

* 5 月 heartbeat jitter
* 6 月 padding traffic pattern
* 7 月 low entropy mode
* 7 月之后继续性能优化

([GitHub][18])

---

# 二十一、Mieru 的 low entropy 是非常值得研究的设计

传统：

```text
密文
████████████████
```

统计熵非常高。

Mieru 可以变成：

```text
████░░████░░████
```

也就是人为增加非密文字节，使整体统计性质发生变化。

官方文档给出的模式包括：

```text
32-bit payload / 32-bit padding
40 / 24
48 / 16
56 / 8
```

代价是：

```text
带宽 ×
CPU ×
有效 MTU ↓
```

例如最激进的 32-bit 模式理论上可能接近 **2 倍数据膨胀**。([GitHub][19])

所以 Mieru 的这个功能可以理解成：

> **牺牲效率，换统计特征。**

---

# 二十二、但是 Mieru 也有自己的“历史包袱”

2023 年 Mieru 开发者曾公开承认：

* 中国方向已有 TCP/UDP 阻断；
* 原有协议无法提供足够稳定的代理能力；
* GFW 开始识别这种“高熵全加密流量”。

([GitHub][20])

这其实反而让我觉得 Mieru 的发展方向比较值得看：

> 它没有假装“原来的协议足够隐蔽”，而是在持续修改 traffic statistics。

2026 年它依然在这么做。

---

# 二十三、QUIC 这一边，现在证据反而是最充分的

这是目前所有协议里：

> **“GFW 已经具体怎么打”证据最硬的一类。**

2025 USENIX Security 的测量确认：

```text
QUIC Initial
   ↓
GFW 解密
   ↓
ClientHello
   ↓
SNI
   ↓
QUIC-specific blocklist
```

而且研究者测到：

> 被命中的连接后续 UDP 流量会受到持续时间约 180 秒的残余阻断。([GFW Report][21])

更重要的是：

> GFW 的 QUIC blocklist 与传统机制不同。

这意味着：

```text
QUIC
不是
“TCP 审查体系换成 UDP”
```

而是：

> **已经成为一套独立审查系统。**

---

# 二十四、但 QUIC 并不是“毫无办法”

这反而是一个特别有意思的研究结果。

2025 论文发现：

* QUIC Initial fragmentation
* SNI slicing
* 某些 QUIC version / packet layout 行为

能够干扰当时 GFW 的 Initial 解析。

这一成果随后已经被：

* quic-go
* Firefox
* Hysteria
* Xray
* V2Ray
* sing-box
* TUIC

等生态吸收。([GFW Report][21])

所以今天看 Hysteria2：

> 不能再拿 2024 年初版本和现在比较。

现在 Hysteria 生态已经主动引入：

* Chrome QUIC fingerprint parroting
* Gecko obfuscation
* BBR profile
* hop interval randomization

sing-box 当前版本说明中也明确列出这些 Hysteria2 能力。([GitHub][22])

---

# 二十五、所以 Hysteria2 到 2026 年的正确定位应该是

不是：

> “最隐蔽”

而是：

> **“性能极强，而且 QUIC 反审查工程也在快速迭代。”**

这是完全不同的评价。

---

# 二十六、TUIC 则更接近“标准 QUIC VPN”

TUIC v5 的设计包括：

* 0-RTT TCP
* UDP fragmentation
* TLS exporter authentication
* 多用户
* QUIC transport

最新的 TUIC 代码库还在重构/维护，IPv6 UDP relay 也明确支持。([GitHub][23])

它的优点：

```text
QUIC 正规性
+
性能
+
多路复用
```

但也因此：

> 它和 Hysteria2 一样，继承了 QUIC 的审查攻击面。

---

# 二十七、真正让我重新提高评价的，反而是 AmneziaWG 3.1

这个项目 2026 年变化很大。

官方现在把 3.1 描述为：

```text
动态 packet size
+
packet sequence
+
timing
+
protocol metadata
+
header protection
+
content padding
```

而不仅仅是简单：

```text
随机改几个 header
```

([Amnezia 文档][24])

甚至加入：

```text
CPS
=
Custom Protocol Signature
```

允许形成类似：

```text
QUIC Initial
DNS
其他 UDP 协议
```

的外观。

---

# 二十八、不过 AmneziaWG 有一个本质局限

它依然是：

```text
UDP VPN
```

所以：

```text
TCP/443 HTTPS
```

那条生态链：

```text
Chrome
HTTP/2
TLS
CDN
reverse proxy
```

它都不具备。

因此我会把它定位成：

> **非常好的 VPN transport obfuscation。**

而不是：

> **HTTPS camouflage。**

---

# 二十九、ECH 是下一张非常大的牌

这一点上一版没有讲够。

现在 ECH 已正式进入 IETF RFC：

**RFC 9849 — TLS Encrypted Client Hello。**([IETF Datatracker][25])

它真正解决的是：

```text
TLS 1.3
仍然暴露：

SNI
ALPN
```

而 ECH：

```text
ClientHelloOuter
        │
        └── encrypted ClientHelloInner
                       │
                     SNI
```

因此审查者无法像以前一样直接读取 inner SNI。

---

# 三十、但 ECH 并没有变成“翻墙专用协议”

这是最容易产生误解的地方。

ECH 要求：

```text
client
+
client-facing server
+
backend
```

配合起来。

RFC 也明确强调：

> ECH 本身不能隐藏 server IP。

而且通常需要：

```text
ECH configuration
+
DNS HTTPS/SVCB
```

之类的协同。([IETF Datatracker][25])

所以：

```text
自己一台 VPS
↓
自己生成 ECH
↓
自己用
```

并不会自动获得“大型互联网 CDN”的匿名集合。

---

# 三十一、不过 ECH 已经进入 sing-box / Xray

这一点非常关键。

sing-box 当前 TLS 配置已经：

```text
ech.enabled
ech.config
ech.config_path
ech.query_server_name
```

并明确支持 ECH。([GitHub][3])

Xray 也已经实现 ECH client/server 支持。([GitHub][26])

因此未来真正有意思的组合可能不是：

```text
REALITY vs ECH
```

而是：

```text
正常 TLS
+
ECH
+
HTTP/2
+
真实 frontend
```

这种更接近“下一代标准 Web 隐私基础设施”。

---

# 三十二、MASQUE 也值得注意

IETF 当前的：

```text
CONNECT-UDP
```

已经标准化。

RFC 9298 定义：

```text
HTTP
 ↓
CONNECT-UDP
 ↓
UDP tunnel
```

HTTP/2、HTTP/3 都可以使用 Extended CONNECT。并且标准直接允许 IPv6 target。([IETF Datatracker][27])

这意味着从协议发展方向上：

```text
Hysteria2 / TUIC
```

只是“专用 QUIC tunnel”，

而：

```text
MASQUE
```

则是：

> **把 VPN/UDP forwarding 融入标准 HTTP/3 生态。**

---

# 三十三、问题是：GFW 已经注意到了 MASQUE

2025 QUIC 论文明确发现：

> Cloudflare WARP 的 MASQUE 子域名遭到 GFW 封锁。

论文还指出：

> Hysteria 的官方域名和用户自定义 Hysteria proxy domain 都曾进入相关封锁体系。

([GFW Report][21])

所以：

> **“标准化 = 不会封”同样是错误的。**

---

# 三十四、现在我给你一个真正的“打击面矩阵”

| 攻击手段                  | REALITY | XHTTP |   Naive |   AnyTLS | Restls |     Mieru |        Hysteria2 |             TUIC | AmneziaWG |
| --------------------- | ------: | ----: | ------: | -------: | -----: | --------: | ---------------: | ---------------: | --------: |
| IP 封锁                 |      有效 |    有效 |      有效 |       有效 |     有效 |        有效 |               有效 |               有效 |        有效 |
| DNS/SNI               |      中等 |    中等 |      中等 |       中等 |     中等 | 无 TLS SNI |               中等 |               中等 |         无 |
| TLS fingerprint       |     强防御 |   强防御 | **非常强** | 依赖外层 TLS |      强 |       不适用 | QUIC fingerprint | QUIC fingerprint |       不适用 |
| 包长分析                  |      中等 | **强** |   **强** |    **强** | **很强** |    **很强** |               中等 |               中等 |        很强 |
| timing analysis       |      中等 |    中等 |      中等 |       中等 |     很强 |        很强 |               中等 |               中等 |        很强 |
| 主动探测                  |       强 |     强 | **非常强** |        强 |      强 |        中高 |               中高 |               中高 |         中 |
| protocol-specific DPI |       中 |     中 |       低 |      低/中 |      低 |         中 |            **高** |            **高** |        中高 |
| 生态复杂度                 |       中 |     高 |       中 |        中 |      高 |         中 |                中 |                中 |         中 |
| 当前性能                  |      很高 |     高 |       高 |        高 |     中高 |         高 |           **极高** |           **极高** |    **极高** |

这里的“攻击面”不是严格实验评分，而是根据协议结构和当前公开证据建立的工程判断。

---

# 三十五、如果只看“最难同时满足的几个指标”，会发现一个三角矛盾

```text
           隐蔽
            ▲
           / \
          /   \
         /     \
        /       \
       ▼---------▼
    性能         标准化
```

你通常无法同时做到：

```text
极高性能
+
完全像普通浏览器
+
完全标准协议
+
任意代理流量
+
低资源
```

---

# 三十六、例如：

## Naive

```text
隐蔽 █████
性能 ████
标准化 █████
```

因为它直接靠真实浏览器网络栈。

---

## Hysteria2

```text
隐蔽 ███
性能 █████
标准化 ███
```

因为它极度重视性能，而不是完全模仿浏览器。

---

## Restls

```text
隐蔽 █████
性能 ███
标准化 ██
```

因为它可以精细控制 traffic shape，但这种自由度本身带来复杂度。

---

## Mieru

```text
隐蔽 ████
性能 ████
标准化 ██
```

路线完全不同。

---

# 三十七、还有一个经常被忽略的“大敌”：连接生命周期

现在的 DPI 不一定只看：

```text
前 10 个包
```

它还可以看：

```text
连接持续时间
请求数
并发数
idle 时间
上传/下载比例
连接复用次数
RST 数量
重连模式
```

这也是为什么：

```text
Restls Script
```

解决不了全部问题。

因为它自己也承认：

> 大量持续流量的真实行为很难完全伪装。([GitHub][17])

---

# 三十八、因此以后“混淆参数越多越好”很可能是错误方向

比如：

```text
随机 padding
随机 delay
随机 packet
随机 header
随机 TLS
随机 session
```

看起来很强。

但如果：

```text
所有代理用户
↓
都随机成同一种随机分布
```

那么：

> “随机”本身就会成为指纹。

这实际上是一个非常重要的统计学问题。

---

# 三十九、Naive 的思路恰恰绕开了这个陷阱

它不是：

```text
随机模拟浏览器
```

而是：

```text
尽量真的使用浏览器技术栈
```

这就形成：

```text
normal browser distribution
```

而不是：

```text
proxy's random distribution
```

这是为什么我现在比上一版更加重视 Naive。

---

# 四十、但 Naive 也面对“Chrome 版本同步”的新问题

当前 Naive 最新 release 已经到了：

**v150.0.7871.63-1**，2026 年 7 月发布。([GitHub][28])

官方还明确要求：

> 用户应使用最新版本，使签名保持与 Chrome 一致。([GitHub][2])

这意味着：

```text
Chrome 更新
     ↓
TLS / HTTP2 behavior 改变
     ↓
Naive 需要跟
```

所以 Naive 的隐蔽性不是：

> 一次部署永久不变。

它实际上是：

> **持续跟浏览器升级。**

---

# 四十一、这其实就是未来抗 DPI 的主线

可以把整个历史简化成：

```text
Shadowsocks
     ↓
TLS
     ↓
uTLS
     ↓
REALITY
     ↓
XHTTP / AnyTLS / Restls
     ↓
真实 Browser Stack
     ↓
ECH + HTTP/2/3 + MASQUE
```

也就是说：

> **未来很可能不是继续创造越来越奇怪的“翻墙协议”，而是尽可能直接进入普通互联网协议生态。**

---

# 四十二、因此如果你让我现在重新规划“IPv6 443 节点池”

我不会只部署：

```text
VLESS Reality
```

也不会只部署：

```text
Hysteria2
```

而会从 transport diversity 出发：

```text
                    IPv6
                      │
          ┌───────────┴───────────┐
          │                       │
       TCP/443                  UDP/443
          │                       │
    ┌─────┼──────┐          ┌─────┼─────┐
    │     │      │          │     │     │
 Naive  REALITY AnyTLS     Hysteria TUIC AWG
    │     │      │          │
    │     │      │          QUIC path
    │     │      │
    └─────┴──────┘
```

这样每一种失败原因都不一样。

---

# 四十三、我现在对各方案的长期定位

### VLESS + REALITY + Vision

**成熟度：高**

**性能：高**

**隐蔽性：高**

**现实风险：已经进入针对性对抗阶段**

2026 年已有多个 REALITY IP / 节点被封的现场报告，但没有足够公开独立证据证明“协议已经被完全识别”。([GitHub][5])

---

### VLESS + XHTTP + REALITY

**技术设计：非常高**

**性能：高**

**生态：快速增长**

**当前风险：高于上一版评估**

原因就是：

```text
2026 compatibility bugs
+
memory/leak reports
+
疑似主动探测/握手阻断报告
```

([GitHub][9])

---

### NaiveProxy + HTTP/2

**技术理念：非常强**

尤其是：

```text
真实 Chrome stack
+
frontend
+
HTTP/2 multiplex
+
padding
```

这是目前 TCP/443 里面我最希望长期跟踪的一条路线。([GitHub][2])

它唯一非常明显的短板：

> **它高度依赖 Chrome 的网络生态，并且 SNI / endpoint / IP 本身仍然可能成为封锁对象。**

---

### AnyTLS

**很值得继续观察。**

尤其适合研究：

```text
TLS-in-TLS
+
动态 padding
+
session multiplex
```

当前生态越来越成熟，但还没有像 Naive/REALITY 那样经过这么长时间的大规模现实对抗。([GitHub][14])

---

### Restls

**研究价值 > 实际生态价值。**

它非常适合研究：

> “DPI 到底通过什么 packet pattern 判断代理？”

因为它甚至允许人为改变这些模式。([GitHub][16])

---

### Mieru

**非常值得关注的非 TLS 路线。**

特别是 2026 年的：

```text
padding
heartbeat jitter
low entropy
```

已经明显是在和统计流量分析正面交手。([GitHub][18])

---

### Hysteria2

**性能极强。**

但是应该把：

```text
QUIC
```

视为一个独立攻击面，而不是隐蔽层。

现在的 Chrome QUIC fingerprint / Gecko / SNI-slicing 等升级是实实在在的重要增强。([GFW Report][21])

---

### TUIC

**高性能标准 QUIC 路线。**

适合作为 QUIC 第二传输，不应把它理解成不同于 Hysteria2 的“反 DPI 世界”。([GitHub][23])

---

### AmneziaWG 3.1

**2026 年非常值得重新评价。**

尤其是它已经从：

```text
改几个 packet header
```

进化到：

```text
size
+
sequence
+
timing
+
metadata
+
CPS
+
header protection
```

([Amnezia 文档][24])

但它仍属于：

> UDP VPN obfuscation

而不是：

> HTTPS camouflage。

---

# 四十四、最后是我认为最重要的一张“未来路线图”

```text
2020
│
├─ SS / VMess
│
2022
│
├─ TLS fingerprint resistance
├─ uTLS
│
2023
│
├─ REALITY
│
2024
│
├─ XHTTP
├─ QUIC censorship begins
│
2025
│
├─ GFW QUIC Initial inspection confirmed
├─ QUIC SNI slicing
├─ Hysteria Chrome fingerprint
│
2026
│
├─ XHTTP further evolution
├─ VLESS Encryption / ML-KEM
├─ AnyTLS dynamic padding
├─ Mieru low entropy
├─ AmneziaWG 3.1 statistical camouflage
├─ ECH standardized
└─ MASQUE becoming standard Web VPN substrate
```

这说明整个领域正在从：

> **“发明一个更神奇的加密协议”**

向：

> **“把 VPN 流量融进普通互联网协议统计分布”**

转变。

这才是最值得研究的方向。

---

## 我目前会特别关注的 5 条线

| 方向                                     | 现在的技术价值 | 原因                          |
| -------------------------------------- | ------: | --------------------------- |
| **Naive + HTTP/2 + frontend**          |      极高 | 最接近真实浏览器栈                   |
| **VLESS + XHTTP + 新 VLESS Encryption** |      极高 | Xray 正在主动转型                 |
| **REALITY 演进**                         |       高 | 仍是重要 TCP/443 方案，但开始进入主动对抗   |
| **AnyTLS / Restls / Mieru**            |       高 | 正面研究 traffic-shape analysis |
| **Hysteria2 + QUIC camouflage**        |      极高 | 性能最强，但与 GFW 进入持续攻防          |

而 **ECH + HTTP/2/3 + MASQUE** 是我认为值得放到下一阶段研究的东西，因为它代表的不是某一个翻墙项目，而是**标准互联网自身正在向“加密、隐藏 SNI、HTTP 原生 VPN”演化**。RFC 9849 已在 2026 年正式标准化 ECH；MASQUE 的 CONNECT-UDP 也已经是正式标准。([IETF Datatracker][25])

下一步最有价值的是把这份研究进一步做成一个**“2026 GFW/DPI 攻击面模型”**：逐项分析 **TCP SYN → TLS ClientHello → ServerHello → 首 10 个 record → HTTP/2 frames → 长连接 → 重连 → IP reputation → 主动探测**，然后把 REALITY、Naive、XHTTP、AnyTLS、Restls、Mieru、Hysteria2、TUIC、AmneziaWG 按这些阶段逐格对照。这样就能看出每个方案**究竟在哪一个阶段暴露，而不是笼统说“抗封/不抗封”**。

[1]: https://gfw.report/blog/blocking_of_tls_based_circumvention_tools/en/?utm_source=chatgpt.com "Large scale blocking of TLS-based censorship circumvention tools in China"
[2]: https://github.com/klzgrad/naiveproxy/blob/master/README.md?utm_source=chatgpt.com "naiveproxy/README.md at master · klzgrad/naiveproxy · GitHub"
[3]: https://github.com/sagernet/sing-box/blob/testing/docs/configuration/shared/tls.md?utm_source=chatgpt.com "sing-box/docs/configuration/shared/tls.md at testing · SagerNet/sing-box · GitHub"
[4]: https://github.com/klzgrad/naiveproxy/issues/706?utm_source=chatgpt.com "换区 na就连不上 · Issue #706 · klzgrad/naiveproxy · GitHub"
[5]: https://github.com/XTLS/Xray-core/discussions/5758?utm_source=chatgpt.com "REALITY IPs blocked · XTLS Xray-core · Discussion #5758 · GitHub"
[6]: https://github.com/XTLS/Xray-core/issues/6421?utm_source=chatgpt.com "VLESS + REALITY + XHTTP(443) suspected active probing / China-side TCP handshake blocking evidence · Issue #6421 · XTLS/Xray-core · GitHub"
[7]: https://github.com/XTLS/Xray-core/blob/main/infra/conf/transport_internet.go?utm_source=chatgpt.com "Xray-core/infra/conf/transport_internet.go at main · XTLS/Xray-core · GitHub"
[8]: https://github.com/XTLS/Xray-core/discussions/4113?utm_source=chatgpt.com "XHTTP: Beyond REALITY · XTLS Xray-core · Discussion #4113 · GitHub"
[9]: https://github.com/XTLS/Xray-core/issues/6482?utm_source=chatgpt.com "XHTTP + REALITY stops working after upgrading v26.6.27 → v26.7.11 · Issue #6482 · XTLS/Xray-core · GitHub"
[10]: https://github.com/XTLS/Xray-core/blob/main/infra/conf/vless.go?utm_source=chatgpt.com "Xray-core/infra/conf/vless.go at main · XTLS/Xray-core · GitHub"
[11]: https://github.com/XTLS/Xray-core/discussions/5568?utm_source=chatgpt.com "This feature VLESS without flow is deprecated and being migrated to VLESS with flow. Please update your config(s) according to release note and documentation before removal. · XTLS Xray-core · Discussion #5568 · GitHub"
[12]: https://github.com/XTLS/Xray-core/blob/main/proxy/vless/encryption/client.go?utm_source=chatgpt.com "Xray-core/proxy/vless/encryption/client.go at main · XTLS/Xray-core · GitHub"
[13]: https://github.com/anytls/anytls-go/releases?utm_source=chatgpt.com "Releases · anytls/anytls-go · GitHub"
[14]: https://github.com/anytls/anytls-go/blob/main/docs/protocol.md?utm_source=chatgpt.com "anytls-go/docs/protocol.md at main · anytls/anytls-go · GitHub"
[15]: https://github.com/anytls/anytls-go/blob/main/docs/faq.md?utm_source=chatgpt.com "anytls-go/docs/faq.md at main · anytls/anytls-go · GitHub"
[16]: https://github.com/3andne/restls?utm_source=chatgpt.com "GitHub - 3andne/restls: Restls Protocol: A Perfect Impersonation of TLS; Restls协议: 对TLS的完美伪装 · GitHub"
[17]: https://github.com/3andne/restls/blob/main/Restls-Script%3A%20Hide%20Your%20Proxy%20Traffic%20Behavior.md?utm_source=chatgpt.com "restls/Restls-Script: Hide Your Proxy Traffic Behavior.md at main · 3andne/restls · GitHub"
[18]: https://github.com/enfein/mieru/releases?utm_source=chatgpt.com "Releases · enfein/mieru · GitHub"
[19]: https://github.com/enfein/mieru/blob/main/docs/traffic-pattern.md?utm_source=chatgpt.com "mieru/docs/traffic-pattern.md at main · enfein/mieru · GitHub"
[20]: https://github.com/enfein/mieru/issues/54?utm_source=chatgpt.com "mieru v2 and beyond · Issue #54 · enfein/mieru · GitHub"
[21]: https://gfw.report/publications/usenixsecurity25/en/?utm_source=chatgpt.com "Exposing and Circumventing SNI-based QUIC Censorship of the Great Firewall of China"
[22]: https://github.com/SagerNet/sing-box/releases?utm_source=chatgpt.com "Releases · SagerNet/sing-box · GitHub"
[23]: https://github.com/tuic-protocol/tuic/releases?utm_source=chatgpt.com "Releases · tuic-protocol/tuic · GitHub"
[24]: https://docs.amnezia.org/documentation/amnezia-wg/?utm_source=chatgpt.com "AmneziaWG | Amnezia Docs"
[25]: https://datatracker.ietf.org/doc/html/9849?utm_source=chatgpt.com "RFC 9849 - TLS Encrypted Client Hello"
[26]: https://github.com/XTLS/Xray-core/blob/main/transport/internet/tls/ech.go?utm_source=chatgpt.com "Xray-core/transport/internet/tls/ech.go at main · XTLS/Xray-core · GitHub"
[27]: https://datatracker.ietf.org/doc/rfc9298/?utm_source=chatgpt.com "RFC 9298 - Proxying UDP in HTTP"
[28]: https://github.com/klzgrad/naiveproxy/releases?utm_source=chatgpt.com "Releases · klzgrad/naiveproxy · GitHub"


将视角从“非 443”切换到 “IPv6 + 443 端口” 时，代理体系的博弈逻辑会发生本质性的范式转变：

1.  掩护背景彻底改变（从“异常高熵”到“最大公约数”）： 非 443 最大的痛点是“端口本身就是特征”；而在现代互联网中，由于浏览器
    RFC 8305（Happy Eyeballs v2）的双栈优选机制，大量正常 Web 流量天然优先通过 IPv6:443
    访问各大云厂商与内容节点。因此，IPv6 的 443 端口拥有全世界规模最大、最自然的真实背景流量掩护。
2.  端口双工（TCP/443 与 UDP/443 并行）： 在同一 IPv6 地址上，TCP/443（标准 HTTPS）与 UDP/443（标准
    HTTP/3 / QUIC）在传输层完全独立，单 IP 即可同时实现 TCP 拟态与 QUIC 极速的双轨覆盖。
3.  地址空间维度碾压（/64 级地址池与封锁粒度）： 大多数 VPS 厂商分配 IPv6 时提供 /64 子网（2^{64} 个地址）。研究表明 GFW 在
    IPv6 上的动态自动化封锁主要集中在 /128（单 IP 级别），这为 443 端口带来了 IPv4 无法比拟的“零成本地址轮换”抗灾能力。
4.  技术陷阱（MTU 黑洞与国际路由不对称）： IPv6 头部固定为 40 字节且网络中间节点严禁分片（仅源端可分片，依赖
    PMTUD/ICMPv6），配置不当极易在移动端/PPPoE 环境产生“MTU 黑洞”导致握手卡死；同时三大运营商的 IPv6 跨境回程质量差异极大。

基于 2025–2026 年最新公开学术测量（如 USENIX
Security 2025、ProtoScan 2025、gfw.report）及各大开源核心项目规范，针对 IPv6:443 场景
的主流组合、抗封锁机理、性能效率及打击现状进行全面系统对比。

一、 核心基础：GFW 在 IPv6:443 上的审查现状与技术边界

在评估协议前，必须厘清当前 IPv6 与 443 端口的真实审查环境，破除“IPv6 免墙”的误区：

1.  IPv6 DPI 深度已与 IPv4 基本对齐：
      - 早期（2020 年前）GFW 对 IPv6 存在漏判和延迟；但 2024–2025 年多项针对 IPv6 的大规模测量（如 ProtoScan
        论文）证实：GFW 对 IPv6 流量的 TLS ClientHello SNI 解析、TCP 伪造 RST+ACK
        注入、双向状态跟踪、以及针对 QUIC Initial 的解密与 SNI 过滤，均已实现与 IPv4 几乎同等的审查覆盖。
      - 2025 年 8 月 20 日发生的针对 TCP/443 无条件 RST 异常事件中，测量数据记录到 IPv4 与 IPv6 的 TCP/443
        握手均遭到了无差别重置，证实 443 端口的核心阻断策略在 IPv6 上同样生效。
2.  IPv6 封锁粒度特征（/128 vs /64）：
      - 动态自动封锁通常为 /128： 当特定节点因大流量或流量特征触发自动黑名单时，GFW 的流表通常记录具体的 IPv6
        单播地址（/128），路由未完全黑洞化前，同一 /64 内的其他 IPv6 往往依然可用。
      - 静态/骨干网惩罚可能升级为 /64 甚至 /56：
        如果某个前缀内大量地址连续触发严重违规，或属于廉价高危机房，运营商边际路由器可能直接对整段下发 Null 路由。
3.  IPv6 MTU 严苛约束：
      - IPv6 规范要求链路最小 MTU 为 1280 字节。由于跨境链路上部分节点丢弃 ICMPv6 Type 2（Packet Too
        Big）报文，导致 PMTUD（路径 MTU 发现）失效。
      - 在 443 端口运行 TCP 或 QUIC 时，若包长（UDP Payload / TCP MSS）加上 40 字节 IPv6 报头超过路径真实
        MTU，会产生静默丢包（黑洞），常被误判为“协议被墙”。

二、 适合 IPv6:443 的主流代理协议与组合全景

根据传输层与伪装机制，适合 IPv6:443 的方案分为三大流派：

流派 A：TCP/443 —— 真实 Web / 仿冒 TLS 流量

1.  VLESS + REALITY + XTLS-Vision（借牌伪装）
2.  NaiveProxy + HTTP/2（真 Chrome 协议栈）
3.  VLESS / Trojan + XHTTP + TLS + Fallback（HTTP 原生与真实分流）
4.  Trojan + TLS 1.3 + Web Fallback（经典标准 HTTPS 回落）
5.  AnyTLS + TLS 1.3（现代动态填充与会话复用）
6.  ShadowTLS v3 + Shadowsocks 2022（外挂式 TLS 握手转发）
7.  Restls + Shadowsocks（脚本化 TLS 行为模拟）

流派 B：UDP/443 —— 真实 HTTP/3 伪装与极致性能

8.  Hysteria 2（默认 HTTP/3 伪装模式）（原生 443/UDP 标准形态）
9.  Hysteria 2 + Salamander / Gecko 混淆（放弃 H3 外观，转入高熵/随机分片）
10. TUIC v5（标准 QUIC 代理多路复用）

流派 C：IPv6 独有高阶架构组合

11. IPv6 /64 子网动态轮换（Any-IP / SLAAC）+ 任意 443 协议（将 IP 阻断的恢复成本降为零）
12. 单 IPv6:443 全双工混合（TCP REALITY + UDP Hysteria2 并行）

三、 维度深度对比表（IPv6:443 场景专属）

评级说明：

  - P（被动识别防御）： 对包长、熵值、时序、JA3/JA4/uTLS、QUIC 握手的抗分析能力。
  - A（主动探测防御）： 审查设备发起探针、回放、探测包时的响应与欺骗能力。
  - IPv6 适配度： 对 IPv6 头部开销、MTU 敏感性、PMTUD 失败容忍度、多地址绑定的适应力。
  - T（审查打击现状）： 截至 2026 年基于公开学术论文与实测记录的针对性程度。

| 方案组合                               | 传输/端口   | 加密机制                       | P：被动识别防御                          | A：主动探测防御                         | IPv6 适配度与性能效率                           | T：公开审查打击现状                                | IPv6:443 综合推荐指数              |
| :--------------------------------- | :------ | :------------------------- | :-------------------------------- | :------------------------------- | :-------------------------------------- | :---------------------------------------- | :--------------------------- |
| **VLESS + REALITY + Vision**       | TCP/443 | TLS 1.3 + 真实证书伪造           | ★★★★☆<br>(uTLS模拟，需挑好SNI)          | ★★★★★<br>(直接透传到真实目标站)            | ★★★★★<br>内核TCP，开销极低；需注意 SNI 需具备 AAAA 记录 | 中-高<br>(认知度最高，uTLS指纹正被持续收紧建模)             | ★★★★★<br>（TCP 443 首选主力）      |
| **NaiveProxy**                     | TCP/443 | TLS 1.3 (Chromium 真实栈)     | ★★★★★<br>(无伪造指纹，纯正 Chrome 行为)     | ★★★★★<br>(Caddy 前端反代，完美合法响应)     | ★★★☆☆<br>单核开销偏高，内存需求大，弱网性能受 TCP 限制      | 中<br>(长期被关注，但在 443 纯净环境极难定性)              | ★★★★☆<br>（高合规/抗审查顶格备用）       |
| **VLESS + XHTTP + TLS + Fallback** | TCP/443 | TLS 1.3 + 自有域名证书           | ★★★★☆<br>(流/包分离，标头 Padding)       | ★★★★★<br>(Nginx/Caddy 真实回落站点)    | ★★★★☆<br>原生多路复用，CDN 亲和度极好，CPU 适中        | 中-低<br>(Xray 推荐替代 WS，特征比旧协议更散)            | ★★★★☆<br>（自备域名下的新一代稳态主力）     |
| **Trojan + TLS + Fallback**        | TCP/443 | TLS 1.3 + 自有域名证书           | ★★★☆☆<br>(长连接包长时序存在统计特征)          | ★★★★★<br>(非法请求回落真实站)             | ★★★★☆<br>成熟稳定，内存极低，跑满千兆带宽               | 中<br>(老牌协议，对非标准行为的流量分析趋严)                 | ★★★☆☆<br>（传统标准 HTTPS 备用）     |
| **AnyTLS**                         | TCP/443 | TLS 1.3 + 动态填充协议           | ★★★★☆<br>(动态 Padding Scheme 消除包长) | ★★★★☆<br>(依赖外部 Fallback 机制)      | ★★★★☆<br>sing-box 原生低开销，多会话复用           | 低-中<br>(新协议，公开规则少，官方承认默认需演进)              | ★★★★☆<br>（异构 TCP/443 优选）     |
| **ShadowTLS v3 + SS2022**          | TCP/443 | TLS 1.3 外层 + AEAD 内层       | ★★★★☆<br>(借用真实 TLS 握手)            | ★★★★☆<br>(V3 challenge-response) | ★★★★★<br>Rust 编写，极低延迟与极低内存              | 中<br>(有针对 ServerFinished 等报文长度差异的分析)      | ★★★★☆<br>（轻量小内存 VPS 优选）      |
| **Hysteria 2 (默认 HTTP/3 模式)**      | UDP/443 | TLS 1.3 (QUIC) + HTTP/3 伪装 | ★★★★☆<br>(天然像正常 IPv6 H3 流量)       | ★★★★☆<br>(未认证请求回落伪装网页)           | ★★★★★<br>Brutal/BBR 极强抗丢包；**需防 MTU 黑洞** | **高**<br>(USENIX 2025 证实 QUIC SNI 专项解密阻断) | ★★★★★<br>（UDP 443 速度与弱网之王）   |
| **Hysteria 2 (+Salamander/Gecko)** | UDP/443 | QUIC + 随机化混淆/分片            | ★★★★☆<br>(完全抹去 QUIC 头，呈高熵/碎片)     | ★★☆☆☆<br>(失去 H3 伪装，遇探针表现为非标 UDP) | ★★★★☆<br>碎片化导致 CPU 升高，抗丢包能力稍损           | **高**<br>(在 443 运行高熵 UDP 容易因行为异常被标记)      | ★★★☆☆<br>（443 端口不推荐；更适合高位端口） |
| **TUIC v5**                        | UDP/443 | TLS 1.3 (QUIC 原生)          | ★★★☆☆<br>(纯 QUIC 多路复用，无额外混淆)      | ★★★☆☆<br>(抗探针主要依赖认证丢弃)           | ★★★★★<br>0-RTT 握手，连接迁移对 IPv6 移动网极佳      | **高**<br>(受 QUIC SNI 过滤直接影响)              | ★★★☆☆<br>（UDP 备用方案）          |

四、 关键协议在 IPv6:443 下的实战技术细节与陷阱

1. VLESS + REALITY + Vision (TCP/443)

  - IPv6 专属核心要点：Target / SNI 必须拥有活跃的 AAAA 记录！
    许多新手直接套用 IPv4 时代的配置（例如选一个仅有 IPv4 的目标域名）。在 IPv6 连接中，如果客户端向你的 IPv6:443 发起
    ClientHello，SNI 为 target.com，而防火墙或主动探测系统查询发现 target.com 根本没有全球单播 IPv6
    地址（AAAA），或者你 VPS 服务端的 REALITY 回落 Dest 试图向该目标发起 IPv6 握手失败，伪装会在毫秒内被戳穿。
  - 优选目标： 必须选择全球权威、原生双栈支持、且全球 Anycast 的 IPv6 站点（如
    gateway.icloud.com、addons.mozilla.org、swdist.apple.com、www.microsoft.com）。
  - 抗封锁优势： REALITY 运行在 IPv6:443 是最完美的场景。因为正常浏览器向苹果、微软等大厂发起的 IPv6 请求就是
    [IPv6]:443，完全符合现实统计分布。

2. Hysteria 2（默认模式 vs 混淆模式）在 UDP/443

  - 为什么在 443 端口必须优先选择“默认 HTTP/3 伪装”？
    在非 443 端口上，由于 HTTP/3 极少见，使用 Salamander（高熵化）或 Gecko（分片）是合理的；但在 IPv6:443 上，标准
    HTTP/3 本来就是大势所趋（Google、Cloudflare、Meta 等超过一半的 IPv6 流量都是 UDP/443 的
    QUIC/H3）。此时保持标准的 HTTP/3 伪装、配置合法的证书与伪装站点，远比在 443 端口传输一堆“无法被识别为 QUIC 的高熵无序 UDP
    碎片”更加合规自然。
  - IPv6 MTU 灾难防御：
    IPv6 头部占用 40 字节，比 IPv4 多 20 字节。移动蜂窝网络（如中国移动/电信 5G SA）的 IPv6 MTU
    经常被限制在 1280–1340。若 QUIC 发送时未处理 MTU，导致包长超过底层限制，数据包将被直接丢弃且无 ICMP 报错。
    配置建议： 客户端与服务端必须确认底层 UDP Socket 的 MTU 探测策略，必要时在客户端限制初始包大小（或保持默认 QUIC 最小 1200
    字节握手协商），防止握手阶段直接死锁。

3. VLESS + XHTTP + TLS + Fallback (TCP/443)

  - 新一代 HTTP 拟态： Xray 的 XHTTP 模块专为解决现代审查而生。在 IPv6:443 下，通过自有域名（配置 AAAA
    记录）申请真实证书，配合 HTTP/2 多路复用和上下行流量分离开关（packet-up / stream-up），外部审查设备看到的是完完全全的合法
    IPv6 HTTP/2 交互。未授权访问直接 fallback 回落到本机绑定的 Nginx/Caddy 静态站，抗主动探测能力达到顶峰。

4. NaiveProxy (TCP/443)

  - 指纹防御天花板： 在 IPv6 环境下，使用 Chromium 网络栈的 NaiveProxy 发起的流量与本地 Chrome 浏览器通过 IPv6
    浏览外网的表现无任何二进制或时序上的偏差。如果所在的省份或运营商对 uTLS 的各类模拟方案（Vision /
    REALITY）进行了深度行为建模检测，NaiveProxy 是最强的绝对屏障。

五、 IPv6 独有抗封锁武器：/64 子网动态调度机制

这是 IPv4（单 IP 昂贵、换 IP 需付费）完全无法复制的降维打击手段：

1. 原理

云服务商（如 Hetzner、Vultr、Linode、OVH 等）通常免费为实例路由一个 /64 IPv6 前缀（包含
18,446,744,073,709,551,616 个地址）。通过 Linux 内核的 Any-IP 特性：

# 将整个 /64 网段绑定到本地 loopback，无需逐个配置 IP
ip route add local 2001:db8:1234:5678::/64 dev lo

此时，你的 Xray / sing-box 服务端监听在 [::]:443，即可响应发往这个 /64 内任意一个 IPv6 地址的 443 端口请求。

2. 防御效果

  - 规避 IP 被动黑名单： 客户端无需固定连接某一个 IP。可以在客户端配置脚本，每天、每小时，甚至每次重连时随机生成该 /64
    网段内的一个主机后缀访问 443 端口。
  - 针对主动探测的终极免疫： GFW 的主动探测系统通常在捕获某个会话后，回放或扫描该会话的目标 IP。如果每次会话使用的 IPv6
    主机地址都随机变化，GFW 的探测探针只会打到一个“已被客户端弃用”的地址上，探测数据库完全失效。
  - 成本为零： 即使某个 /128 遭到单点封锁，客户端在毫秒级内自动切换到同网段下一 IP，业务丝毫不中断。

六、 总结收敛：IPv6:443 最佳落地实践拓扑

在 IPv6:443 场景下，最成熟、健壮且互不干扰的部署形态为 “单机双协议、全双工标准端口占用”：

                              客户端 (双栈/IPv6 优先)
                                         │
                   ┌─────────────────────┴─────────────────────┐
                   │                                           │
         TCP/443 (标准 HTTPS)                        UDP/443 (标准 HTTP/3)
                   │                                           │
                   ▼                                           ▼
       [ 主力 TCP 拟态通道 ]                        [ 极速 UDP 传输通道 ]
      VLESS + REALITY + Vision                      Hysteria 2 (默认模式)
                   │                                           │
        (备用: XHTTP / AnyTLS)                        (备用: TUIC v5)
                   │                                           │
                   └─────────────────────┬─────────────────────┘
                                         │
                             服务端单一 IPv6 监听 (或 /64 Any-IP 池)

实施检查清单（Checklist）

1.  真实性验证： REALITY 的 SNI 目标域名必须使用 dig -t AAAA <domain> 验证其具备有效 IPv6 地址，严禁使用
    IPv4-only 站点。
2.  端口复用： 系统内核放行 sysctl -w net.ipv4.ip_forward=1 与
    net.ipv6.conf.all.forwarding=1，确保防火墙同时放行 TCP 443 与 UDP 443。
3.  避免 MTU 黑洞： 若在移动蜂窝网络（5G/LTE）下使用 Hysteria 2 / TUIC，如遇高丢包或打不开大网页，在客户端限制 QUIC
    发送尺寸或确认网络 MTU（通常保证 payload 适应 1280–1350 之间）。
4.  回落兜底： 若采用自有域名（XHTTP / Trojan / Naive），务必在服务端配置合法静态 Web 页面（如
    Caddy/Nginx），确保探测器直接访问 https://[你的IPv6]:443 时呈现真实正常站点。
    
    
IPv6 场景下：443 与非 443 代理协议/组合的系统比较
先说明边界：我无法实时联网检索或验证 2026-09-18 当日的新增信息；以下基于你提供文件中列出的公开资料、截至知识范围内的协议设计与公开研究脉络整理。文件里引用的 USENIX Security 2025 QUIC 审查研究、GFW Report 关于 TLS 类工具与 2025-08-20 TCP/443 RST 异常的材料，应作为判断“当前打击面”的高权重证据；GitHub issue、社区测速和单一地区报告只能视为线索，而非通用封锁率。
另外，这里讨论的是网络连通性与隐私/抗审查工程，不代表任何地区的法律意见；部署时应遵守适用法律和服务商条款。
----------------------------------------
1. 先给结论：不要寻找“最抗封协议”，而要建设异构故障域
你的条件分为两类：
 1. 仅 IPv6 的服务器：客户端通过 IPv6 直连服务端；
 2. IPv6 或 IPv6/NATv4：服务端有 IPv6，可能还通过 NAT IPv4 或双栈路径提供服务；
 3. 同时要覆盖：
 * TCP/443：正常 HTTPS 背景最强；
 * UDP/443：正常 HTTP/3/QUIC 背景最强；
 * 非 443 TCP/UDP：用于规避端口级异常、QoS、单端口封锁或做异构备用。
最合理的整体思路不是部署十几个协议，而是至少覆盖四个不同故障域：
A. TCP / 443 / TLS-或REALITY伪装
B. TCP / 非443 / TLS会话或真实HTTP承载
C. UDP / 443或高端口 / QUIC
D. UDP / 非QUIC、WireGuard-derived 或自定义加密流量
推荐的工程化最小矩阵：
TCP/443：
 VLESS + REALITY + Vision
 或 NaiveProxy + HTTP/2 + 真实前端
TCP/8443 或 2053：
 AnyTLS
 ShadowTLS v3 + Shadowsocks 2022
 Trojan + TLS + 真实 fallback
 VLESS + XHTTP + TLS
UDP/443：
 Hysteria2 默认 HTTP/3 伪装
 或 TUIC v5
UDP/20000-50000：
 Hysteria2 + Port Hopping
 可按实际网络测试 Salamander/Gecko
 或 AmneziaWG
其中：
 * 443 TCP 主线优先看：REALITY、Naive、XHTTP+TLS；
 * 非443 TCP 备用优先看：AnyTLS、ShadowTLS+SS2022、Trojan+fallback、XHTTP+TLS；
 * UDP 性能通道优先看：Hysteria2；
 * QUIC 之外的 UDP 后手优先看：AmneziaWG；
 * 研究性异构后手：Mieru、Restls+SS。
----------------------------------------
2. 判断维度：不要把“抗封锁”压缩成一个星级
建议采用以下七个维度。
维度 含义 加密安全 是否使用现代 TLS 1.3、QUIC/TLS 或 AEAD；是否有前向保密、重放防护等 P：被动识别防御 对 TLS/QUIC 指纹、包长、熵、时序、连接行为、TLS-in-TLS 特征的抵抗力 A：主动探测防御 探测者发送错误握手、重放、HTTP 请求、TLS 探针时，服务端是否能表现得像正常服务 R：端口/IP 韧性 面对单端口限速、端口封锁、IP+端口封锁、短时端口异常的恢复能力 E：性能效率 RTT、吞吐、弱网丢包能力、CPU/RAM、并发能力 D：异构容灾价值 与已有 VLESS+REALITY、Hy2 等路线是否真正不同 T：公开打击证据 是否已有高质量论文、GFW Report 或重复出现的可信现场报告表明其被重点识别/阻断
特别注意：
> T 高不等于协议弱。
> 它表示“已有较强公开证据表明审查方研究或打击过这类流量”。
例如 QUIC 的 T 很高，不是因为 Hysteria2/TUIC 一定不能用，而是因为已有公开论文证明 GFW 可解析 QUIC Initial 并按 SNI 过滤。
----------------------------------------
3. IPv6 的真实作用与限制
3.1 IPv6 不是天然免封锁
IPv6 只改变地址空间、路由和运维弹性，不会自动改变以下事实：
IPv6 TCP/443：
 仍可解析 TLS ClientHello
 仍可进行 SNI 筛选
 仍可基于 TLS 行为分类
 仍可实施主动探测
 仍可按 IPv6 地址或前缀阻断
IPv6 UDP/443：
 仍可解析 QUIC Initial
 仍可实施 QUIC SNI 审查
 仍可按五元组、地址或端口丢包
所以“IPv6+443”并不比“IPv4+443”天然更隐身；它的优势主要在于：
 * 直连路径可能更好；
 * 部分网络环境 IPv6 出口策略不同；
 * 如果服务商给出可路由 /64，理论上有更多地址调度空间；
 * TCP/443 与 UDP/443 可以在同一 IPv6 地址并行部署。
3.2 /64 地址池不是自动“无限换 IP 抗封”
文件里关于 /64 地址池的思路有工程价值，但应谨慎使用。
正确理解：
一个 IPv6 /64
 ≠
必然可以无限换 /128 而不被关联
一个 IP 被封
 ≠
同一 /64 一定安全
审查或运营商可以按不同粒度处理：
 * 单一 /128；
 * 小前缀；
 * 整个 /64；
 * ASN、机房、路由段；
 * 域名/SNI；
 * 行为关联后的地址集合。
因此 IPv6 地址轮换可以作为恢复和测试手段，但不能当成协议伪装的替代品，更不应高频随机跳 IP；异常轮换行为本身可能增加统计异常。
3.3 IPv6 的主要工程风险：MTU 与 PMTU 黑洞
IPv6 中间路由器不负责分片，依赖源端和 ICMPv6 Packet Too Big 进行路径 MTU 发现。跨境移动网络、PPPoE、隧道环境中，ICMPv6 处理不良可能造成：
小请求能连
大网页打不开
QUIC 握手卡顿
视频/下载断续
尤其受影响的通常是：
 * Hysteria2；
 * TUIC；
 * Naive 的 HTTP/3；
 * MASQUE；
 * UDP 上的 AmneziaWG；
 * 任何在 IPv6 上传较大 UDP 包的协议。
因此，IPv6 下的 QUIC 不稳定时，第一件事不是认定“被封”，而应先排查：
路径 MTU
MSS
UDP payload
ICMPv6 Packet Too Big
防火墙
回程路由
----------------------------------------
4. 443 端口：适合 IPv6 的协议与组合
443 的最大优势是协议语义自然：
TCP/443 = HTTPS
UDP/443 = HTTP/3 / QUIC
但也意味着它是高关注端口，可能遭遇更强 TLS、QUIC、主动探测和端口策略审查。
----------------------------------------
4.1 TCP/443：VLESS + REALITY + Vision
结构
IPv6
 ↓
TCP/443
 ↓
REALITY / TLS-like handshake
 ↓
VLESS
 ↓
XTLS Vision（可选 flow）
 ↓
代理流量
优势
 * 不需要自有域名或公网 CA 证书；
 * 443 上外部行为更符合 HTTPS 的常见分布；
 * 对未授权连接可以回落到目标站，主动探测处理能力较强；
 * Vision 在性能和 TLS-in-TLS 处理上具有实际价值；
 * TCP 对 IPv6 MTU 问题通常比 UDP/QUIC 更宽容；
 * Xray 生态、客户端支持、部署经验比较成熟。
局限与风险
 1. REALITY 不等于真实浏览器。
 它通常依赖 uTLS 类指纹模仿；ClientHello 像 Chrome，不代表 TCP、TLS record、HTTP 行为、连接生命周期都像真实 Chrome。
 2. 443 更自然，不等于不被识别。
 文件中提到的 2026 年 REALITY/XHTTP 现场阻断或疑似主动探测报告，不能证明全面协议破解，但足以说明 REALITY 已进入持续对抗阶段。
 3. 目标域名和端口逻辑要自洽。
 在 TCP/443 使用 REALITY 时，目标站点也使用 443 的行为最自然；如果服务端放在 8443，REALITY 本身不失效，但“常见 HTTPS 端口一致性”会下降。
 4. IP 信誉仍然是独立风险。
 即使握手像正常 TLS，长时间高带宽、固定模式、多用户共享 IP 仍可能导致 IP/端口层面的处理。
评价
指标 评价 加密 高：TLS/REALITY 体系 P：被动识别 高，但不等于真实浏览器全栈 A：主动探测 高 E：性能 很高，尤其 Vision 场景 IPv6 适配 很高 T：公开打击证据 中到高，现实对抗迹象多 D：异构价值 若已有同类 REALITY，新增价值低；与其他协议并存时高
定位
> TCP/443 的成熟主力。
> 适合作为一条核心 TCP 通道，但不应成为唯一协议栈。
----------------------------------------
4.2 TCP/443：NaiveProxy + HTTP/2 + Caddy/前端
结构
IPv6
 ↓
TCP/443
 ↓
真实 TLS
 ↓
HTTP/2 CONNECT
 ↓
Caddy / frontend
 ↓
NaiveProxy
核心区别
Naive 的重要价值不只是“Chrome 指纹”，而是尽量使用 Chromium 网络栈和 HTTP/2 行为：
真实 TLS 行为
+
HTTP/2 多路复用
+
浏览器网络栈
+
padding / fragmentation
+
frontend / application fronting
它比单纯 uTLS 模拟更接近真实 Web 会话模型。
优势
 * TLS/HTTP/2 指纹路线很强；
 * 有真实 Caddy/前端时，主动探测返回普通 Web 行为；
 * 很适合自有域名、真实证书、真实站点；
 * 在“像普通浏览器 HTTPS”这个目标上具有独特优势；
 * TCP/443 与 IPv6 环境天然匹配。
局限
 * 仍然存在 SNI、域名、IP 信誉与访问模式问题；
 * 更依赖客户端版本跟随 Chromium 演进；
 * 部署与维护成本高于 REALITY；
 * 对低内存 VPS、弱 CPU 的压力通常高于 VLESS+REALITY；
 * HTTP/3/QUIC 路线不宜因为“更现代”就盲目启用，TCP/H2 通常是更稳妥的主线。
评价
指标 评价 加密 高：标准 TLS P：被动识别 很高，尤其真实 Chromium/H2 路线 A：主动探测 很高，前端和真实站点配置得当时 E：性能 中高，CPU/RAM 成本相对高 IPv6 适配 很高 T：公开打击证据 中，存在地区/社区报告但缺统一测量 D：异构价值 很高
定位
> 自有域名与真实站点条件下，最有价值的 TCP/443 异构路线之一。
> 不是“绝对无敌”，但和 REALITY 的暴露面明显不同。
----------------------------------------
4.3 TCP/443：VLESS + XHTTP + TLS + Fallback
结构
IPv6
 ↓
TCP/443
 ↓
真实 TLS 证书
 ↓
HTTP/1.1 / HTTP/2 的 XHTTP
 ↓
VLESS
 ↓
Nginx/Caddy fallback
XHTTP 也可能搭配 REALITY，但若目标是异构容灾，重点应区分：
VLESS + XHTTP + TLS
与：
VLESS + XHTTP + REALITY
前者与主力 REALITY 的差异更大。
优势
 * 比传统 WS/gRPC 更现代，支持更多 HTTP 传输模式；
 * 支持 header padding、连接复用、上下行不同模型；
 * 自有域名+真实证书+真实网站时主动探测处理比较自然；
 * 可以适应反向代理/CDN 等 Web 生态；
 * 对 TCP/443 和 IPv6 均适合。
局限
 * XHTTP 仍是快速演进中的 transport；
 * 配置复杂度、版本兼容性与实现变化需要持续跟踪；
 * “使用 HTTP/2”不等于“行为一定像真实浏览器”；
 * 如果与 REALITY 一起使用，虽然 transport 有差异，但仍共享 Xray/VLESS/REALITY 生态，异构性有限；
 * H3 模式重新进入 UDP/QUIC 审查面，不应和 TCP/H2 混为一谈。
评价
指标 评价 加密 高：TLS 或 REALITY P：被动识别 高到很高，取决于模式和前端 A：主动探测 高，取决于真实 fallback E：性能 高；多路复用适合大量小连接 IPv6 适配 很高 T：公开打击证据 中；实现和现场报告需持续关注 D：异构价值 TLS 版高；REALITY 版中等
定位
> 有域名、有真实站点、希望采用现代 HTTP 承载层时的优选。
> 用于 TCP/443 或非443均可；若做异构备用，优先是“XHTTP+标准 TLS”，不是“XHTTP+REALITY”。
----------------------------------------
4.4 TCP/443：Trojan + TLS + 真实 Web fallback
结构
IPv6
 ↓
TCP/443
 ↓
TLS 1.3
 ↓
Trojan auth
 ↓
代理
错误请求
 ↓
Nginx/Caddy/真实网站
优势
 * 老牌成熟，客户端覆盖广；
 * TLS 与真实站点 fallback 的主动探测防护理念简单而可靠；
 * 资源开销低；
 * 自有域名和证书环境中部署直观；
 * 在 TCP/443 与 IPv6 下很自然。
局限
 * 外层是标准 TLS，但长期代理行为不自动等同普通浏览行为；
 * 老协议、研究较充分，不能再将其视为“新鲜未知”；
 * 如果内部再大量访问 HTTPS，可能出现 TLS-in-TLS 统计问题；
 * 单独使用时对包长、时序和连接行为的可塑性不如 AnyTLS/Restls/XHTTP。
评价
指标 评价 加密 高 P：被动识别 中高 A：主动探测 很高，正确 fallback 时 E：性能 高 IPv6 适配 很高 T：公开打击证据 中，高认知老协议 D：异构价值 中高
定位
> 传统、稳定、易维护的 TLS 备用；适合真实建站附带的代理服务。
----------------------------------------
4.5 TCP/443：AnyTLS + TLS
结构
IPv6
 ↓
TCP/443 或 TCP/8443
 ↓
TLS
 ↓
AnyTLS authentication
 ↓
session multiplex
 ↓
stream + padding / waste
核心价值
AnyTLS 不只是“另一个 TLS 代理”，它把以下能力放进协议设计：
 * session multiplex；
 * stream；
 * padding；
 * 可改变的 padding scheme；
 * 服务端可下发新的 padding 策略。
这使它适合应对“固定流量特征被识别后需要更新”的问题。
优势
 * TCP/TLS 路线，与 QUIC 完全不同；
 * 对 TLS-in-TLS、长度模式、会话复用有专门设计；
 * 可配置、可升级的 padding 机制；
 * sing-box 等生态支持提高；
 * 非443尤其自然，443 也可使用。
局限
 * TLS ClientHello/ServerHello 是否像 Chrome，不是 AnyTLS 单独解决的；
 * 新协议，长期、大规模、同条件的公开对抗测量不足；
 * fallback/主动探测效果取决于具体实现与前端配置；
 * “公开打击少”不能推导为“难以识别”。
评价
指标 评价 加密 高：TLS P：被动识别 高，尤其包长与会话模式 A：主动探测 中高到高，依赖 fallback 设计 E：性能 高 IPv6 适配 很高 T：公开打击证据 低，主要因为较新 D：异构价值 很高
定位
> 最值得部署为 TCP 非443异构备用之一。
> 在 443 也可用，但其相对优势在 8443/2053 这类 TCP TLS 备用入口更明显。
----------------------------------------
4.6 TCP/443：ShadowTLS v3 + Shadowsocks 2022
结构
IPv6
 ↓
TCP/443 或 TCP/8443
 ↓
ShadowTLS v3
 ↓
Shadowsocks 2022 AEAD
 ↓
代理流量
设计价值
 * ShadowTLS 提供真实 TLS 外观、认证与探测处理；
 * SS2022 提供独立 AEAD 代理层；
 * 外层 TLS 与内层代理数据层相互解耦；
 * 与 Xray/REALITY 路线明显不同；
 * Rust 实现通常对小内存 VPS 比较友好。
优势
 * 异构性高；
 * 性能高、资源消耗低；
 * ShadowTLS v3 面向主动探测演进；
 * SS2022 生态与客户端支持较广；
 * 非443端口适配非常自然。
局限
 * 已存在关于握手长度、ServerFinished 等潜在检测向量的讨论；
 * 真实 TLS 外壳不代表长期代理数据层完全像普通 Web；
 * SS2022 本身不是 Web camouflage；
 * 没有足够公开数据证明它在所有网络和敏感期都有稳定优势。
评价
指标 评价 加密 高：TLS + AEAD P：被动识别 高 A：主动探测 高 E：性能 高到很高 IPv6 适配 很高 T：公开打击证据 中 D：异构价值 很高
定位
> 低配 VPS、Alpine/OpenRC、需要第二套独立 TCP 栈时非常有价值。
----------------------------------------
4.7 TCP/443：Restls + SS
核心思路
Restls 的目标不是单纯套 TLS，而是控制 TLS 交互早期的：
包长
响应次数
padding
时序
前几组 record 行为
它适合研究和对抗固定的 TLS-in-TLS、首包长度、早期时序签名。
优势
 * 可编程地修改初期交互行为；
 * 对“首几个包就被识别”的假设针对性强；
 * 和 REALITY、Naive、AnyTLS 的设计哲学明显不同；
 * 作为冷备或实验线路，异构价值高。
局限
 * 生态、客户端支持、运维成熟度弱于主流；
 * 早期包伪装不代表长期大流量行为可伪装；
 * 配置越自由，越需要真实测试，错误脚本可能形成更明显的新指纹；
 * 性能通常会因行为控制与额外交互受到影响。
定位
> 研究型、冷备型方案，不建议替代主力。
----------------------------------------
4.8 TCP/443：Mieru
结构
IPv6
 ↓
TCP/443 或任意端口
 ↓
Mieru 自定义 AEAD 协议
 ↓
padding / fragmentation / low entropy / jitter
核心特点
Mieru 不走“伪装 HTTPS”路线，而是：
XChaCha20-Poly1305
+
时间相关密钥
+
随机填充
+
动态分段
+
低熵模式
+
心跳抖动
+
流量模式控制
优势
 * 不依赖证书、域名、SNI、Web 前端；
 * TCP/UDP 均可；
 * IPv6 支持明确；
 * 与 TLS、QUIC 完全不同；
 * 低熵和分片策略是对统计识别的正面应对；
 * 异构价值极高。
局限
 * 放在 TCP/443 上时，它不是 TLS/HTTPS，这一点非常重要；
 * “443 上出现非 TLS 高熵/非标准流量”可能本身就是异常；
 * 有历史上被识别或阻断的项目自述，说明它不是未被关注；
 * 公开中国实测数据不足；
 * 低熵模式会牺牲带宽、CPU、有效 MTU。
定位
> 更适合非443或作为极端情形下的异构冷备；不应将“TCP/443+Mieru”当成标准 HTTPS 伪装。
----------------------------------------
5. UDP/443：QUIC 协议族
最关键的背景是：
> GFW 已经能够解密 QUIC Initial，读取内层 TLS ClientHello/SNI，并按 QUIC 专用规则阻断。
因此：
Hysteria2 ≠ 天然隐身
TUIC ≠ 天然隐身
Juicity ≠ 天然隐身
XHTTP/H3 ≠ 自动绕过 QUIC 审查
它们的主要优势应理解为：
弱网性能
+
多路复用
+
UDP relay
+
连接迁移
+
不同于 TCP 的故障域
而不是“QUIC 加密所以 DPI 看不见”。
----------------------------------------
5.1 UDP/443：Hysteria2 默认 HTTP/3 伪装
结构
IPv6
 ↓
UDP/443
 ↓
QUIC / TLS 1.3
 ↓
Hysteria2
 ↓
TCP / UDP relay
优势
 * 高 RTT、高丢包、移动网络下性能通常强；
 * QUIC 无 TCP 队头阻塞；
 * 对 UDP、实时应用、下载、游戏等场景更友好；
 * 同一 IP 可以同时运行 TCP/443 与 UDP/443；
 * 默认模式追求看起来像 HTTP/3；
 * 有伪装站点/认证失败后的回落思路；
 * Port Hopping 可改善单端口 UDP QoS/封锁问题。
局限
 * QUIC Initial/SNI 审查已有高质量公开证据；
 * QUIC 的失败模式与 TCP 不同，尤其 IPv6 MTU、移动网络、UDP QoS；
 * 用户态 QUIC 通常比内核 TCP 更耗 CPU；
 * “默认 HTTP/3 外观”和“Salamander/Gecko 高混淆”是不同策略，不能叠加后简单称为更自然。
评价
指标 评价 加密 高：QUIC/TLS 1.3 P：被动识别 默认 H3 为中高；取决于环境 A：主动探测 中高 R：端口限制 高，Port Hopping 针对单端口有效 E：性能 很高，弱网强 IPv6 适配 高，但 MTU 敏感 T：公开打击证据 高 D：异构价值 高，相对 TCP 栈
定位
> UDP/443 的性能主力和 TCP 故障域备用。
> 不应被定位成“最隐蔽的协议”。
----------------------------------------
5.2 Hysteria2 + Port Hopping
正确认知
Port Hopping 的作用是：
IP + 单 UDP 端口
被限速/封锁
 ↓
切换到同一服务端的其他 UDP 端口
它不解决：
整个 UDP 被封
QUIC 被识别
IP 被封
SNI 被封
因此：
能解决 不能解决 单端口 QoS 全 UDP 阻断 IP+端口封锁 IP 级封锁 某端口黑洞 QUIC Initial/SNI 识别 某端口被限速 域名/SNI 处理
IPv6 与端口范围
若服务端在 IPv6 下使用高端口范围，应确认：
 * 防火墙 IPv6 规则放行；
 * nftables/iptables 对 IPv6 生效；
 * 云厂商安全组放行 UDP 范围；
 * 客户端网络允许出站高 UDP 端口；
 * NAT64、企业网、校园网不会过滤该范围。
定位
> 非443 UDP 的端口级韧性工具，不是 DPI 隐身工具。
----------------------------------------
5.3 Hysteria2 + Salamander
Salamander 的核心是把 QUIC 包外形改变为随机字节，使其不再是标准可解析 QUIC。
优点
 * 可能避开针对标准 QUIC/HTTP3 的直接解析；
 * 对抗某些明确依赖 QUIC 报文结构的设备；
 * 与端口跳跃一起可以增强单端口受限情况下的适应性。
风险
 * 流量不再像普通 HTTP/3；
 * 在 UDP/443 上，“不可解析的高熵 UDP”未必比标准 QUIC 更自然；
 * 高熵随机 UDP 也可以成为统计异常；
 * 失去部分“HTTP/3 正常背景”优势。
端口建议
UDP/443：
 优先考虑标准 H3 外观；
 Salamander 需要根据本地实测决定。
UDP/20000-50000：
 Salamander + hopping 的逻辑更自洽。
----------------------------------------
5.4 Hysteria2 + Gecko
Gecko 是更进一步的握手分片、随机大小与 padding 路线。
价值
 * 目标是削弱 QUIC 长头握手包的长度和几何特征；
 * 和 Salamander 一样，不应被理解为“更像 HTTP/3”；
 * 更接近“隐藏 QUIC 形状”的策略。
风险
 * 新、实验性、生态支持和实际对抗测量较少；
 * 额外分片/padding 增加 CPU、时延和 MTU 复杂度；
 * IPv6 移动路径、较小 MTU 下尤其需实测；
 * 不应在没有测试的前提下作为唯一 UDP 入口。
定位
> 实验性 QUIC 对抗策略；更适合高端口或特定受限网络实测后的备用。
----------------------------------------
5.5 UDP/443：TUIC v5
特点
QUIC
+
TLS
+
0-RTT
+
multiplex
+
TCP relay
+
UDP relay
优势
 * 高性能；
 * QUIC 原生；
 * TCP/UDP relay；
 * 连接迁移等对移动网络有价值；
 * 生态已经比较成熟。
局限
 * 与 Hy2 同属 QUIC/TLS Initial 审查面；
 * 不能视为“UDP 被封后的备用”；
 * 相对 Hy2，端口跳跃、混淆工具链和高丢包激进调优往往不是主要强项；
 * 主动探测和伪装策略通常不如带真实 Web fallback 的 TCP TLS 路线自然。
定位
> 第二个 QUIC 实现，而不是第二个独立网络栈。
> 适合 Hy2 具体实现或兼容性问题时切换，不适合替代 TCP/TLS 异构灾备。
----------------------------------------
5.6 UDP/443：Juicity
Juicity 也是 QUIC/TLS 路线，协议简洁、IPv4/IPv6/TCP/UDP 支持明确。
定位
 * 可作为 QUIC 第三实现；
 * 生态和客户端覆盖通常弱于 Hy2/TUIC；
 * 同样继承 QUIC 审查面；
 * 更适合实验和备用，不建议把它作为唯一高可用方案。
----------------------------------------
6. 非443端口：适合 IPv6 与 IPv6/NATv4 的方案
非443的作用不是“天然更安全”，而是降低和 TCP/443、UDP/443 同时失效的风险。
真实价值包括：
端口级异常时的备用
避开某些单端口 QoS
与主线路形成不同监听面
在特定网络中绕开 443 特定策略
服务端多入口与故障隔离
代价是：
非443 TLS 长连接更不常见
企业/校园/酒店可能只放行 80/443
随机高端口上的加密流更容易显得异常
----------------------------------------
6.1 非443 TCP：端口选型
推荐的 Web/TLS 替代端口
8443
2053
2083
2087
2096
它们在全球 Web/CDN 生态中存在真实 HTTPS 用途。
但不要误解：
> 它们不是“在中国一定被当成普通 HTTPS”的白名单端口。
实际优先级：
用途 推荐端口 TCP + TLS 备用 8443 首选，其次 2053/2083/2087/2096 Hysteria2 Port Hopping 20000-50000 等范围 冷备独立协议 18888、30000+ 等，但需实测客户端网络出站 不建议伪装使用 22、3306、6379、53 等业务语义很强的端口
----------------------------------------
6.2 TCP/8443：AnyTLS
这是最值得优先考虑的非443 TCP 组合之一。
IPv6 或 IPv6/NATv4
 ↓
TCP/8443
 ↓
TLS
 ↓
AnyTLS
适用原因
 * AnyTLS 不依赖端口 443 的特殊逻辑；
 * TLS+padding+session multiplex 与主力 REALITY 有明显差异；
 * 8443 在 Web 生态中仍有一定合理性；
 * 对小 VPS 的开销通常可控；
 * 适合长期作为“主力失效时的 TCP 第二入口”。
推荐定位
> 非443 TCP 第一备用候选。
----------------------------------------
6.3 TCP/8443 或 2053：ShadowTLS v3 + SS2022
IPv6 或 IPv6/NATv4
 ↓
TCP/8443 或 2053
 ↓
ShadowTLS v3
 ↓
SS2022
适用原因
 * 与 REALITY、XHTTP、Naive 都不同；
 * 外层 TLS 和内层 AEAD 分离；
 * Rust 生态下性能和资源消耗较好；
 * ShadowTLS 可将监听端口和回源 TLS 目标解耦；
 * 很适合小内存 VPS、Alpine/OpenRC 类环境。
推荐定位
> 非443 TCP 的成熟异构备用。
----------------------------------------
6.4 TCP/8443：Trojan + TLS + fallback
IPv6 或 IPv6/NATv4
 ↓
TCP/8443
 ↓
TLS
 ↓
Trojan
 ↓
正常 Web fallback
适用原因
 * 逻辑非常成熟；
 * 非认证探测可以返回真实网站；
 * 易维护，客户端覆盖好；
 * 适合已经运行 Nginx/Caddy 网站的服务器。
注意点
在非443上，必须让站点和端口行为尽量自洽：
https://example.com:8443
应能返回合理页面，而不是只有代理服务无响应。
推荐定位
> 传统稳健备用；比 AnyTLS/ShadowTLS 更易维护，但新增抗识别价值略低。
----------------------------------------
6.5 TCP/8443：VLESS + XHTTP + TLS
IPv6 或 IPv6/NATv4
 ↓
TCP/8443
 ↓
真实 TLS
 ↓
XHTTP
 ↓
VLESS
优势
 * 具有 HTTP 承载层；
 * 可以部署真实域名、证书、fallback；
 * 与 TCP/443 的 REALITY 主力存在 transport 差异；
 * 适合 HTTP/2 代理、反代、CDN 兼容路线。
局限
 * 非443削弱“普通浏览器 HTTPS”的自然性；
 * XHTTP 版本演进快；
 * 如果只是把 VLESS+REALITY 从 443 搬到 8443，异构价值较小；如果采用标准 TLS + HTTP 模式，价值更高。
推荐定位
> 有域名、真实 Web 前端、需要 HTTP 生态备用时的候选。
----------------------------------------
6.6 TCP 非443：NaiveProxy + H2
技术上可行：
https://example.com:8443
Naive 的 Chromium/TLS/H2 优势仍在。
但其最强的“像正常浏览器访问网站”优势会因端口非443而有所折损：
Chrome 访问 example.com:443
比
Chrome 长期访问 example.com:8443
更常见
推荐定位
> 可以部署，但更适合作为 443 的高仿真线路，而不是非443的第一选择。
----------------------------------------
6.7 TCP 非443：Restls + SS
适合冷备和实验，不适合作为唯一生产备用。
原因不是协议设计差，而是：
 * 客户端/服务端生态较小；
 * 脚本化行为需要持续验证；
 * 维护门槛高；
 * 对端口异常度的缓解有限。
----------------------------------------
6.8 TCP 非443：Mieru
Mieru 在非443反而比放在 TCP/443 更自洽，因为它本来就不声称自己是 HTTPS。
IPv6 或 IPv6/NATv4
 ↓
TCP/18888 或其他端口
 ↓
Mieru AEAD + padding/fragment
适合场景
 * 不想依赖域名/证书；
 * 需要完全不同于 TLS/QUIC 的协议；
 * 作为极端情况的冷备；
 * 想研究低熵、动态填充、分片、流量模式策略。
局限
 * 随机高端口上的长期加密流量仍可能被统计识别；
 * 某些网络直接封禁高端口；
 * 公开对抗证据不足，不能把“新/冷门”当成抗封保证。
推荐定位
> 真正异构的研究型冷备。
----------------------------------------
7. WireGuard 与 AmneziaWG：IPv6、NATv4、非443下的定位
7.1 原生 WireGuard
原生 WireGuard 的加密、性能、IPv6 支持都非常好，但它固定的 UDP 握手和数据包形态可识别性强。
因此：
WireGuard/UDP/443
在“伪装成 HTTPS/HTTP3”角度并不自然。
结论
> 原生 WG 适合可信网络、企业 VPN、低审查环境；不应作为高审查环境下的主要伪装传输。
----------------------------------------
7.2 AmneziaWG
AmneziaWG 的价值在于对 WireGuard 的：
 * 包头；
 * 包长；
 * 包序列；
 * 时序；
 * 元数据；
 * 伪包；
 * 协议签名；
做额外混淆和随机化。
优势
 * 性能通常仍很高；
 * IPv6 与 NATv4 环境适配较好；
 * 可以在非443 UDP 端口运行；
 * 与 TLS 和 QUIC 明显不同；
 * 适合做真正的 UDP 非QUIC 后手。
局限
 * 它不是 HTTPS，也不是 HTTP/3；
 * 在 UDP/443 上出现“非标准 QUIC 的 UDP 流”可能不自然；
 * 俄罗斯 DPI 对抗经验不能直接推导到 GFW；
 * 中国公开长期测量不足；
 * 统计随机化如果过于固定，也可能形成新的 fingerprint。
端口建议
UDP/443：
 不作为首选，除非实测网络环境更适合。
UDP/高端口：
 更符合其“混淆 VPN”定位。
推荐定位
> Hysteria2/TUIC 之外的 UDP 异构备用。
----------------------------------------
8. Shadowsocks 2022 单独使用
IPv6 或 IPv6/NATv4
 ↓
TCP/UDP 任意端口
 ↓
SS2022 AEAD
优势
 * 极低 CPU；
 * 兼容性好；
 * TCP/UDP 都简单；
 * 在路由器、小设备上友好；
 * 适合内网、低审查或作为内层数据层。
局限
 * 裸 SS2022 是高熵加密流；
 * 没有 Web/TLS 外观；
 * 不是强主动探测伪装方案；
 * 在 TCP/443 尤其不自然；
 * 在高审查网络中不适合当唯一出口。
定位
> 轻量数据层或低优先级备用，不宜作为高审查环境主力。
----------------------------------------
9. 汇总矩阵：443 与非443全部主要组合
下面采用相对评价。它表示结构与公开证据下的工程判断，而不是通用封锁成功率。
组合 推荐端口 适用地址 加密 P：被动识别 A：主动探测 E：性能 R：端口韧性 T：公开打击证据 异构价值 VLESS + REALITY + Vision TCP/443 IPv6 TLS/REALITY 高 高 很高 中 中高 主力基准 VLESS + XHTTP + REALITY TCP/443 IPv6 TLS/REALITY/H2 高 高 高 中 中 中 VLESS + XHTTP + TLS + fallback TCP/443、8443 IPv6/IPv6+NATv4 TLS 高 高 高 中 中 高 NaiveProxy + H2 + Caddy TCP/443 IPv6 标准 TLS/H2 很高 很高 中高 中 中 很高 Trojan + TLS + fallback TCP/443、8443 IPv6/IPv6+NATv4 TLS 中高 很高 高 中 中 中高 AnyTLS + TLS TCP/443、8443 IPv6/IPv6+NATv4 TLS 高 中高 高 中 低 很高 ShadowTLS v3 + SS2022 TCP/443、8443、2053 IPv6/IPv6+NATv4 TLS + AEAD 高 高 高到很高 中 中 很高 Restls + SS TCP/443、8443 IPv6/IPv6+NATv4 TLS + AEAD 很高（设计上） 高 中 中 低 很高 Mieru TCP 非443优先 IPv6/IPv6+NATv4 AEAD 中高到高 中高 高 中 低 很高 SS2022 裸协议 非443 IPv6/IPv6+NATv4 AEAD 低到中 中 很高 中 中 中 Hysteria2 默认 H3 UDP/443 IPv6 QUIC/TLS 中高 中高 很高 中 高 高 Hysteria2 + hopping UDP/高端口范围 IPv6/IPv6+NATv4 QUIC/TLS 中高 中高 很高 高（仅端口层） 高 高 Hy2 + Salamander UDP/高端口优先 IPv6/IPv6+NATv4 QUIC/TLS + obfs 中高 中 高 高（配 hopping） 高 高 Hy2 + Gecko UDP/高端口优先 IPv6/IPv6+NATv4 QUIC/TLS + 分片/pad 中高到高 中 高 高（配 hopping） 高 高 TUIC v5 UDP/443、8443 IPv6/IPv6+NATv4 QUIC/TLS 中 中 很高 中 高 中 Juicity UDP/443、非443 IPv6/IPv6+NATv4 QUIC/TLS 中 中 高到很高 中 高 中 原生 WireGuard UDP/非443 IPv6/IPv6+NATv4 Noise/WG 低 低 很高 中 中高 中 AmneziaWG UDP/高端口优先 IPv6/IPv6+NATv4 WG + obfs 中高 中 很高 中高 中国证据不足 很高
----------------------------------------
10. “打击力度现状”应如何严谨理解
10.1 证据最强：QUIC/UDP 审查
对 QUIC 的公开证据最明确：
QUIC Initial
 ↓
可被解密
 ↓
读取 ClientHello/SNI
 ↓
QUIC


11. 继续：按审查链路逐阶段比较
比“协议排行榜”更有价值的方式，是观察一个连接在不同阶段可能暴露什么。
DNS/地址发现
 ↓
TCP SYN 或 UDP 首包
 ↓
TLS ClientHello / QUIC Initial
 ↓
ServerHello / 早期 record
 ↓
认证与首个代理请求
 ↓
HTTP/2 或 QUIC 帧行为
 ↓
长连接包长、时序、上下行比例
 ↓
重连、并发、流量规模
 ↓
主动探测、IP reputation、关联封锁
----------------------------------------
11.1 DNS、AAAA 与地址发现阶段
REALITY
通常可减少对自有证书和域名的依赖，但客户端仍可能暴露：
 * 目标 SNI；
 * 服务器 IP；
 * 连接时间与频率；
 * 目标 IP 与实际服务行为之间的关联。
Naive、XHTTP、Trojan、AnyTLS
通常依赖自有域名和证书，因此多出：
域名 → AAAA → IPv6 地址 → 443/8443
这一整条关联链。
优点是站点可真实建站；缺点是：
 * 域名可能被污染或列入名单；
 * AAAA 记录暴露服务端；
 * 证书、DNS、IP、SNI 可被关联；
 * CDN 可能降低源站暴露，但并不能自动消除域名侧风险。
Mieru、原生 SS、AmneziaWG
不一定需要域名和证书，减少了 DNS/SNI 依赖；但代价是：
没有 Web 生态外观
因此这是一个典型取舍：
路线 主要优势 主要代价 域名/TLS/Web 行为更自然 DNS、SNI、证书、域名信誉风险 无域名/自定义加密 少一层域名依赖 协议与统计行为更容易成为特征
----------------------------------------
11.2 TCP SYN 阶段
这一阶段协议尚未开始，因此主要暴露：
 * IP；
 * TCP 端口；
 * IPv6 前缀；
 * SYN/SYN-ACK 时间；
 * TCP options；
 * 是否接受连接；
 * 地址信誉。
443
优点：
IPv6:443
符合普通 Web 连接。
缺点：
 * 443 是高关注端口；
 * 审查系统容易把该端口纳入专门策略；
 * 服务器被判定后，可能直接在 SYN/SYN-ACK 阶段阻断。
8443/2053
优点：
 * 可规避针对 443 的短时端口级异常；
 * 与常见 Web/TLS 服务仍有一定关联；
 * 不必和主力 REALITY 共用同一个监听面。
缺点：
 * 仍然不是主流浏览器默认端口；
 * 只允许 80/443 的企业网、校园网可能无法访问；
 * 高并发加密长连接在非443上可能显得更异常。
高端口
优点：
 * 适合 UDP 端口跳跃；
 * 可分散单端口封锁；
 * 适合冷备和端口池。
缺点：
 * TCP/TLS 长连接在随机高端口上不太像普通 Web；
 * 出站防火墙可能直接拒绝；
 * 地址和端口组合变化大，不能因此认为更隐蔽。
----------------------------------------
11.3 TLS ClientHello 阶段
这是 TCP/TLS 协议的关键暴露点。
REALITY
依靠：
 * uTLS 或类似指纹模拟；
 * 目标 serverName；
 * REALITY 特有的验证逻辑；
 * shortId 等参数。
优点是可以避开自签证书和自有证书的显眼特征。
风险是：
 * 模拟指纹并不等于真实浏览器；
 * TLS 指纹之外还有 record、包长和连接生命周期；
 * 目标 SNI 可能与服务端 IP、端口、IPv6 连通性产生关联；
 * 公开研究与现场报告显示其已进入持续攻防。
NaiveProxy
使用 Chromium 网络栈路线，理论上在 ClientHello 和 HTTP/2 行为上更接近真实 Chrome。
但必须保持：
 * 客户端版本更新；
 * 服务器端与客户端兼容；
 * ALPN、TLS 版本、HTTP/2 设置保持合理；
 * 真实 frontend 行为一致。
AnyTLS
AnyTLS 的重点不是直接模拟 Chrome ClientHello，而是：
TLS transport
+
代理会话层流量整形
所以应将其拆成两个问题：
TLS 指纹：
 由 TLS 实现负责
连接内部包长、padding、session：
 由 AnyTLS 负责
Trojan 与 ShadowTLS
二者都可以让外层呈现正常 TLS，但不能自动消除所有内层代理行为：
TLS 外层自然
≠
后续代理流量完全像浏览器
Mieru/SS/AmneziaWG
没有标准 TLS ClientHello，因此不会被 TLS 指纹识别，但会进入另一个类别：
非 TLS 加密协议
在 443 上尤其需要谨慎，因为“443 上不是标准 TLS/QUIC”可能本身就是异常。
----------------------------------------
11.4 QUIC Initial 阶段
UDP/QUIC 方案主要暴露：
 * QUIC version；
 * Initial 包结构；
 * TLS ClientHello/SNI；
 * Initial 包大小；
 * 首包分片方式；
 * 连接 ID；
 * 重传和握手时序。
公开研究表明，GFW 可以利用 QUIC Initial 的公开结构解密其中的 TLS ClientHello，并对 SNI 进行筛选。
因此：
Hysteria2 默认模式
优点：
 * 尽量保持 HTTP/3/QUIC 外观；
 * 适合 UDP/443；
 * 认证失败时可以提供伪装响应。
风险：
 * 标准 QUIC 结构可被解析；
 * SNI 可能进入 QUIC 专用规则；
 * 过大的连接流量、异常拥塞控制和长连接仍可能被分类。
Salamander
改变 QUIC 包外形，使其难以按标准 QUIC 解析。
风险：
 * 变成随机或高熵 UDP；
 * 在 UDP/443 上失去 H3 正常背景；
 * 可能从“可解析 QUIC”转为“非标准随机 UDP”类别。
Gecko
进一步处理握手分片和 padding，目标是扰乱 QUIC Initial 的包长和布局。
风险：
 * 实现较新；
 * 受版本和客户端支持影响；
 * 分片、padding、MTU、重传行为需要实测；
 * 不能仅根据“包更乱”得出抗封更强。
TUIC/Juicity
采用标准或接近标准的 QUIC 结构，因而继承：
QUIC Initial 可解析
+
SNI 可审查
+
UDP QoS
这一共同攻击面。
----------------------------------------
12. 主动探测：各方案真正的差异
主动探测不是简单“服务端有没有响应”，而是探测系统可以尝试：
错误 TLS ClientHello
随机 HTTP 请求
错误密码
协议重放
截断握手
伪造真实客户端请求
多次连接比较响应
----------------------------------------
12.1 VLESS + REALITY
优势：
 * 可以将未认证流量表现为目标站 TLS；
 * 不容易直接返回“代理协议错误”；
 * 正确配置时主动探测反馈更接近真实服务器。
风险：
 * 探测者可以重复比较；
 * 目标站 TLS 版本、证书链、ALPN、时序不一致可能暴露；
 * 如果服务端对不同探测请求产生稳定、特殊的响应模式，也可能形成指纹。
----------------------------------------
12.2 NaiveProxy
有真实 frontend 时，主动探测通常最自然：
普通请求 → 正常网站
认证代理 → Naive 后端
这比“错误密码后直接关闭连接”更接近正常 Web 服务。
但前提是：
 * 前端站点真实存在；
 * 默认路径、错误页面、HTTP 状态码合理；
 * 不应把代理服务直接暴露为唯一应用；
 * 不要使用明显空白或固定错误响应。
----------------------------------------
12.3 Trojan
Trojan 的经典方案就是：
认证成功 → 代理
认证失败 → fallback 网站
因此主动探测防御成熟。
不过长期流量仍可能通过：
 * 连接持续时间；
 * 内外层 TLS 关系；
 * 包长；
 * 下载/上传比例；
 * 并发行为；
被进一步分类。
----------------------------------------
12.4 AnyTLS
AnyTLS 是否具备良好主动探测防御，取决于：
 * 是否有真实 fallback；
 * 认证失败时如何处理；
 * 错误请求是否返回正常 TLS/HTTP；
 * 是否存在稳定的协议错误响应；
 * padding/session 行为是否被固定化。
因此不能只写：
AnyTLS = 主动探测强
更准确是：
> 协议具备改善流量行为的机制，但主动探测防御取决于外层 TLS、fallback 和部署方式。
----------------------------------------
12.5 ShadowTLS + SS2022
ShadowTLS v3 有挑战响应和外层 TLS 设计，主动探测防御能力较强。
但应注意：
ShadowTLS 的探测处理
≠
SS2022 数据层隐身
两者是不同层次。
----------------------------------------
12.6 Mieru
Mieru 可以通过认证失败静默处理，减少直接暴露服务类型的机会。
但：
静默丢弃
不是始终最自然。
对于真实网站：
错误请求 → HTTP 400/404/页面
比较自然。
对于自定义加密协议：
错误请求 → 无响应
也可能合理，但如果所有探测都得到相同超时，长期比较后也可能形成服务指纹。
----------------------------------------
12.7 Hysteria2/TUIC
它们可以设计认证失败后的伪装或丢弃行为，但要面对：
 * QUIC Initial 可解析性；
 * SNI；
 * 认证失败后的响应；
 * UDP 重传；
 * 连接 ID 和握手时序；
 * 是否表现为真实 H3/QUIC。
所以主动探测不是它们最主要的风险，真正明显的风险往往是：
QUIC 协议识别
+
UDP 行为
+
长时间高速流量
----------------------------------------
13. 被动流量分析：首包只是开始
现在的流量分析可能不止看前几个包。
13.1 首包与早期包长
容易暴露：
 * TLS-in-TLS；
 * 固定代理握手；
 * 固定认证长度；
 * 固定 padding；
 * QUIC Initial 大小；
 * WebSocket Upgrade；
 * gRPC content-type；
 * HTTP/2 SETTINGS 和后续帧序列。
针对策略：
方案 主要应对手段 REALITY TLS 外观、Vision、指纹模拟 AnyTLS session、padding、长度模式控制 Restls 可编程 TLS/record 行为 XHTTP HTTP 传输、header padding、上下行模式 Naive Chromium 栈、H2、多路复用、padding Mieru 分段、低熵、padding、jitter Hy2 QUIC 仿真、分片、Salamander/Gecko AmneziaWG 包长、序列、时序、伪包和协议签名
13.2 长连接阶段
无论早期包多么自然，以下行为仍可能显眼：
连接持续 30 分钟
持续高带宽
上下行比例长期异常
大量连接共享同一 IP
几乎没有普通网页的短连接特征
大量固定并发
同一客户端周期性重连
因此“抗封”不仅是协议配置，还与使用方式有关：
 * 不要将单节点长期作为极大流量下载专用通道；
 * 不要让很多用户共享同一高风险 IP；
 * 不要频繁建立大量短连接；
 * 注意连接复用与重连节奏；
 * 分离日常流量和大流量流量。
这些属于行为风险管理，不是某个协议能够完全解决的问题。
----------------------------------------
14. 性能与资源效率详细比较
14.1 TCP 与 QUIC 的一般差异
场景 TCP/TLS QUIC 单流稳定下载 很好 很好 高丢包 有队头阻塞 通常更有优势 多路复用 需要应用层实现 原生 stream CPU 通常较低 用户态协议栈通常更高 MTU 敏感性 相对低 较高 移动网络迁移 一般 连接迁移有优势 企业/校园网兼容 通常更好 可能被限 UDP 公开 QUIC 专项审查 不适用 已存在
----------------------------------------
14.2 小内存 VPS
优先级通常可以这样理解：
低资源优先
Shadowsocks-rust
Trojan
ShadowTLS + SS2022
中等资源
VLESS + REALITY
AnyTLS
XHTTP
Mieru
需要更谨慎评估
NaiveProxy
Hysteria2 高并发
TUIC 高并发
这不是绝对排名，实际 CPU 会受到：
 * 加密算法；
 * 并发连接数；
 * TLS session；
 * 多路复用；
 * padding；
 * 日志；
 * 内核版本；
 * 网卡中断；
 * 上游带宽；
影响。
----------------------------------------
14.3 性能定位
组合 更适合 REALITY + Vision 日常综合使用、TCP 高吞吐 XHTTP Web、API、很多小流、多路复用 Naive H2 真实浏览器语义、常规 Web AnyTLS TCP 综合备用、会话复用 ShadowTLS+SS2022 小 VPS、高并发、低开销 Trojan 稳定、简单、成熟 Restls 研究和行为整形 Mieru 独立加密路线、可接受整形开销 Hy2 高 RTT、丢包、移动网、UDP/实时 TUIC QUIC 多路复用与标准化倾向 AmneziaWG 全局 VPN、高性能 UDP 异构
----------------------------------------
15. IPv6-only、IPv6+NATv4 的兼容性比较
15.1 IPv6-only 服务器
最容易直接适配
VLESS + REALITY
AnyTLS
Trojan
ShadowTLS + SS2022
XHTTP + TLS
Naive H2
Mieru TCP
这些主要依赖 TCP，服务端监听：
[::]:443
[::]:8443
即可覆盖 IPv6。
需重点测试
Hysteria2
TUIC
Juicity
AmneziaWG
原因是 UDP 路径、MTU、云厂商安全组和用户网络出站限制更敏感。
----------------------------------------
15.2 IPv6 + NATv4
这里要区分三种情形：
A. 服务器本身只有 IPv6，IPv4 通过 NAT64/共享出口访问上游
对客户端来说仍是 IPv6 入口，代理内核需要正常访问 IPv4-only 上游。应确认：
 * DNS64 是否工作；
 * 上游 IPv4 访问是否由 NAT64 提供；
 * 代理程序是否优先解析 AAAA；
 * 部分目的站点是否因 IPv4-only 而访问失败。
B. 服务器有 IPv6 和一个 NAT IPv4 公网入口
如果 NAT 只支持 TCP/443，而不支持任意 UDP 或端口映射，则：
TCP 方案可以保留
UDP 方案可能不可用
尤其要确认 NAT 是否能转发：
 * UDP/443；
 * UDP 高端口范围；
 * 多端口映射；
 * IPv6 端口与 IPv4 端口是否一致。
C. 客户端是 IPv4-only，通过 NAT64/转换访问 IPv6 服务器
这时：
 * TCP 通常较容易工作；
 * UDP/QUIC 可能受 464XLAT、NAT64、运营商 UDP 策略影响；
 * IPv6 字面地址、SNI、证书和 Happy Eyeballs 行为需分别测试。
----------------------------------------
15.3 Happy Eyeballs 不应被忽略
如果服务端同时有：
A
AAAA
客户端可能同时尝试 IPv4/IPv6，哪个先成功就使用哪个。
因此需要分别验证：
客户端最终走的是 IPv6 还是 IPv4
服务端日志中的远端地址族
代理上游访问使用的地址族
IPv6 失败时是否自动回落到 IPv4
如果要测试 IPv6-only，最好使用：
 * 仅 AAAA；
 * 客户端禁用 IPv4 的测试环境；
 * 服务端日志和抓包确认；
 * 分开的 IPv6/IPv4 测试节点。
----------------------------------------
16. CDN、反代与 IPv6 的组合
16.1 最适合普通 CDN 的协议
VLESS + XHTTP + TLS
VLESS + WS + TLS
VLESS + gRPC + TLS
Trojan + TLS
Naive + HTTP frontend
其中从现代 HTTP 兼容性和新部署角度：
XHTTP + TLS
通常比新建 WebSocket 更值得研究。
16.2 不适合普通 HTTP CDN 的协议
Hysteria2
TUIC
Juicity
WireGuard
AmneziaWG
它们需要：
 * L4 TCP/UDP proxy；
 * 专门 UDP 中继；
 * 支持 QUIC/UDP 的边缘服务；
 * 或直接暴露源站。
普通 CDN 的橙云代理并不会自动转发任意 UDP。
16.3 CDN 的实际代价
优点：
 * 隐藏或弱化源站 IP 暴露；
 * 享受边缘网络；
 * 对 Web/TLS 流量有更自然的背景；
 * 源站被封时可能保留维护通道。
代价：
 * CDN 运营商可以看到连接元数据；
 * 长连接与代理流量可能违反服务条款；
 * CDN 可能限制长连接、上传、请求体大小或流量；
 * 源站和域名仍有可关联性；
 * CDN 节点本身也可能被特定地区限制；
 * IPv6 到 CDN、CDN 到源站不一定都是端到端 IPv6。
----------------------------------------
17. 哪些组合是真正的异构？
假设你已经有：
VLESS + REALITY + Vision
那么新增价值大致如下。
新方案 与 REALITY 的差异 异构价值 VLESS + REALITY 换端口 很小 低 VLESS + XHTTP + REALITY transport 有差异，但共享 REALITY/Xray 中 VLESS + XHTTP + TLS HTTP 行为和证书体系不同 高 AnyTLS TLS session/padding 不同 很高 Trojan + TLS 传统 TLS/认证逻辑不同 中高 ShadowTLS + SS2022 TLS 外壳与 AEAD 数据层不同 很高 Naive H2 Chromium/H2/前端不同 很高 Mieru 无 TLS、自定义 AEAD/traffic pattern 很高 Restls + SS TLS 行为剧本不同 很高 Hysteria2 UDP/QUIC 网络栈不同 高 TUIC 仍是 UDP/QUIC 中 AmneziaWG UDP/WireGuard-derived 很高 裸 SS2022 非 TLS AEAD，但特征较明显 中
----------------------------------------
18. 最终推荐：按资源和目标分层
18.1 极简、低资源、优先稳定
主力：
 TCP/443 VLESS + REALITY + Vision
TCP 备用：
 TCP/8443 ShadowTLS v3 + SS2022
UDP 备用：
 UDP/443 Hysteria2 默认模式
适合：
 * 单台小 VPS；
 * Alpine/OpenRC；
 * 不想维护过多组件；
 * 需要 TCP、TLS、UDP 三种故障域。
----------------------------------------
18.2 兼顾隐蔽与异构
TCP/443：
 VLESS + REALITY + Vision
TCP/8443：
 AnyTLS
TCP/2053 或另一台机器：
 ShadowTLS v3 + SS2022
TCP/8443 或 CDN：
 VLESS + XHTTP + TLS + 真实 fallback
UDP/443：
 Hysteria2
UDP/高端口：
 Port Hopping；按实测选择 Salamander/Gecko
这已经覆盖：
REALITY
TLS-session
HTTP-native
QUIC
四种明显不同的模式。
----------------------------------------
18.3 有域名、注重 Web 行为
TCP/443：
 NaiveProxy + H2 + Caddy/真实站点
TCP/8443：
 VLESS + XHTTP + TLS
UDP/443：
 Hysteria2 默认 H3
备用：
 ShadowTLS + SS2022
适合：
 * 有自有域名；
 * 能维护真实静态站；
 * 需要较强主动探测防护；
 * 接受较高部署复杂度。
----------------------------------------
18.4 极端异构储备
TCP/443：
 REALITY
TCP/8443：
 AnyTLS
TCP/2053：
 ShadowTLS + SS2022
TCP/18888：
 Mieru 或 Restls + SS
UDP/443：
 Hysteria2
UDP/高端口：
 AmneziaWG 或 TUIC
注意这不是建议同时承载全部用户，而是保留不同恢复路径。
----------------------------------------
19. 建议的优先级
第一优先级
1. VLESS + REALITY + Vision，TCP/443
原因：
 * 成熟；
 * 性能高；
 * IPv6 适配好；
 * 不依赖自有证书；
 * 仍是最实用的主力 TCP 路线之一。
2. AnyTLS，TCP/8443
原因：
 * 与 REALITY 有明显差异；
 * padding/session 机制适合长期演进；
 * 非443适配自然；
 * 当前公开针对性研究较少，但不能过度解读。
3. ShadowTLS v3 + SS2022，TCP/2053 或 8443
原因：
 * 成熟异构 TCP；
 * 低资源；
 * Rust 生态性能好；
 * 与 Xray/REALITY 技术栈不同。
----------------------------------------
第二优先级
4. VLESS + XHTTP + TLS，TCP/8443
适合：
 * 有域名；
 * 需要真实 HTTP 生态；
 * 需要反代/CDN 兼容；
 * 想避免继续使用传统 WS/gRPC。
5. NaiveProxy + H2，TCP/443
适合：
 * 重视真实浏览器协议栈；
 * 能维护 Caddy/前端；
 * 接受较高资源和维护成本。
----------------------------------------
第三优先级
6. Hysteria2，UDP/443
适合：
 * 高延迟；
 * 高丢包；
 * 移动网；
 * 实时应用；
 * 需要 TCP 之外的性能通道。
但应把 QUIC 专项审查风险放在显眼位置。
7. Hysteria2 + Port Hopping，UDP高端口范围
适合：
 * 单端口 UDP QoS；
 * 单端口封锁；
 * 不能解决全 UDP 阻断。
8. TUIC
适合作为第二 QUIC 实现，不是 QUIC 网络层之外的真正备用。
----------------------------------------
第四优先级：研究型
Mieru
Restls + SS
AmneziaWG
Juicity
这些方案有独特技术价值，但在正式使用前应重点验证：
 * 目标客户端支持；
 * IPv6 连通性；
 * 多 ISP；
 * MTU；
 * 高并发；
 * 长连接；
 * 版本兼容；
 * 真实网络下的长期稳定性。
----------------------------------------
20. 推荐的实测方法
不能用“网上五星”替代实测。建议建立同机、同带宽、同时间的矩阵。
20.1 测试环境
至少分开：
中国电信
中国联通
中国移动
海外 VPS/探针
IPv6-only 网络
IPv4-only/NAT64 网络
家庭宽带
移动蜂窝
校园/企业网络
20.2 基础连通性
记录：
 * DNS/AAAA 解析时间；
 * TCP SYN 成功率；
 * TCP 握手 RTT；
 * TLS 握手 RTT；
 * QUIC Initial 成功率；
 * UDP 首包丢失；
 * IPv6 与 IPv4 分别统计；
 * 443、8443、2053、随机高端口分别统计。
20.3 性能
分别测试：
单流下载
多流下载
上行
下行
短连接
长连接
100/500/1000 并发
记录：
 * p50/p95/p99 延迟；
 * 单流吞吐；
 * 多流总吞吐；
 * CPU；
 * RSS；
 * 文件描述符；
 * 重传；
 * QUIC stream reset；
 * TCP retransmission；
 * 连接建立失败率。
20.4 弱网测试
模拟：
0.5%、1%、3%、5% 丢包
20ms、80ms、150ms、250ms RTT
带宽抖动
MTU 1280、1300、1350、1400、1500
重点观察：
 * Hysteria2/TUIC；
 * XHTTP；
 * Naive H2；
 * Mieru low entropy；
 * AmneziaWG。
20.5 长期行为
至少观察：
24小时
72小时
7天
记录：
 * 是否出现单向阻断；
 * 是否出现端口级限速；
 * 是否出现 IP 级故障；
 * 是否出现 SNI 特定失败；
 * 是否出现某 ISP 特定失败；
 * 重启或换端口后是否恢复；
 * 同一 IP 不同协议是否同时失效。
----------------------------------------
21. 最后结论
443、IPv6-only
最值得部署的是：
TCP/443：
 VLESS + REALITY + Vision
 NaiveProxy + H2
 VLESS + XHTTP + TLS
 AnyTLS
UDP/443：
 Hysteria2
 TUIC
推荐排序不是单一排行榜，而是：
 * 主力 TCP：REALITY；
 * 高仿真 TCP：Naive/H2；
 * Web 原生 TCP：XHTTP+TLS；
 * 新型异构 TCP：AnyTLS；
 * 性能 UDP：Hysteria2；
 * 第二 QUIC：TUIC。
非443、IPv6 或 IPv6/NATv4
最值得部署的是：
TCP/8443：
 AnyTLS
 VLESS + XHTTP + TLS
 Trojan + fallback
TCP/2053 或另一非443端口：
 ShadowTLS v3 + SS2022
TCP/18888 等冷备：
 Mieru
 Restls + SS
UDP高端口：
 Hysteria2 + Port Hopping
 根据实测选择 Salamander/Gecko
UDP非QUIC：
 AmneziaWG
最终推荐矩阵
主力：
 IPv6 TCP/443 VLESS + REALITY + Vision
第一 TCP 备用：
 IPv6 TCP/8443 AnyTLS
第二 TCP 备用：
 IPv6 TCP/2053 ShadowTLS v3 + SS2022
Web 备用：
 IPv6 TCP/8443 VLESS + XHTTP + TLS + fallback
浏览器仿真备用：
 IPv6 TCP/443 NaiveProxy + H2 + Caddy
UDP 性能：
 IPv6 UDP/443 Hysteria2 默认 H3
UDP 端口韧性：
 IPv6 UDP/20000-50000 Hysteria2 + Port Hopping
非QUIC UDP冷备：
 IPv6 UDP/高端口 AmneziaWG
研究型异构：
 IPv6 TCP/高端口 Mieru 或 Restls + SS
最核心的判断是：
> REALITY、Naive、XHTTP、AnyTLS 主要解决 TCP/TLS/Web 流量分类问题；Hysteria2、TUIC 主要解决 UDP/QUIC 性能和传输问题；Mieru、AmneziaWG、Restls 则提供不同于前述体系的异构路线。
因此最合理的架构不是“找一个无敌协议”，而是：
TCP/REALITY
+
TCP/AnyTLS 或 ShadowTLS
+
TCP/HTTP-native
+
UDP/QUIC
+
可选非QUIC冷备
同时要把这些因素视为同等重要：
协议特征
TLS/QUIC 指纹
SNI/DNS
IP reputation
端口策略
IPv6 路由
MTU/PMTUD
运营商 QoS
客户端兼容性
长期连接行为
没有任何协议能够保证永久不被识别或阻断；真正有长期价值的是异构传输、可更新实现、合理流量行为、多个地址/端口入口以及持续实测。


下面按截至 2026 年 9 月 17 日仍有维护、主流客户端可用、适合自建通用代理/VPN 的公开协议来梳理。范围覆盖 Xray、sing-box、Hysteria、TUIC、NaiveProxy、Shadowsocks、WireGuard 系协议；不把已停止维护、私有机场协议和只能用于特定研究系统的传输逐一穷举。

先说明一个关键限制：

目前没有公开、持续、覆盖全国运营商的测试能准确给出 Reality、XHTTP、Hysteria2、AnyTLS、TUIC 各自的“被封概率”。  
对 Shadowsocks、裸 VMess、完全加密流量、传统 TLS 代理和 QUIC 检测，有公开测量；对较新的协议，更多只能依据协议设计、公开论文和少量现场报告判断。“尚无公开封锁报告”不等于“无法识别”。

一、先给结论

最推荐的总体组合

方案 A：性能、兼容性、抗封锁最均衡

TCP/443：VLESS + REALITY + RAW + Vision
UDP/443：Hysteria2 + TLS + HTTP/3 伪装
QUIC 正常可用：优先默认 H3 模式
QUIC 被针对或特征明显：再考虑 Salamander/Gecko

TCP 和 UDP 是不同传输层，因此可以同时使用数字端口 443，不需要端口复用器。Reality 负责普适 TCP 入口，Hysteria2 负责高丢包、移动网络和大带宽场景。Reality/Vision 是 Xray 当前低开销直连方案；Hysteria2 基于 QUIC，可同时代理 TCP、UDP，并提供 HTTP/3 伪装和可选混淆。([xtls.github.io](https://xtls.github.io/en/config/transport.html?utmsource=openai))

方案 B：更侧重流量分析对抗

TCP/443：VLESS + XHTTP + REALITY
UDP/443：Hysteria2 + Gecko
可选再准备一个 XHTTP + TLS + CDN 入口

XHTTP 支持 HTTP 化传输、头部填充、连接复用以及上下行分离，能减少传统单长连接代理的部分形态特征；代价是配置和排错明显比 Vision 复杂，纯测速、低延迟直连未必优于 Vision。XHTTP 可以与 REALITY、普通 TLS、CDN、H2/H3 组合。([github.com](https://github.com/XTLS/Xray-core/discussions/4113?utmsource=openai))

方案 C：浏览器流量拟态优先

TCP/443：NaiveProxy HTTPS/H2
UDP/443：Hysteria2
备用：非 443 的 REALITY

NaiveProxy 复用 Chromium 网络栈，目标是让 TLS、HTTP/2 和连接行为尽量接近普通 Chrome；它还使用认证路由、填充和标准前端防御主动探测。缺点是资源开销、版本更新频率和客户端生态不如 Xray/sing-box。([github.com](https://github.com/klzgrad/naiveproxy/blob/master/README.md?plain=1&utmsource=openai))

非 443、IPv6/NATv4 最实用组合

如果 NAT 服务商同时映射同一数字端口的 TCP 和 UDP：

TCP/映射端口：VLESS + REALITY + Vision
UDP/同一映射端口：Hysteria2
另备 AmneziaWG 3.1 或 TUIC 作为 UDP 替代

如果只给一个 TCP 端口：优先 Reality，其次 AnyTLS。  
如果只给一个 UDP 端口：优先 Hysteria2，其次 AmneziaWG、TUIC/Juicity。

二、IPv6 和 NATv4 对协议选择的实际影响

IPv6-only 服务器的前提

AAAA-only 节点要求客户端网络真正具有 IPv6 可达性；如果客户端只有 IPv4，RAW Reality、Hysteria2、TUIC 等直连协议不会自动跨越 IPv4/IPv6。

若需要让 IPv4-only 客户端连接 IPv6-only 源站，可以考虑：

双栈 CDN 前置的 XHTTP/WS/gRPC 等 HTTP 类传输；
单独的双栈中转；
为 NATv4 映射建立另一条节点记录。

普通 CDN 能前置 HTTP 类协议，但不能把普通代理 CDN 当作任意 RAW TCP 或 UDP 转发器。以 Cloudflare 普通代理为例，支持 443、8443、2053、2083、2087、2096 等 HTTPS 端口，而非任意端口。([developers.cloudflare.com](https://developers.cloudflare.com/fundamentals/reference/network-ports/?utmsource=openai))

只有 443、没有 80 端口时的证书

对于 Hysteria2、TUIC、Trojan、NaiveProxy、AnyTLS、普通 TLS XHTTP，推荐：

域名设置 AAAA 到服务器；
使用 DNS-01 自动签证书；
不要依赖关闭验证的 insecure 模式。

DNS-01 不要求服务器开放 80 或 443；Let’s Encrypt 也支持 IPv6 验证。TLS-ALPN-01 可以在 443 完成，但 ACME 客户端支持较有限。Reality 本身通常不需要申请普通域名证书。([letsencrypt.org](https://letsencrypt.org/docs/challenge-types/?utmsource=openai))

NATv4 的限制

NATv4 本身不改变协议加密能力，但会影响：

外部端口是否固定；
TCP、UDP 是否同时映射；
能否映射大范围端口；
UDP 会话超时；
是否为对 UDP 打洞不友好的对称 NAT。

因此，Hysteria2 端口跳跃在公网 IPv6 上较容易实现，在共享 NATv4 上通常不实用，因为它要求整个 UDP 端口范围均可到达或被转发。Hysteria2 官方支持多端口和端口范围跳跃；sing-box 新版还提供 Hysteria Realm，通过 STUN/打洞连接 NAT 后服务器，但成功率仍受 NAT 类型影响，不能完全等价于固定公网端口。([v2.hysteria.network](https://v2.hysteria.network/docs/advanced/Port-Hopping/?utmsource=openai))

三、主流协议完整分类

  协议/组合   底层   IPv6 443   非443 / NATv4   综合定位  

  VLESS + REALITY + Vision   TCP   非常适合   非常适合   直连主力，性能和抗主动探测均衡  
  VLESS + XHTTP + REALITY   TCP/HTTP化   非常适合   适合   更重视流量形态和中间件穿透  
  VLESS + XHTTP + TLS/CDN   H2/H3   非常适合   取决于 CDN 支持端口   隐藏源站、抵抗源 IP 封锁  
  NaiveProxy   HTTPS/H2/H3   最适合443   技术可用，但高端口拟态变弱   浏览器栈拟态较强  
  AnyTLS   TLS/TCP   适合   适合   新型 TLS 隧道，低延迟、复用和填充  
  Trojan + TLS   TLS/TCP   适合   可用   传统方案，当前不如 Reality/XHTTP  
  ShadowTLS v3 + SS2022   TLS外观/TCP   适合   适合   真实 TLS 握手外观，加密由 SS 提供  
  Hysteria2   QUIC/UDP   非常适合 UDP/443   非常适合   高丢包、大带宽、移动网络  
  TUIC v5   QUIC/UDP   适合   适合   低延迟、连接复用，拟态弱于 HY2  
  Juicity   QUIC/UDP   适合   适合   TUIC 类方案，生态较小  
  Shadowsocks 2022 裸跑   TCP/UDP   不建议占443   可用但风险较高   极高效率，弱点是随机流量特征  
  AmneziaWG 3.1   UDP VPN   技术可用   非常适合非443   WireGuard 性能，加传输混淆  
  WireGuard 原版   UDP VPN   不建议   仅低审查环境   快，但固定指纹明显  
  VMess/裸 VMess   TCP   不建议   不建议   兼容性遗留方案  
  VMess/VLESS + WS/gRPC/TLS   TLS/HTTP   可用   可用   成熟，但通常应迁移到 XHTTP  
  OpenVPN/stunnel/SSH 隧道   TCP/UDP   技术可用   技术可用   可识别性高，不宜作为强审查主力  
  mKCP/裸随机 UDP   UDP   不建议   不建议   随机高熵流量，没有正常协议外观  
  SOCKS/HTTP 明文暴露   TCP   不适合   不适合   无安全性，不应公网开放  

四、逐协议深入对比

VLESS + REALITY + Vision

加密与混淆

VLESS 本身是轻量认证/传输协议，通常依赖外层 REALITY；REALITY 修改 TLS 行为，使握手外观看起来与指定目标网站的正常 TLS 类似，并保留前向安全性。Vision 对内层 TLS 数据进行直接复制和针对性填充，减少重复加解密。([github.com](https://github.com/XTLS/Xray-docs-next/blob/main/docs/en/config/transports/reality.md?utmsource=openai))

性能

CPU 和内存开销低；
长连接、下载、视频表现好；
在低丢包线路上通常属于第一梯队；
TCP 丢包会造成队头阻塞，不适合严重丢包环境下追求极限速度。

抗封锁

优点是握手和主动探测防御较强；主要风险已从“能否读懂协议头”转移到：

长时间单 TCP 流；
内层 TLS 握手流量形态；
IP/端口关联；
客户端 TLS 指纹；
运营商针对特定线路或数据中心的策略。

2024 年 USENIX 研究表明，包括 VLESS、Trojan、REALITY 在内的隧道，理论上可能通过“封装在外层隧道中的内层 TLS 握手”进行协议无关分类；复用和填充能降低检测效果，但不能保证不可识别。([usenix.org](https://www.usenix.org/conference/usenixsecurity24/presentation/xue-fingerprinting?utmsource=openai))

综合：加密 5/5，握手伪装 5/5，行为隐蔽 3.5/5，效率 5/5。

VLESS + XHTTP + REALITY/TLS

XHTTP 不是单独的加密算法，而是 HTTP 化传输层，可组合：

VLESS + XHTTP + REALITY 直连；
VLESS + XHTTP + TLS 直连；
VLESS + XHTTP + TLS + CDN；
上行 CDN、下行 Reality；
上行 Reality、下行 CDN；
H2/H3、IPv4/IPv6 分离。

XHTTP 提供 packet-up、stream-up、stream-one、头部填充、XMUX 和上下行分离。其主要价值不是“比 Reality 更强的密码学”，而是让流量经过标准 HTTP 中间件并改变连接形态。([github.com](https://github.com/XTLS/Xray-core/discussions/4113?utmsource=openai))

性能排序

大致可理解为：

直连 stream-one/stream-up：性能较好；
CDN H2/H3：受 CDN 路由、限流、回源影响；
packet-up：兼容性强，但请求数量和开销较大；
上下行分离：抗分析和线路优化更灵活，配置复杂度最高。

综合：加密 5/5，握手伪装 5/5，行为隐蔽 4/5，效率 4/5，部署复杂度 5/5。

NaiveProxy

NaiveProxy 使用 Chromium 网络栈，尽量复现普通浏览器的 TLS/HTTP 行为；通过 HTTP/2 或 HTTP/3 CONNECT、认证路由、填充和前端网站处理未认证请求。官方特别要求持续更新版本，以保持客户端指纹与当前 Chrome 接近。([github.com](https://github.com/klzgrad/naiveproxy/blob/master/README.md?plain=1&utmsource=openai))

优点

TLS 指纹比普通 Go TLS 隧道更自然；
H2 多路复用可以弱化一连接一目标的特征；
主动探测得到正常前端响应；
443 语义最合理。

缺点

客户端和服务端资源开销较高；
依赖 Chromium 版本持续更新；
客户端支持面不如 VLESS/Hysteria2；
高端口运行时，虽然加密不受影响，但“普通浏览器访问 HTTPS”的合理性下降。

综合：加密 5/5，拟态 5/5，行为隐蔽 4/5，效率 3.5/5。

AnyTLS

AnyTLS 在 TLS 之上进行连接复用，并通过灵活分包和填充缓解 TLS-in-TLS 特征；UDP 当前通常通过 UDP-over-TCP 承载。其协议在 2025 年更新至 v2，sing-box、mihomo 等已经支持，但参考实现本身明确提示默认示例配置可能不安全，生产使用必须正确验证证书。([github.com](https://github.com/anytls/anytls-go/blob/main/docs/protocol.md?utmsource=openai))

优点

TCP 连接建立和复用效率好；
专门考虑嵌套 TLS 指纹；
配置相对简洁；
适合只有一个 TCP 映射端口的 NATv4。

缺点

协议较新，缺乏长期公开对抗测量；
UDP-over-TCP 在丢包时会出现明显队头阻塞；
伪装质量依赖 TLS 栈、证书、ALPN 和填充参数；
错误地启用 insecure 会破坏服务器身份认证。

综合：加密 5/5，拟态 4/5，行为隐蔽 4/5，TCP效率 4.5/5，UDP效率 2/5；证据可信度低于 Reality/Naive。

ShadowTLS v3 + Shadowsocks 2022

ShadowTLS 只负责暴露真实 TLS 1.3 握手和对后续数据进行 TLS 记录层式封装，本身并不承担完整的数据加密，因此必须和 Shadowsocks 2022 等加密代理组合。v3 增加了双向认证、HMAC 完整性、抗重放、抗主动探测和抗流量劫持设计。([github.com](https://github.com/ihciah/shadow-tls/blob/master/docs/protocol-v3-en.md?utmsource=openai))

Shadowsocks 2022 使用现代 AEAD、强制重放保护，并优化 UDP 会话；但它使用预共享密钥，没有前向安全性。([shadowsocks.org](https://shadowsocks.org/doc/sip022.html?utmsource=openai))

判断

比裸 SS2022 强很多；
性能不错；
443 比随机高端口合理；
实现和客户端生态小于 VLESS；
缺少近年的大规模独立封锁测量。

综合：加密 4/5，握手拟态 4.5/5，行为隐蔽 3.5/5，效率 4/5。

Trojan + TLS

Trojan 使用正常 TLS 加密，并可把认证失败或非代理请求回落到真实网站。它曾经是 443 端口的标准方案，但普通 TLS 客户端指纹、连接行为和内层握手仍可能被分类。

2022 年 10 月发生过大规模 TLS 代理封锁报告，涉及 Trojan、Xray、V2Ray TLS+WebSocket、VLESS 和 gRPC；当时 NaiveProxy 没有收到类似报告。该事件说明“拥有真实证书和网站回落”不等于无法识别。([gfw.report](https://gfw.report/blog/blockingoftlsbasedcircumventiontools/en/))

综合：加密 5/5，握手拟态 3.5/5，行为隐蔽 2.5/5，效率 4/5。

如果客户端已经支持 Reality 或 XHTTP，通常没有必要新建纯 Trojan 节点。

Hysteria2

Hysteria2 基于 QUIC，使用 TLS，支持 TCP/UDP 代理；认证失败时可表现为普通 HTTP/3 服务。默认模式试图保持 H3 外观，也可以使用 Salamander；新版 Gecko 会对 QUIC 长头初始包进行分片和随机填充，以降低握手包形态稳定性。([v2.hysteria.network](https://v2.hysteria.network/docs/developers/Protocol/?utmsource=openai))

性能

高丢包、长距离、移动网络表现突出；
没有 TCP 连接级队头阻塞；
支持连接迁移和端口跳跃；
对服务器带宽估值和拥塞参数较敏感；
在严格限速或 UDP QoS 环境中可能反而不稳定。

混淆模式取舍

默认 H3：更像正常 QUIC，但 QUIC Initial 可以被旁观者解密并读取 ClientHello/SNI；
Salamander/Gecko：隐藏标准 QUIC 形态，但会更接近高熵 UDP 数据；
端口跳跃：对“按 UDP 端口限速/封锁”有效，对全 UDP 封锁无效。([v2.hysteria.network](https://v2.hysteria.network/docs/advanced/Port-Hopping/?utmsource=openai))

GFW Report 的 2025 研究确认，GFW 自 2024 年 4 月 7 日起已经能够大规模解密 QUIC Initial，并根据其中的 SNI 进行过滤。因此，“使用 QUIC”不能再直接等同于“看不到 SNI”；这项能力主要针对被封域名，并不代表所有 Hysteria2/TUIC 都会被识别。([gfw.report](https://gfw.report/publications/usenixsecurity25/en))

综合：加密 5/5，主动探测防御 4/5，行为隐蔽 3/5，弱网性能 5/5。

TUIC v5 与 Juicity

TUIC 使用 QUIC 多路复用 TLS 加密流，支持 TCP/UDP 转发和 BBR/Cubic/NewReno。sing-box 明确建议关闭 TUIC 0-RTT，因为存在重放风险。([github.com](https://github.com/tuic-protocol/tuic/blob/master/SPEC.md?utmsource=openai))

Juicity 基于 TUIC 思路，重点改进 UDP 处理和稳定性，但生态规模更小。([github.com](https://github.com/juicity/juicity/blob/main/README.md?utmsource=openai))

与 Hysteria2 相比：

TUIC/Juicity 更接近“高效 QUIC 代理”；
Hysteria2 更强调 HTTP/3 伪装、未认证请求处理和可选混淆；
TUIC 的协议规范没有像 HY2 那样强制定义完整的探测伪装行为。

综合：加密 5/5，拟态 3/5，效率 5/5，抗封锁证据 2.5/5。

适合作为 UDP 备用入口，不建议单独承担最高审查环境下的唯一入口。

Shadowsocks 2022 裸跑

SS2022 的密码设计和效率很好，但网络外观仍是随机高熵字节流。GFW 的公开研究显示，它可以依据第一数据包的长度、熵、ASCII 比例等特征识别完全加密流量，并配合主动探测；该系统曾影响 Shadowsocks、Outline、VMess、Obfs4 等。([gfw.report](https://gfw.report/publications/imc20/en/?utmsource=openai))

因此：

适合内网、可信专线、低审查环境；
可以作为 ShadowTLS 内层；
不建议在公网 443 裸跑；
换随机端口只能降低扫描噪音，不能解决流量分类问题。

综合：加密 4/5，拟态 1/5，效率 5/5，当前封锁风险高。

AmneziaWG 3.1 与原版 WireGuard

WireGuard 密码学强、性能极高，但固定消息头和可预测包长容易识别。AmneziaWG 保留 WireGuard 的 NoiseIK、Curve25519 和 ChaCha20-Poly1305，同时增加头部保护、动态包长、内容填充、时间扰动和协议签名包。([docs.amnezia.org](https://docs.amnezia.org/documentation/amnezia-wg/?utmsource=openai))

适用性

很适合非 443 UDP 映射端口；
适合需要完整 VPN/TUN 而不是单纯代理的设备；
路由器和手机能耗表现好；
UDP/443 技术上可用，但它并不是真正的 HTTP/3 服务器，拟态可信度取决于配置；
官方 3.1 主要是针对 2026 年俄罗斯大规模 WireGuard/AWG 封锁演进，缺少针对中国的独立公开验证。([docs.amnezia.org](https://docs.amnezia.org/documentation/amnezia-wg/?utmsource=openai))

AmneziaWG：加密 5/5，混淆设计 4/5，性能 5/5，中国实证可信度 2.5/5。  
原版 WireGuard：加密 5/5，混淆 1/5，性能 5/5。

五、当前“打击力度”与证据等级

已有强实证：压力高

裸 Shadowsocks、VMess、Obfs4、其他随机完全加密 TCP

公开研究确认 GFW 能依据首包熵、长度、ASCII 特征进行被动分类，并继续主动探测或触发短期阻断。随机加密流量不是没有特征，而是“随机本身就是一类特征”。([gfw.report](https://gfw.report/blog/modifiedshadowsocks/zh/?utmsource=openai))

OpenVPN 和简单套壳 OpenVPN

研究在百万用户级 ISP 环境中识别出超过 85% 的 OpenVPN 流，并成功识别 41 种混淆配置中的 34 种。SSH、stunnel、简单 Shadowsocks 外套等，往往仍保留 OpenVPN 的包长和确认包行为。([doi.org](https://doi.org/10.1145/3618117?utmsource=openai))

有历史实证：压力中高

Trojan、TLS+WS、TLS+gRPC、传统 Xray/V2Ray TLS

2022 年出现过大规模针对 TLS 代理服务端口的封锁。更换端口后，部分案例进一步发展为整个 IP 被封。([gfw.report](https://gfw.report/blog/blockingoftlsbasedcircumventiontools/en/))

这并不表示这些协议现在必然立即被封，而是说明普通 TLS 外观和网站回落不足以提供长期不可识别性。

审查能力已确认，但不等于协议已被攻破

Hysteria2、TUIC、Juicity 等 QUIC 系

GFW 已经能够解密 QUIC Initial 并按 SNI 过滤，也会检查新 UDP 流的首包；因此 QUIC 协议不能依靠“握手完全不可见”这一旧假设。不过，目前公开论文证明的是 QUIC SNI 审查，不是对 Hysteria2/TUIC 的全国性协议精准封锁。([gfw.report](https://gfw.report/publications/usenixsecurity25/en))

当前缺乏大规模公开实证

Reality、XHTTP、AnyTLS、ShadowTLS v3、AmneziaWG 3.1

截至当前，没有公开、可复现的全国性测量证明这些协议已被完全精准攻破。2026 年 7 月出现过一例 VLESS+REALITY+XHTTP/443 疑似探测后中国方向握手阻断的现场报告，但它是单个 GitHub issue，不能证明协议级全面失效，也可能涉及 IP 污染、端口封锁或运营商策略。([github.com](https://github.com/XTLS/Xray-core/issues/6421?utmsource=openai))

关于“443 是否正在被重点打击”

443 的确是审查设备最关注的端口之一，但不能简单理解为“443 一定比高端口危险”。

2025 年 8 月 20 日曾发生一次面向所有跨境 TCP/443 的无条件 RST 注入，持续约 74 分钟，22、80、8443 等端口未出现相同行为；这更像短暂异常或错误策略，而不是长期常态。([gfw.report](https://gfw.report/blog/gfwunconditionalrst20250820/zh/))

443 的优点是正常流量巨大、协议语义合理；缺点是审查设备投入最多。高端口降低了热点端口集中度，但一条“23456 上长期运行的 HTTPS/QUIC 大流量”同样可能成为特征。

六、针对你的两类端口环境，具体选择

场景 1：仅 IPv6、只有 443

最佳通用配置

TCP/443：VLESS + REALITY + Vision
UDP/443：Hysteria2
准备一个不同入口的 XHTTP+CDN 作为客户端无 IPv6或源 IP 被阻断时的备用

优点：

TCP/UDP 双路径；
一条偏兼容，一条偏性能；
Reality 不需要普通证书；
Hysteria2 可用 DNS-01 证书；
不需要复杂 TCP 端口复用。

高对抗配置

TCP/443：VLESS + XHTTP + REALITY
UDP/443：Hysteria2 + Gecko
CDN：XHTTP + TLS/H2 或 H3
不要让所有备用入口最终暴露为完全相同的 IP、传输和域名关系

简洁配置

TCP/443：NaiveProxy 或 AnyTLS
UDP/443：Hysteria2

不建议

SS2022 裸跑 443；
VMess 裸跑；
WireGuard 原版占 UDP/443；
在同一个 TCP/443 上堆叠大量 Trojan、WS、gRPC、Naive、Reality，除非你非常熟悉首包分流、SNI/ALPN 和 fallback；
关闭 TLS 验证。

场景 2：非 443、公网 IPv6

推荐：

TCP：Reality/Vision；
UDP：Hysteria2；
高丢包：HY2 优先；
完整 VPN：AmneziaWG；
低延迟 UDP 备用：TUIC/Juicity；
有完整 UDP 端口范围时，可启用 Hysteria2 端口跳跃。

如果希望保留普通 TLS 的语义，8443、2053、2083、2087、2096 等比完全随机端口更合理，但这只是降低异常感，不是安全保证。部分端口也能被普通 CDN 代理。([developers.cloudflare.com](https://developers.cloudflare.com/fundamentals/reference/network-ports/?utm_source=openai))

场景 3：非 443、IPv6 + NATv4

NAT 同时给 TCP/UDP 映射

TCP：Reality
UDP：Hysteria2
可使用同一数字端口，前提是服务商确实分别映射 TCP 和 UDP。

只给 TCP 映射

优先级：

Reality + Vision
XHTTP + Reality
AnyTLS
NaiveProxy
ShadowTLS v3 + SS2022
Trojan

只给 UDP 映射

优先级：

Hysteria2
AmneziaWG 3.1
TUIC
Juicity
原版 WireGuard——仅在不存在强 DPI 时

不固定映射或 UDP 映射经常变化

Hysteria Realm/STUN 打洞可以尝试；
如果 NAT 类型不友好，最终仍应使用固定中转或 TCP 入口；
不要在只有一个 NAT UDP 端口时配置大范围端口跳跃。

七、最终排名

抗封锁优先

XHTTP + REALITY
Reality + Vision
NaiveProxy
XHTTP + TLS + CDN
AnyTLS
ShadowTLS v3 + SS2022
Hysteria2
Trojan
TUIC/Juicity
SS2022 裸跑 / VMess / WireGuard 原版

前四名的实际顺序会因威胁模型变化：防主动探测时 Reality 很强；防源 IP 封锁时 CDN XHTTP 更强；追求浏览器 TLS 指纹时 Naive 更自然。

性能优先

Hysteria2——高丢包、长距离
Reality + Vision——正常 TCP 线路
TUIC/Juicity
AmneziaWG
AnyTLS
ShadowTLS + SS2022
XHTTP 直连
NaiveProxy
XHTTP + CDN
OpenVPN/多层套娃

最值得实际部署的组合

首选：

TCP/443 Reality/Vision
UDP/443 Hysteria2
XHTTP/CDN 独立备用路径

IPv6/NATv4 非 443：

TCP 映射 Reality
UDP 映射 Hysteria2 或 AmneziaWG
只有 TCP 时增加 AnyTLS
不把裸 SS、VMess、WireGuard 当主力

最重要的不是寻找一个“永不被封”的协议，而是准备不同传输层、不同流量外观、不同前置方式的两到三条路径。TCP Reality 与 UDP Hysteria2 的组合，比在同一传输层堆叠五种相似 TLS 代理更有实际冗余价值。


下面给出一份以中国大陆跨境网络环境、服务端仅有公网 IPv6，以及可能通过 NAT 获得 IPv4 出口为前提的研究型比较。重点是协议选择、端口组合、流量外观、性能和截至 2026 年 9 月可公开验证的封锁能力，不提供规避监管的一键部署脚本或可直接复制的隐蔽配置。

先说结论：

443/TCP 首选“真实 HTTPS 外观”方案，即 NaïveProxy，或 VLESS + REALITY + Vision。前者的浏览器栈一致性更强，后者效率和直连性能优。
443/UDP 首选 Hysteria 2，其次 TUIC v5。Hysteria 2 对高丢包链路更激进，TUIC 更偏低延迟和标准 QUIC 能力。
最佳组合不是单一协议，而是 TCP 443 + UDP 443 双栈并存，再增加一个非 443 备用入口。
仅 IPv6 并不会天然提高抗封锁能力。它可能降低被传统 IPv4 扫描碰到的概率，但 IPv6 地址、端口、SNI、QUIC Initial、TLS 指纹和流量统计特征仍然可以被识别。
NATv4 对主要协议选择影响不大。客户端到代理服务器走 IPv6，服务器访问 IPv4 目标时再由 NAT64、双栈出口或应用层转发完成。不要把“IPv6 入站”和“IPv4 出站”误认为同一种能力。
一、先厘清“443 端口”的实际含义

同一个数字端口可以同时存在两项独立服务：

TCP 443：HTTPS、HTTP/2、TLS、REALITY、NaïveProxy、Trojan、WebSocket/gRPC 等。
UDP 443：QUIC、HTTP/3、Hysteria 2、TUIC、MASQUE 等。

因此，一台仅有公网 IPv6 的服务器可以在 443/TCP 和 443/UDP 上同时运行不同协议，这通常是最有价值的组合。TCP 和 UDP 不冲突，只要实现、证书和前置服务的端口占用关系处理正确即可。

2025 年的研究已经确认，QUIC 的 Client Initial 虽然在协议形式上被加密，但其初始密钥可以由旁路观察者根据公开字段推导，因此审查设备能够解出 ClientHello 和 SNI。研究还观察到中国自 2024 年 4 月开始进行基于 QUIC SNI 的定向过滤，并使用流状态和后续丢包进行约 180 秒的残留封锁。

此外，2025 年 8 月 20 日约 00:34 至 01:48，曾出现一次针对跨境 TCP 443 的无条件 RST+ACK 注入异常。该事件只持续约 74 分钟，没有同样影响 TCP 22、80、8443，因此不能理解为 TCP 443 已长期不可用，但它说明“把所有可用性压在 TCP 443 上”存在系统性风险。

二、适合 TCP 443、仅 IPv6 入站的协议
1. NaïveProxy over HTTPS/HTTP2
流量外观

NaïveProxy 最大的优势不是“再加一层加密”，而是直接复用 Chromium 网络栈，使客户端的 TLS 参数、HTTP/2 前言、多路复用行为尽量接近真实 Chrome。它还使用应用层前置处理未认证访问，并通过填充和分片降低长度分析价值。项目明确要求及时更新客户端，以保持和 Chrome 网络栈的特征同步。

优点
TLS 指纹一致性在公开方案中属于第一梯队。
未认证探测可以表现为正常网站。
HTTP/2 多路复用对大量短连接友好。
443/TCP 与正常 HTTPS 的协议语义最接近。
IPv6 原生入站没有协议层限制。
缺点
Chromium 网络栈相对重，CPU、内存和二进制体积通常高于轻量协议。
多层 HTTP/TLS 封装会增加一些开销。
必须维持真实证书、域名和正常网站行为。
长时间、高占空比、单目的地大流量仍可能在统计层面显得异常。
如果版本长期不更新，客户端指纹可能落后于真实浏览器。
评价
加密安全：A
流量外观：A+
主动探测防御：A+
性能：B+
维护成本：中高
适合：443/TCP 主入口、重视流量真实性的场景

2022 年针对 TLS 代理的大规模封锁报告中，Trojan、VLESS、WebSocket、gRPC 等均有服务器受到影响，而当时没有收到 NaïveProxy 被封锁的报告。该历史结果不能证明今天“绝对安全”，但支持了“真实浏览器栈比单纯模拟 TLS 更稳健”的判断。

2. VLESS + RAW + REALITY + XTLS Vision
流量外观

VLESS 本身是轻量、无状态的代理承载协议，真正承担加密和伪装的是 REALITY。REALITY 是一种修改后的 TLS 安全层，借用目标站点的外观和握手特性；官方文档将其与 RAW、XHTTP、gRPC 传输组合，并表示配合 Vision 流控可显著提高性能。

优点
相比 TLS-in-TLS、WebSocket-in-TLS，额外封装较少。
Vision 针对数据传输路径优化，吞吐和 CPU 效率优秀。
不一定要求服务器自己持有常规站点证书。
对传统主动探测有较好的拒绝和回落行为。
非常适合 IPv6 直连和大流量传输。
缺点
它是“尽力拟态”，不是直接运行完整 Chromium 网络栈。
目标站点、SNI、ALPN、网络路径和客户端指纹若组合不自然，仍可能形成关联特征。
失败认证连接会向目标转发，官方特别提示可能被扫描者滥用为转发器，应有流量限制和前置过滤。
对实现版本和客户端支持依赖较强。
评价
加密安全：A
流量外观：A
主动探测防御：A
性能：A+
维护成本：中
适合：443/TCP 高性能直连主入口

这是“效率优先”的 443/TCP 首选。与 NaïveProxy 相比，REALITY + Vision 通常性能更高；与普通 TLS + WebSocket 相比，其嵌套和固定应用层特征更少。

3. VLESS + XHTTP + TLS/REALITY

XHTTP 的定位更偏向将代理流量拆分并映射到 HTTP 语义，可配合 REALITY，也可用于更复杂的前置或转发结构。与 RAW + Vision 相比，它通常具有更好的 HTTP 化和路径适应能力，但实现复杂度、请求数量和协议开销也可能更高。REALITY 官方目前列明可以与 XHTTP 组合。

评价
流量外观：A
抗主动探测：A
效率：A-
复杂度：高
适合：移动网络、需要 HTTP 化或更灵活传输行为的 443/TCP 备用入口
4. Trojan over TLS

Trojan 的思路是让连接看起来像标准 TLS/HTTPS，并将未认证请求回落到网站。它仍然可用，但单独的 Trojan 并不能保证客户端 TLS 指纹、包长和时序与浏览器一致。2022 年公开测量曾记录 Trojan、VLESS、WebSocket、gRPC 等 TLS 代理遭到集中封锁，并怀疑客户端 ClientHello 指纹是识别因素之一。

评价
加密安全：A
流量外观：B
主动探测防御：B+
性能：A-
现状：可作为兼容入口，不建议作为最高对抗环境中的唯一主协议

如果客户端使用真实浏览器网络栈或维护良好的 uTLS 指纹，风险会低于固定 TLS 指纹实现，但仍不等于 NaïveProxy。

5. VLESS/VMess + WebSocket 或 gRPC + TLS

这类方案的优势主要是生态成熟、可通过常规反向代理部署，并适合与 CDN 或边缘代理结合。问题在于 TLS 外层、HTTP 升级行为、路径、持续连接模式、包长和多路复用模式可能联合形成特征。2022 年的公开报告已经记录过相关节点被精准封锁，因此今天更适合作为兼容或 CDN 场景，不适合作为最优直连方案。

相对排序
gRPC + TLS：语义比普通裸 TLS 丰富，性能中上，但长连接模式可能明显。
WebSocket + TLS：兼容性最好，额外封装较多，性能和隐蔽性均不占优势。
VMess：协议本身较复杂，今天通常不如 VLESS 轻量。
VLESS：应优先于 VMess，但上层传输和 TLS 实现才是决定抗识别能力的关键。
6. Tor WebTunnel、obfs4

WebTunnel 将 Tor 网桥流量放入类似常规 HTTPS/WebSocket 的行为中，需要域名、有效 TLS 证书和可配置的网站服务；obfs4 则属于“看起来像随机数据”的可插拔传输。Tor 官方目前支持 obfs4、Snowflake 和 WebTunnel。

在中国现有检测模型下，obfs4 的“随机高熵”不是绝对优势。公开研究表明，GFW 会通过常见协议豁免、可打印字符比例、比特比例和高熵特征筛查“完全加密但不像已知协议”的 TCP 流量，Shadowsocks、VMess 和 obfs4 都在影响范围内。

评价
WebTunnel：隐蔽性 A，性能 C+，匿名性 A
obfs4：隐蔽性 B-，性能 B-，抗主动探测 A
适合：需要 Tor 匿名网络，而不只是普通代理的情形
三、适合 UDP 443 或其他 UDP 端口的协议
1. Hysteria 2

Hysteria 2 构建在标准 QUIC 和 QUIC Datagram 上，支持 TCP、UDP 转发；未认证访问应表现为正常 HTTP/3 服务。其规范包含随机填充，并提供 Salamander 和 Gecko 混淆层，后者用于拆分和随机填充具有明显形状的 QUIC 握手数据报。

性能
丢包和高带宽时延积环境下表现突出。
Brutal 类速率模型可更主动地争取带宽。
TCP 业务被映射到 QUIC 流，不会出现传统 TCP-over-TCP 的双重拥塞问题。
UDP 转发使用 QUIC Datagram，效率较高。
风险
标准 QUIC Initial 可以被解密检查 SNI。
UDP 可能被运营商 QoS、限速或整体阻断。
过于激进的固定发送速率和持续大流量可能形成统计异常。
Salamander 会隐藏标准 QUIC 外观，但反过来呈现“未知高熵 UDP”，因此不是无条件提升，应看网络是否允许非 QUIC UDP。
评价
弱网吞吐：A+
延迟：A
标准 HTTP/3 外观：A-
加 Salamander/Gecko 后的多态性：A-
运营商兼容性：B
适合：UDP 443 主入口、移动网、高丢包链路
2. TUIC v5

TUIC 是基于 QUIC 的 TCP/UDP 代理规范，目标包括 0-RTT、完整锥形 NAT 兼容、可靠或不可靠 UDP、流多路复用、连接迁移和用户态拥塞控制。当前规范版本为 0x05，官方协议仓库不提供唯一官方实现，而由 sing-box、mihomo、shoes 等实现。

与 Hysteria 2 的差异
TUIC 更贴近“通用 QUIC 代理”，设计简洁。
Hysteria 2 更强调恶劣链路、高丢包下的带宽争取和 HTTP/3 伪装。
TUIC 可选拥塞控制更适合稳定或中等丢包网络。
TUIC 没有与 Hysteria 2 的 Salamander/Gecko 完全等价的统一规范层。
实际抗封锁效果高度依赖具体实现的 TLS/QUIC 指纹，而不是只看 TUIC 名称。
评价
延迟：A+
稳定网络效率：A
弱网吞吐：A-
流量外观：B+
实现一致性：B
适合：UDP 443 次入口、低延迟业务
3. NaïveProxy over QUIC/HTTP3

NaïveProxy 客户端也支持 quic:// 形式，通过 Chromium QUIC/HTTP3 栈承载 CONNECT 隧道，并继续应用填充逻辑。其优势是 QUIC/TLS 指纹更贴近 Chrome，但 HTTP/3 路径的部署兼容性通常不如 HTTPS/HTTP2。

评价
QUIC 指纹真实性：A+
主动探测防御：A+
性能：A
部署成熟度：B
适合：UDP 443 高隐蔽备用，但应保留 TCP 回退
4. MASQUE / CONNECT-UDP / CONNECT-IP

MASQUE 不是传统“小众翻墙协议”，而是 IETF 标准体系。RFC 9298 定义通过 HTTP 代理 UDP，可运行在 HTTP/1.1、HTTP/2 或 HTTP/3 上；HTTP/2 和 HTTP/3 使用 Extended CONNECT。RFC 9484 则进一步涵盖 CONNECT-IP。

优点
标准化程度高。
与浏览器、操作系统 VPN 和企业代理生态更容易融合。
HTTP/3 + CONNECT-UDP 的协议语义合理，不是简单“模拟”。
同时适合 IPv4、IPv6 目标，RFC 明确支持域名、IPv4 和 IPv6 字面地址。
局限
标准协议不自动等于抗封锁协议。
代理域名、SNI、认证模式或服务 IP 暴露后，仍可直接封锁。
自建客户端生态不如 VLESS、Hysteria 2、TUIC 完整。
更适合企业、系统级代理或研究型部署。
评价
标准化：A+
协议外观：A
性能：A
即用生态：B-
长期潜力：A+
四、非 443 端口可选协议

非 443 并不意味着协议必须改变。Hysteria 2、TUIC、SS2022、REALITY、NaïveProxy 和 Trojan 均可运行在其他端口，但“端口与流量语义是否合理”会影响外观。

非 443/TCP

推荐顺序：

VLESS + REALITY + Vision
NaïveProxy HTTPS
VLESS + XHTTP + REALITY
Trojan TLS
SS2022
WebSocket/gRPC + TLS
obfs4

运行 HTTPS 外观协议时，8443、2053、2083、2087、2096 等端口在互联网中确实存在 TLS 服务，但大规模审查可以直接识别“非标准端口上的 TLS”。因此非 443 的意义主要是故障隔离和备用路径，不是提高 TLS 拟态真实性。

非 443/UDP

推荐顺序：

Hysteria 2 + Gecko/Salamander，或标准 H3 模式
TUIC v5
NaïveProxy QUIC
MASQUE/CONNECT-UDP
WireGuard 类 VPN，仅适合低对抗环境

非 443 UDP 的优势是可以避开只针对 UDP 443 的 QUIC/SNI策略，同时使用端口轮换还能缓解单端口质量问题。但随机端口、高熵 UDP 和长期恒定速率也更容易被运营商视为非典型应用流量。

五、SS2022 应放在哪个层级

SS2022 使用 AEAD 和预共享密钥，强制完整重放保护，改进了基于会话的 UDP 代理，并使用 BLAKE3 派生子密钥。其明确不提供前向保密，因为设计选择是省去握手开销。

它的优点是：

实现简单、开销低。
AES-NI 设备上的 AES-GCM 性能很好。
UDP 效率高。
重放防御明显优于旧版 Shadowsocks。

它的核心问题是流量“类似随机字节流”，而不是“类似正常 HTTPS”。2023 年公开研究已经证明，高熵全加密流量可通过豁免式启发算法识别，SS、VMess、obfs4 等均受影响。

所以：

SS2022 适合非 443、高性能内网或低对抗网络。
不建议把 SS2022 裸跑作为唯一跨境主入口。
再套 TLS 不一定有益，不正确的 TLS-in-TLS 可能造成包长和时序特征，同时降低性能。
六、推荐组合矩阵
组合	入站要求	流量外观	主动探测	弱网性能	CPU效率	综合建议NaïveProxy HTTPS	TCP 443，IPv6	A+	A+	B+	B	443 隐蔽主入口
VLESS + REALITY + Vision	TCP 443，IPv6	A	A	A	A+	443 性能主入口
VLESS + XHTTP + REALITY	TCP 443/非443	A	A	A-	B+	移动/复杂路径备用
Hysteria 2 HTTP/3	UDP 443，IPv6	A-	A-	A+	A-	UDP 主入口
Hysteria 2 + Gecko/Salamander	UDP 非443优先	B 到 A-	A	A+	A-	非标准 UDP 备用
TUIC v5	UDP 443/非443	B+	B+	A	A	低延迟备用
NaïveProxy QUIC	UDP 443	A+	A+	A	B	高质量 QUIC 备用
MASQUE H3	UDP 443	A	取决于认证与前置	A	A-	标准化长期方案
Trojan TLS	TCP 443	B	B+	B+	A-	兼容入口
SS2022	TCP/UDP 非443	C+	B	A-	A+	低对抗、高效率
WS/gRPC + TLS	TCP 443/非443	B-	B	B	B	CDN/兼容场景
obfs4	TCP 非443	C+	A	B-	B	Tor 专用
WebTunnel	TCP 443	A	A	C+	C+	Tor 高对抗入口

这里的等级是架构评估，不是实测基准值。真实结果会受到客户端实现、运营商、路由、证书、域名信誉、IPv6 前缀、机房 ASN、并发量和业务流量形态影响。

七、针对“仅 IPv6 443 + 非 443 IPv6/NATv4”的最合理架构
方案 A：高性能双入口
TCP 443：VLESS + REALITY + Vision
UDP 443：Hysteria 2
非 443 UDP：TUIC 或 Hysteria 2 备用
出站：服务器根据目标选择原生 IPv6 或 NAT IPv4

特点：

视频、下载和大流量优先 REALITY/Vision。
弱网和 UDP 应用优先 Hysteria 2。
UDP 被限制时自动回落 TCP。
维护难度适中，综合性能最好。
方案 B：高拟态双入口
TCP 443：NaïveProxy HTTPS/HTTP2
UDP 443：NaïveProxy QUIC 或 Hysteria 2 HTTP/3
非 443 TCP：REALITY/Vision 备用
非 443 UDP：TUIC 备用

特点：

主入口尽可能复用 Chromium 网络行为。
TCP 和 UDP 均有浏览器协议外观。
CPU、内存和更新成本高于方案 A。
适合更重视对抗 TLS/QUIC 指纹分析的环境。
方案 C：稳定与匿名优先
TCP 443：WebTunnel/Tor bridge
TCP 非 443：obfs4 bridge
普通代理另设 REALITY 或 NaïveProxy

特点：

Tor 提供多跳匿名性，普通代理不提供同等级匿名性。
性能明显较低。
适合匿名需求，不适合追求极限下载吞吐。
八、截至 2026 年的“打击力度”判断
1. TCP 随机高熵协议：高风险

GFW 已公开表现出识别“完全加密但不符合常见协议”的能力，并使用比特比例、可打印字符、常见协议指纹等启发式豁免机制。该体系并不要求准确识别你使用的是 SS、VMess 还是 obfs4，只需要判断“它不像被允许的常见协议”。

受影响最大：

裸 SS/SS2022
裸 VMess
obfs4
私有随机加密协议
2. 普通 TLS 代理：中高风险

普通 TLS 并不等于真实浏览器。ClientHello、ALPN、TLS 扩展顺序、HTTP 层行为、证书、包长与连接时序都可能成为联合特征。2022 年的集中封锁已经证明，仅仅使用 TLS、WebSocket 或 gRPC 不足以保证长期稳定。

相对更优：

NaïveProxy，因其直接复用 Chromium 栈。
REALITY + 维护良好的浏览器指纹。
WebTunnel，因其实际进入标准 Web 服务路径。
3. QUIC/HTTP3 代理：风险正在上升

2025 年研究显示，GFW 能够解密 QUIC Initial、检查 SNI，并在发现黑名单 SNI 后对相同源 IP、目的 IP 和目的端口组合实施后续 UDP 丢包。超过 90% 的观测连接在一秒内受到阻断，但系统受负载影响且存在地域、时间和实现差异。

同时，QUIC 已成为大规模正常互联网流量的一部分，完全禁止 QUIC 的附带损害很高。因此，当前更现实的手段包括：

SNI 黑名单
QUIC 版本和 Initial 结构分析
UDP 443 选择性丢包
目标 IP/ASN 封锁
速率限制或 QoS
对可疑服务进行主动验证

这意味着 Hysteria 2、TUIC 和 NaïveProxy QUIC 仍有现实价值，但不应成为唯一入口。

4. 443 端口：不是护身符

2025 年 8 月的 74 分钟无条件 TCP 443 RST 事件说明，审查系统具备大范围干预 443 的技术能力，即使当时更可能是新设备、特殊状态或错误配置，而非永久政策。备用 TCP/UDP 入口和非 443 端口仍然必要。

九、最终排序
只选一个 TCP 443
NaïveProxy HTTPS，隐蔽性优先
VLESS + REALITY + Vision，性能优先
VLESS + XHTTP + REALITY，复杂路径适应优先
Trojan TLS
gRPC/WS + TLS
只选一个 UDP 443
Hysteria 2
NaïveProxy QUIC
TUIC v5
MASQUE/CONNECT-UDP
只选一个非 443
高性能：REALITY/Vision
弱网：Hysteria 2
低延迟：TUIC
简单低开销：SS2022
匿名：Tor obfs4
HTTP 拟态：NaïveProxy 或 WebTunnel
综合最优组合

TCP 443：NaïveProxy 或 REALITY/Vision
 UDP 443：Hysteria 2
 非 443 备用：TUIC 或第二个 REALITY 入口
 IPv4 目标：由服务端 NATv4/NAT64 出站解决

这是在隐蔽性、性能、UDP 可用性、TCP 回退和单端口故障隔离之间最均衡的结构。最不推荐的是“单一 IP、单一 TCP 443、单一私有协议、长期高流量且没有任何备用通道”的部署方式。