# Use std::chrono calendar and timezone features (C++20)

**Category:** Standard Library Utilities  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/chrono>  

---

## Topic Overview

C++20 extends `<chrono>` with calendar types (year, month, day), timezone support, and formatting - replacing the error-prone C `<ctime>` API. The old `<ctime>` API is a grab-bag of functions that use global buffers, give you raw integers with no units, and silently let you mix days with seconds. The C++20 chrono additions give you actual types for calendar concepts, so the compiler catches mistakes at compile time.

### Calendar Types

Here's the new syntax for working with dates. The `/` operator is overloaded to build calendar types, which is unusual but quite readable once you're used to it:

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

Notice `.ok()` - the type tracks whether the date is valid, so you can construct `February/30` without an immediate crash and then check it.

### Time Zones

`zoned_time` pairs a time point with a timezone and handles all the offset and DST arithmetic for you. You just name the IANA timezone and let the library convert:

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

The chrono literals (`h`, `min`, `s`) make duration arithmetic very clean. You can add durations of different units directly:

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

`system_clock` measures Unix time, which ignores leap seconds - it pretends each day has exactly 86400 seconds. `utc_clock` accounts for leap seconds, so it represents the actual elapsed time since the UTC epoch. For most applications, `system_clock` is the right choice. You only need `utc_clock` if you're doing sub-second calculations that must remain correct across a leap second boundary.

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

`floor<days>` truncates the current time point to midnight, then `weekday{today}` extracts the day of week. The subtraction `Friday - wd` gives a signed count of days to advance - if the result is zero or negative (today is Friday or past it in the week), you add 7 to get next Friday.

### Q3: Why use std::chrono over ctime

`<chrono>` gives you type safety so you can't accidentally add seconds to milliseconds without a cast, no string parsing for timezones, no thread-safety issues (`localtime` uses a static buffer that can be clobbered from another thread), and compile-time unit conversion via `std::ratio`. The short answer is that `chrono` prevents the most common time-handling bugs at the type level, while `<ctime>` lets you commit all of them silently.

---

## Notes

- MSVC and GCC have full timezone support; Clang/libc++ timezone support arrived in LLVM 19.
- The IANA timezone database is bundled or loaded from the OS.
- `std::format` supports chrono types: `std::format("{:%Y-%m-%d %H:%M}", time_point)`.
- Always use `floor<days>` to truncate time points to dates (not `time_point_cast` which rounds).
