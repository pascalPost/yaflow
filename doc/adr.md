# Architecture decision record (ADR)

## Realization of runtime selection of single and double precision

### Issue

Yaflow should allow for single or double precision selection on runtime for user
convenience. The alternative and easy way would be compile time selection with
recompilation when wanting to switch. For realization there are basically two
alternatives:
- template with runtime dispatch
- separate compilation units with different dependent builds and runtime
  dispatch to the respective compilation unit

### Template based solution

Example of a plugin system to allow for runtime dispatch.

#### plugin.hpp
``` cpp
// plugin system interface
template<typename FloatT>
struct plugin {
  virtual ~plugin() = default;

  // to be overloaded by plugins
  virtual void run(const FloatT) const;
};
```

#### plugin.cpp
``` cpp
// helper needed to give compile time error for constexpr if else statement
template <bool flag = false> void static_no_match() {
  static_assert(flag, "no match");
}

// example implementation (more realistically this woulde be a derived class)
template <typename FloatT>
void plugin<FloatT>::run(const FloatT) const {
  if constexpr (std::is_same_v<FloatT, float>) {
    std::cout << "SP" << std::endl;
  } else if constexpr (std::is_same_v<FloatT, double>) {
    std::cout << "DP" << std::endl;
  } else {
    static_no_match();
  }
}

// SP and DP compilations in this compilation unit
template struct plugin<float>;
template struct plugin<double>;
```

#### main.cpp
``` cpp
int main () {

  // simple runtime dispatch based on construction of the respective template instanstiation

  const auto test_sp = plugin<float>();
  test_sp.run();

  const auto test_dp = plugin<double>();
  test_dp.run();
}
```

### Simple Compilation based solution

#### definitions.hpp
``` cpp
// floating point type
#if defined __DOUBLE_PRECISION__
using FloatT = double;
#elif defined __SINGLE_PRECISION__
using FloatT = float;
#else
#error SINGLE OR DOUBLE PRECISION NEEDS TO BE SPECIFIED
#endif
```

#### plugin.hpp
``` cpp
/// plugin system interface
struct plugin {
  virtual ~plugin() = default;
  virtual void run(const FloatT) const;
};
```

#### plugin.cpp
``` cpp
// helper needed to give compile time error for constexpr if else statement
template <bool flag = false> void static_no_match() {
  static_assert(flag, "no match");
}

// example implementation (more realistically this woulde be a derived class)
void plugin::run(const FloatT) const {
  if constexpr (std::is_same_v<FloatT, float>) {
    std::cout << "SP" << std::endl;
  } else if constexpr (std::is_same_v<FloatT, double>) {
    std::cout << "DP" << std::endl;
  } else {
    static_no_match();
  }
}
```

#### CMakeLists.txt

- Separate compilations into separate libs. This could also be just object files
  as described here:
  https://gitlab.kitware.com/cmake/community/-/wikis/doc/tutorials/Object-Library
- Options to turn off specific compilations may be added.

``` cmake
# single precision compilation
add_library(single_precision plugin.cpp)
target_compile_definitions(single_precision PUBLIC __SINGLE_PRECISION__)

# double precision compilation
add_library(double_precision plugin.cpp)
target_compile_definitions(double_precision PUBLIC __DOUBLE_PRECISION__)

# main yaflow executable
add_executable(test main.cpp)

# link single precision and double precision library compilations
target_link_libraries(yaflow yaflow_single_precision)
target_link_libraries(yaflow yaflow_double_precision)
```

#### main.cpp
``` cpp
struct plugin {
  virtual void run(const float);
  virtual void run(const double);
}

int main() {
  const auto test = plugin();

  test.run(float{});
  test.run(double{});
}
```

### Decision

For now, I will realize the second option based on compilation, as it reduces
the code clutter with templates and might make it more easy for contributors.
The same technique will also be used for runtime selectable computation backends
like CPU, CPU SSE (or other vector instruction sets) and GPU. 