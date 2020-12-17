# 记一次设置 Windows 服务器安全组无效的坑



本次踩坑源于客户需求在生产环境安装 Deep Security Agent，关于 Deep Security 可查阅官网 [Deep Security](https://www.trendmicro.com/en_us/business/products/hybrid-cloud/deep-security.html)。

Deep Security 主要包含两个部分，其中一个是 manager，另一个是 Agent，顾名思义，manager 管理了各个 Agent，由于种种历史原因，Deep Security 仅安装在我们的生产环境，测试环境中并没有安装，所以此次安装 Agent，我们需要在测试环境安装完整的 Deep Security(包含 manager 以及 Agent)

一切安装依照官方文档，安装各种数据库依赖等，都还算顺利，直到最后，Deep Security 的 manager 需要对 Agent 开发 4122，以及 4120 的端口，Agent 需要对 manager 开放 4118 的端口。

于是 AWS Security Group 一顿设置，我们完成了该要求。

manager 开放如下 inbound:
| Type       | Protocol | Port range | Source                  | Description - optional |
| ---------- | -------- | ---------- | ----------------------- | ---------------------- |
| Custom TCP | TCP      | 4122       | Agent-Security Group ID | Agent to manager       |
| Custom TCP | TCP      | 4120       | Agent-Security Group ID | Agent to manager       |

manager 开放如下的 outbound:
| Type       | Protocol | Port range | Destination             | Description - optional |
| ---------- | -------- | ---------- | ----------------------- | ---------------------- |
| Custom TCP | TCP      | 4118       | Agent-Security Group ID | manager to Agent       |

Agent 开放如下的 inbound：
| Type       | Protocol | Port range | Source                    | Description - optional |
| ---------- | -------- | ---------- | ------------------------- | ---------------------- |
| Custom TCP | TCP      | 4118       | manager-Security Group ID | manager to Agent       |


Agent 开放如下的 outbound:
| Type       | Protocol | Port range | Destination               | Description - optional |
| ---------- | -------- | ---------- | ------------------------- | ---------------------- |
| Custom TCP | TCP      | 4122       | manager-Security Group ID | Agent to manager       |
| Custom TCP | TCP      | 4120       | manager-Security Group ID | Agent to manager       |

一切准备就绪，却发现 manager，Agent无法正常的通信，错误显示 Agent to manager heartbeat failed.

于是尝试使用 telnet 检查端口通信是否正常：

* 在 manager 服务器上执行：
```shell
telnet agent-ip 4118
```
显示通信正常
* 在 agent 服务器上执行
```shell
telnet manager-ip 4122
telnet manager-ip 4120
```
均通信失败！！！

咦！我明明设置了安全组开放对应的端口，为什么不能正常通信呢？此时开始怀疑是否本机Deep Security进程出现问题，没有正常监视本机 4122 以及 4120 端口

于是在 manager 机器上我重新测试了本地的 4122 4120 端口
```shell
telnet localhost 4122
telnet localhost 4120
```
通信正常！本地没有问题，应用正常监听

此时已经大概猜到问题所在，想起 windows 其实自身也有一个防火墙在，如果我们回想本机上运行一些如 nginx 的应用程序监听本地端口时，经常会弹窗询问我们是否同意监听 8080 端口。其实这就是操作防火墙开放端口的询问。

于是我们在 manager 这台 windows 服务器上以管理员身份执行如下的命令来开放 4122，4120 的端口
```shell
netsh advfirewall firewall add rule name="ds_server" dir=in protocol=tcp localport=4122 action=allow
netsh advfirewall firewall add rule name="ds_server" dir=in protocol=tcp localport=4120 action=allow
```
![](https://upload-images.jianshu.io/upload_images/13279928-4e84972f94c6b2f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
开放本机防火墙之后，到 Agent 服务器上再次使用 telnet 测试与manager的通信，此时通信正常。

***如今已经是云时代了，所有服务器都上云，由于云服务商为我们提供了很多的安全设置，很多时候我们容易忽略了服务器自身的防火墙，导致即使是在云服务商的管理平台上开放了端口，服务器仍然不能正常访问。***