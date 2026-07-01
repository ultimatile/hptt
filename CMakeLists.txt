
cmake_minimum_required(VERSION 3.10 FATAL_ERROR)
project (HPTT C CXX)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(ENABLE_IBM OFF)

if(CMAKE_SYSTEM_PROCESSOR STREQUAL "ppc64le")
  set(ENABLE_IBM ON)
endif()

if(CMAKE_CXX_COMPILER_ID STREQUAL "Intel")
  set(HPTT_CXX_FLAGS ${HPTT_CXX_FLAGS} -qopenmp -xhost)
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  if(ENABLE_IBM)
    set(HPTT_CXX_FLAGS ${HPTT_CXX_FLAGS} -fopenmp)
  else()
    set(HPTT_CXX_FLAGS ${HPTT_CXX_FLAGS} -fopenmp -march=native -mtune=native)
  endif()
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
  set(HPTT_CXX_FLAGS ${HPTT_CXX_FLAGS} -fopenmp -march=native)
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang")
  # AppleClang (macOS's /usr/bin/c++) reports a compiler id distinct from
  # "Clang", so the generic Clang branch never fires here. It also rejects a
  # bare -fopenmp, and CMake's FindOpenMP cannot locate Homebrew's keg-only
  # libomp unaided -- left alone, HPTT builds single-threaded even when libomp
  # is installed. Discover libomp and seed the FindOpenMP hint variables so the
  # imported OpenMP::OpenMP_CXX target resolves. OpenMP stays optional (every
  # omp use in the sources is guarded by _OPENMP): when libomp is absent, fall
  # back to a single-threaded build rather than failing configure.
  if(NOT LIBOMP_PREFIX)
    find_program(BREW_EXECUTABLE brew)
    if(BREW_EXECUTABLE)
      execute_process(COMMAND ${BREW_EXECUTABLE} --prefix libomp
                      OUTPUT_VARIABLE LIBOMP_PREFIX
                      OUTPUT_STRIP_TRAILING_WHITESPACE
                      ERROR_QUIET)
    endif()
  endif()
  foreach(_omp_candidate "${LIBOMP_PREFIX}" /opt/homebrew/opt/libomp /usr/local/opt/libomp)
    if(_omp_candidate AND EXISTS "${_omp_candidate}/include/omp.h")
      set(LIBOMP_PREFIX "${_omp_candidate}")
      break()
    endif()
  endforeach()
  if(LIBOMP_PREFIX AND EXISTS "${LIBOMP_PREFIX}/include/omp.h")
    set(OpenMP_CXX_FLAGS "-Xclang -fopenmp -I${LIBOMP_PREFIX}/include")
    set(OpenMP_CXX_LIB_NAMES "omp")
    set(OpenMP_omp_LIBRARY "${LIBOMP_PREFIX}/lib/libomp.dylib")
    # COMPONENTS CXX (needs CMake >= 3.10): the project also enables C, but no C
    # source uses OpenMP, so probing OpenMP_C would fail on the unseeded C hints.
    find_package(OpenMP COMPONENTS CXX)
  endif()
  if(NOT OpenMP_CXX_FOUND)
    message(WARNING
      "OpenMP (libomp) not found for AppleClang; building HPTT single-threaded. "
      "Install libomp (e.g. `brew install libomp`) or pass "
      "-DLIBOMP_PREFIX=<path-to-libomp> to enable multithreading.")
  endif()
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "PGI")
  set(HPTT_CXX_FLAGS ${HPTT_CXX_FLAGS} -silent -w -Mnovect)
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "XL")
  set(HPTT_CXX_FLAGS ${HPTT_CXX_FLAGS} -qsmp=omp)
endif()

if(ENABLE_AVX)
  set(HPTT_CXX_FLAGS ${HPTT_CXX_FLAGS} -mavx -DHPTT_ARCH_AVX)
elseif(ENABLE_ARM)
  set(HPTT_CXX_FLAGS ${HPTT_CXX_FLAGS} -mfpu=neon -DHPTT_ARCH_ARM)
elseif(ENABLE_IBM)
  set(HPTT_CXX_FLAGS ${HPTT_CXX_FLAGS} -mtune=native -DHPTT_ARCH_IBM -maltivec -mabi=altivec)
endif()

set(HPTT_SRCS src/hptt.cpp src/plan.cpp src/transpose.cpp src/utils.cpp)

add_library(hptt STATIC ${HPTT_SRCS})
target_compile_features(hptt PUBLIC cxx_std_11)
target_include_directories(hptt PUBLIC ${PROJECT_SOURCE_DIR}/include)
#target_compile_definitions(hptt PRIVATE ${HPTT_CXX_COMPILE_DEFS})
target_compile_options(hptt PUBLIC ${HPTT_CXX_FLAGS})

# On AppleClang the OpenMP compile flags and libomp link both travel through this
# imported target (see the AppleClang branch above). Other compilers carry
# -fopenmp inline via HPTT_CXX_FLAGS and never create this target, so the guard
# keeps their builds byte-identical.
if(TARGET OpenMP::OpenMP_CXX)
  target_link_libraries(hptt PUBLIC OpenMP::OpenMP_CXX)
endif()

install(TARGETS hptt
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib)

set(HPTT_INCLUDES
    include/compute_node.h
    include/hptt_types.h
    include/hptt.h
    include/macros.h
    include/plan.h
    include/utils.h
    include/transpose.h)

install(FILES ${HPTT_INCLUDES}
        DESTINATION include)
