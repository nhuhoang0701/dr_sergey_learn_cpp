# Use the Repository pattern to abstract data access in C++

**Category:** Project Architecture

---

## Topic Overview

The **Repository pattern** provides a collection-like interface for accessing domain objects, hiding the details of data storage (database, file, network). The domain code works with `IRepository<T>` — it doesn't know if data comes from SQLite, PostgreSQL, an in-memory map, or a REST API. This decouples persistence from business logic.

### Repository vs Direct Data Access

| Aspect | Direct Access | Repository Pattern |
| --- | --- | --- |
| Domain code | Contains SQL/file I/O | Pure business logic |
| Testing | Needs real DB | In-memory fake |
| Switching storage | Rewrite domain | Implement new repo |
| Query logic | Scattered | Centralized |
| Compile deps | Domain links sqlite3 | Domain links nothing |

---

## Self-Assessment

### Q1: Implement a generic repository interface and concrete implementations

**Answer:**

```cpp

#include <memory>
#include <optional>
#include <string>
#include <vector>
#include <functional>

// === Domain entity ===
struct User {
    int id = 0;
    std::string name;
    std::string email;
    bool active = true;
};

// === Repository interface (defined in domain layer) ===
template<typename T, typename Id = int>
class IRepository {
public:
    virtual ~IRepository() = default;
    virtual std::optional<T> find_by_id(Id id) const = 0;
    virtual std::vector<T> find_all() const = 0;
    virtual std::vector<T> find_where(
        std::function<bool(const T&)> predicate) const = 0;
    virtual void add(const T& entity) = 0;
    virtual void update(const T& entity) = 0;
    virtual void remove(Id id) = 0;
    virtual size_t count() const = 0;
};

using IUserRepository = IRepository<User>;

// === In-memory implementation (for testing) ===
class InMemoryUserRepository : public IUserRepository {
public:
    std::optional<User> find_by_id(int id) const override {
        auto it = std::find_if(users_.begin(), users_.end(),
            [id](const User& u) { return u.id == id; });
        if (it != users_.end()) return *it;
        return std::nullopt;
    }

    std::vector<User> find_all() const override {
        return users_;
    }

    std::vector<User> find_where(
            std::function<bool(const User&)> pred) const override {
        std::vector<User> result;
        std::copy_if(users_.begin(), users_.end(),
                     std::back_inserter(result), pred);
        return result;
    }

    void add(const User& user) override {
        auto u = user;
        u.id = next_id_++;
        users_.push_back(u);
    }

    void update(const User& user) override {
        auto it = std::find_if(users_.begin(), users_.end(),
            [&](const User& u) { return u.id == user.id; });
        if (it != users_.end()) *it = user;
    }

    void remove(int id) override {
        users_.erase(
            std::remove_if(users_.begin(), users_.end(),
                [id](const User& u) { return u.id == id; }),
            users_.end());
    }

    size_t count() const override { return users_.size(); }

private:
    std::vector<User> users_;
    int next_id_ = 1;
};

// === SQLite implementation (for production) ===
class SqliteUserRepository : public IUserRepository {
public:
    explicit SqliteUserRepository(const std::string& db_path) {
        sqlite3_open(db_path.c_str(), &db_);
    }
    ~SqliteUserRepository() { sqlite3_close(db_); }

    std::optional<User> find_by_id(int id) const override {
        sqlite3_stmt* stmt;
        sqlite3_prepare_v2(db_,
            "SELECT id, name, email, active FROM users WHERE id = ?",
            -1, &stmt, nullptr);
        sqlite3_bind_int(stmt, 1, id);

        std::optional<User> result;
        if (sqlite3_step(stmt) == SQLITE_ROW) {
            result = User{
                sqlite3_column_int(stmt, 0),
                reinterpret_cast<const char*>(sqlite3_column_text(stmt, 1)),
                reinterpret_cast<const char*>(sqlite3_column_text(stmt, 2)),
                sqlite3_column_int(stmt, 3) != 0
            };
        }
        sqlite3_finalize(stmt);
        return result;
    }
    // ... other methods similarly wrap SQL queries ...

private:
    sqlite3* db_ = nullptr;
};

```

### Q2: Use repository in domain service with tests

**Answer:**

```cpp

// === Domain service (depends only on IUserRepository) ===
class UserService {
public:
    explicit UserService(IUserRepository& repo) : repo_(repo) {}

    void deactivate_inactive_users(int days_threshold) {
        auto users = repo_.find_where([](const User& u) {
            return u.active;
        });
        for (auto& user : users) {
            // Business rule: deactivate if inactive
            user.active = false;
            repo_.update(user);
        }
    }

    std::optional<User> get_user(int id) {
        return repo_.find_by_id(id);
    }

    int active_user_count() {
        return static_cast<int>(
            repo_.find_where([](const User& u) { return u.active; }).size());
    }

private:
    IUserRepository& repo_;
};

// === Tests — fast, no database ===
#include <gtest/gtest.h>

TEST(UserServiceTest, DeactivatesAllActiveUsers) {
    InMemoryUserRepository repo;
    repo.add({0, "Alice", "a@b.com", true});
    repo.add({0, "Bob", "b@b.com", true});
    repo.add({0, "Charlie", "c@b.com", false});

    UserService service(repo);
    EXPECT_EQ(service.active_user_count(), 2);

    service.deactivate_inactive_users(30);
    EXPECT_EQ(service.active_user_count(), 0);
}

TEST(UserServiceTest, GetUserReturnsNulloptForMissing) {
    InMemoryUserRepository repo;
    UserService service(repo);
    EXPECT_FALSE(service.get_user(999).has_value());
}

```

### Q3: Add query specifications for complex filters

**Answer:**

```cpp

// === Specification pattern for complex queries ===
template<typename T>
class ISpecification {
public:
    virtual ~ISpecification() = default;
    virtual bool is_satisfied_by(const T& entity) const = 0;
};

class ActiveUsersSpec : public ISpecification<User> {
public:
    bool is_satisfied_by(const User& u) const override {
        return u.active;
    }
};

class EmailDomainSpec : public ISpecification<User> {
public:
    explicit EmailDomainSpec(std::string domain)
        : domain_(std::move(domain)) {}
    bool is_satisfied_by(const User& u) const override {
        return u.email.ends_with("@" + domain_);
    }
private:
    std::string domain_;
};

// Composite: AND specification
template<typename T>
class AndSpec : public ISpecification<T> {
public:
    AndSpec(const ISpecification<T>& a, const ISpecification<T>& b)
        : a_(a), b_(b) {}
    bool is_satisfied_by(const T& entity) const override {
        return a_.is_satisfied_by(entity) && b_.is_satisfied_by(entity);
    }
private:
    const ISpecification<T>& a_;
    const ISpecification<T>& b_;
};

// Extended repository with specification support
class IUserRepositoryEx : public IUserRepository {
public:
    virtual std::vector<User> find_matching(
        const ISpecification<User>& spec) const {
        return find_where([&spec](const User& u) {
            return spec.is_satisfied_by(u);
        });
    }
};

// Usage:
// ActiveUsersSpec active;
// EmailDomainSpec company("company.com");
// AndSpec<User> active_company(active, company);
// auto users = repo.find_matching(active_company);

```

---

## Notes

- Repository is defined in the **domain layer** — concrete implementations live in **infrastructure**
- `InMemoryRepository` is both a test double and a useful prototype implementation
- The domain service should never know the storage technology
- For simple CRUD, a generic `IRepository<T>` works; add domain-specific methods as needed
- Don't expose database concepts (transactions, connections) through the repository interface
- Specification pattern keeps complex query logic testable and composable
- In production, the SQLite/PostgreSQL repository translates specifications to SQL for efficiency
