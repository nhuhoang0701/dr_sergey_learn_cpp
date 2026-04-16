# Build HTTP Servers and Clients with Boost.Beast and cpp-httplib

**Category:** Networking & I/O  
**Standard:** C++17/20  
**Reference:** [Boost.Beast Docs](https://www.boost.org/doc/libs/release/libs/beast/doc/html/index.html), [cpp-httplib](https://github.com/yhirose/cpp-httplib)  

---

## Topic Overview

C++ has no standard HTTP library, so developers choose between full-featured async frameworks and lightweight sync libraries. **Boost.Beast** provides HTTP and WebSocket built on Boost.Asio, offering complete control over async I/O at the cost of complexity. **cpp-httplib** is a header-only, single-file library for simple synchronous HTTP — ideal for internal tools, REST APIs with modest concurrency, and rapid prototyping.

The design philosophies differ fundamentally. Beast exposes HTTP as message objects (`http::request`, `http::response`) composed with a body type (string, file, empty) and fields — it does not provide routing, middleware, or JSON handling. You build those on top. cpp-httplib provides a complete server with routing, multipart form parsing, and built-in compression — batteries included but single-threaded per connection.

| Feature                | Boost.Beast                    | cpp-httplib                  |
| --- | --- | --- |
| Async support          | Full (Asio coroutines, callbacks) | None (thread-per-connection) |
| Header-only            | Yes (with Boost)              | Yes (single header)          |
| TLS support            | Via Asio SSL                  | Via OpenSSL (optional)       |
| HTTP/2                 | No (HTTP/1.1 only)            | No                           |
| Routing                | Manual                        | Built-in regex routes        |
| WebSocket              | Yes                           | No                           |
| Dependencies           | Boost                         | None (or OpenSSL for HTTPS)  |
| Ideal for              | High-perf async servers       | Internal tools, prototypes   |

```cpp

Beast Architecture:                cpp-httplib Architecture:
┌──────────────┐                   ┌──────────────────┐
│  io_context  │                   │  httplib::Server  │
│  ┌────────┐  │                   │  ┌────────────┐  │
│  │ strand  │  │                   │  │  Router    │  │
│  │┌──────┐│  │                   │  │  Handler[] │  │
│  ││stream││  │                   │  └────────────┘  │
│  │└──────┘│  │                   │  Thread Pool     │
│  └────────┘  │                   └──────────────────┘
└──────────────┘

```

For production HTTP servers in C++, Beast is the right choice when you need async I/O, WebSocket upgrade, or fine-grained control. cpp-httplib is the right choice when you need an HTTP endpoint in 50 lines with no build system changes.

---

## Self-Assessment

### Q1: Build an async HTTP server with Boost.Beast and C++20 coroutines that handles GET and POST with JSON

```cpp

#include <boost/beast/core.hpp>
#include <boost/beast/http.hpp>
#include <boost/asio/co_spawn.hpp>
#include <boost/asio/detached.hpp>
#include <boost/asio/use_awaitable.hpp>
#include <boost/asio/ip/tcp.hpp>
#include <iostream>
#include <string>

namespace beast = boost::beast;
namespace http  = beast::http;
namespace asio  = boost::asio;
using tcp = asio::ip::tcp;

// Simple router: match method + target
http::response<http::string_body>
handle_request(http::request<http::string_body>&& req) {
    auto const make_response = [&](http::status status,
                                    std::string body,
                                    std::string content_type = "application/json") {
        http::response<http::string_body> res{status, req.version()};
        res.set(http::field::server, "Beast-Server/1.0");
        res.set(http::field::content_type, content_type);
        res.keep_alive(req.keep_alive());
        res.body() = std::move(body);
        res.prepare_payload();
        return res;
    };

    // Route: GET /health
    if (req.method() == http::verb::get && req.target() == "/health") {
        return make_response(http::status::ok, R"({"status":"ok"})");
    }

    // Route: POST /echo
    if (req.method() == http::verb::post && req.target() == "/echo") {
        return make_response(http::status::ok, std::string(req.body()));
    }

    // Route: GET /
    if (req.method() == http::verb::get && req.target() == "/") {
        return make_response(http::status::ok,
            R"({"message":"Hello from Beast"})");
    }

    return make_response(http::status::not_found,
        R"({"error":"Not found"})");
}

asio::awaitable<void> handle_session(tcp::socket socket) {
    beast::flat_buffer buffer;
    try {
        while (true) {
            // Read request
            http::request<http::string_body> req;
            co_await http::async_read(socket, buffer, req,
                                       asio::use_awaitable);

            // Handle and send response
            auto resp = handle_request(std::move(req));
            bool keep_alive = resp.keep_alive();

            co_await http::async_write(socket, resp,
                                        asio::use_awaitable);

            if (!keep_alive) break;
        }
    } catch (const std::exception& e) {
        // Connection closed or error — normal for HTTP
    }
    beast::error_code ec;
    socket.shutdown(tcp::socket::shutdown_send, ec);
}

asio::awaitable<void> listener(tcp::endpoint endpoint) {
    auto executor = co_await asio::this_coro::executor;
    tcp::acceptor acceptor(executor, endpoint);
    acceptor.set_option(asio::socket_base::reuse_address(true));

    while (true) {
        auto socket = co_await acceptor.async_accept(asio::use_awaitable);
        asio::co_spawn(executor, handle_session(std::move(socket)),
                       asio::detached);
    }
}

int main() {
    asio::io_context ioc{1};
    asio::co_spawn(ioc,
        listener(tcp::endpoint(tcp::v4(), 8080)),
        asio::detached);
    std::cout << "Listening on :8080\n";
    ioc.run();
}

```

### Q2: Build the equivalent simple HTTP server and client with cpp-httplib in minimal code

```cpp

// server.cpp — cpp-httplib with routing, logging middleware, and JSON
#include "httplib.h"  // single header
#include <iostream>
#include <string>
#include <mutex>
#include <unordered_map>

int main() {
    httplib::Server svr;

    // Thread-safe in-memory store
    std::unordered_map<std::string, std::string> store;
    std::mutex mu;

    // Logging middleware — runs before every handler
    svr.set_pre_routing_handler([](const httplib::Request& req,
                                    httplib::Response& /*res*/) {
        std::cout << req.method << " " << req.path << "\n";
        return httplib::Server::HandlerResponse::Unhandled; // continue
    });

    // GET /health
    svr.Get("/health", [](const httplib::Request&, httplib::Response& res) {
        res.set_content(R"({"status":"ok"})", "application/json");
    });

    // GET /items/:id — path parameter
    svr.Get(R"(/items/(\w+))", [&](const httplib::Request& req,
                                     httplib::Response& res) {
        auto id = req.matches[1].str();
        std::lock_guard lock(mu);
        if (auto it = store.find(id); it != store.end()) {
            res.set_content(
                R"({"id":")" + id + R"(","value":")" + it->second + R"("})",
                "application/json");
        } else {
            res.status = 404;
            res.set_content(R"({"error":"not found"})", "application/json");
        }
    });

    // POST /items/:id
    svr.Post(R"(/items/(\w+))", [&](const httplib::Request& req,
                                      httplib::Response& res) {
        auto id = req.matches[1].str();
        {
            std::lock_guard lock(mu);
            store[id] = req.body;
        }
        res.status = 201;
        res.set_content(R"({"created":")" + id + R"("})",
                        "application/json");
    });

    // Error handler
    svr.set_error_handler([](const httplib::Request&, httplib::Response& res) {
        res.set_content(
            R"({"error":")" + std::to_string(res.status) + R"("})",
            "application/json");
    });

    std::cout << "Serving on http://localhost:8080\n";
    svr.listen("0.0.0.0", 8080);
}

```

```cpp

// client.cpp — cpp-httplib client
#include "httplib.h"
#include <iostream>

int main() {
    httplib::Client cli("http://localhost:8080");
    cli.set_connection_timeout(5);  // seconds
    cli.set_read_timeout(5);

    // POST
    auto post_res = cli.Post("/items/foo", "bar_value", "text/plain");
    if (post_res && post_res->status == 201) {
        std::cout << "Created: " << post_res->body << "\n";
    }

    // GET
    auto get_res = cli.Get("/items/foo");
    if (get_res && get_res->status == 200) {
        std::cout << "Got: " << get_res->body << "\n";
    }

    // GET with headers
    httplib::Headers headers = {{"Authorization", "Bearer token123"}};
    auto auth_res = cli.Get("/health", headers);
    if (auth_res) {
        std::cout << "Health: " << auth_res->body << "\n";
    }
}

```

### Q3: How do you serve static files and handle multipart file uploads with Beast

```cpp

#include <boost/beast/core.hpp>
#include <boost/beast/http.hpp>
#include <boost/asio.hpp>
#include <filesystem>
#include <fstream>
#include <iostream>

namespace beast = boost::beast;
namespace http  = beast::http;
namespace fs    = std::filesystem;

// MIME type lookup
std::string_view mime_type(std::string_view path) {
    if (path.ends_with(".html")) return "text/html";
    if (path.ends_with(".css"))  return "text/css";
    if (path.ends_with(".js"))   return "application/javascript";
    if (path.ends_with(".json")) return "application/json";
    if (path.ends_with(".png"))  return "image/png";
    if (path.ends_with(".jpg"))  return "image/jpeg";
    return "application/octet-stream";
}

// Serve a static file using Beast's file_body for zero-copy sendfile
http::response<http::file_body>
serve_file(const http::request<http::string_body>& req,
           const fs::path& doc_root) {
    // Sanitize path — prevent directory traversal
    std::string target(req.target());
    if (target.empty() || target[0] != '/' ||
        target.find("..") != std::string::npos) {
        throw std::runtime_error("Bad target");
    }

    fs::path file_path = doc_root / target.substr(1);
    if (fs::is_directory(file_path))
        file_path /= "index.html";

    // Canonicalize and verify it's under doc_root
    auto canonical = fs::weakly_canonical(file_path);
    auto root_canonical = fs::weakly_canonical(doc_root);
    if (canonical.string().find(root_canonical.string()) != 0)
        throw std::runtime_error("Path traversal attempt");

    http::response<http::file_body> res{http::status::ok, req.version()};
    res.set(http::field::content_type, mime_type(file_path.string()));

    beast::error_code ec;
    res.body().open(file_path.string().c_str(),
                    beast::file_mode::scan, ec);
    if (ec) throw std::runtime_error("File not found");

    res.prepare_payload();
    res.keep_alive(req.keep_alive());
    return res;
}

// Parse multipart boundary from Content-Type
std::string extract_boundary(std::string_view content_type) {
    auto pos = content_type.find("boundary=");
    if (pos == std::string_view::npos) return {};
    auto boundary = content_type.substr(pos + 9);
    // Remove surrounding quotes if present
    if (boundary.starts_with("\"") && boundary.ends_with("\""))
        boundary = boundary.substr(1, boundary.size() - 2);
    return std::string(boundary);
}

// Minimal multipart body extraction (production: use a proper parser)
void handle_upload(const http::request<http::string_body>& req,
                   const fs::path& upload_dir) {
    auto ct = std::string(req[http::field::content_type]);
    auto boundary = extract_boundary(ct);
    if (boundary.empty()) return;

    // Find file data between boundaries (simplified)
    const auto& body = req.body();
    std::string delim = "--" + boundary;
    auto start = body.find("\r\n\r\n");
    auto end   = body.rfind(delim);
    if (start != std::string::npos && end != std::string::npos) {
        start += 4; // skip \r\n\r\n
        auto data = body.substr(start, end - start - 2);
        std::ofstream out(upload_dir / "uploaded_file",
                          std::ios::binary);
        out.write(data.data(), static_cast<std::streamsize>(data.size()));
    }
}

```

---

## Notes

- Beast does **not** provide routing — build a `std::unordered_map<string, Handler>` or use a trie for path matching.
- Beast's `file_body` uses OS-native `sendfile()` on Linux for zero-copy file serving — much faster than reading into a string.
- cpp-httplib's regex routing uses `std::regex` — avoid complex patterns in hot paths due to `std::regex` compilation overhead.
- For JSON, pair Beast with `nlohmann/json` or `simdjson`; cpp-httplib has no built-in JSON but integrates trivially.
- Always sanitize file paths to prevent directory traversal (`..`); canonicalize and verify the resolved path is within the document root.
- cpp-httplib supports gzip compression with `svr.set_compress(true)` when compiled with zlib.
- For HTTP/2, neither Beast nor cpp-httplib supports it — consider `nghttp2` or use a reverse proxy (nginx) in front.
