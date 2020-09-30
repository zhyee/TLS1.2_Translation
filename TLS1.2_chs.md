## 1. 介绍

TLS协议的主要目标是保护两个应用之间传输数据的私密性和完整性。此协议由两层组成：TLS记录协议和TLS握手协议。最底层是构建于一些可靠传输协议（例如TCP）之上的TLS记录协议。TLS记录协议在两方面保证了连接的安全性：

- 连接是私密的。协议使用对称加密算法（例如AES，RC4等）加密数据。每个连接使用的对称加密的Key是唯一生成的，并且使用另一个协议（例如TLS握手协议）来加密传输。记录协议也可以不加密使用。

- 连接是可靠的。消息在传输过程中会使用keyed MAC进行完整性检验。MAC计算使用安全的hash函数（例如SHA-1等等）。记录协议操作可以没有MAC，但通常只有另一个协议正在使用记录协议进行安全参数的交换时使用这种模式。

TLS记录协议用于封装某些更高层次的协议。TLS握手协议就是这样一种封装后的协议，它允许服务端和客户端验证对方的真实性和在传输第一个字节的数据前交换加密算法和加密key。TLS握手协议在三个方面保证连接的安全性。

- 各端的身份使用非对称的或者公钥算法（例如RSA，DSA等等）进行真实性检验。这种身份校验是可选的，但通常至少一方需要进行这种校验。

- 协商共享秘钥的过程是安全的：交换的秘钥对窃听者是不可用的，且任何未识别的连接无法获取秘钥，即使是一个能够把自己放到连接之间的攻击者也无法获取。

- 协商的过程是可靠的：没有攻击者能够在不被连接各方侦察到的情况下修改协商通信。

TLS的一大优势是它独立于应用层协议，更高层的协议可以透明的构建在TLS之上。然而，TLS协议并不会指定如何用协议来增强安全性，如何初始化TLS握手和如何解析交换的身份认证证书依赖于协议的设计者和实现者去决断。

## 1.1. 需要知道的术语

此文档中的术语 “MUST”，“MUST NOT”，“REQUIRED”，“SHALL”，“SHALL NOT”，“SHOULD”，“SHOULD NOT”，"RECOMMENDED"，“MAY”和“OPTIONAL”在[RFC 2119](https://www.rfc-editor.org/rfc/rfc2119)中做了解释。

## 1.2. 和TLS1.1版的主要差异

此文档是TLS 1.1协议的修订版，提升了协议的灵活性，尤其是对加密算法的协商过程。主要的差异点有：

- MD5/SHA-1组合的伪随机数函数（PRF）已经被加密套件指定的PRFs替换。此文档中的所有加密套件都是使用P_SHA256。

- 数字签名元素中MD5/SHA-1的组合已经被一个单独的hash函数取代，签名元素现在包括一个字段明确指定所使用的hash算法。

- 大量削减客户端和服务端指定hash算法和签名算法的能力操作能够指定使用何种hash算法和签名算法。注意这样也会放开一些之前版本签名算法和hash算法的限制

- 增加更多的数据模式来支持基于身份验证的加密。

- 扩展的定义和AES加密套件从第三方（TLSEXT和TLSAES）合并过来。

- 更严格的检查EncryptedPreMaster的版本号。

- 一些要求更加的严格。

- 数据长度校验现在依赖于加密套件（默认的仍是 12）。

- 删除了对Bleichenbacher/Klima攻防的描述。

- 很多场景下现在必须发送报警。

- 证书请求发起之后，如果没有可用的证书，客户端必须发送一个空证书列表。

- 使用TLS_RSA_WITH_AES_128_CBC_SHA算法实现加密套件是现在的强制要求。

- 增加HMAC-SHA256加密套件

- 移除IDEA和DES加密套件。它们现在已经过时了，并且会在一个单独的文档中描述。

- 对SSLv2版hello的向后兼容现在是MAY而不是SHOULD，而发送SSLv2版hello现在是SHOULD NOT。对它的支持将来也可能会变成SHOULD NOT。

- 增加了限制性的“fall-through”语法到表现语言中以便允许不同的情况有相同的编码。

- 增加了一个实现危险系数的章节。

- 一般性的说明和编辑工作。

## 2. 协议的目标

TLS协议的目标，按优先级排序如下：

    1. 加密的安全性：TLS应该用于双方建立安全的连接。

    2. 互操作性：独立的程序员应该能够使用TLS来开发应用程序，并且在不知道另一个人的代码的前提下成功的能与之交换参数。

    3. 可扩展性：TLS寻求实现一种框架，以便需要时能够方便的添加进来新的公钥和批量加密方法。这包括实现两个子目标：防止产生创建新协议的需要（并冒着产生新缺陷的风险）和避免实现一个新的完整的安全库。

    4. 相对的高效：加密操作相对是高CPU密集型的工作，尤其是公钥加密。因此，TLS协议已经支持了一个可选的会话缓存机制来减少创建连接的次数。此外，我们还把关注点放到了减少网络事件上。

## 3. 此文档的目标

此文档和TLS协议本身基于由网景公司发布的SSL 3.0协议说明发展而来。此协议和SSL 3.0协议之间的差距并不显著，但是差异也足够让TLS和SSL 3.0之间无法互相兼容（尽管每种协议都支持降级到前一个版本的机制）。本文档主要面向想要实现协议的读者和对协议做加密分析的用户。此说明书也是用这种思路来编写，且打算实现两组人不同的需求。为此，很多算法相关的数据结构和规则正文中（和附录相对），提供了更加容易的访问方式。

此文档并不打算提供任何服务定义或接口定义相关的详情。尽管它的确包含了策略的选择，但这是保持健壮的安全性的需要。

## 4. 表示语法

此文档使用一种外部的表示法来表示数据格式。接下来这种非常基础和有点随意定义的表示法将会使用。这种语法在结构上有几种来源。尽管它在语法上和编程语言“C”很相似，在XDR上和“C”的语法和缩进都很相似，借鉴太多相似的东西可能是一个风险。这种表示语言的目的仅仅局限于TLS文档；并没有超出特定目标的一般性的应用。

## 4.1. 基础数据块大小

所有数据项的表示是明确指定的。基础的数据块大小是1字节（也就是8位）。多字节数据项由多字节来组合，从左至右，从高位到低位。在字节流中，一个多字节的数据项（以数字为例）组合如下（使用C表示）:
```c
value = (byte[0] << 8*(n-1)) | (byte[1] << 8*(n-2)) | ... | byte[n-1];
```

这种多字节值的字节序就是常见的网络字节序或者叫大端法。

## 4.2. 其他约定

注释以“/\*”开始和“\*/”结束。

可选的组成部分用双括号“[[]]”括住来表示。

单字节没有特定含义的实体被认为是不透明的数据类型。

## 4.3. 向量

向量（一维数组）是一系列同类型的数据的流。向量的大小可能在代码编写阶段指定也可能在运行阶段才指定。另一方面，长度是指向量中字节的数量，而不是元素的数量。指定一种固定长度的新类型向量T'，其中每个元素的数据类型为T，它的语法是：

```
T T'[n];
```

这里，T'占用数据流中的n个字节，n是类型T大小的倍数。数据流中不存储向量的长度本身。

在接下来的这个例子里，Datum被定义成三个连续字节的不会被协议做特定解释的数据类型，而Data由三个连续的Datum类型组成，一共9个字节。

```
opaque Datum[3];      /* 三个没有特定含义的字节 */
Datum Data[9];        /* 3个连续的3字节向量 */
```

变长向量通过指定一个合法的长度范围（包括边界在内）来定义，使用记号<floor..ceiling>。当编码这种数据时，字节流在传输向量的实际内容之前，会传输向量的长度。保存长度所需的字节以存储向量指定的最大（上限）长度所需的字节为准。实际长度指定为0的变长向量被认为是空向量。

```
T T'<floor..ceiling>;
```

在下面这个例子中，mandatory是一个必须包含300到400字节的opaque类型数据的向量，它不能为空。它的实际长度需要两个字节的uint16来表示，这足够存储值400了（查看4.4节）。另一方面，longer可以表示到800字节的数据，或者400个uint16元素，它可以为空。向量编码的头部将会添加两个字节的字段表示实际长度。编码后的向量必须是单个元素长度的偶数倍（例如，17字节的uint16类型的向量是不合法的）。

```
opaque mandatory<300..400>;    /* 表示长度的字段为2个字节，不能为空 */
uint16 longer<0..800>;         /* 0到400个 16bit的 无符号整数 */
 ```

 ## 4.4. 数字

 基本的数字数据类型是无符号的byte（uint8）。更大的其他数字数据类型由固定长度的一系列byte串接组成，正如4.1节中所介绍的，它们同样是无符号的。预定义了以下几种数据类型：

```
uint8 uint16[2];
uint8 uint24[3];
uint8 uint32[4];
uint8 uint64[8];
```

本文档的所有值，都是用网络字节序（大端法）来存储的；用十六进制01 02 03 04表示的uint32和十进制的数16909060相等。

注意在某些场景下（例如DH参数）需要使用opque向量来表示整数。在这种场景下，它们被当做无符号的整型（例如，即使设置了最高有效位，0开头的八进制也是不必要的）。

## 4.5. 枚举

另一种支持的较少出现的数据类型叫做枚举。枚举类型的值仅仅能够使用定义时声明的值。每一次定义都是不同的类型。只有相同类型的枚举值才能够相互赋值和比较。枚举中的每个元素必须赋值，如下面所示的例子。由于枚举值的元素是无序的，它们能够以任何顺序包含任何不同的值。

```
enum {e1(v1), e2(v2), ... , en(vn) [[, (n)]]} Te;
```

枚举值在字节流中占用的空间和它的最大序列值相同。下面的定义将会使用1个字节来保存类型Color中的字段。

```
enum { red(3), blue(5), white(7) } Color;
```

枚举可以指定一个没有相关联的tag的值来强制定义宽度，从而可以避免定义一个不必要的元素。

在下面的这个例子中，Taste会占用数据流中的两个字节，但是只能赋值1，2或4。

```
enum { sweet(1), sour(2), bitter(4), (32000) } Taste;
```

枚举值元素的名称需要带上所在的枚举类型名称，在第一个例子中，对Color第二个元素的完全限定引用的写法是Color.blue。如果赋值的目标已经指定了类型，这种限定可以省略。

```
Color color = Color.blue;   /* 重复指定，是合法的 */
Color color = blue;         /* 正确，隐式类型指定 */
```

对于从来不会转换成外部表示的枚举，其中的数字信息是可以省略的。

```
enum { low, medium, high } Amout;
```

## 4.6. 结构化的类型

结构体可以由基础数据类型构造而成。每种结构体会定义一种新的，唯一的类型。定义的语法和C语言非常的相似。

```
struct {
  T1 f1;
  T2 f2;
  ...
  Tn fn;
} [[T]]
```

一个结构体中的字段可以使用类型名称来引用，和枚举的使用语法非常类似。例如，T.f2指向上面定义中的第二个字段。

结构体的定义支持嵌套。

## 4.6.1. 变体结构

定义结构体时可能会存在变体字段，变体字段的值可能由当前环境中的一些已知信息来决定。选择器必须是由一个结构体中可能用到的值所组成的枚举类型。select中定义的枚举中每个值都应该有一个case和它对应。case选项允许包含有限的fall-through：如果两个case选项紧挨着并且之间没有字段的话，那么它们则使用相同的解释。因此，在下面的例子中，“orange”和“banana”同时指向V2。注意这是TLS 1.2协议中的新语法。

变体结构体的主题内容可以用一个符号名称来引用。变体选项的值是在运行时确定的而不是在语言定义时。

```
struct {
  T1 f1;
  T2 f2;
  ....
  Tn fn;
   select (E) {
      case e1: Te1;
      case e2: Te2;
      case e3: case e4: Te3;
      ....
      case en: Ten;
   } [[fv]];
} [[Tv]];
```

例如：

```
  enum { apple, orange, banana } VariantTag;

  struct {
    uint16 number;
    opaque string[0..10]; /* 变长 */
  } V1;

  struct {
    uint32 number;
    opaque string[10];  /* 固定长度 */
  } V2;

  struct {
    select (VariantTag) { /* 选择器的值是隐式的 */
      case apple:
        V1;  /* 变体内结构体值, tag = apple */

      case orange:
      case banana:
        V2;  /* 变体结构体值，tag = orange 或 banana */    
    } variant_body;  /*变体结构体可选的符号名称*/
  } VariantRecord;
```

## 4.7 加密属性

五种加密的操作 - 数字签名，流式加密，块式加密，使用额外数据的授权加密（AEAD）和公钥加密，这些加密方法分别被用于数字签名，流式加密，块式加密， AEAD加密和公钥加密。一个字段的加密处理方式通过在字段类型声明前面添加一个合适的关键词来指定。当前会话状态必然要有加密key（查看第6.1节）。

一个数字签名的元素会被编码成DigitallySigned结构体：
```
struct {
  SignatureAndHashAlgorithm algorithm;
  opaque signature<0..2^16-1>;
} DigitallySigned;
```

algorithm字段指定使用的算法（查看第7.4.1.4.1节来了解此字段的定义）。注意算法字段的介绍已经和老的协议版本不一样了。signature字段是用这些算法在元素内容上计算出来的结果。内容本身没有出现在报文中而是简单的进行计算。签名的长度受算法和key长度决定。

在RSA签名中，向量值为[PKCS1](https://www.rfc-editor.org/rfc/rfc5246.html#ref-PKCS1)中定义的RSASSA-PKCS1-v1_5签名方案生成的签名。正如PKCS1中所说明的，DigestInfo必须是DER编码（可辨别编码规则）的（[X680](https://www.rfc-editor.org/rfc/rfc5246.html#ref-X680) [X690](https://www.rfc-editor.org/rfc/rfc5246.html#ref-X690)）。对于没有参数的hash算法（包括SHA-1），DigestInfo.AlgorithmIdentifier.paramters属性必须为NULL，但在实现上必须同时接受无参数或带NULL参数的调用。注意早期的TLS版本使用不同的RSA签名方法，其不包括DigestInfo编码。

在DSA算法中，20字节的SHA-1哈希算法直接通过数字签名算法运行，而不需要进行额外的hash。这会生成两个值，r和s。DSA签名算法是一个非透明的向量，它的内容用DER编码为：
```
Dss-Sig-Value ::= SEQUENCE {
    r INTEGER,
    s INTEGER
}
```

提示：在当前的术语下，DSA指代Digital Signature Algorithm，而DSS指代NIST标准。在最初的SSL和TLS文档中，“DSS”是普遍使用的。本文档中使用“DSA”来指代算法，“DSS”来指代标准，且在代码点定义上使用“DSS”来保持历史的一致性。

在流式密码加密中，原文会与从一个加密安全的伪随机数生成器生成的相同长度的字符串进行异或运算。

在块式密码加密中，每一组原文被加密成一组密文。所有的块密码加密使用CBC（Cipher Block Chaining）模式，并且所有的项目都是块加密的，最终长度会是加模块长度的整数倍。

在AEAD加密方式中，原文同时进行加密和完整性的保护。输入可能是任何长度，AEAD加密输出通常比输入要长，以便容纳完整性检验值。

在公钥加密中，一个公钥算法用于加密数据且加密后的数据只能由相对应的私钥来解密。公钥加密后的元素会编码成一个原始向量vector<0..2^16-1>，它的长度由加密算法和key来决定。

RSA加密使用PKCS1中定义的RSAES-PKCS1-v1_5加密方法来完成。

在下面的这个例子中：

```
stream-ciphered struct {
    uint8 field1;
    uint8 field2;
    digitally-signed opaque {
      uint8 field3<0..255>;
      uint8 field4;
    };
} UserType;
```

内部结构体的内容（field3和field4）被用做签名/hash算法的输入，然后整个结构体用流式加密方法来加密。此结构体的长度，用字节来表示，将是两个字节用来存储field1和field2，加上两字节来存储签名和哈希算法，加上两字节用来存储签名的长度，再加上生成的签名的长度，由于用于加密的算法和key的在加解密结构体之前已经知道，所以签名的长度也是确定的。


## 4.8. 常量

可以定义常量用于说明的目的，通过定义一个期望类型的符号并赋值给它。

非确定类型（opaque，可变长度向量和包含opaque的结构体）不能用于赋值。一个多字段的结构体或向量的字段是不能省略的。

例如：

```
struct {
  uint8 f1;
  uint8 f2;
} Example1;

Example1 ex1 = {1, 4};  /* assigns f1 = 1, f2 = 4 */
```


## 5. HMAC和伪随机数方法

TLS记录层使用秘钥散列消息认证码（MAC）来保护消息的完整性。本文档中的加密套件使用基于哈希函数的HMAC的机制，这定义在[HMAC](https://www.rfc-editor.org/rfc/rfc5246.html#ref-HMAC)。如果需要其他的套件，相应的套件可能定义了它们自己的MAC机制。

此外，需要一种机制来把消息扩展为分组数据以便构造密钥或者实现验证。伪随机数生成函数（PRF）能够把输入当做秘钥，种子和一个标识符，然后产生任意长度的输出。

在本节中，我们基于HMAC定义一个伪随机数。这个伪随机函数使用SHA-256哈希算法，此哈希算法同时应用在本文档和TLS1.2协议之前的TLS文档中的所有加密套件中。新的加密套件必须显式指定一个PRF，并且PRF应该配合SHA-256或者更加健壮的哈希函数来使用。

首先，我们定义一个数据扩充函数：P_hash(secret, data)，其使用一个独立的哈希函数能够扩充秘钥和种子为任意长度的输出：

```
P_hash(secret, seed) = HMAC_hash(secret, A(1) + seed) +
                       HMAC_hash(secret, A(2) + seed) +
                       HMAC_hash(secret, A(3) + seed) + ...
```

这里的“+”表示拼接的意思。

函数A() 被定义为：

```
A(0) = seed
A(i) = HMAC_hash(secret, A(i-1))
```

P_hash 能够按需要迭代多次来产生所需要长度的数据。例如：如果使用P_SHA256来生成80字节的数据，则必须迭代三次（到A(3)结束）生成96字节的数据；最后一次迭代的后16字节将被丢弃，最终输出80字节的数据。

TLS的PRF是在秘钥上应用P_hash函数来实现的：
```
PRF(secret, label, seed) = P_<hash>(secret, label + seed)
```

label 是一个ACSII码字符串。给定label字符串时不应该包含长度或者null结尾。例如，label “slithy toves” 进行哈希处理时会被当做：

```
73 6C 69 74 68 79 20 74 6F 76 65 73
```

## 6. TLS记录协议

TLS记录协议是一个分层的协议。在每一层中，消息可能包含长度、描述、以及内容等字段。记录协议处理消息以便传输，把数据拆分成可管理的数据块，以及应用压缩，MAC，加密等可选的操作，最后传输这个结果。接受数据的处理步骤则是解密、校验数据、解压缩、重新组装数据块，然后交给更高层的协议。

此文档中介绍了四个使用记录协议的协议，它们分别是：握手协议，报错协议，切换加密方式协议，应用层数据协议。为了便于TLS协议的扩展，其他的数据记录类型也可以被记录协议所支持。在TLS内容类型注册中心，新的记录内容类型值由IANA分配，在[第12节]（https://www.rfc-editor.org/rfc/rfc5246.html#section-12）有所描述。

协议的实现一定不能发送未在此文档中定义的记录类型，除非通过一些扩展来协商实现。如果在TLS协议实现中一方收到一个非预期的记录类型，则必须发送unexpected_message错误。

使用TLS协议的任何协议设计时必须小心处理任何可能通过它的攻击。作为一个实践方面的因素，这意味着协议设计者必须意识到TLS提供了哪些安全属性，以及未提供哪些安全属性，不能依赖于未提供的安全属性。

尤其注意记录的类型和长度是不被加密保护的，如果这些信息本身是敏感的，应用程序设计者可能需要采取一些措施（填充、cover traffic）来减少信息的泄露。

## 6.1. 连接状态

TLS的连接状态是TLS记录协议的操作环境。它指定了一个压缩算法，一个加密算法和一个MAC算法。额外的，这些算法的参数是已知的：比如在连接的读和写任一方向上的MAC秘钥和块加密算法的秘钥。逻辑上来说，总是存在4种明显的连接状态：当前读和当前写状态以及等待读和等待写状态。所有的记录都是在当前读和当前写状态下处理。等待状态下的安全参数可以由TLS握手协议来设置，且ChangeCipherSpec可以选择性的使其中一个等待状态变为当前状态，在这种情况下合适的当前状态被应用并替换等待状态。之后等待状态会被重新初始化成一个空状态。把没有使用安全参数初始化的状态转变成当前状态是非法的。最初的当前状态总是指定没有加密，没有压缩或者MAC会被使用。

TLS连接的读写状态的安全参数通过以下值来设置：

- 连接终端
用于判断连接中的实体是“客户端”还是“服务端”。

- PRF 算法

一个用于从主秘钥生成keys的算法（查看第5节和6.3节）。

- 分组加密算法

一个用于分组加密的算法。相关指标包括算法的key大小，是否是块数据、流数据或AEAD加密，加密块的大小（如果存在的话），以及初始化向量（或nonces）的显式或隐式长度。

- MAC算法
一个用于消息认证的算法。相关指标包含MAC算法返回值的大小。

- 压缩算法
一种用于数据压缩的算法。此参数必须包含压缩所需要的所有信息。

- 主秘钥

连接双方公用的一个48字节的秘钥。

- 客户端随机数

客户端提供的一个32字节的随机数。

- 服务端随机数

服务端提供的一个32字节的随机数。

这些参数用语言来表示如下：

```
enum {server, client} ConnectionEnd;

enum {tls_prf_sha256} PRFAlgorithm;

enum {null, rc4, 3des, aes} BulkCipherAlgorithm;

enum {stream, block, aead} CipherType;

enum {null, hmac_md5, hmac_sha1, hmac_sha256, hmac_sha384, hmac_sha512} MACAlgorithm;

enum {null(0), (255)} CompressionMethod;

/* 在CompressionMethod, PRFAlgorithm, BulkCipherAlgorithm 和 MACAlgorithm参数中指定的算法是可以被添加的 */

struct {
    ConnectionEnd         entity;
    PRFAlgorithm          prf_algorithm;
    BulkCipherAlgorithm   bulk_cipher_algorithm;
    CipherType            cipher_type;
    uint8                 enc_key_length;
    uint8                 block_length;
    uint8                 fixed_iv_length;
    uint8                 record_iv_length;
    MACAlgorithm          mac_algorithm;
    uint8                 mac_length;
    uint8                 mac_key_length;
    CompressionMethod     compression_method;
    opaque                master_secret[48];
    opaque                client_random[32];
    opaque                server_random[32];
} SecurityParameters;

```

记录层会使用安全参数来生成以下6个项目（不是所有的加密方法都需要其中所有的项，这时会置空）：

client write MAC key
server write MAC key
client write encryption key
server write encryption key
client write IV
server write IV

当接收和处理记录时服务端会使用客户端写参数，反之亦然。从安全参数创建这些项目所用到的算法在第6.3节有所描述。

一旦设置了安全参数并且key已经生成，则可以通过使它们成为当前状态来初始化连接状态。这些当前状态必须为每一个已处理的记录更新。每一个连接状态包含下列元素：

- 压缩状态
压缩算法的当前状态。

- 加密状态
加密算法的当前状态。这包括用于连接的预定的key。对于流式加密来说，这也包含允许数据流继续进行加密或者解密所需的所有状态信息。

- MAC key
用于此连接的MAC key，如上面所创建的。

- 序列数
每个连接状态包含一个序列号，读和写状态的序列号各自维护。无论什么时候当连接状态被激活时，序列号必须设置为0。序列号是uint64类型，不能超过 2^64-1。序列号不能被包裹，如果一个TLS协议的实现需要包裹一个序列号，必须重新协商。序列号会随着记录而数值增长。在一个特别的连接状态下传输的首个记录

## 6.2 记录层

TLS 记录层从更高层接收未解释的数据，此数据存放在任意大小的非空块中。

## 6.2.1. 分片

记录层分片信息块到TLS纯文本记录，每个纯文本记录携带2^14字节或更少的数据。客户端消息的界限并不会保留在记录层（例如，多个相同内容类型的客户端消息可能会被合并到一个TLS纯文本记录，或者单个消息可能会被分片成多个记录）。

```
struct {
  uint8 major;
  uint8 minor;
} ProtocolVersion;

enum {
  change_cipher_spec(20), alert(21), handshake(22), application_data(23), (255)
} ContentType;

struct {
  ContentType type;
  ProtocolVersion version;
  uint16 length;
  opaque fragment[TLSPlaintext.length];
} TLSPlaintext;
```

- type
用于处理附带的分片数据的更上层协议。

- version
应用协议的版本。本文档所描述的是TLS 1.2版本，使用版本号{3, 3}来表示。3.3的版本号是有历史原因的，源自使用{3, 1}来表示TLS 1.0（查看附录A.1）。注意支持多版本TLS协议的客户端在收到服务端的Hello消息前可能并不知道需要使用哪个版本的协议。查看附录E关于ClientHello应该使用哪个记录层协议版本号的讨论。

- length
紧接着的TLSPlaintext.fragment的长度（字节为单位）。长度不能超过2^14。

- fragment
应用数据。此数据是透明的并且由type字段指定的更高层次协议当做独立的数据块来处理。

协议实现一定不要发送长度为0的握手，警告，或切换加密细节内容类型的数据分片。由于作为一个通信分析手段可能是有用的，所以长度为0的应用数据分片可能会被发送。

注意：不同的TLS记录层类容类型的数据可能会穿插在一起。应用数据一般相比其他的类容类型有较低的传输优先级。然而，记录必须按照它们被记录层所保护的相同顺序交给网络。在握手到随后的第一次传输，接收数据的一方必须接收和处理穿插的应用层通信数据。

## 6.2.2. 记录的压缩和解压缩

所有的记录都会使用在当前会话状态中定义的压缩算法来进行压缩。总会存在一个活动的压缩算法；然而，一开始它会被定义成CompressionMethod.null。压缩算法会把一个TLSPlaintext结构转换成一个TLSCompressed结构。无论何时连接状态变得活跃时，压缩方法都会使用默认的状态信息来初始化。[RFC3749](https://www.rfc-editor.org/rfc/rfc3749)介绍了TLS的压缩算法。

压缩必须是无损的，并且不能让内容长度增长超过1024字节。如果解压缩方法遇到一个TLSCompressed.fragment解压的长度超过了2^14字节，必须报告一个致命的解压缩失败错误。

```
struct {
  ContentType type; /* 和TLSPlaintext.type相同 */
  ProtocolVersion version; /* 和TLSPlaintext.version相同 */
  uint16 length;
  opaque fragment[TLSCompressed.length];
} TLSCompressed;
```

- length
TLSCompressed.fragment内容的长度（以字节为单位）。
此长度不能超过2^14 + 1024。

- fragment

TLSPlaintext.fragment压缩后的形式。
注意：CompressionMethod.null 操作是一个恒等操作，所有字段都不会改变。

实现上注意：解压缩方法需要确保消息不会导致内部缓冲区溢出。

## 6.2.3. 记录载体的保护

加密和MAC函数会把TLSCompressed结构转换成TLSCiphertext结构。解密函数做相反的操作。记录的MAC值包含一列数以便能够检测到消息的丢失，添加或者重复。

```
struct {
  ContentType type;
  ProtocolVersion version;
  uint16 length;
  select (SecurityParameters.cipher_type) {
    case stream: GenericStreamCipher;
    case block: GenericBlockCipher;
    case aead:  GenericAEADCipher;
  } fragment;
} TLSCiphertext;
```
- type
type字段等同于TLSCompressed.type。

- version
version字段等同于TLSCompressed.version。

- length
TLSCiphertext.fragment的内容长度（单位为字节）。
长度一定不能超过2^14 + 2048。

- fragment
TLSCompressed.fragment加密后的形式，包含MAC。

## 6.2.3.1. Null或标准流式加密

流式加密（包括BulkCipherAlgorithm.null；查看附录A.6）转换TLSCompressed.fragment结构到TLSCiphertext.fragment结构或反向转换。

```
stream-ciphered struct {
  opaque content[TLSCompressed.length];
  opaque MAC[SecurityParameters.mac_length];
} GenericStreamCipher;
```

MAC以如下方式创建：

MAC(MAC_write_key, seq_num +
                        TLSCompressed.type +
                        TLSCompressed.version +
                        TLSCompressed.length +
                        TLSCompressed.fragment);

 这里的“+”表示字符串拼接。

 - seq_num
 此记录的序列号。

 - MAC
 由SecurityParameters.mac_algorithm指定的MAC算法。

 注意MAC值是在加密之前计算的。流式加密器会加密整个数据块，包括MAC数据。对于不使用同步向量的流式加密器（例如RC4）来说，一条记录结束时的流式加密状态会简单的用于随后的数据包。如果加密套件使用TLS_NULL_WITH_NULL_NULL，加密由恒等操作（例如，数据不会被加密，MAC的大小为0，意味着没有使用MAC）组成。对于null和流式加密来说，TLSCiphertext.length的长度是TLSCompressed.length的长度加上SecurityParamters.mac_length的长度。

## 6.2.3.2. CBC分组加密

对于分组加密（例如3DES或AES），加密和MAC方法转换TLSCompressed.fragment结构到分组的TLSCiphertext.fragment结构。

```
struct {
  opaque IV[SecurityParameters.record_iv_length];
  block-ciphered struct {
    opaque content[TLSCompressed.length];
    opaque MAC[SecurityParameters.mac_length];
    uint8 padding[GenericBlockCipher.padding_length];
    uint8 padding_length;
  }
} GenericBlockCipher;
```

第6.2.3.1节介绍了MAC的生成。

- IV
初始向量（IV）的选择应该具备随机性，并且是无法预测的。注意TLS1.1版本之前是没有IV字段的，上一个记录的最后加密文本块（CBC残留）被当做IV来使用。之所以做了修改是为了阻止[CBCATT](https://www.rfc-editor.org/rfc/rfc5246.html#ref-CBCATT)当中介绍的攻击方式。对于分组加密来说，IV的长度就是SecurityParameters.record_iv_length的长度，它的值等于SecurityParameters.block_size。

- padding
Padding被添加到原文之后用于使原文的长度为分组加密块长度的整数倍。padding可以是255字节以内的任意长度，只要它们使得TLSCiphertext.length是分组长度的整数倍即可。超过必要的长度可能会导致针对协议的令人不快的攻击，这种攻击它基于分析交换消息的长度。padding数据中的每个字节都为填充内容的长度。数据接收者必须做padding的校验，如果出错必须使用bad_record_mac来报警以表明出错。

- padding_length
padding长度必须能保证GenericBlockCipher结构的总长度是分组长度的整数倍。合法的padding长度是0到255，包括255。这个长度指定padding字段的长度而不是padding_length字段本身。

加密后的数据长度（TLSCiphertext.length）是一个长度大于SecurityParameters.block_length、TLSCompressed.length、SecurityParameters.mac_length和padding_length四者之和的值。

例如：如果分组长度是8字节，待加密内容（TLSCompressed）的长度是61字节，MAC长度是20字节，那么填充前的数据长度是82字节（这不包括IV的长度）。因此，padding长度可以为6以便使得整个数据长度是8字节（8是分组的长度）的整数倍。padding的长度可以是6，14，22...一直到254。如果padding长度需要取最小值，那么padding长度就是6字节，且每个字节的值都为6。因此，分组加密前的GenericBlockCipher最后的8个字节将为 `xx 06 06 06 06 06 06 06`，这里的xx是MAC的最后一个字节。

注意：对于CBC（Cipher Block Chaining）模式的分组加密，在任何密文被传输之前，记录的全部原文都是已知的，这点非常的关键。否则有可能让攻击者发动[CBCATT](https://www.rfc-editor.org/rfc/rfc5246.html#ref-CBCATT)中所介绍的攻击。

实现上的注意事项：Canvel等。[CBCTIME](https://www.rfc-editor.org/rfc/rfc5246.html#ref-CBCTIME)已经证明了一种基于计算MAC所需时间的针对CBC padding的时序攻击。为了阻止这种攻击，协议的实现必须确保不管padding是否正确，记录的处理时间应该是相同的。一般来说，最佳的实现方法是即使padding是不正确的也去计算MAC值，然后直到那时才去拒绝数据包。例如，如果填充是不正确的，协议实现应该假设长度为0的填充并计算MAC值。由于MAC计算的性能基于数据分片的大小范围，这里留下了一个小的时序通道。但是相信这不足以被利用，由于MAC的大体积和时序信号的小体积。

## 6.2.3.3. AEAD加密

对于AEAD加密（例如CCM和GCM），AEAD方法转换TLSCompressed.fragment结构到AEADTLSCiphertext.fragment结构，或者反向转换。

```
struct {
  opaque nonce_explicit[SecurityParameters.record_iv_length];
  aead-ciphered struct {
    opaque content[TLSCompressed.length];
  };
} GenericAEADCipher;
```

AEAD加密把输入当做一个单独的key，原文和验证检测中包含的“额外数据”，如[AEAD](https://www.rfc-editor.org/rfc/rfc5246.html#ref-AEAD)第2.1节中所介绍的。这个key要么是client_write_key，要么是server_write_key。没有使用MAC。
