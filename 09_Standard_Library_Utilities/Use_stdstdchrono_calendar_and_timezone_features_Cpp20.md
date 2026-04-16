# Use std::chrono calendar and timezone features (C++20)

**Category:** Standard Library Utilities  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/chrono>  

---

## Topic Overview

C++20 extends `<chrono>` with calendar types (year, month, day), timezone support, and formatting — replacing the error-prone C `<ctime>` API.

### Calendar Types

```cpp

#include <chrono>
#include <iostream>

using namespace std::chrono;

int main() {
    // Create calendar dates
    auto today = year{2024}/December/25;  // 2024-12-25
    auto also_today = 2024y/12/25d;       // Same thing

    std::cout << today << "\n";           // 2024-12-25

    // Check validity
    auto bad = 2024y/February/30d;
    std::cout << bad.ok() << "\n";        // false (Feb 30 doesn't exist)

    // Day of week
    auto weekday = year_month_day{today};
    auto wd = std::chrono::weekday{sys_days{weekday}};
    std::cout << wd << "\n";              // Wed

    // Last day of month
    auto last_feb = 2024y/February/last;
    year_month_day last_day{last_feb};
    std::cout << last_day << "\n";        // 2024-02-29 (leap year!)
}

```

### Time Zones

```cpp

#include <chrono>
#include <iostream>

using namespace std::chrono;

void timezone_example() {
    // Current time in UTC
    auto now = system_clock::now();
    std::cout << "UTC: " << now << "\n";

    // Convert to a specific timezone
    auto ny = locate_zone("America/New_York");
    auto local = zoned_time{ny, now};
    std::cout << "NYC: " << local << "\n";

    auto tokyo = zoned_time{"Asia/Tokyo", now};
    std::cout << "Tokyo: " << tokyo << "\n";

    // Timezone arithmetic
    auto meeting_utc = sys_days{2024y/June/15} + 14h;
    auto meeting_ny = zoned_time{"America/New_York", meeting_utc};
    auto meeting_london = zoned_time{"Europe/London", meeting_utc};
    std::cout << "Meeting: " << meeting_ny << " / " << meeting_london << "\n";
}

```

### Duration Arithmetic

```cpp

#include <chrono>
#include <iostream>

void duration_example() {
    using namespace std::chrono_literals;
    auto duration = 3h + 45min + 30s;
    std::cout << duration << "\n";  // 13530s

    // Format as hours:minutes:seconds
    auto h = duration_cast<hours>(duration);
    auto m = duration_cast<minutes>(duration - h);
    auto s = duration - h - m;
    std::cout << h.count() << "h " << m.count() << "m " << s.count() << "s\n";
}

```

---

## Self-Assessment

### Q1: What's the difference between system_clock and utc_clock

`system_clock` measures Unix time (ignoring leap seconds). `utc_clock` accounts for leap seconds. For most applications, `system_clock` is correct. Use `utc_clock` only if you need sub-second accuracy across leap second boundaries.

### Q2: How to find the next occurrence of a weekday

```cpp

auto next_friday() {
    auto today = floor<days>(system_clock::now());
    auto wd = weekday{today};
    auto days_until = (Friday - wd).count();
    if (days_until <= 0) days_until += 7;
    return today + days{days_until};
}

```

### Q3: Why use std::chrono over ctime

Type safety (can't mix seconds and milliseconds), no string parsing for timezones, no thread-safety issues (`localtime` uses static buffer), and compile-time unit conversion. `chrono` prevents the most common time-handling bugs at the type level.

---

## Notes

- MSVC and GCC have full timezone support; Clang/libc++ timezone support arrived in LLVM 19.
- The IANA timezone database is bundled or loaded from the OS.
- `std::format` supports chrono types: `std::format("{:%Y-%m-%d %H:%M}", time_point)`.
- Always use `floor<days>` to truncate time points to dates (not `time_point_cast` which rounds).
