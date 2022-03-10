Here is an attempt to list all the rules we will follow when using a linter to clean up the code style and best practices.  It is based on the Resharper tool from JetBrains, which borrows heavily from the clang-tidy rules:

## Resharper Rules
- CppMemberFunctionMayBeConst
    - Member functions that can be marked `const` should be
- CppEnforceNestedNamespacesStyle
    - Use the modern namespace style of `litecore::actor {` vs `litecore { actor {`
- CppUnusedIncludeDirective
    - Remove header includes that are not used (or guard them if they are used only on certain platforms)
- CppVariableCanBeMadeConstexpr
    - Use `constexpr` for constant values whenever possible
- CppIfCanBeReplacedByConstexprIf
    - Use `if constexpr(x)` instead of `if(x)` on `constexpr` values to strip unused code
- CppUseStructuredBinding
    - Use C++17 structured binding for clearer variable names
    - e.g. `for (std::pair<int, int> : c) { printf("%d / %d", c.first, c.second); }`
    -     => `for(auto& [err, detail] : c) { printf("%d / %d", err, detail); }`
- CppDeclaratorNeverUsed
    - Remove unused var

## Clang-tidy Rules
- modernize-deprecated-headers
    - Use C++ style C header includes (e.g. `<cassert>` not `<assert.h>`)
- modernize-use-nodiscard
    - Use `[[nodiscard]]` modifier for functions when applicable
- performance-unnecessary-value-param
    - Use a const reference instead of pass by value in methods
- clang-diagnostic-nullable-to-nonnull-conversion
    - Don't pass a nullable pointer to a method taking nonnull arguments
- modernize-loop-convert
    - Use the newer syntax `for(auto& x : y)`
- clang-diagnostic-reorder-ctor
    - When initializing in a constructor, preserve the declarative order of the class variables
- clang-diagnostic-gnu-zero-variadic-macro-arguments
    - macros declared with variadic args (i.e. X(y, ...)) must have at least one variadic argument
- modernize-pass-by-value
    - Pass by value and use `std::move` where possible
- bugprone-implicit-widening-of-multiplication-result
    - Multiplication on type int widened to larger type
- cppcoreguidelines-pro-type-member-init
    - Initialize all variables in every constructor
- modernize-avoid-bind
    - Prefer lambda to `std::bind`
- bugprone-reserved-identifier
    - Don't use reserved identifiers as macro names