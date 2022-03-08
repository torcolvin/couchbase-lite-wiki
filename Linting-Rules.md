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


## Clang-tidy Rules
- modernize-deprecated-headers
    - Use C++ style C header includes (e.g. `<cassert>` not `<assert.h>`)
- modernize-use-nodiscard
    - Use `[[nodiscard]]` modifier for functions when applicable
