# 在 Haskell 中自下而上重新实现"git clone"

[toc]

为了实施`git clone`，将涵盖以下领域：

- **git 协议**用于使git 客户端能够使用各种传输机制（本机 git 协议、ssh、http）从远程 git 服务器检索所需的更改集，
- **包文件**格式，用于将最少量的数据从服务器传输到客户端（更普遍地用于有效地将打包对象存储在存储库中），
- git 用于以 blob 形式存储提交、树、标签和实际文件内容的底层**对象存储**
- 以及用于跟踪工作目录中文件更改的**索引格式。**



**在最后当我们执行下面语句的时候，就成功实现了clone操作**

```bash
 hgit clone git://github.com/juretta/git-pastiche.git # Where hgit is the name for our custom binary
```



## 克隆过程

该`clone`操作经历了一系列阶段。

它首先使用 git URL 调用命令（有关有效的 URL 格式，请参见[“git clone”手册页中的“GIT URLS”部分）](http://www.kernel.org/pub/software/scm/git/docs/git-clone.html)

```
$> git clone git://host:port/repo_path
```

1. 解析克隆 url 以提取主机、端口和存储库路径信息。
2. 使用原生 git 传输协议通过 TCP 连接到 git 服务器。
3. 协商需要从服务器传输到客户端的对象。这包括接收远程存储库的当前状态（以 ref 广告的形式），其中包括服务器拥有的 ref 以及它指向的每个 ref 的提交哈希。
4. 请求所需的 refs 并接收包含从远程服务器请求的 refs 可访问的所有对象的包文件。
5. 在磁盘上创建一个有效的 git 存储库目录和文件结构。
6. 将对象和引用存储在磁盘上。
7. 使用代表存储库指向的 ref 尖端的文件和目录填充工作目录（考虑到符号链接、权限等）。
8. 创建与该提示和磁盘上的文件相对应的索引文件（暂存区）。

本文将大致遵循这些步骤，因此涵盖以下三个主要部分：

- git 传输和打包线协议
- 包文件格式
- 本地对象存储和暂存区

![Git 克隆概述](images/git-clone-overview@2x.png)



## Git 传输和打包线协议

为了传输存储库数据（提交、标签、文件内容、引用信息），git 客户端和服务器进程协商更新客户端（获取或克隆时）或服务器（推送时）所需的最小数据量。

git 支持四种主要的传输协议来传输存储库数据：如果源可以通过本地文件系统访问，则为本地协议，以及以下三种远程访问协议：SSH、本机 git 协议和 HTTP。

使用 URL 方案的远程传输协议和本地协议共享连接客户端和服务器上正在执行`file://`的各种命令的底层方法。`*-pack`对于获取操作，这意味着在客户端上连接`git fetch-pack`并在`git upload-pack`服务器上运行。对于推送操作，`git send-pack`客户端上的命令将连接到`git receive-pack`服务器上的命令：

![在客户端和服务器上打包命令](images/pack-client-server.png)

注意：对于克隆操作而不是fork`git-fetch-pack`命令，clone`fetch-pack.c#fetch_pack`直接通过`transport.c`层调用函数，避免一次fork操作。

为了更容易测试和调试，重点将只放在*git 协议*上，这是一个简单的、未经身份验证的传输，它在客户端和服务器之间使用全双工通信，服务器通常侦听端口[9418](http://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xml)。

### 传输协议

git 中的存储库数据交换发生在多个阶段（详细信息可以在`Documentation/technical/pack-protocol.txt`作为 git 存储库一部分的文件中找到）：

1. 参考发现
2. 包文件协商
3. 包文件传输

#### 参考发现

参考发现阶段允许客户端检测服务器拥有哪些数据。服务器在 ref 列表中提供此信息，该列表为每个 ref 显示服务器具有（分支和标签）它具有的最近提交。这允许客户端确定它是否已经是最新的，或者它需要什么参考来更新客户端。服务器响应如下所示：

```bash
$> git ls-remote -h -t git://github.com/git/git.git | head -n 10
3a3101c62ecfbde184934f590bab5d84d7ae64a0        refs/heads/maint
21ccebec0dd1d7e624ea2f22af6ac93686daf34f        refs/heads/master
2c8b7bf47c81acd2a76c1f9c3be2a1f102b76d31        refs/heads/next
d17d3d235a4cd1cb1c6840b4a5d99d651c714cc9        refs/heads/pu
5f3c2eaeab02da832953128ae3af52c6ec54d9a1        refs/heads/todo
d5aef6e4d58cfe1549adef5b436f3ace984e8c86        refs/tags/gitgui-0.10.0
3d654be48f65545c4d3e35f5d3bbed5489820930        refs/tags/gitgui-0.10.0^{}
33682a5e98adfd8ba4ce0e21363c443bd273eb77        refs/tags/gitgui-0.10.1
729ffa50f75a025935623bfc58d0932c65f7de2f        refs/tags/gitgui-0.10.1^{}
ca9b793bda20c7d011c96895e9407fac2df9648b        refs/tags/gitgui-0.10.2
```

*注意*：`git ls-remote`可用于列出远程存储库中的引用。这恰好与初始克隆阶段[1](https://stefan.saasen.me/articles/git-clone-in-haskell-from-the-bottom-up/#fn:1)期间发生的 ref 远程查找相同（这称为 ref 广告或引用发现步骤）。

#### 能力

作为参考发现的一部分，客户端和服务器交换有关服务器支持的功能的信息。然后，客户端可以请求某些功能对后续通信有效。

这些功能作为服务器返回的第一个 ref 的一部分进行通信，与`SHA1`,`ref name`对之间用一个`\NUL`字节分隔：

```
3b1031798a00fdf9b574b5857b1721bc4b0e6bac HEAD\0multi_ack thin-pack side-band side-band-64k ofs-delta shallow no-progress include-tag multi_ack_detailed agent=git/1.8.1
```

这些功能在中进行了详细描述，`Documentation/technical/protocol-capabilities.txt`稍后将在它们与实现相关时进行解释。



#### 包文件协商

在引用和能力发现之后，客户端和服务器尝试确定要更新的客户端或服务器所需的最小包文件。

在完整克隆的最简单场景中，客户端请求服务器拥有的所有 ref 和所有必需的对象。对于后续更新（例如`fetch`），客户端不仅定义了它想要的内容，而且告诉服务器它拥有什么 refs，以便服务器可以确定要发送给客户端的最小包文件。



### 包线格式

git 的协议有效负载广泛使用所谓的数据包行（或`pkt-line`技术文档中使用的）格式。A`pkt-line`是一个可变长度的二进制字符串，长度编码在`pkt-line`.

例子：

```
003f3b1031798a00fdf9b574b5857b1721bc4b0e6bac refs/heads/master\n
```

前四个字节`003f`是整个字符串的长度（包括前 4 个长度字节），以十六进制表示（`003f`十六进制 = 63 十进制）。

这可以按如下方式实现（对于狂热的 Haskellers：*我试图避免本文中示例的无点风格*）：

```
-- Create a packet line prefixed with the overall length. Length is 4 byte, hexadecimal, padded with 0
pktLine :: String -> String
pktLine msg = printf "%04s%s" (toHex . (4 +) $ length msg) msg
```

`pkt-line`长度为 0的，即`0000`称为`flush-pkt`。一种特殊情况的数据包线路，用于表示已到达通信交换中商定的切换点。

下表（取自`Documentation/technical/protocol-common.txt`）显示了一些`pkt-line`示例：

| PKT线          | 实际价值   |
| :------------- | :--------- |
| “0006a\n”      | “一个”     |
| “0005a”        | “一个”     |
| “000bfoobar\n” | “foobar\n” |
| “0004”         | “”         |

### 客户端 - 服务器交换

要直观了解 git 协议的工作原理，最好尝试观察客户端和服务器之间的通信。

可以在一个或多个 git 存储库的父目录中使用以下命令启动一个简单的 git 服务器：

```
$> cd /path/to/git/repos # there are 3 git repositories here
$> ls 
git-bottom-up spy zlib
$> git daemon --reuseaddr --verbose  --base-path=. --export-all
[39932] Ready to rumble
```

- `--base-path=.`将使用的克隆尝试映射`git://example.com/hello`到路径`./hello`
- `--export-all`启用从看起来像 git 存储库的所有目录的克隆。否则，在导出任何存储库数据之前`git daemon`验证目录是否具有魔法文件。`git-daemon-export-ok`有关详细信息，请参阅[git-daemon](http://www.kernel.org/pub/software/scm/git/docs/git-daemon.html)。

随着 git 服务器运行并可以访问现有的 git 存储库（应该是没有工作副本的裸存储库），我们现在可以观察客户端和服务器之间的交换。

```
$> export GIT_TRACE_PACKET=1
$> git ls-remote git://127.0.0.1/git-bottom-up
packet:          git> git-upload-pack /git-bottom-up\0host=127.0.0.1\0
packet:          git< 3b1031798a00fdf9b574b5857b1721bc4b0e6bac HEAD\0multi_ack thin-pack side-band side-band-64k ofs-delta shallow no-progress include-tag multi_ack_detailed agent=git/1.8.1
packet:          git< 3b1031798a00fdf9b574b5857b1721bc4b0e6bac refs/heads/master
packet:          git< c4bf7555e2eb4a2b55c7404c742e7e95017ec850 refs/remotes/origin/master
packet:          git< 0000
packet:          git> 0000
3b1031798a00fdf9b574b5857b1721bc4b0e6bac	HEAD
3b1031798a00fdf9b574b5857b1721bc4b0e6bac	refs/heads/master
c4bf7555e2eb4a2b55c7404c742e7e95017ec850	refs/remotes/origin/master
```

Git 有许多调试设置，可以通过设置各种环境变量来启用。设置`GIT_TRACE_PACKET`启用日志输出，其中包含有关客户端和服务器交换的数据包的信息。除了`GIT_TRACE_PACKET`标志之外，以下环境变量对于调试 git 命令很有用：

- `GIT_TRACE_PACKET`- 显示数据包线路信息
- `GIT_TRACE` - 显示一般命令执行调试信息
- `GIT_CURL_VERBOSE`- 使用 http 传输时显示 curl 调试信息（包括 HTTP 标头）
- `GIT_DEBUG_SEND_PACK`- 启用调试输出`upload-pack`
- `GIT_TRANSPORT_HELPER_DEBUG`- 启用[远程助手的调试输出](https://www.kernel.org/pub/software/scm/git/docs/git-remote-helpers.html)

从上面的输出中可以看出，输出显示了解码和剥离长度字段*后*`GIT_TRACE_PACKET`的数据包行。要查看在线上实际交换的内容，有必要使用[Tcpdump](http://www.tcpdump.org/)、[Wireshark](http://www.wireshark.org/)等工具捕获数据包，或者在本例中使用：[ngrep](http://ngrep.sourceforge.net/)：

```
$> sudo ngrep -P "*" -d lo0 -W byline port 9418 and dst host localhost
interface: lo0 (127.0.0.0/255.0.0.0)
filter: (ip) and ( port 9418 and dst host localhost )
#####
T 127.0.0.1:49949 -> 127.0.0.1:9418 [AP]
0032git-upload-pack /git-bottom-up*host=127.0.0.1*
##
T 127.0.0.1:9418 -> 127.0.0.1:49949 [AP]
00ab3b1031798a00fdf9b574b5857b1721bc4b0e6bac HEAD*multi_ack thin-pack side-band side-band-64k ofs-delta shallow no-progress include-tag multi_ack_detailed agent=git/1.8.1

##
T 127.0.0.1:9418 -> 127.0.0.1:49949 [AP]
003f3b1031798a00fdf9b574b5857b1721bc4b0e6bac refs/heads/master

##
T 127.0.0.1:9418 -> 127.0.0.1:49949 [AP]
0048c4bf7555e2eb4a2b55c7404c742e7e95017ec850 refs/remotes/origin/master

##
T 127.0.0.1:9418 -> 127.0.0.1:49949 [AP]
0000
##
T 127.0.0.1:49949 -> 127.0.0.1:9418 [AP]
0000
```

这显示了线路上的实际交换，`*`作为不可打印字符的占位符。*注意：如果您跟随，请更新要监听的接口，`lo0`是 Mac OS X/BSD 上的环回接口，最有可能`lo`在 Linux 上 - 检查`ifconfig`*.

为了模拟有时使调试更容易的低带宽/高延迟场景，我发现使用`dummynet`via`ipfw`作为流量整形工具很有用。

要将 git 本机协议端口的带宽限制为 20KByte/s，请`9418`使用：

```
$> sudo ipfw pipe 1 config bw 20KByte/s
$> sudo ipfw add 1 pipe 1 src-port 9418
```

之后删除带宽限制：

```
$> sudo ipfw delete 1    
```

这对于观察难以捕获的快速本地执行很有用。

有了必要的工具来验证和观察行为，我们现在可以研究实现一个使用 git 传输协议的客户端。

### 实现 ref 发现

为了启动 ref 发现，客户端在端口 9418 上建立到服务器的 TCP 套接字连接，并以数据包行格式发出单个命令。

发现请求的 ABNF 是：

```
git-proto-request = request-command SP pathname NUL [ host-parameter NUL ]
request-command   = "git-upload-pack" / "git-receive-pack" / "git-upload-archive"   ; case sensitive
pathname          = *( %x01-ff ) ; exclude NUL
host-parameter    = "host=" hostname [ ":" port ]
```

数据包行格式的上传包请求示例是：

```
0032git-upload-pack /git-bottom-up\0host=localhost\0
```

这`localhost`是目标系统上的目标主机和`/git-bottom-up`存储库路径。请注意，通过请求在`upload-pack`远程端使用，我们发起了一个`clone/fetch/ls-remote`请求，用于将数据从服务器传输到客户端。

以下示例定义了一个将构造初始请求命令的函数：

```
gitProtoRequest :: String -> String -> String
gitProtoRequest host repo = pktLine $ "git-upload-pack /" ++ repo ++ "\0host="++host++"\0"
```

这允许我们创建一个最小的客户端，实现`ls-remote`如下：

```
data Remote = Remote {
    getHost         :: String
  , getPort         :: Maybe Int
  , getRepository   :: String
} deriving (Eq, Show)

lsRemote' :: Remote -> IO [PacketLine]
lsRemote' Remote{..} = withSocketsDo $
    withConnection getHost (show $ fromMaybe 9418 getPort) $ \sock -> do
        let payload = gitProtoRequest getHost getRepository
        send sock payload
        response <- receive sock
        send sock flushPkt -- Tell the server to disconnect
        return $ parsePacket $ L.fromChunks [response]
```

这个

- 创建到给定的 TCP 连接`Remote`（从 git URL 中提取），
- 以数据包行格式发送协议请求，
- 从套接字读取响应，
- 发送一个刷新数据包 ( `0000`) 以终止连接，
- 将响应解析为`PacketLine`数据结构（这主要是为了稍后轻松提取功能）

这使用以下功能齐全且完整的 git 协议[TCP 客户端](https://bitbucket.org/ssaasen/git-bottom-up/src/master/src/Git/TcpClient.hs)：

```bash
{-# LANGUAGE OverloadedStrings, ScopedTypeVariables, BangPatterns #-}

-- | A git compatible TcpClient that understands the git packet line format.
module Git.Remote.TcpClient (
   withConnection
 , send
 , receiveWithSideband
 , receiveFully
 , receive
) where

import qualified Data.ByteString.Char8 as C
import qualified Data.ByteString as B
import Network.Socket hiding                    (recv, send)
import Network.Socket.ByteString                (recv, sendAll)
import Data.Monoid                              (mempty, mappend)
import Numeric                                  (readHex)

withConnection :: HostName -> ServiceName -> (Socket -> IO b) -> IO b
withConnection host port consumer = do
    sock <- openConnection host port
    r <- consumer sock
    sClose sock
    return r


send :: Socket -> String -> IO ()
send sock msg = sendAll sock $ C.pack msg


-- | Read packet lines.
receive :: Socket -> IO C.ByteString
receive sock = receive' sock mempty
    where receive' s acc = do
            maybeLine <- readPacketLine s
            maybe (return acc) (receive' s . mappend acc) maybeLine

-- =================================================================================

openConnection :: HostName -> ServiceName -> IO Socket
openConnection host port = do
        addrinfos <- getAddrInfo Nothing (Just host) (Just port)
        let serveraddr = head addrinfos
        sock <- socket (addrFamily serveraddr) Stream defaultProtocol
        connect sock (addrAddress serveraddr)
        return sock

-- | Read a git packet line (variable length binary string prefixed with the overall length). 
-- Length is 4 byte, hexadecimal, padded with 0.
readPacketLine :: Socket -> IO (Maybe C.ByteString)
readPacketLine sock = do
        len <- readFully mempty 4
        if C.null len then return Nothing else -- check for a zero length return -> server disconnected
            case readHex $ C.unpack len of
                ((l,_):_) | l > 4 -> do
                     line <- readFully mempty (l-4)
                     return $ Just line
                _                 -> return Nothing
    where readFully acc expected = do
            line <- recv sock expected
            let len  = C.length line
                acc' = acc `mappend` line
                cont = len /= expected && not (C.null line)
            if cont then readFully acc' (expected - len) else return acc'
```

除了通常的设置和连接套接字的仪式外，该`readPacketLine`函数还包含 TcpClient 的 git 特定部分。

第一步是从套接字读取 4 个字节，以确定要读取多少字节才能完全消耗一个数据包行。`readFully`是一个递归函数，用于确保从套接字读取请求的字节数，因为合约`recv`不保证可以一次读取请求的字节数。

根据数据包行的长度，我们消耗数据包行的其余部分或返回`Nothing`。如果长度表明我们收到了一个空包（即`0004`刷新包`0000`），我们停止读取并返回`Nothing`（注意：对于不熟悉 Haskell 的读者，Haskell 的返回与命令式语言中使用的返回完全不同，后者终止执行，在Haskell 它用于将纯值包装在容器中，这里是`IO`monad）。

在服务器返回 ref 通告后，客户端可以通过发送一个刷新数据包来终止连接（`0000`例如，如果客户端已经是最新的）或进入协商阶段，确定从服务器发送到客户端的最佳包文件。

### 实现包文件协商

现在客户端已经了解了服务器发布的所有 refs，它现在可以从服务器请求它需要的 refs。克隆用例是最简单的用例，因为完整克隆只需请求服务器拥有的所有 refs。

因此，一般流程是：

```bash
Client -> Initate proto request
          Ref advertisement                     <- Server
Client -> Negotiation request (list of refs the client wants)
          Send packfile                         <- Server
```

（对于克隆案例）协议请求的简化 ABNF 如下所示：

```
upload-request    =  want-list
        		       flush-pkt
want-list         =  first-want
        		       *additional-want
first-want        =  PKT-LINE("want" SP obj-id SP capability-list LF)
additional-want   =  PKT-LINE("want" SP obj-id LF)
```

客户端列出它想要的所有提交 ID，`want`使用数据包行格式作为前缀。它添加了它希望在第一条想要的行上生效的功能：

```
T(6) 127.0.0.1:55494 -> 127.0.0.1:9418 [AP]
0077want 8c25759f3c2b14e9eab301079c8b505b59b3e1ef multi_ack_detailed side-band-64k thin-pack ofs-delta agent=git/1.8.2
0032want 8c25759f3c2b14e9eab301079c8b505b59b3e1ef
0032want 4574b4c7bb073b6b661abd0558a639f7a32b3f8f
```

协议合约要求必须发送至少一个需要的命令，并且客户端不能请求服务器未公布的提交 ID。

根据客户端想要的 refs，服务器将生成一个包文件，该文件将包含所有必需的 refs 以及可从这些 refs 访问的对象。然后服务器将打包文件发送到客户端存储的临时位置，然后创建本地 git 存储库。

在客户端用来实现克隆操作的以下两个函数中可以很容易地观察到这个流程。实现包文件协商并返回实际的原始包文件（作为严格的`receivePack`字节字符串）和服务器通告的引用列表。该列表稍后将用于在本地存储库中重新创建参考。

```
receivePack :: Remote -> IO ([Ref], B.ByteString)
receivePack Remote{..} = withSocketsDo $
    withConnection getHost (show $ fromMaybe 9418 getPort) $ \sock -> do
        let payload = gitProtoRequest getHost getRepository
        send sock payload
        response <- receive sock
        let pack    = parsePacket $ L.fromChunks [response]
            request = createNegotiationRequest ["multi_ack_detailed",
                        "side-band-64k",
                        "agent=git/1.8.1"] pack ++ flushPkt ++ pktLine "done\n"
        send sock request
        !rawPack <- receiveWithSideband sock (printSideband . C.unpack)
        return (mapMaybe toRef pack, rawPack)
    where printSideband str = do
                        hPutStr stderr str
                        hFlush stderr
```

该`createNegotiationRequest`函数创建`want`客户端发送回服务器的行，用应该有效的功能修改第一行。我们需要过滤服务器发布的 refs。如果远程存储库有任何带注释的标记对象，则 ref 广告将包含标记对象的对象 ID 和标记指向的提交的对象 ID。这称为去皮参考。如果有一个剥离的 ref，它会紧跟标签对象 ref 并有一个`^{}`后缀。例如：

```
1eeeb26fb00aec91b6927cadf2f3f8d0ecacd5a1	refs/tags/v3.2.9.rc3
db1d5f40714a47c58c13ff7d9643e8a0dec6bef8	refs/tags/v3.2.9.rc3^{}
```

我们过滤两个剥离的 refs 并且只包含`refs/heads`and`refs/tags`命名空间中的 refs：

```
-- PKT-LINE("want" SP obj-id SP capability-list LF)
-- PKT-LINE("want" SP obj-id LF)
createNegotiationRequest :: [String] -> [PacketLine] -> String
createNegotiationRequest capabilities = concatMap (++ "") . nub . map (pktLine . (++ "\n")) . foldl' (\acc e -> if null acc then first acc e else additional acc e) [] . wants . filter filterPeeledTags . filter filterRefs
                    where wants              = mapMaybe toObjId
                          first acc obj      = acc ++ ["want " ++ obj ++ " " ++ unwords capabilities]
                          additional acc obj = acc ++ ["want " ++ obj]
                          filterPeeledTags   = not . isSuffixOf "^{}" . C.unpack . ref
                          filterRefs line    = let r = C.unpack $ ref line
                                                   predicates = map ($ r) [isPrefixOf "refs/tags/", isPrefixOf "refs/heads/"]
                                               in or predicates
```

我们想要实现的一项能力是`side-band`能力。这指示服务器发送与包文件本身交错的多路复用进度报告和错误信息（请参阅 参考资料`Documentation/technical/protocol-capabilities.txt`）。`side-band`和`side-band-64k`功能是互斥的，仅在 git 将使用的有效负载数据包的大小上有所不同（在 的情况下为 1000 字节与 65520 字节）`side-band-64k`。

该`receiveWithSideband`函数知道如何解复用包文件响应：

```
receiveWithSideband :: Socket -> (B.ByteString -> IO a) -> IO B.ByteString
receiveWithSideband sock f = recrec mempty
    where recrec acc = do
            !maybeLine <- readPacketLine sock
            let skip = recrec acc
            case maybeLine of
                Just "NAK\n" -> skip -- ignore here...
                Just line -> case B.uncons line of
                                Just (1, rest)  -> recrec (acc `mappend` rest)
                                Just (2, rest)  -> f ("remote: " `C.append` rest) >> skip -- FIXME - scan for linebreaks and prepend "remote: " accordingly (see sideband.c)
                                Just (_, rest)  -> fail $ C.unpack rest
                                Nothing         -> skip
                Nothing   -> return acc
```

它读取数据包行长度之后的第一个字节。边带信道指标是：

- `1`数据包行的其余部分是包文件的一部分 - 这是有效负载通道
- `2`这是服务器发送的进度信息 - 客户端在 STDERR 上打印，前缀为`remote: "`
- `3`这是错误信息，它将导致客户端在 STDERR 上打印出消息并以错误代码退出（在我们的示例中未实现）

这个`clone'`功能：

```
clone' :: GitRepository -> Remote -> IO ()
clone' repo remote@Remote{..} = do
        (refs,packFile) <- receivePack remote
        let dir = pathForPack repo
            -- E.g. in native git this is something like .git/objects/pack/tmp_pack_6bo2La
            tmpPack = dir </> "tmp_pack_incoming"
        _ <- createDirectoryIfMissing True dir
        B.writeFile tmpPack packFile
        _ <- runReaderT (createGitRepositoryFromPackfile tmpPack refs) repo
        removeFile tmpPack
        runReaderT checkoutHead repo
```

- 接收包文件和广告参考列表
- 将包文件写入已知的临时位置
- 解析包文件，然后
- 检查`HEAD`工作目录中的

为了理解和实现最后两个步骤，我们接下来仔细看看包文件格式。

## 打包文件格式

pack 文件用于有效地传输或存储多个 git 对象。当用作存储优化时，该`*.pack`文件附带一个索引文件，允许在包文件中有效查找对象。当用作传输机制时，包文件将按原样传输并在本地创建索引。

打包格式在 git 源代码的[Documentation/technical/pack-format.txt中进行了描述。](https://bitbucket.org/ssaasen/git/src/master/Documentation/technical/pack-format.txt)

包文件包括：

- 一个 12 字节的包

  文件头，包含：

  - 一个 4 字节的魔法字节，其值为`'P' 'A' 'C' 'K'`(decimal `1346454347`)
  - 一个 4 字节的包文件版本
  - 包文件中的 4 字节对象数

- n对象

  - 一个可变长度的对象标头，其中包含对象的类型（见下文）和随后的膨胀/未压缩数据的长度
  - *仅*适用于 deltified 对象：20 字节的基础对象名称（对于 类型的对象`OBJ_REF_DELTA`）或相对于 delta 对象在类型对象的包中的位置的相对（负）偏移量`OBJ_OFS_DELTA`- 见下文）。
  - zlib 压缩/压缩对象数据

- 以上所有内容的20 字节 SHA1**校验和**作为预告片

包文件中的对象是 commit、tag、tree 和 blob 对象。一个 deltified 对象有一个指向基础对象的指针，并且作为它的有效负载，基础版本和该对象的后续版本之间的增量。有关 git 使用的增量编码的讨论，请参见下文。

![打包文件格式](https://stefan.saasen.me/articles/git-clone-in-haskell-from-the-bottom-up/images/packfile@2x.png)

在我们进一步研究包文件格式之前，有必要指出一些工具和命令，这些工具和命令可用于更好地了解对象在包文件中的表示方式。

该`git verify-pack`命令可用于验证包文件。这仅适用于`*.pack`具有随附索引文件的文件。

```
[4888] λ > git verify-pack -v .git/objects/pack/pack-85376214c718d1638a7aa83af4d20302d3fd8efc.pack
2486a54b0fa69143639407f94082cab866a91e08 commit 228 153 12
e8aa4319f3fe937cfb498bf944fa9165078d8245 commit 179 122 165
0376e0620690b259bfc8e381656c07217e4f0b8c tree   317 299 287
d06be33046be894124d2c1d86af7230f17773b3f blob   74 72 586
812387998220a52e6b50cce4a11abc65bbc1ec97 blob   22 29 658
60596d939fad1364fa0b179828b4406761463b8d blob   1466 778 687
42d5f4d92691c3b90b2b66ecb79dfb60773fa1a1 blob   1109 452 1465
99618348415a3a0f78222e54c03c51e638fbad41 blob   466 340 1917 1 42d5f4d92691c3b90b2b66ecb79dfb60773fa1a1
```

非分类对象的条目具有以下格式：

```
SHA1 type size size-in-pack-file offset-in-packfile
```

deltified 对象的条目使用：

```
SHA1 type size size-in-packfile offset-in-packfile depth base-SHA1
```

这在实现读取和重新创建存储在包文件中的对象的代码时会派上用场。

另一个内置命令是`git unpack-objects`，该命令从包文件创建松散的目标文件，并且不需要存在索引文件。因此，它可以用于从还没有索引文件的远程存储库接收到的包文件。此命令需要在现有的 git 存储库中执行，并且打包文件中包含的对象将被解压缩到该`.git/objects`存储库的目录中：

```
git unpack-objects --strict < test-pack.pack
```

在读取和比较包文件数据时，任何合适的 hexeditor（例如`xxd`，`od`或[HexFiend](http://ridiculousfish.com/hexfiend/)，Mac 上的 GUI 应用程序）都是有用的。

*注意：*作为 git clone 过程的一部分，需要读取和理解包文件，以通过比较校验和来验证传入的对象，并为存储在本地存储库中的包生成索引文件。对于这个克隆实现，我将包文件解压缩为松散对象（每个对象一个文件），而不是简化流包文件处理 - 这只是一种快捷方式，因为它不必要地创建了大量的对象文件，这使得克隆大存储库比它需要的慢。

### 打包文件头

读取包文件头很简单。该函数读取 4 个字节*组*中的前 12 个字节（首先使用最*重要*字节`parsePackFile`的大端序*）*。它比较魔术字节以确保我们正在读取 git pack 文件，然后继续读取包文件头定义的对象数量：

```
parsePackFile :: I.Iteratee ByteString IO Packfile
parsePackFile = do
    magic       <- endianRead4 MSB -- 4 bytes, big-endian
    version'    <- endianRead4 MSB
    numObjects' <- endianRead4 MSB
    if packMagic == magic
                then parseObjects version' numObjects'
                else return InvalidPackfile
  where packMagic = fromOctets $ map (fromIntegral . ord) "PACK"
```

### 打包文件对象

包文件对象标头使用[可变长度](http://en.wikipedia.org/wiki/Variable-length_quantity)无符号整数编码的变体，该编码在第一个字节中包含对象类型。

第一个字节包括：

- 最高有效位 (MSB) 确定我们是否需要读取更多字节来获得编码对象大小（如果设置了 MSB，我们将读取下一个字节）

- 对象类型的 3 位（来自

  `cache.h`

  ）：

  - `OBJ_COMMIT`= 1
  - `OBJ_TREE`= 2
  - `OBJ_BLOB`= 3
  - `OBJ_TAG`= 4
  - `OBJ_OFS_DELTA`= 6
  - `OBJ_REF_DELTA`= 7

- 4 位大小（如果设置了 MSB 并且需要读取更多字节，则为部分大小）

第一个字节之后的后续字节包含八位字节最低有效 7 位中总长度的一部分，而 MSB 再次用于指示是否需要读取更多字节。

下图显示了一个由两个八位字节组成的示例：

![打包对象头](https://stefan.saasen.me/articles/git-clone-in-haskell-from-the-bottom-up/images/pack-object-header@2x.png)

读取对象头很简单，我们读取一个字节，从低半字节提取对象类型、初始大小信息以及是否继续读取大小信息通过查看MSB：

```
byte <- I.head -- read 1 byte
let objectType  = byte `shiftR` 4 .&. 7      -- shift right and bitwise AND
                                             -- to mask the bit that was the MSB before shifting
    initialSize = fromIntegral $ byte .&. 15 -- mask type and MSB
-- recursivley read the following bytes if the MSB is set
size <- if isMsbSet byte then parseObjectSize initialSize 0 else return initialSize
```

为了检查是否设置了最高有效位，我们定义了`isMsbSet`我们将在其他地方使用的函数：

```
isMsbSet x = (x .&. 0x80) /= 0  -- 0x80 = 128 decimal
```

以下字节更容易读取，因为我们只关心 7 个最低有效位：

```
parseObjectSize size' iter = do
    nextByte <- I.head
    let add           = (coerce (nextByte .&. 127) :: Int) `shiftL` (4 + (iter * 7)) -- shift depends on the number of iterations
        acc           = size' + fromIntegral add
    if isMsbSet nextByte then
        parseObjectSize acc (iter + 1)
    else
        return acc
    where coerce = toEnum . fromEnum
```

那么总长度为：

```
size0 + size1 + … + sizeN
```

将每个部分`0, 4 + (n-1) * 7`向左移动后。`size0`是最小的，`sizeN`最重要的部分。

在读取对象头的最后一个字节后，我们得到了膨胀/未压缩对象的大小及其类型。

虽然 Haskell 通常非常简洁，但 C 实现无可否认地实现了同样的效果，但要少得多：

```
type = (c >> 4) & 7;
size = (c & 15);
shift = 4;
while (c & 0x80) {
	pack = fill(1);
	c = *pack;
	use(1);
	size += (c & 0x7f) << shift;
	shift += 7;
}  
```

回到我们的 Haskell 实现，这里是读取单个包文件对象的函数的完整定义是（参见[src/Git/Pack/Packfile.hs](https://bitbucket.org/ssaasen/git-in-haskell-from-the-bottom-up/src/master/src/Git/Pack/Packfile.hs)）：

```
parsePackObject :: I.Iteratee ByteString IO (Maybe PackfileObject)
parsePackObject = do
    byte <- I.head -- read 1 byte
    let objectType' = byte `shiftR` 4 .&. 7 -- shift right and masking the 4th least significtan bit
        initial     = fromIntegral $ byte .&. 15
    size' <- if isMsbSet byte then parseObjectSize initial 0 else return initial
    obj <- toPackObjectType objectType'
    !content <- I.joinI $ enumInflate Zlib defaultDecompressParams I.stream2stream
    return $ (\t -> PackfileObject t size' content) <$> obj

-- Map the internal representation of the object type to the PackObjectType
toPackObjectType :: (Show a, Integral a) => a -> I.Iteratee ByteString IO (Maybe PackObjectType)
toPackObjectType 1  = return $ Just OBJ_COMMIT
toPackObjectType 2  = return $ Just OBJ_TREE
toPackObjectType 3  = return $ Just OBJ_BLOB
toPackObjectType 4  = return $ Just OBJ_TAG
toPackObjectType 6  = do
    offset <- readOffset 0 0
    return $ Just (OBJ_OFS_DELTA offset)
toPackObjectType 7  = do 
    baseObj <- replicateM 20 I.head -- 20-byte base object name SHA1
    return $ Just (OBJ_REF_DELTA baseObj)
toPackObjectType _  = return Nothing
```

根据我们的类型

- 当对象是a `commit`、`tag`或`tree``blob`
- 如果这是一个 detified 对象（即当对象类型为`6`or时`7`），则读取基本对象 id 或偏移量，然后读取压缩的增量数据

一个有趣的挑战是包文件对象头包含*未压缩*对象的大小，但不包含压缩对象的*大小*。为了读取对象，识别对象边界（即下一个对象在包文件中从哪里开始）的唯一方法是实际膨胀 zlib 压缩数据，从而消耗作为 zlib 压缩内容一部分的以下字节。虽然 C 中的 zlib API（即`inflate()`）指示它是否已到达压缩数据的末尾并产生了所有未压缩的输出（参见http://www.zlib.net/zlib_how.html），但我只找到了 Iteratee基于实现（[迭代压缩](http://hackage.haskell.org/package/iteratee-compress)) 以适合在 Haskell 中实现相同的目标。该`parsePackObject`函数的以下摘录从包文件流中对 zlib 压缩数据进行膨胀，并`PackfileObject`使用对象类型信息、包中的大小信息和膨胀数据创建一个：

```
    !content <- I.joinI $ enumInflate Zlib defaultDecompressParams I.stream2stream
    return $ (\t -> PackfileObject t size' content) <$> obj
```

这允许`Git.Pack.Packfile`模块完全读取包文件并创建一个内部包文件表示，其中包含`PackfileObject`具有完全膨胀内容的 s 列表。

在将这个包文件写入磁盘之前，我们需要处理 deltified 对象，因为我们的内部表示包含 deltified 和 undeltified（即完整）对象的混合。

### 增量编码

packfile 中的 deltified 表示（包文件对象类型 6 和 7）使用 delta 压缩来最小化需要传输和/或存储的数据量。Deltification*只*发生在包文件中。

以下（来自[“文件系统对 Delta 压缩的支持”](http://mail.xmailserver.net/xdfs.pdf)）给出了 delta 压缩的一般定义：

> 增量压缩包括将目标版本的内容表示为一些现有源内容的变异（增量），以实现相同的目标，即空间或时间的减少。通常，目标和源是相关的文件版本并且具有相似的内容。

因此，增量编码允许基于原始源文件和一个或多个增量文件重新创建文件版本。后续的 delta 文件可以应用于前一个 delta 文件及其源的修补结果（在 git 中称为“delta 链” - 该命令将在使用标志`git verify-pack`调用时打印 delta 链长度的直方图）。`--verbose`增量编码的一个重要属性是它可以用于**二进制文件和文本**文件。

git 中使用的 delta 压缩算法最初基于[xdelta](http://xdelta.org/)和[LibXDiff](http://www.xmailserver.org/xdiff-lib.html)，但针对 git 用例进行了进一步简化（请参阅 git 邮件列表上的[“diff'ing files”](http://git.661346.n2.nabble.com/diff-ing-files-td6446460.html)线程）。以下讨论基于 git 使用的 delta 文件格式`patch-delta.c`以及`diff-delta.c`来自 git 源的文件。

#### 增量表示的格式

git delta 编码算法是一种`copy/insert`基于算法（这在 中很明显`patch-delta.c`）。delta 表示包含一个 delta 标头和一系列用于`copy`or`insert`指令的操作码。

来自[“增量压缩的文件系统支持”](http://mail.xmailserver.net/xdfs.pdf)：

> delta 算法的复制/插入类使用字符串匹配技术来定位源版本和目标版本中的匹配偏移量，然后为每个匹配范围发出一系列复制指令，并插入指令以覆盖不匹配的区域

指令包含到源缓冲区的`copy`偏移量以及从该偏移量开始从源缓冲区复制到目标缓冲区的字节数。`insert`操作码本身是从增量缓冲区复制到目标的字节数。这将包含已添加且此时不属于源缓冲区的字节。

增量缓冲区以包含源缓冲区和目标缓冲区长度的标头开始，以便能够验证恢复/修补的结果。长度再次被编码为可变长度整数，其中 MSB 指示是否有另一个可变长度八位字节。

增量缓冲区布局：

```
| Varint - Lengt of the source/base buffer | 
| Varint - Length of the target buffer     |
| n copy/insert instructions               |
```

#### 测试

为了测试 delta 编码实现，git 源允许我们构建一个`test-delta`二进制文件，该二进制文件可用于生成 delta 数据并从源和 delta 表示恢复目标文件。

```
$> cd ~/dev/git/git-source
$> make configure
$> ./configure
$> make test-delta
```

这将生成一个`test-delta`带有*delta*和补丁*模式*的二进制文件：

```
[4926] λ > ./test-delta
usage: test-delta (-d|-p) <from_file> <data_file> <out_file>
```

生成增量文件：

```
./test-delta -d test-delta.c test-delta-new.c out.delta
```

根据增量恢复原始文件：

```
./test-delta -p test-delta.c out.delta restored-test-delta-new.c
```

验证两者实际上是否相同：

```
diff -q restored-test-delta-new.c test-delta-new.c
```

然后可以使用此工作流生成任意增量文件并测试我们自己的实现。



#### 补丁算法示例

在源文件（`zlib.c`来自 git 源的文件）中，函数定义在同一个文件中向下移动，并添加了新注释。

![差异](images/delta-changes@2x.png)

使用`test-delta`我们可以使用该文件的旧版本和新版本生成增量文件：

```
./test-delta -d zlib.c zlib-changed.c zlib-delta
```

这是生成的增量文件：

```
[4950] λ > xxd -b zlib-delta
0000000: 10010001 00101110 10101100 00101110 10110000 11010001  ......
0000006: 00000001 00010111 00101111 00100000 01010100 01101000  ../ Th
000000c: 01101001 01110011 00100000 01101001 01110011 00100000  is is
0000012: 01100001 00100000 01101110 01100101 01110111 00100000  a new
0000018: 01100011 01101111 01101101 01101101 01100101 01101110  commen
000001e: 01110100 10110011 11001110 00000001 00100111 00000001  t...'.
0000024: 10110011 01011111 00000011 01101100 00010000 10010011  ._.l..
000002a: 11110101 00000010 01101011 10110011 11001011 00010011  ..k...
0000030: 01000110 00000011
```

第一步是读取源和目标长度。设置第一个字节的 MSB 的事实表明我们需要读取下一个字节以获取大小信息，因此前两个字节构成了源缓冲区的长度：

```
10010001 00101110 
```

读取可变长度整数简单地归结为：

```
1. Mask the MSB                  10010001 & 127 -> 00010001 = 17
2. Left shift the 2nd byte       00101110 << 7              = 5888
    by (iteration * 7). As this is the first additional byte this is (1 * 7)   
3. Bitwise OR 1st and 2nd byte   17 | 5888                  = 5905
```

这与我们在读取包文件中的对象头时看到的方法相同。我们可以检查我们读取的长度实际上是正确的：

```
[4868] λ > wc -c zlib.c
    5905 zlib.c
```

接下来的两个字节是目标缓冲区的长度（即 5932 字节）。

实现这一点的重要部分是：

```
where   decodeSize offset = do
            skip offset
            byte <- getWord8
            next (maskMsb byte) 7 byte $ succ offset
        next base shift byte' count | isMsbSet byte' = do
             b <- getWord8
             let len = base .|. ((maskMsb b) `shiftL` shift)
             next len (shift + 7) b $ succ count
        next finalLen _ _ count                  = return (finalLen, count)
        maskMsb byte                             = fromIntegral $ byte .&. 0x7f
```

where`decodeSize`将使用源的长度信息的偏移量（在这种情况下为 0）和目标大小（表示源大小所需的字节数）调用。

读取头信息后，剩下的字节是要么`copy`或`insert`指令。下一个字节 ( `10110000`) 是`copy`基于 MSB 已设置这一事实的指令。然后可以如下提取源缓冲区的偏移量和要复制的字节数。

从 LSB 开始：

```
10110000 & 0x01 - 1st bit not set
10110000 & 0x02 - 2nd bit not set
10110000 & 0x04 - 3rd bit not set
10110000 & 0x08 - 4th bit not set
```

没有设置任何偏移位，我们不读取任何偏移值，因此偏移量为 0。这意味着我们从源缓冲区的开头进行复制。

```
10110000 & 0x10 - 5th bit is set. We read the next byte (11010001)
10110000 & 0x20 - 6th bit is set. We read the next byte (00000001), left
        shift it by 8 and OR it to the previously read value:
        
        11010001 | (00000001 << 8) = 209 | 256 = 465
        
00000000 & 0x40 - 7th bit is not set.
```

`465`是从源缓冲区复制到目标缓冲区的字节数，从偏移量 0 开始。

下一个字节`00010111`（字节 8）是插入指令（未设置 MSB）。插入指令只是从增量复制到目标缓冲区的字节数。

*注意*：不同数字表示之间的转换可以在 shell 中快速完成，使用：

```
$> echo $(( 16#A4 ))      # convert hexadecimal into decimal  
164
$> echo $(( 2#00010111 )) # convert binary into decimal
23
```

在这种情况下，我们将 delta 中的 23 个字节复制到目标中。这些是：

```
$> < zlib-delta tail -c +9 | head -c 23
/ This is a new comment
```

完整的复制/插入指令集是：

1. 将 465 个字节从源复制到目标，从偏移量 0 开始
2. 将*增量*缓冲区中的 23 个字节插入目标
3. 复制 295 个字节，从偏移量 462 开始
4. 复制 4204 字节，从偏移量 863 开始
5. 复制 107 个字节，从偏移量 757 开始
6. 复制 838 字节，从偏移量 5067 开始

这可以很容易地手动验证：

```
head -c 465 zlib.c >> manual-target-zlib.c
< zlib-delta tail -c +9 | head -c 23    >> manual-target-zlib.c
< zlib.c tail -c +463   | head -c 295   >> manual-target-zlib.c
< zlib.c tail -c +864   | head -c 4204  >> manual-target-zlib.c
< zlib.c tail -c +758   | head -c 107   >> manual-target-zlib.c
< zlib.c tail -c +5068  | head -c 838   >> manual-target-zlib.c 
```

这会产生相同的恢复文件：

```
[4905] λ > diff manual-target-zlib.c zlib-changed.c && echo $?
0
```

查看测试文件的大小（以字节为单位），我们可以看到，通过使用 delta 编码，可以显着减少相似文件的空间需求：

```
[4908] λ > wc -c zlib-delta zlib.c zlib-changed.c
  50 zlib-delta
5905 zlib.c
5932 zlib-changed.c
```

与独立存储两个版本相比，基本版本为 5905 + 增量为 50 个字节 ()5905 + 5932)。

Git 通过将压缩的基本和增量对象 zlib 存储在包文件中，进一步降低了空间需求。

#### delta编码算法的实现

对于克隆实现，我们只需要处理更简单的实现补丁操作，即根据源和增量重新创建目标内容。幸运的是，这比基于源文件和目标文件创建合适的 delta 文件的任务要容易得多（有关此问题的进一步讨论，请参见[2 ）。](https://stefan.saasen.me/articles/git-clone-in-haskell-from-the-bottom-up/#fn:2)

我们的`Git.Pack.Delta`模块公开的主要函数是`patch`接受源和增量字节串（字节串是 Haskell 版本的字节向量/数组）并返回重新创建的目标字节串的函数。

```
patch :: B.ByteString -- ^ Source/Base
      -> B.ByteString -- ^ Delta
      -> Either String B.ByteString
patch base delta = do
        header <- decodeDeltaHeader delta
        if B.length base == sourceLength header then
            fst $ runGet (run (getOffset header) base delta) delta
        else Left "Source length check failed"
```

跳过上面提到的标头解析，主要构建块是：

```
-- | Parse the delta file and transform the source into the target ByteString
run :: Int -> B.ByteString -> B.ByteString -> Get B.ByteString
run offset source delta = do
    skip offset
    cmd <- getWord8
    runCommand cmd B.empty source de       
```

我们根据从增量开始到增量有效负载的偏移量跳过标头。`runCommand`我们读取第一个字节，它是操作码，并使用以下函数执行复制或插入指令：

```
-- | Execute the @copy/insert@ instructions defined in the delta buffer to
-- restore the target buffer
runCommand :: Word8 -> B.ByteString -> B.ByteString -> t -> Get B.ByteString
runCommand cmd acc source delta = do
    result <- choose cmd
    finished <- isEmpty
    let acc' = B.append acc result
    if finished then return acc'
       else do
        cmd' <- getWord8
        runCommand cmd' acc' source delta
  where choose opcode | isMsbSet opcode = copyCommand opcode source
        choose opcode                   = insertCommand opcode
```

如果设置了最高有效字节，则为副本，否则为插入指令。如果它是一个插入命令，则命令本身就是从增量复制到目标缓冲区的字节数，因此我们只需应用该`insertCommand`函数。

```
-- | Read @n@ bytes from the delta and insert them into the target buffer
insertCommand :: Integral a => a -> Get B.ByteString
insertCommand = getByteString . fromIntegral
```

复制指令稍微复杂一些，并满足更大的偏移量（源中的偏移量）和大小（要复制的字节数）长度（使用`readCopyInstruction`函数解决）：

```
-- | Copy from the source into the target buffer
copyCommand :: Word8 -> B.ByteString -> Get B.ByteString
copyCommand opcode source = do
        (offset, len) <- readCopyInstruction opcode
        return $ copy len offset source
    where copy len' offset'             = B.take len' . B.drop offset'  	

readCopyInstruction :: (Integral a) => Word8 -> Get (a, a)
readCopyInstruction opcode = do
        -- off -> offset in the source buffer where the copy will start
        -- this will read the correct subsequent bytes and shift them based on
        -- the set bit
        offset <- foldM readIfBitSet 0 $ zip [0x01, 0x02, 0x04, 0x08] [0,8..]
        -- bytes to copy
        len'   <- foldM readIfBitSet 0 $ zip [0x10, 0x20, 0x40] [0,8..]
        let len = if coerce len' == 0 then 0x10000 else len'
        -- FIXME add guard condition from `patch-delta.c`: if (unsigned_add_overflows(cp_off, cp_size) || ...
        return $ (coerce offset, coerce len)
    where calculateVal off shift           = if shift /= 0 then (\x -> off .|. (x `shiftL` shift)::Int) . fromIntegral else fromIntegral
          readIfBitSet off (test, shift)   = if opcode .&. test /= 0 then liftM (calculateVal off shift) getWord8 else return off
          coerce                           = toEnum . fromEnum
```

要测试 delta 实现，我们可以使用以下简单`main`函数和`test-delta`git 源中的命令生成的 delta 文件：

```
main :: IO ()
main = do
    (sourceFile:deltaFile:_) <- getArgs
    source <- B.readFile sourceFile
    delta <- B.readFile deltaFile
    header <- decodeDeltaHeader delta
    print header
    print $ B.length source
    either putStrLn (B.writeFile "target.file") $ patch source delta
```

使用原始源和增量，这将创建一个补丁`target.file`：

```
$> runhaskell -isrc src/Git/Pack/Delta.hs zlib.c zlib-delta
DeltaHeader {sourceLength = 5905, targetLength = 5932, getOffset = 4}
5905
```

使用 at working`patch`函数，我们现在可以根据包含在包文件中的 deltified 和基本对象重新创建实际内容。

### Ref 与 Ofs 增量

包文件格式定义了两种不同类型的 deltified 对象：`OBJ_OFS_DELTA`和`OBJ_REF_DELTA`. 它们仅在包文件中标识基础（或源）对象的方式上有所不同。`OBJ_REF_DELTA`使用标识对象的 20 字节 SHA1，而`OBJ_OFS_DELTA`使用包文件中 delta 对象头的负偏移量（如上面的包文件部分所述）。最初添加delta编码时，git以基于ref的delta开始，`OBJ_OFS_DELTA`后来引入了object类型，`#eb32d236`主要是为了减小pack文件的大小。客户端是否支持在包文件中基于偏移量的增量可以在包文件协商期间通过设置`ofs-delta`功能（如果服务器指示支持）来发出信号。

来自`Documentation/technical/protocol-capabilities.txt`：

> ofs-delta 服务器可以发送，并且客户端可以理解 PACKv2，其中 delta 通过包中的位置而不是 obj-id 来引用其基础。也就是说，他们可以在包文件中发送/读取 OBJ_OFS_DELTA（又名类型 6）。

作为克隆实现的简化，我们*不在*请求中使用此功能，因此只需要实现基于 ref 的基对象的查找。

### 概括

我们现在有

- 一种将包文件读入`PackObject`包含膨胀对象的 s列表的方法
- `patch`基于基础对象和增量内容重新创建对象的函数。

因此，给定我们收到的包文件，我们可以完全重新创建对象。

## Git 存储库

要创建一个有效的 git 存储库，我们需要创建一个目录布局和一组文件，git 需要将这个目录识别为一个有效的 git 存储库。从概念上讲，文件和目录可以大致分为对象存储区、引用区和索引区。

![Git 存储库布局和对象](https://stefan.saasen.me/articles/git-clone-in-haskell-from-the-bottom-up/images/git-repository.png)

[Git Repository Layout 手册页](https://www.kernel.org/pub/software/scm/git/docs/gitrepository-layout.html)包含有关 git 目录内容的详细信息。

## 对象存储

要正确填充 git 对象存储，我们需要了解 git 支持的对象类型、它如何表示这些对象以及它们如何存储在磁盘上。

Git 对对象有两种不同的表示形式。“松散对象”格式将每个对象存储在`.git/objects/`目录中的单独文件中。Git 定期将松散的对象打包到我们上面讨论的打包文件中，利用增量压缩来减少类似文件的空间需求。松散对象使用 SHA1 哈希作为文件名存储，其中前两个字母数字字符用作目录名`.git/objects/`（这导致对象目录的简单 16*16 分区），其余 38 个字符作为实际文件名。下面的例子显示了松散的对象和一个带有索引文件的包文件：

```
[4866] λ > tree .git/objects/
.git/objects/
├── 08
│   └── 24d8f1ed19e4e07cf03e40aeebe07b95a68f7d
├── 61
│   └── 3956b77de7b48bdd82375716c1f1b78fd30764
├── d4
│   └── d697777ba37a1588269b2639fb93d14af8e781
├── fc
│   └── f5367cdfdc59e08428afa5a0d62893bcca0cf0
├── info
│   └── packs
└── pack
    ├── pack-5faf642231915b153fd273701866c5526c680bc6.idx
    └── pack-5faf642231915b153fd273701866c5526c680bc6.pack
```

可以使用以下方式查询对象存储`git cat-file`：

```
[4869] λ > git cat-file -t fcf5367cdfdc59e08428afa5a0d62893bcca0cf0
tree
[4870] λ > git cat-file -p fcf5367cdfdc59e08428afa5a0d62893bcca0cf0
100644 blob 613956b77de7b48bdd82375716c1f1b78fd30764	README.md
040000 tree e9378f166d4ddbf93a4bc1c91af2d9f3ea34ebdd	_src
040000 tree 2dba9669371668a7030d79e66521742660df9818	images
```

### 磁盘上的对象

git 处理常用操作的 4 种对象类型是：

- 存储提交消息、日期和作者`commit`/提交信息，并指向单个树对象。一个提交可以有零个（对于根提交），一个或多个父提交 ID。指向父级的提交指针形成提交图。
- A存储*没有*任何元信息（例如文件名）`blob`的实际文件内容。
- A`tree`包含路径名和权限以及指向 blob 或树对象的指针 - 它表示跟踪内容的目录和文件名。
- 带注释的`tag`标签存储标签消息、日期和创建标签的人的身份。Git 还使用所谓[的轻量级](http://www.kernel.org/pub/software/scm/git/docs/git-tag.html)标签，它们只是引用层次结构中的指针，没有任何额外的标签元信息。

有关更多详细信息，请参阅http://git-scm.com/book/en/Git-Internals-Git-Objects。

如前所述，git 存储单个包文件并生成随附的索引文件，而不会在初始克隆期间创建任何松散对象。我们没有使用相同的方法（并且因为我们已经读取了打包文件），而是简单地解包对象并将它们存储为松散对象。这显然是一种效率较低的方法，并且使我们的克隆操作比需要的慢得多。

由于我们的内部包文件表示已经包含解压缩的对象，我们只需要正确存储它们（重新创建 detified 对象），而不需要提供从头开始*创建每个对象类型的功能。*

#### 对象存储格式

存储在磁盘上的对象内容使用以下格式：

来自http://git-scm.com/book/en/Git-Internals-Git-Objects

> Git 构造一个以对象类型开头的标头，在本例中为 blob。然后，它添加一个空格，后跟内容的大小，最后是一个空字节
>
> Git 将标题和原始内容连接起来，然后计算新内容的 SHA-1 校验和。

返回对象（文件）内容的`encodeObject`正确磁盘表示。给定包文件中的未压缩内容作为字节串，它返回一对 SHA1 哈希和具有正确标头的磁盘内容表示：

```
-- header: "type size\0"
-- sha1 $ header ++ content
encodeObject :: ObjectType -> C.ByteString -> (ObjectId, C.ByteString)
encodeObject objectType content = do
    let header       = headerForBlob (C.pack $ show objectType)
        blob         = header `C.append` content
        sha1         = hsh blob
    (sha1, blob)
    where headerForBlob objType = objType `C.append` " " `C.append` C.pack (show $ C.length content) `C.append` "\0"
          hsh = toHex . SHA1.hash
```

`zlib`Git 使用压缩将对象存储在磁盘上。该`writeObject`函数使用我们的编码函数存储任何对象以创建对象内容：

```
writeObject :: GitRepository -> ObjectType -> C.ByteString -> IO FilePath
writeObject GitRepository{..} objectType content = do
    let (sha1, blob) = encodeObject objectType content
        (path, name) = pathForObject getName sha1
        filename     = path </> name
    _ <- createDirectoryIfMissing True path
    L.writeFile filename $ compress blob
    return filename
    where compress data' = Z.compress $ L.fromChunks [data']
   
    
-- Partition the namespace -> (2 chars,38 chars)
pathForObject :: String -> String -> (FilePath, String)
pathForObject repoName sha | length sha == 40 = (repoName </> ".git" </> "objects" </> pre, rest)
    where pre  = take 2 sha
          rest = drop 2 sha
pathForObject _ _                             = ("", "")
```

使用这个函数，我们现在可以使用以下函数解压打包文件：

```
unpackPackfile :: Packfile -> WithRepository ()
unpackPackfile InvalidPackfile = error "Attempting to unpack an invalid packfile"
unpackPackfile (Packfile _ _ objs) = do
        repo <- ask
        unresolvedObjects <- writeObjects objs
        liftIO $ forM_ unresolvedObjects $ writeDelta repo
    where   writeObjects (x@(PackfileObject (OBJ_REF_DELTA _) _ _):xs) = liftM (x:) (writeObjects xs)
            writeObjects (PackfileObject objType _ content : xs) = do
                repo <- ask
                _ <- liftIO $ writeObject repo (tt objType) content
                writeObjects xs
            writeObjects []     = return []

            tt OBJ_COMMIT       = BCommit
            tt OBJ_TREE         = BTree
            tt OBJ_BLOB         = BBlob
            tt OBJ_TAG          = BTag
            tt _                = error "Unexpected blob type"

            writeDelta repo (PackfileObject ty@(OBJ_REF_DELTA _) _ content) = do
                    base <- case toObjectId ty of
                        Just sha -> liftIO $ readObject repo sha
                        _        -> return Nothing
                    if isJust base then
                        case patch (getBlobContent $ fromJust base) content of
                            Right target -> do
                                            let base'        = fromJust base
                                            filename <- writeObject repo (objType base') target
                                            return $ Just filename
                            Left _       -> return Nothing
                    else return Nothing -- FIXME - base object doesn't exist yet
            writeDelta _repo _ = error "Don't expect a resolved object here"
```

`unpackPackfile`使用 2-pass 方法。它首先直接写出所有未分类的对象，并累积一个未解析的分类对象列表。然后，它将`writeDelta`函数应用于每个 deltified 对象，这些对象查找我们刚刚存储的基本对象，并通过使用 base 和 delta 内容应用 patch 函数来重新创建 undeltified 对象。该`readObject`函数与该函数相反，它`writeObject`知道如何从本地 repo 中读取对象。

这组函数允许我们将包文件中包含的所有对象（包括 deltified 和 undeltified）解包到本地`.git/objects`对象存储中。

## 参考文献

有了对象，git 现在需要一个进入提交图的入口点。分支和标签是那些入口点，分支和标签的提示称为**refs**，这些名称引用 git 存储库中对象的基于 SHA1 的对象 ID。refs 存储在`$GIT_DIR/refs`目录下（例如`.git/refs/heads/master`，或以打包格式存储在 下`.git/packed-refs`）。Refs 直接包含 40 个十六进制数字 SHA1 或对另一个 ref 的符号 ref（例如`ref: refs/heads/master`）。特殊符号 ref`HEAD`指的是当前分支。

git 存储库中的示例 ref 结构是：

```
.git/refs/
├── heads
│   └── master
├── remotes
│   └── origin
│       ├── HEAD
│       ├── master
│       └── pu
└── tags
    ├── 0.9
    └── 1.0
```

目录中的 refs`refs/heads`是存储库的本地分支。该`refs/tags`目录包含两个标签。在带注释的标签的情况下，标签指向标签对象。对于轻量级标记，标记文件包含标记提交本身的对象 ID。该`refs/remotes`目录包含每个配置的远程的子目录（在本例中为默认`origin`）。在示例中，上游存储库有两个分支`master`，`pu`并且 symref`HEAD`标识上游存储库的默认分支。

通过列出分支`git branch`显示相同的结果：

```
$> git branch
* master
$> git branch --remotes
  origin/HEAD
  origin/master
  origin/pu
```

因此，每个 ref 名称都映射到给定目录中具有相同名称的文件，其内容只是它引用的 SHA1，后跟一个新行：

```
$> cat .git/refs/remotes/origin/master
8c25759f3c2b14e9eab301079c8b505b59b3e1ef
```

规范的 git 实现使用优化并将 refs 以打包的形式存储在`$GIT_DIR/packed-refs`文件中。该文件类似于 ref 广告输出，并将 ref 映射到其在该单个文件中的 object-id：

```
$> cat .git/packed-refs
# pack-refs with: peeled
1865311797f9884ec438994d002b33f05e2f4844 refs/heads/delta-encoding-ffi
6bc699aad89341be9d07293815d0fa14f2e162ab refs/heads/fake-ref-creation
c666c23749af1e86169bed8ee0d1a2ac598e6ab0 refs/heads/master
496bf4f0724dd411855b374255b825f9b66cbfd0 refs/heads/sideband
```

*注意：*为了简化我们的实现，我们忽略了这个优化，并再次将每个 ref 存储在一个单独的文件中。

### 执行

在`createGitRepositoryFromPackfile`我们从函数中调用的`clone'`函数中，我们可以观察到创建工作 git 存储库所需的基本步骤：

```
createGitRepositoryFromPackfile :: FilePath -> [Ref] -> WithRepository ()
createGitRepositoryFromPackfile packFile refs = do
    pack <- liftIO $ packRead packFile
    unpackPackfile pack
    createRefs refs
    updateHead refs
```

我们解压缩包文件并在`.git/objects`目录中创建对象（值得重复 - 这*不是*本机 git 客户端的工作方式 - 本机客户端创建一个索引文件并使用包文件代替），然后我们创建 refs 最后是特殊的符号参考`HEAD`。

需要创建的 ref 从初始 ref 广告中已知，并且只是从对象 id（在本例中为提交 id）到完整 ref 名称的映射。

例如：

```
21ccebec0dd1d7e624ea2f22af6ac93686daf34f        refs/heads/master
2c8b7bf47c81acd2a76c1f9c3be2a1f102b76d31        refs/heads/next
```

我们使用`Ref`数据类型来对这对进行建模，并使用`Refs`初始 ref 广告中的这个列表来创建正确的 ref 文件：

```
data Ref = Ref {
    getObjId        :: C.ByteString
  , getRefName      :: C.ByteString
} deriving (Show, Eq)

createRefs :: [Ref] -> WithRepository ()
createRefs refs = do
    let (tags, branches) = partition isTag $ filter (not . isPeeledTag) refs
    writeRefs "refs/remotes/origin" branches
    writeRefs "refs/tags" tags
    where simpleRefName  = head . reverse . C.split '/'
          isPeeledTag    = C.isSuffixOf "^{}" . getRefName
          isTag          = (\e -> (not . C.isSuffixOf "^{}" $ getRefName e) && (C.isPrefixOf "refs/tags" $ getRefName e))
          writeRefs refSpace     = mapM_ (\Ref{..} -> createRef (refSpace ++ "/" ++ (C.unpack . simpleRefName $ getRefName)) (C.unpack getObjId)) 

createRef :: String -> String -> WithRepository ()
createRef ref sha = do
    repo <- ask
    let (path, name) = splitFileName ref
        dir          = getGitDirectory repo </> path
    _ <- liftIO $ createDirectoryIfMissing True dir
    liftIO $ writeFile (dir </> name) (sha ++ "\n")
```

我们再次过滤掉剥离的标签引用并将引用划分为标签和分支，这些标签和分支存储在各自的目录中。

*注意：*虽然`origin`是克隆来源的远程存储库的默认名称，但本机`git clone`命令可以选择`--origin <name>`将远程名称设置为`origin`克隆时以外的名称。由于我们的克隆命令目前不支持任何选项，我们只需`origin`在实现中使用默认远程名称。

创建 refs 后，符号 ref`HEAD`被创建，它指向上游存储库用作默认分支的相同 ref（通过其`HEAD`symref）。

```
updateHead :: [Ref] -> WithRepository ()
updateHead [] = fail "Unexpected invalid packfile"
updateHead refs = do
    let maybeHead = findHead refs
    unless (isNothing maybeHead) $
        let sha1 = C.unpack $ getObjId $ fromJust maybeHead
            ref = maybe "refs/heads/master" (C.unpack . getRefName) $ findRef sha1 refs
            in
            do
                createRef ref sha1
                createSymRef "HEAD" ref
    where isCommit ob = objectType ob == OBJ_COMMIT
          findHead = find (\Ref{..} -> "HEAD" == getRefName)
          findRef sha = find (\Ref{..} -> ("HEAD" /= getRefName && sha == (C.unpack getObjId)))
```

该`updateHead`函数尝试解析上游`HEAD`ref 的 commit-id，然后查找与该 object-id 对应的 ref 名称，以便使用该`createSymRef`函数创建 symref：

```
createSymRef :: String -> String -> WithRepository ()
createSymRef symName ref = do
        repo <- ask
        liftIO $ writeFile (getGitDirectory repo </> symName) $ "ref: " ++ ref ++ "\n"
```

symref`HEAD`然后看起来类似于：

```
$> cat .git/HEAD
ref: refs/heads/master
```

此时本地 git 存储库实际上是可用的（例如，类似`git log`或`git checkout`工作的命令），尽管有一个空的工作副本。

## 工作副本和索引

填充对象存储并且所有引用都到位后，下一步是“签出”与存储库快照`HEAD`指向的文件匹配的文件。

为了检查电流`HEAD`，我们需要：

- 阅读`HEAD`symref 并解析它最终指向的提交，以用作我们签出的工作副本的提示。
- 鉴于此提交，我们需要解析我们的提交指向的顶级树。
- 基于树，我们写出顶层树包含的所有 blob，并递归读取子树条目以完全恢复工作副本中的目录和文件。

这要求我们能够从对象存储中检索对象，然后解析每个对象类型并创建适当的内存表示以启用树遍历并最终正确创建文件和目录。

### 读取对象

读取对象是一个两步过程。第一步，从文件系统中检索已经涵盖并由`readObject`函数实现：

```
readObject :: GitRepository -> ObjectId -> IO (Maybe Object)
readObject GitRepository{..} sha = do
    let (path, name) = pathForObject getName sha
        filename     = path </> name
    exists <- doesFileExist filename
    if exists then do
        bs <- C.readFile filename
        return $ parseObject sha $ inflate bs
    else return Nothing
    where inflate blob = B.concat . L.toChunks . Z.decompress $ L.fromChunks [blob]
```

`readObject`从给定 SHA1 的文件系统中查找正确的文件并解压缩内容。正如上面已经提到的，内容的前缀是一个包含对象类型和对象总体大小的标题，由一个`\NUL`字节与实际对象内容分隔：

```
object-type SP size \NUL object-content
```

该`parseObject`函数解析文件内容并创建`Object`数据类型的实例，提取对象类型和对象内容：

```
data Object = Object {
    getBlobContent  :: B.ByteString
  , objType         :: ObjectType
  , sha             :: ObjectId
} deriving (Eq, Show)

parseObject :: ObjectId -> C.ByteString -> Maybe Object
parseObject sha1 obj = eitherToMaybe $ parseOnly (objParser sha1) obj

-- header: "type size\0"
-- header ++ content
objParser :: ObjectId -> Parser Object
objParser sha1 = do
   objType' <- string "commit" <|> string "tree" <|> string "blob" <|> string "tag"
   char ' '
   _size <- takeWhile isDigit
   nul
   content <- takeByteString
   return $ Object content (obj objType') sha1
   where obj "commit"   = BCommit
         obj "tree"     = BTree
         obj "tag"      = BTag
         obj "blob"     = BBlob
         obj _          = error "Invalid object type" -- The parser wouldn't get here anyway
```

这仍然是对象内容的紧密表示，仅包含有关类型和对象 ID 的一些元信息。为了使用 git 对象，例如从提交对象解析父提交或从树对象读取树条目，我们需要第二级对象解析，将通用（但已标记）`Object`转换为更具体的对象表示。

#### 犯罪

git 中的提交对象类似于：

```
[4807] λ > git cat-file -p 3e879c7fd33cc3deecd99892033957dedc308e92
tree b11bff45acf0941c7ea5629dfff05760764423cd
parent c3a8276092194bd3ff80d7d6a4523c0f1c0e2df2
author Stefan Saasen <stefan@saasen.me> 1353116070 +1100
committer Stefan Saasen <stefan@saasen.me> 1353116070 +1100

Bump version to 1.6
```

我们`Commit`在程序中使用数据类型来表示：

```
data Commit = Commit {
    getTree        :: B.ByteString
  , getParents     :: [B.ByteString] -- zero (root), one ore more (merges) parents
  , getSha         :: B.ByteString
  , getAuthor      :: Identity
  , getCommiter    :: Identity
  , getMessage     :: B.ByteString
} deriving (Eq,Show)
```

还有一个简单的基于 Attoparsec 的解析器来解析原始提交内容：

```
commitParser :: Parser Commit
commitParser = do
    tree <- "tree " .*> take 40
    space
    parents <- many' parseParentCommit
    author <- "author " .*> parsePerson
    space
    commiter <- "committer " .*> parsePerson
    space
    space
    message <- takeByteString
    let author'   = Author (getPersonName author) (getPersonEmail author)
        commiter' = Commiter (getPersonName commiter) (getPersonEmail commiter)
    return $ Commit tree parents B.empty author' commiter' message
```

#### 斑点

blob 类型的对象只包含 git 跟踪的内容。这是实际的文件内容，因此无需解析或读取。内容`blob`将按原样写入工作副本中的相应文件。

#### 树

引用http://git-scm.com/book/en/Git-Internals-Git-Objects以获得树对象的简洁描述：

> […] 树对象，它解决了存储文件名的问题，还允许您将一组文件存储在一起。Git 以类似于 UNIX 文件系统的方式存储内容，但稍微简化了一些。所有内容都存储为树和 blob 对象，树对应于 UNIX 目录条目，而 blob 或多或少对应于 inode 或文件内容。单个树对象包含一个或多个树条目，每个树条目都包含一个 SHA-1 指针，指向具有相关模式、类型和文件名的 blob 或子树。

查看实际的树对象会立即显示具有每个条目列出的节点、对象类型、SHA1 对象 ID 和文件名的底层模型：

```
[4809] λ > git cat-file -t 19ae5beb4abeea465bfc4aef82fb9373099431c0
tree
[4810] λ > git cat-file -p 19ae5beb4abeea465bfc4aef82fb9373099431c0
100644 blob c364d6f7508e2f6d1607a9d73e6330d68ec7d62a    .ghci
100644 blob c3270b6a3e56c40a570beb1185a53ac1cd48ccd3    .gitignore
100644 blob 38781a3632ce2bd32d7380c6678858afe1f38b19    LICENSE
100644 blob ed4a59a07241be06c3b0ecbbbe89bb4f037c0c70    README.md
100644 blob 200a2e51d0b46fa8a38d91b749f59f20eb97a46d    Setup.hs
040000 tree 754352894497d94b3f50a2353044ded0f592bbb1    example
100644 blob 2fdb4f2db32695c50a0fcae80bd6dca24e7ba7bd    hgit.cabal
040000 tree 58e3ef91a07d0be23ae80f20b8cc18cb7825e1a3    src
100755 blob 0d954128938097e4fc0b666f733b63b27cf14437    test-with-coverage.sh
040000 tree 0b4d3861577e115c29001f38e559440ce27b19b0    tests
```

Git 支持以下模式（来自[git-fast-import](https://www.kernel.org/pub/software/scm/git/docs/git-fast-import.html)）：

- `100644`or `644`：一个普通的（不可执行的）文件。
- `100755`or `755`：一个正常但可执行的文件。
- `120000`：符号链接，文件的内容将是链接目标。
- `160000`: 一个 gitlink，对象的 SHA-1 指的是另一个存储库中的提交。它们用于实现子模块。
- `040000`: 一个子目录。指向另一个树对象。

实际的树对象内容存储为一组具有以下格式的树条目：

```
tree         = 1*tree-entry
tree-entry   = mode SP path NUL sha1

mode         = 6DIGIT
sha1         = 20HEXDIG
path         = UTF8-octets
```

例如：

```
100644 .ghci\NUL\208k\227\&0F\190\137A$\210\193\216j\247#\SI\ETBw;?100644 RunMain.hs\NUL\240i\182\&3g\183\194\241-\131\187W\137\ESC\CAN\f\SOHX\180\174
```

我们使用以下函数将树内容解析为几个简单的数据结构：

```
data Tree = Tree {
    getObjectId :: ObjectId
  , getEntries  :: [TreeEntry]
} deriving (Eq, Show)

data TreeEntry = TreeEntry {
    getMode    :: C.ByteString
  , getPath    :: C.ByteString
  , getBlobSha :: C.ByteString
} deriving (Eq, Show)

parseTree :: ObjectId -> C.ByteString -> Maybe Tree
parseTree sha' input = eitherToMaybe $ parseOnly (treeParser sha') input

-- from e.g. `ls-tree.c`, `tree-walk.c`
treeParser :: ObjectId -> Parser Tree
treeParser sha' = do
    entries <- many' treeEntryParser
    return $ Tree sha' entries
    
treeEntryParser :: Parser TreeEntry
treeEntryParser = do
    mode <- takeTill (== ' ')
    space
    path <- takeTill (== '\0')
    nul
    sha' <- take 20
    return $ TreeEntry mode path sha'
```

有了检索和解析 git 对象的能力，我们现在可以实现检查与给定树对应的文件的功能。我们的入口点是`checkoutHead`函数：

```
checkoutHead :: WithRepository ()
checkoutHead = do
    repo <- ask
    let dir = getName repo
    tip <- readHead
    maybeTree <- resolveTree tip
    indexEntries <- maybe (return []) (walkTree [] dir) maybeTree
    writeIndex indexEntries
```

第一步是解析 symref`HEAD`指向的 commit-id：

```
readHead :: WithRepository ObjectId
readHead = readSymRef "HEAD"

readSymRef :: String -> WithRepository ObjectId
readSymRef name = do
    repo <- ask
    let gitDir = getGitDirectory repo
    ref <- liftIO $ C.readFile (gitDir </> name)
    let unwrappedRef = C.unpack $ strip $ head $ tail $ C.split ':' ref
    obj <- liftIO $ C.readFile (gitDir </> unwrappedRef)
    return $ C.unpack (strip obj)
  where strip = C.takeWhile (not . isSpace) . C.dropWhile isSpace
```

*注意：这是解析 ref 的一个非常简化的版本。`refs.c#resolve_ref_unsafe`处理松散和打包的引用，甚至是旧的符号链接样式引用。对于我们的简单用例，这不是必需的，因为符号 ref 将在上一步中由我们自己的代码编写。*

第二步是解析这个提交指向的树对象：

```
-- | Resolve a tree given a <tree-ish>
-- Similar to `parse_tree_indirect` defined in tree.c
resolveTree :: ObjectId -> WithRepository (Maybe Tree)
resolveTree sha' = do
        repo <- ask
        blob <- liftIO $ readObject repo sha'
        maybe (return Nothing) walk blob
    where walk  (Object _ BTree sha1)                = do
                                                      repo <- ask
                                                      liftIO $ readTree repo sha1
          walk  c@(Object _ BCommit _)               = do
                                                        let maybeCommit = parseCommit $ getBlobContent c
                                                        maybe (return Nothing) extractTree maybeCommit
          walk _                                   = return Nothing

extractTree :: Commit -> WithRepository (Maybe Tree)
extractTree commit = do
    let sha' = C.unpack $ getTree commit
    repo <- ask
    liftIO $ readTree repo sha'
```

解析树涉及读取和解析提交，然后从提交中提取树对象 ID。

如果树查找成功，我们可以开始深度优先遍历树，在工作副本中创建文件。这涉及为`tree`条目 (mode `40000`) 创建目录并创建包含相应 blob 内容的文件，否则：

```
walkTree :: [IndexEntry] -> FilePath -> Tree -> WithRepository [IndexEntry]
walkTree acc parent tree = do
    let entries = getEntries tree
    foldM handleEntry acc entries
    where handleEntry acc' (TreeEntry "40000" path sha') = do
                                let dir = parent </> toFilePath path
                                liftIO $ createDirectory dir
                                maybeTree <- resolveTree $ toHex sha'
                                maybe (return acc') (walkTree acc' dir) maybeTree
          handleEntry acc' (TreeEntry mode path sha') = do
                        repo <- ask
                        let fullPath = parent </> toFilePath path
                        content <- liftIO $ readObject repo $ toHex sha'
                        maybe (return acc') (\e -> do
                                liftIO $ B.writeFile fullPath (getBlobContent e)
                                let fMode = fst . head . readOct $ C.unpack mode
                                liftIO $ setFileMode fullPath fMode
                                indexEntry <- asIndexEntry fullPath sha'
                                return $ indexEntry : acc') content
          toFilePath = C.unpack
          asIndexEntry path sha' = do
                stat <- liftIO $ getFileStatus path
                indexEntryFor path Regular sha' stat
```

*注意：*虽然我们确实处理文件模式（`644`和`755`），但我们目前忽略了其他 git 模式（符号链接和 gitlinks（用于子模块支持））。

`HEAD`在遍历树之后，我们现在可以完整检出与symref指向的提交所标识的存储库快照对应的所有文件。

在该 git 存储库中运行`git status`命令会显示我们所有新创建的文件都未跟踪和计划删除，因为我们还没有创建 git 索引文件。

### Git 索引

作为我们的最后一步，我们需要创建与磁盘上的文件状态匹配的索引文件，这样`git status`就不会报告任何未完成的更改。该索引也称为“暂存区”或目录缓存（不应与包文件附带的索引混淆）。

在 git 中，索引用于跟踪工作副本中的更改并组装将成为下一次提交的一部分的更改。

从`git add`手册页：

> “索引”保存了工作树内容的快照，正是这个快照被作为下一次提交的内容。因此，在对工作目录进行任何更改之后，在运行 commit 命令之前，您必须使用 add 命令将任何新的或修改的文件添加到索引中。

使用一个简单的示例，我们可以观察索引如何跟踪文件更改和状态。

现有存储库跟踪单个文件`LICENSE`并`README`在其工作副本中有一个未跟踪的文件：

```
[4862] λ > git ls-files -scot
? README
H 100644 2831d9e6097f965062d0bb4bdc06e89919632530 0     LICENSE
```

虽然 refs 指向提交 id，但索引文件指向每个文件的 blob object-id。`2831d9`是`LICENSE`blob 的对象 ID：

```
[4863] λ > git cat-file -t 2831d9e6097f965062d0bb4bdc06e89919632530
blob
```

在此阶段，此存储库包含的唯一对象是一个 blob、一棵树和一个提交对象：

```
[4864] λ > tree .git/objects/
.git/objects/
├── 28
│   └── 31d9e6097f965062d0bb4bdc06e89919632530
├── 85
│   └── cbc8d3e3eb1579fc941485b85076d7a97900dd
├── f3
│   └── 8d3a2b142f851984fecc9db9cf34439bb5e47a
├── info
└── pack
```

因此，实际的索引文件只包含一个条目，即`LICENSE`文件的条目。基于`README`文件命令的索引条目的缺失，例如`git status`or`git ls-files -o`可以推断出文件未被跟踪。

索引文件条目包含元数据（例如修改时间、权限）、文件路径和blob 对象的SHA1。

为了构建或暂存下一次提交，当更改（例如整个文件或部分更改`git add -p`）通过`git add`.

这意味着提交更改会导致为将文件绑定到它们的 blob 对象的索引条目以及引用新创建的树的提交对象创建所需的树对象。

观察存储库更改（包括临时创建锁定文件）的有用工具 - 特别是当对象数量很大时 - 使用文件系统通知在运行命令时收到更改通知（例如Linux 上的[inotifywait](http://linux.die.net/man/1/inotifywait)或[spy](https://bitbucket.org/ssaasen/spy) on Mac OS X）：

![混帐索引间谍](https://stefan.saasen.me/articles/git-clone-in-haskell-from-the-bottom-up/images/git-add-spy.png)

可以看出，添加`README`using `git add`（或 using `git update-index --add`）会导致在`.git/objects/ce`目录中创建一个新的 blob 对象文件：

```
[4866] λ > git cat-file -t cebdca635c102a886e8d48c5479b6a7c348c194f
blob
```

现在索引包含两个条目，并且`README`将成为下一次提交的一部分。

![图片](https://stefan.saasen.me/articles/git-clone-in-haskell-from-the-bottom-up/images/git-index.png)

#### 索引格式

目录缓存索引文件本身的格式在`Documentation/technical/index-format.txt`.

索引（存储在 中`.git/index`）有一个 12 字节的索引头，其结构与包文件头的结构相同：

- 4 字节签名或魔术字节`'D' 'I' 'R' 'C'`（用于*dircache*）
- 4 字节版本号
- 4字节的索引条目数

接下来是一些*排序*的索引条目，其中每个索引条目包含 stat(2) 数据（ctime、mtime、device、inode、uid、gid、filesize）、SHA1 对象 id、git 类型（常规文件、符号链接、 gitlink）、unix 权限、几个状态标志以及索引条目所在文件的路径和路径长度（“根据需要填充 1-8 个 nul 字节以将条目填充为 8 个字节的倍数，同时保持名称NUL 终止”）。路径名包括存储库顶级目录中的目录。

该`git ls-files`命令可用于显示索引内容的详细视图：

```
[5003] λ > git ls-files -s --debug
100644 2831d9e6097f965062d0bb4bdc06e89919632530 0       LICENSE
  ctime: 1365582812:0
  mtime: 1365582812:0
  dev: 16777220 ino: 11640465
  uid: 501      gid: 20
  size: 8       flags: 0
…
```

这与`stat`输出匹配：

```
[5004] λ > stat LICENSE
  File: "LICENSE"
  Size: 8            FileType: Regular File
  Mode: (0644/-rw-r--r--)         Uid: (  501/ ssaasen)  Gid: (   20/   staff)
Device: 1,4   Inode: 11640465    Links: 1
Access: Wed Apr 10 21:05:28 2013
Modify: Wed Apr 10 18:33:32 2013
Change: Wed Apr 10 18:33:32 2013
```

通过一种验证索引创建的方式，实现现在自然地遵循以下观察：索引内容大部分*不是*存储库的一部分（例如 ctime/mtime、uid/gid），而是缓存树/目录条目，因此需要在创建时遍历目录树。上面介绍的`walkTree`函数正是这样做的，在检查特定树（即创建必要的文件）时，它会创建并返回`IndexEntry`项目列表：

```
walkTree :: [IndexEntry] -> FilePath -> Tree -> WithRepository [IndexEntry]
walkTree acc parent tree = do
   [...]
                        content <- liftIO $ readObject repo $ toHex sha'
                        maybe (return acc') (\e -> do
   [...]
->                              indexEntry <- asIndexEntry fullPath sha'
                                return $ indexEntry : acc') content

          asIndexEntry path sha' = do
                stat <- liftIO $ getFileStatus path
->              indexEntryFor path Regular sha' stat
```

给定文件路径、git 文件类型（我们目前只考虑常规文件，不考虑符号链接或 gitlinks）、SHA1 对象 ID 和函数返回实例的`stat`信息：`indexEntryFor``IndexEntry`

```
data IndexEntry = IndexEntry {
    ctime       :: Int64
  , mtime       :: Int64
  , device      :: Word64
  , inode       :: Word64
  , mode        :: Word32
  , uid         :: Word32
  , gid         :: Word32
  , size        :: Int64
  , sha         :: [Word8]
  , gitFileMode :: GitFileMode
  , path        :: String
} deriving (Eq)

indexEntryFor :: FilePath -> GitFileMode -> B.ByteString -> FileStatus -> WithRepository IndexEntry
indexEntryFor filePath gitFileMode' sha' stat = do
        repo <- ask
        let fileName = makeRelativeToRepoRoot (getName repo) filePath
        return $ IndexEntry (coerce $ statusChangeTime stat) (coerce $ modificationTime stat)
                        (coerce $ deviceID stat) (coerce $ fileID stat) (coerce $ fileMode stat)
                        (coerce $ fileOwner stat) (coerce $ fileGroup stat) (coerce $ fileSize stat)
                        (B.unpack sha') gitFileMode' fileName
        where coerce = fromIntegral . fromEnum
```

作为检查当前`HEAD`（见上文）的最后一步，我们将索引写入磁盘：

```
checkoutHead :: WithRepository ()
checkoutHead = do
    repo <- ask
    let dir = getName repo
    tip <- readHead
    maybeTree <- resolveTree tip
    indexEntries <- maybe (return []) (walkTree [] dir) maybeTree
    writeIndex indexEntries
```

编写索引需要正确排序索引条目（按名称字段升序排序）并创建 indexHeader：

```
encodeIndex :: Index -> WithRepository B.ByteString
encodeIndex toWrite = do
    let indexEntries = sortIndexEntries $ getIndexEntries toWrite
        numEntries   = toEnum . fromEnum $ length indexEntries
        header       = indexHeader numEntries
        entries      = mconcat $ map encode indexEntries
        idx          = toLazyByteString header `L.append` entries
    return $ lazyToStrictBS idx `B.append` SHA1.hashlazy idx

indexHeader :: Word32 -> Builder
indexHeader num =
        putWord32be magic      -- The signature is { 'D', 'I', 'R', 'C' } (stands for "dircache")
        <> putWord32be 2       -- Version (2, 3 or 4, we use version 2)
        <> putWord32be num     -- Number of index entries
    where magic = fromOctets $ map (fromIntegral . ord) "DIRC"
```

使用`Data.Binary`每个`IndexEntry`可以使用遵循`index-format`规范的以下类型类定义以二进制形式编写：

```
-- see `read-cache.c`, `cache.h` and `built-in/update-index.c`.
instance Binary IndexEntry where
    put (IndexEntry cs ms dev inode' mode' uid' gid' size' sha' gitFileMode' name')
        = do
            put $ coerce cs                     -- 32-bit ctime seconds
            put zero                            -- 32-bit ctime nanosecond fractions
            put $ coerce ms                     -- 32-bit mtime seconds
            put zero                            -- 32-bit mtime nanosecond fractions
            put $ coerce dev                    -- 32-bit dev
            put $ coerce inode'                 -- 32-bit ino
            put $ toMode gitFileMode' mode'     -- 32-bit mode, see below
            put $ coerce uid'                   -- 32-bit uid
            put $ coerce gid'                   -- 32-bit gid
            put $ coerce size'                  -- filesize, truncated to 32-bit
            mapM_ put sha'                      -- 160-bit SHA-1 for the represented object - [Word8]
            put flags                           -- 16-bit
            mapM_ put finalPath                 -- variable length - [Word8] padded with \NUL
        where zero = 0 :: Word32
              pathName                  = name'
              coerce  x                 = (toEnum $ fromEnum x) :: Word32
              toMode gfm fm             = (objType gfm `shiftL` 12) .|. permissions gfm fm
              flags                     = (((toEnum . length $ pathName)::Word16) .&. 0xFFF) :: Word16 -- mask the 4 high order bits 
              -- FIXME: length if the length is less than 0xFFF; otherwise 0xFFF is stored in this field.
              objType Regular           = 8         :: Word32     -- regular file     1000
              objType SymLink           = 10        :: Word32     -- symbolic link    1010
              objType GitLink           = 14        :: Word32     -- gitlink          1110
              permissions Regular fm    = fromIntegral fm :: Word32     -- 0o100755 or 0o100644
              permissions _ _           = 0         :: Word32
              !finalPath                = let n     = CS.encode (pathName ++ "\0")
                                              toPad = 8 - ((length n - 2) `mod` 8)
                                              pad   = C.replicate toPad '\NUL'
                                              padded = if toPad /= 8 then n ++ B.unpack pad else n
                                          in padded
    get = readIndexEntry
```

## 重新实现克隆命令

随着最后一块到位，`git clone`在 Haskell 中从头开始构建的命令现在可以执行并按预期工作：

```
[4900] λ > cabal configure
[4901] λ > cabal build
[4902] λ > cabal copy
[4903] λ > hgit clone git://github.com/juretta/git-pastiche.git
remote: Counting objects: 149, done.
remote: Compressing objects: 100% (103/103), done.
remote: Total 149 (delta 81), reused 113 (delta 45)
ssaasen@monteiths:~/temp [0]
[4903] λ > cd git-pastiche/
ssaasen@monteiths:~/temp/git-pastiche (± master ✓ ) [0]
[4903] λ > git status
# On branch master
nothing to commit, working directory clean
ssaasen@monteiths:~/temp/git-pastiche (± master ✓ ) [0]
[4901] λ > git log --oneline --graph --decorate
* fe484e4 (HEAD, origin/master, origin/HEAD, master) Use eval to evaluate either 'tac' or 'tail -r'
* cb48fc5 Use tac by default for reverse output (if available)
```

## 少了什么东西？

尽管克隆可以工作，但与 git 实现相比，还是缺少很多东西：

- 我们不遍历图看是否全连接
- 目前没有实现symlinks和gitlinks的创建
- 我们没有显式地验证提交 id——尽管我们在存储文件时生成它们，但我们会隐式地进行验证
- 等等…

## 结论

[“Git 概念”](https://www.kernel.org/pub/software/scm/git/docs/user-manual.html#git-concepts)简洁地说：

> Git 是建立在少数简单但强大的想法之上的。

能够重建 git 命令的一个（尽管很小）子集，同时主要依赖于对数据结构、文件格式和协议的研究，而不是实际来源的研究证明了这一说法。[3](https://stefan.saasen.me/articles/git-clone-in-haskell-from-the-bottom-up/#fn:3)

## 脚注

1. 见`transport.c#transport_get_remote_refs` [↩](https://stefan.saasen.me/articles/git-clone-in-haskell-from-the-bottom-up/#fnref:1)
2. [Delta 压缩的文件系统支持](http://mail.xmailserver.net/xdfs.pdf) [↩](https://stefan.saasen.me/articles/git-clone-in-haskell-from-the-bottom-up/#fnref:2)
3. 虽然反过来肯定更容易上手。查看[Git 源代码的鸟瞰图](https://www.kernel.org/pub/software/scm/git/docs/user-manual.html#birdview-on-the-source-code) [↩](https://stefan.saasen.me/articles/git-clone-in-haskell-from-the-bottom-up/#fnref:3)
