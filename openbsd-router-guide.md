# OpenBSD 路由器指南

**涉及网络分段防火墙，DHCP，支持 Unbound DNS，域名阻断等等** 

- OpenBSD：7.5
- 发布日期：2020-11-05
- 更新日期：2024-04-05
- 版本：2.1.9 

## 介绍

在本指南中，我们将探讨如何利用廉价和“低端”硬件构建一个功能强大的 OpenBSD 路由器，具备防火墙功能，分段局域网，带有域名阻断的 DNS，DHCP 等功能。

我们将使用一个设置，在此设置中，路由器将局域网（LAN）分成三个独立的网络：一个用于家里的成年人，一个用于孩子，还有一个用于公共服务器（DMZ），例如私有网页服务器或邮件服务器。我们还将看看如何使用 DNS 屏蔽广告、色情和其他互联网网站。OpenBSD 路由器还可以用于小型到中型办公室。


## 本指南使用的排版约定

* 终端命令、文件名和路径、配置参数等使用 Fixed-width （等宽）字体。
* 以 root 用户身份必须键入的终端命令以 # 井号开始，并且命令使用粗体文本。
* 作为常规用户可键入的终端命令以 $ 美元符号开始，并且命令使用粗体文本。

## 为什么需要防火墙？

注意：当前此指南仅涉及 IPv4，因为大多数人仍未使用 IPv6，许多 ISP 也仍然仅使用 IPv4，但 IPv6 计划在将来的指南更新中加入。

几乎无论您如何从家里或办公室连接到互联网，您都需要一个真正的防火墙，将您与 ISP 提供的调制解调器或路由器隔开。

消费者级调制解调器或路由器极少接收固件更新，它们经常容易受到网络攻击的影响，这些攻击可以将这些设备变成僵尸网络，例如 Mirai 恶意软件。许多消费者级调制解调器或路由器也是一些最大分布式拒绝服务（DDoS）攻击的原因之一。

你和 ISP 调制解调器或路由器之间的防火墙无法保护你的调制解调器或路由器设备免受攻击，但它可以保护你内部网络中的计算机和设备，并帮助你监控和控制进出本地网络的流量。

如果你的本地网络和 ISP 调制解调器或路由器之间没有防火墙，基本上可以将其视为开放的政策，就像将家里的门敞开一样，因为你无法信任 ISP 的设备。

在你的本地网络和互联网之间放置一个真正的防火墙总是一个非常好的主意，而且使用 OpenBSD 你可以得到一个非常可靠的解决方案。

## 硬件

您不必购买昂贵的硬件来获得您家或办公室的有效路由器和防火墙。即使使用廉价的"低端"硬件，您也可以获得非常稳定的解决方案。

我已经使用了配备英特尔四核赛扬处理器的 ASRock Q1900DC-ITX 主板构建了多个解决方案。

![ASRock Q1900DC-ITX motherboard](https://openbsdrouterguide.net/includes/img/asrock-q1900dc-itx.webp)

我承认，这是一块相当 "糟糕" 的主板，但它完成了工作，我有几个构建在千兆网络上运行多年，完全饱和，防火墙、DNS 等等都在加班，CPU 几乎毫不费力。

ASRock Q1900DC-ITX 主板有一个优点，它配有一个直流输入插孔，与 9~19V 电源适配器兼容，非常省电。不幸的是，ASRock Q1900DC-ITX 主板已经停产，但我只是用它作为一个例子，我也用过其他几块便宜的主板。

注意：其他主板生产商的许多低功率品牌也可以使用，例如 PC Engines 的著名 APU2。另一个选择是来自 Qotom 或 Jetway 的迷你电脑之一。

我也使用过 ASRock Q1900-ITX（它不带 DC-In 插孔），配合 PicoPSU。

![PicoPSU power supply](https://openbsdrouterguide.net/includes/img/picopsu.webp)

你可以找到不同品牌和版本的 PicoPSU，有些质量比其他的好。我有两个不同品牌的，一个是原版，一个是便宜的山寨货，它们都表现非常好，并且与使用普通电源相比，它们节省了相当多的电力。

最后，我使用的是在 Ebay 上找到的便宜的 Intel 山寨版四口port网卡，就像这个。

![Intel Quad NIC](https://openbsdrouterguide.net/includes/img/intel-quad-nic.webp)

我知道在乎你关心的网络上使用质量好的硬件会更好，但本教程是关于如何使用相对便宜的硬件也能得到一个非常实用的产品，并且至少根据我的经验，它会继续为你提供良好的服务多年。

我建议您寻找一个支持 OpenBSD 的低功耗迷你 ITX 主板，例如英特尔赛扬或英特尔 i3 处理器。这些主板通常价格便宜，耗电量较低，占用空间也少。如果您的网络速度为千兆位，则不建议使用英特尔 Atom CPU，因为它们通常无法处理大量流量，但实际效果可能有所不同。

如果您有多台计算机想连接到同一个局域网上，您可能还需要几个廉价的千兆位交换机 :)

## 为什么选择 OpenBSD？

实际上，你可以通过其他 BSD 之一flavors或许多不同的 Linux 发行版获得一种类似的设置，但 OpenBSD 专门为这类任务设计，非常合适。它不仅在基本安装中提供了所有必需的软件，而且具有显著更好的安全性和大量内置的改进措施。我强烈推荐 OpenBSD 而不是其他任何操作系统用于这种任务。

此外，OpenBSD 很特别，这并非言过其实。手册页非常易读，通常是您创建所需各种服务配置文件时唯一需要的信息。OpenBSD 项目对软件和手册页都有非常高的质量要求。

本指南不会告诉您如何安装 OpenBSD。 如果以前没有这样做过，我建议您启动一些虚拟机，或看看是否有未使用且支持的硬件可以玩耍。 OpenBSD 是最容易和最快速安装的操作系统之一。 不要害怕非 GUI 方法，一旦尝试过，您将真正欣赏其简单性。 当有疑问时，请使用默认设置。

在踏上这段旅程之前，请务必查阅 OpenBSD 文档！ 不仅一切都有很好的文档记录，而且您很可能会在其中找到所需的所有答案。 阅读 OpenBSD FAQ 并查看我们将使用的各种软件的不同手册页面。

关于 OpenBSD 的一处非常有用的信息来源是 OpenBSD 邮件列表档案。 还要确保通过订阅公告和安全公告邮件列表来及时了解相关信息。

提示：请考虑支持 OpenBSD！即使您在日常生活中不使用 OpenBSD，但或许在 Linux 上使用 OpenSSH，那么您实际上正在使用 OpenBSD 项目的软件。考虑进行小额但稳定的捐赠，以支持 OpenBSD 开发人员开发的所有优秀软件的进一步发展！

## 网络

路由器基本上是一种在两个或多个独立网络之间调节网络流量的设备。路由器将确保面向本地网络的网络流量不会进入互联网，互联网上不面向您本地网络的流量也会留在互联网上。

注意：路由器有时也被称为网关，这通常没问题，但事实上，真正的网关连接不同的系统，而路由器连接相似的网络。网关的一个例子是连接计算机网络与电信网络的设备。

在本教程中，我们正在构建一个路由器，我们有 4 个相同类型的网络可供使用。其中一个是互联网，另外三个是内部分段的局域网（LAN）。有些人喜欢使用虚拟局域网，但在本教程中，我们将使用上面插图中的四个 port 网卡。如果你更喜欢，你也可以通过使用多个 port 网卡来达到相同的效果，只需确保主板上有足够的空间和空闲的 PCI 插槽。你也可以使用主板上的以太网 port，但这取决于设备的驱动程序和支持情况。我使用 Realtek PCI 千兆以太网控制器没有遇到任何问题，这个控制器通常随许多主板一起提供，尽管我推荐 Intel 而不是 Realtek。

当然，如果你不需要将网络分割为几个部分，你不必这样做，从本指南中更改设置将非常容易，但为了向您展示如何通过将其分成独立的局域网来保护您的子女，我决定使用这种方法，这不仅可以使用 DNS 拦截（所有段都可以得到这个）、阻止广告和色情，而且您甚至可以制作一个通过列表，只传递您希望他们访问的互联网部分。关于通过列表的最后一部分是困难的，通常不建议，除非您的孩子只需要非常有限的访问权限，但通过一些工作是可行的，指南将向您展示您可以做到这一点的一种方式。

这是我们即将建立的网络示意图：

```
                       Internet
                          |
                    xxx.xxx.xxx.xxx
                    ISP Modem (WAN)
                      10.24.0.23
                          |
                       OpenBSD
                      10.24.0.50
                  (router/firewall)
                          |
     ┌────────────────────+────────────────────┐
     |                    |                    |
    NIC1                 NIC2                 NIC3 (DMZ)
192.168.1.1          192.168.2.1          192.168.3.1
LAN1 switch          LAN2 switch          LAN3 switch
     |                    |                    |
     └─ 192.168.1.x       ├─ 192.168.2.x       └─ 192.168.3.2
        Grown-up PC       |  Child PC1            Public web server
                          |
                          └─ 192.168.2.x
                             Child PC2
```

以 10.24.0 开头的 IP 地址是您的 ISP 路由器或调制解调器提供的任何 IP 地址，可能完全不同。以 192.168 开头的 IP 地址是我们在本指南中用于本地区域网络（LAN）的 IP 地址。

本指南不涉及任何类型的无线连接。无线芯片固件因其易受漏洞和攻击而臭名昭著，如果可以的话，我建议您不要使用任何形式的无线连接。如果确实需要无线连接，我强烈建议您完全禁用 ISP 路由器或调制解调器上的无线访问（如果可能的话），然后购买最好的无线路由器放在防火墙后面的隔离段中。这样一来，如果您的无线设备受到攻击，您可以更好地控制结果并限制损害。您还可以进一步设置无线路由器，使其连接的任何设备都具有自己的 IP 地址，这些 IP 直接通过无线路由器传递，同时阻止从无线路由器本身直接发出的流量。这样，您可以防止无线路由器向外传输数据。您还可以使用 OpenBSD 支持的无线适配器，并使您的 OpenBSD 路由器作为实际接入点运行，但我更倾向于将无线部分分隔到单独的无线路由器或防火墙本身后面的另一台 OpenBSD 机器作为无线接入点。

注意：目前据我所知，没有一个 OpenBSD 无线驱动程序是完全没有问题的。

### 设置网络

我们首先要设置的是在我们的 OpenBSD 路由器上的不同网卡。在我的特定机器上，我已经通过 BIOS 禁用了主板上集成的网卡，我只打算使用四个 Intel 的仿制网卡。

如果您正在按照本教程操作，且只需要基本防火墙，则至少需要两个独立的网卡。

在开始之前，请确保您已经阅读并理解了 hostname.if 手册页中的不同选项。还要看一下 OpenBSD FAQ 中的网络部分。

因为我使用的是 Intel，em 驱动程序是 OpenBSD 加载的驱动程序，每个 NIC 上的port都列为单独的卡。这意味着每张卡都以 emX 列出，其中 X 是给定卡上实际的port号码。

一个 dmesg 列出了四个 ports 的 NIC，如下所示：

<pre><b># dmesg</b>
em0 at pci2 dev 0 function 0 "Intel I350" rev 0x01: msi, address a0:36:9f:a1:66:b8
em1 at pci2 dev 0 function 1 "Intel I350" rev 0x01: msi, address a0:36:9f:a1:66:b9
em2 at pci2 dev 0 function 2 "Intel I350" rev 0x01: msi, address a0:36:9f:a1:66:ba
em3 at pci2 dev 0 function 3 "Intel I350" rev 0x01: msi, address a0:36:9f:a1:66:bb</pre>

这表明我的卡被识别为 Intel I350-T4 PCI Express 四端口 Port 千兆位 NIC。

下一步是找出哪个 port 物理上与上面列出的编号匹配。您可以通过手动将一根以太网线插入一个活跃（已打开）的交换机，调制解调器或路由器中，依次将其插入每个 port，以查看哪个 port 被激活，然后将其记录在某个地方。

您可以使用 ifconfig 命令检查活动状态。 没有以太网电缆的port将在 status 字段中列为 no carrier ，而带有电缆连接的port将列为 active 。就像这样：

<pre><b># ifconfig</b>
em1: flags=8843<up,broadcast,running,simplex,multicast> mtu 1500
        lladdr a0:36:9f:a1:66:b9
        index 2 priority 0 llprio 3
        media: Ethernet autoselect (none)
        <b>status: active</b>
em2: flags=8843<up,broadcast,running,simplex,multicast> mtu 1500
        lladdr a0:36:9f:a1:66:ba
        index 3 priority 0 llprio 3
        media: Ethernet autoselect (none)
        <b>status: no carrier</b></up,broadcast,running,simplex,multicast></up,broadcast,running,simplex,multicast></pre>

我们将使用 em0 port作为连接到来自 ISP 的调制解调器或路由器的那个。换句话说，互联网。在我的具体情况下，我从 ISP 那里获得了一个公共 IP 地址，如果您想要从家中运行类似 Web 服务器的东西，则需要该 IP 地址，但如果您不需要，可以使用 DHCP 设置卡。

在我这种情况下，我需要为 em0 输入特定的固定 IP 地址，然后我的 ISP 将流量从我的公共 IP 转发过来。为此，我使用以下信息设置 em0 卡：

<pre><b># echo 'inet 10.24.0.50 255.255.254.0 NONE' > /etc/hostname.em0</b></pre>

如果您不需要公共 IP 地址，并且通过 ISP 使用 DHCP 获取 IP，则只需输入 dhcp 即可：

<pre><b># echo 'dhcp' > /etc/hostname.em0</b></pre>

然后，我将使用我之前说明的 IP 地址设置 NIC ports 的其余部分。

```
# echo 'inet 192.168.1.1 255.255.255.0 NONE' > /etc/hostname.em1
# echo 'inet 192.168.2.1 255.255.255.0 NONE' > /etc/hostname.em2
# echo 'inet 192.168.3.1 255.255.255.0 NONE' > /etc/hostname.em3
```

查看 hostname.if 以获取更多信息。

然后，我需要设置 ISP 网关的 IP。根据你的 ISP 设置，这个 IP 地址可能不同于 ISP 模式或路由器的 IP 地址。如果不添加 /etc/mygate ，路由表中不会添加默认网关。如果你的 IP 是通过 ISP 模式或路由器的 DHCP 获取的，那么你就不需要 /etc/mygate 。如果在任何 hostname.ifX 中使用 dhcp 指令，则将忽略 /etc/mygate 中的条目。这是因为通过 DHCP 服务器获取 IP 地址的网卡还会获得提供的网关路由信息。

最后，但同样重要的是，我们需要启用 IP 转发。IP 转发是使 IP 数据包能够在路由器的各个网络接口之间传输的过程。默认情况下，OpenBSD 不会在各个网络接口之间转发 IP 数据包。换句话说，路由功能（也称为网关功能）被禁用了。

我们可以使用以下命令来启用 IP 转发：

```
# sysctl net.inet.ip.forwarding=1
# echo 'net.inet.ip.forwarding=1' >> /etc/sysctl.conf
```

现在 OpenBSD 将能够将 IPv4 数据包从一个网卡转发到另一个网卡。或者，就像在我们的特定情况中有四个port网卡，从一个port到另一个。如果需要 IPv6，请查看手册。

## DHCP

现在我们准备设置动态主机配置协议（DHCP）服务，我们将为连接到不同局域网的不同 PC 和设备运行该服务。在开始之前，请确保您已阅读并理解 dhcpd.conf 手册中的各种选项。还请查看 dhcpd 支持的选项的 dhcp-options 手册。

我们有选项将特定的 IP 地址绑定到连接到我们不同 LAN ports 的特定计算机或设备。如果我们要将来自互联网的任何流量转发到诸如 Web 服务器之类的东西，则需要这样做。我们可以通过相关计算机的 NIC 上的 MAC 地址将特定的 IP 地址绑定到特定计算机。

在这种情况下，我将为 DHCP 保留从 10 到 254 的所有 IP 地址，同时将为我可能需要的任何固定地址留下一些剩余的地址。

使用您喜欢的文本编辑器编辑 /etc/dhcpd.conf ，并设置它以满足您的需求。

```
subnet 192.168.1.0 netmask 255.255.255.0 {
    option domain-name-servers 192.168.1.1;
    option routers 192.168.1.1;
    range 192.168.1.10 192.168.1.254;
}
subnet 192.168.2.0 netmask 255.255.255.0 {
    option domain-name-servers 192.168.2.1;
    option routers 192.168.2.1;
    range 192.168.2.10 192.168.2.254;
}
subnet 192.168.3.0 netmask 255.255.255.0 {
    option domain-name-servers 192.168.3.1;
    option routers 192.168.3.1;
    range 192.168.3.10 192.168.3.254;
    host web.example.com {
        fixed-address 191.168.3.2;
        hardware ethernet 61:20:42:39:61:AF;
        option host-name "webserver";
    }
}
```

option domain-name-servers 行指定了我们将在路由器上运行的 DNS 服务器。

此外，作为公共局域网上充当 Web 服务器的计算机已获得了固定 IP 地址，并提供了固定的主机名。

另外，如果您不想将网络分段为不同的部分，而只想拥有一个局域网，那么您可以只留下其他子网，这样您就只有这个。

```
subnet 192.168.1.0 netmask 255.255.255.0 {
    option domain-name-servers 192.168.1.1;
    option routers 192.168.1.1;
    range 192.168.1.10 192.168.1.254;
}
```

然后我们只需要确保启用并启动 dhcpd 服务：

```
# rcctl enable dhcpd
# rcctl start dhcpd
```

注意：请查看附录中添加域名选项到 DHCP 和使用 FQDN 部分，了解如何轻松地将完全限定域名（FQDN）添加到您的设置中，以及如何在 DHCP 中使用 domain-name 选项以避免每次需要时都必须输入 FQDN。本节还将向您展示如果您的局域网有多个计算机或设备连接，如何避免记住 IP 地址。

## PF - 一个包过滤防火墙

包过滤防火墙检查每个穿越防火墙的数据包，并根据检查数据包的 IP 和协议头中的字段来决定是否接受或拒绝单个数据包，根据您指定的规则。

包过滤器通过检查每个传输控制协议/因特网协议（TCP/IP）数据包中包含的源 IP 和port地址以及目标 IP 和ports地址来工作。 TCP/IP ports是分配给特定服务的数字，用于识别每个数据包的目的服务。

简单数据包过滤防火墙的一个常见弱点是防火墙独立检查每个数据包，而不考虑之前通过防火墙的数据包和接下来可能跟随的数据包。这称为“无状态”防火墙。利用无状态数据包过滤器相对容易。来自 OpenBSD 的 PF 不是无状态防火墙，而是有状态防火墙。

有状态的防火墙会跟踪已打开的连接，并且只允许符合现有连接或建立新的允许连接的流量通过。当匹配规则上指定状态时，防火墙会动态生成每个预期在会话期间交换的数据包的内部规则。它具有足够的匹配能力来确定数据包是否对会话有效。任何不能正确匹配会话模板的数据包都会被自动拒绝。

有状态过滤的一个优点是它非常快速。它允许您专注于阻止或通过新会话。如果通过了新会话，那么其后续数据包都会被自动允许通过，而任何冒名顶替的数据包都会被自动拒绝。如果阻止了新会话，则不允许其后续数据包通过。有状态过滤还提供了高级匹配能力，能够抵御攻击者使用的各种攻击方法的洪水。

网络地址转换（NAT）使防火墙后面的私有网络可以共享单个公共 IP 地址。NAT 允许私有网络中的每台计算机都可以通过互联网访问，而无需多个互联网帐户或多个公共 IP 地址。当数据包从防火墙出口前往互联网时，NAT 会自动将私有网络中的计算机或设备的 IP 地址转换为单个公共 IP 地址。NAT 还会对返回的数据包执行相反的转换。使用 NAT，您可以将特定流量重定向到来自互联网的公共 IP 地址上的特定服务器或位于本地网络某处的几台服务器，通常是由port号或一系列port号确定。

Packet Filter (PF) 是 OpenBSD 的防火墙系统，用于过滤 TCP/IP 流量并执行 NAT。PF 还能够规范和调节 TCP/IP 流量，提供带宽控制和数据包优先级。

PF 由整个 OpenBSD 团队积极维护和开发。

## PF 设置

在我们开始之前，我假设您已经阅读了 PF 用户指南和 pf.conf 手册，尤其是后者非常重要。即使您不完全理解所有不同的选项，也请确保阅读文档！要全面深入地了解 PF 的功能，请查阅 pf 手册。

另外，我要先说，虽然 PF 的语法非常易读，但编写防火墙规则时很容易出错。即使是资深和经验丰富的系统管理员在编写防火墙规则时也会犯错。

编写防火墙规则要求您仔细规划您的目标，理解如何实施不同的规则以达到期望的结果，同时要小心防止出错并意外地将自己登出 :) 我想我们都曾经在匆忙、疲劳或仅仅是一时失误中经历过这种情况，我自己也经历过好几次。

请注意：我已经尽力保持简单，并尽量使用大量注释来解释每条规则的作用。同时，我已经测试了每条规则并监控了其影响，并且通常尽力避免复杂性和错误。

最重要的部分是您不要做出任何假设。始终彻底测试您的规则。如果有什么问题，尽量从规则中尽可能多地移除内容，以便剩下非常基本的部分。然后逐一引入一条规则，直到您发现某个规则造成问题为止。然后逐步处理设置。

真正困难的部分是记住数据包如何到达一个网卡，然后如何被转发到另一个网卡上的机器，并正确地将这个"旅程"与传入、传出、阻止传入、阻止传出、从哪里到哪里的术语关联起来。这些术语通常并不完全按照我们的想法运作。

我试图思考的方式或多或少是真的想象自己在“数据流中流动”，站在特定的 NIC port，看着数据进出。这样我记得要考虑数据是从特定设备（附加到特定的port）“进”或“出”等。听起来很傻，但这种方法帮助过我不止一次。

### 澄清

我想首先澄清 PF 中一些常见默认设置和关键字。

当我们谈论我们传入或传出的流量时，一个记住我们正在处理什么的好办法是以数据包为单位思考。我们将来自计算机的数据包传入到网络接口卡（与该网络接口卡连接的计算机）中，我们将来自网络接口卡的数据包传出到连接到它的计算机中。

格式要么是然后我们在目的地上过滤数据包：

<pre><b>from</b> <i>source IP</i> <b>to</b> <i>destination IP</i> <b>[on]</b> <i> port</i></pre>

要么我们在源上过滤数据包：

<pre><b>from</b> <i>source IP</i> <b>[on]</b> <i>port</i> <b>to</b> <i>destination</i></pre>

请注意， [on] 部分不是 PF 语法的一部分。

* `quick`

  * 如果数据包匹配 pass ， block 或 match 规则，并带有 quick 修饰符，则数据包将在不检查后续过滤规则的情况下传递。带有 quick 修饰符的规则将成为最后匹配的规则。
* `keep state`

  * 对于特定的 pass 或 block 规则，您无需指定 keep state 修饰符。当数据包第一次匹配到 pass 或 block 规则时，默认情况下会创建状态条目。只有当没有规则匹配数据包时，默认操作是在不创建状态的情况下传递数据包。
* on 接口 / any

  * 这个规则仅适用于进入或穿出这个特定接口或接口组的数据包。 on any 修饰符 - 将匹配除环回接口外的所有现有接口。
* `inet`/`inet6`

  * inet 和 inet6 修饰符意味着此规则仅适用于进入或穿出这个特定路由域的数据包，即 IPv4 或 IPv6。您可以将规则应用于特定的路由域而不指定网络接口卡。在这种情况下，该规则将匹配所有该特定属性的所有网络接口卡上的所有流量。通过指定 inet ，您明确地处理的是 IPv4 流量。
* `proto`

  * 使用 proto 修饰符来限制协议。规则仅适用于该协议的数据包，其他协议不受影响。您可以在 /etc/protocols 中查找协议。常见协议包括 ICMP、TCP 和 UDP。
* in 和 out

  * 这是流量定向中最容易出错的部分之一。数据包始终进入或通过以太网 port 进行传出，在以太网接口上。 in 和 out 适用于通过物理以太网 port 进入和传出数据包，以太网电缆连接的地方。如果都未指定，则规则将匹配两个方向的数据包。 in 和 out 永远不用于处理从一个网卡到另一个网卡的流量，这是通过网络地址转换（NAT）来完成的，使用选项 nat-to 和 rdr-to 。 in 和 out 仅处理连接卡上物理以太网 port 的流量。
* 源和目的地

  * The 源和目的地 rule modifiers apply only to packets with the specified source and destination addresses and ports. Both the 主机名或 IP 地址, port, and OS specifications are optional. When we're dealing with a router with multiple NICs it's easy to think like this: I want to pass in packets from the external NIC (the NIC attached to the Internet) and then have them go to the first LAN NIC and from there out to a specific computer on that LAN, meaning we follow the "trail of data" in our minds, and then write that out into something like this: pass in on $ext_if from $ext_if to $dmz port 80 . But this will not make the HTTP traffic "magically" appear on port 80 on the LAN with a computer attached with a specific IP address. We would also require a specific pass out rule and furthermore need to determine exactly on which machine we want the data to end up. Unless you are really dealing with a very specific requirement, you never need such rules in your ruleset! The Unicast Reverse Path Forwarding (uRPF) features of PF will protect your internal network very well and with a basic setup of correct network address translation (NAT), with the 定制 option, and redirection with the 软件包 option, PF will handle the packages from the inside to the outside and vice versa. The 镜像 parameter is equivalent to writing Wrapper . Without explicitly declaring the direction, the default is 默认 . This rule: 过程 translates into this: 压缩包 There is also no need to use bug , the issue command part is the default. You do however need the 凭据
* 源和目的地

  * 网络地址转换（NAT）选项可以修改与有状态连接相关的数据包的源地址或目标地址以及port。PF 会修改数据包中指定的地址和/或 port，并在必要时重新计算 IP、TCP 和 UDP 校验和。 nat-to 选项指定在数据包通过指定的接口时需要更改 IP 地址。这种技术允许在翻译主机（OpenBSD 路由器）上使用一个或多个 IP 地址来支持内部网络（即 LAN）中更多计算机的网络流量。 nat-to 选项通常用于外向流量，即从内部网络重定向到 Internet。 nat-to 本地 IP 地址是不支持的。 rdr-to 选项通常用于内向流量，即从 Internet 重定向到内部网络。
* 列出项和地址范围以及 ports

  * 当你需要指定多个项时，例如多个 port 数字，你可以用空格或逗号分隔它们。像这样 port { 53 853 } 或像这样 port { 53, 853 } 。地址范围使用 - 运算符指定。例如 192.168.1.2 - 192.168.1.10 表示从 192.168.1.2 到 192.168.1.10 的所有 IP 地址（包括这两个地址）。ports 的范围有多个参数，请查看 pf.conf 的手册页并搜索文本 Ports，而 ports 的范围使用这些运算符来指定。

警告: 请注意，每当经过数据包过滤器处理的数据包通过网络接口进入或离开时，过滤规则将按顺序评估，从第一个到最后一个。对于 block 和 pass ，最后匹配的规则决定采取的操作。如果没有规则匹配数据包，则默认操作是在不创建状态的情况下传递数据包。对于 match ，每次匹配时都会评估规则。

### 域名或主机名解析

如果您决定在 PF 设置中使用主机名和/或域名，则需要了解所有的域名解析和主机名解析都是在规则集加载时完成的。这意味着当主机的 IP 地址或域名发生更改时，必须重新加载规则集才能反映内核中的更改。并非每次运行特定规则时，具有列出主机名或域名的规则就会进行新的 DNS 查找以获取该特定主机名或域名。 DNS 查找仅在加载规则集时发生。

这也意味着在启动 PF 之前，您必须确保正在使用的 DNS 服务器正常运行，否则 PF 将在加载规则集时失败，因为无法解析主机名或域名。

在 OpenBSD 上，PF 在 Unbound 或任何其他安装的 DNS 服务器之前启动，从安全的角度来看，这是正确的做法。

我建议在使用 PF 规则时避免使用主机名或域名，尽可能使用 IP 地址。虽然可以使用主机名和域名，但直接使用 IP 地址是最简单和最安全的做法。

### 规则集

在测试机器上测试您的规则集是一个好主意。通常有多种实现相同结果的方法。在我看来，最好的方式是最清晰易懂的方式。

警告：除非您确实知道自己在做什么，否则请勿在您正在远程登录的设备上编写新的规则集。被远程机器强制注销从来都不是什么有趣的事情。

尝试尽可能保持规则清晰简洁，尽量使用默认值。不过，如果需要更清晰地理解规则，也可以不怕指定修饰符，即使它们与默认值相同。配置文件中的文本可能会说 any to any ，这样可以更容易理解特定的规则。

您可以随时解析规则集并检查错误，而无需部署它，使用命令 pfctl -nf /etc/pf.conf 。一旦使用命令 pfctl -f /etc/pf.conf 加载了规则集，您可以使用 pfctl -s rules 命令查看 PF 如何翻译规则集，我建议您定期使用这个命令。

我喜欢用章节和注释来组织我的规则集，所以在这个示例中我也会这样做。

使用您最喜欢的文本编辑器打开文件 /etc/pf.conf 。

首先，我们设置一些宏来更好地记住我们用于什么的网络接口控制器。使用网络接口控制器的宏还可以轻松更改卡的驱动程序名称，如果我们购买新卡或多个新卡。

```
#---------------------------------#
# Macros
#---------------------------------#

ext_if="em0" # External NIC connected to the ISP modem (Internet).
g_lan="em1"  # Grown-ups LAN.
c_lan="em2"  # Children's LAN.
dmz="em3"    # Public LAN (DMZ).
```

接下来，我们设置一个非路由 IP 地址表。我们这样做是因为非常常见的网络配置错误是一种让具有非路由地址的流量流向互联网的错误。我们将在我们的规则集中使用该表来阻止任何尝试通过路由器的外部网络接口发起联系到非路由地址的行为。

```
#---------------------------------#
# Tables
#---------------------------------#

# This is a table of non-routable private addresses.
table <martians> { 0.0.0.0/8 10.0.0.0/8 127.0.0.0/8 169.254.0.0/16     \
                   172.16.0.0/12 192.0.0.0/24 192.0.2.0/24 224.0.0.0/3 \
                   192.168.0.0/16 198.18.0.0/15 198.51.100.0/24        \
                   203.0.113.0/24 }
```

警告：请注意，宏和表格总是放在 /etc/pf.conf 的顶部。

然后我们开始一个默认的阻止策略，并设置一些保护功能。

```
#---------------------------------#
# Protect and block by default
#---------------------------------#

set skip on lo0

# Spoofing protection for all NICs.
block in from no-route
block in quick from urpf-failed

# Block non-routable private addresses.
# We use the "quick" parameter here to make this rule the last.
block in quick on $ext_if from <martians> to any
block return out quick on $ext_if from any to <martians>

# Default blocking all traffic in on all LAN NICs from any computer or device
# attached.
block return in on { $g_lan $c_lan $dmz }

# Default blocking all traffic in on the external NIC from the Internet/ISP,
# we'll log that too.
block drop in log on $ext_if

# Allow ICMP.
match in on $ext_if inet proto icmp icmp-type {echoreq } tag ICMP_IN
block drop in on $ext_if proto icmp
pass in proto icmp tagged ICMP_IN max-pkt-rate 100/10

# We need the router to have access to the Internet, so we'll default allow
# packets to pass out from our router through the external NIC to the Internet.
pass out inet from $ext_if
```

martians 宏中的 IP 地址构成 RFC1918 地址，不应在互联网上使用。由于这些 IP 地址不属于互联网，因此被称为“火星人”，因为它们就像来自火星一样。这些地址也被称为 bogons。到达和从这些地址的流量将在路由器的外部接口上被丢弃。

注意：尽管我们在 block drop in log on $ext_if 语句中隐式阻止了“martians” IP 地址，这默认阻止了所有内容，但我们仍首先显式地阻止“martians” IP 地址。这是最佳实践，因为即使有正确配置的路由器处理网络地址转换，我们也需要防范因错误配置而导致的问题。常见的错误配置包括允许非可路由地址的流量流向互联网。由于来自非可路由地址的流量可能参与多种 DoS 攻击技术及其他问题，从安全角度考虑，通过外部接口显式阻止非可路由地址的流入网络被认为是一种最佳实践。

在本指南的早期版本（1.5.0 版本之前），我在上述设置中使用了“scrub”语句，但在与 OpenBSD 团队的 Henning Brauer（谢谢 Henning！）协商并进行进一步研究后，我决定将其移除，因为它处理非常特定的边缘情况（请参阅文档）。如果您的网络中的主机生成了带有“dont-fragment”标志的分段数据包，则只需 scrub 规则。默认的 PF 行为在没有 scrub 规则的情况下更适合一般使用。

OpenBSD FAQ 中包含了一个非常基本的路由器示例设置，其中 scrub 具有一些特定的值，但我的建议是仅在确实需要时才使用 scrub 。如果确实需要，将其插入到配置中 set skip 循环接口规则之后，像这样：

```
set skip on lo0
match in all scrub
```

然后根据您的需要添加任何参数到 scrub 规则中。

我过去也在欺骗保护部分使用以下反欺骗规则：

```
antispoof quick for { $g_lan $c_lan $dmz }
```

自从使用 PF 的单播反向路径转发（uRPF）功能提供与反欺骗规则相同的功能后，我已经删除了 antispoof 规则，因此我们不再需要它，而是直接使用 block in quick from urpf-failed 规则。

antispoof 修改器的以下信息仅供教育目的保留。

欺骗是指有人伪造 IP 地址。 antispoof 修改器会扩展为一组过滤规则，这些规则将阻止所有具有来自网络（直接连接到指定接口）的源 IP 的流量通过任何其他接口进入系统。有时也称为"渗入"或"渗透"。

PF 将上述 antispoof 指令转换为以下内容：

```
block drop in quick on ! em1 inet from 192.168.1.0/24 to any
block drop in quick inet from 192.168.1.1 to any
block drop in quick on ! em2 inet from 192.168.2.0/24 to any
block drop in quick inet from 192.168.2.1 to any
block drop in quick on ! em3 inet from 192.168.3.0/24 to any
block drop in quick inet from 192.168.3.1 to any
```

如果我们以 em1 NIC 规则 block drop in quick on ! em1 inet from 192.168.1.0/24 to any 为例，那意味着：阻止任何来自 IP 地址范围从 192.168.1.1 到 192.168.1.255 的网络流量，这些流量不是源自 em1 NIC 本身，并且正在前往任何地方。由于 em1 NIC 是负责该特定范围内所有 IP 地址的 NIC，则不应该从任何其他 NIC 发起带有这种 IP 地址的流量。

警告： antispoof 的使用应限制于已分配 IP 地址的接口，这意味着，如果您有未使用的 NIC，或者ports 在一个 NIC 上，请确保为每个分配 IP 地址，或者不要将它们包含在 antispoof 选项中。

如前所述，我已移除 antispoof 规则，而是使用严格的 uRPF 检查。当一个数据包通过 uRPF 检查时，数据包的源 IP 地址将在路由表中查找。如果出站接口在路由表中找到，并且条目与数据包刚刚进入的接口相同，则 uRPF 检查通过。否则，可能数据包已经伪造其源地址，并且将其阻塞。

在我们的设置中，我们允许 ICMP，尽管一些网络管理员完全阻止 ICMP。人们主要完全阻止 ICMP 是因为不必要的行为，比如网络发现攻击，隐秘通信渠道，ping 扫描，ping 洪泛，ICMP 隧道和 ICMP 重定向。然而，ICMP 远不止是回应 ping。如果我们完全阻止 ICMP，诊断、可靠性和网络性能可能会因此受到影响，因为重要的机制在限制 ICMP 协议时被禁用了。

一些不应该阻止 ICMP 的原因：

* 路径 MTU 发现（PMTUD）用于确定连接源和目的地的网络设备上的最大传输单元大小，以避免 IP 分段。TCP 依赖于 ICMP 类型 3 代码 4 的数据包进行“路径 MTU 发现”。当一个数据包超过连接路径上网络设备的 MTU 大小时，将返回 ICMP 类型 3、代码 4 和最大数据包大小。当这些 ICMP 消息被阻止时，目的系统不断请求未传递的数据包，源系统则继续无休止地重新发送，但没有任何效果。这种行为可能导致 ICMP 黑洞（拥塞的 IP 连接和中断的传输）。
* 存活时间（TTL）定义了数据包的寿命。ICMP 被阻止的网络不会接收类型 11，时间超过，代码 0，传输中超时的错误消息。这意味着源主机不会收到通知，以增加数据的寿命，以成功到达目的地，如果数据报无法到达目的地。
* 由于阻止了 ICMP 重定向而导致性能不佳。ICMP 重定向用于通知路由器从源主机到目的主机的直接路径。这减少了数据必须传输到目的地的跳数。如果阻止了 ICMP，主机将无法了解到达目的地的最佳路由。

在上述设置中，我们允许 ICMP，但对路由器将回答的 ping 请求数量设置“速率限制”。使用 max-pkt-rate 100/10 修饰符，如果我们在 10 秒内收到超过 100 个 ping，则路由器将停止响应 ping。

注意：如果出于某种原因你仍然想彻底阻止 ICMP，请简单地删除“允许 ICMP”注释后的 3 条规则。

现在我们来到家里大人们的 LAN 段。

```
#---------------------------------#
# Grown-ups LAN Setup
#---------------------------------#

# Allow any computer or device on the grown-ups LAN to send data packets in
# through the NIC. This means any computer attached to this network interface
# can pass in data reaching anywhere, i.e. the Internet or any of the computers
# attached to the router.
pass in on $g_lan

# Always block DNS queries not addressed to our DNS server.
block return in quick on $g_lan proto { udp tcp } to ! $g_lan port { 53 853 }

# I have a network printer I don't want to "phone home", so I block that.
# The network printer has the IP address 192.168.1.8.
block in quick on $g_lan from 192.168.1.8

# Allow data packets to pass from the router out through the NIC to the
# computers or devices attached to it on the grown-ups NIC.
# Without this we can't even ping computers attached to the grown-ups NIC from
# the router itself.
pass out on $g_lan inet keep state
```

在这个例子中，我有一个连接到大人们网络的网络打印机，我不希望它访问互联网或任何其他地方（以防它有某种间谍固件）。我通过阻止任何来自 IP 地址 192.168.1.8 到任何 IP 地址的数据进入 em1 来实现这一点。

此外，我们确保只要 DNS 请求不是发往我们的 DNS 服务器，就始终会阻止在port 53（普通 DNS）和 853（DNS over TLS）上的所有 DNS 请求。

注意：以前我曾将所有未发往我们的 DNS 服务器的port 53 上的流量重定向回我们的 DNS 服务器。我这样做是因为当我们阻止port 53 上的 DNS 请求时，无论使用 return 还是 drop ，请求都会在客户端上超时，这会导致大多数客户端回复延迟。我已经改为阻止，因为我认为这是更正确的方法。所有客户端都需要意识到通信在port 53 上被阻止，除非是发往我们的 DNS 服务器。这在我们排除网络故障时也很重要。如果我们从我们的 DNS 服务器收到重定向的回复，我们可能没有注意到自己已被重定向。

注意：DNS 主要使用用户数据报协议（UDP）在port号端口 53 上提供请求，但当答案长度超过 512 个字节并且客户端和服务器都支持 EDNS 时，会使用更大的 UDP 数据包。否则，查询将再次使用传输控制协议（TCP）发送。一些 DNS 解析器实现对所有查询使用 TCP。因此，我们需要在port号端口 53 的规则中同时支持 UDP 和 TCP 协议。

局域网的儿童部分非常相似。

```
#---------------------------------#
# Childrens LAN Setup
#---------------------------------#

# Allow any computers or devices on the childrens LAN to send data packets in
# through the NIC. This means any computer attached to this network interface
# can pass in data reaching anywhere, i.e. the Internet or any of the computers
# attached to the router.
pass in on $c_lan

# Always block DNS queries not addressed to our DNS server.
block return in quick on $c_lan proto { udp tcp} to ! $c_lan port { 53 853 }

# Allow data packets to pass from the router out through the NIC to the
# computers or devices attached to it on the grown-ups NIC.
# Without this we can't even ping computers attached to the childrens NIC from
# the router itself.
pass out on $c_lan inet keep state
```

目前，无论是成年人还是孩子都可以同样访问互联网。在儿童通过列表部分提到了一个更严格的设置。

然后我们进入 DMZ，也就是有一个面向公众的网络服务器的网卡。由于我们有一个面向公众的网络服务器，所以我们设置了一些限制。如果网络服务器被入侵，入侵者将很难找出我们内部网络上的其他位置。

我们除了 DHCP 外，阻塞了所有访问，以便 Web 服务器可以从我们的路由器获得 IP 地址，然后只有在需要更新机器或执行其他操作时才手动打开其他东西。当需要打开事物时，我已经注释了我们需要的选项，保留了限制性部分的启用。当您需要更新服务器时，要开放 DNS 和对互联网的一般访问。

注意：与其每次需要为 Web 服务器更新而手动更改规则集，我们也可以使用锚点，但为简单起见，我们在这里不这样做。

```
#---------------------------------#
# DMZ Setup
#---------------------------------#

# Allow any computer or device attached to the DMZ NIC to make DNS queries
# (uncomment if you need it).
#pass in on $dmz inet proto udp from any port 53

# Always block DNS queries not addressed to our DNS server on the router.
block return in quick on $dmz proto { udp tcp} to ! $dmz port { 53 853 }

# When you want any computer attached to the DMZ NIC to access the Internet
# uncomment this to open up for access. This is relevant when you need to
# upgrade a computer.
#pass in on $dmz inet

# No matter what, we do not want the DMZ segment to reach any of the other
# network segments so we explicitly use a block last.
#
# We have several options. If we use this:
#
#   block drop in on $dmz to 192.168/16
#
# Then we block for all subnets, but this also means that the computers
# attached to the DMZ NIC cannot do DNS queries when they need to be upgraded.
#
# In my opinion it is much better to be explicit and block the specific
# segments we want blocked.
#
# This blocks computers on the DMZ NIC from reaching any computers or devices
# on the other two networking segments.
block drop in on $dmz to { $g_lan:network $c_lan:network }

# Last, we need to pass out data packets coming from the DMZ NIC to computers
# attached to it, otherwise nobody can "talk" to them.
# Without this we cannot even ping computers attached to the DMZ NIC from the
# router itself.
pass out on $dmz inet keep state
```

现在我们来谈谈网络地址转换 (NAT)。这是路由器从网络的一个部分路由数据包到另一个部分的地方，在这种特定情况下，是从我们的内部网络到外部的互联网，然后从外部的互联网返回的任何回复，回到传输的发起者。我更喜欢 :network 参数，它翻译为与网络适配器连接的网络，并且我更喜欢为每个相关部分制定一个规则。

```
#---------------------------------#
# NAT
#---------------------------------#

pass out on $ext_if inet from $g_lan:network to any nat-to ($ext_if)
pass out on $ext_if inet from $c_lan:network to any nat-to ($ext_if)
pass out on $ext_if inet from $dmz:network to any nat-to ($ext_if)
```

PF 将跟踪所有流量，例如，成人 LAN 上的 Web 浏览器请求互联网上某个网站的网页时，来自互联网上的 Web 服务器的响应经由我们的外部 NIC，然后通过我们的内部成人 LAN NIC 直接传输到发起请求的计算机。

最后，我们来到规则集的重定向部分。这是我们允许来自互联网的流量进入到位于 DMZ NIC 上的公共 Web 服务器的地方。如果您没有任何需要重定向的公共面向服务器，当然可以跳过这部分。在本例中，我仅允许 IPv4 流量。

```
#---------------------------------#
# Redirects
#---------------------------------#

# Our web server is 192.168.3.2 - let the Internet have access to it.
pass in on $ext_if inet proto tcp to $ext_if port { 80 443 } rdr-to 192.168.3.2
```

警告：重定向始终放在规则集的最后！

这就是我们基本设置防火墙规则的全部内容。

### 儿童通行证名单

如果您想要为孩子们阻止整个互联网，但又想例外几个网站或者几个游戏服务器，您需要找出这些服务的 IP 地址，并创建一个使用这些 IP 地址的通行证名单。

如果它是一个单一网站，有一个单一的 IP 地址，那很容易，你可以在子部分的最后放置这个规则（你需要用相关的 IP 地址替换 x.x.x.x 部分）：

```
#---------------------------------#
# Childrens LAN Setup
#---------------------------------#

# Allow any computer or device attached to the childrens NIC to make DNS
# queries.
pass in on $c_lan inet proto udp from any port 53

# Always block DNS queries not addressed to our DNS server.
block return in quick on $c_lan proto { udp tcp} to ! $c_lan port { 53 853 }

# Then allow any computer or device attached on the childrens LAN to reach
# the IP address x.x.x.x only.
pass in on $c_lan to x.x.x.x
```

如果网站有多个 IP 地址，你需要弄清楚它们是什么。有时候域名查找可以一次性显示所有相关的 IP 地址，比如 $ dig example.com ANY 或 drill example.com ANY 。其他时候，你需要在一天中的不同时间间隔多次进行查找，以获取完整的 IP 地址范围。你还可以通过设置自动脚本来实现这一点。

有时候你可能需要联系相关公司，询问你是否可以获得 IP 范围以加入你的许可列表（有些公司将信息公开，而其他公司出于恶意使用的担忧拒绝发布信息）。一旦确定了 IP 范围，你可以将其放入 PF table ，然后使用它。

在此示例中，我们将向规则的表部分添加一个新表，然后更改子规则中的设置。

```
#---------------------------------#
# Tables
#---------------------------------#

...

# Whitelist for the children.
table <passlist> { x.x.x.x y.y.y.y z.z.z.z }
```

然后在子部分更改：

```
pass in on $c_lan to x.x.x.x
```

 到：

```
pass in on $c_lan to <passlist>
```

无法一次性获取所有所需的 IP 地址到一个通过列表中，但通过监视网络，例如使用 tcpdump，在游戏尝试访问服务器时，可以逐步组合一个有效的列表。我已经成功地在 Mojang 登录服务器、Minecraft 服务器和其他多个游戏中实现了这一点。

#### 使用持久表

用于通过列表收集 IP 的另一种方法是结合使用持久表和 /etc/rc.local 和域名查找。 /etc/rc.local 仅在启动 PF 后运行，因此域名解析问题不会对 PF 造成任何问题。

如果您想要使用持久表解决方案，可以通过在 /etc/pf.conf 中的表部分添加一个持久表来实现：

```
table <passlist> persist
```

在儿童部分，您仍然需要传递数据，该数据进入传递列表，就像上面所述：

```
pass in on $c_lan to <passlist>
```

然后在 /etc/rc.local 中，您可以添加以下命令：

```
pfctl -t passlist -T add example.com
```

您想要 PF 查找的域在哪里。

每当您的孩子因为有效的 IP 地址可能已更改而无法访问时，您可以登录防火墙，然后手动运行命令来手动更新表格，增加更多 IP 地址：

<pre><b># pfctl -t passlist -T add examples.com</b></pre>

如果您想查看已添加到列表中的 IP 地址，可以使用以下命令进行查看：

<pre><b># pfctl -t passlist -T show</b>
74.6.143.25
74.6.143.26
74.6.231.20
74.6.231.21
98.137.11.163
98.137.11.164
216.58.208.110
2001:4998:24:120d::1:0
2001:4998:24:120d::1:1
2001:4998:44:3507::8000
2001:4998:44:3507::8001
2001:4998:124:1507::f000
2001:4998:124:1507::f001
2a00:1450:400e:80e::200e
</pre>

最终，您可以将收集到的所有 IP 地址（在它们被清除之前）添加到物理文件中，因为 persist 选项也可以从文件中获取输入：

```
table <passlist> persist file "/etc/pf-passlist.txt"
```

注意：使用 add 选项添加的 IP 地址将不会添加到 pfctl 中。持久表可以存在于内存中或文件中，但 add 选项不能将数据写入磁盘，只能写入内存。从文件加载的持久表需要使用文本编辑器手动编辑。

### 加载规则中

一旦您完成設置您的規則集，您可以使用以下方式測試您的規則：

<pre><b># pfctl -nf /etc/pf.conf</b></pre>

如果一切都正常，您可以通過移除 -n 選項來加載規則集：

<pre><b># pfctl -f /etc/pf.conf</b></pre>

使用下面的方式查看翻譯結果：

<pre><b># pfctl -s rules</b></pre>

### 不要尝试阻止 DHCP

只是作为一个注记，在 FreeBSD 上你无法通过 PF 阻止对 dhcpd（port 67）的访问，因为在 FreeBSD 上 dhcpd 和 dhclient 默认都使用 bpf 来接收和发送数据包。这意味着数据包在 PF 进行任何过滤之前就已经被发送和接收了。

由于 bpf 提供了一种协议无关的方式来访问数据链路层的原始接口，所以网络上的所有数据包，即使是发送给其他主机的，也都可以通过 bpf 访问到。

查看 OpenBSD 邮件列表上允许使用 pf 的 dhcpd 的帖子，以获取有关此行为的相关评论。

### 记录和监控

这是我设置的外部 NIC 上尝试访问被阻止的 PF 日志示例输出。我已经清理了一些输出，并删除了一些特定数据，0.0.0.0 当然不是我的公共 IP 地址，但你已经知道了吧 ;)

```command
# tcpdump -n -e -ttt -r /var/log/pflog
23:11:12 rule 14/(match) block in on em0: 45.129.33.4.45980 > 0.0.0.0.3422: S 1501043655:1501043655(0) win 1024
23:11:12 rule 14/(match) block in on em0: 45.129.33.4.45980 > 0.0.0.0.3481: S 311078394:311078394(0) win 1024
23:11:31 rule 14/(match) block in on em0: 176.214.44.229.25197 > 0.0.0.0.23: S 2084440900:2084440900(0) win 33620
23:11:33 rule 14/(match) block in on em0: 45.129.33.4.45980 > 0.0.0.0.3431: S 2774981044:2774981044(0) win 1024
23:11:43 rule 14/(match) block in on em0: 81.68.114.52.17191 > 0.0.0.0.23: S 1346864438:1346864438(0) win 26375
23:12:08 rule 14/(match) block in on em0: 193.27.229.26.53865 > 0.0.0.0.443: S 1057596009:1057596009(0) win 1024
23:12:31 rule 14/(match) block in on em0: 45.129.33.4.45980 > 0.0.0.0.4186: S 1233742605:1233742605(0) win 1024
23:12:44 rule 14/(match) block in on em0: 74.120.14.70.65509 > 0.0.0.0.9125: S 1836577847:1836577847(0) win 1024 <mss 1460> [tos 0x20]
23:12:44 rule 14/(match) block in on em0: 45.129.33.4.45980 > 0.0.0.0.4128: S 2112968453:2112968453(0) win 1024
23:13:15 rule 14/(match) block in on em0: 45.129.33.4.45980 > 0.0.0.0.3669: S 3627248539:3627248539(0) win 1024
23:13:19 rule 14/(match) block in on em0: 45.129.33.4.45980 > 0.0.0.0.3654: S 3889665614:3889665614(0) win 1024
23:13:29 rule 14/(match) block in on em0: 45.129.33.129.42239 > 0.0.0.0.4997: S 2249816896:2249816896(0) win 1024
23:13:37 rule 14/(match) block in on em0: 45.129.33.4.45980 > 0.0.0.0.3612: S 3797528151:3797528151(0) win 1024
23:14:03 rule 14/(match) block in on em0: 190.207.89.17.64372 > 0.0.0.0.445: S 1097568353:1097568353(0) win 8192 <mss 1460,nop,wscale 2,nop,nop,sackOK> (DF)
23:14:15 rule 14/(match) block in on em0: 45.129.33.4.45980 > 0.0.0.0.4219: S 2834775769:2834775769(0) win 1024
23:14:39 rule 14/(match) block in on em0: 45.129.33.4.45980 > 0.0.0.0.3702: S 1855726637:1855726637(0) win 1024
23:14:39 rule 14/(match) block in on em0: 45.129.33.4.45980 > 0.0.0.0.4210: S 3052103070:3052103070(0) win 1024
```

你可以看到，这里相当繁忙，而且我没有在这个设置上运行面向互联网的任何东西。

您还可以实时监视 PF：

```
# tcpdump -n -e -ttt -i pflog0
```

## DNS

域名服务（DNS）用于将域名转换为 IP 地址或反之。例如，当您在网页浏览器的地址栏中输入 wikipedia.org 时，权威 DNS 服务器将域名"wikipedia.org"转换为 IPv4 地址，如 91.198.174.192，以及/或 IPv6 地址，如 2620:0:862:ed1a::1。

DNS 还用于存储关于特定域名所属的邮件服务器等信息，以及许多其他用途。

如果您正在运行类 UNIX 操作系统，可以启动终端并尝试使用 host 进行手动域名查询：

<pre><b>$ host wikipedia.org</b>
wikipedia.org has address 91.198.174.192
wikipedia.org has IPv6 address 2620:0:862:ed1a::1
wikipedia.org mail is handled by 10 mx1001.wikimedia.org.
wikipedia.org mail is handled by 50 mx2001.wikimedia.org.</pre>

注意：如果您尚未安装主机，请根据您所在的平台安装 bind 或 dnsutils 。您也可以使用 bind 提供的 dig 或 ldns 提供的 drill 等工具。

以下列出了与 DNS 相关的一些术语：

* `Forward DNS`

  * 将主机名和域名映射到 IP 地址。
* `Reverse DNS`

  * IP 地址到主机名和域名的映射。
* `Resolver`

  * 一种系统，通过它机器查询名称服务器以获取区域信息，即“DNS 服务器”的另一个名称。
* `Root zone`

  * 互联网区域层次结构的起始部分。所有区域都属于根区域，类似于文件系统中所有文件都属于根目录。

这是区域的一个示例

* . （一个句号）通常在文档中指 root 区域。
* org. 是根区域下的顶级域名（TLD）。
* wikipedia.org. 是 org. TLD 下的一个区域。
* 1.168.192.in-addr.arpa 是引用所有 IP 地址的区域，这些 IP 地址位于 192.168.1.* IP 地址空间之下。

当互联网上的计算机需要解析域名时，解析器从右向左将名称拆分为其标签。第一个组件，顶级域（TLD），通过根服务器查询以获取负责的权威服务器。针对每个标签的查询返回更具体的名称服务器，直到一个名称服务器返回原始查询的答案。

尽管任何本地 DNS 服务器都可以实现自己的私有根域名服务器，但术语“根域名服务器”通常用于描述实现互联网官方全球域名系统的根域名空间域的十三个著名根域名服务器。解析器使用一个小 3 KB 的 root.hints 文件，由 Internic 发布，用于引导这个根服务器地址的初始列表。对于包括 Unbound 在内的许多软件，此列表已内置到软件中。

在根区域数据库上，您可以查找顶级域的委派详细信息，包括诸如.com、.org 以及国家代码顶级域（TLD）如.uk 和.de 的 TLD。

注意：由于可以查找顶级域的委派详细信息，您可能期望可以更深入地实际查找特定域名服务器在其数据库中注册的每个域。例如，我们可以获取.dk TLD 的负责顶级域服务器列表，可能期望可以查询这些列出的任一名称服务器的完整授权服务器数据库，然后查询其中一个以获取其数据库中注册的所有域。但 DNS 并不是这样工作的。获取 DNS 服务器完整数据库映射只有两种方法：要么您必须可以访问相关区域文件，要么您需要通过递归 DNS 服务器检查 DNS 流量并基于收集的数据重建区域数据，直到获取所有内容，而这几乎是不可能的。

有两种 DNS 服务器配置类型：

* `Authoritative`

  * 授权名称服务器公布其控制下领域的 IP 地址。这些服务器被列为其各自领域授权链顶部的服务器，并且能够提供明确的答案。授权名称服务器可以是主名称服务器，也称为主服务器，即它们包含原始数据集，也可以是辅助或从属名称服务器，其中包含通常通过直接与主服务器同步获得的数据副本。授权名称服务器只提供通过原始来源配置的数据来自 DNS 查询的答案，例如，域管理员。每个 DNS 区域必须分配一组授权名称服务器。这组服务器存储在带有名称服务器（NS）记录的父域区域中。授权服务器通过在其响应中设置称为“权威答案”（AA）比特的协议标志来指示其供应明确答案的状态。您可以使用网络工具（如 dig 或 drill）查找域名，该工具将回复显示一个显示 DNS 服务器是否为权威的授权标志。
* `Recursive`

  * 递归服务器，有时称为“DNS 缓存”或“仅缓存名称服务器”，为应用程序提供 DNS 名称解析，通过将客户端应用程序的请求中继到权威名称服务器链，完全解析网络名称。它们还（通常）缓存结果以回答在一定过期时间内的潜在未来查询。大多数互联网用户访问由他们的 ISP 或公共 DNS 服务提供商提供的公共递归 DNS 服务器。理论上，仅有权威名称服务器就足以操作互联网。但是，仅有权威名称服务器运行时，每个 DNS 查询必须从域名系统的根区开始进行递归查询，并且每个用户系统都必须实施能够递归操作的解析器软件。为了提高效率，在互联网上减少 DNS 流量并提高终端用户应用程序的性能，域名系统支持递归解析器。递归 DNS 查询是指 DNS 服务器通过根据需要查询其他名称服务器来完全回答查询的查询。

名称服务器可以同时是权威和递归的，但建议不要合并配置类型。为了能够执行其工作，权威服务器应始终对所有客户端可用。另一方面，由于递归查找所需时间比权威响应长得多，因此递归服务器应仅对有限数量的客户端可用，否则它们容易受到分布式拒绝服务（DDoS）攻击。

注意：如有需要，我建议您阅读《Linux 网络管理员指南》第 6 章中的“DNS 工作原理”。我还建议您阅读维基百科上的域名服务（DNS）条目。

## 我向您介绍，Unbound

Unbound 是一个递归、缓存和验证的开源 DNS 解析器，具有以下特性：

* 在过期之前，缓存可选择预取流行项目。
* DNS over TLS (DoT) forwarding and server, with domain-validation.
* DNS over HTTPS (DoH).
* Query Name Minimization.
* DNSSEC-验证缓存的激进使用。
* 用于本地根区域副本的权威区域。
* DNS64。
* DNSCrypt.
* DNSSEC 验证。
* EDNS 客户子网。

Unbound 被设计为快速和安全，它融入了基于开放标准的现代特性。2019 年底，Unbound 经过严格审计。

提示: 使用 Unbound 而不是其他一些简单的仅缓存解析器（例如 dnsmasq）的主要原因之一是，如果您在 Unbound 的配置中不使用 forward 选项，Unbound 将直接查询根服务器，使用根提示文件中列出的它们注册的 IP 地址。这将使您摆脱 ISP DNS 服务器以及任何公共 DNS 服务器（如 Google 或 Cloudflare），避免它们记录、出售和操纵的数据。诸如 dnsmasq 之类的简单缓存服务器总是将查询转发到另一个服务器，而 Unbound 直接查询根服务器，并沿着域链逐级向下工作，直到从相关域的注册权威 DNS 服务器获取相关记录。这意味着确切知道您要找的内容的 DNS 服务器也是有权回答问题的服务器。

警告: 如果您的 ISP 劫持 DNS 流量，Unbound 不会对您有任何帮助。请参阅 DNS 劫持部分，了解如何确定您的 DNS 流量是否被劫持。

在我们的 Unbound 设置中，对于诸如“wikipedia.org”这样的域的查询将如下所示：

1. 您的浏览器向操作系统发送一个查询，问的是“wikipedia.org 的 IP 地址是多少”？
2. 操作系统，更具体地说是 C 库中的解析程序，提供对互联网域名系统的访问，然后将 DNS 请求转发到/etc/resolv.conf 中列出的域名服务器（在类 UNIX 操作系统上）。
3. Unbound 收到查询，并首先查找其缓存中的'wikipedia.org'，如果未找到，则 Unbound 查询其 Root Hints 文件中列出的根域'.org'的一个根服务器。
4. 根服务器回复有关'.org'顶级域的相关服务器的引荐。
5. 然后 Unbound 向一个相关服务器发送查询，请求'wikipedia.org'的权威 DNS 服务器。
6. 服务器将回复一个引用到为“wikipedia.org”注册的权威域名服务器。
7. Unbound 然后向其中一个权威域名服务器发送查询，并请求“wikipedia.org”的 IP 地址。
8. 权威域名服务器通过发送其“A”（IPv4）和/或“AAAA”（IPv6）记录中列出的 IP 地址来回复该域名“wikipedia.org”。
9. Unbound 接收来自权威域名服务器的 IP 地址，并将答案返回给客户端。
10. 如果启用，Unbound 将缓存该信息一段预定的时间，以备将来查询相同域名时使用。

您可以尝试自行进行 DNS trace ，以查看上述情况。在此示例中，我使用的是启用了 trace 选项的 drill 工具。

<pre><b># drill -T wikipedia.org</b>
.       518400  IN      NS      l.root-servers.net.
.       518400  IN      NS      k.root-servers.net.
.       518400  IN      NS      e.root-servers.net.
.       518400  IN      NS      a.root-servers.net.
.       518400  IN      NS      m.root-servers.net.
.       518400  IN      NS      h.root-servers.net.
.       518400  IN      NS      i.root-servers.net.
.       518400  IN      NS      f.root-servers.net.
.       518400  IN      NS      c.root-servers.net.
.       518400  IN      NS      b.root-servers.net.
.       518400  IN      NS      g.root-servers.net.
.       518400  IN      NS      d.root-servers.net.
.       518400  IN      NS      j.root-servers.net.
org.    172800  IN      NS      a0.org.afilias-nst.info.
org.    172800  IN      NS      a2.org.afilias-nst.info.
org.    172800  IN      NS      b0.org.afilias-nst.org.
org.    172800  IN      NS      b2.org.afilias-nst.org.
org.    172800  IN      NS      c0.org.afilias-nst.info.
org.    172800  IN      NS      d0.org.afilias-nst.org.
wikipedia.org.  86400   IN      NS      ns0.wikimedia.org.
wikipedia.org.  86400   IN      NS      ns1.wikimedia.org.
wikipedia.org.  86400   IN      NS      ns2.wikimedia.org.
wikipedia.org.  600     IN      A       91.198.174.192</pre>

注意：Unbound 能够验证接收到的响应是否正确。通常使用域名系统安全扩展（DNSSEC）或者使用 0x20 编码的随机比特来防止欺骗尝试。除了 0x20 编码的随机比特之外，在 OpenBSD 上默认启用 Unbound 上的所有其他验证设置，例如硬化黏合和强化的 dnssec 剥离数据。

## 使用 DNS 拦截

DNS 拦截，也称为过滤或 DNS 欺骗，是一种过程，在这种过程中，您向进行查询的客户端提供一个“虚假”回复。我们通过回复 NXDOMAIN（表示不存在的域）或重定向到与域所有者意图的 IP 地址不同的另一个 IP 地址来阻止对有效 IP 地址的请求。

这使我们能够创建一个或多个域名阻止列表，而不是为某个域名提供正确的 IP 地址，我们会返回该域名"不存在"的消息，从而阻止应用程序进一步与预期目标的通信。

通常所有 DNS 请求都会使用 UDP 或 TCP 协议发送到port 53，通过设置一个 DNS 服务器（这就是我们使用 Unbound 所做的），并确保所有发送到port 53 的流量都到达我们的 DNS 服务器或者被阻止，我们可以确保所有 DNS 回复都源自我们内部运行在 OpenBSD 路由器上的 Unbound 服务器。

注意：您不能完全信任 DNS 阻止，因为 DNS 阻止是可以被规避的。即使我们已经有了一个坚实的方法，某些人仍然可以使用 VPN 服务来规避这种设置。我们并不试图建立一个 100%防错系统 - 尽管我们稍后在指南中会进一步讨论这个问题 - 我们只是在尝试以更好的方式保护我们的家人。我们还需要考虑到互联网的其他访问点，例如手机、朋友的手机和房屋、公共互联网接入等等。

### NXDOMAIN 与重定向

当我们想要使用 DNS 阻止一个域名时，我们可以选择几种方法，但最流行的两种方法之一是将 DNS 查询重定向到本地 IP 地址，例如 127.0.0.1 或 0.0.0.0，或者返回一个"不存在的互联网域名定义"（NXDOMAIN）。 NXDOMAIN 是对“不存在的互联网或内部域名”的标准回复。如果无法使用 DNS 解析域名，则会发生 NXDOMAIN 条件。

我们可以尝试使用 host 命令解析不存在的域名：

<pre><b>$ host a1b7c3n9m3b0.com</b>
Host a1b7c3n9m3b0.com not found: 3(NXDOMAIN)</pre>

自从域名"a1b7c3n9m3b0.com"没有被任何人注册（至少在我写这个的时候），我们会收到"NXDOMAIN"响应。

我们还可以使用 drill 。从 drill 的输出中，"HEADER"部分的 rcode 字段是相关信息：

<pre><b>$ drill a1b7c3n9m3b0.com</b>
;; ->>HEADER<<- opcode: QUERY, <b>rcode: NXDOMAIN</b>, id: 39710
…</pre>

或者如果您更喜欢 dig ，那么相关信息位于"HEADER"部分的 status 字段中：

<pre><b>$ dig a1b7c3n9m3b0.com</b>
; <<>> DiG 9.16.8 <<>> +search a1b7c3n9m3b0.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, <b>status: NXDOMAIN</b>, id: 48858
…</pre>

使用 NXDOMAIN 回复不仅是根据 RFC 8020 的正确方式来阻止域名，而且也是最佳方式，因为将重定向到像 127.0.0.1 或 0.0.0.0 这样的 IP 地址只会使发起 DNS 查询的客户端与自身通信。

可能是因为浏览器会回复类似 Firefox can't establish a connection to the server at 0.0.0.0. 的内容。然而，因为 IP 地址 0.0.0.0 简单地转换为我们本地机器，我们仍然可以 ping 该地址，因为它与 ping 127.0.0.1 是同义的：

<pre><b>$ ping 0.0.0.0</b>
PING 0.0.0.0 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.019 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.049 ms</pre>

因此，我建议您始终使用 NXDOMAIN 回复，这也是我们在本教程中将要使用的方式。

提示：Unbound 可以处理大量需要返回 NXDOMAIN 响应的域名列表，但无法很好地处理需要重定向的大量域名列表。如果出于某种原因你坚持要重定向而不是使用 NXDOMAIN，我建议你使用 --addn-hosts=<file> 选项设置 dnsmasq，然后使 dnsmasq 监听port号端口，并让 dnsmasq 重定向所有被阻止的域名，然后将正常的 DNS 查询转发给 Unbound。然后设置 Unbound 监听一个非标准的端口，比如port 5353。与 Unbound 相反，dnsmasq 可以很好地处理大量重定向，但无法很好地处理大量的 NXDOMAIN 域名，它会变得非常慢。

## DNS over HTTPS（DoH）存在的问题

随着 DNS over HTTPS（DoH）的引入，DNS 阻止变得更加困难，虽然我确实尊重从隐私角度推广 DoH 的原始想法，但从安全角度看，DoH 是一个糟糕的构想，并且是错误的方法。

由于已经增加的能够提供 DoH 的公共 DNS 服务器数量，任何应用程序现在都可以使用 DoH，并完全绕过私人和企业级别的 DNS 阻断。不仅如此，DoH 还大大打开了应用开发人员设置他们自己的 DoH 服务器并让他们的应用程序使用这些服务器而不是内部网络连接的常规 DNS 服务器的大门。这在涉及专有软件时尤为棘手，因为您不仅看不到源代码，还无法更改任何 DoH 设置。

由于 DoH，我们不能简单地阻止像广告和色情这样的域名，我们还必须通过防火墙开始阻止公共 DoH 服务器。然而，尽管保留一个不断增长的公共 DoH 服务器 IP 地址列表已经够棘手了，但保留未知公共 DoH 服务器的列表，这些服务器可能被像 IoT 设备固件这样的专有软件利用，这几乎是不可能的。

对企业来说，DoH 也是一个彻底的噩梦，因为它基本上使覆盖的 DNS 设置得以实现。这使得提供带有广告和色情阻断的过滤解决方案（如本指南中所述）变得不可能，同时也使得系统管理员无法跨操作系统监视 DNS 设置以防止 DNS 劫持攻击。拥有多个具有其独特 DoH 设置的应用程序是一场噩梦。

DoH 也完全搞乱了用于安全目的的 DNS 流量的网络分析和监视。2019 年，Godlua，一个 Linux DDoS 机器人，是第一个利用 DoH 来隐藏其 DNS 流量的恶意应用程序。

而且，也许最重要的是，DoH 不能防止用户跟踪。某些部分的 HTTPS 连接未加密，如 SNI 字段（尽管它慢慢在改进），OCSP 连接，当然还有目标 IP 地址，这在我看来是需隐藏的通信中最关键的部分！

那些真正需要隐私的人，例如在隐私受损政策国家的记者，不能信任 DoH！目标服务器的 IP 地址无法通过 DoH 隐藏，即使流量本身的所有内容都加密了。如果有人真正需要加密通信，那个人需要一种完全不同的策略，而不是 DoH。

这让我想知道，世界上是谁认为 DoH 是一个好主意的？难道他们不理解使用 HTTPS 进行通信的基础知识吗？或者这个议程可能是由一些私人 DNS 服务公司，比如 Google 和 Cloudflare 推动的，他们通过进一步收集用户数据获利？

一些公共 DNS 服务提供商表示，从隐私的角度来看，DoH 比替代方案（如 DNS over TLS (DoT)）更好，因为 DNS 查询被隐藏在更大的 HTTPS 流量中。这减少了网络管理员的可见性，但为用户提供了更多隐私。

那条消息有问题。虽然初始域名查找的确隐藏在 HTTPS 流量中，但由 DoH 服务器提供的目标 IP 地址并没有。当客户端应用访问目标 IP 地址时，源 IP 地址和目标 IP 地址都会在 ISP 级别（以及可能多个其他级别）被记录。

当无法立即准确确定用户试图访问的目标网络服务器的域名时，特别是如果网络服务器在相同的 IP 地址下运行多个域名，这绝对不是不可能甚至不难的。

注意：在附录中，您可以找到一个名为检查 DNS over HTTPS（DoH）的部分，我们将在其中演示目标 IP 地址在 DoH 通信中如何被揭示。您还可以找到一个名为阻止 DNS over HTTPS（DoH）的部分，在其中我们使用 PF 防火墙来阻止已知的公共 DoH 服务器。

## 设置 Unbound

### 基本 text: 基本设置 text: 基本设置 : 基本设置

Setting up Unbound is very easy as Unbound not only comes with great defaults, but it is also very well documented. Before we begin I advice that you take a look at the OpenBSD man page for [unbound](https://man.openbsd.org/unbound), [unbound-checkconf](https://man.openbsd.org/unbound-checkconf) and [unbound.conf](https://man.openbsd.org/unbound.conf).

Because Unbound is [chrooted](https://en.wikipedia.org/wiki/chroot) on OpenBSD, the configuration file `unbound.conf` doesn't reside in `/etc`, as it otherwise normally would, instead it resides in `/var/unbound/etc/`.

复制现有的 Unbound 配置文件：

<pre><b># mv /var/unbound/etc/unbound.conf /var/unbound/etc/unbound.conf.backup</b></pre>

然后使用你喜欢的文本编辑器创建一个新的 /var/unbound/etc/unbound.conf 文件，并填入以下内容：

```
server:

    # Logging (default is no).
    # Uncomment this section if you want to enable logging.
    # Note enabling logging makes the server (significantly) slower.
    # verbosity: 2
    # log-queries: yes
    # log-replies: yes
    # log-tag-queryreply: yes
    # log-local-actions: yes

    interface: 127.0.0.1
    interface: 192.168.1.1
    interface: 192.168.2.1
    interface: 192.168.3.1

    # In case you need Unbound to listen on an alternative port, this is the
    # syntax:
    # interface: 127.0.0.1@5353

    # Control who has access.
    access-control: 0.0.0.0/0 refuse
    access-control: ::0/0 refuse
    access-control: 127.0.0.0/8 allow
    access-control: ::1 allow
    access-control: 192.168.1.0/24 allow
    access-control: 192.168.2.0/24 allow
    access-control: 192.168.3.0/24 allow

    # "id.server" and "hostname.bind" queries are refused.
    hide-identity: yes

    # "version.server" and "version.bind" queries are refused.
    hide-version: yes

    # Cache elements are prefetched before they expire to keep the cache up to date.
    prefetch: yes

    # Our LAN segments.
    private-address: 192.168.0.0/16

    # We want DNSSEC validation.
    auto-trust-anchor-file: "/var/unbound/db/root.key"

# Enable the usage of the unbound-control command.
remote-control:
    control-enable: yes
    control-interface: /var/run/unbound.sock
```

我已经对上面的选项进行了注释，如果你需要进一步解释配置，请查看 unbound.conf 的手册页中的每个设置。

默认情况下，日志记录到 syslog。如果您想更改这一点，您可以在 Unbounds chroot 中创建一个日志文件，然后让 Unbound 记录到该文件：

```
# mkdir /var/unbound/log
# touch /var/unbound/log/unbound.log
# chown -R root._unbound /var/unbound/log
# chmod -R 774 /var/unbound/log
```

然后在 unbound.conf 文件中，在日志部分添加以下选项：

```
logfile: "/log/unbound.log"
use-syslog: no
log-time-ascii: yes
```

注意：我们没有使用完整路径到日志文件，因为 Unbound 被限制。使用上面的 logfile 选项，日志文件最终会出现在 /var/unbound/log/unbound.log. 中。

 然后重新启动 Unbound：

```
# rcctl restart unbound
```

在上述设置中，我已允许 Unbound 监听环回接口 (127.0.0.1)，以便本地网络应用需要时可以进行查找。在我们的 OpenBSD 路由器上，我已将我们的 Unbound DNS 服务器列为 /etc/resolv.conf ，因为我不希望路由器上的任何内容查询 ISP DNS 服务器：

```
nameserver 127.0.0.1
```

如果您正在使用 DHCP 在外部 NIC 上获取 IP 地址（连接到您的 ISP 调制解调器或路由器的接口），您需要确保 dhcpleased 不会更改 /etc/resolv.conf 。编辑 /etc/dhcpleased.conf 并添加：

```
interface em0 { ignore dns }
```

这将确保我们只列出本地 DNS 服务器。

 启用 Unbound：

<pre><b># rcctl enable unbound</b></pre>

每当您更改 Unbound 配置时，可以选择仅重新启动 Unbound：

<pre><b># rcctl restart unbound</b></pre>

或者简单地刷新配置选项（这也会刷新缓存）：

<pre><b># unbound-control reload</b></pre>

您可以通过运行以下命令列出 Unbound 启动时使用的设置（对于在 OpenBSD 上运行的任何服务都适用）：

<pre><b># rcctl get unbound</b></pre>

如果您想获取一些统计数据，可以运行：

<pre><b># unbound-control stats_noreset</b>
thread0.num.queries=2056
thread0.num.queries_ip_ratelimited=0
thread0.num.cachehits=678
thread0.num.cachemiss=1378
thread0.num.prefetch=15
thread0.num.expired=0
…</pre>

你也可以获取缓存的转储：

<pre><b># unbound-control dump_cache|less</b></pre>

如果你想查看 Unbound 为特定域名查询的域名服务器，你可以这样做：

<pre><b># unbound-control lookup wikipedia.org</b></pre>

如果你想为特定域名刷新缓存，你可以这样做：

<pre><b># unbound-control flush example.com</b></pre>

请查看 unbound-control 的手册页，以获取更多选项和命令。

### 覆盖荒谬低的 TTL 设置

一个让人非常困扰的问题是人们为他们的域设置荒谬低的 TTL 值。由于某种原因，将默认值设置为 60 秒似乎成了一种潮流。

低 TTL 的问题在于它使 DNS 缓存变得完全无效。只要 TTL 未过期，查询将仅使用缓存的回复。尽管 RFC 规定 TTL 必须得到尊重，但使用如此低的数值 DNS 变得极其低效。因此，我建议您通过将自己的默认设置为一小时来覆盖 TTL 设置。另一个提高 DNS 请求速度的方法是在更新记录之前通过提供过时记录来减少延迟，而不是反之。

```
cache-min-ttl: 3600
serve-expired: yes
```

增加 TTL 的一个理论问题是，一个域可能会获得新的 IP 地址，然后无法解析，因为缓存中有旧条目。但是，在实践中，遇到过时域的风险微乎其微，将默认最低 TTL 设置为一小时非常值得，因为可以改善缓存的使用。

### 让我们屏蔽一些域名!

现在我们来讨论关于域名阻断的有趣部分。

我创建了一个简单的脚本叫做 DNSBlockBuster，它会自动从各种在线来源下载一组主机文件，将它们连接成一个文件，进行清理，然后将结果转换为适用于 Unbound 和 dnsmasq 的域名阻断列表。它主要用于阻止广告、色情网站和追踪。

使用 DNSBlockBuster，您可以选择创建一个白名单，以防主机文件中列出的任何域名对您来说是误报，并且您可以添加自己的阻断列表，以手动阻止一些未在主机文件中列出的域名。您还可以轻松添加新的阻断列表或移除任何提供的阻断列表。

您当然可以不使用我的脚本，但在本教程中我将使用这个脚本。

当前脚本创建了一个包含近两百万个域名的大型域名列表，当加载整个阻止列表时，Unbound 总共占用约 705MB 内存。

为了防止 Unbound 在加载如此庞大的列表时超时，请编辑 /etc/rc.conf.local 并添加以下内容：

```
unbound_timeout=240
```

 然后重新启动 Unbound：

<pre><b># rcctl restart unbound</b></pre>

查看 DNSBlockBuster 文档中的使用部分，了解如何使用它。这很简单易行。

创建好 Unbound 的阻止列表后，请将其放在 /var/unbound/etc/ 中，然后编辑 Unbound 配置文件 /var/unbound/etc/unbound.conf ，在配置文件的 server 部分（ remote-control 部分之前）的某处插入以下内容：

```
include: "/var/unbound/etc/unbound-blocked-hosts.conf"

# Enable the usage of the unbound-control command.
remote-control:
    control-enable: yes
    control-interface: /var/run/unbound.sock
```

使用以下命令重新加载 Unbound：

<pre><b># unbound-control reload</b></pre>

如果您在另一个终端中运行 top 命令，您会注意到 Unbound 在初始化加载列表时会占用相当多的 CPU。还请注意内存使用情况。

您现在可以通过查询列表中的一个被阻止的域来测试我们的 DNS 阻断功能：

<pre><b>$ drill 3lift.com</b>
;; ->>HEADER<<- opcode: QUERY, <b>rcode: NXDOMAIN</b>, id: 55906
…</pre>

然后尝试使用 Cloudflare 的 DNS 服务器：

<pre><b>$ drill 3lift.com @1.1.1.1</b>
;; ->>HEADER<<- opcode: QUERY, <b>rcode: NOERROR</b>, id: 48771
…</pre>

从查询结果可以看出，我们的 DNS 服务器通过返回 NXDOMAIN 阻止了对 3lift.com 域名的访问，而 Cloudflare 的 DNS 服务器则返回了正确的 IP 地址。

## DNS 安全

DNS 安全是一个广泛的主题。在这一部分中，我们将处理一些主要关注我们自己运行 DNS 服务器的主题。

DNS 协议是未加密的，并且默认情况下不考虑任何机密性、完整性或身份验证。如果您使用不受信任的网络或恶意的 ISP，您的 DNS 查询可以被窃听并且响应可以被篡改。此外，ISP 可以进行 DNS 劫持。

### DNS 劫持

DNS 劫持意味着您执行的 DNS 查询被重定向到另一个 DNS 服务器。这通常是通过将来自一个目的地的所有流量重定向到另一个目的地来完成的。

确定您的 ISP 是否劫持了您的 DNS 流量的最简单方法之一是直接查询权威 DNS 服务器。

我们可以使用多种工具来实现这一点。在这个例子中，我们首先将使用 drill 。选项，在这个例子中是相同的 dig 。我们会再次使用域名“wikipedia.org”。

首先，我们需要获取权威服务器。它们将显示在“ANSWER SECTION”中：

<pre><b>$ drill NS wikipedia.org</b>
;; ->>HEADER<<- opcode: QUERY, rcode: NOERROR, id: 28789
;; flags: qr rd ra ; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 0
;; QUESTION SECTION:
;; wikipedia.org.       IN      NS

;; ANSWER SECTION:
<b>wikipedia.org.  85948   IN      NS      ns2.wikimedia.org.
wikipedia.org.  85948   IN      NS      ns0.wikimedia.org.
wikipedia.org.  85948   IN      NS      ns1.wikimedia.org.</b>

;; AUTHORITY SECTION:

;; ADDITIONAL SECTION:

;; Query time: 1 msec
;; SERVER: 127.0.0.1
;; WHEN: Thu Nov  5 07:53:19 2020
;; MSG SIZE  rcvd: 95</pre>

然后，我们需要直接查询其中一个权威服务器。需要注意的重要字段是“HEADER”字段中的标志。为了回答是权威的，标志 aa 必须被列出。

<pre><b>$ drill @ns1.wikimedia.org wikipedia.org</b>
;; ->>HEADER<<- opcode: QUERY, rcode: NOERROR, id: 57611
;; flags: qr <b>aa</b> rd ; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
;; QUESTION SECTION:
;; wikipedia.org.       IN      A

;; ANSWER SECTION:
wikipedia.org.  600     IN      A       91.198.174.192

;; AUTHORITY SECTION:

;; ADDITIONAL SECTION:

;; Query time: 127 msec
;; SERVER: 208.80.153.231
;; WHEN: Thu Nov  5 07:56:10 2020
;; MSG SIZE  rcvd: 47</pre>

这表明我们收到的回复没有被劫持，因为回复是权威的。让我们尝试给 Cloudflare 公共 DNS 服务器相同的查询：

<pre><b>$ drill @1.1.1.1 wikipedia.org</b>
;; ->>HEADER<<- opcode: QUERY, rcode: NOERROR, id: 40562
;; flags: qr rd ra ; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
;; QUESTION SECTION:
;; wikipedia.org.       IN      A

;; ANSWER SECTION:
wikipedia.org.  555     IN      A       91.198.174.192

;; AUTHORITY SECTION:

;; ADDITIONAL SECTION:

;; Query time: 3 msec
;; SERVER: 1.1.1.1
;; WHEN: Thu Nov  5 08:02:58 2020
;; MSG SIZE  rcvd: 47</pre>

请注意，“HEADER”字段中缺少 aa 标志。这意味着回复不是权威性的。

另一个更简单的工具是 nslookup。首先查询权威名称服务器：

<pre><b>nslookup -querytype=NS wikipedia.org</b>
Server:         127.0.0.1
Address:        127.0.0.1#53

Non-authoritative answer:
wikipedia.org   nameserver = ns1.wikimedia.org.
wikipedia.org   nameserver = ns2.wikimedia.org.
wikipedia.org   nameserver = ns0.wikimedia.org.</pre>

然后尝试查询我们自己的 DNS 服务器的域名：

<pre><b>$ nslookup wikipedia.org</b>
Server:         127.0.0.1
Address:        127.0.0.1#53

<b>Non-authoritative answer:</b>
Name:   wikipedia.org
Address: 91.198.174.192

Server:         ns2.wikimedia.org
Address:        91.198.174.239#53

Name:   wikipedia.org
Address: 91.198.174.192</pre>

Non-authoritative 明确显示，该回复并非来自权威 DNS 服务器。没问题，我们查询了自己的 DNS 服务器。让我们尝试直接查询其中一个权威服务器：

<pre><b>$ nslookup wikipedia.org ns0.wikimedia.org</b>
Server:         ns0.wikimedia.org
Address:        208.80.154.238#53

Name:   wikipedia.org
Address: 91.198.174.192</pre>

Non-authoritative 已经消失，我们收到的回复是权威的，这意味着我们的 DNS 查询没有被劫持。

我现在已经启用了一个我知道会拦截 DNS 查询以保护客户免受 DNS 泄漏的 VPN 服务，现在我将再次查询其中一个权威服务器：

<pre><b>$ nslookup wikipedia.org ns0.wikimedia.org</b>
Server:         ns0.wikimedia.org
Address:        208.80.154.238#53

<b>Non-authoritative answer:</b>
Name:   wikipedia.org
Address: 91.198.174.192
Name:   wikipedia.org
Address: 2620:0:862:ed1a::1</pre>

预期的结果是，即使我直接查询权威服务器，答案也不具权威性。DNS 流量被劫持，回复被重定向到另一个未知的 DNS 服务器。

DNS 劫持，无论是由 ISP 还是其他人执行，都是非常棘手的问题。首先，我们无法完全信任从 DNS 服务器得到的答案。其次，即使 DNS 回复确实传递了未经篡改的数据，DNS 流量也已被某种未知原因劫持，这可能是数据收集和日志记录，或者完全不同的原因。

注：一些 ISP，如 Optimum Online、Comcast、Time Warner、Cox Communications、RCN、Rogers、Charter Communications、Verizon、Virgin Media、Frontier Communications、Bell Sympatico、Airtel、OpenDNS 等，开始在不存在的域名（NXDOMAIN）上实施 DNS 劫持，以此通过显示广告来赚钱。DNS 服务器将请求重定向到一个包含广告的伪 IP 地址，该 IP 地址包含一个网站。我不知道还有多少 ISP 和公共 DNS 服务提供商在这样做。

#### DNS 劫持预防

如果您发现您的 DNS 流量在port 53 被劫持，您基本上有三个选项可以保护自己：

1. 如果您有选择，那就换您的 ISP！您的 ISP 不应该劫持您的 DNS 流量。就这样。
2. 在不会劫持或屏蔽port 53 端口的托管中心上设置自己的远程 DNS 服务器。然后让你的远程 DNS 服务器在一个非标准的port端口监听 DNS 连接，并转发所有的 DNS 查询到你的远程 DNS 服务器。
3. 使用一个可信赖的 VPN，它不会劫持 DNS 流量，或者如果它确实这样做了，确保你可以信任他们的非记录政策。

### DNS 欺骗

DNS 欺骗，又称 DNS 缓存投毒，与 DNS 劫持不同。在 DNS 劫持攻击中，流量被重定向到另一个目的地，而在 DNS 欺骗攻击中，数据本身被篡改。通常这两种攻击策略会结合使用。

在 DNS 欺骗攻击中，篡改的数据被引入 DNS 解析器的缓存中，导致名称服务器返回错误的结果，例如错误的 IP 地址。

#### DNS 欺骗预防

這種類型的攻擊可以在傳輸層或應用層上通過建立連接後進行端到端驗證來進行緩解。這方面的一個常見例子是使用傳輸層安全性（TLS）和數字簽名。

安全 DNS（DNSSEC）使用與可信公鑰證書簽署的加密數字簽名來確定數據的真實性。 DNSSEC 可以防篡改，但許多 DNS 管理員仍未實施。

截至 2020 年，所有原始頂級域都支持 DNSSEC，大多數大型國家的國家代碼頂級域也支持 DNSSEC，但許多國家代碼頂級域仍未支持。

## 附录

### 检查 DNS over HTTPS（DoH）

我想说明的事实是，DoH 实际上并未提供真正的隐私保护，因为源 IP 地址和目标 IP 地址在 HTTPS 通信中都可以清晰地看到。

首先，我确保在成年人的局域网中的一台计算机上已禁用了 DoH，我正在使用 tcpdump 监视 em1 网卡上的流量。我还已经在 Unbound 上启用了日志记录，只是为了避免在 syslog 中积聚太多的 DNS 噪声，并且我正在使用 tail 来监视日志。

我将在浏览器中转到" wikipedia.org"，然后查看路由器上的监视显示出什么。

<pre><b># tcpdump -n -i em1 src host 192.168.1.5 and not arp</b>
tcpdump: listening on em1, link-type EN10MB
23:30:33.494352 192.168.1.5.55724 > 192.168.1.1.53: 58136+ A? wikipedia.org.(31) (DF)
23:30:33.774439 192.168.1.5.58372 > 192.168.1.1.53: 58448+ A? www.wikipedia.org.(35) (DF)
23:30:34.184287 192.168.1.5.46639 > 192.168.1.1.53: 15167+ A? www.wikipedia.org.(35) (DF)
…</pre>

<pre><b># tail -f /var/unbound/log/unbound.log</b>
Nov 05 23:30:33 unbound[12636:0] query: 192.168.1.5 wikipedia.org. A IN
Nov 05 23:30:33 unbound[12636:0] reply: 192.168.1.5 wikipedia.org. A IN NOERROR 0.097209 0 47
Nov 05 23:30:33 unbound[12636:0] query: 192.168.1.5 www.wikipedia.org. A IN
Nov 05 23:30:33 unbound[12636:0] reply: 192.168.1.5 www.wikipedia.org. A IN NOERROR 0.154989 0 80
Nov 05 23:30:34 unbound[12636:0] query: 192.168.1.5 www.wikipedia.org. A IN
Nov 05 23:30:34 unbound[12636:0] reply: 192.168.1.5 www.wikipedia.org. A IN NOERROR 0.000000 1 80
…</pre>

当然，我们既在界面流量上看到了查询，也在 Unbound 日志中看到了。

我已经在 Firefox 中启用了 DoH 并禁用了常规 DNS，通过将 network.trr.mode 的值设置为 4 。然后，我更改了 Network settings 并将 Cloudflare 设置为 DoH 提供商。

技巧：如果您仅通过首选项面板启用 Firefox 中的 DoH，Firefox 仍会将常规 DNS 作为备用。为了强制 Firefox 仅使用 DoH，您可以将 network.trr.mode 的值设置为。

在 URL 栏中输入 about:config 并按 Enter 键访问 Firefox 的隐藏配置面板。

步骤 2：查找设置 network.trr.mode 。这控制 DoH 支持。该设置支持四个值：

1 - DoH 被禁用。2 - DoH 已启用，但 Firefox 根据哪个返回更快的查询响应同时使用 DoH 和常规 DNS。3 - DoH 已启用，并且常规 DNS 作为备份。4 - DoH 已启用，并且常规 DNS 被禁用。5 - DoH 被禁用。

步骤 3：查找设置 network.trr.bootstrapAddress 。这控制你的 DoH 服务器的数字 IP 地址。将 1.1.1.1 的值输入到该字段并按 Enter 键。

这次我将访问 "freebsd.org"。

<pre><b># tcpdump -n -i em1 src 192.168.1.5 and not arp</b>
tcpdump: listening on em1, link-type EN10MB
00:21:10.944243 192.168.1.5.32856 > 1.1.1.1.443: P 2223446146:2223446202(56) ack 157857007 win 501 (DF)
00:21:10.948719 192.168.1.5.46584 > 96.47.72.84.80: S 922508523:922508523(0) win 64240 <mss 1460,sackok,timestamp="" 1673624773="" 0,nop,wscale="" 7=""> (DF)
00:21:11.133801 192.168.1.5.33298 > 96.47.72.84.443: S 3275123911:3275123911(0) win 64240 <mss 1460,sackok,timestamp="" 1673624958="" 0,nop,wscale="" 7=""> (DF)
…</mss></mss></pre>

<pre><b># tail -f /var/unbound/log/unbound.log</b>
Nov 05 23:30:33 unbound[12636:0] query: 192.168.1.5 wikipedia.org. A IN
Nov 05 23:30:33 unbound[12636:0] reply: 192.168.1.5 wikipedia.org. A IN NOERROR 0.097209 0 47
Nov 05 23:30:33 unbound[12636:0] query: 192.168.1.5 www.wikipedia.org. A IN
Nov 05 23:30:33 unbound[12636:0] reply: 192.168.1.5 www.wikipedia.org. A IN NOERROR 0.154989 0 80
Nov 05 23:30:34 unbound[12636:0] query: 192.168.1.5 www.wikipedia.org. A IN
Nov 05 23:30:34 unbound[12636:0] reply: 192.168.1.5 www.wikipedia.org. A IN NOERROR 0.000000 1 80
…</pre>

从网络接口监视中可以看出，连接已经建立到 Cloudflare 的 DNS 服务器 1.1.1.1 的 443 端口（HTTPS），然后我们立即访问了 IP 目的地址 96.47.72.84。同时，在 Unbound 日志中什么也没有发生， tail 仍然只显示之前的查询。

如果在路由器上进行常规的 DNS 查询，我们可以验证 IP 地址 96.47.72.84 确实是 "freebsd.org" 的 IP 地址。

此外，在这个特定的例子中，我们甚至可以通过将目标 IP 地址 96.47.72.84 输入到浏览器的地址栏中，直接进入 "freebsd.org" 的网站。

这表明，即使 DoH 绕过了常规的 DNS 查询，它仍无法隐藏目标 IP 地址，该地址仍然以明文形式出现在通信流量中。

### 阻止 DNS over HTTPS（DoH）

先前，DNSBlockBuster 脚本中已经包含了一些 DoH 域名，我随机地加入了这些域名，但我已经从 DNS 服务器中移除了 DoH 阻止，因为这确实需要在防火墙层面进行。

在我看来，通过域名阻止 DoH 并没有太多意义，因为首先必须查找域名。大多数使用 DoH 的客户端直接将 DoH 服务器的主机 IP 地址编码到源代码中。

如果你不使用 IPv6，可以阻止所有传出的 IPv6 流量，然后只使用 IPv4 列表。在 /etc/pf.conf 的“默认保护和阻止”部分中更改 pass out 参数为 pass out inet 。这样你只允许传出的 IPv4 流量，不需要专门阻止 IPv6 的 DoH IP 地址。

从 awesome-lists 下载 IP 列表，并编辑列表以满足您的需求，并将其放在磁盘的某个位置。

我已创建一个子目录 /etc/pf-block-lists ，我在那里放置所有我需要的 PF IP 地址段列表。

然后在 /etc/pf.conf 的"Tables"部分为 PF 创建一个持久文件：

```
# Public DoH servers.
table <block_doh> persist file "/etc/pf-block-lists/doh-ipv4.txt"
```

如果您需要 IPv6，也可以添加：

```
table <block_doh> persist file "/etc/pf-block-lists/doh-ipv4.txt" file "/etc/pf-block-lists/doh-ipv6.txt"
```

然后将 block 添加到防火墙的“默认情况下保护和阻止”部分：

```
# Let's block DoH.
block in quick on { $g_lan $c_lan $dmz } to <block_doh>
```

 使用以下命令重新加载：

<pre><b># pfctl -f /etc/pf.conf</b></pre>

检查列表：

<pre><b># pfctl -vvt block_doh -T show</b></pre>

如果过一段时间后，你想查看实际使用的 IP 地址，可以过滤输出：

<pre><b># pfctl -vvt block_doh -T show | awk '/[/ {p+=$4; b+=$6} END {print p, b}'</b></pre>

### 将域名选项添加到 DHCP 并使用 FQDN

如果我们设置我们的网络，使所有计算机和设备具有固定的 IP 地址和主机名，许多工具将无法使用这些主机名直接运行，除非将域名添加到 DNS 服务器。这是因为像 host 这样的网络工具期望查找要求是完全合格域名（FQDN）上的主机名。

假设我在我的 LAN 上设置了一台计算机，主机名为“foo”，固定 IP 地址为 192.168.1.7。我可能不记得“foo”是带有该地址的计算机，或者我可能不记得哪个主机与 IP 地址 192.168.1.7 相关联。

有了 FQDN，我们可以进行这样的查找：

<pre><b>$ host foo.example.com</b>
foo.example.com has address 192.168.1.7</pre>

并且我们可以做：

<pre><b># host 192.168.1.7</b>
7.1.168.192.in-addr.arpa domain name pointer foo.example.com</pre>

然而，每次都输入完整域名都很烦人。如果我们为 /etc/resolv.conf 添加了 domain-name 选项，域名将被自动附加。我们现在只需这样做：

<pre><b>$ host foo</b>
foo.example.com has address 192.168.1.7</pre>

一些人建议您注册一个域名，然后在 LAN 上内部使用该域名，虽然这确实有效，但完全不必要。根据 RFC 8375，您应该使用 .home.arpa 域，因为这是用于小型网络内部（如家庭网络）的。

让我们首先对 /etc/dhcpd.conf 配置进行一些更改。为了简单起见，我将仅使用公共 LAN 示例中的 Web 服务器，但您可以扩展到任何您喜欢的段，并且如果需要，还可以跨段使用此配置。

在我们当前的设置中，我们已经将域名 example.com 附加到 Web 服务器，所以我们可以直接使用它。但是，如果您没有需要真实域名的公共服务器，只需将其更改为 home.arpa 。我已将我们的 Web 服务器名称更改为“lilo”（是的，来自《双头家伙》中的 Lilo & Stitch，因为它比 Luke 或 Yoda 更酷！）。

```
subnet 192.168.1.0 netmask 255.255.255.0 {
    option domain-name-servers 192.168.1.1;
    option domain-name "example.com";
    option routers 192.168.1.1;
    range 192.168.1.10 192.168.1.254;
}
subnet 192.168.2.0 netmask 255.255.255.0 {
    option domain-name-servers 192.168.2.1;
    option domain-name "example.com";
    option routers 192.168.2.1;
    range 192.168.2.10 192.168.2.254;
}
subnet 192.168.3.0 netmask 255.255.255.0 {
    option domain-name-servers 192.168.3.1;
    option domain-name "example.com";
    option routers 192.168.3.1;
    range 192.168.3.10 192.168.3.254;
    host lilo.example.com {
        fixed-address 191.168.3.2;
        hardware ethernet 61:20:42:39:61:AF;
        option host-name "lilo";
    }
}
```

如果您喜欢使用多个域名而不是只使用一个，比如说为了您的专业 Web 开发使用 example.com ，然后为您的私人 LAN 使用 home.arpa ，您可以在 /etc/dhcpd.conf 中使用 domain-search 选项设置搜索域，而不是使用 domain-name 选项。两者之间的区别在于，使用 domain-name 时只会追加单个域名，但是使用 domain-search 选项时，可以添加多个域名，然后逐一“搜索”直到找到主机位置。

domain-search 选项的外观如下：

```
option domain-search "example.com", "home.arpa"
```

然后我们需要设置 Unbound 来处理我们的固定 IP 地址。在这个例子中，我们只有一个 Web 服务器，但您可以使用尽可能多的主机。您可以直接编辑 Unbound 的主配置文件，但我更喜欢将其放入一个单独的文件中，然后从主文件中包含它。创建一个新文件，命名为类似 /var/unbound/etc/unbound-local.conf 并设置您的主机：

```
local-data: "lilo.example.com IN A 192.168.3.2"
local-data-ptr: "192.168.3.2 lilo.example.com"
```

或者如果您使用 home.arpa 版本：

```
local-data: "lilo.home.arpa IN A 192.168.3.2"
local-data-ptr: "192.168.3.2 lilo.home.arpa"
```

请注意 local-data-ptr 字段中的 IP 地址是反向的，这不是错误。

然后将以下内容添加到我们的 /var/unbound/etc/unbound.conf ：

```
private-address: 192.168.0.0/16
private-domain: example.com # Use home.arpa instead if you need that.
include: "/var/unbound/etc/unbound-local.conf"
```

重新启动 dhcpd 和 Unbound：

```
# rcctl restart dhcpd
# rcctl restart unbound
```

如果您从其中一个 LAN 上连接的计算机中拔出以太网电缆，然后重新插入，您会注意到 /etc/resolv.conf 已添加了 domain 选项：

```
domain example.com
nameserver 192.168.1.1
```

您可以将上述示例扩展到跨越所有段的多个域和多个主机。

### 添加 pf-badhost

當您設置好您的 OpenBSD 路由器後，我強烈建議您將 pf-badhost 添加到您的安裝程序中！

pf-badhost 是由 Jordan Geoghegan 製作的輕量級安全腳本，可以阻止許多互聯網上最大的惱人之處。藉助該腳本，諸如 SSH 和 SMTP 的暴力破解器等令人討厭的行為基本上被消除了。

pf-badhost 定期從眾所周知的垃圾郵件寄信人 IP 數據庫（如 Spamhaus、Firehol、Emerging Threats 和 Binary Defense）中提取 IP 地址，這些 IP 地址經常被記錄為不良 IP 地址。然後，pf-badhost 將收集的 IP 地址添加到 PF 防火牆中作為默認封鎖的表。

### unbound-adblock

unbound-adblock 是由 Jordan Geoghegan 制作的另一个脚本，允许您阻止在线广告网络。如果您更喜欢，可以将 unbound-adblock 用作我的 DNSBlockBuster 的替代方案。

### 推荐阅读

* OpenBSD PF - OpenBSD FAQ 中的用户指南。
* Michael Warren Lucas 的《绝对 OpenBSD，第二版》。自从 Michael 写这本书以来，部分 PF 语法已经发生了变化，但仍然非常有用。
* Michael Warren Lucas 的《系统管理员的网络技术》。
* [ 开放 BSD 和你](https://home.nuug.no/~peter/openbsd_and_you/#1)
* [停止使用荒谬低的 DNS TTLs](https://blog.apnic.net/2019/11/12/stop-using-ridiculously-low-dns-ttls/)

### 相关链接

* [支持 DNSCrypt 和 DNS-over-HTTP2 协议的大量公共 DNS 解析器列表](https://github.com/DNSCrypt/dnscrypt-resolvers/blob/master/v3/public-resolvers.md)
* [公开可用的 DoH 服务器的 cURL](https://github.com/curl/curl/wiki/DNS-over-HTTPS)
* [另一个 DoH 服务器列表](https://github.com/oneoffdallas/dohservers)

### 如何贡献指南？

如果您有任何意见、更正或认为合适的更改，请考虑贡献。

* 在 GitHub 上克隆
* 提交一个 pull 请求以供考虑

你也可以使用电子邮件 :)

请注意：我不接受指南的翻译，因为我无法确保翻译能够保持更新或者正确。

### 由创建和维护

[ Unix 摘要](https://unixdigest.com/)

OpenBSD 路由器指南获得知识共享署名 4.0 国际许可证。

如果您发现这篇内容有用，请考虑在 Patreon 上支持我 :)
