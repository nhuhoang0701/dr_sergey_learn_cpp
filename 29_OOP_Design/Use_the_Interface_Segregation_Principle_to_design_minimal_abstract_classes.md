# Use the Interface Segregation Principle to design minimal abstract classes

**Category:** OOP Design

---

## Topic Overview

**Interface Segregation Principle (ISP):** Clients should not be forced to depend on interfaces they don't use. In C++, this means splitting "fat" abstract base classes into small, focused interfaces.

### Fat vs Segregated Interfaces

```cpp

FAT interface (violation):              SEGREGATED interfaces:
┌─────────────────────┐               ┌──────────┐ ┌──────────┐ ┌──────────┐
│ IDevice              │               │ IReadable│ │ IWritable│ │ ISeekable│
│ - read()            │     →         │ - read() │ │ - write()│ │ - seek() │
│ - write()           │               └──────────┘ └──────────┘ └──────────┘
│ - seek()            │
│ - flush()           │               ┌──────────┐
│ - get_status()      │               │IFlushable│
│ - set_baudrate()    │               │ - flush()│
│ - calibrate()       │               └──────────┘
└─────────────────────┘
Read-only devices must implement write()=no-op → violation

```

---

## Self-Assessment

### Q1: Refactor a fat interface into segregated interfaces

**Answer:**

```cpp

#include <cstdint>
#include <span>
#include <string>
#include <memory>
#include <vector>

// ═══════════ BAD: Fat interface — forces unnecessary implementations ═══════════
class IDocument {
public:
    virtual ~IDocument() = default;
    virtual std::string read() = 0;
    virtual void write(std::string_view content) = 0;
    virtual void print() = 0;
    virtual void fax() = 0;              // Not all documents can be faxed!
    virtual void encrypt() = 0;           // Not all need encryption!
    virtual void export_pdf() = 0;        // Not all can export to PDF!
};

// ReadOnlyDocument is FORCED to implement write(), fax(), encrypt()...
class ReadOnlyDocument : public IDocument {
    void write(std::string_view) override { throw std::runtime_error("Read-only!"); }
    void fax() override { throw std::runtime_error("Not supported!"); }
    void encrypt() override { throw std::runtime_error("Not needed!"); }
    // These throw at RUNTIME — should be caught at COMPILE TIME
};

// ═══════════ GOOD: Segregated interfaces ═══════════
class IReadable {
public:
    virtual ~IReadable() = default;
    virtual std::string read() const = 0;
};

class IWritable {
public:
    virtual ~IWritable() = default;
    virtual void write(std::string_view content) = 0;
};

class IPrintable {
public:
    virtual ~IPrintable() = default;
    virtual void print() const = 0;
};

class IEncryptable {
public:
    virtual ~IEncryptable() = default;
    virtual void encrypt(std::span<const uint8_t> key) = 0;
    virtual void decrypt(std::span<const uint8_t> key) = 0;
};

class IExportable {
public:
    virtual ~IExportable() = default;
    virtual std::vector<uint8_t> export_pdf() const = 0;
};

// Now each class implements ONLY what it supports:
class TextDocument : public IReadable, public IWritable, public IPrintable {
    std::string content_;
public:
    std::string read() const override { return content_; }
    void write(std::string_view c) override { content_ = c; }
    void print() const override { /* send to printer */ }
};

class SecureDocument : public IReadable, public IWritable, public IEncryptable {
    std::string content_;
    bool encrypted_ = false;
public:
    std::string read() const override { return content_; }
    void write(std::string_view c) override { content_ = c; }
    void encrypt(std::span<const uint8_t> key) override { encrypted_ = true; }
    void decrypt(std::span<const uint8_t> key) override { encrypted_ = false; }
};

class ReadOnlyReport : public IReadable, public IPrintable, public IExportable {
    std::string data_;
public:
    explicit ReadOnlyReport(std::string data) : data_(std::move(data)) {}
    std::string read() const override { return data_; }
    void print() const override { /* print report */ }
    std::vector<uint8_t> export_pdf() const override { return {}; }
    // NO write, NO encrypt — compile-time enforcement!
};

// Functions accept only the interface they need:
void backup(const IReadable& doc) {       // Only needs to read
    auto content = doc.read();
}

void edit(IWritable& doc) {               // Only needs to write
    doc.write("Updated content");
}

void secure_and_store(IReadable& r, IEncryptable& e) {
    auto data = r.read();
    uint8_t key[] = {1,2,3,4};
    e.encrypt(key);
}

```

### Q2: Explain ISP in the context of embedded C++ hardware interfaces

**Answer:**

```cpp

#include <cstdint>

// ═══════════ Embedded example: Sensor interfaces ═══════════

// BAD: One fat sensor interface
class ISensor {
public:
    virtual ~ISensor() = default;
    virtual double read() = 0;
    virtual void calibrate(double ref) = 0;  // Some sensors can't be calibrated!
    virtual void set_range(int range) = 0;   // Some have fixed range!
    virtual void enable_interrupt() = 0;      // Some are polled only!
    virtual void sleep() = 0;                 // Some can't sleep!
};

// GOOD: Segregated sensor interfaces
class IReadable_Sensor {
public:
    virtual ~IReadable_Sensor() = default;
    virtual double read() = 0;
};

class ICalibratable {
public:
    virtual ~ICalibratable() = default;
    virtual bool calibrate(double reference) = 0;
};

class IConfigurable {
public:
    virtual ~IConfigurable() = default;
    virtual void set_range(int range) = 0;
    virtual int get_range() const = 0;
};

class IInterruptDriven {
public:
    virtual ~IInterruptDriven() = default;
    using Callback = void(*)(double value, void* ctx);
    virtual void enable_interrupt(Callback cb, void* ctx) = 0;
    virtual void disable_interrupt() = 0;
};

class IPowerManaged {
public:
    virtual ~IPowerManaged() = default;
    virtual void sleep() = 0;
    virtual void wake() = 0;
    virtual double power_consumption_mw() const = 0;
};

// Concrete sensors implement only applicable interfaces:
class SimpleThermistor : public IReadable_Sensor {
    volatile uint32_t* adc_;
public:
    explicit SimpleThermistor(uint32_t* adc) : adc_(adc) {}
    double read() override { return (*adc_) * 0.01 - 40.0; }
    // No calibration, no interrupts, no power management
};

class AccelerometerIMU : public IReadable_Sensor,
                         public ICalibratable,
                         public IConfigurable,
                         public IInterruptDriven,
                         public IPowerManaged {
    int range_ = 2;
public:
    double read() override { return 0.0; /* read SPI */ }
    bool calibrate(double ref) override { return true; }
    void set_range(int r) override { range_ = r; }
    int get_range() const override { return range_; }
    void enable_interrupt(Callback cb, void* ctx) override { /* configure EXTI */ }
    void disable_interrupt() override { /* disable EXTI */ }
    void sleep() override { /* set low-power mode register */ }
    void wake() override { /* clear low-power bit */ }
    double power_consumption_mw() const override { return 0.5; }
};

// Power management code only depends on IPowerManaged:
void enter_low_power(std::vector<IPowerManaged*>& devices) {
    for (auto* d : devices) d->sleep();
}
// SimpleThermistor is NOT passed here — compile-time safety

```

### Q3: Show ISP with concepts (C++20) as an alternative to abstract classes

**Answer:**

```cpp

#include <concepts>
#include <string>
#include <iostream>

// ═══════════ ISP with concepts — zero-overhead, compile-time interfaces ═══════════

template<typename T>
concept Readable = requires(const T& t) {
    { t.read() } -> std::convertible_to<std::string>;
};

template<typename T>
concept Writable = requires(T& t, std::string_view sv) {
    { t.write(sv) };
};

template<typename T>
concept Serializable = requires(const T& t) {
    { t.serialize() } -> std::convertible_to<std::vector<uint8_t>>;
    { T::deserialize(std::declval<std::span<const uint8_t>>()) } -> std::same_as<T>;
};

// Functions constrained to exactly the interface they need:
void backup(const Readable auto& doc) {
    auto content = doc.read();
    // Only requires read() — nothing else
}

void update(Writable auto& doc) {
    doc.write("new content");
}

void sync(Readable auto& r, Writable auto& w) {
    w.write(r.read());
}

// Concrete types — no inheritance needed!
struct PlainTextFile {
    std::string content;
    std::string read() const { return content; }
    void write(std::string_view s) { content = s; }
};

struct ConfigFile {
    std::string data;
    std::string read() const { return data; }
    // No write() — it's read-only
};

int main() {
    PlainTextFile f{"hello"};
    ConfigFile c{"key=value"};

    backup(f);   // OK: PlainTextFile satisfies Readable
    backup(c);   // OK: ConfigFile satisfies Readable
    update(f);   // OK: PlainTextFile satisfies Writable
    // update(c); // COMPILE ERROR: ConfigFile doesn't satisfy Writable
    return 0;
}

```

---

## Notes

- ISP in C++: prefer many small ABCs (2-4 methods) over one large one
- With C++20 concepts, ISP is enforced at compile time with zero overhead
- Fat interfaces force implementers to throw `std::runtime_error("Not supported")` — a design smell
- In embedded: ISP lets you write power management, calibration, and interrupt code that accepts only devices with those capabilities
- Multiple inheritance of pure interfaces is fine in C++ (unlike some languages) — no diamond problem with stateless ABCs
