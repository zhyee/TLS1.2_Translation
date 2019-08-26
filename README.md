# TLS1.2_Translation

传输层安全协议（The Transport Layer Security Protocol）1.2版翻译。

TLS1.2协议是目前https用于加密传输数据所使用的协议，并广泛使用在浏览器、邮箱、即时通信、VoIP、网络传真等应用程序中。

TLS协议的前身是网景公司推出的SSL（Secure Sockets Layer，安全套接层协议），之后被IETF组织标准化并改名TLS，第一个TLS版本是1.0，以SSL3.0版本为基础发展而来，两者之间差异并不明显但并不兼容，最新的TLS协议版本是1.3版，于2018年8月推出。下面列出SSL/TLS主要版本的发布日期。

| 协议 | 发布时间 | 说明 |
| :----: | :----: | :---- |
| SSL 1.0 | 从未发布 | 存在严重安全漏洞，从未公开过 |
| SSL 2.0 | 1995年2月 | 存在数个严重的安全漏洞 |
| SSL 3.0 | 1996年11月 | 被广泛使用的协议版本，TLS协议的基础 |
| TLS 1.0 | 1999年1月 | 以SSL 3.0为基础发展而来，被IETF标准化 |
| TLS 1.1 | 2006年4月 | TLS 1.0的升级版 |
| TLS 1.2 | 2008年8月 | 应该是目前的主要使用版本 |
| TLS 1.3 | 2018年8月 | 最新的TLS版本 |

由于TLS 1.2仍是当下的主要使用版本，所以本项目选择TLS 1.2的RFC进行翻译，旨在方便大家更了解https的工作原理以及TLS的实现细节，喜欢的可以点点star。

IETF TLS 1.2 RFC英文原版地址： [https://www.rfc-editor.org/rfc/rfc5246.html](https://www.rfc-editor.org/rfc/rfc5246.html)
