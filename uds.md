# Unix Domain Socket (UDS) 工作原理详解

## 一、核心概念：什么是 Unix Domain Socket？

Unix Domain Socket（简称 **UDS**，也称 **UNIX socket**）是一种高级的**进程间通信（IPC）机制**，用于在同一台主机上的进程之间进行高效、可靠的数据交换。

它和网络套接字（TCP/IP）使用几乎相同的 API（如 `socket()`, `bind()`, `listen()`, `accept()`, `connect()` 等），但**实现方式完全不同**：

* **寻址方式**：通过 **文件系统路径名** 来标识通信端点（如 `/tmp/my_socket`）。
* **数据传输**：数据不会经过磁盘，也不走网络协议栈，而是直接在**内核内存缓冲区**中完成传输。

可以把它理解为一个“**带门牌号的通信通道**”：文件路径就是门牌号，真正的通信在内核完成。

---

## 二、工作原理详解

UDS 的通信过程与网络套接字相似，分为以下几个阶段：

1. **创建 (`socket()`)**
2. **绑定 (`bind()`)**
3. **监听 (`listen()`)**
4. **接受连接 (`accept()`)**
5. **发起连接 (`connect()`)**
6. **数据传输 (`read()/write()` 或 `send()/recv()`)**

### 流程图

[Server Process]                               [Client Process]

1. socket() 创建套接字                        1. socket() 创建套接字
2. bind() 绑定路径 (/tmp/mysocket)            2. connect() 发起连接请求
3. listen() 开始监听                          |
4. accept() 等待连接  <-----------------------+
        |
        +--> 返回新 fd (cfd) ---------------------> 连接建立成功
              (用于和客户端通信)                (获得通信 fd)

---------------------------------------------------------------
              数据传输（read()/write() 或 send()/recv()）
---------------------------------------------------------------

[Server 用户空间] -> [内核 socket 缓冲区] <- [Client 用户空间]

备注：
- 路径 (/tmp/mysocket) 只是一个“门牌号”，用于寻址。
- 数据并不会写入磁盘，而是在内核内存中完成传输。
---

### 1. 创建 Socket (`socket()`)

```c
int sfd = socket(AF_UNIX, SOCK_STREAM, 0);
```

* **地址族**：`AF_UNIX`（或 `AF_LOCAL`）
* **类型**：

  * `SOCK_STREAM`：面向连接的可靠字节流（类似 TCP）
  * `SOCK_DGRAM`：无连接、不可靠的数据报（类似 UDP）

此时 `sfd` 只是一个未绑定的 socket。

---

### 2. 绑定地址 (`bind()`)

服务器将 socket 绑定到文件系统路径：

```c
struct sockaddr_un addr;
addr.sun_family = AF_UNIX;
strncpy(addr.sun_path, "/tmp/mysocket", sizeof(addr.sun_path) - 1);

bind(sfd, (struct sockaddr*)&addr, sizeof(addr));
```

* **结果**：在 `/tmp/mysocket` 生成一个特殊的 **socket 文件**：

  ```bash
  srwxr-xr-x 1 user group 0 Aug 29 12:00 /tmp/mysocket
  ```
* **作用**：仅作为寻址入口，不存储数据。

---

### 3. 监听连接 (`listen()`)

```c
listen(sfd, 5);
```

* `backlog = 5`：允许最多 5 个待处理的连接请求。

---

### 4. 接受连接 (`accept()`)

```c
int cfd = accept(sfd, NULL, NULL);
```

* `accept()` 返回一个 **新的 fd (`cfd`)**，专门用于和某个客户端通信。
* 原始的 `sfd` 继续用于监听。

---

### 5. 发起连接 (`connect()`)

客户端通过路径连接：

```c
connect(cfd, (struct sockaddr*)&addr, sizeof(addr));
```

* 内核检查路径是否存在并有 socket 监听。
* 成功后，客户端和服务器之间建立“连接通道”。

---

### 6. 数据传输

通信方式和文件 I/O 类似：

```c
write(cfd, "hello", 5);
read(cfd, buf, sizeof(buf));
```

* **数据流向**：

  * 用户空间 → 内核 socket 缓冲区 → 另一进程的用户空间
* **完全在内核内存中完成**，不经过磁盘、不走 TCP/IP 协议栈。

---

### 7. 无连接模式 (`SOCK_DGRAM`)

* 不需要 `listen()` / `accept()`
* 双方 `bind()` 各自路径
* 使用 `sendto()` / `recvfrom()` 通信

---

## 三、核心特性与优势

1. **高性能**

   * 避开网络协议栈，比本地 TCP loopback 更快。
   * 常用于数据库、Docker 等高性能 IPC。

2. **可靠通信**

   * `SOCK_STREAM` 提供可靠性保障。
   * `SOCK_DGRAM` 在本机上几乎不会丢包。

3. **传递文件描述符**

   * 可用 `sendmsg()` + `SCM_RIGHTS` 把 **文件描述符**（如文件、socket、管道等）传给另一个进程。
   * 接收方可直接使用，仿佛自己打开的一样。

4. **传递进程凭证**

   * 内核自动提供客户端的 UID、GID、PID 信息，便于权限控制。

5. **文件系统权限控制**

   * socket 文件权限 (`chmod`, `chown`) 可直接限制谁能连接。

---

## 四、总结

Unix Domain Socket 的本质是：

* **用文件路径作为寻址方式**
* **在内核中通过高效的内存缓冲区完成数据交换**
* **为本地进程提供了类似网络套接字但更快、更安全的通信机制**

它是许多系统服务（数据库、X11、Docker、systemd 等）的首选本地通信方式。