# Use AI to explain and document complex C++ template metaprogramming

**Category:** AI-Assisted C++ Development

---

## Topic Overview

Template metaprogramming (TMP) is one of the most complex areas of C++. AI assistants excel at **explaining** existing TMP code, **generating documentation** for template libraries, **simplifying** complex patterns, and **teaching** TMP concepts with step-by-step instantiation traces. This makes previously impenetrable code accessible.

### AI TMP Assistance Areas

| Task | AI Effectiveness | Key Prompt Strategy |
| --- | --- | --- |
| Explain existing TMP code | Excellent | Paste code + "explain step by step" |
| Generate instantiation traces | Excellent | "Show how this instantiates for T=int" |
| Simplify SFINAE to concepts | Good | Paste SFINAE + target standard |
| Document template APIs | Excellent | Paste template + "document for users" |
| Debug template errors | Good | Paste full error + source |
| Write new TMP from scratch | Medium | Very specific requirements needed |

---

## Self-Assessment

### Q1: Use AI to explain complex TMP code

**Answer:**

```cpp

=== PROMPT ===

"Explain this template metaprogramming code step by step.
Show the instantiation trace for TypeList<int, double, string>.
Explain what each template does and why."

```

```cpp

// Complex TMP code to explain:
template<typename... Ts> struct TypeList {};

// Get element at index
template<size_t N, typename List> struct TypeAt;
template<size_t N, typename Head, typename... Tail>
struct TypeAt<N, TypeList<Head, Tail...>>
    : TypeAt<N-1, TypeList<Tail...>> {};
template<typename Head, typename... Tail>
struct TypeAt<0, TypeList<Head, Tail...>> {
    using type = Head;
};

// Transform each type in a list
template<template<typename> class F, typename List>
struct Transform;
template<template<typename> class F, typename... Ts>
struct Transform<F, TypeList<Ts...>> {
    using type = TypeList<typename F<Ts>::type...>;
};

// Filter types matching a predicate
template<template<typename> class Pred, typename List>
struct Filter;
template<template<typename> class Pred>
struct Filter<Pred, TypeList<>> {
    using type = TypeList<>;
};
template<template<typename> class Pred, typename Head, typename... Tail>
struct Filter<Pred, TypeList<Head, Tail...>> {
    using Rest = typename Filter<Pred, TypeList<Tail...>>::type;
    using type = std::conditional_t<
        Pred<Head>::value,
        typename Prepend<Head, Rest>::type,
        Rest>;
};


// === AI EXPLANATION ===
//
// "This is a compile-time type list manipulation library.
//
// 1. TypeList<Ts...>: A variadic template that holds a
//    list of types. Like std::tuple but for types only.
//
// 2. TypeAt<N, List>: Gets the Nth type from a TypeList.
//    Works by recursion:
//    - TypeAt<2, TypeList<int,double,string>>
//    - -> TypeAt<1, TypeList<double,string>>     (N-1, drop Head)
//    - -> TypeAt<0, TypeList<string>>             (N-1, drop Head)
//    - -> type = string                           (N=0, take Head)
//
// 3. Transform<F, List>: Applies a metafunction F to each
//    type. Like std::transform but at compile time.
//    Transform<std::add_pointer, TypeList<int,double>>
//    -> TypeList<int*, double*>
//
// 4. Filter<Pred, List>: Keeps only types where Pred<T>
//    is true. Like std::copy_if at compile time.
//    Filter<std::is_integral, TypeList<int,double,char>>
//    -> TypeList<int, char>"

```

### Q2: Generate documentation for template libraries

**Answer:**

```cpp

=== PROMPT ===

"Generate user-facing documentation for this template
matrix class. Include:

- Brief description of each template parameter
- Constraints and requirements
- Usage examples with expected output
- Common error messages and what they mean
- Performance characteristics"

```

```cpp

// Code to document:
template<typename T, size_t Rows, size_t Cols,
         MatrixLayout Layout = MatrixLayout::RowMajor>
    requires std::is_arithmetic_v<T>
class Matrix { /*...*/ };

// === AI-generated documentation ===

/// @brief Fixed-size matrix with compile-time dimensions.
///
/// @tparam T      Element type. Must be arithmetic (int, float,
///                double, etc.). Complex types not supported.
/// @tparam Rows   Number of rows (must be > 0).
/// @tparam Cols   Number of columns (must be > 0).
/// @tparam Layout Memory layout: RowMajor (default) or ColMajor.
///                RowMajor: elements in a row are contiguous.
///                ColMajor: elements in a column are contiguous.
///                Choose ColMajor for interop with Fortran/LAPACK.
///
/// ## Quick Start
/// ```cpp
/// // 3x3 identity matrix
/// Matrix<double, 3, 3> I = Matrix<double,3,3>::identity();
///
/// // Matrix multiplication (dimensions checked at compile time)
/// Matrix<float, 2, 3> A = {{1,2,3}, {4,5,6}};
/// Matrix<float, 3, 2> B = {{1,2}, {3,4}, {5,6}};
/// auto C = A * B;  // Matrix<float, 2, 2>
///
/// // Dimension mismatch = compile error
/// // auto bad = A * A;  // ERROR: 2x3 * 2x3 doesn't match
/// ```
///
/// ## Common Errors
/// | Error Message | Cause | Fix |
/// |---|---|---|
/// | "constraints not satisfied" | T is not arithmetic | Use int/float/double |
/// | "no match for operator*" | Dimension mismatch | Check Cols_A == Rows_B |
/// | "cannot convert" | Layout mismatch | Explicit convert or match layouts |
///
/// ## Performance
/// - Stack-allocated for small matrices (no heap)
/// - Dimensions known at compile time enable SIMD vectorization
/// - RowMajor: fast row iteration, slow column iteration
/// - ColMajor: fast column iteration, slow row iteration

```

### Q3: Use AI to trace template instantiation

**Answer:**

```cpp

=== PROMPT ===

"Trace the template instantiation of this expression:
  auto result = serialize(std::vector<std::pair<int, std::string>>{});

Show every template that gets instantiated, in order,
with the concrete types substituted."

```

```cpp

// Serialization library with ADL + SFINAE
template<typename T>
auto serialize(const T& val) -> decltype(val.serialize()) {
    return val.serialize();  // Has .serialize() method
}

template<typename T>
auto serialize(const T& val)
    -> std::enable_if_t<std::is_arithmetic_v<T>, std::string> {
    return std::to_string(val);  // Arithmetic types
}

template<typename T>
std::string serialize(const std::vector<T>& vec) {
    std::string result = "[";
    for (size_t i = 0; i < vec.size(); ++i) {
        if (i > 0) result += ",";
        result += serialize(vec[i]);  // Recurse
    }
    return result + "]";
}

template<typename K, typename V>
std::string serialize(const std::pair<K, V>& p) {
    return "{" + serialize(p.first) + ":" + serialize(p.second) + "}";
}

// === AI INSTANTIATION TRACE ===
//
// serialize(std::vector<std::pair<int, std::string>>{})
//
// 1. Try overload 1: val.serialize() -> SFINAE fail
//    (vector has no .serialize())
// 2. Try overload 2: is_arithmetic_v<vector<...>> -> false
//    SFINAE fail
// 3. Try overload 3: serialize(const vector<T>&)
//    -> MATCH with T = pair<int, string>
//    Instantiates: serialize<pair<int,string>>(vector<...>)
//
// Inside the loop, calls serialize(vec[i]):
// 4. Try overload 1: pair has no .serialize() -> fail
// 5. Try overload 2: is_arithmetic_v<pair> -> false -> fail
// 6. Try overload 3: not a vector -> fail
// 7. Try overload 4: serialize(const pair<K,V>&)
//    -> MATCH with K=int, V=string
//
// For p.first (int):
// 8. Overload 2 matches: is_arithmetic_v<int> = true
//    Returns: std::to_string(42) -> "42"
//
// For p.second (string):
// 9. Overload 1: string has no .serialize() -> fail
// 10. Overload 2: is_arithmetic_v<string> -> false -> fail
// 11. No match! COMPILATION ERROR.
//     Need to add: std::string serialize(const std::string& s)

```

---

## Notes

- AI is **exceptionally good** at explaining existing TMP code — paste and ask "what does this do?"
- For **instantiation traces**, specify concrete types: "trace with T=std::vector<int>"
- Ask AI to **simplify TMP to C++20 concepts** — often results in much cleaner code
- AI can generate **static_assert test cases** for TMP: "verify these types pass/fail the trait"
- For Boost.Hana or Boost.Mp11, AI knows the idioms and can explain/generate code
- When AI-generated TMP doesn't compile, **paste the error back** and ask for a fix
- Use AI to **generate tutorial progressions**: "explain SFINAE in 5 steps, from basic to advanced"
