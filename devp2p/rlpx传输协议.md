# RLPx 传输协议

本规范定义了 RLPx 传输协议，其是用于以太坊节点之间的通信的基于 TCP 的传输协议。 该协议传输属于一个或多个“capabilities”的加密消息，这些“capabilities”是在连接建立期间协商的。RLPx以RLP序列化格式命名。该名称不是首字母缩写词，也没有特殊含义。

当前协议版本为**5**。 您可以本文档的结尾找到过去版本的更改列表。

## 约定

`X || Y`:	表示X和Y的串联 ;

`X ^ Y`:	表示X和Y按字节异或;

`X[:N]`:	表示X的N字节前缀;

`[X, Y, Z, ...]`:	表示递归编码为RLP的列表;

`keccak256(MESSAGE)`:	以太坊使用的keccak256哈希函数;

`ecies.encrypt(PUBKEY, MESSAGE, AUTHDATA)` :	 RLPx 使用的非对称认证加密函数。AUTHDATA 是经过身份验证的数据，它不是生成的密文的一部分，而是在生成消息标签之前就已经写入了 HMAC-256；

`ecdh.agree(PRIVKEY, PUBKEY)`:	是 PRIVKEY 和 PUBKEY 之间的椭圆曲线 Diffie-Hellman 秘钥协商协议；

## ECIES 加密

ECIES (Elliptic Curve Integrated Encryption Scheme)是在 RLPx 握手中使用的一种椭圆曲线非对称加密方案。 RLPx使用的密码系统是

- 以生成点`G`生成的椭圆曲线secp256k1；
- `KDF(k, len)`:	NIST SP 800-56 Concatenation Key Derivation Function；
- `MAC(k, m)`: HMAC使用SHA-256哈希方法；
- `AES(k, iv, m)`：CTR模式下的AES-128加密函数；

Alice 想发送一条可以被Bobs私钥<code>k<sub>B</sub></code>解密的加密消息。Alice知道Bobs的公钥<code>K<sub>B</sub></code>。

为了加密消息`m`，Alice生成了一个随机数`r`,和其对应的椭圆曲线公钥`R = r * G`，并计算共享秘密<code>S = P<sub>x</sub></code>，其中 <code>(P<sub>x</sub>, P<sub>y</sub>) = r * K<sub>B</sub></code>，她导出用于加密和认证的key material：<code>k<sub>E</sub> || k<sub>M</sub> = KDF(S, 32)</code> 以及一个随机初始化的向量`iv`，Alice将加密后的信息`R || iv || c || d`发送给Bob，其中<code>c = AES(k<sub>E</sub>, iv, m)</code>并且<code>d = MAC(sha256(k<sub>M</sub>), iv || c)</code> 。

Bob为了能解密消息`R||iv||c||d`，他需要计算出共享的秘密<code>S = P<sub>x</sub></code>，其中<code>(P<sub>x</sub>, P<sub>y</sub>) = k<sub>B</sub> * R</code> ，以及<code>k<sub>E</sub> || k<sub>M</sub> = KDF(S, 32)</code>，Bob验证以下条件是否成立<code>d == MAC(sha256(k<sub>M</sub>), iv || c)</code>然后获得了纯文本消息<code>m = AES(k<sub>E</sub>, iv || c)</code>

## Node身份

所有加密操作均基于 secp256k1 椭圆曲线。每个节点都应维护一个静态 secp256k1 私钥，该私钥在会话之间保存和恢复。建议只能手动重置私钥，例如，通过删除文件或数据库条目。

## Initial Handshake初始握手

RLPx 连接是通过创建 TCP 连接并进行握手协商临时密钥来建立的，以便进一步加密和验证通信。握手过程：

1. 发起者连接接受者，并发送`auth`消息；
2. 接受者接收，解密和验证`auth`(检查签名的反推出的数据==`keccak256(ephemeral-pubk)`)；
3. 接受者根据`remote-ephemeral-pubk` 和`nonce`生成`auth-ack`；
4. 接受者导出共享密钥并发送第一个包含[Hello]的加密过后的帧消息；
5. 发起者接收到`auth-ack`并导出密钥；
6. 发起者发送他的第一个包含[Hello]的加密过后的帧消息；
7. 接受者收到并认证第一个加密过后的帧消息；
8. 发起者收到并认证第一个加密过后的帧消息；
9. 如果第一个加密帧的 MAC 在双方都有效，则加密握手完成；

如果第一个帧数据包的验证失败，任何一方都可以断开连接。

握手消息：

    auth = auth-size || enc-auth-body
    auth-size = enc-auth-body的大小，按16位整数大端编码
    auth-vsn = 4
    auth-body = [sig, initiator-pubk, initiator-nonce, auth-vsn, ...]
    enc-auth-body = ecies.encrypt(recipient-pubk, auth-body || auth-padding, auth-size)
    auth-padding = arbitrary data
    
    ack = ack-size || enc-ack-body
    ack-size = enc-ack-body的大小，按16位整数大端编码
    ack-vsn = 4
    ack-body = [recipient-ephemeral-pubk, recipient-nonce, ack-vsn, ...]
    enc-ack-body = ecies.encrypt(initiator-pubk, ack-body || ack-padding, ack-size)
    ack-padding = arbitrary data

实现上必须忽略`auth-vsn` 和 `ack-vsn`的不匹配。也忽略`auth-body` 和`ack-body`多余的列表元素。

秘钥由以下的握手消息生成：

    static-shared-secret = ecdh.agree(privkey, remote-pubk)
    ephemeral-key = ecdh.agree(ephemeral-privkey, remote-ephemeral-pubk)
    shared-secret = keccak256(ephemeral-key || keccak256(nonce || initiator-nonce))
    aes-secret = keccak256(ephemeral-key || shared-secret)
    mac-secret = keccak256(ephemeral-key || aes-secret)

## Framing帧

握手之后的所有消息都按帧传输。每个帧传输属于某个capability的加密消息。

按帧传输的目的是在单个连接上多路复用多个capability。其次，由于帧消息为消息身份验证代码提供了合理的界限点，能够很好地支持对消息流直接加密和身份验证。帧通过握手期间生成的密钥进行加密和验证。

帧头提供有关消息大小和消息所属的capability的信息，帧消息会加上padding，以便与加密的块大小字节对齐。

    frame = header-ciphertext || header-mac || frame-ciphertext || frame-mac
    header-ciphertext = aes(aes-secret, header)
    header = frame-size || header-data || header-padding
    header-data = [capability-id, context-id]
    capability-id = integer，always 0
    context-id = integer，always 0
    header-padding = 将header的大小填充0到与16字节对齐
    frame-ciphertext = aes(aes-secret, frame-data || frame-padding)
    frame-padding = 将frame-data填充0到与16字节对齐

 `frame-data` 和 `frame-size`定义在下面。

### MAC

RLPx 中的消息认证使用两个 keccak256 状态，每个通信方向一个：`egress-mac` 和 `ingress-mac` 的keccak状态不断使用
发送（egress）或接收（ingress）字节的密文更新。在初始握手之后，MAC 状态初始化如下：

发起者：

    egress-mac = keccak256.init((mac-secret ^ recipient-nonce) || auth)
    ingress-mac = keccak256.init((mac-secret ^ initiator-nonce) || ack)

接受者：

    egress-mac = keccak256.init((mac-secret ^ initiator-nonce) || ack)
    ingress-mac = keccak256.init((mac-secret ^ recipient-nonce) || auth) 

发送帧时，通过使用要发送的数据更新`egress-mac`状态来计算相应的 MAC 值。更新是通过将header和加密的输出的MAC进行异或。这样做是为了确保对明文MAC和密文执行统一的操作。 所有MAC都以明文形式发送。

    header-mac-seed = aes(mac-secret, keccak256.digest(egress-mac)[:16]) ^ header-ciphertext
    egress-mac = keccak256.update(egress-mac, header-mac-seed)
    header-mac = keccak256.digest(egress-mac)[:16]

计算`frame-mac`：

    egress-mac = keccak256.update(egress-mac, frame-ciphertext)
    frame-mac-seed = aes(mac-secret, keccak256.digest(egress-mac)[:16]) ^ keccak256.digest(egress-mac)[:16]
    egress-mac = keccak256.update(egress-mac, frame-mac-seed)
    frame-mac = keccak256.digest(egress-mac)[:16]

通过以与`egress-mac`相同的方式更新`ingress-mac`状态，然后并比较ingress帧中`header-mac`和`frame-mac`的值来验证ingress帧的MAC有效性。这应该在解密 `header-ciphertext` 和 `frame-ciphertext` 之前完成，以便保证消息的完整性。

# Capability Messaging

初始握手之后的所有消息都与capability相关联，可以在单个 RLPx 连接上同时使用任意数量的capability。capability由简短的 ASCII 名称（最多八个字符）和版本号标识。双方通过hello消息来交换各自支持的capability，因此所有p2p连接必须都支持hello消息。

## 消息编码

Hello消息按以下方式编码：

    frame-data = msg-id || msg-data
    frame-size = len(frame-data),按照24位整数大端编码

其中，`msg-id`是RLP编码的整数，来标识这个消息，`msg-data`是包含消息数据的RLP列表。

所有在hello之后的消息采用snappy算法压缩：

```
frame-data = msg-id || snappyCompress(msg-data)
frame-size = length of frame-data encoded as a 24bit big-endian integer
```

注意压缩过后的消息 `frame-size` 指的是`msg-data`压缩后的大小。因为压缩过后的数据在解压后也许非常大，所以实现应该在解压前检查解压数据大小，[snappy压缩格式的header]([snappy/format_description.txt at master · google/snappy (github.com)](https://github.com/google/snappy/blob/master/format_description.txt))携带原始数据大小。如果传递的未压缩过的数据超过16MB，则直接拒接并关闭网络连接。

## 基于消息ID的多路复用

尽管消息帧层支持`capability-id`，但是当前版本的RLPx不使用此字段来实现多路复用，而是直接使用消息的ID。

每一个capability都会被赋予足够多的消息ID空间，capability应该静态地指定他们需要使用的消息ID数。当双方通过交换Hello消息后都清楚知道他们共同支持的capability（包括版本号），并且能够对消息ID空间的组成达成共识。

消息ID从0x10开始向上紧凑排序(0x00-0x0f为p2p capability保留)，并按照字母顺序给每个共同支持的capability（相同版本，相同名称）分配消息ID。capability名称大小写敏感，双方不共享的capability将被忽略。如果共同支持相同capability（名字相同）的多个版本，则数字最高的获胜，其它的被忽略。

##  p2p Capability

所有的连接都支持p2p capability。在握手完成之后，双方都要么发送Hello消息要么发送Disconnect消息。接收到Hello消息后会话被激活，其它消息才会被继续发送。出于前向兼容性的原因，实现必须忽略协议版本中的任何差异，与较低版本的对等方通信时，实现应尝试模仿该版本。在协议协商后的任何时间，都可能会发送Disconnect消息。

### Hello (0x00)

`[protocolVersion: P, clientId: B, capabilities, listenPort: P, nodeKey: B_64, ...]`

握手完成后双方发送的第一个包，并且此连接过程只会发送一次。只有收到hello消息后，才能发送其它消息。实现必须忽略Hello消息中多余的列表元素，因为它们可能会被未来版本使用。

- `protocolVersion`： `p2p`的能力版本号, **5**；
- `clientId` ：客户端软件识别，为可读性的字符串 (如. "Ethereum(++)/1.0.0")；
- `capabilities`：支持的capability和他们的版本号:`[[cap1, capVersion1], [cap2, capVersion2], ...]`；
- `listenPort`：客户端正在监听的端口 (在当前连接经过的接口上). 0代表客户端没有监听端口；
- `nodeId`： 节点的secp256k1公钥；

### Disconnect (0x01)

`[reason: P]`

通知对方即将断开连接。如果接收到此消息，peer应该立刻断开网络连接。当发送时，一个良好的客户端会给对方2秒的时间来让对方关闭网络然后再关闭对方。

`reason` 可选的整形数来描述断开原因:

| Reason | Meaning                                                      |
| ------ | :----------------------------------------------------------- |
| `0x00` | Disconnect requested                                         |
| `0x01` | TCP sub-system error                                         |
| `0x02` | Breach of protocol, e.g. a malformed message, bad RLP, ...   |
| `0x03` | Useless peer                                                 |
| `0x04` | Too many peers                                               |
| `0x05` | Already connected                                            |
| `0x06` | Incompatible P2P protocol version                            |
| `0x07` | Null node identity received - this is automatically invalid  |
| `0x08` | Client quitting                                              |
| `0x09` | Unexpected identity in handshake                             |
| `0x0a` | Identity is the same as this node (i.e. connected to itself) |
| `0x0b` | Ping timeout                                                 |
| `0x10` | Some other reason specific to a subprotocol                  |

### Ping (0x02)

`[]`

请求对方立刻回复Pong

### Pong (0x03)

`[]`

回应Ping消息



# Change Log

### Known Issues in the current version

- The frame encryption/MAC scheme is considered 'broken' because `aes-secret` and `mac-secret` are reused for both reading and writing. The two sides of a RLPx connection generate two CTR streams from the same key, nonce and IV. If an attacker knows one plaintext, they can decrypt unknown plaintexts of the reused keystream.
- General feedback from reviewers has been that the use of a keccak256 state as a MAC accumulator and the use of AES in the MAC algorithm is an uncommon and overly complex way to perform message authentication but can be considered safe.

### Version 5 (EIP-706, September 2017)

[EIP-706](https://eips.ethereum.org/EIPS/eip-706) 添加了snappy压缩

### Version 4 (EIP-8, December 2015)

[EIP-8](https://eips.ethereum.org/EIPS/eip-8) changed the encoding of `auth-body` and `ack-body` in the initial handshake to RLP, added a version number to the handshake and mandated that implementations should ignore additional list elements in handshake messages and [Hello](https://github.com/ethereum/devp2p/blob/master/rlpx.md#hello-0x00).

# References

- Elaine Barker, Don Johnson, and Miles Smid. NIST Special Publication 800-56A Section 5.8.1, Concatenation Key Derivation Function. 2017.
  URL https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-56ar.pdf
- Victor Shoup. A proposal for an ISO standard for public key encryption, Version 2.1. 2001.
  URL http://www.shoup.net/papers/iso-2_1.pdf
- Mike Belshe and Roberto Peon. SPDY Protocol - Draft 3. 2014.
  URL http://www.chromium.org/spdy/spdy-protocol/spdy-protocol-draft3
- Snappy compressed format description. 2011.
  URL https://github.com/google/snappy/blob/master/format_description.txt

Copyright © 2014 Alex Leverington. [This work is licensed under a Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-nc-sa/4.0/).
