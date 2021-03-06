# python OVS 之 stream

git 地址：*https://github.com/openvswitch/ovs/tree/master/python/ovs*

这个 ovs 模块是访问 openstack 的 python 包。

stream 模块是用来建立与 ovsdb 的链接。

## `class Stream(object)`

```
    __S_CONNECTING = 0
    __S_CONNECTED = 1
    __S_DISCONNECTED = 2

    # Kinds of events that one might wait for.
    W_CONNECT = 0               # Connect complete (success or failure).
    W_RECV = 1                  # Data received.
    W_SEND = 2                  # Send buffer room available.

    _SOCKET_METHODS = {}

    IPTOS_PREC_INTERNETCONTROL = 0xc0
    DSCP_DEFAULT = IPTOS_PREC_INTERNETCONTROL >> 2
``` 

用于存储可以进行链接的方法。目前 ovs 实现两种方法，分别是 unix socket 和 TCP/IP sockert。我们以 TCP socket 向下分析。

参考：[Unix domain socket 和 TCP/IP socket 的区别](http://jaminzhang.github.io/network/the-difference-between-unix-domain-socket-and-tcp-ip-socket/)

### `def register_method(method, cls)`

静态方法，注册 socket 方法。在本模块中，有两个地方调用了这个方法：

```
Stream.register_method("unix", UnixStream)
Stream.register_method("tcp", TCPStream)
```

### `def _find_method(name)`

静态方法，根据方法名称找到实现类

### `def is_valid_name(name)`

静态方法，检测是否支持该名称的方法

### `def open(name, dscp=DSCP_DEFAULT)`

静态方法。

`name`：连接方法。（例如：`tcp:127.0.0.1:6640`）
`dscp`：TOS 值。默认为 48

1. 调用子类的 `_open` 方法实现 socket 的建立。
2. 调用 `check_connection_completion` 检查 socket 建立连接的状态
3. 创建 `Stream` 实例

### `def __init__(self, socket, name, status)`

```
    def __init__(self, socket, name, status):
        self.socket = socket
        self.name = name
        if status == errno.EAGAIN:
            self.state = Stream.__S_CONNECTING
        elif status == 0:
            self.state = Stream.__S_CONNECTED
        else:
            self.state = Stream.__S_DISCONNECTED

        self.error = 0
```

* 参数说明：
 1. `socket`：与 ovsdb 建立的 socket 连接
 2. `name`：连接的名称（例如：`tcp:127.0.0.1:6640`）
 3. `status`：socket 连接的状态（正在连接、已连接、未连接）

### `def open_block(error_stream)`

静态方法。这个方法接受的参数是 `open` 方法的返回值。

尝试完成 socket 的链接，若是链接成功，则返回（0，stream）；连接失败则关闭 socket 连接，并返回（负数，stream）

### `def connect(self)`

尝试完成 socket 的链接。

* 返回值：
 1. `EAGAIN`：正在链接
 2. `0`：链接成功
 3. 负数：连接失败

### `def __scs_connecting(self)`

该方法调用 `ovs.socket_util.check_connection_completion` 再次检查连接的状态

### `def close(self)`

关闭 socket 连接

### `def send(self, buf)`

调用 socket 发送 buf 内存储的消息

## `class TCPStream(Stream)`

```
class TCPStream(Stream):
    @staticmethod
    def _open(suffix, dscp):
        error, sock = ovs.socket_util.inet_open_active(socket.SOCK_STREAM,
                                                       suffix, 0, dscp)
        if not error:
            sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
        return error, sock
```

1. 创建 socket
2. 建立与 suffix 的连接
3. 设定 socket 有数据时立即发送



