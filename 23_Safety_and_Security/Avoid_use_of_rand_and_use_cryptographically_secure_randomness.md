# Avoid use of rand() and use cryptographically secure randomness

**Category:** Safety & Security  
**Item:** #737  
**Reference:** <https://en.cppreference.com/w/cpp/numeric/random/random_device>  

---

## Topic Overview

This topic covers why `rand()` is broken for security and how to use OS entropy and CSPRNG — complementing #653 (seeding mt19937 properly, OS entropy for key material).

```cpp

Randomness quality ladder:

  rand()              -> Predictable, biased, NOT for security
  std::mt19937        -> Statistical quality, NOT cryptographic
  std::random_device  -> OS entropy (usually /dev/urandom or CryptGenRandom)
  getrandom()         -> Direct OS entropy (Linux 3.17+)
  BCryptGenRandom()   -> Direct OS entropy (Windows)
  Hardware RNG (RDRAND) -> CPU instruction, feeds OS pool

```

| Source | Uniform? | Unpredictable? | Speed | Use for |
| --- | --- | --- | --- | --- |
| `rand()` | No (modulo bias) | No (LCG) | Fast | Nothing (deprecated) |
| `mt19937` | Yes | No (624 outputs reveal state) | Fast | Simulations, games |
| `random_device` | Yes | Yes (OS) | Slow | Seeding PRNGs |
| `getrandom(2)` | Yes | Yes | Slow | Crypto keys, tokens |

---

## Self-Assessment

### Q1: rand() problems

```cpp

#include <iostream>
#include <cstdlib>
#include <ctime>
#include <map>

int main() {
    // Problem 1: Predictable - same seed = same sequence
    srand(42);
    std::cout << "seed=42: ";
    for (int i = 0; i < 5; ++i) std::cout << rand() << ' ';
    std::cout << '\n';

    srand(42);  // reset seed
    std::cout << "seed=42: ";
    for (int i = 0; i < 5; ++i) std::cout << rand() << ' ';
    std::cout << '\n';
    // Identical output! Attacker knowing seed can predict ALL values.

    // Problem 2: Common srand(time(0)) has ~1s resolution
    // Attacker guesses current time -> guesses seed -> predicts output
    srand(time(nullptr));
    std::cout << "\nsrand(time(0)): " << rand() << '\n';
    // Attacker at same second gets same value!

    // Problem 3: Modulo bias
    // rand() % 10 is NOT uniform if RAND_MAX+1 is not divisible by 10
    srand(0);
    std::map<int, int> hist;
    for (int i = 0; i < 100000; ++i)
        ++hist[rand() % 3];  // biased!

    std::cout << "\nrand() % 3 distribution (should be ~33333 each):\n";
    for (auto& [val, count] : hist)
        std::cout << "  " << val << ": " << count << '\n';
    // RAND_MAX varies by platform; bias = (RAND_MAX+1) % 3

    // Problem 4: Low bits often have short periods (LCG)
    std::cout << "\nLow bit pattern: ";
    srand(1);
    for (int i = 0; i < 20; ++i)
        std::cout << (rand() & 1);  // may alternate 0,1,0,1,...
    std::cout << '\n';
}

```

### Q2: std::random_device + CSPRNG

```cpp

#include <iostream>
#include <random>
#include <array>
#include <algorithm>

int main() {
    // random_device: OS-provided entropy
    std::random_device rd;

    // Check entropy source
    std::cout << "random_device entropy: " << rd.entropy() << '\n';
    // Linux: typically 32 (bits), Windows MSVC: 32
    // Note: MinGW may return 0 (deterministic!) - check your platform!

    // Generate cryptographically random values
    std::cout << "\nOS random values: ";
    for (int i = 0; i < 5; ++i)
        std::cout << rd() << ' ';
    std::cout << '\n';

    // For non-crypto use: seed mt19937 PROPERLY
    // Bad:  mt19937 gen(rd());  // only 32 bits of seed for 19937-bit state!
    // Good: seed the FULL state (624 * 32 = 19968 bits)
    std::array<std::random_device::result_type, std::mt19937::state_size> seed_data;
    std::generate(seed_data.begin(), seed_data.end(), std::ref(rd));
    std::seed_seq seeds(seed_data.begin(), seed_data.end());
    std::mt19937 gen(seeds);  // fully seeded!

    // Use uniform distribution (no modulo bias!)
    std::uniform_int_distribution<int> dist(1, 100);
    std::cout << "\nUniform [1,100]: ";
    for (int i = 0; i < 10; ++i)
        std::cout << dist(gen) << ' ';
    std::cout << '\n';

    // For security-sensitive: use random_device directly (slow but secure)
    std::uniform_int_distribution<uint64_t> token_dist;
    uint64_t auth_token = token_dist(rd);  // crypto-quality
    std::cout << "\nAuth token: " << auth_token << '\n';
}

```

### Q3: OS APIs for crypto random bytes

```cpp

#include <iostream>
#include <cstring>
#include <iomanip>
#include <array>

#ifdef _WIN32
  #include <windows.h>
  #include <bcrypt.h>
  #pragma comment(lib, "bcrypt.lib")
#else
  #include <sys/random.h>  // getrandom (Linux 3.17+, glibc 2.25+)
#endif

// Cross-platform secure random bytes
bool get_random_bytes(void* buf, size_t len) {
#ifdef _WIN32
    // BCryptGenRandom: Windows Vista+
    NTSTATUS status = BCryptGenRandom(
        nullptr,
        static_cast<PUCHAR>(buf),
        static_cast<ULONG>(len),
        BCRYPT_USE_SYSTEM_PREFERRED_RNG);
    return BCRYPT_SUCCESS(status);
#else
    // getrandom: blocks until entropy pool initialized, no fd needed
    ssize_t n = getrandom(buf, len, 0);
    return (n == static_cast<ssize_t>(len));
#endif
}

int main() {
    // Generate 32 bytes of crypto-quality randomness
    std::array<unsigned char, 32> key;
    if (!get_random_bytes(key.data(), key.size())) {
        std::cerr << "Failed to get random bytes!\n";
        return 1;
    }

    std::cout << "Crypto key (32 bytes): ";
    for (unsigned char b : key)
        std::cout << std::hex << std::setw(2) << std::setfill('0')
                  << static_cast<int>(b);
    std::cout << '\n';

    // Generate a session token
    uint64_t token;
    get_random_bytes(&token, sizeof(token));
    std::cout << "Session token: " << std::dec << token << '\n';

    std::cout << "\nOS entropy sources:\n";
    std::cout << "  Linux: getrandom(2) -> /dev/urandom pool\n";
    std::cout << "  Windows: BCryptGenRandom -> CNG\n";
    std::cout << "  macOS: getentropy(2) or arc4random_buf()\n";
    std::cout << "  All feed from hardware entropy (RDRAND, interrupt timing)\n";
}

```

---

## Notes

- Complementary to #653 (seeding mt19937 + OS entropy details).
- `std::random_device` on MinGW may be deterministic! Always verify `entropy() > 0`.
- For crypto: never use mt19937 (observing 624 outputs reveals the entire state).
- `getrandom(buf, len, GRND_RANDOM)` blocks if pool empty; `GRND_NONBLOCK` returns error.
- CWE-338: Use of Cryptographically Weak PRNG — one of the most common security bugs.
