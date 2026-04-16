# Write JNI (Java Native Interface) wrappers for Android C++ code

**Category:** Interoperability  
**Item:** #696  
**Reference:** <https://docs.oracle.com/en/java/javase/17/docs/specs/jni/>  

---

## Topic Overview

This topic covers **advanced JNI patterns**: implementing native methods with correct naming conventions, converting between `jstring` and `std::string` with proper UTF-8 handling, and using `RegisterNatives` for dynamic method registration (avoiding fragile name-mangled dispatch).

### Static vs Dynamic JNI Registration

```cpp

Static Registration (naming convention):
  Java: native void process()  in com.example.Foo
  C++:  Java_com_example_Foo_process(JNIEnv*, jobject)
  → JVM finds by name at runtime (slow first call, fragile names)

Dynamic Registration (RegisterNatives):
  C++:  any_name(JNIEnv*, jobject)  // any function name
  Register: JNINativeMethod methods[] = {
    {"process", "()V", (void*)any_name}
  };
  → Registered in JNI_OnLoad (fast, refactorable, less error-prone)

```

### String Conversion Pitfalls

| JNI Function | Returns | Encoding | Must Release? |
| --- | --- | --- | :---: |
| `GetStringUTFChars` | `const char*` | Modified UTF-8 | Yes (`ReleaseStringUTFChars`) |
| `GetStringChars` | `const jchar*` | UTF-16 | Yes (`ReleaseStringChars`) |
| `GetStringRegion` | copies into buffer | UTF-16 | No (no alloc) |
| `GetStringUTFRegion` | copies into buffer | Modified UTF-8 | No (no alloc) |
| `NewStringUTF` | `jstring` | Modified UTF-8 | No (new object) |

---

## Self-Assessment

### Q1: Implement a native method with the JNI signature Java_com_example_Foo_bar and register it

**Answer:**

```java

// ═══════════ Java side: com/example/crypto/CryptoLib.java ═══════════
package com.example.crypto;

public class CryptoLib {
    static { System.loadLibrary("crypto_jni"); }

    // Native methods
    public native byte[] encrypt(byte[] data, String key);
    public native byte[] decrypt(byte[] data, String key);
    public native String hash(String input, String algorithm);
}

```

```cpp

// ═══════════ C++ side: crypto_jni.cpp ═══════════
#include <jni.h>
#include <string>
#include <vector>
#include <numeric>

// ─── Static registration: name MUST match Java_<package>_<class>_<method> ───
extern "C" {

JNIEXPORT jbyteArray JNICALL
Java_com_example_crypto_CryptoLib_encrypt(
    JNIEnv* env,
    jobject thiz,       // 'this' in Java
    jbyteArray data,
    jstring key)
{
    // Get key
    const char* keyStr = env->GetStringUTFChars(key, nullptr);
    if (!keyStr) return nullptr;
    std::string keyStd(keyStr);
    env->ReleaseStringUTFChars(key, keyStr);

    // Get data array
    jbyte* dataPtr = env->GetByteArrayElements(data, nullptr);
    jint dataLen = env->GetArrayLength(data);

    // Simple XOR "encryption" (for demonstration — NOT secure!)
    std::vector<jbyte> encrypted(dataLen);
    for (jint i = 0; i < dataLen; ++i) {
        encrypted[i] = dataPtr[i] ^ keyStd[i % keyStd.size()];
    }

    env->ReleaseByteArrayElements(data, dataPtr, JNI_ABORT);

    // Create result array
    jbyteArray result = env->NewByteArray(dataLen);
    env->SetByteArrayRegion(result, 0, dataLen, encrypted.data());
    return result;
}

JNIEXPORT jstring JNICALL
Java_com_example_crypto_CryptoLib_hash(
    JNIEnv* env, jobject thiz,
    jstring input, jstring algorithm)
{
    const char* inputStr = env->GetStringUTFChars(input, nullptr);
    if (!inputStr) return nullptr;

    // Simple hash for demo
    uint64_t hash = 0;
    for (const char* p = inputStr; *p; ++p) {
        hash = hash * 31 + *p;
    }
    env->ReleaseStringUTFChars(input, inputStr);

    char hexBuf[17];
    snprintf(hexBuf, sizeof(hexBuf), "%016llx", (unsigned long long)hash);
    return env->NewStringUTF(hexBuf);
}

}  // extern "C"

```

### Q2: Convert between JNI jstring and std::string correctly including UTF-8 handling

**Answer:**

```cpp

#include <jni.h>
#include <string>

// ═══════════ RAII wrapper for JNI string access ═══════════
class JniString {
    JNIEnv* env_;
    jstring jstr_;
    const char* chars_;
public:
    JniString(JNIEnv* env, jstring jstr)
        : env_(env), jstr_(jstr), chars_(nullptr)
    {
        if (jstr_) {
            chars_ = env_->GetStringUTFChars(jstr_, nullptr);
        }
    }

    ~JniString() {
        if (chars_) {
            env_->ReleaseStringUTFChars(jstr_, chars_);
        }
    }

    // Convert to std::string (copies — safe after release)
    std::string str() const {
        return chars_ ? std::string(chars_) : std::string();
    }

    const char* c_str() const { return chars_; }
    bool valid() const { return chars_ != nullptr; }

    // Non-copyable
    JniString(const JniString&) = delete;
    JniString& operator=(const JniString&) = delete;
};

// ═══════════ UTF-8 handling details ═══════════
extern "C"
JNIEXPORT jstring JNICALL
Java_com_example_App_processText(JNIEnv* env, jobject thiz, jstring input)
{
    // Method 1: GetStringUTFChars (Modified UTF-8)
    // WARNING: JNI "Modified UTF-8" differs from standard UTF-8:
    //   - U+0000 encoded as 0xC0 0x80 (not 0x00)
    //   - Supplementary chars (U+10000+) use surrogate pairs in CESU-8
    {
        JniString str(env, input);
        if (!str.valid()) return nullptr;
        std::string processed = str.str();
        // processed is Modified UTF-8 — fine for ASCII, careful with emoji/CJK
    }

    // Method 2: GetStringChars (UTF-16) — correct for all Unicode
    {
        const jchar* utf16 = env->GetStringChars(input, nullptr);
        jint len = env->GetStringLength(input);

        // Process UTF-16 data...
        // Convert to std::u16string or std::wstring
        std::u16string u16str(reinterpret_cast<const char16_t*>(utf16), len);

        env->ReleaseStringChars(input, utf16);

        // For proper UTF-8: use ICU or manual conversion from UTF-16
    }

    // Method 3: GetStringUTFRegion (no allocation, no release needed)
    {
        jint len = env->GetStringUTFLength(input);
        std::string buf(len, '\0');
        env->GetStringUTFRegion(input, 0, env->GetStringLength(input), buf.data());
        // buf contains Modified UTF-8 — efficient, no release needed
    }

    // Return a new Java String from C++ string
    std::string result = "processed text";
    return env->NewStringUTF(result.c_str());
    // NewStringUTF expects Modified UTF-8 input
}

```

### Q3: Use RegisterNatives for dynamic JNI registration instead of name-mangled static dispatch

**Answer:**

```cpp

#include <jni.h>
#include <string>
#include <android/log.h>

#define TAG "NativeReg"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, TAG, __VA_ARGS__)

// ═══════════ Native implementations (any function name!) ═══════════
static jstring native_greet(JNIEnv* env, jobject thiz, jstring name) {
    const char* nameStr = env->GetStringUTFChars(name, nullptr);
    std::string greeting = "Hello, " + std::string(nameStr) + "!";
    env->ReleaseStringUTFChars(name, nameStr);
    return env->NewStringUTF(greeting.c_str());
}

static jint native_add(JNIEnv* env, jobject thiz, jint a, jint b) {
    return a + b;
}

static jdoubleArray native_linspace(
    JNIEnv* env, jobject thiz, jdouble start, jdouble end, jint count)
{
    jdoubleArray result = env->NewDoubleArray(count);
    if (count <= 1) {
        env->SetDoubleArrayRegion(result, 0, 1, &start);
        return result;
    }
    std::vector<jdouble> values(count);
    for (jint i = 0; i < count; ++i) {
        values[i] = start + (end - start) * i / (count - 1);
    }
    env->SetDoubleArrayRegion(result, 0, count, values.data());
    return result;
}

// ═══════════ Registration table ═══════════
// Format: { "javaMethodName", "(JNI signature)return", functionPointer }
static JNINativeMethod methods[] = {
    {"greet",    "(Ljava/lang/String;)Ljava/lang/String;", (void*)native_greet},
    {"add",      "(II)I",                                   (void*)native_add},
    {"linspace", "(DDI)[D",                                 (void*)native_linspace},
};

// ═══════════ JNI_OnLoad: register all native methods ═══════════
extern "C" JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {
    JNIEnv* env;
    if (vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6) != JNI_OK) {
        return JNI_ERR;
    }

    // Find the Java class
    jclass clazz = env->FindClass("com/example/myapp/MathLib");
    if (!clazz) {
        LOGI("ERROR: class not found");
        return JNI_ERR;
    }

    // Register ALL methods at once
    int count = sizeof(methods) / sizeof(methods[0]);
    if (env->RegisterNatives(clazz, methods, count) < 0) {
        LOGI("ERROR: RegisterNatives failed");
        return JNI_ERR;
    }

    LOGI("Registered %d native methods", count);
    env->DeleteLocalRef(clazz);
    return JNI_VERSION_1_6;
}

// ═══════════ Advantages over static registration ═══════════
//
// Static:  Java_com_example_myapp_MathLib_greet(...)
//   ✗ Rename package/class → must rename C++ function
//   ✗ Long ugly names
//   ✗ Symbol lookup at first call (slower)
//   ✗ Proguard can break it (name obfuscation)
//
// Dynamic: RegisterNatives in JNI_OnLoad
//   ✓ Any C++ function name
//   ✓ Rename via table update, not function rename
//   ✓ All methods registered upfront (fail-fast)
//   ✓ Works with Proguard (use @Keep on native methods)
//   ✓ Smaller symbol table (no exported Java_* names)

```

```java

// ═══════════ Java side: MathLib.java ═══════════
package com.example.myapp;

public class MathLib {
    static { System.loadLibrary("mathlib"); }

    // These names match the RegisterNatives table — NOT the C++ function names
    public native String greet(String name);
    public native int add(int a, int b);
    public native double[] linspace(double start, double end, int count);
}

```

---

## Notes

- `RegisterNatives` is the preferred approach for production Android apps
- JNI signatures use `L<fully/qualified/name>;` for objects, `[` prefix for arrays
- Use `javap -s com.example.MyClass` to generate correct JNI signatures
- Modified UTF-8 ≠ standard UTF-8 — beware with emoji and CJK supplementary characters
- `GetStringUTFRegion` avoids allocation — best for short strings you'll immediately copy
- Always check for `nullptr` after `GetStringUTFChars` — indicates `OutOfMemoryError`
- Keep native methods annotated with `@Keep` when using Proguard/R8

---

## Notes

_Add your own notes, examples, and observations here._

```cpp

// Your practice code

```
