# Use the Clang AST dump to understand how the compiler sees your code

**Category:** Tooling & Debugging  
**Item:** #801  
**Reference:** <https://clang.llvm.org/docs/IntroductionToTheClangAST.html>  

---

## Topic Overview

The Clang AST (Abstract Syntax Tree) is the compiler's internal representation of your source code. Dumping it reveals how the compiler interprets expressions, resolves overloads, and applies implicit conversions.

```cpp

Source code:              AST:
int x = 1 + 2;           VarDecl 'x' 'int'
                            ImplicitCastExpr <IntegralCast>
                              BinaryOperator '+'
                                IntegerLiteral 1
                                IntegerLiteral 2

```

---

## Self-Assessment

### Q1: Run `-Xclang -ast-dump` and navigate the tree

```cpp

// ast_demo.cpp
#include <string>

struct Widget {
    int id;
    std::string name;

    Widget(int i, std::string n) : id(i), name(std::move(n)) {}
};

int main() {
    Widget w(42, "hello");
    auto copy = w;
    return copy.id;
}

```

```bash

# Dump the full AST:
$ clang++ -std=c++20 -Xclang -ast-dump -fsyntax-only ast_demo.cpp

# Key nodes in output:
# TranslationUnitDecl
# |-CXXRecordDecl 'Widget'
# | |-FieldDecl 'id' 'int'
# | |-FieldDecl 'name' 'std::string'
# | |-CXXConstructorDecl Widget 'void (int, std::string)'
# | | |-ParmVarDecl 'i' 'int'
# | | |-ParmVarDecl 'n' 'std::string'
# | | `-CompoundStmt
# | |   |-CXXCtorInitializer 'id'
# | |   |  `-ImplicitCastExpr
# | |   |    `-DeclRefExpr 'i'
# | |   `-CXXCtorInitializer 'name'
# | |     `-CallExpr 'std::move'
# | |       `-DeclRefExpr 'n'
# | |-CXXConstructorDecl implicit copy Widget
# | `-CXXConstructorDecl implicit move Widget
# |
# `-FunctionDecl 'main'
#   `-CompoundStmt
#     |-DeclStmt
#     | `-VarDecl 'w' 'Widget'
#     |   `-CXXConstructExpr 'Widget(int, std::string)'
#     |-DeclStmt
#     | `-VarDecl 'copy' 'Widget'  <-- 'auto' deduced to Widget
#     |   `-CXXConstructExpr 'Widget(const Widget&)'  <-- copy ctor
#     `-ReturnStmt
#       `-ImplicitCastExpr
#         `-MemberExpr '.id'
#           `-DeclRefExpr 'copy'

# Filter to specific function:
$ clang++ -std=c++20 -Xclang -ast-dump \
    -Xclang -ast-dump-filter=main -fsyntax-only ast_demo.cpp

# Colored output:
$ clang++ -std=c++20 -Xclang -ast-dump -fcolor-diagnostics \
    -fsyntax-only ast_demo.cpp

```

### Q2: Use AST to understand overload resolution

```cpp

// overload_demo.cpp
#include <iostream>
#include <string>

void process(int x)         { std::cout << "int: " << x << '\n'; }
void process(double x)      { std::cout << "double: " << x << '\n'; }
void process(std::string x) { std::cout << "string: " << x << '\n'; }

int main() {
    process(42);       // which overload?
    process(3.14);     // which overload?
    process(42.0f);    // which overload? (float -> double or int?)
    process("hello");  // which overload? (const char* -> string?)
}

```

```bash

$ clang++ -std=c++20 -Xclang -ast-dump -Xclang -ast-dump-filter=main \
    -fsyntax-only overload_demo.cpp

# AST reveals:
# CallExpr 'process'
#   `-IntegerLiteral 42          -> calls process(int)  [exact match]
#
# CallExpr 'process'
#   `-FloatingLiteral 3.14       -> calls process(double)  [exact match]
#
# CallExpr 'process'
#   `-ImplicitCastExpr <FloatingCast>   <-- float promoted to double
#     `-FloatingLiteral 42.0f    -> calls process(double)  [promotion]
#
# CallExpr 'process'
#   `-ImplicitCastExpr <UserDefinedConversion>  <-- implicit conversion
#     `-CXXConstructExpr 'std::string(const char*)'  <-- string ctor called
#       `-StringLiteral "hello"  -> calls process(string)  [user conversion]

# The AST shows EVERY implicit conversion and which overload was chosen

```

### Q3: AST matchers for clang-tidy custom checks

```cpp

// Anti-pattern to detect: calling .size() == 0 instead of .empty()
// Bad:  if (vec.size() == 0)
// Good: if (vec.empty())

```

```bash

# Use clang-query to test AST matchers interactively:
$ clang-query ast_demo.cpp -- -std=c++20

clang-query> match binaryOperator(
  hasOperatorName("=="),
  hasOperandExpression(
    cxxMemberCallExpr(callee(cxxMethodDecl(hasName("size"))))
  ),
  hasOperandExpression(
    integerLiteral(equals(0))
  )
)
# Match found! Points to: vec.size() == 0

```

```cpp

// Conceptual clang-tidy check (C++ code for the check itself):
// In a clang-tidy plugin:
void SizeComparedToZero::registerMatchers(ast_matchers::MatchFinder* Finder) {
    Finder->addMatcher(
        binaryOperator(
            hasOperatorName("=="),
            hasLHS(cxxMemberCallExpr(
                callee(cxxMethodDecl(hasName("size"))))
                .bind("size_call")),
            hasRHS(integerLiteral(equals(0)))
        ).bind("comparison"),
        this);
}

void SizeComparedToZero::check(const MatchFinder::MatchResult& Result) {
    const auto* Match = Result.Nodes.getNodeAs<BinaryOperator>("comparison");
    diag(Match->getBeginLoc(),
         "use .empty() instead of .size() == 0");
}

```

---

## Notes

- Use `clang-query` to interactively test AST matchers before writing clang-tidy checks.
- `-ast-dump-filter=functionName` limits output to specific declarations.
- C++ Insights (cppinsights.io) also shows implicit conversions in a readable format.
- The AST shows implicit copy/move constructors, conversions, and template instantiations.
- Use `-ast-dump=json` for machine-parseable output.
