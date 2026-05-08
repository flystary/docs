# Linux 系统调用
## 介绍
内核链接 https://github.com/torvalds/linux.git
## x86-64
系统调用源文件: arch/x86/entry/syscalls/syscall_64.tbl
github链接: https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl


### read
#### 解释
- 系统调用编号: 0
- 函数原型
```c
ssize_t read(int fd, void *buf, size_t count)
```
- 功能
从文件描述符 fd 中最多读取 count 个字节的数据，存入由 buf 指向的缓冲区。
  - 成功时返回实际读取的字节数（≥0）；
  - 返回 0 表示已到达文件末尾（EOF）；
  - 返回 -1 表示发生错误（如无效文件描述符、I/O 错误等），并设置 errno

- 注意
  - 这是原始系统调用，无用户态缓冲（与 fread 等 stdio 函数不同）；
  - 对终端、管道、socket 等设备，read 默认会阻塞，直到有数据可读；
  - 不保证一次读满 count 字节（尤其在非普通文件上）；
  - buf 必须指向进程有效的可写内存区域；
  - 文件描述符 fd 必须是通过 open、pipe、socket、dup 等获得的有效描述符；
  - 多线程共享同一 fd 时，read 会原子地推进文件偏移（对普通文件而言）。

- 使用场景
  - 从标准输入（fd = 0）读取用户命令或数据；
  - 读取配置文件、日志等普通文件内容；
  - 接收网络 socket 或管道中的消息；
  - 实现 cat、cp、shell、自定义 I/O 库等底层工具的核心逻辑。
#### 示例:
```c
#include <unistd.h>

int main() {
    char buffer[256];
    ssize_t n;

    // 从标准输入（fd=0）循环读取并回显到标准输出（fd=1）
    while ((n = read(0, buffer, sizeof(buffer))) > 0) {
        write(1, buffer, n); // write 是系统调用 #1
    }

    return 0;
}

```

### write
#### 解释
- 系统调用编号: 1
- 函数原型
```c
#include <unistd.h>
ssize_t write(int fd, const void *buf, size_t count);

```
- 功能
将 count 个字节的数据从缓冲区 buf 写入到文件描述符 fd 所代表的文件或设备中。
  - 成功时返回实际写入的字节数（通常等于 count，但在特殊设备上可能更少）；
  - 返回 -1 表示发生错误（如无效 fd、磁盘满、管道破裂等），并设置 errno。

- 注意
  - 这是底层系统调用，无用户态缓冲（与 printf/fwrite 不同）；
  - 对普通文件，write 通常是原子的（只要写入量 ≤ PIPE_BUF 或文件系统限制）；
  - 向已关闭的管道或 socket 写入会触发 SIGPIPE 信号（默认终止进程）；
  - buf 必须指向有效的、可读的用户空间内存；
  - 标准输出（fd = 1）和标准错误（fd = 2）是最常用的写入目标；
  - 多线程共享同一 fd 时，write 会原子地推进文件偏移（对普通文件）。

- 使用场景
  - 向终端输出信息（如实现 echo、cat）；
  - 写入日志文件或配置文件；
  - 通过管道或 socket 发送数据；
  - 实现 shell、守护进程、网络服务等的基础 I/O 操作。

#### 示例:
```c
#include <unistd.h>

int main() {
    const char msg[] = "Hello, x86-64 system call!\n";
    // 写入标准输出（fd = 1）
    write(1, msg, sizeof(msg) - 1);  // -1 排除末尾的 '\0'
    return 0;
}

```

### open
#### 解释
- 系统调用编号: 2
- 函数原型
```c
#include <fcntl.h>
int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
打开或创建一个文件，并返回一个文件描述符（file descriptor），该描述符是非负整数，用于后续的 read、write、close 等系统调用。
  - 若文件不存在且指定了 O_CREAT，则按 mode 指定的权限创建新文件；
  - 成功时返回最小的可用文件描述符（≥0）；
  - 失败时返回 -1 并设置 errno（如 ENOENT 文件不存在，EACCES 权限不足等）。

- 注意
  - flags 必须包含以下之一：O_RDONLY（只读）、O_WRONLY（只写）、O_RDWR（读写）；
  - 常用附加标志：
    - O_CREAT：文件不存在时创建；
    - O_TRUNC：打开时清空文件内容；
    - O_APPEND：每次写入都追加到文件末尾；
    - O_EXCL：与 O_CREAT 联用，确保原子创建（避免覆盖）。
  - 当使用 O_CREAT 时，必须提供第三个参数 mode（如 0644），否则行为未定义；
  - 返回的文件描述符是进程私有的，但可通过 fork 继承；
  - 文件描述符在进程内全局共享，多线程需注意同步。

- 使用场景
  - 读取配置文件、日志文件；
  - 创建临时文件或输出文件；
  - 实现 cat、cp、touch 等命令的基础；
  - 打开设备文件（如 /dev/tty）、FIFO、特殊文件等。

#### 示例:
```c
#include <fcntl.h>
#include <unistd.h>

int main() {
    // 以只读方式打开文件
    int fd = open("input.txt", O_RDONLY);
    if (fd == -1) {
        // 错误处理：简单写错误信息到 stderr (fd=2)
        write(2, "open failed\n", 13);
        return 1;
    }

    char buf[128];
    ssize_t n;
    while ((n = read(fd, buf, sizeof(buf))) > 0) {
        write(1, buf, n);  // 输出到 stdout
    }

    close(fd);
    return 0;
}

```

### close
#### 解释
- 系统调用编号: 3
- 函数原型
```c
#include <unistd.h>
int close(int fd);
```
- 功能
关闭指定的文件描述符 fd，释放内核中与该描述符关联的资源。
  - 当一个文件的所有打开实例都被关闭后，若其已被 unlink 删除，则文件数据将被真正释放；
  - 成功时返回 0；失败时返回 -1 并设置 errno（如 EBADF 表示无效文件描述符）。

- 注意
  - 文件描述符表是进程级共享的，close(fd) 会影响整个进程（包括所有线程）；
  - 重复关闭同一个已关闭的 fd 是未定义行为，通常返回 EBADF；
  - 标准流（0: stdin, 1: stdout, 2: stderr）也可以被关闭，但可能导致后续 I/O 异常；
  - 关闭管道或 socket 的一端会通知对端（如读端收到 EOF）；
  - 即使 close 失败（如因延迟写入错误），文件描述符也会被标记为不可用，不应再次使用。

- 使用场景
  - 文件、设备、管道、socket 等 I/O 对象使用完毕后的资源回收；
  - 子进程中关闭不需要继承的文件描述符，防止句柄泄漏；
  - 实现 I/O 重定向后清理原始描述符（如 dup2(new_fd, 1); close(new_fd);）；
  - 编写健壮的系统程序（如 shell、守护进程）时确保无资源泄露。

#### 示例:
```c
#include <unistd.h>
#include <fcntl.h>

int main() {
    // 打开文件（系统调用 #2）
    int fd = open("example.txt", O_RDONLY);
    if (fd == -1) {
        write(2, "open failed\n", 13);
        return 1;
    }

    char buf[64];
    ssize_t n = read(fd, buf, sizeof(buf));  // 系统调用 #0
    if (n > 0) {
        write(1, buf, n);  // 系统调用 #1
    }

    // 关闭文件描述符 —— 触发系统调用 #3
    if (close(fd) == -1) {
        write(2, "close failed\n", 14);
        return 1;
    }

    return 0;
}

```

### stat
#### 解释
- 系统调用编号: 4
- 函数原型
```c
#include <sys/stat.h>
int stat(const char *pathname, struct stat *statbuf);
```
- 功能
获取指定路径 pathname 所指向文件的元信息（metadata），并将结果填充到 statbuf 指向的 struct stat 结构体中。
  - 不要求对文件内容有读权限，但需要对路径上所有目录有执行（x）权限；
  - 若路径是符号链接，则自动解引用（即返回目标文件的信息，而非链接本身）；
  - 成功时返回 0，失败时返回 -1 并设置 errno（如 ENOENT 文件不存在、EACCES 权限不足等）。

- 注意
  - 与 fstat(fd, ...)（编号 5）不同，stat 通过路径名操作，而 fstat 通过文件描述符操作；
  - 与 lstat 不同，stat 会跟随符号链接；若要获取链接自身信息，应使用 lstat；
  - struct stat 包含：文件类型、权限、硬链接数、UID/GID、大小、时间戳（atime/mtime/ctime）等；
  - 常用于判断文件是否存在、是否为目录、获取文件大小等；
  - 是实现 ls -l、file、shell 条件测试（如 [ -f file ]）的核心系统调用之一。

- 使用场景
  - 检查文件是否存在或是否为目录；
  - 获取文件大小（statbuf->st_size）；
  - 实现文件浏览器、备份工具、构建系统等需要文件属性的程序；
  - 安全检查（如验证配置文件是否被篡改，通过比对 mtime）。

#### 示例:
```c
#include <sys/stat.h>
#include <unistd.h>

int main() {
    struct stat sb;
    const char *path = "example.txt";

    // 调用 stat 系统调用（编号 4）
    if (stat(path, &sb) == -1) {
        write(2, "stat failed\n", 13);
        return 1;
    }

    // 简单输出文件大小（需转换为字符串，此处简化）
    char msg[64];
    int len = 0;
    long size = sb.st_size;
    // 手动拼接字符串避免使用 printf
    len += write(1, "File size: ", 12);
    // 简化：假设 size < 10000
    char digits[20];
    int i = 0;
    if (size == 0) digits[i++] = '0';
    else {
        long tmp = size;
        while (tmp) {
            digits[i++] = '0' + (tmp % 10);
            tmp /= 10;
        }
    }
    // 反向写入
    while (i > 0) {
        write(1, &digits[--i], 1);
    }
    write(1, " bytes\n", 8);

    return 0;
}
```

### fstat
#### 解释
- 系统调用编号: 5
- 函数原型
```c
#include <sys/stat.h>
int fstat(int fd, struct stat *statbuf);
```
- 功能
获取与文件描述符 fd 关联的文件的元信息（metadata），并将结果填充到 statbuf 指向的 struct stat 结构体中。
  - 不会触发磁盘 I/O（元数据通常已缓存在内存中）；
  - 成功时返回 0，失败时返回 -1 并设置 errno（如 EBADF 表示无效文件描述符）。

- 注意
  - 与 stat()（编号 4）不同，fstat 通过已打开的文件描述符操作，因此不需要路径解析，也不会受后续文件重命名或删除的影响；
  - 即使原文件已被 unlink 删除，只要还有进程持有该 fd，fstat 仍能成功获取其原始元信息；
  - 对管道、socket、设备等特殊文件也有效（但部分字段如 st_size 可能无意义）；
  - 常用于在不重新打开文件的情况下检查其属性（如大小、类型、时间戳）；
  - fstat是实现高效文件处理（如避免 TOCTOU 竞态条件）的重要工具。

- 使用场景
  - 在读取文件前获取其大小以分配缓冲区；
  - 验证已打开的文件是否为普通文件、目录或设备；
  - 实现 cp、tar 等工具时保留文件属性；
  - 守护进程监控日志文件轮转（通过比较 inode 变化）；
  - 安全编程中避免路径遍历攻击（因不依赖路径名）。

#### 示例:
```c
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    int fd = open("example.txt", O_RDONLY);
    if (fd == -1) {
        write(2, "open failed\n", 13);
        return 1;
    }

    struct stat sb;
    // 调用 fstat 系统调用（编号 5）
    if (fstat(fd, &sb) == -1) {
        write(2, "fstat failed\n", 14);
        close(fd);
        return 1;
    }

    // 输出文件大小（简化版，无 printf）
    char msg[] = "File size via fstat: ";
    write(1, msg, sizeof(msg) - 1);

    // 手动输出数字
    long size = sb.st_size;
    if (size == 0) {
        write(1, "0", 1);
    } else {
        char buf[20];
        int i = 0;
        long tmp = size;
        while (tmp) {
            buf[i++] = '0' + (tmp % 10);
            tmp /= 10;
        }
        while (i > 0) {
            write(1, &buf[--i], 1);
        }
    }
    write(1, " bytes\n", 8);

    close(fd);
    return 0;
}

```

### lstat
#### 解释
- 系统调用编号: 6
- 函数原型
```c
#include <sys/stat.h>
int lstat(const char *pathname, struct stat *statbuf);
```
- 功能
获取指定路径 pathname 所指向文件的元信息（metadata），并将结果填充到 statbuf 指向的 struct stat 结构体中。
  - 关键特性：如果 pathname 是一个符号链接（软链接），lstat 不会解引用它，而是返回符号链接自身的信息（如链接路径长度、类型为 S_IFLNK）；
  - 成功时返回 0，失败时返回 -1 并设置 errno（如 ENOENT 表示路径不存在）。

- 注意
  - 与 stat() 的唯一区别在于对符号链接的处理：
    - stat("link") → 返回 目标文件 的信息；
    - lstat("link") → 返回 链接文件本身 的信息；
  - struct stat 中可通过 st_mode 字段配合宏判断文件类型
     - S_ISREG(mode)：普通文件
     - S_ISDIR(mode)：目录
     - S_ISLNK(mode)：符号链接
     - S_ISFIFO(mode)：管道
     - S_ISSOCK(mode)：socket
     - S_ISCHR(mode)：字符设备
     - S_ISBLK(mode)：块设备
     - S_ISFIFO(mode)：FIFO
  - 链接自身的“大小”（st_size）等于其目标路径字符串的字节长度（不含 \0）；
  - 若路径不是符号链接，lstat 行为与 stat 完全相同；
  - 常用于实现 ls -l（显示 lrwxrwxrwx）、文件类型检测、安全扫描等场景。

- 使用场景
  - 判断一个路径是否为符号链接；
  - 获取符号链接的目标路径长度（配合 readlink）；
  - 实现文件系统遍历工具（如 find、tree）时避免无限循环；
  - 安全审计中检测异常符号链接（如指向敏感文件的链接）；
  - 编写兼容性脚本或系统工具时精确识别文件类型。

#### 示例:
```c
#include <sys/stat.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    if (argc != 2) {
        write(2, "Usage: lstat_test <path>\n", 25);
        return 1;
    }

    struct stat sb;
    // 调用 lstat：若 argv[1] 是符号链接，返回链接自身信息
    if (lstat(argv[1], &sb) == -1) {
        write(2, "lstat failed\n", 14);
        return 1;
    }

    const char *type = "unknown";
    if (S_ISREG(sb.st_mode))  type = "regular file";
    else if (S_ISDIR(sb.st_mode)) type = "directory";
    else if (S_ISLNK(sb.st_mode)) type = "symbolic link";
    else if (S_ISCHR(sb.st_mode)) type = "character device";
    else if (S_ISBLK(sb.st_mode)) type = "block device";
    else if (S_ISFIFO(sb.st_mode)) type = "FIFO/pipe";
    else if (S_ISSOCK(sb.st_mode)) type = "socket";

    write(1, argv[1], strlen(argv[1]));
    write(1, " is a ", 7);
    write(1, type, strlen(type));
    write(1, "\n", 1);

    // 如果是符号链接，显示其“大小”（即目标路径长度）
    if (S_ISLNK(sb.st_mode)) {
        char msg[64];
        int len = 0;
        len += write(1, " -> target path length: ", 24);
        // 手动输出 st_size
        long size = sb.st_size;
        if (size == 0) {
            write(1, "0", 1);
        } else {
            char buf[20];
            int i = 0;
            long tmp = size;
            while (tmp) {
                buf[i++] = '0' + (tmp % 10);
                tmp /= 10;
            }
            while (i > 0) write(1, &buf[--i], 1);
        }
        write(1, " bytes\n", 8);
    }

    return 0;
}

```

### poll
#### 解释
- 系统调用编号: 7
- 函数原型
```c
#include <poll.h>
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```
- 功能
监视多个文件描述符，等待其中任意一个变为可读、可写或发生异常。
  - fds：指向 struct pollfd 数组的指针；
  - nfds：数组中元素个数；
  - timeout：超时时间（毫秒），-1 表示永久阻塞，0 表示立即返回（轮询）。
  - 成功时返回就绪的文件描述符数量（≥0）；
  - 失败时返回 -1 并设置 errno（如 EINTR 被信号中断）。
每个 struct pollfd 定义如下：
```c
struct pollfd {
    int   fd;         // 要监视的文件描述符
    short events;     // 关注的事件（输入）
    short revents;    // 实际发生的事件（输出）
};
```
常见事件标志：
  - POLLIN：有数据可读；
  - POLLOUT：可写；
  - POLLERR：发生错误；
  - POLLHUP：对端关闭连接；
  - POLLNVAL：无效的 fd。

- 注意
  - poll 是水平触发（Level-Triggered） 的 I/O 多路复用机制；
  - 与 select 相比，无文件描述符数量限制（FD_SETSIZE 问题）；
  - 每次调用仍需遍历整个 fds 数组，高并发下性能不如 epoll（Linux 特有）；
  - 超时精度为毫秒级；
  - 即使没有 fd 就绪，只要超时到期就返回 0；
  - 被信号中断时返回 -1 且 errno == EINTR，通常应重试。
- 使用场景
  - 实现单线程多客户端网络服务器（如简易 HTTP 服务）；
  - 同时监听标准输入和 socket（如交互式客户端）；
  - 需要跨平台兼容的 I/O 多路复用（poll 是 POSIX 标准，比 epoll 可移植）；
  - 中等规模并发（几十到几百连接）的轻量级应用。
#### 示例:
```c
#include <poll.h>
#include <unistd.h>
#include <stdio.h>

int main() {
    struct pollfd pfd;
    pfd.fd = STDIN_FILENO;   // 监听标准输入（fd=0）
    pfd.events = POLLIN;     // 关注可读事件

    write(1, "Waiting for input (5 sec timeout)...\n", 39);

    int ret = poll(&pfd, 1, 5000);  // 等待最多 5 秒

    if (ret == -1) {
        write(2, "poll error\n", 12);
        return 1;
    } else if (ret == 0) {
        write(1, "Timeout! No input.\n", 20);
    } else {
        if (pfd.revents & POLLIN) {
            char buf[256];
            ssize_t n = read(STDIN_FILENO, buf, sizeof(buf));
            if (n > 0) {
                write(1, "You typed: ", 12);
                write(1, buf, n);
            }
        }
    }

    return 0;
}

```
### lseek
#### 解释
- 系统调用编号: 8
- 函数原型
```c
#include <unistd.h>
off_t lseek(int fd, off_t offset, int whence);
```
- 功能
重新定位文件描述符 fd 的读写偏移位置（file offset）。
  - 根据 whence 的值，将文件指针移动到指定位置：
    - SEEK_SET（0）：从文件开头开始计算；
    - SEEK_CUR（1）：从当前位置开始计算；
    - SEEK_END（2）：从文件末尾开始计算。
  - 成功时返回新的文件偏移量（以字节为单位，从文件开头起算）；
  - 失败时返回 (off_t)-1 并设置 errno（如 ESPIPE 表示不支持 seek 的设备）。

- 注意
  - 仅对普通文件有效；对管道、FIFO、socket、终端等设备调用 lseek 会失败并返回 ESPIPE；
  - 允许将偏移设到文件末尾之后（用于创建稀疏文件）；
  - 不会修改文件内容，仅改变下一次 read/write 的起始位置；
  - 文件偏移是内核维护的 fd 属性，多线程共享同一 fd 时，lseek 会影响所有线程；
  - 常用于随机访问（如数据库、二进制文件解析）或获取文件大小（lseek(fd, 0, SEEK_END)）。

- 使用场景
  - 获取文件大小（无需读取内容）；
  - 跳过文件头部（如跳过 ELF/BMP/ZIP 文件头）；
  - 实现 tail -c N（从末尾读取 N 字节）；
  - 数据库或日志系统中的随机记录访问；
  - 在已打开的文件中来回移动读写位置。

#### 示例:
```c
#include <unistd.h>
#include <fcntl.h>

int main() {
    int fd = open("data.bin", O_RDONLY);
    if (fd == -1) {
        write(2, "open failed\n", 13);
        return 1;
    }

    // 移动到文件末尾，获取文件大小
    off_t size = lseek(fd, 0, SEEK_END);
    if (size == (off_t)-1) {
        write(2, "lseek failed\n", 14);
        close(fd);
        return 1;
    }

    // 输出文件大小（简化，无 printf）
    char msg[] = "File size: ";
    write(1, msg, sizeof(msg) - 1);

    // 手动输出数字
    if (size == 0) {
        write(1, "0", 1);
    } else {
        char buf[32];
        int i = 0;
        off_t tmp = size;
        while (tmp) {
            buf[i++] = '0' + (tmp % 10);
            tmp /= 10;
        }
        while (i > 0) {
            write(1, &buf[--i], 1);
        }
    }
    write(1, " bytes\n", 8);

    close(fd);
    return 0;
}
```

### mmap
#### 解释
- 系统调用编号: 9
- 函数原型
```c
#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```
- 功能
将文件或设备映射到进程的虚拟地址空间，使得对内存的读写直接对应于对文件的 I/O 操作（即“内存映射 I/O”）。
  - 成功时返回映射区域的起始地址（void *）；
  - 失败时返回 MAP_FAILED（即 (void *) -1）并设置 errno。

- 注意
  - addr 通常设为 NULL，由内核选择映射地址；
  - prot 指定内存保护方式，如 PROT_READ、PROT_WRITE；
  - flags 常用值：
    - MAP_SHARED：修改对其他进程可见，并写回文件；
    - MAP_PRIVATE：修改仅对本进程私有（写时复制），不写回文件；
  - fd 必须指向支持 mmap 的对象（普通文件、共享内存等），且需与 prot 权限匹配；
  - offset 必须是页大小（通常 4096 字节）的整数倍；
  - 映射后可通过指针直接访问文件内容，无需 read/write；
  - 使用完毕后必须调用 munmap()（系统调用编号 11）解除映射；
  - 对大文件处理、进程间共享内存、高效 I/O（如数据库、多媒体）极为重要。

- 使用场景
  - 高效读取大文件（避免多次 read 系统调用开销）；
  - 实现进程间共享内存（配合 shm_open 或匿名映射）；
  - 加载可执行文件或动态库（ELF 加载器的核心机制）；
  - 数据库系统中实现 buffer pool；
  - 图像/视频处理中直接操作文件内容。

#### 示例:
```c
#define _POSIX_C_SOURCE 200809L
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    const char *filename = "example.txt";
    int fd = open(filename, O_RDONLY);
    if (fd == -1) {
        write(2, "open failed\n", sizeof("open failed\n") - 1);
        return 1;
    }

    struct stat sb;
    if (fstat(fd, &sb) == -1) {
        write(2, "fstat failed\n", sizeof("fstat failed\n") - 1);
        close(fd);
        return 1;
    }

    if (sb.st_size == 0) {
        close(fd);
        return 0;
    }

    void *mapped = mmap(NULL, (size_t)sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
    if (mapped == MAP_FAILED) {
        write(2, "mmap failed\n", sizeof("mmap failed\n") - 1);
        close(fd);
        return 1;
    }

    write(1, mapped, (size_t)sb.st_size);

    munmap(mapped, (size_t)sb.st_size);
    close(fd);
    return 0;
}

```

### mprotect
#### 解释
- 系统调用编号: 10
- 函数原型
```c
#include <sys/mman.h>
int mprotect(void *addr, size_t len, int prot);
```
- 功能
修改调用进程地址空间中一段已映射内存区域的访问权限。
  - addr：内存区域起始地址，必须是系统页大小（通常 4096 字节）的整数倍；
  - len：要修改权限的字节数；
  - prot：新的保护标志，可为以下值的按位或：
    - PROT_NONE：不可访问；
    - PROT_READ：可读；
    - PROT_WRITE：可写；
    - PROT_EXEC：可执行。
  - 成功时返回 0；失败时返回 -1 并设置 errno（如 EINVAL、ENOMEM 等）。

- 注意
  - 所修改的内存区域必须完全包含在已有映射中（如通过 mmap、malloc 或程序加载器分配）；
  - 权限变更影响整个内存页，即使只请求部分字节；
  - 常用于实现：
    - 写时复制（Copy-on-Write） 的底层机制；
    - 动态代码生成与执行（如 JIT 编译器：先写入代码，再设为可执行）；
    - 内存安全防护（如将敏感数据区域设为 PROT_NONE 防止意外访问）；
    - 调试器/沙箱 控制内存访问行为；
  - 将栈或堆内存设为可执行可能触发安全机制（如 DEP/NX bit），但 mprotect 本身允许此操作；
  - 不适用于未映射的虚拟地址（会返回 ENOMEM）。

- 使用场景
  - 实现自修改代码或 JIT 引擎（如 V8、LuaJIT）；
  - 构建内存隔离沙箱；
  - 在安全研究中绕过或测试 NX（No-eXecute）保护；
  - 动态加载插件后启用代码执行权限；
  - 实现高效的内存池管理（如标记空闲块为不可访问以捕获 use-after-free）。

#### 示例:
```c
#include <sys/mman.h>
#include <unistd.h>
#include <string.h>

int main() {
    // 获取系统页大小
    long page_size = sysconf(_SC_PAGESIZE);
    if (page_size == -1) return 1;

    // 分配一页可读可写的内存
    void *page = mmap(NULL, page_size, PROT_READ | PROT_WRITE,
                      MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    if (page == MAP_FAILED) return 1;

    // 写入一段 x86-64 的机器码：mov $0x42, %eax; ret
    unsigned char code[] = {0xb8, 0x42, 0x00, 0x00, 0x00, 0xc3};
    memcpy(page, code, sizeof(code));

    // 将该页改为可读可执行（禁止写）
    if (mprotect(page, page_size, PROT_READ | PROT_EXEC) != 0) {
        write(2, "mprotect failed\n", 17);
        munmap(page, page_size);
        return 1;
    }

    // 将函数指针指向该内存并调用
    int (*func)(void) = (int (*)(void))page;
    int result = func();  // 返回 0x42

    // 输出结果（手动转换数字）
    char msg[] = "JIT function returned: ";
    write(1, msg, sizeof(msg) - 1);

    // 简化输出十进制（假设 result < 1000）
    if (result == 0) {
        write(1, "0", 1);
    } else {
        char buf[10];
        int i = 0, tmp = result;
        while (tmp) {
            buf[i++] = '0' + (tmp % 10);
            tmp /= 10;
        }
        while (i > 0) write(1, &buf[--i], 1);
    }
    write(1, "\n", 1);

    munmap(page, page_size);
    return 0;
}

```

### munmap
#### 解释
- 系统调用编号: 11
- 函数原型
```c
#include <sys/mman.h>
int munmap(void *addr, size_t length);
```
- 功能
解除由 mmap 建立的内存映射区域。
  - 将从 addr 开始、长度为 length 的虚拟内存区域从进程地址空间中移除；
  - 若该区域是 MAP_SHARED 映射且有修改，内核会在适当时候将脏页写回文件；
  - 成功时返回 0；失败时返回 -1 并设置 errno（如 EINVAL 表示无效地址或长度）。

- 注意
  - addr 必须是之前 mmap 返回的页面对齐地址；
  - length 应与 mmap 时指定的长度一致（或覆盖整个映射区域）；
  - 即使 munmap 失败，也不应重复释放同一区域；
  - 文件描述符 fd 在 mmap 后即可关闭（映射不依赖 fd 是否打开）；
  - 未调用 munmap 而直接退出进程，内核会自动清理映射，但显式调用是良好实践；
  - 是 mmap（编号 7）的配套系统调用，共同实现内存映射 I/O 的完整生命周期。

- 使用场景
  - 释放通过 mmap 映射的大文件内存区域；
  - 清理进程间共享内存段；
  - 动态加载/卸载模块后回收地址空间；
  - 实现高效内存池或自定义分配器后的资源回收。
  - 避免虚拟地址空间耗尽（尤其在 32 位系统或大量映射场景中）。

#### 示例:
```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    const char *filename = "example.bin";
    int fd = open(filename, O_RDONLY);
    if (fd == -1) {
        write(2, "open failed\n", 13);
        return 1;
    }

    struct stat sb;
    if (fstat(fd, &sb) == -1) {
        write(2, "fstat failed\n", 14);
        close(fd);
        return 1;
    }

    if (sb.st_size == 0) {
        close(fd);
        return 0;
    }

    // 映射文件到内存（系统调用 #7）
    void *addr = mmap(NULL, sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
    if (addr == MAP_FAILED) {
        write(2, "mmap failed\n", 13);
        close(fd);
        return 1;
    }

    // 使用映射内容（例如输出前32字节）
    size_t n = (sb.st_size > 32) ? 32 : sb.st_size;
    write(1, addr, n);
    if (sb.st_size > 32) write(1, "...\n", 4);

    // 解除映射 —— 触发系统调用 #11
    if (munmap(addr, sb.st_size) == -1) {
        write(2, "munmap failed\n", 15);
        close(fd);
        return 1;
    }

    close(fd);
    return 0;
}

```

### brk
#### 解释
- 系统调用编号: 12
- 函数原型
```c
void *brk(void *addr);
// 这是原始系统调用接口。C 标准库通常提供封装函数 int brk(void *addr) 或更常用的 void *sbrk(intptr_t increment)，但行为依赖 glibc 实现。
```
- 功能
调整程序堆（heap）的顶部边界（program break），从而动态扩展或收缩进程的堆内存区域。
  - 参数 addr 指定新的堆顶地址；
  - 成功时返回 0（在某些封装中返回新堆顶地址）；
  - 失败时返回 -1（或 (void *) -1），并设置 errno（如 ENOMEM 表示超出内存限制）。

- 注意
  - 堆位于 BSS 段之上，向高地址增长；
  - brk(addr) 将堆顶设为 addr，要求 addr ≥ 当前堆顶（现代 Linux 允许缩小，但极少使用）；
  - 实际内存分配通常由 malloc/free 管理，它们在小内存请求时使用 brk/sbrk，大内存则用 mmap；
  - 不推荐直接使用 brk：它是低级接口，易出错，且与标准库内存管理器冲突；
  - 多线程环境下，brk 不是线程安全的（glibc 的 malloc 使用 mmap 避免此问题）；
  - 新映射的内存初始化为 0（由内核保证）；
  - 地址必须按页对齐（内核会自动向上取整到页边界）。

- 使用场景
  - 实现自定义内存分配器（如简易 malloc）；
  - 嵌入式或无 libc 环境下的手动内存管理；
  - 理解操作系统内存布局和堆的工作原理；
  - 性能敏感场景下绕过 libc 分配器（极少见）

#### 示例:
```c
#include <unistd.h>
#include <sys/syscall.h>
#include <stdint.h>

// 直接调用系统调用（避免 glibc 封装歧义）
static inline void *sys_brk(void *addr) {
    return (void *)syscall(SYS_brk, addr);
}

int main() {
    void *old_break = sys_brk(0);  // 获取当前堆顶

    if (old_break == (void *)-1) {
        write(2, "brk get failed\n", 16);
        return 1;
    }

    // 请求额外一页内存（假设页大小 4096）
    long page_size = sysconf(_SC_PAGESIZE);
    void *new_break = (void *)((uintptr_t)old_break + page_size);

    if (sys_brk(new_break) == (void *)-1) {
        write(2, "brk set failed\n", 16);
        return 1;
    }

    // 现在 [old_break, new_break) 是可用堆内存
    char *buf = (char *)old_break;
    buf[0] = 'H';
    buf[1] = 'i';
    buf[2] = '\n';

    write(1, buf, 3);  // 输出 "Hi\n"

    // 注意：通常不显式“释放”堆内存，但可尝试缩小（不推荐）
    // sys_brk(old_break);  // 可能被忽略或导致未定义行为

    return 0;
}

```

### rt_sigaction
#### 解释
rt_sigaction 是 "real-time signal action" 的缩写，是 POSIX 实时信号扩展的一部分。尽管名称含 “rt”，但它用于所有信号（包括普通信号如 SIGINT、SIGTERM）的处理设置，在现代 Linux 中已完全取代旧的 sigaction 系统调用。
- 系统调用编号: 13
- 函数原型
```c
// 函数原型（glibc 封装）
#include <signal.h>
int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);

// 底层系统调用
long rt_sigaction(int sig, struct kernel_sigaction *act,
                  struct kernel_sigaction *oact, size_t sigsetsize);

```
- 功能
设置或查询指定信号 signum 的处理方式（handler）
  - 捕获信号并执行自定义函数；
  - 忽略信号（SIG_IGN）；
  - 恢复默认行为（SIG_DFL，如终止、忽略、暂停等）；
  - 设置信号掩码（在信号处理期间临时屏蔽其他信号）；
  - 控制信号处理的附加行为（如重启被中断的系统调用）。
struct sigaction 关键字段
```c
struct sigaction {
    void     (*sa_handler)(int);      // 传统处理函数（仅接收信号编号）
    void     (*sa_sigaction)(int, siginfo_t *, void *); // 扩展处理（需 SA_SIGINFO）
    sigset_t  sa_mask;                // 信号处理期间要阻塞的信号集
    int       sa_flags;               // 标志位
    void    (*sa_restorer)(void);     // 已废弃，应设为 NULL
};
```
常用 sa_flags
  - SA_RESTART：自动重启被信号中断的慢速系统调用（如 read, write）；
  - SA_SIGINFO：使用 sa_sigaction 而非 sa_handler，可获取详细信号信息（发送者 PID、UID、额外数据等）；
  - SA_RESETHAND：处理一次后恢复为默认行为（类似 signal() 的一次性语义）；
  - SA_NODEFER：不自动阻塞当前信号（避免递归处理）。

- 注意
  - 不能用于不可靠信号（如 SIGKILL、SIGSTOP） —— 它们无法被捕获、忽略或阻塞；
  - 信号处理函数必须是异步信号安全（async-signal-safe） 的（不能调用 printf, malloc, exit 等）；
  - 多线程程序中，信号默认只递送给任意一个线程，建议用 pthread_sigmask + 专用信号线程统一处理；
  - rt_sigaction 支持更大的信号集（最多 64 或更多信号），兼容传统和实时信号（SIGRTMIN ~ SIGRTMAX）；
  - glibc 的 signal() 函数内部也使用 rt_sigaction 实现，但语义较弱（不推荐用于生产代码）。

- 使用场景
  - 捕获 SIGINT（Ctrl+C）实现优雅退出；
  - 处理 SIGCHLD 回收僵尸子进程；
  - 使用 SIGUSR1/SIGUSR2 实现进程间控制（如重载配置）；
  - 实现守护进程的信号驱动事件循环；
  - 调试或监控程序行为（如捕获 SIGSEGV 生成 core 前记录上下文）。

#### 示例:
```c
#include <signal.h>
#include <unistd.h>
#include <string.h>

volatile sig_atomic_t keep_running = 1;

void handle_sigint(int sig) {
    // 异步信号安全：只写 volatile sig_atomic_t 变量
    keep_running = 0;
}

int main() {
    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.sa_handler = handle_sigint;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART;  // 重启被中断的 read/write

    if (sigaction(SIGINT, &sa, NULL) == -1) {
        write(2, "sigaction failed\n", 17);
        return 1;
    }

    const char msg[] = "Running... Press Ctrl+C to exit.\n";
    write(1, msg, sizeof(msg) - 1);

    while (keep_running) {
        // 模拟工作（例如 sleep 1 秒）
        char buf[1];
        read(0, buf, 0);  // 阻塞但会被信号中断（因无 SA_RESTART？但此处有）
        // 实际中可用 pause() 或 select/poll
    }

    write(1, "Exiting cleanly.\n", 18);
    return 0;
}

```

### rt_sigprocmask
#### 解释
是 "real-time signal process mask" 的缩写，用于检查和/或修改当前进程（或线程）的信号掩码（signal mask）。尽管名称含 “rt”，它适用于所有信号（包括标准信号如 SIGINT 和实时信号 SIGRTMIN+），是现代 Linux 中管理信号阻塞的核心机制。
- 系统调用编号: 14
- 函数原型
```c
// glic封装
#include <signal.h>
int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);

// 底层系统调用
long rt_sigprocmask(int how, sigset_t *set, sigset_t *oldset, size_t sigsetsize);
```
- 功能
控制哪些信号被阻塞（blocked） —— 即暂时不递送给进程/线程，直到解除阻塞。
  - how 决定操作类型：
    - SIG_BLOCK：将 set 中的信号加入当前掩码（新增阻塞）；
    - SIG_UNBLOCK：从掩码中移除 set 中的信号（解除阻塞）；
    - SIG_SETMASK：完全替换当前掩码为 set
  - set：要操作的信号集（可为 NULL，表示不修改）；
  - oldset：若非 NULL，则返回修改前的信号掩码；
  - 成功返回 0，失败返回 -1 并设置 errno。
无法阻塞 SIGKILL 和 SIGSTOP —— 它们始终会被立即处理。
信号集操作宏（常用）
```c
#include <signal.h>
// 设置信号集
void sigemptyset(sigset_t *set);  // 清空
void sigfillset(sigset_t *set);   // 设置所有信号
void sigaddset(sigset_t *set, int signum);  // 添加信号
void sigdelset(sigset_t *set, int signum);  // 删除信号
// 测试信号集
int sigismember(const sigset_t *set, int signum);  // 判断信号是否在集内
int sigisemptyset(const sigset_t *set);  // 判断信号集是否为空
int sigissigset(const sigset_t *set);  // 判断信号集是否为所有信号
```

- 注意
  - 线程语义：在多线程程序中，每个线程有独立的信号掩码；
    - 主线程的掩码不影响新创建的线程（新线程继承创建者的掩码）；
    - 推荐在主线程中阻塞所有信号，再创建专用“信号处理线程”调用 sigwait() 统一处理；
  - 异步安全性：修改信号掩码本身是安全的，但需注意与信号处理函数的交互；
  - 原子性：sigprocmask 是原子操作，常用于临界区保护（临时屏蔽信号防止中断）；
  - 与 sigaction 配合：sa_mask 字段可在信号处理期间自动阻塞额外信号；
  - 实时信号（SIGRTMIN ~ SIGRTMAX）支持排队，而标准信号不支持（多次发送只递送一次）。

- 使用场景
  - 在临界区临时屏蔽信号，防止处理函数干扰共享数据；
  - 实现同步信号处理（如用 sigwait() + 全阻塞）避免异步处理复杂性；
  - 守护进程中统一管理信号接收；
  - 调试时阻塞特定信号以观察程序行为；
  - 初始化阶段阻塞所有信号，后续由专门线程处理。
#### 示例:
```c
#include <signal.h>
#include <unistd.h>
#include <string.h>

volatile int shared_counter = 0;

void handler(int sig) {
    shared_counter++;  // 简化示例（实际应更谨慎）
}

int main() {
    // 设置 SIGINT 处理函数
    struct sigaction sa;
    sa.sa_handler = handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART;
    sigaction(SIGINT, &sa, NULL);

    sigset_t block_sigint, oldmask;
    sigemptyset(&block_sigint);
    sigaddset(&block_sigint, SIGINT);

    const char msg1[] = "Entering critical section (SIGINT blocked)...\n";
    write(1, msg1, sizeof(msg1) - 1);

    // 阻塞 SIGINT
    sigprocmask(SIG_BLOCK, &block_sigint, &oldmask);

    // 模拟临界区操作（此时即使收到 SIGINT 也不会执行 handler）
    sleep(3);  // 按 Ctrl+C 不会中断此 sleep（因 SA_RESTART + 阻塞）

    const char msg2[] = "Leaving critical section, restoring mask.\n";
    write(1, msg2, sizeof(msg2) - 1);

    // 恢复原信号掩码（若之前未阻塞，则 SIGINT 会在此刻递送）
    sigprocmask(SIG_SETMASK, &oldmask, NULL);

    char buf[32];
    int val = shared_counter;
    // 手动拼接数字输出
    if (val == 0) {
        write(1, "No SIGINT handled.\n", 19);
    } else {
        write(1, "Handled ", 9);
        // 简单输出数字
        char digits[10];
        int i = 0, tmp = val;
        do { digits[i++] = '0' + (tmp % 10); tmp /= 10; } while (tmp);
        while (i--) write(1, &digits[i], 1);
        write(1, " SIGINT(s).\n", 13);
    }

    return 0;
}

```

### rt_sigreturn
#### 解释
- 系统调用编号: 15
- 函数原型
```c
/* 无标准用户态函数原型 —— 由内核和 glibc 内部使用 */
long rt_sigreturn(void);
// 应用程序通常不会也不应该直接调用 rt_sigreturn。它由 C 库（glibc）在信号处理函数末尾自动插入的“返回桩”（signal trampoline）调用，属于内核与运行时协作的底层机制。
```
- 功能
rt_sigreturn 是 "real-time signal return" 的缩写，用于从信号处理函数返回到被中断的用户态上下文。它是一个内核态系统调用，由内核在信号处理函数返回时自动调用。
  - 当进程收到一个信号（如 SIGINT）且设置了自定义处理函数时，内核会：
    - 保存当期执行上下文（寄存器、程序计数器、栈指针等）;
    - 构造一个特殊的栈帧，包含：
      - 被中断时的完整 CPU 状态（struct rt_sigframe）；
      - 一个指向 rt_sigreturn 系统调用的“返回地址”；
    - 跳转到用户注册的信号处理函数；
    - 当信号处理函数执行 return 时，实际会跳转到这个“返回桩”；
    - 返回桩调用 rt_sigreturn，通知内核：“我处理完了，请恢复之前的上下文”。
  - rt_sigreturn 的作用就是
    - 从栈上读取之前保存的 rt_sigframe；
    - 恢复所有寄存器状态（包括 %rip 指向被中断的下一条指令）；
    - 恢复信号掩码（如果处理期间修改过）；
    - 无缝继续原程序执行，仿佛信号从未发生（除了副作用）。
  - 数据结构 在x86-64上， 内核在用户栈上压入如下结构（位于信号处理函数栈帧之下）

```c
struct rt_sigframe {
  /* 返回地址：指向 rt_sigreturn 的桩代码 */
  char retcode[8];        // 包含 mov $15, %rax; syscall

  struct ucontext uc;     // 包含完整的寄存器上下文和信号掩码
};
```

  - 其中 ucontext 包含：
    - uc_mcontext: 通用寄存器、浮点寄存器等；
    - uc_sigmask: 被中断时的信号掩码；
    - uc_stack: 当前栈信息。

- 注意
  - 不要手动调用 rt_sigreturn：会导致栈损坏、寄存器混乱、程序崩溃；
  - 若信号处理函数不正常返回（如调用 longjmp、exit、或无限循环），则 rt_sigreturn 不会被调用，可能导致：
    - 信号掩码未恢复；
    - 栈布局错乱；
    - 后续信号处理异常；
  - 在 SA_ONSTACK 模式下（使用备用信号栈），rt_sigreturn 会负责切换回原栈；
  - 此机制确保了信号处理的透明性和可重入性。


- 使用场景
  - 无直接使用场景 —— 它是操作系统信号机制的内部实现细节；
  - 理解该调用有助于：
    - 调试信号相关崩溃（如栈溢出、非法返回）；
    - 编写安全的信号处理函数（确保正常返回）；
    - 开发嵌入式系统或自定义 libc（需实现信号 trampoline）；
    - 进行二进制逆向或漏洞利用分析（如 SigReturn-Oriented Programming, SROP 攻击）

#### 示例:
```c
#include <signal.h>
#include <unistd.h>
#include <string.h>

void sig_handler(int sig) {
    write(STDOUT_FILENO, "Signal handled.\n", 16);
    // 正常 return → 触发 glibc 插入的 rt_sigreturn 桩
}

int main() {
    struct sigaction sa;
    sa.sa_handler = sig_handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    sigaction(SIGUSR1, &sa, NULL);

    write(STDOUT_FILENO, "Sending SIGUSR1...\n", 20);
    raise(SIGUSR1);  // 发送信号

    write(STDOUT_FILENO, "Back in main.\n", 15);
    return 0;
}

```
  - 使用strace观察rt_sigreturn
```shell
gcc -o sigdemo sigdemo.c
strace ./sigdemo 2>&1 | grep rt_sigreturn
rt_sigreturn({mask=[]}) = 0
```

### ioctl
#### 解释
- 系统调用编号: 16
- 函数原型
```c
#include <sys/ioctl.h>
int ioctl(int fd, unsigned long request, ... /* arg */);
```
- 功能
对文件描述符 fd 所代表的设备或特殊文件执行设备特定的控制操作。
  - request 是一个由内核驱动定义的“命令码”，用于指定要执行的操作；
  - 第三个参数 arg 可以是指针、整数或结构体地址，具体含义由 request 决定；
  - 成功时返回值取决于具体操作（通常为 0），失败时返回 -1 并设置 errno（如 ENOTTY 表示设备不支持该操作）。

- 注意
  - 主要用于字符设备、终端、网络接口、磁盘、串口、GPU 等底层硬件或伪设备；
  - 普通常规文件（如 /home/user/file.txt）通常不支持 ioctl，调用会失败并返回 ENOTTY；
  - 命令码 request 通常通过宏生成，例如：
    - _IO(type, nr)：无参数的命令；
    - _IOR(type, nr, size)：从内核读取数据；
    - _IOW(type, nr, size)：向内核写入数据；
    - _IOWR(type, nr, size)：双向读写；
  - 不同设备的 ioctl 命令互不兼容，需查阅对应驱动文档（如 termios(3), netdevice(7)）；
  - 在现代 Linux 中，部分功能正被更安全、可脚本化的接口（如 sysfs、procfs、netlink）替代，但 ioctl 仍是硬件控制的事实标准。

- 使用场景
  - 获取或设置终端属性（如窗口大小、回显开关）；
  - 配置串口通信参数（波特率、停止位、校验方式）；
  - 查询或控制网络接口（IP 地址、MTU、状态）；
  - 发送 ATA/SCSI 命令到磁盘设备；
  - 控制摄像头、GPU、USB 设备等专用硬件；
  - 实现自定义内核模块的用户态控制通道。

#### 示例:
```c
#include <sys/ioctl.h>
#include <unistd.h>
#include <string.h>

int main() {
    struct winsize w;
    // 使用 TIOCGWINSZ 获取终端窗口尺寸
    if (ioctl(STDOUT_FILENO, TIOCGWINSZ, &w) != 0) {
        write(2, "ioctl failed\n", 14);
        return 1;
    }

    // 手动构造输出字符串（避免使用 printf）
    char out[] = "Terminal: XX rows, YY cols\n";
    // 替换 XX 为 w.ws_row
    int row = w.ws_row;
    out[11] = '0' + (row / 10);
    out[12] = '0' + (row % 10);
    // 替换 YY 为 w.ws_col
    int col = w.ws_col;
    out[20] = '0' + (col / 10);
    out[21] = '0' + (col % 10);

    write(1, out, sizeof(out) - 1);
    return 0;
}

```

```c
#include <sys/socket.h>
#include <sys/ioctl.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <net/if.h>
#include <string.h>
#include <unistd.h>

int main() {
    const char *iface = "eth0";  // 可改为 "eth0", "wlan0" 等

    int sock = socket(AF_INET, SOCK_DGRAM, 0);
    if (sock == -1) {
        write(2, "socket failed\n", 15);
        return 1;
    }

    struct ifreq ifr;
    strncpy(ifr.ifr_name, iface, IFNAMSIZ - 1);

    // 1. 获取 IP 地址
    if (ioctl(sock, SIOCGIFADDR, &ifr) == 0) {
        struct sockaddr_in* addr = (struct sockaddr_in*)&ifr.ifr_addr;
        char ip_str[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &addr->sin_addr, ip_str, INET_ADDRSTRLEN);
        write(1, "IP: ", 4);
        write(1, ip_str, strlen(ip_str));
        write(1, "\n", 1);
    }

    // 2. 获取 MTU
    if (ioctl(sock, SIOCGIFMTU, &ifr) == 0) {
        char msg[] = "MTU: XXXXX";
        int mtu = ifr.ifr_mtu;
        // 简单转数字（假设 MTU < 100000）
        msg[5] = '0' + (mtu / 10000) % 10;
        msg[6] = '0' + (mtu / 1000) % 10;
        msg[7] = '0' + (mtu / 100) % 10;
        msg[8] = '0' + (mtu / 10) % 10;
        msg[9] = '0' + mtu % 10;
        write(1, msg, strlen(msg));
        write(1, "\n", 1);
    }

    // 3. 获取接口状态
    if (ioctl(sock, SIOCGIFFLAGS, &ifr) == 0) {
        if (ifr.ifr_flags & IFF_UP) {
          char *up = "Status: UP\n";
            write(1, up, strlen(up));
        } else {
          char *down = "Status: DOWN\n";
            write(1, down, strlen(down));
        }
    }

    close(sock);
    return 0;
}
```

### pread64
#### 解释
- 系统调用编号: 17
- 函数原型
```c
#include <unistd.h>
ssize_t pread64(int fd, void *buf, size_t count, off64_t offset);
// 在 glibc 中，通常通过 #define _GNU_SOURCE 后直接使用 pread()，它在 64 位系统上自动等价于 pread64。底层系统调用名为 pread64（编号 17），用于支持大文件偏移。
```
- 功能
从文件描述符 fd 对应的文件中，从指定的绝对偏移量 offset 开始读取 count 字节数据到缓冲区 buf，且不会改变文件的当前读写位置（file offset）。
  - 成功时返回实际读取的字节数（可能小于 count，如遇到 EOF）；
  - 失败时返回 -1 并设置 errno（如 EBADF、EINVAL、EFAULT 等）。

- 注意
  - 原子性保证：对普通文件，在 PIPE_BUF（通常 4KB）以内的 pread64 操作是原子的（多线程/多进程并发安全）；
  - 不更新文件偏移：与 read() 不同，pread64 是“定位读”，适合多线程随机访问同一文件；
  - 要求 fd 指向可寻址设备（如普通文件），对管道、FIFO、socket 调用会失败（ESPIPE）；
  - offset 必须 ≥ 0，否则返回 EINVAL；
  - 若 offset 超出文件末尾，返回 0（表示 EOF）；
  - 在 32 位系统上需显式使用 pread64 + -D_FILE_OFFSET_BITS=64 支持大文件，但在 x86-64 上默认即支持。

- 使用场景
  - 多线程程序中并发读取同一文件的不同区域（避免 lseek + read 的竞态）；
  - 数据库、日志系统、文件系统工具实现高效随机 I/O；
  - 实现零拷贝或内存映射前的预读验证；
  - 需要精确控制读取位置而不干扰其他 I/O 操作的场合。

#### 示例:
```c
#include <unistd.h>
#include <fcntl.h>
#include <string.h>


int main() {
    int fd = open("/etc/passwd", O_RDONLY);
    if (fd == -1) {
        char *faild = "open failed\n";
        write(2, faild, strlen(faild));
        return 1;
    }

    char buf[64];
    ssize_t n = pread64(fd, buf, 21, 10);


    if (n == -1) {
        write(2, "pread64 failed\n", 16);
        close(fd);
        return 1;
    }

    if (n > 0) {
        buf[n] = '\0';
        write(1, "Read: ", 6);
        write(1, buf, n);
        write(1, "\n", 1);
    } else {
        write(1, "EOF or empty read\n", 19);
    }

    close(fd);
    return 0;
}

```

```c
#define _GNU_SOURCE
#include <unistd.h>
#include <fcntl.h>
#include <pthread.h>
#include <string.h>
#include <stdio.h>

#define THREADS 4
#define CHUNK_SIZE 32

typedef struct {
    int fd;
    off64_t offset;
    int id;
} thread_arg_t;


void* reader_thread(void* arg) {
   thread_arg_t* t = (thread_arg_t*)arg;

   char buf[CHUNK_SIZE + 1];
   ssize_t n = pread64(t->fd, buf, CHUNK_SIZE, t->offset);
   if (n <= 0) {
      write(2, "Read error or EOF\n", 19);
      return NULL;
   }

   buf[n] = '\0';
   char msg[64];
   strcpy(msg, "Thread ");
   msg[7] = '0' + t->id;
   strcat(msg, ": ");
   strcat(msg, buf);
   strcat(msg, "\n");
   write(1, msg, strlen(msg));

   return NULL;
}


int main() {
   const char* filename = "/etc/passwd";
   int fd = open(filename, O_RDONLY);
   if (fd == -1) {
      write(2, "open failed\n", 13);
      return 1;
   }


   pthread_t threads[THREADS];
   thread_arg_t args[THREADS];

   for (int i = 0; i < THREADS; i++) {
      args[i].fd = fd;
      args[i].offset = i * CHUNK_SIZE;
      args[i].id = i;
      pthread_create(&threads[i], NULL, reader_thread, &args[i]);
   }

   for (int i = 0; i < THREADS; i++) {
      pthread_join(threads[i], NULL);
   }

   close(fd);
   return 0;
}

```

### pwrite64
#### 解释
- 系统调用编号: 18
- 函数原型
```c
#include <unistd.h>
ssize_t pwrite64(int fd, const void *buf, size_t count, off64_t offset);
// 在 x86-64 系统上，glibc 中的 pwrite() 默认就是 pwrite64。若需显式使用 64 位偏移（尤其在 32 位系统），应定义 _GNU_SOURCE 并链接大文件支持（-D_FILE_OFFSET_BITS=64）。
```
- 功能
向文件描述符 fd 对应的文件中，从指定的绝对偏移量 offset 开始写入 count 字节的数据（来自 buf），且不会改变文件的当前读写位置（file offset）。
  - 成功时返回实际写入的字节数（通常等于 count，除非磁盘满或信号中断）；
  - 失败时返回 -1 并设置 errno（如 EBADF, EINVAL, ENOSPC, EFBIG 等）。

- 注意
  - 原子性：对普通文件，当写入大小 ≤ PIPE_BUF（通常为 4096 字节）时，pwrite64 是原子操作，适合多线程/多进程并发写入不同区域；
  - 不更新文件偏移：与 write() 不同，它是“定位写”，避免了 lseek + write 的竞态条件；
  - 仅适用于可寻址设备（如普通文件），对管道、socket、FIFO 调用会失败（返回 -1，errno = ESPIPE）；
  - offset 必须 ≥ 0，否则返回 EINVAL；
  - 若 offset 超过当前文件末尾，文件中间会填充空洞（hole，读取时返回 0 字节）；
  - 如果磁盘空间不足，可能只写入部分数据（返回值 < count）。

- 使用场景
  - 多线程/多进程并发写入同一文件的不同区域（如日志分片、数据库页写入）；
  - 实现无锁的随机写入（配合 pread64）；
  - 文件修复、镜像生成、磁盘工具等需要精确控制写入位置的程序；
  - 避免因共享文件偏移导致的 I/O 错乱。

#### 示例:
```c
#define _GNU_SOURCE
#include <unistd.h>
#include <fcntl.h>
#include <string.h>

int main() {
    // 创建或截断一个文件用于测试
    int fd = open("test_output.bin", O_RDWR | O_CREAT | O_TRUNC, 0644);
    if (fd == -1) {
        write(2, "open failed\n", 13);
        return 1;
    }

    const char *msg1 = "Hello from offset 0\n";
    const char *msg2 = "World at offset 64\n";

    // 在偏移 0 写入
    if (pwrite64(fd, msg1, strlen(msg1), 0) == -1) {
        write(2, "pwrite64 @0 failed\n", 20);
        close(fd);
        return 1;
    }

    // 在偏移 64 写入（中间形成空洞）
    if (pwrite64(fd, msg2, strlen(msg2), 64) == -1) {
        write(2, "pwrite64 @64 failed\n", 22);
        close(fd);
        return 1;
    }

    write(1, "Write completed. Check test_output.bin\n", 38);

    close(fd);
    return 0;
}

```

### readv
#### 解释
- 系统调用编号: 19
- 函数原型
```c
#include <sys/uio.h>
ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
```
- 功能
从文件描述符 fd 中一次性读取数据，并分散（scatter）到多个不连续的缓冲区中，这些缓冲区由 iov 数组描述。
  - iov 是一个 struct iovec 数组，每个元素包含：
    - void *iov_base：缓冲区起始地址；
    - size_t iov_len：该缓冲区长度；
  - iovcnt 表示 iov 数组中元素的个数（必须 ≥ 1 且 ≤ IOV_MAX，通常为 1024）；
  - 成功时返回实际读取的总字节数（可能小于所有缓冲区总和，如遇 EOF）；
  - 失败时返回 -1 并设置 errno。
 I/O 模式称为 “分散读”（scatter read），是高效 I/O 的关键技术之一。
- 注意
  - 数据按顺序填入 iov[0]、iov[1]……直到读完或缓冲区满；
  - 所有缓冲区必须有效（不能重叠，地址需可写）；
  - 若 iovcnt 超出 IOV_MAX（可通过 sysconf(_SC_IOV_MAX) 查询），返回 EINVAL；
  - 对管道、socket、普通文件等均有效；
  - 是原子操作（对 pipe/FIFO，若总大小 ≤ PIPE_BUF）；
  - 不会改变 fd 的文件偏移量（如果是可寻址设备，行为同 read，即会推进偏移）。

- 使用场景
  - 网络编程中接收协议头 + 负载到不同缓冲区（避免内存拷贝）；
  - 高性能日志系统将时间戳、级别、消息体分别写入不同内存块；
  - 实现零拷贝或减少 memcpy 的 I/O 路径；
  - 与 writev 配合，实现高效的“向量化 I/O”（vectored I/O）。

struct iovec 结构说明
```c
struct iovec {
    void  *iov_base;    /* 缓冲区起始地址 */
    size_t iov_len;     /* 缓冲区字节长度 */
};
```
#### 示例:
```c
#define _GNU_SOURCE
#include <sys/uio.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>

int main() {
    int fd = open("/etc/passwd", O_RDONLY);
    if (fd == -1) {
        write(2, "open failed\n", 13);
        return 1;
    }

    char header[20];
    char body[30];
    char footer[10];

    struct iovec iov[3] = {
        { .iov_base = header, .iov_len = sizeof(header) - 1 },
        { .iov_base = body,   .iov_len = sizeof(body) - 1   },
        { .iov_base = footer, .iov_len = sizeof(footer) - 1 }
    };

    ssize_t total = readv(fd, iov, 3);
    if (total == -1) {
        write(2, "readv failed\n", 14);
        close(fd);
        return 1;
    }

    // 确保字符串终止
    header[sizeof(header) - 1] = '\0';
    body[sizeof(body) - 1] = '\0';
    footer[sizeof(footer) - 1] = '\0';

    write(1, "Header: ", 8);
    write(1, header, strlen(header));
    write(1, "\nBody: ", 7);
    write(1, body, strlen(body));
    write(1, "\nFooter: ", 9);
    write(1, footer, strlen(footer));
    write(1, "\n", 1);

    close(fd);
    return 0;
}

```

### writev
#### 解释
- 系统调用编号: 20
- 函数原型
```c
#include <sys/uio.h>
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);
```
- 功能
将多个不连续的内存缓冲区中的数据一次性写入文件描述符 fd，这些缓冲区由 iov 数组描述。
  - iov 是一个 struct iovec 数组，每个元素包含：
  - void *iov_base：缓冲区起始地址；
  - size_t iov_len：该缓冲区长度；
  - iovcnt 表示 iov 数组中元素的个数（必须 ≥ 1 且 ≤ IOV_MAX，通常为 1024）；
  - 成功时返回实际写入的总字节数（通常等于所有缓冲区长度之和，除非被信号中断或磁盘满）；
  - 失败时返回 -1 并设置 errno。

- 注意
  - 数据按顺序从 iov[0]、iov[1]……写入，形成连续输出流；
  - 所有缓冲区必须有效（地址可读，不能重叠）；
  - 若 iovcnt 超出 IOV_MAX（可通过 sysconf(_SC_IOV_MAX) 查询），返回 EINVAL；
  - 对管道、socket、普通文件等均有效；
  - 对 pipe/FIFO，若总写入大小 ≤ PIPE_BUF（通常 4096 字节），操作是原子的；
  - 会推进文件偏移量（如用于普通文件）。

- 使用场景
  - 网络编程中发送协议头 + 负载，避免拼接内存（减少拷贝）；
  - 日志系统将时间戳、日志级别、消息内容分别存储在不同变量中，直接写出；
  - 实现零拷贝或高性能消息序列化；
  - 与 readv 配合，构建高效的“向量化 I/O”（vectored I/O）通道。
#### 示例:
```c
#define _GNU_SOURCE
#include <sys/uio.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>

int main() {
    int fd = open("output.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd == -1) {
        write(2, "open failed\n", 13);
        return 1;
    }

    const char *prefix = "[INFO] ";
    const char *message = "User logged in successfully.";
    const char *suffix = "\n";

    struct iovec iov[3] = {
        { .iov_base = (void*)prefix,  .iov_len = strlen(prefix)  },
        { .iov_base = (void*)message, .iov_len = strlen(message) },
        { .iov_base = (void*)suffix,  .iov_len = strlen(suffix)  }
    };

    ssize_t total = writev(fd, iov, 3);
    if (total == -1) {
        write(2, "writev failed\n", 15);
        close(fd);
        return 1;
    }

    write(1, "Wrote ", 6);
    // 简单输出 total（假设 < 1000）
    char num[8];
    int n = total, i = 0;
    if (n == 0) { write(1, "0", 1); }
    else {
        while (n > 0) { num[i++] = '0' + (n % 10); n /= 10; }
        while (i > 0) write(1, &num[--i], 1);
    }
    write(1, " bytes to output.txt\n", 21);

    close(fd);
    return 0;
}

```

### access
#### 解释
- 系统调用编号: 21
- 函数原型
```c
#include <unistd.h>
int access(const char *pathname, int mode);
```
- 功能
检查调用进程是否对指定路径 pathname 具有某种访问权限。
  - 不返回文件描述符，仅做权限测试；
  - 权限检查基于真实用户 ID（real UID）和真实组 ID（real GID），而非有效 ID（effective UID/GID），因此常用于安全敏感场景（如 setuid 程序）；
  - 成功（权限满足）时返回 0；失败（无权限或文件不存在）时返回 -1 并设置 errno。
  - 参数 mode 可取以下值（可按位或组合）：
    - F_OK：仅检查文件是否存在；
    - R_OK：检查是否可读；
    - W_OK：检查是否可写；
    - X_OK：检查是否可执行。
access() 检查的是实际用户权限，而 open() 使用的是有效用户权限。这是关键区别！
- 注意
  - 存在 TOCTTOU 风险（Time-of-Check vs Time-of-Use）：
  - 即使 access() 返回成功，后续 open() 仍可能失败（因文件在检查后被删除/权限变更）。因此，不应依赖 access() 做安全决策后再调用 open()；更安全的做法是直接 open() 并处理错误。
  - 对符号链接：access() 默认跟随符号链接（检查目标文件权限）；
  - 若路径包含不可遍历的目录（如缺少 x 权限），即使文件存在也会返回 -1（errno = EACCES）；
  - 在 setuid/setgid 程序中，access() 是少数使用 real UID/GID 的系统调用，用于防止提权漏洞。

- 使用场景
  - 脚本或工具在操作前快速验证文件是否存在或是否可读（如配置文件检查）；
  - 安全程序（如 passwd）在 setuid 上下文中验证普通用户对某文件的真实访问能力；
  - 日志轮转工具检查日志文件是否可写；
  - 跨平台兼容性代码（Windows 也有 _access）。

#### 示例:
```c
#include <unistd.h>
#include <string.h>

int main() {
    const char *file = "/etc/passwd";

    // 1. 检查文件是否存在
    if (access(file, F_OK) == 0) {
        write(1, "File exists.\n", 14);
    } else {
        write(1, "File does not exist.\n", 22);
    }

    // 2. 检查是否可读
    if (access(file, R_OK) == 0) {
        write(1, "Readable.\n", 11);
    }

    // 3. 检查是否可写（普通用户通常不可写 /etc/passwd）
    if (access(file, W_OK) != 0) {
        write(1, "Not writable (as expected).\n", 29);
    }

    // 4. 检查一个不存在的文件
    if (access("/nonexistent", F_OK) != 0) {
        write(1, "Confirmed: /nonexistent not found.\n", 35);
    }

    return 0;
}

```

### pipe
#### 解释
写1读0，用完就关，防崩防堵
- 系统调用编号: 22
- 函数原型
```c
#include <unistd.h>
int pipe(int pipefd[2]);
```
- 功能
创建一个匿名管道（anonymous pipe），用于单向进程间通信（IPC）。
  - 创建一个匿名管道（anonymous pipe），用于单向进程间通信（IPC）。
  - 成功时返回 0，并在 pipefd 数组中填入两个文件描述符：
    - pipefd[0]：读端（read end）
    - pipefd[1]：写端（write end）
  - 失败时返回 -1 并设置 errno（如 EMFILE、ENFILE 等）。
数据流向：写入 pipefd[1] → 从 pipefd[0] 读出
- 关键特性
  - 单向通信：只能从写端写、读端读；
  - 字节流语义：无消息边界，读取长度由调用者指定；
  - 内核缓冲区：默认大小通常为 64KB（可通过 fcntl(pipefd[1], F_SETPIPE_SZ, size) 调整）；
  - 阻塞行为：
    - 读空管道 → 阻塞（除非设为非阻塞）；
    - 写满管道 → 阻塞；
    - 所有写端关闭后，读端 read() 返回 0（EOF）；
    - 所有读端关闭后，写端 write() 触发 SIGPIPE 信号（默认终止进程），并返回 -1（errno = EPIPE）；
    - 仅限有亲缘关系的进程：通常用于父子进程通信（因 fd 不能跨无关进程传递，除非配合 Unix domain socket）；
    - 原子写入保证：写入 ≤ PIPE_BUF（POSIX 要求 ≥ 512，Linux 为 4096）字节的操作是原子的。

- 注意
  - 必须关闭不用的端
    - 子进程应关闭写端，父进程关闭读端；
    - 否则读端永远等不到 EOF（因为仍有写端打开）。
  - 避免 SIGPIPE 崩溃
若可能向已关闭读端的管道写入，应：
    - 忽略 SIGPIPE：signal(SIGPIPE, SIG_IGN);
    - 或检查 write() 返回值。
  - 不要用于双向通信
如需双向，应创建两个管道，或改用 socketpair()。
  - 管道不是文件
    - 不支持 lseek、pread、pwrite；
    - 调用这些会返回 -1（errno = ESPIPE）。
写1读0，用完就关，防崩防堵

- 使用场景
  - 父子进程间传递数据（如 shell 中的 ls | grep foo）；
  - 多线程/多进程中协调控制流（如通知退出）；
  - 实现简单的生产者-消费者模型；
  - 作为 fork() + exec() 后重定向 stdin/stdout 的基础。

#### 示例:
```c
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>

int main() {
    int pipefd[2];
    if (pipe(pipefd) == -1) {
        write(2, "pipe failed\n", 13);
        return 1;
    }

    pid_t pid = fork();
    if (pid == -1) {
        write(2, "fork failed\n", 13);
        return 1;
    }

    if (pid == 0) {
        // 子进程：读取数据
        close(pipefd[1]); // 关闭写端

        char buf[32];
        ssize_t n = read(pipefd[0], buf, sizeof(buf) - 1);
        if (n > 0) {
            buf[n] = '\0';
            write(1, "Child received: ", 16);
            write(1, buf, n);
        }
        close(pipefd[0]);
        _exit(0);
    } else {
        // 父进程：发送数据
        close(pipefd[0]); // 关闭读端

        const char *msg = "Hello from parent!\n";
        write(pipefd[1], msg, strlen(msg));
        close(pipefd[1]);

        wait(NULL); // 等待子进程结束
    }

    return 0;
}

```

### select
#### 解释
- 系统调用编号: 23
- 函数原型
```c
#include <sys/select.h>

int select(int nfds,
           fd_set *restrict readfds,
           fd_set *restrict writefds,
           fd_set *restrict exceptfds,
           struct timeval *restrict timeout);
```
- 功能
监视多个文件描述符（fd），等待其中任意一个变为可读、可写或发生异常条件。
  - 成功时返回就绪的文件描述符总数（可能为 0，表示超时）；
  - 失败时返回 -1 并设置 errno（如 EINTR 表示被信号中断）。

- 注意
  - nfds 必须是所有被监视 fd 中的最大值加 1（例如监控 fd 3 和 7，则 nfds = 8）；
  - 调用后，readfds/writefds/exceptfds 集合会被内核修改为仅包含就绪的 fd，下次调用前必须重新初始化；
  - 最大支持 fd 数通常为 1024（由 FD_SETSIZE 限制），不适合高并发场景；
  - 对管道、socket、终端等支持良好，但对普通文件始终视为“就绪”（无意义）；
  - 超时参数 timeout 若为 NULL 则永久阻塞，若为 {0,0} 则立即返回（非阻塞轮询）。

- 使用场景
  - 单线程同时处理多个 I/O 流（如网络服务器监听多个客户端）；
  - 同时响应标准输入和 socket 数据（如交互式客户端）；
  - 实现带超时的 I/O 操作；
  - 跨平台兼容代码（POSIX 标准，Windows 也支持类似机制）
#### 示例:
```c
#define _GNU_SOURCE
#include <sys/select.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>

int main() {
    fd_set read_fds;
    struct timeval timeout;
    char buf[256];

    while (1) {
        // 1. 初始化 fd_set
        FD_ZERO(&read_fds);
        FD_SET(STDIN_FILENO, &read_fds);   // 监听标准输入

        // 2. 设置超时：3 秒
        timeout.tv_sec = 3;
        timeout.tv_usec = 0;

        // 3. 调用 select（nfds = STDIN_FILENO + 1 = 1）
        int ready = select(1, &read_fds, NULL, NULL, &timeout);

        if (ready == -1) {
            perror("select failed");
            break;
        } else if (ready == 0) {
            write(STDOUT_FILENO, "Timeout: no input in 3 seconds.\n", 34);
            continue;
        }

        // 4. 检查哪些 fd 就绪
        if (FD_ISSET(STDIN_FILENO, &read_fds)) {
            ssize_t n = read(STDIN_FILENO, buf, sizeof(buf) - 1);
            if (n <= 0) break; // EOF 或错误
            buf[n] = '\0';
            write(STDOUT_FILENO, "You entered: ", 13);
            write(STDOUT_FILENO, buf, n);
        }
    }

    return 0;
}
```

### sched_yield
#### 解释
- 系统调用编号: 24
- 函数原型
```c
#include <sched.h>
int sched_yield(void);
```
- 功能
主动让出当前 CPU 时间片，将运行机会让给其他就绪态的线程或进程。
  - 成功时返回 0；
  - 失败时返回 -1（实际上在 Linux 中几乎不会失败，通常可忽略返回值）。

- 注意
  - 不阻塞当前线程，仅将其移至就绪队列末尾（具体行为依赖调度策略）；
  - 在默认的 CFS 调度器（SCHED_OTHER）下效果微弱，可能被内核忽略；
  - 不是同步机制，不能用于线程间协调或等待事件；
  - 在 实时调度策略（如 SCHED_FIFO）中，可让同优先级的其他线程运行；
  - 调用后 CPU 使用率仍为 100%（若处于忙循环中），不能替代 sleep 或条件变量。

- 使用场景
  - 用户态自旋锁中减少总线竞争（“让步自旋”）；
  - 同优先级的实时线程间手动协作轮转；
  - 极少数对调度延迟敏感的底层库实现；
  - 教学演示自愿调度行为

#### 示例:
```c
#define _GNU_SOURCE
#include <sched.h>
#include <unistd.h>
#include <sys/syscall.h>

// 辅助函数：输出字符串（避免使用 printf）
static void write_str(const char *msg) {
    write(STDOUT_FILENO, msg, strlen(msg));
}

int main() {
    write_str("Before sched_yield()\n");

    // 主动让出 CPU
    if (sched_yield() == -1) {
        write_str("sched_yield failed\n");
    }

    write_str("After sched_yield()\n");

    // 示例：在自旋中让步（不推荐用于生产，仅演示）
    volatile int flag = 0;
    int count = 0;
    while (flag == 0 && count < 1000000) {
        count++;
        if (count % 100000 == 0) {
            sched_yield(); // 减少对其他线程的干扰
        }
    }

    write_str("Spin loop finished.\n");
    return 0;
}

```

### mremap
#### 解释
- 系统调用编号: 25
- 函数原型
```c
#include <sys/mman.h>

void *mremap(void *old_address, size_t old_size,
             size_t new_size, int flags, ... /* void *new_address */);
```
- 功能
重新映射（resize 或 relocate）一个已存在的内存映射区域。
  - 成功时返回新的映射起始地址；
  - 失败时返回 MAP_FAILED（即 (void *) -1），并设置 errno。
 核心用途：高效地扩大或缩小通过 mmap 分配的内存区域，避免手动 munmap + mmap + memcpy 的开销。
  - old_address	原映射区域的起始地址（必须由 mmap 返回且页对齐）
  - old_size	原映射区域大小（必须 > 0）
  - new_size	新的映射大小（必须 > 0）
  - flags	控制行为：
    - MREMAP_MAYMOVE：允许内核将映射移到新地址
    - MREMAP_FIXED（需配合 MREMAP_MAYMOVE）：指定目标地址（见下文）
若 flags 包含 MREMAP_FIXED，则需传入第 5 个参数 void *new_address，指定新映射的精确地址（必须页对齐且不与现有映射冲突）。

- 使用模式
  - 扩容内存
```c
void *ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                 MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

// 尝试扩容到 8192 字节，若无法原地扩展则允许移动
void *new_ptr = mremap(ptr, 4096, 8192, MREMAP_MAYMOVE);
if (new_ptr == MAP_FAILED) {
    // 处理错误
}
// 注意：此后应使用 new_ptr，原 ptr 可能已失效！
```
  - 缩容内存
```c
void *new_ptr = mremap(ptr, 8192, 2048, 0); // 不需要 MAYMOVE
// 数据前 2048 字节保留，其余丢弃
```
  - 移动内存
```c
void *target = (void *)0x10000000; // 必须页对齐且空闲
void *new_ptr = mremap(ptr, 4096, 4096,
                       MREMAP_MAYMOVE | MREMAP_FIXED, target);
```
- 注意
  - 地址可能改变	若未指定 MREMAP_MAYMOVE 且无法原地扩容，则失败；若指定，则返回新地址，原指针立即失效
  - 内容保留	新映射的前 min(old_size, new_size) 字节内容保持不变；
若扩容，新增部分未初始化（类似 malloc，非零）
  - 仅适用于 mmap 区域	对 malloc/brk 分配的堆内存无效
  - 线程安全	是（但多线程共享同一映射时需同步访问）
  - 错误码	常见 errno：
    - ENOMEM：无法分配新空间
    - EFAULT：地址无效
    - EINVAL：参数错误（如未对齐）

- 使用场景
  - 自定义内存分配器
  - 动态可扩展的缓冲区
  - 共享内存区域的动态调整
  - 垃圾回收器（Garbage Collector）中的堆管理
  - 高性能日志系统（Ring Buffer / Circular Log）
  - 虚拟机或沙箱中的内存模拟

#### 示例:
```c
#define _GNU_SOURCE
#include <sys/mman.h>
#include <unistd.h>
#include <string.h>

// 辅助函数：安全写字符串
static void write_str(const char *s) {
    write(1, s, strlen(s));
}

int main() {
    size_t page_size = getpagesize();
    void *addr = mmap(NULL, page_size, PROT_READ | PROT_WRITE,
                      MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    if (addr == MAP_FAILED) {
        write_str("mmap failed\n");
        return 1;
    }

    // 写入初始数据
    strcpy((char *)addr, "Hello");

    // 扩容到 2 页
    void *new_addr = mremap(addr, page_size, 2 * page_size, MREMAP_MAYMOVE);
    if (new_addr == MAP_FAILED) {
        write_str("mremap failed\n");
        munmap(addr, page_size);
        return 1;
    }

    // 注意：使用 new_addr！原 addr 可能已无效
    strcat((char *)new_addr, ", world!");

    write_str((char *)new_addr);
    write(STDOUT_FILENO, "\n", 1);

    munmap(new_addr, 2 * page_size);
    return 0;
}

```

### msync
#### 解释
- 系统调用编号: 26
- 函数原型
```c
#include <sys/mman.h>
int msync(void *addr, size_t length, int flags);
```
- 功能
将通过 mmap 映射的内存区域中已修改的脏页（dirty pages）同步写回到底层存储设备（通常是磁盘文件），确保内存内容与文件内容一致。
  - 成功时返回 0；
  - 失败时返回 -1，并设置 errno（如 EINVAL、ENOMEM）。
  - 控制 mmap 映射的内存与文件之间的数据一致性，确保修改“真正落盘”。
  - 参数说明
    - addr	映射区域的起始地址（必须页对齐，且属于某次 mmap 的结果）
    - length	要同步的字节数（内核会自动对齐到页边界）
    - flags	同步行为标志（必须指定以下之一）：
      - MS_SYNC：同步阻塞——等待所有 I/O 完成后才返回
      - MS_ASYNC：异步提交——发起写回后立即返回（不保证完成）

可选附加标志：
• MS_INVALIDATE：使其他映射同一文件的进程的缓存失效（强制重新加载）
- 注意
  - 地址必须页对齐：addr 必须是系统页大小（如 4096）的整数倍，否则返回 EINVAL；
  - 长度自动对齐：内核会将 length 向上对齐到页边界；
  - 仅同步脏页：未修改的页不会被写回；
  - MS_SYNC 是阻塞的：调用会等待 I/O 完成，可能耗时较长；
  - MS_ASYNC 不保证完成：仅发起写回请求，立即返回；
  - 不能替代 fsync：若需同步文件元数据（如大小、时间戳），仍需调用 fsync(fd)；
  - 对 MAP_PRIVATE 无效：私有映射的修改不会写回文件，调用 msync 无实际效果（但可能不报错）。

- 使用场景
  - 数据库或日志系统：在写入 WAL（预写日志）后，调用 msync(MS_SYNC) 确保崩溃后可恢复；
  - 配置文件热更新：修改 mmap 映射的配置文件后，显式同步以保证持久化；
  - 多进程共享内存文件：写进程修改后同步，读进程可看到最新数据（配合 MS_INVALIDATE 更可靠）；
  - 持久化内存编程（PMEM）：在非易失性内存（如 Intel Optane）上确保数据落盘；
  - 调试或测试：强制将缓存写入磁盘以验证文件内容。

#### 示例:
```c
#define _GNU_SOURCE
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

// 辅助函数：写字符串到 stdout (fd=1)
static void write_str(const char *msg) {
    const char *p = msg;
    while (*p) p++;
    write(1, msg, p - msg);
}

int main() {
    const char *file = "/tmp/data.txt";
    int fd = open(file, O_RDWR | O_CREAT, 0644);
    if (fd == -1) {
        write(2, "open failed\n", 13); // stderr = 2
        return 1;
    }

    // 确保文件至少一页大小
    ftruncate(fd, 4096);

    // 创建共享映射
    void *addr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (addr == MAP_FAILED) {
        write(2, "mmap failed\n", 13);
        close(fd);
        return 1;
    }

    // 修改映射内容
    const char *msg = "Hello, msync!\n";
    memcpy(addr, msg, 14);

    // 同步到磁盘（阻塞直到完成）
    if (msync(addr, 4096, MS_SYNC) == 0) {
        write_str("Data synced to disk successfully.\n");
    } else {
        write(2, "msync failed\n", 13);
    }

    munmap(addr, 4096);
    close(fd);
    return 0;
}

```

### mincore
#### 解释
- 系统调用编号: 27
- 函数原型
```c
#include <sys/mman.h>
int mincore(void *addr, size_t length, unsigned char *vec);
```
- 功能
查询指定内存映射区域中哪些页面当前驻留在物理内存（即在 RAM 中，未被换出到 swap 或文件）。
  - 成功时返回 0；
  - 失败时返回 -1，并设置 errno（如 ENOMEM、EINVAL）。
  - vec[i] & 1 == 1 表示第 i 个页在内存中；
  - vec[i] & 1 == 0 表示第 i 个页不在内存中（可能在 swap 或磁盘上）。

- 注意
  - 地址必须页对齐：addr 必须是系统页大小（如 4096）的整数倍，否则返回 EINVAL；
  - vec 数组大小：需至少为 (length + page_size - 1) / page_size 字节；
  - 仅适用于 mmap 区域：对堆（malloc）、栈等非映射内存行为未定义；
  - 权限要求低：即使无读权限，也可查询页面是否在内存中（出于安全考虑，Linux 返回模糊结果，但通常仍可用）；
  - 不触发缺页中断：仅查询状态，不会将页面加载到内存；
  - 对匿名映射和文件映射均有效；
  - 结果可能过时：多线程或内核回收可能导致状态变化。

- 使用场景
  - 高性能 I/O 预取优化：在访问大文件前，用 mincore 检查哪些页已在缓存，避免重复预读；
  - 数据库/缓存系统：统计热点数据的内存驻留率，指导淘汰策略；
  - 性能分析工具（如 perf、vmtouch）：测量工作集大小（Working Set Size）；
  - 科学计算：在处理超大数组前，判断是否需显式预加载（madvise(MADV_WILLNEED)）；
  - 虚拟内存调试：诊断 swap 频繁问题，确认关键数据是否常驻内存。

#### 示例:
```c
#define _GNU_SOURCE
#include <sys/mman.h>
#include <unistd.h>
#include <fcntl.h>

// 写字符串到 stdout (fd=1)
static void write_str(const char *s) {
    const char *p = s;
    while (*p) p++;
    write(1, s, p - s);
}

// 写整数到 stdout
static void write_int(int n) {
    if (n == 0) { write(1, "0", 1); return; }
    char buf[16];
    int i = 0, neg = 0;
    if (n < 0) { neg = 1; n = -n; }
    while (n) { buf[i++] = '0' + (n % 10); n /= 10; }
    if (neg) write(1, "-", 1);
    while (i--) write(1, &buf[i], 1);
}

int main() {
    size_t page_size = getpagesize();
    size_t region_size = 4 * page_size; // 4 页

    // 创建匿名映射
    void *addr = mmap(NULL, region_size, PROT_READ | PROT_WRITE,
                      MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    if (addr == MAP_FAILED) {
        write(2, "mmap failed\n", 13);
        return 1;
    }

    // 触发第 0 页和第 2 页的缺页（写入使其驻留内存）
    *((volatile char *)addr) = 1;
    *((volatile char *)addr + 2 * page_size) = 1;

    // 准备 vec 数组（4 字节，每页 1 字节）
    unsigned char vec[4] = {0};

    // 查询页面驻留状态
    if (mincore(addr, region_size, vec) == -1) {
        write(2, "mincore failed\n", 16);
        munmap(addr, region_size);
        return 1;
    }

    // 输出结果
    write_str("Page residency (1=in memory):\n");
    for (int i = 0; i < 4; i++) {
        write_str("Page ");
        write_int(i);
        write_str(": ");
        write_int(vec[i] & 1);
        write(1, "\n", 1);
    }

    munmap(addr, region_size);
    return 0;
}

```

### madvise
#### 解释
- 系统调用编号: 28
- 函数原型
```c
#include <sys/mman.h>

int madvise(void *addr, size_t length, int advice);
```
- 功能
向内核提供关于内存使用模式的建议（advice），帮助内核优化虚拟内存管理策略（如页面换入/换出、预读、缓存等）。
  - 成功返回 0；
  - 失败返回 -1，并设置 errno（如 EINVAL）。
  -
- 注意
  - 地址必须页对齐：addr 应为页大小整数倍（但 Linux 通常自动对齐，不严格报错）；
  - 长度可任意：内核会自动扩展到页边界；
  - 不影响正确性：错误的 advice 只会导致性能下降，不会导致崩溃；
  - 作用范围是虚拟内存区间，与是否已访问无关；
  - 某些 advice 需要特权（如 MADV_HWPOISON 仅 root 可用）；
  - 多线程安全：可并发调用，影响整个进程的虚拟地址空间。

- 使用场景
  - MADV_SEQUENTIAL	顺序访问大文件（如视频流、日志回放）→ 内核减少预读或及时回收已读页
  - MADV_RANDOM	随机访问（如数据库索引）→ 禁用预读，避免污染页缓存
  - MADV_WILLNEED	即将大量读取 → 主动预读到内存（类似 readahead）
  - MADV_DONTNEED	释放不再需要的内存 → 立即丢弃脏页（对 MAP_SHARED 会写回！）或清空干净页，立即释放物理内存
  - MADV_FREE（Linux ≥4.5）	标记可回收内存（如 Go runtime）→ 比 DONTNEED 更高效，延迟释放，保留数据直到内存压力大
  - MADV_HUGEPAGE / MADV_NOHUGEPAGE	控制透明大页（THP）→ 提升 TLB 效率（适合大缓冲区）
  - MADV_MERGEABLE / MADV_UNMERGEABLE	启用 KSM（Kernel Same-page Merging）→ 虚拟化中节省内存（合并相同页）
  - MADV_DONTFORK	禁止子进程继承映射 → 用于安全敏感内存（如加密密钥）

#### 示例:
```c
#define _GNU_SOURCE
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>

static void write_str(const char *s) {
    const char *p = s;
    while (*p) p++;
    write(1, s, p - s);
}

int main() {
    const char *file = "/tmp/large_file";
    int fd = open(file, O_RDONLY);
    if (fd == -1) {
        write(2, "open failed\n", 13);
        return 1;
    }

    // 获取文件大小
    off_t size = lseek(fd, 0, SEEK_END);
    lseek(fd, 0, SEEK_SET);

    // 映射文件
    void *addr = mmap(NULL, size, PROT_READ, MAP_PRIVATE, fd, 0);
    if (addr == MAP_FAILED) {
        write(2, "mmap failed\n", 13);
        close(fd);
        return 1;
    }

    // 场景1: 告知内核我们将顺序读取 → 减少缓存占用
    if (madvise(addr, size, MADV_SEQUENTIAL) != 0) {
        write(2, "madvise SEQUENTIAL failed\n", 27);
    }

    // 场景2: 预读前 1MB 数据到内存
    size_t preload = (1024 * 1024 < size) ? 1024 * 1024 : size;
    if (madvise(addr, preload, MADV_WILLNEED) != 0) {
        write(2, "madvise WILLNEED failed\n", 26);
    }

    // 模拟读取（触发缺页）
    volatile char sum = 0;
    for (off_t i = 0; i < preload; i += 4096) {
        sum += ((char *)addr)[i];
    }

    write_str("Accessed first 1MB with hints.\n");

    // 场景3: 释放后续未使用的部分（假设只用前1MB）
    if (size > preload) {
        if (madvise((char *)addr + preload, size - preload, MADV_DONTNEED) == 0) {
            write_str("Unused pages released.\n");
        }
    }

    munmap(addr, size);
    close(fd);
    return 0;
}

```
MADV_SEQUENTIAL：内核在读取后可能更快地回收页面；
MADV_WILLNEED：提前加载数据，减少后续访问延迟；
MADV_DONTNEED：立即释放物理内存（对私有映射直接丢弃，对共享映射会先写回脏页）。

### shmget
#### 解释
- 系统调用编号: 29
- 函数原型
```c
#include <sys/shm.h>

int shmget(key_t key, size_t size, int shmflg);
```
- 功能
获取或创建一个 System V 共享内存段（shared memory segment）的标识符（shmid）。
  - 成功时返回 非负整数（即共享内存 ID）；
  - 失败时返回 -1，并设置 errno（如 EINVAL、EEXIST、ENOMEM）。
共享内存是进程间通信（IPC）最快的方式之一，允许多个进程直接读写同一块物理内存。

- 注意
  - key 的作用：
    - 若为 IPC_PRIVATE（值为 0），则创建私有共享段（仅父子进程可通过 fork 继承）；
    - 否则通常由 ftok() 生成唯一 key（基于文件路径 + proj_id）；
  - size 要求：
    - 必须 > 0；
    - 实际分配大小向上对齐到 PAGE_SIZE（但使用时仍按原 size 访问）；
  - shmflg 标志：
    - IPC_CREAT：若不存在则创建；
    - IPC_EXCL：与 IPC_CREAT 同用，若已存在则失败；
    - 权限位：如 0666（类似文件权限）；
  - 资源限制：受内核参数限制（如 kernel.shmmax 最大段大小）；
  - 生命周期：
    - 共享段独立于进程存在，需显式删除（shmctl(..., IPC_RMID, ...)）；
    - 系统重启后自动清除；
  - 现代替代方案：POSIX 共享内存（shm_open + mmap）更推荐，但 System V 仍广泛用于遗留系统。
- 使用场景
  - 高性能进程间数据共享：如数据库缓存、实时交易系统；
  - 多进程协作处理大块数据：避免频繁复制（如图像处理流水线）；
  - 遗留系统维护：许多老 Unix/Linux 应用（如 Oracle 早期版本）依赖 System V IPC；
  - 无亲缘关系进程通信：通过 ftok 生成公共 key 实现任意进程共享；
  - 作为其他 IPC 基础：某些消息队列或信号量实现底层使用共享内存。
#### 示例:
```c
#define _GNU_SOURCE
#include <sys/shm.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

static void write_str(const char *s) {
   const char *p = s;
   while (*p) p++;
   write(1, s, p -s);
}


static void write_int(int n) {
   if (n == 0) {write(1, "0", 1); return;}
   char buf[16];
   int i = 0, neg = 0;
   if (n < 0) {neg = 1; n = -n;}
   while (n) {buf[i++] = '0' + (n % 10); n /= 10;}
   if (neg) write(1, "-", 1);
   while (i--) write(1, &buf[i], 1);
}

int main() {
   key_t key = ftok("/proc/self/exe", 'A');

   if (key == -1) {
      write(2, "shmget failed\n", 15);
      return 1;
   }

   int shmid = shmget(key, 4096, IPC_CREAT | 0666);
   if (shmid == -1) {
      write(2, "shmget failed\n", 15);
      return 1;
   }

   write_str("Shared memory ID: ");
   write_int(shmid);
   write(1, "\n", 1);

   shmctl(shmid, IPC_RMID, NULL);
   return 0;
}

```

### shmat
#### 解释
- 系统调用编号: 30
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
将一个 System V 共享内存段（由 shmget 创建）附加（attach）到当前进程的虚拟地址空间，返回其映射地址。
  - 成功时返回 指向共享内存的指针；
  - 失败时返回 (void *) -1（即 MAP_FAILED），并设置 errno（如 EINVAL、ENOMEM）。

- 注意
  - shmid：必须是由 shmget 返回的有效共享内存 ID；
  - shmaddr：
    - 若为 NULL（最常用），内核自动选择合适地址；
    - 若非 NULL，需页对齐，且通常需配合 SHM_RND 使用（向下取整到 SHMLBA 边界）；
  - shmflg：
   - SHM_RDONLY：以只读方式附加；
   - SHM_REMAP：允许覆盖已有映射（谨慎使用）；
   - 通常传 0 表示可读写；
  - 引用计数：内核维护附加次数，只有所有进程都 shmdt 后，段才可被 IPC_RMID 删除；
  - 地址生命周期：直到调用 shmdt 或进程退出；
  - 不能与 mmap 混用：System V 共享内存是独立于 POSIX mmap 的机制。

- 使用场景
  - 多进程共享数据结构：如全局配置、计数器、环形缓冲区；
  - 高性能服务通信：Web 服务器主进程与工作进程共享会话状态；
  - 科学计算：多个 worker 进程协作处理同一数据集；
  - 遗留系统集成：与使用 System V IPC 的老程序交互；
  - 避免数据拷贝：相比管道、消息队列，零拷贝提升吞吐量。

#### 示例:
```c
#define _GNU_SOURCE
#include <sys/shm.h>
#include <unistd.h>

// 写字符串到 stdout (fd=1)
static void write_str(const char *s) {
    const char *p = s;
    while (*p) p++;
    write(1, s, p - s);
}

// 写整数到 stdout
static void write_int(long n) {
    if (n == 0) { write(1, "0", 1); return; }
    char buf[24];
    int i = 0;
    int neg = 0;
    if (n < 0) { neg = 1; n = -n; }
    while (n) { buf[i++] = '0' + (n % 10); n /= 10; }
    if (neg) write(1, "-", 1);
    while (i--) write(1, &buf[i], 1);
}

int main() {
    // 假设已通过 shmget 获得 shmid（此处硬编码演示，实际应传递或约定）
    // 为简化，本例假设 shmid=65536 存在（请先运行 shmget 示例创建）
    int shmid = 65536;

    // 附加共享内存段
    void *addr = shmat(shmid, NULL, 0);
    if (addr == (void *)-1) {
        write(2, "shmat failed\n", 14);
        return 1;
    }

    write_str("Attached at address: 0x");
    write_int((long)addr);
    write(1, "\n", 1);

    // 写入数据（假设前 8 字节存一个整数）
    *(volatile int *)addr = 42;
    write_str("Wrote value: 42\n");

    // 读取验证
    int val = *(int *)addr;
    write_str("Read value: ");
    write_int(val);
    write(1, "\n", 1);

    // 分离（重要！）
    if (shmdt(addr) == 0) {
        write_str("Detached successfully.\n");
    } else {
        write(2, "shmdt failed\n", 14);
    }

    return 0;
}

```

### shmctl
#### 解释
- 系统调用编号: 31
- 函数原型
```c
#include <sys/shm.h>

int shmctl(int shmid, int cmd, struct shmid_ds *buf);
```
- 功能
对 System V 共享内存段执行控制操作，如获取状态、修改权限或标记删除。
  - 成功时返回 0；
  - 失败时返回 -1，并设置 errno（如 EINVAL、EPERM、EIDRM）。
管理共享内存生命周期和属性的核心接口，用于获取共享内存状态、修改权限和标记删除。
- 注意
  - shmid：必须是由 shmget 返回的有效共享内存 ID；
  - cmd 常用值：
    - IPC_STAT：获取段信息到 buf（需有效指针）；
    - IPC_SET：设置段的 uid、gid、mode（需属主或特权）；
    - IPC_RMID：立即标记删除（实际销毁在所有进程 shmdt 后发生）；
  - buf 参数:
    - IPC_STAT / IPC_SET 时需指向 struct shmid_ds；
    - IPC_RMID 时可传 NULL；
  - 删除行为：
    - 调用 IPC_RMID 后，shmid 立即失效（后续 shmctl/shmat 报 EINVAL）；
    - 但内存不会立即释放，直到所有附加进程调用 shmdt；
  - 权限要求：
    - IPC_RMID / IPC_SET 需满足：有效用户 = 段创建者/属主，或为 root；
  - 不可逆操作：IPC_RMID 一旦成功，无法撤销。
- 使用场景
  - 清理资源：程序退出前删除不再需要的共享段（防止泄漏）；
  - 动态调整权限：允许其他用户组访问共享内存；
  - 监控与调试：查询段大小、附加进程数、最后访问时间等；
  - 优雅关闭：服务停止时标记删除，等待工作进程自然分离；
  - 安全加固：启动后修改权限为仅限特定用户访问。
#### 示例:
```c
#define _GNU_SOURCE
#include <sys/shm.h>
#include <unistd.h>

static void write_str(const char *s) {
    const char *p = s;
    while (*p) p++;
    write(1, s, p - s);  // stdout = 1
}

// 简易整数输出（十进制）
static void write_int(long n) {
    if (n == 0) { write(1, "0", 1); return; }
    char buf[24];
    int i = 0, neg = 0;
    if (n < 0) { neg = 1; n = -n; }
    while (n) { buf[i++] = '0' + (n % 10); n /= 10; }
    if (neg) write(1, "-", 1);
    while (i--) write(1, &buf[i], 1);
}

int main() {
    key_t key = ftok("/tmp", 'S');
    if (key == -1) {
        write(2, "ftok failed\n", 13);
        return 1;
    }

    // 创建共享内存段（4KB）
    int shmid = shmget(key, 4096, IPC_CREAT | 0666);
    if (shmid == -1) {
        write(2, "shmget failed\n", 15);
        return 1;
    }

    write_str("Created shmid: ");
    write_int(shmid);
    write(1, "\n", 1);

    // 场景1: 查询共享段信息
    struct shmid_ds info;
    if (shmctl(shmid, IPC_STAT, &info) == 0) {
        write_str("Size: ");
        write_int(info.shm_segsz);
        write_str(" bytes\n");
        write_str("Attached procs: ");
        write_int(info.shm_nattch);
        write(1, "\n", 1);
    }

    // 场景2: 修改权限（例如禁止写入）
    if (shmctl(shmid, IPC_STAT, &info) == 0) {
        info.shm_perm.mode = 0444; // 只读
        if (shmctl(shmid, IPC_SET, &info) == 0) {
            write_str("Permission changed to 0444.\n");
        }
    }

    // 场景3: 标记删除（关键！防止系统残留）
    if (shmctl(shmid, IPC_RMID, NULL) == 0) {
        write_str("Marked for deletion.\n");
        // 此时 shmid 已无效，但内存仍存在（直到所有 shmdt）
    }

    // 注意：即使标记删除，仍可附加（只要还没被真正销毁）
    void *addr = shmat(shmid, NULL, 0);
    if (addr == (void *)-1) {
        write_str("Attach after RMID failed (expected after detach).\n");
    } else {
        write_str("Attached even after RMID (still in use).\n");
        shmdt(addr);
    }

    return 0;
}

```

### dup
#### 解释
- 系统调用编号: 32
- 函数原型
```c
#include <unistd.h>

int dup(int oldfd);
```
- 功能
复制一个已存在的文件描述符（file descriptor），返回新的最小可用文件描述符，该新描述符与原描述符共享同一文件表项（即指向相同的内核 I/O 状态，如文件偏移、状态标志等）。
  - 成功时返回 新的文件描述符（≥ 0）；
  - 失败时返回 -1，并设置 errno（如 EBADF）。

- 注意
  - 共享状态：
    - read/write 会更新共同的文件偏移量；
    - 修改文件状态标志（如 O_APPEND）会影响两者；
  - 独立的文件描述符标志：
    - FD_CLOEXEC（close-on-exec）是 per-fd 的，dup 默认不继承该标志（但 Linux 行为可能因版本略有差异，建议显式设置）；
  - 返回最小可用 fd：从 0 开始找第一个未被占用的描述符；
  - 不能用于无效 fd：若 oldfd 未打开，返回 EBADF；
  - 线程安全：是原子操作，多线程可安全调用；
  - 与 dup2/dup3 区别：
    - dup：自动分配新 fd；
    - dup2：指定目标 fd（若已被占用则先关闭）；
    - dup3：同 dup2，但可显式设置 O_CLOEXEC。

- 使用场景
  - 重定向标准流：
```c
int fd = open("log.txt", O_WRONLY | O_CREAT, 0644);
close(STDOUT_FILENO);     // 关闭 stdout (1)
dup(fd);                  // 新 fd 为 1，stdout 指向 log.txt
close(fd);                // 原 fd 不再需要
printf("This goes to file\n"); // 输出到文件
```
  - 保存原始标准输出（用于恢复）：
```c
int saved_stdout = dup(STDOUT_FILENO); // 保存副本
// ... 重定向 stdout ...
dup2(saved_stdout, STDOUT_FILENO);     // 恢复
close(saved_stdout);
```
  - 简化管道使用：
在父子进程中将管道端绑定到标准输入/输出；
  - 实现 shell 重定向：如 command > file 的底层机制；
  - 调试或日志复制：将输出同时写入终端和文件（需配合 tee 或自定义写）。

#### 示例:
```c
#define _GNU_SOURCE
#include <unistd.h>
#include <fcntl.h>

static void write_str(const char *s) {
    const char *p = s;
    while (*p) p++;
    write(1, s, p - s);  // stdout = 1
}

int main() {
    // 打开一个文件
    int fd = open("/tmp/dup_test.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd == -1) {
        write(2, "open failed\n", 13);
        return 1;
    }

    // 复制 fd → 得到新描述符（假设为 3）
    int newfd = dup(fd);
    if (newfd == -1) {
        write(2, "dup failed\n", 12);
        close(fd);
        return 1;
    }

    write_str("Original fd: ");
    // 简易输出整数
    char buf[8];
    int n = fd, i = 0;
    do { buf[i++] = '0' + (n % 10); n /= 10; } while (n);
    while (i--) write(1, &buf[i], 1);

    write_str(", Duplicated fd: ");
    n = newfd; i = 0;
    do { buf[i++] = '0' + (n % 10); n /= 10; } while (n);
    while (i--) write(1, &buf[i], 1);
    write(1, "\n", 1);

    // 通过原 fd 写入
    write(fd, "Hello from fd ", 15);
    char num[8]; n = fd; i = 0;
    do { num[i++] = '0' + (n % 10); n /= 10; } while (n);
    while (i--) write(fd, &num[i], 1);
    write(fd, "\n", 1);

    // 通过新 fd 继续写（偏移量延续！）
    write(newfd, "Continued from newfd\n", 21);

    close(fd);      // 关闭原 fd，但 newfd 仍有效
    close(newfd);   // 此时文件才真正关闭

    write_str("Check /tmp/dup_test.txt for output.\n");
    return 0;
    // close(fd); close(newfd);	两个都关闭，文件真正关闭
}
```

### dup2
#### 解释
- 系统调用编号: 33
- 函数原型
```c
#include <unistd.h>

int dup2(int oldfd, int newfd);```
```
- 功能
将文件描述符 oldfd 复制到指定的描述符 newfd。
  - 如果 newfd 已打开，则先自动关闭它，再进行复制；
  - 如果 oldfd == newfd，则直接返回 newfd（不关闭）；
  - 成功时返回 newfd；
  - 失败时返回 -1，并设置 errno（如 EBADF）。
本质：强制让 newfd 指向与 oldfd 相同的内核文件表项。

- 注意
  - 原子操作：关闭旧 newfd 和复制 oldfd 是一个不可分割的操作（避免竞态）；
  - 共享状态：oldfd 与 newfd 共享文件偏移、状态标志（如 O_APPEND）；
  - 独立 fd 标志：FD_CLOEXEC 是 per-fd 的，dup2 不会复制该标志（Linux 默认新 fd 的 FD_CLOEXEC=0）；
  - newfd 必须有效范围：通常 ≥ 0 且 < RLIMIT_NOFILE；
  - 即使 oldfd 无效，若 newfd 已打开也会被关闭（先关后报错）；
  - 线程安全：是原子系统调用，多线程可安全使用。

- 使用场景
  - 重定向标准流（最常见用途）：
```c
int fd = open("output.log", O_WRONLY | O_CREAT, 0644);
dup2(fd, STDOUT_FILENO);  // stdout (1) → output.log
close(fd);                // 原 fd 不再需要
printf("This goes to log\n"); // 自动写入文件
```
  - 恢复原始标准输出：
```c
int saved_stdout = dup(STDOUT_FILENO); // 保存副本（如 fd=3）
// ... 重定向 stdout ...
dup2(saved_stdout, STDOUT_FILENO);     // 恢复
close(saved_stdout);
```
  - 管道连接进程：
```c
// 子进程中：将管道读端绑定到 stdin
dup2(pipefd[0], STDIN_FILENO);
close(pipefd[0]); close(pipefd[1]);
execlp("sort", "sort", NULL);
```
  - 实现 shell 重定向：如 command 2>&1（dup2(1, 2)）；
  - 统一日志输出：将 stderr 重定向到 stdout 或日志文件。
#### 示例:
```c
#define _GNU_SOURCE
#include <unistd.h>
#include <fcntl.h>

static void write_str(const char *s) {
    const char *p = s;
    while (*p) p++;
    write(1, s, p - s);  // stdout = 1
}

int main() {
    // 打开日志文件
    int logfd = open("/tmp/dup2_demo.log", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (logfd == -1) {
        write(2, "open failed\n", 13);
        return 1;
    }

    // 保存原始 stdout（假设为 fd=3）
    int saved_stdout = dup(STDOUT_FILENO);

    // 将 stdout 重定向到日志文件
    dup2(logfd, STDOUT_FILENO);
    close(logfd);  // 原 logfd 可关闭，stdout 已指向文件

    // 此时 printf / write(1, ...) 都写入文件
    write_str("This goes to log file.\n");

    // 恢复原始 stdout
    dup2(saved_stdout, STDOUT_FILENO);
    close(saved_stdout);

    write_str("This goes back to terminal.\n");
    return 0;
}

```

### pause
#### 解释
- 系统调用编号: 34
- 函数原型
```c
#include <unistd.h>
int pause(void);
```
- 功能
使当前进程（或线程）进入睡眠状态，直到接收到一个信号（signal）为止。
  - 永不正常返回（即不会返回 0）；
  - 唯一可能的返回是被信号中断后返回 -1，并设置 errno = EINTR。
本质：主动挂起执行，等待异步事件（通过信号通知）。
- 注意
  - 无限期阻塞：除非有信号到达，否则进程会一直休眠；
  - 仅响应“未被忽略且未被阻塞”的信号：
    - 若信号被 signal(SIG_XXX, SIG_IGN) 忽略 → pause() 不唤醒；
    - 若信号被 sigprocmask 阻塞 → pause() 不唤醒（信号处于 pending 状态）；
  - 返回值总是 -1，且 errno 一定为 EINTR（“Interrupted system call”）；
  - 不是可重入的安全点：在信号处理函数中调用 pause() 可能导致死锁；
  - 现代替代方案：
    - 使用 sigsuspend()（更安全，可临时替换信号掩码）；
    - 使用 select/poll/epoll + 信号文件描述符（signalfd）实现统一事件循环。

- 使用场景
  - 简单守护进程主循环：
```c
signal(SIGTERM, handle_exit);
while (1) {
    pause();  // 等待信号
    // 实际处理在信号处理函数中完成
}
```
  - 同步等待特定事件（配合全局标志）：
```c
volatile sig_atomic_t got_signal = 0;
void handler(int sig) { got_signal = 1; }

signal(SIGUSR1, handler);
while (!got_signal) {
    pause();  // 高效等待，不忙等
}
```
  - 教学示例：演示信号基本机制；
  - 旧式 Unix 编程：在没有高级 I/O 多路复用的时代用于事件等待。
#### 示例:
```c
#define _GNU_SOURCE
#include <unistd.h>
#include <signal.h>
#include <stdio.h>

static volatile sig_atomic_t quit = 0;

void handle_sigterm(int sig) {
    quit = 1;
}

// 简易写字符串（避免 printf 在信号中不安全）
static void write_str(const char *s) {
    const char *p = s;
    while (*p) p++;
    write(1, s, p - s);
}

int main() {
    // 注册信号处理函数
    signal(SIGTERM, handle_sigterm);
    signal(SIGINT, handle_sigterm);   // Ctrl+C

    write_str("Process started. PID: ");
    // 输出当前 PID
    char pid_buf[16];
    int pid = getpid(), i = 0;
    do { pid_buf[i++] = '0' + (pid % 10); pid /= 10; } while (pid);
    while (i--) write(1, &pid_buf[i], 1);
    write(1, "\n", 1);

    write_str("Waiting for SIGINT or SIGTERM...\n");

    // 主循环：高效等待信号
    while (!quit) {
        pause();  // 阻塞直到信号到来
    }

    write_str("Exiting cleanly.\n");
    return 0;
}

```

### nanosleep
#### 解释
- 系统调用编号: 35
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### gettimer
#### 解释
- 系统调用编号: 36
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### alarm
#### 解释
- 系统调用编号: 37
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setitimer
#### 解释
- 系统调用编号: 38
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### getpid
#### 解释
- 系统调用编号: 39
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### sendfile
#### 解释
- 系统调用编号: 40
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### socket
#### 解释
- 系统调用编号: 41
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### connect
#### 解释
- 系统调用编号: 42
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### accept
#### 解释
- 系统调用编号: 43
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### sendto
#### 解释
- 系统调用编号: 44
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### recvfrom
#### 解释
- 系统调用编号: 45
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### sendmsg
#### 解释
- 系统调用编号: 46
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### recvmsg
#### 解释
- 系统调用编号: 47
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### shutdown
#### 解释
- 系统调用编号: 48
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### bind
#### 解释
- 系统调用编号: 49
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### listen
#### 解释
- 系统调用编号: 50
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### getsockname
#### 解释
- 系统调用编号: 51
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### getpeername
#### 解释
- 系统调用编号: 52
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### socketpair
#### 解释
- 系统调用编号: 53
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 54
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### getsocktopt
#### 解释
- 系统调用编号: 55
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### clone
#### 解释
- 系统调用编号: 57
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### fork
#### 解释
- 系统调用编号: 58
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### execve
#### 解释
- 系统调用编号: 59
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### exit
#### 解释
- 系统调用编号: 60
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### wait4
#### 解释
- 系统调用编号: 61
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### kill
#### 解释
- 系统调用编号: 62
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### uname
#### 解释
- 系统调用编号: 63
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### semget
#### 解释
- 系统调用编号: 64
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### semop
#### 解释
- 系统调用编号: 65
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### semctl
#### 解释
- 系统调用编号: 66
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### shmdt
#### 解释
- 系统调用编号: 67
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### msgget
#### 解释
- 系统调用编号: 68
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### msgsnd
#### 解释
- 系统调用编号: 69
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### msgrcv
#### 解释
- 系统调用编号: 70
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### msgctl
#### 解释
- 系统调用编号: 71
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### fcntl
#### 解释
- 系统调用编号: 72
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### flock
#### 解释
- 系统调用编号: 73
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### fsync
#### 解释
- 系统调用编号: 74
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### fdatesync
#### 解释
- 系统调用编号: 75
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### truncate
#### 解释
- 系统调用编号: 76
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### ftruncate
#### 解释
- 系统调用编号: 77
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### getdents
#### 解释
- 系统调用编号: 78
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### getcwd
#### 解释
- 系统调用编号: 79
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### chdir
#### 解释
- 系统调用编号: 80
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### fchdir
#### 解释
- 系统调用编号: 81
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### rename
#### 解释
- 系统调用编号: 82
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### mkdir
#### 解释
- 系统调用编号: 83
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### rmdir
#### 解释
- 系统调用编号: 84
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### creat
#### 解释
- 系统调用编号: 85
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### link
#### 解释
- 系统调用编号: 86
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### unlink
#### 解释
- 系统调用编号: 87
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### symlink
#### 解释
- 系统调用编号: 88
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### readlink
#### 解释
- 系统调用编号: 89
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### chmod
#### 解释
- 系统调用编号: 90
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### fchmod
#### 解释
- 系统调用编号: 91
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### chown
#### 解释
- 系统调用编号: 92
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### fchown
#### 解释
- 系统调用编号: 93
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### lchown
#### 解释
- 系统调用编号: 94
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### umask
#### 解释
- 系统调用编号: 95
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### gettimeofday
#### 解释
- 系统调用编号: 96
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### getrusage
#### 解释
- 系统调用编号: 98
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### sysinfo
#### 解释
- 系统调用编号: 99
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### times
#### 解释
- 系统调用编号: 100
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### ptrace
#### 解释
- 系统调用编号: 101
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### getuid
#### 解释
- 系统调用编号: 102
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### syslog
#### 解释
- 系统调用编号: 103
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### getid
#### 解释
- 系统调用编号: 104
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c

```

### setuid
#### 解释
- 系统调用编号: 105
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setgid
#### 解释
- 系统调用编号: 106
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### geteuid
#### 解释
- 系统调用编号: 107
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### getegid
#### 解释
- 系统调用编号: 108
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### getpgid
#### 解释
- 系统调用编号: 109
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### getppid
#### 解释
- 系统调用编号: 110
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### getpgrp
#### 解释
- 系统调用编号: 111
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsid
#### 解释
- 系统调用编号: 112
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setreuid
#### 解释
- 系统调用编号: 113
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setregid
#### 解释
- 系统调用编号: 114
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### getgroups
#### 解释
- 系统调用编号: 115
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setgroups
#### 解释
- 系统调用编号: 116
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setresuid
#### 解释
- 系统调用编号: 117
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### getresuid
#### 解释
- 系统调用编号: 118
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setresgid
#### 解释
- 系统调用编号: 119
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### getresgid
#### 解释
- 系统调用编号: 120
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### getpgid
#### 解释
- 系统调用编号: 121
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setfsuid
#### 解释
- 系统调用编号: 122
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setfsgid
#### 解释
- 系统调用编号: 123
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### getsid
#### 解释
- 系统调用编号: 124
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### capget
#### 解释
- 系统调用编号: 125
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### capset
#### 解释
- 系统调用编号: 126
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### rt_sigpending
#### 解释
- 系统调用编号: 127
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### rt_sigtimedwait
#### 解释
- 系统调用编号: 128
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### rt_sigqueueinfo
#### 解释
- 系统调用编号: 129
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### rt_sigsuspend
#### 解释
- 系统调用编号: 130
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### sigaltstack
#### 解释
- 系统调用编号: 131
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### utime
#### 解释
- 系统调用编号: 132
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### mknod
#### 解释
- 系统调用编号: 133
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### uselib
#### 解释
- 系统调用编号: 134
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### personality
#### 解释
- 系统调用编号: 135
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### setsocktopt
#### 解释
- 系统调用编号: 136
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### setsocktopt
#### 解释
- 系统调用编号: 137
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 138
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 139
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 140
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 141
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 142
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 143
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 144
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 145
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 146
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```
### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```


### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```

### setsocktopt
#### 解释
- 系统调用编号: 147
- 函数原型
```c
int open(const char *pathname, int flags, mode_t mode);
```
- 功能
  -
- 注意
  -
- 使用场景
  -
#### 示例:
```c


```




## ARM64
## MIPS

# 参考资料
```shell
编号 系统调用 函数原型 功能简述 注意事项 典型用途
------ -------- -------- -------- -------- --------
0 read ssize_t read(int fd, void buf, size_t count); 从文件描述符读取数据 可能返回少于请求字节数；EINTR 可重试 读文件、stdin、socket
1 write ssize_t write(int fd, const void buf, size_t count); 向文件描述符写入数据 pipe/socket 可能部分写入 写日志、网络发送
2 open int open(const char pathname, int flags, mode_t mode); 打开或创建文件 已被 openat 取代；mode 仅在 O_CREAT 时有效 打开普通文件
3 close int close(int fd); 关闭文件描述符 失败可能因延迟写错误 文件操作收尾
4 stat int stat(const char path, struct stat buf); 获取文件状态（路径） 解引用符号链接 检查文件属性
5 fstat int fstat(int fd, struct stat buf); 获取文件状态（fd） 不受路径变更影响 已打开文件的状态查询
6 lstat int lstat(const char path, struct stat buf); 获取文件状态（不解析符号链接） 对符号链接本身操作 实现 ls -l
7 poll int poll(struct pollfd fds, nfds_t nfds, int timeout); I/O 多路复用（poll） O(n) 复杂度；已被 epoll 取代 中小规模 I/O 监控
8 lseek off_t lseek(int fd, off_t offset, int whence); 移动文件偏移量 对 pipe/socket 无效 随机访问文件
9 mmap void mmap(void addr, size_t len, int prot, int flags, int fd, off_t off); 映射文件或内存到地址空间 off 必须页对齐 共享内存、大文件访问
10 mprotect int mprotect(void addr, size_t len, int prot); 修改内存保护属性 addr 需页对齐 JIT、内存权限控制
11 munmap int munmap(void addr, size_t length); 解除内存映射 地址和长度需匹配 mmap 释放映射内存
12 brk int brk(void addr); 设置程序断点（堆顶） 现代 malloc 多用 mmap 极少直接使用
13 rt_sigaction int rt_sigaction(int sig, const struct sigaction act, struct sigaction oldact, size_t sigsetsize); 设置信号处理函数 glibc 封装为 sigaction() 自定义信号响应
14 rt_sigprocmask int rt_sigprocmask(int how, const sigset_t set, sigset_t oldset, size_t sigsetsize); 修改信号掩码 封装为 sigprocmask() 临界区屏蔽信号
15 rt_sigreturn long rt_sigreturn(void); 从信号处理函数返回 由内核自动调用 信号处理内部机制
16 ioctl int ioctl(int fd, unsigned long request, ...); 设备控制操作 request 依赖设备驱动 终端控制、磁盘参数设置
17 pread64 ssize_t pread64(int fd, void buf, size_t count, off64_t offset); 从指定偏移读取（不移动指针） 原子操作（对 regular file） 多线程安全文件读
18 pwrite64 ssize_t pwrite64(int fd, const void buf, size_t count, off64_t offset); 向指定偏移写入（不移动指针） 原子操作 多线程安全文件写
19 readv ssize_t readv(int fd, const struct iovec iov, int iovcnt); 分散读（多个缓冲区） 减少 syscall 次数 高效网络/文件读
20 writev ssize_t writev(int fd, const struct iovec iov, int iovcnt); 集中写（多个缓冲区） 同上 高效网络/文件写
21 access int access(const char path, int mode); 检查文件访问权限 使用真实 UID/GID（非 effective） 安全检查（如 shell）
22 pipe int pipe(int pipefd[2]); 创建匿名管道 pipefd[0]=读, [1]=写 shell 管道、父子通信
23 select int select(int nfds, fd_set readfds, fd_set writefds, fd_set exceptfds, struct timeval timeout); I/O 多路复用（select） O(n) 且有 fd 数量限制 老旧代码兼容
24 sched_yield int sched_yield(void); 主动让出 CPU 无阻塞 协作式调度
25 mremap void mremap(void old_addr, size_t old_size, size_t new_size, int flags, ...); 重新映射虚拟内存 可扩展/收缩 mmap 区域 动态调整内存映射
26 mincore int mincore(void addr, size_t length, unsigned char vec); 查询内存页是否在物理内存中 用于预取优化 内存局部性分析
27 madvise int madvise(void addr, size_t length, int advice); 建议内核如何处理内存 如 MADV_DONTNEED 释放页 内存使用提示
28 shmget int shmget(key_t key, size_t size, int shmflg); 获取 System V 共享内存 ID 已被 POSIX shm 取代 老旧 IPC
29 shmat void shmat(int shmid, const void shmaddr, int shmflg); 附加 System V 共享内存 返回映射地址 老旧共享内存使用
30 shmctl int shmctl(int shmid, int cmd, struct shmid_ds buf); 控制 System V 共享内存 如 IPC_RMID 删除 共享内存管理
31 dup int dup(int oldfd); 复制文件描述符 返回最小可用 fd fd 重定向
32 dup2 int dup2(int oldfd, int newfd); 复制 fd 到指定编号 若 newfd 已打开则先关闭 标准输入/输出重定向
33 pause int pause(void); 等待信号 永久阻塞直到信号 信号等待循环
34 nanosleep int nanosleep(const struct timespec req, struct timespec rem); 高精度睡眠 可被信号中断 精确延时
35 getitimer int getitimer(int which, struct itimerval curr_value); 获取间隔定时器值 which: ITIMER_REAL/VIRTUAL/PROF 老旧定时器查询
36 alarm unsigned int alarm(unsigned int seconds); 设置 SIGALRM 定时器 秒级精度；覆盖前一个 简单超时（老旧）
37 setitimer int setitimer(int which, const struct itimerval new_val, struct itimerval old_val); 设置间隔定时器 微秒级；可周期触发 性能采样、超时
38 getpid pid_t getpid(void); 获取当前进程 ID 恒定不变 进程标识
39 sendfile ssize_t sendfile(int out_fd, int in_fd, off_t offset, size_t count); 零拷贝文件传输 仅支持 file → socket 高效静态文件服务
40 socket int socket(int domain, int type, int protocol); 创建套接字 domain: AF_INET/AF_UNIX 网络编程起点
41 connect int connect(int sockfd, const struct sockaddr addr, socklen_t addrlen); 连接远程地址 TCP 阻塞直到完成 客户端连接
42 accept int accept(int sockfd, struct sockaddr addr, socklen_t addrlen); 接受连接 返回新通信 fd 服务器接收连接
43 sendto ssize_t sendto(int sockfd, const void buf, size_t len, int flags, const struct sockaddr dest_addr, socklen_t addrlen); 发送数据（可指定目标） UDP 常用 无连接通信
44 recvfrom ssize_t recvfrom(int sockfd, void buf, size_t len, int flags, struct sockaddr src_addr, socklen_t addrlen); 接收数据（可获取来源） UDP 常用 无连接通信
45 sendmsg ssize_t sendmsg(int sockfd, const struct msghdr msg, int flags); 发送结构化消息 支持 SCM_RIGHTS 传递 fd 高级 socket 操作
46 recvmsg ssize_t recvmsg(int sockfd, struct msghdr msg, int flags); 接收结构化消息 同上 高级 socket 操作
47 shutdown int shutdown(int sockfd, int how); 关闭套接字部分功能 how: SHUT_RD/WR/RDWR 优雅关闭连接
48 bind int bind(int sockfd, const struct sockaddr addr, socklen_t addrlen); 绑定本地地址 服务端必须调用 监听特定 IP/端口
49 listen int listen(int sockfd, int backlog); 开始监听连接 backlog 为队列长度 服务端准备就绪
50 getsockname int getsockname(int sockfd, struct sockaddr addr, socklen_t addrlen); 获取本地地址 用于查询绑定的端口 动态端口获取
51 getpeername int getpeername(int sockfd, struct sockaddr addr, socklen_t addrlen); 获取对端地址 用于日志记录 客户端信息获取
52 socketpair int socketpair(int domain, int type, int protocol, int sv[2]); 创建一对连接的套接字 常用于本地通信 线程/进程间通信
53 setsockopt int setsockopt(int sockfd, int level, int optname, const void optval, socklen_t optlen); 设置套接字选项 如 SO_REUSEADDR 网络调优
54 getsockopt int getsockopt(int sockfd, int level, int optname, void optval, socklen_t optlen); 获取套接字选项 调试、状态查询 网络诊断
55 clone int clone(int (fn)(void ), void stack, int flags, void arg, ...); 创建线程或轻量进程 栈需手动分配 pthread 底层、容器
56 fork pid_t fork(void); 创建子进程（完整复制） COW 优化 并发、守护进程
57 vfork pid_t vfork(void); 创建子进程（共享地址空间） 子进程必须 exec 或 _exit 已弃用（危险）
58 execve int execve(const char filename, char const argv[], char const envp[]); 执行新程序 成功不返回 shell 执行命令
59 exit void exit(int status); 终止进程（调用清理函数） 实际调用 _exit 前 flush stdio 正常退出
60 wait4 pid_t wait4(pid_t pid, int wstatus, int options, struct rusage rusage); 等待子进程并获取资源统计 wait/waitpid 是其封装 回收僵尸进程
61 kill int kill(pid_t pid, int sig); 向进程发送信号 pid=-1: 所有进程（除 init） 进程控制
62 uname int uname(struct utsname buf); 获取系统信息 包含内核版本、主机名等 软件兼容性检测
63 semget int semget(key_t key, int nsems, int semflg); 获取 System V 信号量集 老旧 IPC 进程同步（老旧）
64 semop int semop(int semid, struct sembuf sops, size_t nsops); 操作 System V 信号量 原子 P/V 操作 老旧同步
65 semctl int semctl(int semid, int semnum, int cmd, ...); 控制 System V 信号量 如 SETVAL 初始化 信号量管理
66 shmdt int shmdt(const void shmaddr); 分离 System V 共享内存 与 shmat 配对 老旧共享内存释放
67 msgget int msgget(key_t key, int msgflg); 获取 System V 消息队列 老旧 IPC 消息传递（老旧）
68 msgsnd int msgsnd(int msqid, const void msgp, size_t msgsz, int msgflg); 发送消息 消息大小有限制 老旧消息队列
69 msgrcv ssize_t msgrcv(int msqid, void msgp, size_t msgsz, long msgtyp, int msgflg); 接收消息 可按类型接收 老旧消息队列
70 msgctl int msgctl(int msqid, int cmd, struct msqid_ds buf); 控制消息队列 如 IPC_RMID 删除 消息队列管理
71 fcntl int fcntl(int fd, int cmd, ...); 文件描述符控制 如 F_SETFD/F_SETFL fd 属性修改
72 flock int flock(int fd, int operation); 对文件加建议性锁 非强制锁 文件互斥访问
73 fsync int fsync(int fd); 同步文件数据和元数据到磁盘 阻塞直到完成 数据持久化
74 fdatasync int fdatasync(int fd); 同步文件数据（不含元数据） 比 fsync 快 日志写入优化
75 truncate int truncate(const char path, off_t length); 截断文件（路径） 扩展时填充零 文件大小调整
76 ftruncate int ftruncate(int fd, off_t length); 截断文件（fd） 同上 已打开文件截断
77 getdents int getdents(unsigned int fd, struct dirent dirp, unsigned int count); 读取目录项（32位） 已被 getdents64 取代 老旧目录遍历
78 getcwd char getcwd(char buf, size_t size); 获取当前工作目录 buf 可为 NULL（自动分配） 路径解析
79 chdir int chdir(const char path); 改变当前工作目录 影响所有线程 目录切换
80 fchdir int fchdir(int fd); 通过 fd 改变工作目录 更安全（避免 TOCTOU） 安全路径操作
81 rename int rename(const char oldpath, const char newpath); 重命名文件或目录 原子操作 文件移动/重命名
82 mkdir int mkdir(const char pathname, mode_t mode); 创建目录 mode 受 umask 影响 目录创建
83 rmdir int rmdir(const char pathname); 删除空目录 目录必须为空 目录删除
84 creat int creat(const char pathname, mode_t mode); 创建并打开文件 等价于 open(O_CREAT\ O_WRONLY\ O_TRUNC) 已废弃（用 open 替代）
85 link int link(const char oldpath, const char newpath); 创建硬链接 不能跨文件系统 文件别名
86 unlink int unlink(const char pathname); 删除文件链接 文件内容在无链接后释放 文件删除
87 symlink int symlink(const char target, const char linkpath); 创建符号链接 target 可不存在 软链接创建
88 readlink ssize_t readlink(const char path, char buf, size_t bufsiz); 读取符号链接内容 不追加 '\0' 解析软链接
89 chmod int chmod(const char path, mode_t mode); 改变文件权限（路径） 需要文件所有者或 root 权限修改
90 fchmod int fchmod(int fd, mode_t mode); 改变文件权限（fd） 更安全 已打开文件权限修改
91 chown int chown(const char path, uid_t owner, gid_t group); 改变文件所有者（路径） 需 root 或自己文件 所有权转移
92 fchown int fchown(int fd, uid_t owner, gid_t group); 改变文件所有者（fd） 更安全 已打开文件所有权修改
93 lchown int lchown(const char path, uid_t owner, gid_t group); 改变符号链接所有者 不解引用链接 符号链接所有权
94 umask mode_t umask(mode_t mask); 设置文件创建掩码 影响后续 open/mkdir 权限默认值控制
95 gettimeofday int gettimeofday(struct timeval tv, struct timezone tz); 获取时间（微秒） tz 参数已废弃 老旧高精度时间
96 getrlimit int getrlimit(int resource, struct rlimit rlim); 获取资源限制 如 RLIMIT_NOFILE 资源查询
97 setrlimit int setrlimit(int resource, const struct rlimit rlim); 设置资源限制 需特权（部分） 资源限制（如 ulimit）
98 getrusage int getrusage(int who, struct rusage usage); 获取资源使用统计 who: RUSAGE_SELF/CHILDREN 性能分析
99 sysinfo int sysinfo(struct sysinfo info); 获取系统信息 包含内存、负载 系统监控
100 times clock_t times(struct tms buf); 获取进程时间 已被 clock_gettime 取代 老旧时间统计
101 ptrace long ptrace(enum __ptrace_request request, pid_t pid, void addr, void data); 进程跟踪与调试 调试器/strace 底层 调试、沙箱
102 getuid uid_t getuid(void); 获取真实用户 ID 恒定不变 用户身份识别
103 syslog int syslog(int type, char bufp, int len); 读取/控制内核日志 需特权 内核日志访问
104 getgid gid_t getgid(void); 获取真实组 ID 恒定不变 组身份识别
105 setuid int setuid(uid_t uid); 设置用户 ID 需特权或 setuid 程序 降权运行
106 setgid int setgid(gid_t gid); 设置组 ID 同上 组切换
107 geteuid uid_t geteuid(void); 获取有效用户 ID 决定文件访问权限 安全上下文判断
108 getegid gid_t getegid(void); 获取有效组 ID 同上 安全上下文判断
109 setpgid int setpgid(pid_t pid, pid_t pgid); 设置进程组 ID 用于作业控制 shell 作业管理
110 getppid pid_t getppid(void); 获取父进程 ID 父进程退出后变为 1 进程树遍历
111 getpgrp pid_t getpgrp(void); 获取当前进程组 ID 同 getpgid(0) 作业控制
112 setsid pid_t setsid(void); 创建新会话 调用者成为会话首进程 守护进程创建
113 setreuid int setreuid(uid_t ruid, uid_t euid); 设置真实和有效 UID 权限切换 特权分离
114 setregid int setregid(gid_t rgid, gid_t egid); 设置真实和有效 GID 同上 组权限切换
115 getgroups int getgroups(int size, gid_t list[]); 获取补充组列表 size=0 可查询数量 用户组查询
116 setgroups int setgroups(size_t size, const gid_t list); 设置补充组列表 需特权 切换用户身份（如 sudo）
117 setresuid int setresuid(uid_t ruid, uid_t euid, uid_t suid); 设置三类 UID 更细粒度控制 安全敏感程序
118 getresuid int getresuid(uid_t ruid, uid_t euid, uid_t suid); 获取三类 UID 审计用途 权限审计
119 setresgid int setresgid(gid_t rgid, gid_t egid, gid_t sgid); 设置三类 GID 同上 组权限精细控制
120 getresgid int getresgid(gid_t rgid, gid_t egid, gid_t sgid); 获取三类 GID 同上 组权限审计
121 getpgid pid_t getpgid(pid_t pid); 获取进程组 ID 作业控制 进程组查询
122 setfsuid int setfsuid(uid_t fsuid); 设置文件系统 UID 仅用于文件权限检查 NFS 等特殊场景
123 setfsgid int setfsgid(gid_t fsgid); 设置文件系统 GID 同上 特殊文件系统权限
124 getsid pid_t getsid(pid_t pid); 获取会话 ID 作业控制 会话查询
125 capget int capget(cap_user_header_t hdrp, cap_user_data_t datap); 获取进程能力集 需 <sys/capability.h> 能力检查
126 capset int capset(cap_user_header_t hdrp, const cap_user_data_t datap); 设置进程能力集 最小权限原则 容器、特权分离
127 rt_sigpending int rt_sigpending(sigset_t set, size_t sigsetsize); 查询挂起信号 封装为 sigpending() 信号状态查询
128 rt_sigtimedwait int rt_sigtimedwait(const sigset_t set, siginfo_t info, const struct timespec timeout, size_t sigsetsize); 等待信号（带超时） 封装为 sigtimedwait() 带超时的信号等待
129 rt_sigqueueinfo int rt_sigqueueinfo(pid_t pid, int sig, siginfo_t uinfo); 向进程排队发送信号（带数据） 实时信号扩展 进程间传递数据
130 rt_sigsuspend int rt_sigsuspend(const sigset_t mask, size_t sigsetsize); 暂时替换掩码并挂起 封装为 sigsuspend() 信号等待原子操作
131 sigaltstack int sigaltstack(const stack_t ss, stack_t oss); 设置/获取备用信号栈 防止栈溢出 信号安全处理
132 utime int utime(const char filename, const struct utimbuf times); 设置文件访问/修改时间 已被 utimensat 取代 老旧时间戳修改
133 mknod int mknod(const char pathname, mode_t mode, dev_t dev); 创建设备节点或普通文件 需特权（设备节点） 设备文件创建
134 uselib int uselib(const char library); 加载共享库 已废弃（用 dlopen） 老旧动态链接
135 personality int personality(unsigned long persona); 设置进程执行域 兼容其他 Unix 特殊兼容模式
136 ustat int ustat(dev_t dev, struct ustat ubuf); 获取文件系统信息 已废弃 老旧文件系统查询
137 statfs int statfs(const char path, struct statfs buf); 获取文件系统状态 如块大小、空闲空间 磁盘空间查询
138 fstatfs int fstatfs(int fd, struct statfs buf); 获取文件系统状态（fd） 更安全 已打开文件系统查询
139 sysfs int sysfs(int option, ...); 查询内核对象信息 已废弃（用 /sys） 老旧内核信息查询
140 getpriority int getpriority(int which, int who); 获取进程调度优先级 which: PRIO_PROCESS/GROUP/USER 调度策略查询
141 setpriority int setpriority(int which, int who, int prio); 设置进程调度优先级 需特权（提升优先级） 进程优先级调整
142 sched_setparam int sched_setparam(pid_t pid, const struct sched_param param); 设置调度参数 用于 SCHED_FIFO/RR 实时调度
143 sched_getparam int sched_getparam(pid_t pid, struct sched_param param); 获取调度参数 同上 实时调度查询
144 sched_setscheduler int sched_setscheduler(pid_t pid, int policy, const struct sched_param param); 设置调度策略 如 SCHED_OTHER/FIFO/RR 调度策略切换
145 sched_getscheduler int sched_getscheduler(pid_t pid); 获取调度策略 返回策略编号 调度策略查询
146 sched_get_priority_max int sched_get_priority_max(int policy); 获取策略最大优先级 实时调度范围 调度参数配置
147 sched_get_priority_min int sched_get_priority_min(int policy); 获取策略最小优先级 同上 调度参数配置
148 sched_rr_get_interval int sched_rr_get_interval(pid_t pid, struct timespec tp); 获取 SCHED_RR 时间片 实时调度 时间片查询
149 mlock int mlock(const void addr, size_t len); 锁定内存页（防止换出） 需特权或 CAP_IPC_LOCK 实时/安全应用
150 munlock int munlock(const void addr, size_t len); 解锁内存页 与 mlock 配对 内存解锁
151 mlockall int mlockall(int flags); 锁定所有内存页 flags: MCL_CURRENT/MCL_FUTURE 全局内存锁定
152 munlockall int munlockall(void); 解锁所有内存页 与 mlockall 配对 全局内存解锁
153 vhangup int vhangup(void); 挂断终端 需特权 终端控制（极少用）
154 modify_ldt int modify_ldt(int func, void ptr, unsigned long bytecount); 修改局部描述符表 x86 特有；Wine 使用 16位兼容（极少用）
155 pivot_root int pivot_root(const char new_root, const char put_old); 切换根文件系统 容器启动关键步骤 容器、initramfs
156 _sysctl int _sysctl(struct __sysctl_args args); 读写内核参数 已废弃（用 /proc/sys） 老旧内核参数调整
157 prctl int prctl(int option, unsigned long arg2, unsigned long arg3, unsigned long arg4, unsigned long arg5); 控制进程行为 如 PR_SET_NAME, PR_SET_DUMPABLE 进程属性控制
158 arch_prctl int arch_prctl(int code, unsigned long addr); x86-64 架构控制 如 ARCH_SET_FS 设置 TLS 线程本地存储初始化
159 adjtimex int adjtimex(struct timex tx); 调整内核时钟 NTP 守护进程使用 系统时间校准
160 setrlimit (同 97) 设置资源限制 重复条目（历史原因） —
161 chroot int chroot(const char path); 改变根目录 需特权；不改变 cwd 沙箱、容器
162 sync void sync(void); 将所有缓存写入磁盘 阻塞直到完成 系统关机前调用
163 acct int acct(const char filename); 开启/关闭进程记账 需特权 系统审计
164 settimeofday int settimeofday(const struct timeval tv, const struct timezone tz); 设置系统时间 需特权 时间同步
165 mount int mount(const char source, const char target, const char filesystemtype, unsigned long mountflags, const void data); 挂载文件系统 需特权 文件系统挂载
166 umount2 int umount2(const char target, int flags); 卸载文件系统 flags: MNT_FORCE 等 安全卸载
167 swapon int swapon(const char path, int swapflags); 启用交换分区 需特权 内存扩展
168 swapoff int swapoff(const char path); 禁用交换分区 需特权 交换管理
169 reboot int reboot(int magic1, int magic2, int cmd, void arg); 重启或关闭系统 需特权；magic 值固定 系统关机工具底层
170 sethostname int sethostname(const char name, size_t len); 设置主机名 需特权 系统配置
171 setdomainname int setdomainname(const char name, size_t len); 设置 NIS 域名 需特权 网络配置（老旧）
172 iopl int iopl(int level); 修改 I/O 特权级 x86 特有；需特权 硬件直接访问（危险）
173 ioperm int ioperm(unsigned long from, unsigned long num, int turn_on); 设置 I/O 端口权限 x86 特有；需特权 硬件端口访问
174 create_module void create_module(const char name, size_t size); 创建内核模块 已废弃（用 init_module） 老旧模块加载
175 init_module long init_module(void module_image, unsigned long len, const char param_values); 加载内核模块 需特权 insmod 底层
176 delete_module long delete_module(const char name_user, unsigned int flags); 卸载内核模块 需特权 rmmod 底层
177 get_kernel_syms long get_kernel_syms(struct kernel_sym table); 获取内核符号表 已废弃 老旧调试
178 query_module long query_module(const char name, int which, void buf, size_t bufsize, size_t ret); 查询模块信息 已废弃 老旧模块查询
179 quotactl int quotactl(unsigned int cmd, const char special, int id, caddr_t addr); 磁盘配额控制 需特权 配额管理
180 nfsservctl int nfsservctl(int cmd, struct nfsctl_arg argp, void resp); NFS 服务器控制 已废弃 老旧 NFS
181 getpmsg int getpmsg(int fd, struct strbuf ctlptr, struct strbuf dataptr, int bandp, int flagsp); STREAMS 消息接收 已废弃（非 Linux 标准） 老旧 STREAMS
182 putpmsg int putpmsg(int fd, const struct strbuf ctlptr, const struct strbuf dataptr, int band, int flags); STREAMS 消息发送 已废弃 老旧 STREAMS
183 afs_syscall long afs_syscall(void); AFS 文件系统调用 已废弃 老旧 AFS
184 tuxcall long tuxcall(void); TUX Web 服务器调用 已废弃 老旧 TUX
185 security long security(void); 安全模块调用 已废弃（用 LSM hooks） 老旧安全框架
186 gettid pid_t gettid(void); 获取当前线程 ID 每个线程唯一 线程标识
187 readahead ssize_t readahead(int fd, off64_t offset, size_t count); 预读文件数据 提示内核预加载 I/O 性能优化
188 setxattr int setxattr(const char path, const char name, const void value, size_t size, int flags); 设置扩展属性 如 user.mime_type 文件元数据扩展
189 lsetxattr int lsetxattr(const char path, const char name, const void value, size_t size, int flags); 设置扩展属性（不解析链接） 同上 符号链接属性设置
190 fsetxattr int fsetxattr(int fd, const char name, const void value, size_t size, int flags); 设置扩展属性（fd） 更安全 已打开文件属性设置
191 getxattr ssize_t getxattr(const char path, const char name, void value, size_t size); 获取扩展属性 同上 扩展属性读取
192 lgetxattr ssize_t lgetxattr(const char path, const char name, void value, size_t size); 获取扩展属性（不解析链接） 同上 符号链接属性读取
193 fgetxattr ssize_t fgetxattr(int fd, const char name, void value, size_t size); 获取扩展属性（fd） 更安全 已打开文件属性读取
194 listxattr ssize_t listxattr(const char path, char list, size_t size); 列出扩展属性 返回属性名列表 扩展属性枚举
195 llistxattr ssize_t llistxattr(const char path, char list, size_t size); 列出扩展属性（不解析链接） 同上 符号链接属性枚举
196 flistxattr ssize_t flistxattr(int fd, char list, size_t size); 列出扩展属性（fd） 更安全 已打开文件属性枚举
197 removexattr int removexattr(const char path, const char name); 删除扩展属性 同上 扩展属性删除
198 lremovexattr int lremovexattr(const char path, const char name); 删除扩展属性（不解析链接） 同上 符号链接属性删除
199 fremovexattr int fremovexattr(int fd, const char name); 删除扩展属性（fd） 更安全 已打开文件属性删除
200 tkill int tkill(int tid, int sig); 向线程发送信号 已被 tgkill 取代 老旧线程信号
201 time time_t time(time_t tloc); 获取时间（秒） 已被 clock_gettime 取代 老旧时间获取
202 futex long futex(uint32_t uaddr, int op, uint32_t val, const struct timespec timeout, uint32_t uaddr2, uint32_t val3); 快速用户空间互斥锁 pthread mutex 底层 线程同步原语
203 sched_setaffinity int sched_setaffinity(pid_t pid, size_t cpusetsize, const cpu_set_t mask); 设置 CPU 亲和性 绑定线程到 CPU 性能调优
204 sched_getaffinity int sched_getaffinity(pid_t pid, size_t cpusetsize, cpu_set_t mask); 获取 CPU 亲和性 同上 CPU 绑定查询
205 set_thread_area int set_thread_area(struct user_desc u_info); 设置线程局部存储 x86 32位；64位用 arch_prctl TLS 初始化（32位）
206 io_setup int io_setup(unsigned nr_events, aio_context_t ctx_idp); 初始化 AIO 上下文 异步 I/O（POSIX AIO） 老旧异步 I/O
207 io_destroy int io_destroy(aio_context_t ctx_id); 销毁 AIO 上下文 与 io_setup 配对 AIO 资源释放
208 io_getevents int io_getevents(aio_context_t ctx_id, long min_nr, long nr, struct io_event events, struct timespec timeout); 获取 AIO 事件 轮询完成事件 AIO 结果获取
209 io_submit int io_submit(aio_context_t ctx_id, long nr, struct iocb iocbs[]); 提交 AIO 请求 批量提交 AIO 请求发起
210 io_cancel int io_cancel(aio_context_t ctx_id, struct iocb iocb, struct io_event result); 取消 AIO 请求 可能失败 AIO 请求取消
211 get_thread_area int get_thread_area(struct user_desc u_info); 获取线程局部存储 x86 32位 TLS 查询（32位）
212 lookup_dcookie int lookup_dcookie(u64 cookie, char buffer, size_t len); 通过 cookie 查找路径 oprofile 使用 性能分析工具
213 epoll_create int epoll_create(int size); 创建 epoll 实例 size 被忽略；用 epoll_create1 老旧 epoll 创建
214 epoll_ctl_old long epoll_ctl_old(void); 旧版 epoll 控制 已废弃 —
215 epoll_wait_old long epoll_wait_old(void); 旧版 epoll 等待 已废弃 —
216 remap_file_pages long remap_file_pages(void addr, size_t size, int prot, size_t pgoff, int flags); 重映射文件页 已废弃 老旧内存映射优化
217 getdents64 int getdents64(unsigned int fd, struct dirent64 dirp, unsigned int count); 读取目录项（64位） readdir() 底层 目录遍历
218 set_tid_address long set_tid_address(int tidptr); 设置线程 ID 地址 clone 时使用 线程退出通知
219 restart_syscall long restart_syscall(void); 重启被中断的系统调用 内核内部使用 信号处理内部机制
220 semtimedop int semtimedop(int semid, struct sembuf sops, size_t nsops, const struct timespec timeout); 带超时的信号量操作 System V 信号量扩展 老旧同步（带超时）
221 fadvise64 int fadvise64(int fd, off_t offset, off_t len, int advice); 文件访问建议 如 POSIX_FADV_WILLNEED I/O 性能提示
222 timer_create int timer_create(clockid_t clockid, struct sigevent sevp, timer_t timerid); 创建 POSIX 定时器 信号或线程通知 高精度定时
223 timer_settime int timer_settime(timer_t timerid, int flags, const struct itimerspec new_value, struct itimerspec old_value); 设置 POSIX 定时器 同上 定时器启动/修改
224 timer_gettime int timer_gettime(timer_t timerid, struct itimerspec curr_value); 获取 POSIX 定时器值 同上 定时器查询
225 timer_getoverrun int timer_getoverrun(timer_t timerid); 获取 POSIX 定时器溢出次数 同上 定时器精度分析
226 timer_delete int timer_delete(timer_t timerid); 删除 POSIX 定时器 同上 定时器销毁
227 clock_settime int clock_settime(clockid_t clk_id, const struct timespec tp); 设置时钟时间 需特权（部分时钟） 系统时间设置
228 clock_gettime int clock_gettime(clockid_t clk_id, struct timespec tp); 获取高精度时间 推荐 CLOCK_MONOTONIC 性能计时、超时
229 clock_getres int clock_getres(clockid_t clk_id, struct timespec res); 获取时钟分辨率 同上 时间精度查询
230 clock_nanosleep int clock_nanosleep(clockid_t clk_id, int flags, const struct timespec rqtp, struct timespec rmtp); 基于指定时钟的睡眠 同上 精确延时
231 exit_group void exit_group(int status); 终止整个线程组 glibc exit() 底层 进程退出
232 epoll_wait int epoll_wait(int epfd, struct epoll_event events, int maxevents, int timeout); 等待 epoll 事件 高效 O(1) 通知 事件驱动 I/O
233 epoll_ctl int epoll_ctl(int epfd, int op, int fd, struct epoll_event event); 控制 epoll 监听集合 op: ADD/MOD/DEL epoll 注册/修改
234 tgkill int tgkill(int tgid, int tid, int sig); 向指定线程发送信号 tgid=进程ID, tid=线程ID 精确线程控制
235 utimes int utimes(const char filename, const struct timeval times[2]); 设置文件时间戳（微秒） 已被 utimensat 取代 老旧时间戳修改
236 vserver long vserver(void); Linux-VServer 调用 已废弃 老旧容器技术
237 mbind long mbind(void addr, unsigned long len, int mode, const unsigned long nodemask, unsigned long maxnode, unsigned flags); 设置内存绑定策略 NUMA 优化 NUMA 内存分配
238 set_mempolicy long set_mempolicy(int mode, const unsigned long nodemask, unsigned long maxnode); 设置内存策略 NUMA 优化 进程级 NUMA 策略
239 get_mempolicy long get_mempolicy(int mode, unsigned long nodemask, unsigned long maxnode, void addr, unsigned long flags); 获取内存策略 NUMA 查询 NUMA 策略查询
240 mq_open mqd_t mq_open(const char name, int oflag, mode_t mode, struct mq_attr attr); 打开 POSIX 消息队列 需挂载 mqueue 现代消息队列
241 mq_unlink int mq_unlink(const char name); 删除 POSIX 消息队列 同上 消息队列删除
242 mq_timedsend int mq_timedsend(mqd_t mqdes, const char msg_ptr, size_t msg_len, unsigned int msg_prio, const struct timespec abs_timeout); 发送 POSIX 消息（带超时） 同上 消息队列发送
243 mq_timedreceive ssize_t mq_timedreceive(mqd_t mqdes, char msg_ptr, size_t msg_len, unsigned int msg_prio, const struct timespec abs_timeout); 接收 POSIX 消息（带超时） 同上 消息队列接收
244 mq_notify int mq_notify(mqd_t mqdes, const struct sigevent sevp); 注册消息到达通知 同上 异步消息通知
245 mq_getsetattr int mq_getsetattr(mqd_t mqdes, const struct mq_attr mqstat, struct mq_attr omqstat); 获取/设置消息队列属性 同上 消息队列配置
246 kexec_load long kexec_load(unsigned long entry, unsigned long nr_segments, struct kexec_segment segments, unsigned long flags); 加载新内核以快速重启 需特权 kexec 快速重启
247 waitid int waitid(idtype_t idtype, id_t id, siginfo_t infop, int options); 等待子进程（更灵活） POSIX 标准 子进程等待
248 add_key key_serial_t add_key(const char type, const char description, const void payload, size_t plen, key_serial_t ringid); 添加密钥到密钥环 需 CONFIG_KEYS 内核密钥管理
249 request_key key_serial_t request_key(const char type, const char description, const char callout_info, key_serial_t destringid); 请求密钥 同上 密钥请求
250 keyctl long keyctl(int cmd, unsigned long arg2, unsigned long arg3, unsigned long arg4, unsigned long arg5); 控制密钥环 同上 密钥操作
251 ioprio_set int ioprio_set(int which, int who, int ioprio); 设置 I/O 优先级 需特权 I/O 调度优先级
252 ioprio_get int ioprio_get(int which, int who); 获取 I/O 优先级 同上 I/O 优先级查询
253 inotify_init int inotify_init(void); 创建 inotify 实例 已被 inotify_init1 取代 文件系统事件监控
254 inotify_add_watch int inotify_add_watch(int fd, const char pathname, uint32_t mask); 添加 inotify 监控 同上 监控文件变化
255 inotify_rm_watch int inotify_rm_watch(int fd, int wd); 移除 inotify 监控 同上 停止监控
256 migrate_pages long migrate_pages(int pid, unsigned long maxnode, const unsigned long from, const unsigned long to); 迁移进程内存页 NUMA 优化 内存页迁移
257 openat int openat(int dirfd, const char pathname, int flags, mode_t mode); 在指定目录下打开文件 dirfd=AT_FDCWD 表示当前目录 安全路径操作（避免 TOCTOU）
258 mkdirat int mkdirat(int dirfd, const char pathname, mode_t mode); 在指定目录下创建目录 同上 安全目录创建
259 mknodat int mknodat(int dirfd, const char pathname, mode_t mode, dev_t dev); 在指定目录下创建设备节点 同上 安全设备文件创建
260 fchownat int fchownat(int dirfd, const char pathname, uid_t owner, gid_t group, int flags); 在指定目录下改变文件所有者 flags: AT_SYMLINK_NOFOLLOW 安全所有权修改
261 futimesat int futimesat(int dirfd, const char pathname, const struct timeval times[2]); 在指定目录下设置文件时间 已被 utimensat 取代 老旧安全时间设置
262 fstatat int fstatat(int dirfd, const char pathname, struct stat buf, int flags); 在指定目录下获取文件状态 flags: AT_SYMLINK_NOFOLLOW 安全状态查询
263 unlinkat int unlinkat(int dirfd, const char pathname, int flags); 在指定目录下删除文件 flags: AT_REMOVEDIR 删除目录 安全文件删除
264 renameat int renameat(int olddirfd, const char oldpath, int newdirfd, const char newpath); 在指定目录间重命名 同上 安全重命名
265 linkat int linkat(int olddirfd, const char oldpath, int newdirfd, const char newpath, int flags); 在指定目录间创建硬链接 flags: AT_SYMLINK_FOLLOW 安全硬链接
266 symlinkat int symlinkat(const char target, int newdirfd, const char linkpath); 在指定目录下创建符号链接 同上 安全软链接
267 readlinkat ssize_t readlinkat(int dirfd, const char pathname, char buf, size_t bufsiz); 读取指定目录下的符号链接 同上 安全软链接解析
268 fchmodat int fchmodat(int dirfd, const char pathname, mode_t mode, int flags); 在指定目录下改变文件权限 flags: AT_SYMLINK_NOFOLLOW 安全权限修改
269 faccessat int faccessat(int dirfd, const char pathname, int mode, int flags); 在指定目录下检查访问权限 flags: AT_EACCESS 使用 effective UID 安全权限检查
270 pselect6 int pselect6(int nfds, fd_set readfds, fd_set writefds, fd_set exceptfds, const struct timespec timeout, const sigset_t sigmask); 带信号掩码的 select 原子操作 安全 I/O 多路复用
271 ppoll int ppoll(struct pollfd fds, nfds_t nfds, const struct timespec timeout, const sigset_t sigmask); 带信号掩码的 poll 原子操作 安全 poll
272 unshare int unshare(int flags); 解除部分 namespace 共享 如 CLONE_NEWNS/NEWNET 容器、沙箱
273 set_robust_list long set_robust_list(struct robust_list_head head, size_t len); 设置健壮互斥锁列表 futex 健壮锁支持 线程异常退出锁释放
274 get_robust_list long get_robust_list(int pid, struct robust_list_head *head_ptr, size_t len_ptr); 获取健壮互斥锁列表 同上 健壮锁查询
275 splice ssize_t splice(int fd_in, loff_t off_in, int fd_out, loff_t off_out, size_t len, unsigned int flags); 在两个 fd 间移动数据（零拷贝） pipe 作为中介 高效数据转发
276 tee ssize_t tee(int fd_in, int fd_out, size_t len, unsigned int flags); 复制 pipe 数据（不消耗） fd_in/out 必须是 pipe pipe 数据窥探
277 sync_file_range int sync_file_range(int fd, off64_t offset, off64_t nbytes, unsigned int flags); 同步文件指定范围 flags: SYNC_FILE_RANGE_WRITE 部分文件同步
278 vmsplice ssize_t vmsplice(int fd, const struct iovec iov, unsigned long nr_segs, unsigned int flags); 将用户内存拼接到 pipe 零拷贝 高效 pipe 写入
279 move_pages int move_pages(int pid, unsigned long nr_pages, const void pages, const int nodes, int status, int flags); 移动指定页到 NUMA 节点 NUMA 优化 内存页迁移
280 utimensat int utimensat(int dirfd, const char pathname, const struct timespec times[2], int flags); 设置文件时间戳（纳秒） flags: AT_SYMLINK_NOFOLLOW 现代时间戳修改
281 epoll_create1 int epoll_create1(int flags); 创建 epoll（支持标志） flags: EPOLL_CLOEXEC 现代 epoll 创建
282 dup3 int dup3(int oldfd, int newfd, int flags); 复制 fd 到指定编号（带标志） flags: O_CLOEXEC 安全 fd 复制
283 pipe2 int pipe2(int pipefd[2], int flags); 创建带标志的管道 flags: O_CLOEXEC/NONBLOCK 安全管道创建
284 inotify_init1 int inotify_init1(int flags); 创建带标志的 inotify flags: IN_CLOEXEC/NONBLOCK 安全 inotify 创建
285 preadv ssize_t preadv(int fd, const struct iovec iov, int iovcnt, off_t offset); 分散读（带偏移） 原子操作 多缓冲区随机读
286 pwritev ssize_t pwritev(int fd, const struct iovec iov, int iovcnt, off_t offset); 集中写（带偏移） 原子操作 多缓冲区随机写
287 rt_tgsigqueueinfo int rt_tgsigqueueinfo(pid_t tgid, pid_t tid, int sig, siginfo_t uinfo); 向指定线程排队发送信号 实时信号扩展 线程间传递数据
288 perf_event_open int perf_event_open(struct perf_event_attr attr, pid_t pid, int cpu, int group_fd, unsigned long flags); 配置性能监控事件 需特权（部分） 性能分析（perf）
289 recvmmsg int recvmmsg(int sockfd, struct mmsghdr msgvec, unsigned int vlen, unsigned int flags, struct timespec timeout); 批量接收消息 减少 syscall 次数 高吞吐网络接收
290 fanotify_init int fanotify_init(unsigned int flags, unsigned int event_f_flags); 初始化 fanotify 需 CONFIG_FANOTIFY 文件访问监控（杀毒软件）
291 fanotify_mark int fanotify_mark(int fanotify_fd, unsigned int flags, uint64_t mask, int dirfd, const char pathname); 添加 fanotify 监控 同上 监控文件访问
292 prlimit64 int prlimit64(pid_t pid, int resource, const struct rlimit64 new_limit, struct rlimit64 old_limit); 获取/设置资源限制（64位） 支持大文件限制 资源限制（现代）
293 name_to_handle_at int name_to_handle_at(int dirfd, const char name, struct file_handle handle, int mnt_id, int flags); 获取文件句柄 需特权 文件持久化引用
294 open_by_handle_at int open_by_handle_at(int mount_fd, struct file_handle handle, int flags); 通过句柄打开文件 同上 通过句柄访问文件
295 clock_adjtime int clock_adjtime(clockid_t clk_id, struct timex tx); 调整指定时钟 NTP 使用 精确时钟校准
296 syncfs int syncfs(int fd); 同步文件系统 比 sync 更精细 文件系统级同步
297 sendmmsg int sendmmsg(int sockfd, struct mmsghdr msgvec, unsigned int vlen, unsigned int flags); 批量发送消息 减少 syscall 次数 高吞吐网络发送
298 setns int setns(int fd, int nstype); 加入指定 namespace 容器实现关键 容器、沙箱
299 getcpu int getcpu(unsigned cpu, unsigned node, struct getcpu_cache tcache); 获取当前 CPU 和 NUMA 节点 快速查询 NUMA 感知
300 process_vm_readv ssize_t process_vm_readv(pid_t pid, const struct iovec local_iov, unsigned long liovcnt, const struct iovec remote_iov, unsigned long riovcnt, unsigned long flags); 跨进程内存读取 无需 ptrace 权限 进程内存 dump
301 process_vm_writev ssize_t process_vm_writev(pid_t pid, const struct iovec local_iov, unsigned long liovcnt, const struct iovec remote_iov, unsigned long riovcnt, unsigned long flags); 跨进程内存写入 同上 进程内存注入
302 kcmp int kcmp(pid_t pid1, pid_t pid2, int type, unsigned long idx1, unsigned long idx2); 比较内核资源 安全沙箱使用 资源相等性检查
303 finit_module int finit_module(int fd, const char uargs, int flags); 从文件描述符加载模块 需特权 安全模块加载
304 sched_setattr int sched_setattr(pid_t pid, struct sched_attr attr, unsigned int flags); 设置调度属性（扩展） 支持 DEADLINE 调度 实时调度（现代）
305 sched_getattr int sched_getattr(pid_t pid, struct sched_attr attr, unsigned int size, unsigned int flags); 获取调度属性 同上 调度属性查询
306 renameat2 int renameat2(int olddirfd, const char oldpath, int newdirfd, const char newpath, unsigned int flags); 重命名（带标志） flags: RENAME_EXCHANGE/WHITEOUT 原子交换、overlayfs
307 seccomp int seccomp(unsigned int operation, unsigned int flags, void args); 操作安全计算模式 容器沙箱核心 系统调用过滤
308 getrandom ssize_t getrandom(void buf, size_t buflen, unsigned int flags); 获取加密安全随机数 阻塞直到熵足够 密钥生成、nonce
309 memfd_create int memfd_create(const char name, unsigned int flags); 创建匿名内存文件 支持 MFD_CLOEXEC/SEAL 临时文件、沙箱
310 kexec_file_load int kexec_file_load(int kernel_fd, int initrd_fd, unsigned long cmdline_len, const char cmdline_ptr, unsigned long flags); 从文件加载 kexec 内核 需特权 安全 kexec
311 bpf long bpf(int cmd, union bpf_attr attr, unsigned int size); 操作 eBPF 程序/映射 需特权（部分） 网络、跟踪、安全
312 execveat int execveat(int dirfd, const char pathname, char const argv[], char const envp[], int flags); 在指定目录下执行程序 flags: AT_EMPTY_PATH 安全 exec
313 userfaultfd int userfaultfd(int flags); 创建用户态缺页处理 fd 需 CONFIG_USERFAULTFD 虚拟机迁移、垃圾回收
314 membarrier int membarrier(int cmd, unsigned int flags, int cpu_id); 全局内存屏障 用于用户态 RCU 并发算法优化
315 mlock2 int mlock2(const void addr, size_t len, int flags); 锁定内存（带标志） flags: MLOCK_ONFAULT 按需锁定
316 copy_file_range ssize_t copy_file_range(int fd_in, off64_t off_in, int fd_out, off64_t off_out, size_t len, unsigned int flags); 高效复制文件范围 可能零拷贝 文件复制优化
317 preadv2 ssize_t preadv2(int fd, const struct iovec iov, int iovcnt, off_t offset, int flags); 分散读（带偏移和标志） flags: RWF_HIPRI 高优先级 I/O
318 pwritev2 ssize_t pwritev2(int fd, const struct iovec iov, int iovcnt, off_t offset, int flags); 集中写（带偏移和标志） 同上 高优先级 I/O
319 pkey_mprotect int pkey_mprotect(void addr, size_t len, int prot, int pkey); 带保护键的内存保护 需硬件支持 内存隔离
320 pkey_alloc int pkey_alloc(unsigned long flags, unsigned long init_val); 分配保护键 同上 保护键管理
321 pkey_free int pkey_free(int pkey); 释放保护键 同上 保护键释放
322 statx int statx(int dirfd, const char pathname, int flags, unsigned int mask, struct statx statxbuf); 扩展文件状态查询 支持更多属性 现代文件状态
323 io_pgetevents int io_pgetevents(aio_context_t ctx_id, long min_nr, long nr, struct io_event events, struct timespec timeout, const struct __aio_sigset sig); 带信号掩码的 AIO 事件 同上 安全 AIO 等待
324 rseq int rseq(struct rseq rseq, uint32_t rseq_len, int flags, uint32_t sig); 注册重启序列 用于用户态调度 无锁并发优化
325 pidfd_send_signal int pidfd_send_signal(int pidfd, int sig, siginfo_t info, unsigned int flags); 通过 pidfd 发送信号 避免 PID 复用问题 安全进程信号
326 io_uring_setup int io_uring_setup(unsigned entries, struct io_uring_params p); 初始化 io_uring 上下文 高性能异步 I/O 数据库、存储系统
327 io_uring_enter int io_uring_enter(unsigned int fd, unsigned int to_submit, unsigned int min_complete, unsigned int flags, const sigset_t sig); 提交/完成 io_uring I/O 可 zero-syscall 极致 I/O 性能
328 io_uring_register int io_uring_register(unsigned int fd, unsigned int opcode, void arg, unsigned int nr_args); 注册缓冲区或文件 提升性能 io_uring 优化
329 open_tree int open_tree(int dfd, const char path, unsigned int flags); 打开文件系统子树 用于 mount API 现代挂载
330 move_mount int move_mount(int from_dfd, const char from_path, int to_dfd, const char to_path, unsigned int flags); 移动挂载点 新 mount API 安全挂载操作
331 fsopen int fsopen(const char fs_name, unsigned int flags); 创建文件系统上下文 新 mount API 文件系统挂载准备
332 fsconfig int fsconfig(int fs_fd, unsigned int cmd, const char key, const void value, int aux); 配置文件系统参数 新 mount API 文件系统参数设置
333 fsmount int fsmount(int fs_fd, unsigned int flags, unsigned int ms_flags); 创建挂载实例 新 mount API 文件系统挂载执行
334 fspick int fspick(int dfd, const char path, unsigned int flags); 选择现有挂载 新 mount API 挂载克隆
335 pidfd_open int pidfd_open(pid_t pid, unsigned int flags); 打开进程文件描述符 避免 PID 复用 安全进程监控
336 clone3 int clone3(struct clone_args cl_args, size_t size); 创建进程/线程（扩展版） 支持更多标志 现代进程创建
337 close_range int close_range(unsigned int first, unsigned int last, unsigned int flags); 批量关闭文件描述符 flags: CLOSE_RANGE_UNSHARE 高效 fd 清理
338 openat2 int openat2(int dirfd, const char pathname, struct open_how how, size_t size); 打开文件（扩展参数） 支持 resolve flags 安全路径解析
339 pidfd_getfd int pidfd_getfd(int pidfd, int targetfd, unsigned int flags); 从其他进程复制 fd 需 YAMA 安全模块 跨进程 fd 共享
340 faccessat2 int faccessat2(int dirfd, const char pathname, int mode, int flags); 检查访问权限（扩展） 支持 AT_RESOLVE_BENEATH 安全路径访问
341 process_mrelease int process_mrelease(int pidfd, unsigned int flags); 释放进程内存 OOM 优化 内存回收
342 futex_waitv int futex_waitv(struct futex_waitv waiters, unsigned int nr_futexes, unsigned int flags, struct timespec timeout, clockid_t clockid); 批量 futex 等待 高效多条件等待 现代同步原语
343 set_mempolicy_home_node int set_mempolicy_home_node(unsigned long start, unsigned long len, unsigned long home_node, unsigned long flags); 设置内存主页节点 NUMA 优化 内存局部性优化
344 cachestat int cachestat(unsigned int fd, struct cachestat_range range, struct cachestat_result result, unsigned int flags); 查询文件缓存统计 性能分析 缓存命中率分析
345 fchmodat2 int fchmodat2(int dirfd, const char pathname, mode_t mode, int flags); 改变权限（扩展） 支持 AT_SYMLINK_FOLLOW 安全权限修改
346 map_shadow_stack int map_shadow_stack(unsigned long addr, unsigned long size, unsigned long flags); 映射影子栈 Intel CET 支持 控制流完整性
347 futex_wake int futex_wake(uint32_t uaddr, unsigned int nr_wake, unsigned int flags); 唤醒 futex 等待者 futex2 API 现代 futex
348 futex_wait int futex_wait(uint32_t uaddr, uint32_t val, const struct timespec timeout, unsigned int flags); 等待 futex futex2 API 现代 futex
349 futex_requeue int futex_requeue(uint32_t uaddr, uint32_t val, uint32_t uaddr2, unsigned int nr_wake, unsigned int nr_requeue, unsigned int flags); 重排队 futex futex2 API 现代 futex
350 landlock_create_ruleset int landlock_create_ruleset(const struct landlock_ruleset_attr attr, size_t size, __u32 flags); 创建 Landlock 规则集 沙箱安全 应用沙箱
351 landlock_add_rule int landlock_add_rule(int ruleset_fd, enum landlock_rule_type rule_type, const void rule_attr, __u32 flags); 添加 Landlock 规则 同上 沙箱规则
352 landlock_restrict_self int landlock_restrict_self(int ruleset_fd, __u32 flags); 应用 Landlock 规则 同上 沙箱启用
353 memfd_secret int memfd_secret(unsigned int flags); 创建秘密内存区域 隔离敏感数据 安全内存
354 process_madvise int process_madvise(int pidfd, const struct iovec iovec, size_t vlen, int behavior, unsigned int flags); 对其他进程内存提建议 性能优化 跨进程内存提示
355 epoll_pwait2 int epoll_pwait2(int epfd, struct epoll_event events, int maxevents, const struct timespec timeout, const sigset_t sigmask, size_t sigsetsize); epoll 等待（带 timespec） 高精度超时 现代 epoll
356 dup4 int dup4(int oldfd, int newfd, int flags, int reserved); 复制 fd（预留扩展） 未来扩展 —
357 fscrypt_lock int fscrypt_lock(int fd, unsigned int cmd, void arg); 控制文件系统加密 fscrypt 管理 加密文件系统
358 fscrypt_unlock int fscrypt_unlock(int fd, unsigned int cmd, void arg); 解锁文件系统加密 同上 加密文件系统
359 fscrypt_get_policy_ex int fscrypt_get_policy_ex(int fd, void policy, size_t policy_size); 获取加密策略（扩展） 同上 加密策略查询
360 fscrypt_add_key int fscrypt_add_key(int fd, const void key_spec, size_t key_spec_size, const void raw_key, size_t raw_key_size); 添加加密密钥 同上 加密密钥管理
361 fscrypt_remove_key int fscrypt_remove_key(int fd, const void key_spec, size_t key_spec_size); 移除加密密钥 同上 加密密钥移除
362 fscrypt_get_key_status int fscrypt_get_key_status(int fd, const void key_spec, size_t key_spec_size, void status, size_t status_size); 获取密钥状态 同上 加密状态查询
363 pidfd_spawn int pidfd_spawn(int pidfd, const char path, const posix_spawn_file_actions_t file_actions, const posix_spawnattr_t attrp, char const argv[], char const envp[]); 创建进程并返回 pidfd 安全进程创建 现代进程启动
364 cgroup_get_kernfs_fd int cgroup_get_kernfs_fd(int cgroup_fd); 获取 cgroup kernfs fd cgroup 管理 容器资源控制
365 cgroup_kill int cgroup_kill(int cgroup_fd, int sig); 向 cgroup 内所有进程发信号 容器管理 批量信号发送
366 mount_setattr int mount_setattr(int dfd, const char path, unsigned int flags, struct mount_attr attr, size_t size); 设置挂载属性 新 mount API 挂载属性修改
367 quotactl_fd int quotactl_fd(int fd, unsigned int cmd, int id, void addr); 磁盘配额控制（fd 版） 安全配额管理 配额操作
368 landlock_create_ruleset_compat int landlock_create_ruleset_compat(const struct landlock_ruleset_attr attr, size_t size, __u32 flags); Landlock 兼容创建 沙箱 兼容模式
369 memfd_create_volatile int memfd_create_volatile(const char name, unsigned int flags); 创建易失内存文件 临时数据 高性能临时存储
370 futex_waitv2 int futex_waitv2(struct futex_waitv waiters, unsigned int nr_futexes, unsigned int flags, struct timespec timeout, clockid_t clockid); 批量 futex 等待（v2） futex2 扩展 现代同步
371 futex_wake2 int futex_wake2(uint32_t uaddr, unsigned int nr_wake, unsigned int flags); 唤醒 futex（v2） futex2 扩展 现代同步
372 futex_requeue2 int futex_requeue2(uint32_t uaddr, uint32_t val, uint32_t uaddr2, unsigned int nr_wake, unsigned int nr_requeue, unsigned int flags); 重排队 futex（v2） futex2 扩展 现代同步
373 futex_cmp_requeue int futex_cmp_requeue(uint32_t uaddr, uint32_t val, uint32_t uaddr2, unsigned int nr_wake, unsigned int nr_requeue, unsigned int flags); 条件重排队 futex futex2 扩展 现代同步
374 futex_cmp_requeue2 int futex_cmp_requeue2(uint32_t uaddr, uint32_t val, uint32_t uaddr2, unsigned int nr_wake, unsigned int nr_requeue, unsigned int flags); 条件重排队 futex（v2） futex2 扩展 现代同步
375 futex_wait_bitset int futex_wait_bitset(uint32_t uaddr, uint32_t val, const struct timespec timeout, uint32_t bitset, unsigned int flags); 位图等待 futex futex2 扩展 现代同步
376 futex_wait_bitset2 int futex_wait_bitset2(uint32_t uaddr, uint32_t val, const struct timespec timeout, uint32_t bitset, unsigned int flags); 位图等待 futex（v2） futex2 扩展 现代同步
377 futex_wake_bitset int futex_wake_bitset(uint32_t uaddr, unsigned int nr_wake, uint32_t bitset, unsigned int flags); 位图唤醒 futex futex2 扩展 现代同步
378 futex_wake_bitset2 int futex_wake_bitset2(uint32_t uaddr, unsigned int nr_wake, uint32_t bitset, unsigned int flags); 位图唤醒 futex（v2） futex2 扩展 现代同步
379 futex_lock_pi int futex_lock_pi(uint32_t uaddr, const struct timespec timeout, unsigned int flags); PI 互斥锁获取 futex2 扩展 实时互斥锁
380 futex_lock_pi2 int futex_lock_pi2(uint32_t uaddr, const struct timespec timeout, unsigned int flags); PI 互斥锁获取（v2） futex2 扩展 实时互斥锁
381 futex_unlock_pi int futex_unlock_pi(uint32_t uaddr, unsigned int flags); PI 互斥锁释放 futex2 扩展 实时互斥锁
382 futex_unlock_pi2 int futex_unlock_pi2(uint32_t uaddr, unsigned int flags); PI 互斥锁释放（v2） futex2 扩展 实时互斥锁
383 futex_trylock_pi int futex_trylock_pi(uint32_t uaddr, unsigned int flags); PI 互斥锁尝试获取 futex2 扩展 实时互斥锁
384 futex_trylock_pi2 int futex_trylock_pi2(uint32_t uaddr, unsigned int flags); PI 互斥锁尝试获取（v2） futex2 扩展 实时互斥锁
385 futex_wait_requeue_pi int futex_wait_requeue_pi(uint32_t uaddr, uint32_t val, uint32_t uaddr2, const struct timespec timeout, unsigned int flags); 等待并重排队到 PI 锁 futex2 扩展 实时同步
386 futex_wait_requeue_pi2 int futex_wait_requeue_pi2(uint32_t uaddr, uint32_t val, uint32_t uaddr2, const struct timespec timeout, unsigned int flags); 等待并重排队到 PI 锁（v2） futex2 扩展 实时同步
387 futex_cmp_requeue_pi int futex_cmp_requeue_pi(uint32_t uaddr, uint32_t val, uint32_t uaddr2, unsigned int nr_wake, unsigned int flags); 条件重排队到 PI 锁 futex2 扩展 实时同步
388 futex_cmp_requeue_pi2 `int futex_cmp_requeue_pi2(uint32_t uaddr, uint32_t val, uint32_t uaddr2, unsigned int nr_w
```
