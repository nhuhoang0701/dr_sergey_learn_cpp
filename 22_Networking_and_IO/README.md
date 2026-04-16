# Networking & I/O

I/O streams, async I/O, sockets, file handling, and serialization.

**Topics:** 29

## Contents

- [Handle DNS resolution asynchronously without blocking the event loop](Handle_DNS_resolution_asynchronously_without_blocking_the_event_loop.md)
- [Implement a binary protocol framer with length-prefixed messages](Implement_a_binary_protocol_framer_with_length-prefixed_messages.md)
- [Implement a high-performance ring buffer for producer-consumer IO](Implement_a_high-performance_ring_buffer_for_producer-consumer_IO.md)
- [Implement non-blocking IO multiplexing with epoll on Linux](Implement_non-blocking_IO_multiplexing_with_epoll_on_Linux.md)
- [Implement zero-copy IO patterns with scatter-gather iovec readvwritev](Implement_zero-copy_IO_patterns_with_scatter-gather_iovec_readvwritev.md)
- [Implement zero-copy IO with scatter-gather using iovec and readvwritev](Implement_zero-copy_IO_with_scatter-gather_using_iovec_and_readvwritev.md)
- [Understand TCP Nagle algorithm delayed ACK and when to disable them](Understand_TCP_Nagle_algorithm_delayed_ACK_and_when_to_disable_them.md)
- [Understand TCP flow control Nagles algorithm and socket tuning](Understand_TCP_flow_control_Nagles_algorithm_and_socket_tuning.md)
- [Understand zero-copy techniques with scatter-gather IO iovec WSABuf](Understand_zero-copy_techniques_with_scatter-gather_IO_iovec_WSABuf.md)
- [Use Asio coroutines for async TCP clientserver](Use_Asio_coroutines_for_async_TCP_clientserver.md)
- [Use Asio for async TCP networking with coroutines](Use_Asio_for_async_TCP_networking_with_coroutines.md)
- [Use Asio for async networking with coroutines](Use_Asio_for_async_networking_with_coroutines.md)
- [Use io uring for async IO with minimal syscall overhead](Use_io_uring_for_async_IO_with_minimal_syscall_overhead.md)
- [Use io uring for asynchronous IO without syscall overhead](Use_io_uring_for_asynchronous_IO_without_syscall_overhead.md)
- [Use io uring from C++ for zero-syscall async IO on Linux](Use_io_uring_from_C++_for_zero-syscall_async_IO_on_Linux.md)
- [Use memory-mapped files mmap for fast file IO](Use_memory-mapped_files_mmap_for_fast_file_IO.md)
- [Use memory-mapped files with mmap MapViewOfFile for zero-copy IO](Use_memory-mapped_files_with_mmap_MapViewOfFile_for_zero-copy_IO.md)
- [Use memory-mapped files with mmap for high-throughput file access](Use_memory-mapped_files_with_mmap_for_high-throughput_file_access.md)
- [Use non-blocking IO with epoll for scalable event-driven servers](Use_non-blocking_IO_with_epoll_for_scalable_event-driven_servers.md)
- [Use sendfile and splice for kernel-space zero-copy data transfer](Use_sendfile_and_splice_for_kernel-space_zero-copy_data_transfer.md)
- [Use stdnet Networking TS when available for portable async networking](Use_stdnet_Networking_TS_when_available_for_portable_async_networking.md)
- [Wrap POSIX sockets in RAII and use them safely from C++](Wrap_POSIX_sockets_in_RAII_and_use_them_safely_from_C++.md)
- [Wrap POSIX sockets in RAII for safe resource management](Wrap_POSIX_sockets_in_RAII_for_safe_resource_management.md)
- [Write a POSIX TCP server with RAII socket wrappers](Write_a_POSIX_TCP_server_with_RAII_socket_wrappers.md)
- [Build HTTP servers and clients with Boost Beast and cpp-httplib](Build_HTTP_servers_and_clients_with_Boost_Beast_and_cpp-httplib.md)
- [Implement WebSocket communication in C++ with Boost Beast](Implement_WebSocket_communication_in_C++_with_Boost_Beast.md)
- [Integrate TLS into C++ networking with OpenSSL and Botan](Integrate_TLS_into_C++_networking_with_OpenSSL_and_Botan.md)
- [Use gRPC for high performance RPC in C++](Use_gRPC_for_high_performance_RPC_in_C++.md)
- [Use Protocol Buffers and FlatBuffers for efficient network serialization](Use_Protocol_Buffers_and_FlatBuffers_for_efficient_network_serialization.md)

## Notes

- Asio (Boost.Asio) is the de facto standard for async C++ networking — it influenced std::execution
- std::iostream is slow for high-throughput I/O — prefer memory-mapped files or raw ead/write
- Non-blocking I/O with epoll/io_uring (Linux) or IOCP (Windows) scales to thousands of connections
- io_uring is the modern Linux async I/O interface — supports files, sockets, and timers uniformly
- Use std::format (C++20) for string formatting instead of std::stringstream
- Memory-mapped files (mmap/MapViewOfFile) provide zero-copy access to file contents
- Scatter/gather I/O (eadv/writev) reduces system calls for multi-buffer operations
- Serialize with protocol buffers, FlatBuffers, or Cap'n Proto for cross-language binary protocols
- Always validate network input — buffer overflows and format string attacks remain prevalent
- Connection pooling and keep-alive reduce the overhead of establishing new connections
