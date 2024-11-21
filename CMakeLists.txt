cmake_minimum_required(VERSION 3.13)

project(std_coroutines)

option(COROUTINES_FIND_EXPERIMENTAL "find <experimental/coroutine>" ON)

set(CMAKE_CXX_STANDARD 20)

include(GNUInstallDirs)
include(CMakePushCheckState)
include(CheckIncludeFileCXX)
include(CheckCXXSourceCompiles)

cmake_push_check_state()

# figure out correct compiler flags
if(("x${CMAKE_CXX_COMPILER_ID}" MATCHES "x.*Clang"
    AND "x${CMAKE_CXX_SIMULATE_ID}" STREQUAL "xMSVC")
   OR "x${CMAKE_CXX_COMPILER_ID}" STREQUAL "xMSVC")
  set(_CXX_COROUTINES_AWAIT "/await")
elseif("x${CMAKE_CXX_COMPILER_ID}" STREQUAL "xGNU")
  set(_CXX_COROUTINES_AWAIT "-fcoroutines -std=c++20")
elseif("x${CMAKE_CXX_COMPILER_ID}" STREQUAL "xClang"
       OR "x${CMAKE_CXX_COMPILER_ID}" STREQUAL "xAppleClang")
  # set(_CXX_COROUTINES_AWAIT "-stdlib=libc++") set(_CXX_COROUTINES_AWAIT
  # "${_CXX_COROUTINES_AWAIT} -fcoroutines-ts")
else()
  message(FATAL_ERROR "Unknown compiler: ${CMAKE_CXX_COMPILER_ID}")
endif()

# Detect which of Experimental and Final we should look for
set(find_final TRUE)
set(find_experimental ${COROUTINES_FIND_EXPERIMENTAL})

if(find_final)
  set(CMAKE_REQUIRED_FLAGS "${_CXX_COROUTINES_AWAIT}")
  check_include_file_cxx("coroutine" _CXX_COROUTINES_HAVE_HEADER)
  mark_as_advanced(_CXX_COROUTINES_HAVE_HEADER)
  if(_CXX_COROUTINES_HAVE_HEADER)
    # We found the non-experimental header. Don't bother looking for the
    # experimental one.
    set(find_experimental FALSE)
  endif()
else()
  set(_CXX_COROUTINES_HAVE_HEADER FALSE)
endif()

if(find_experimental)
  set(CMAKE_REQUIRED_FLAGS "${_CXX_COROUTINES_AWAIT}")
  check_include_file_cxx("experimental/coroutine"
                         _CXX_COROUTINES_HAVE_EXPERIMENTAL_HEADER)
  mark_as_advanced(_CXX_COROUTINES_HAVE_EXPERIMENTAL_HEADER)
else()
  set(_CXX_COROUTINES_HAVE_EXPERIMENTAL_HEADER FALSE)
endif()

if(_CXX_COROUTINES_HAVE_HEADER)
  set(_have_coro TRUE)
  set(_coro_header coroutine)
  set(_coro_namespace std)
elseif(_CXX_COROUTINES_HAVE_EXPERIMENTAL_HEADER)
  set(_have_coro TRUE)
  set(_coro_header experimental/coroutine)
  set(_coro_namespace std::experimental)
else()
  set(_have_coro FALSE)
endif()

set(CXX_COROUTINES_HAVE_COROUTINES
    ${_have_coro}
    CACHE BOOL "TRUE if we have the C++ coroutines feature")
set(CXX_COROUTINES_HEADER
    ${_coro_header}
    CACHE STRING
          "The header that should be included to obtain the coroutines APIs")
set(CXX_COROUTINES_NAMESPACE
    ${_coro_namespace}
    CACHE STRING "The C++ namespace that contains the coroutines APIs")

set(_found FALSE)

if(NOT CXX_COROUTINES_HAVE_COROUTINES)
  message(FATAL_ERROR "Cloud not find coroutine header")
endif()

add_library(${PROJECT_NAME} INTERFACE)
target_compile_features(${PROJECT_NAME} INTERFACE cxx_std_20)

add_library(std::coroutines ALIAS ${PROJECT_NAME})

# We have some coroutines library available. Do link checks
string(
  CONFIGURE
    [[
    #include <utility>
    #include <@CXX_COROUTINES_HEADER@>

    struct present {
        struct promise_type {
        int result;
        present get_return_object() { return present{*this}; }
        @CXX_COROUTINES_NAMESPACE@::suspend_never initial_suspend() { return {}; }
        @CXX_COROUTINES_NAMESPACE@::suspend_always final_suspend() noexcept { return {}; }
        void return_value(int i) { result = i; }
        void unhandled_exception() {}
        };
        friend struct promise_type;
        present(present&& that) : coro_(std::exchange(that.coro_, {})) {}
        ~present() { if(coro_) coro_.destroy(); }
        bool await_ready() const { return true; }
        void await_suspend(@CXX_COROUTINES_NAMESPACE@::coroutine_handle<>) const {}
        int await_resume() const { return coro_.promise().result; }
    private:
        present(promise_type& promise)
        : coro_(@CXX_COROUTINES_NAMESPACE@::coroutine_handle<promise_type>::from_promise(promise)) {}
        @CXX_COROUTINES_NAMESPACE@::coroutine_handle<promise_type> coro_;
    };

    present f(int n) {
        if (n < 2)
        co_return 1;
        else
        co_return n * co_await f(n - 1);
    }

    int main() {
        return f(5).await_resume() != 120;
    }
]]
    code
  @ONLY)

# Try to compile a simple coroutines program without any compiler flags
set(CMAKE_REQUIRED_FLAGS)
check_cxx_source_compiles("${code}" CXX_COROUTINES_NO_AWAIT_NEEDED)

if(NOT CXX_COROUTINES_NO_AWAIT_NEEDED)
  # Add the -fcoroutines-ts (or /await) flag
  set(CMAKE_REQUIRED_FLAGS "${_CXX_COROUTINES_AWAIT}")
  check_cxx_source_compiles("${code}" CXX_COROUTINES_AWAIT_NEEDED)
  target_compile_options(${PROJECT_NAME} INTERFACE ${_CXX_COROUTINES_AWAIT})

  if(NOT CXX_COROUTINES_AWAIT_NEEDED)
    message(
      FATAL_ERROR
        "Cloud not link coroutine with flags(${_CXX_COROUTINES_AWAIT})")
  endif()
endif()

cmake_pop_check_state()

# Install
install(TARGETS ${PROJECT_NAME} EXPORT ${PROJECT_NAME}Targets)
install(
  EXPORT ${PROJECT_NAME}Targets
  FILE ${PROJECT_NAME}Config.cmake
  DESTINATION ${CMAKE_INSTALL_FULL_DATADIR}/${PROJECT_NAME})
