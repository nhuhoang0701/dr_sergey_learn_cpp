# Use sendfile and splice for kernel-space zero-copy data transfer

**Category:** Networking & I/O  
**Item:** #646  
**Reference:** <https://man7.org/linux/man-pages/man2/sendfile.2.html>  

---

## Topic Overview

sendfile() and splice() transfer data between file descriptors entirely in kernel space, avoiding the copy into user-space buffers.

```cpp

Traditional file-to-socket:
  Disk -> [kernel buf] -> [user buf] -> [kernel buf] -> NIC  (2 copies + 2 syscalls)
           read()           write()

sendfile:
  Disk -> [kernel buf] -> NIC  (0-1 copies, 1 syscall)
           sendfile()

splice:
  fd_in -> [pipe buffer] -> fd_out  (0 copies, data stays in kernel)
           splice()          splice()

```

| Syscall | Source | Destination | Copies | Limitations |
| --- | --- | --- | --- | --- |
| read() + write() | any fd | any fd | 2 | General purpose |
| sendfile() | file (must support mmap) | socket | 0-1 | File → socket only (Linux 2.6+: file → any) |
| splice() | pipe or fd | pipe or fd | 0 | One end must be a pipe |
| tee() | pipe | pipe | 0 | Duplicates pipe data |
| copy_file_range() | file | file | 0 | File → file only (4.5+) |

---

## Self-Assessment

### Q1: sendfile() — file to socket

```cpp

// Linux only
#include <iostream>
#include <cstring>
#include <sys/sendfile.h>
#include <sys/socket.h>
#include <sys/stat.h>
#include <netinet/in.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    // Create a test file
    const char* filepath = "/tmp/sendfile_test.txt";
    {
        int fd = open(filepath, O_CREAT | O_WRONLY | O_TRUNC, 0644);
        const char* content = "Hello from sendfile! Zero-copy transfer.\n";
        write(fd, content, strlen(content));
        close(fd);
    }

    // Create a TCP server socket
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    sockaddr_in addr{AF_INET, htons(9090), {INADDR_ANY}, {}};
    bind(server_fd, (sockaddr*)&addr, sizeof(addr));
    listen(server_fd, 1);

    std::cout << "Listening on :9090 (connect with: nc localhost 9090)\n";
    std::cout << "Sending file via sendfile() (zero-copy)...\n";

    // In production: fork or use epoll; here we accept one client
    int client_fd = accept(server_fd, nullptr, nullptr);

    // Open the file
    int file_fd = open(filepath, O_RDONLY);
    struct stat st;
    fstat(file_fd, &st);

    // sendfile: transfer file -> socket entirely in kernel space
    off_t offset = 0;
    ssize_t sent = sendfile(client_fd, file_fd, &offset, st.st_size);
    //                      ^dest       ^src    ^offset  ^count

    std::cout << "sendfile() sent " << sent << " bytes (zero user-space copies)\n";
    std::cout << "Traditional would need: read() + write() = 2 copies\n";

    close(file_fd);
    close(client_fd);
    close(server_fd);
    unlink(filepath);
}

```

### Q2: splice() — pipe-based zero-copy

```cpp

// Linux only
#define _GNU_SOURCE
#include <iostream>
#include <cstring>
#include <fcntl.h>
#include <unistd.h>
#include <sys/stat.h>

int main() {
    // splice moves data between fds via kernel pipe buffer
    // One end MUST be a pipe

    // Create source file
    const char* src_path = "/tmp/splice_src.txt";
    const char* dst_path = "/tmp/splice_dst.txt";
    {
        int fd = open(src_path, O_CREAT | O_WRONLY | O_TRUNC, 0644);
        const char* data = "Splice zero-copy demo data!\n";
        write(fd, data, strlen(data));
        close(fd);
    }

    int src_fd = open(src_path, O_RDONLY);
    int dst_fd = open(dst_path, O_CREAT | O_WRONLY | O_TRUNC, 0644);

    // Create a pipe as the intermediate buffer
    int pipe_fds[2];
    pipe(pipe_fds);  // pipe_fds[0]=read end, pipe_fds[1]=write end

    struct stat st;
    fstat(src_fd, &st);
    size_t remaining = st.st_size;

    // Step 1: splice from file -> pipe (kernel buffer)
    ssize_t n1 = splice(src_fd, nullptr, pipe_fds[1], nullptr,
                        remaining, SPLICE_F_MOVE);
    std::cout << "splice(file -> pipe): " << n1 << " bytes\n";

    // Step 2: splice from pipe -> file (kernel buffer -> dest)
    ssize_t n2 = splice(pipe_fds[0], nullptr, dst_fd, nullptr,
                        n1, SPLICE_F_MOVE);
    std::cout << "splice(pipe -> file): " << n2 << " bytes\n";

    // Verify
    close(src_fd);
    close(dst_fd);
    close(pipe_fds[0]);
    close(pipe_fds[1]);

    // Read back destination
    dst_fd = open(dst_path, O_RDONLY);
    char buf[256] = {};
    read(dst_fd, buf, 255);
    std::cout << "Copied: " << buf;
    close(dst_fd);

    // Data path: file -> kernel pipe buf -> file
    // ZERO copies to user space!
    std::cout << "\nData never entered user space.\n";
    std::cout << "Traditional: read(src) -> user buf -> write(dst) = 2 copies\n";
    std::cout << "splice:      src -> pipe buf -> dst = 0 copies\n";

    unlink(src_path);
    unlink(dst_path);
}

```

### Q3: sendfile vs mmap+write — when each is faster

| Scenario | sendfile | mmap + write | Winner |
| --- | --- | --- | --- |
| Static file serving (HTTP) | 1 syscall, 0-1 copies | 2 syscalls (mmap + write) | sendfile |
| Large file (>1GB) | Works incrementally via offset | Must handle partial maps | sendfile |
| Need to modify/filter data | Cannot modify in flight | Can transform in user space | mmap+write |
| File-to-file copy | Linux 2.6.33+ sendfile works | Must mmap+write | sendfile |
| Repeated access to same file | No caching benefit | Page cache keeps pages hot | Even |
| Socket with TLS | Cannot use (TLS needs user-space) | Can encrypt then write | mmap+write |
| FreeBSD/macOS | sendfile() available | mmap+write works | Both |

**When sendfile is faster:**

- Simple file → socket transfer (static HTTP serving).
- Streaming large files (offset parameter advances automatically).
- When you don't need to touch the data.

**When mmap+write is better:**

- Need to modify data before sending (compression, encryption, filtering).
- Random access patterns (sendfile is sequential by offset).
- Non-Linux portability (mmap is more portable than sendfile behavior).

**When neither — use splice instead:**

- Proxying: socket → socket (via pipe). nginx uses splice for proxying.
- Any fd → fd where you don't need to touch the data.

---

## Notes

- sendfile on Linux 2.6.33+ supports any out_fd (not just sockets).
- splice requires SPLICE_F_MOVE flag for page-move optimization (hint, not guaranteed).
- nginx uses sendfile for static files and splice for proxying.
- TCP_CORK + sendfile: cork header writes, then sendfile, then uncork = one TCP segment for header+data.
- Windows equivalent: `TransmitFile()` (similar to sendfile).
