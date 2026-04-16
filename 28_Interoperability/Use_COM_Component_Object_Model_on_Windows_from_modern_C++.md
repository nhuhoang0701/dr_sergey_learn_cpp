# Use COM (Component Object Model) on Windows from modern C++

**Category:** Interoperability  
**Item:** #697  
**Standard:** C++11  
**Reference:** <https://docs.microsoft.com/en-us/windows/win32/com/>  

---

## Topic Overview

This topic focuses on **practical modern COM usage** in C++ — using smart pointers (`WRL::ComPtr`, `wil::com_ptr`), RAII initialization, and safe interface querying. COM is still essential for DirectX, Shell APIs, Media Foundation, and WinRT.

### COM Smart Pointer Comparison

| Feature | Raw `IUnknown*` | `WRL::ComPtr` | `wil::com_ptr` |
| --- | :---: | :---: | :---: |
| Auto Release | No | Yes | Yes |
| Header | N/A | `<wrl/client.h>` | `<wil/com.h>` |
| QueryInterface | Manual | `.As<T>()` | `.query<T>()` |
| Error handling | Manual HRESULT | Manual HRESULT | Throws on failure |
| Null check | Manual | `.Get() != nullptr` | `.get() != nullptr` |
| Part of SDK | — | Yes (Windows SDK) | No (NuGet: `wil`) |

### COM Initialization Pattern

```cpp

Program Start
    │
    ▼
CoInitializeEx(NULL, COINIT_APARTMENTTHREADED)
    │
    ▼
┌──────────────────────┐
│  Use COM objects     │ ← CoCreateInstance / QueryInterface
│  ComPtr handles      │
│  AddRef/Release      │
└──────────────────────┘
    │
    ▼
CoUninitialize()  ← MUST match every CoInitializeEx
    │
    ▼
Program End

```

---

## Self-Assessment

### Q1: Initialize COM with CoInitializeEx and wrap IUnknown in a wil::com_ptr for RAII

**Answer:**

```cpp

#include <windows.h>
#include <wrl/client.h>
#include <shobjidl.h>
#include <cstdio>

using Microsoft::WRL::ComPtr;

// ═══════════ RAII COM initializer ═══════════
struct ComInitializer {
    HRESULT hr;
    ComInitializer(DWORD model = COINIT_APARTMENTTHREADED) {
        hr = CoInitializeEx(nullptr, model);
    }
    ~ComInitializer() {
        if (SUCCEEDED(hr)) CoUninitialize();
    }
    bool ok() const { return SUCCEEDED(hr); }

    // Non-copyable
    ComInitializer(const ComInitializer&) = delete;
    ComInitializer& operator=(const ComInitializer&) = delete;
};

int main() {
    // RAII: COM initialized/uninitialized automatically
    ComInitializer com;
    if (!com.ok()) {
        printf("COM init failed: 0x%lx\n", com.hr);
        return 1;
    }

    // Create a Shell item from a path
    ComPtr<IShellItem> item;
    HRESULT hr = SHCreateItemFromParsingName(
        L"C:\\Windows\\System32\\notepad.exe",
        nullptr,
        IID_PPV_ARGS(&item)   // IID_PPV_ARGS extracts IID + **ptr
    );

    if (SUCCEEDED(hr)) {
        PWSTR name = nullptr;
        hr = item->GetDisplayName(SIGDN_NORMALDISPLAY, &name);
        if (SUCCEEDED(hr)) {
            printf("Item: %ls\n", name);
            CoTaskMemFree(name);  // COM strings allocated with CoTaskMemAlloc
        }
    }

    // item automatically Release'd when ComPtr destroyed
    // COM automatically uninitialized when ComInitializer destroyed
    return 0;
}
// Output: Item: notepad.exe

```

### Q2: Query an interface with QueryInterface and handle E_NOINTERFACE gracefully

**Answer:**

```cpp

#include <windows.h>
#include <wrl/client.h>
#include <shobjidl.h>
#include <shlobj.h>
#include <cstdio>

using Microsoft::WRL::ComPtr;

void query_interface_demo() {
    // Create a FileOpenDialog COM object
    ComPtr<IFileOpenDialog> dialog;
    HRESULT hr = CoCreateInstance(
        CLSID_FileOpenDialog,
        nullptr,
        CLSCTX_INPROC_SERVER,
        IID_PPV_ARGS(&dialog)
    );
    if (FAILED(hr)) return;

    // ═══════════ Method 1: ComPtr::As (recommended) ═══════════
    ComPtr<IFileDialog2> dialog2;
    hr = dialog.As(&dialog2);  // QueryInterface wrapper

    if (hr == E_NOINTERFACE) {
        printf("IFileDialog2 not supported on this OS\n");
    } else if (SUCCEEDED(hr)) {
        printf("IFileDialog2 available\n");
        // dialog2 can be used — auto Release on scope exit
    }

    // ═══════════ Method 2: Raw QueryInterface ═══════════
    ComPtr<IUnknown> unknown;
    hr = dialog->QueryInterface(IID_PPV_ARGS(&unknown));
    // This always succeeds — every COM object supports IUnknown

    // ═══════════ Method 3: Query for unrelated interface (will fail) ═══════════
    ComPtr<IShellItem> shell_item;
    hr = dialog.As(&shell_item);
    if (hr == E_NOINTERFACE) {
        printf("FileOpenDialog doesn't implement IShellItem (expected)\n");
    }

    // ═══════════ Safe pattern: attempt and use if available ═══════════
    ComPtr<IFileDialogCustomize> customizer;
    if (SUCCEEDED(dialog.As(&customizer))) {
        // Add a checkbox to the dialog
        customizer->AddCheckButton(100, L"Open as read-only", FALSE);
        customizer->AddComboBox(101);
        customizer->AddControlItem(101, 0, L"Option A");
        customizer->AddControlItem(101, 1, L"Option B");
    }
    // If QI failed, customizer is nullptr — RAII, no leak
}

// ═══════════ HRESULT error handling helper ═══════════
void check_hr(HRESULT hr, const char* context) {
    if (SUCCEEDED(hr)) return;

    switch (hr) {
        case E_NOINTERFACE:
            printf("%s: interface not supported\n", context);
            break;
        case E_OUTOFMEMORY:
            printf("%s: out of memory\n", context);
            break;
        case E_INVALIDARG:
            printf("%s: invalid argument\n", context);
            break;
        case REGDB_E_CLASSNOTREG:
            printf("%s: COM class not registered\n", context);
            break;
        default:
            printf("%s: failed with 0x%08lx\n", context, hr);
            break;
    }
}

```

### Q3: Use WRL::ComPtr or wil::com_ptr instead of raw IUnknown pointers for automatic Release

**Answer:**

```cpp

#include <windows.h>
#include <wrl/client.h>
#include <d2d1.h>
#include <cstdio>
#pragma comment(lib, "d2d1.lib")

using Microsoft::WRL::ComPtr;

// ═══════════ BAD: Raw pointers — easy to leak ═══════════
void raw_pointer_danger() {
    ID2D1Factory* factory = nullptr;
    HRESULT hr = D2D1CreateFactory(
        D2D1_FACTORY_TYPE_SINGLE_THREADED,
        &factory
    );
    if (FAILED(hr)) return;

    ID2D1HwndRenderTarget* target = nullptr;
    D2D1_RENDER_TARGET_PROPERTIES rtProps = D2D1::RenderTargetProperties();
    D2D1_HWND_RENDER_TARGET_PROPERTIES hwndProps = {};

    hr = factory->CreateHwndRenderTarget(rtProps, hwndProps, &target);
    if (FAILED(hr)) {
        factory->Release();  // Must remember to Release!
        return;              // If you forget → LEAK
    }

    // Use target...
    target->Release();      // Must Release in correct order
    factory->Release();     // Miss one → LEAK
}

// ═══════════ GOOD: ComPtr — automatic, exception-safe ═══════════
void comptr_safe() {
    ComPtr<ID2D1Factory> factory;
    HRESULT hr = D2D1CreateFactory(
        D2D1_FACTORY_TYPE_SINGLE_THREADED,
        IID_PPV_ARGS(&factory)
    );
    if (FAILED(hr)) return;

    ComPtr<ID2D1HwndRenderTarget> target;
    D2D1_RENDER_TARGET_PROPERTIES rtProps = D2D1::RenderTargetProperties();
    D2D1_HWND_RENDER_TARGET_PROPERTIES hwndProps = {};

    hr = factory->CreateHwndRenderTarget(
        rtProps, hwndProps, target.GetAddressOf()
    );
    if (FAILED(hr)) return;  // factory auto-Released, no leak

    // Use target...
    // Both auto-Released when function exits — even if exception thrown
}

// ═══════════ ComPtr operations cheat sheet ═══════════
void comptr_operations() {
    ComPtr<ID2D1Factory> ptr;

    // Create
    D2D1CreateFactory(D2D1_FACTORY_TYPE_SINGLE_THREADED, IID_PPV_ARGS(&ptr));

    // Copy (AddRef)
    ComPtr<ID2D1Factory> copy = ptr;

    // Move (no AddRef/Release)
    ComPtr<ID2D1Factory> moved = std::move(ptr);
    // ptr is now nullptr

    // Query interface
    ComPtr<IUnknown> unk;
    moved.As(&unk);  // QI + AddRef on new interface

    // Get raw pointer (no AddRef — don't Release it!)
    ID2D1Factory* raw = moved.Get();

    // Get address for output params (resets first!)
    ComPtr<ID2D1Factory> out;
    // SomeFunc(out.GetAddressOf());  // Releases old, stores new

    // Detach (takes ownership — YOU must Release)
    ID2D1Factory* detached = moved.Detach();
    detached->Release();  // Your responsibility

    // Reset (Release + set to nullptr)
    copy.Reset();

    // Swap
    ComPtr<ID2D1Factory> a, b;
    a.Swap(b);
}

```

---

## Notes

- `IID_PPV_ARGS(&ptr)` macro extracts the IID from the ComPtr's template type — prevents IID/type mismatches
- `wil::com_ptr` (Windows Implementation Libraries) throws `wil::ResultException` on failure instead of returning HRESULT — more C++-idiomatic
- Never call `Release()` on a pointer obtained via `ComPtr::Get()` — the ComPtr still owns it
- `ComPtr::ReleaseAndGetAddressOf()` resets and returns `**` — use for output parameters
- COM objects can live in DLLs or EXEs (out-of-process servers) — ComPtr handles both transparently
- For WinRT, prefer `winrt::com_ptr` from C++/WinRT over WRL
