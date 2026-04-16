# Implement compile-time ORM and query generation via reflection

**Category:** Reflection (C++26)  
**Standard:** C++26  
**Reference:** <https://wg21.link/P2996>  

---

## Topic Overview

### The Vision

With C++26 reflection, you can map C++ structs to database tables **automatically** at compile time — no macros, no external code generators, no runtime schema registration.

### Mapping Structs to SQL Tables

```cpp

#include <meta>
#include <string>
#include <format>

template<typename T>
consteval std::string generate_create_table() {
    std::string sql = "CREATE TABLE ";
    sql += std::meta::identifier_of(^^T);
    sql += " (";

    bool first = true;
    template for (constexpr auto m :
                  std::meta::nonstatic_data_members_of(^^T)) {
        if (!first) sql += ", ";
        sql += std::meta::identifier_of(m);
        sql += " ";
        sql += sql_type_name<typename [:std::meta::type_of(m):]>();
        first = false;
    }

    sql += ");";
    return sql;
}

template<typename T>
consteval auto sql_type_name() -> std::string {
    if constexpr (std::same_as<T, int>) return "INTEGER";
    else if constexpr (std::same_as<T, double>) return "REAL";
    else if constexpr (std::same_as<T, std::string>) return "TEXT";
    else if constexpr (std::same_as<T, bool>) return "BOOLEAN";
    else return "BLOB";
}

struct User {
    int id;
    std::string name;
    std::string email;
    int age;
};

// At compile time:
constexpr auto sql = generate_create_table<User>();
// sql == "CREATE TABLE User (id INTEGER, name TEXT, email TEXT, age INTEGER);"

```

### Generating INSERT Statements

```cpp

template<typename T>
std::string generate_insert(const T& obj) {
    std::string sql = "INSERT INTO ";
    sql += std::meta::identifier_of(^^T);
    sql += " (";

    // Column names
    bool first = true;
    template for (constexpr auto m :
                  std::meta::nonstatic_data_members_of(^^T)) {
        if (!first) sql += ", ";
        sql += std::meta::identifier_of(m);
        first = false;
    }

    sql += ") VALUES (";

    // Values
    first = true;
    template for (constexpr auto m :
                  std::meta::nonstatic_data_members_of(^^T)) {
        if (!first) sql += ", ";
        sql += to_sql_value(obj.[:m:]);
        first = false;
    }

    sql += ");";
    return sql;
}

std::string to_sql_value(int v) { return std::to_string(v); }
std::string to_sql_value(const std::string& v) {
    return "'" + v + "'";  // NOTE: use parameterized queries in production!
}

// Usage:
User u{1, "Alice", "alice@example.com", 30};
auto insert = generate_insert(u);
// "INSERT INTO User (id, name, email, age) VALUES (1, 'Alice', 'alice@example.com', 30);"

```

### Type-Safe Query Builder

```cpp

template<typename T>
struct query_builder {
    std::string select_clause_ = "*";
    std::string where_clause_;

    template<auto MemberPtr>
    query_builder& where_eq(const typename [:std::meta::type_of(MemberPtr):]& value) {
        where_clause_ = std::format("{} = {}",
            std::meta::identifier_of(MemberPtr),
            to_sql_value(value));
        return *this;
    }

    std::string build() const {
        std::string sql = std::format("SELECT {} FROM {}",
            select_clause_,
            std::meta::identifier_of(^^T));
        if (!where_clause_.empty())
            sql += " WHERE " + where_clause_;
        return sql + ";";
    }
};

// Usage:
auto q = query_builder<User>{}
    .where_eq<^^User::age>(30)
    .build();
// "SELECT * FROM User WHERE age = 30;"

```

### Mapping Query Results Back to Structs

```cpp

template<typename T>
T row_to_struct(const sql::Row& row) {
    T obj;
    int col = 0;
    template for (constexpr auto m :
                  std::meta::nonstatic_data_members_of(^^T)) {
        using MemberType = typename [:std::meta::type_of(m):];
        obj.[:m:] = row.get<MemberType>(col++);
    }
    return obj;
}

// Usage:
auto user = row_to_struct<User>(result_row);

```

---

## Self-Assessment

### Q1: Generate a CREATE TABLE statement for a given struct

See the `generate_create_table<T>()` function above. For `struct Order { int id; std::string product; double price; };`, it produces:

```sql

CREATE TABLE Order (id INTEGER, product TEXT, price REAL);

```

### Q2: Why is reflection-based ORM safer than macro-based approaches

| Property | Macros | Reflection |
| --- | --- | --- |
| Type safety | No | Yes — member types are checked at compile time |
| Refactoring | Fragile — renamed members silently break | Safe — renamed members update automatically |
| Compile errors | Cryptic | Clear — reflection errors are at the member level |
| Maintenance | Must manually sync macro definitions | Zero — struct definition IS the schema |

### Q3: What security concern exists in the SQL generation examples

**SQL injection!** The `to_sql_value(string)` function above uses string concatenation, which is vulnerable. In production, always use **parameterized queries**:

```cpp

template<typename T>
auto generate_parameterized_insert() {
    std::string sql = "INSERT INTO " + std::string(std::meta::identifier_of(^^T)) + " (";
    // ... column names ...
    sql += ") VALUES (";
    int n = std::meta::nonstatic_data_members_of(^^T).size();
    for (int i = 0; i < n; ++i) {
        if (i > 0) sql += ", ";
        sql += "?";  // parameterized placeholder
    }
    sql += ");";
    return sql;
}
// Then bind values through prepared statements, never string concatenation.

```

---

## Notes

- Reflection-based ORM eliminates the struct↔schema synchronization problem entirely.
- Always use parameterized queries in production — never concatenate user input into SQL.
- The compile-time nature means zero runtime overhead for schema validation.
- This pattern extends to any serialization format: JSON, MessagePack, Protocol Buffers.
