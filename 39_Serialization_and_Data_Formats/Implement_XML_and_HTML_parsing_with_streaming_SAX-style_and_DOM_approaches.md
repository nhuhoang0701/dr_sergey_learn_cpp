# Implement XML and HTML parsing with streaming SAX-style and DOM approaches

**Category:** Serialization & Data Formats  
**Standard:** C++17  
**Reference:** <https://pugixml.org/> · <https://rapidxml.sourceforge.net/>  

---

## Topic Overview

XML remains common in enterprise, configuration, and document formats. Two parsing approaches exist: **DOM** (load entire tree into memory) and **SAX/streaming** (event-driven, low memory).

### DOM Parsing with pugixml

```cpp

#include <pugixml.hpp>
#include <iostream>
#include <string>

void dom_example() {
    pugi::xml_document doc;
    auto result = doc.load_string(R"(
        <library>
            <book id="1">
                <title>Effective Modern C++</title>
                <author>Scott Meyers</author>
                <year>2014</year>
            </book>
            <book id="2">
                <title>C++ Concurrency in Action</title>
                <author>Anthony Williams</author>
                <year>2019</year>
            </book>
        </library>
    )");

    if (!result) {
        std::cerr << "Parse error: " << result.description() << "\n";
        return;
    }

    // Navigate the DOM tree
    for (auto book : doc.child("library").children("book")) {
        std::cout << "Book #" << book.attribute("id").as_int() << ": "
                  << book.child("title").text().get() << " by "
                  << book.child("author").text().get() << " ("
                  << book.child("year").text().as_int() << ")\n";
    }

    // XPath queries
    auto nodes = doc.select_nodes("//book[year>2015]");
    std::cout << "\nBooks after 2015:\n";
    for (auto& node : nodes) {
        std::cout << "  " << node.node().child("title").text().get() << "\n";
    }
}

```

### SAX-Style Streaming with Expat Wrapper

```cpp

#include <iostream>
#include <string>
#include <functional>
#include <stack>

// Conceptual SAX callback interface
struct SAXHandler {
    virtual void on_start_element(const std::string& name,
                                   const std::vector<std::pair<std::string, std::string>>& attrs) = 0;
    virtual void on_end_element(const std::string& name) = 0;
    virtual void on_text(const std::string& text) = 0;
    virtual ~SAXHandler() = default;
};

class BookCounter : public SAXHandler {
    int count_ = 0;
    bool in_title_ = false;
public:
    void on_start_element(const std::string& name,
                           const std::vector<std::pair<std::string, std::string>>&) override {
        if (name == "book") ++count_;
        in_title_ = (name == "title");
    }

    void on_end_element(const std::string&) override {
        in_title_ = false;
    }

    void on_text(const std::string& text) override {
        if (in_title_) {
            std::cout << "  Title: " << text << "\n";
        }
    }

    int count() const { return count_; }
};

// Usage: feed XML to handler chunk by chunk
// Memory usage: O(depth) vs DOM's O(document_size)

```

### DOM vs SAX Decision Matrix

| Factor | DOM (pugixml) | SAX/Streaming |
| --- | --- | --- |
| Memory | O(document size) | O(tree depth) |
| Random access | Yes | No (forward only) |
| Ease of use | Very easy | More complex |
| Speed | Fast for small docs | Better for huge docs |
| Use case | Config files, small data | Log processing, >100MB XML |

---

## Self-Assessment

### Q1: Write XML from a C++ struct using pugixml

```cpp

#include <pugixml.hpp>
#include <sstream>
#include <string>

struct Config {
    std::string host;
    int port;
    bool debug;
};

std::string to_xml(const Config& cfg) {
    pugi::xml_document doc;
    auto root = doc.append_child("config");
    root.append_child("host").text().set(cfg.host.c_str());
    root.append_child("port").text().set(cfg.port);
    root.append_child("debug").text().set(cfg.debug);

    std::ostringstream oss;
    doc.save(oss, "  ");
    return oss.str();
}
// Produces:
// <?xml version="1.0"?>
// <config>
//   <host>localhost</host>
//   <port>8080</port>
//   <debug>true</debug>
// </config>

```

### Q2: Why is SAX parsing preferred for very large XML files

DOM parsing loads the entire XML tree into memory. A 1GB XML file requires 2-5GB of RAM for the DOM tree. SAX/streaming parsers process one element at a time, using O(depth) memory regardless of file size. For log files, data feeds, and scientific data in XML, streaming is the only practical approach.

### Q3: Handle XML namespaces correctly

```cpp

// XML with namespaces:
// <root xmlns:app="http://example.com/app">
//   <app:config>
//     <app:port>8080</app:port>
//   </app:config>
// </root>

// pugixml does NOT process namespaces automatically.
// Element names include the prefix: "app:config"
// Use the full prefixed name in queries:
auto node = doc.child("root").child("app:config").child("app:port");

```

---

## Notes

- **pugixml** is the recommended C++ XML library: fast, header-only-optional, XPath support.
- **RapidXML** is faster for parsing but lacks XPath and has no output support.
- For HTML parsing (which is malformed XML), use **Gumbo** or **lexbor**.
- Avoid `tinyxml2` for new projects — pugixml is faster and has a better API.
- XML is verbose; prefer JSON or binary formats for new systems unless XML is mandated.
