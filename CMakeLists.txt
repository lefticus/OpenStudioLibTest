cmake_minimum_required(VERSION 3.8)

# Link this 'library' to use the following warnings
add_library(project_warnings INTERFACE)

set(OPENSTUDIO_PATH CACHE PATH "Path to built/install OpenStudio lib")
find_package(openstudio HINTS ${OPENSTUDIO_PATH})



##
## CONAN
##

set(CMAKE_CONAN_EXPECTED_HASH 709180234748692a642f9e5c4d80d328)
set(CMAKE_CONAN_VERSION "v0.14")

if(NOT EXISTS "${CMAKE_BINARY_DIR}/conan.cmake" OR NOT "${CMAKE_CONAN_HASH}" MATCHES "${CMAKE_CONAN_EXPECTED_HASH}")
  # Put it in CMAKE_BINARY_DIR so we don't end up with two when building OpenStudioApplication
  message(STATUS "openstudio: Downloading conan.cmake ${CMAKE_CONAN_VERSION} from https://github.com/conan-io/cmake-conan")
  file(DOWNLOAD "https://github.com/conan-io/cmake-conan/raw/${CMAKE_CONAN_VERSION}/conan.cmake"
    "${CMAKE_BINARY_DIR}/conan.cmake" EXPECTED_MD5 ${CMAKE_CONAN_EXPECTED_HASH})
endif()

include(${CMAKE_BINARY_DIR}/conan.cmake)

conan_check(VERSION 1.0.0 REQUIRED)

message(STATUS "openstudio: RUNNING CONAN")

conan_add_remote(NAME bincrafters
  URL https://api.bintray.com/conan/bincrafters/public-conan)

# Convenience variable to set a consistent version for individual boost packages
set(BOOST_VERSION "1.69.0")

list(APPEND CONAN_OPTIONS "zlib:minizip=True")
# You do want to rebuild packages if there's a newer recipe in the remote (which applies mostly to our own openstudio_ruby where we don't
# bump the actual package version when we make changes) than the binaries were built with
# 'outdated' also acts like 'missing': if no binary, will build them.
list(APPEND CONAN_BUILD "outdated")

# DLM: add option for shared libs if we are building shared?

# This will create the conanbuildinfo.cmake in the current binary dir, not the cmake_binary_dir
conan_cmake_run(REQUIRES
  OpenSSL/1.1.0g@conan/stable
  # Track NREL/stable in general, on a feature branch this could be temporarily switched to NREL/testing
  boost_asio/${BOOST_VERSION}@bincrafters/stable
  boost_program_options/${BOOST_VERSION}@bincrafters/stable
  boost_regex/${BOOST_VERSION}@bincrafters/stable
  boost_filesystem/${BOOST_VERSION}@bincrafters/stable
  boost_crc/${BOOST_VERSION}@bincrafters/stable
  boost_algorithm/${BOOST_VERSION}@bincrafters/stable
  boost_uuid/${BOOST_VERSION}@bincrafters/stable
  boost_log/${BOOST_VERSION}@bincrafters/stable
  boost_numeric_ublas/${BOOST_VERSION}@bincrafters/stable
  boost_functional/${BOOST_VERSION}@bincrafters/stable
  boost_geometry/${BOOST_VERSION}@bincrafters/stable
  pugixml/1.9@bincrafters/stable
  jsoncpp/1.8.4@theirix/stable
  cpprestsdk/2.10.13@bincrafters/stable
  BASIC_SETUP CMAKE_TARGETS NO_OUTPUT_DIRS
  OPTIONS ${CONAN_OPTIONS}
  BUILD ${CONAN_BUILD}
  # Passes `-u, --update`    to conan install: Check updates exist from upstream remotes
  # That and build=outdated should ensure we track the right
  UPDATE
)

##
## END CONAN
##



if(CMAKE_COMPILER_IS_GNUCC)
  option(ENABLE_COVERAGE "Enable coverage reporting for gcc/clang" FALSE)

  if(ENABLE_COVERAGE)
    add_compile_options(--coverage -O0)
  endif()
endif()

if(MSVC)
  target_compile_options(project_warnings INTERFACE /W4)
else()
  target_compile_options(project_warnings
                         INTERFACE
                         -Wall
                         -Wextra # reasonable and standard
                         -Wshadow # warn the user if a variable declaration shadows one from a
                                  # parent context
                         -Wnon-virtual-dtor # warn the user if a class with virtual functions has a
                                            # non-virtual destructor. This helps catch hard to
                                            # track down memory errors
                         -Wold-style-cast # warn for c-style casts
                         -Wcast-align # warn for potential performance problem casts
                         -Wunused # warn on anything being unused
                         -Woverloaded-virtual # warn if you overload (not override) a virtual
                                              # function
                         -Wpedantic # warn if non-standard C++ is used
                         -Wconversion # warn on type conversions that may lose data
                         -Wsign-conversion # warn on sign conversions
                         -Wmisleading-indentation # warn if identation implies blocks where blocks
                                                  # do not exist
                         -Wduplicated-cond # warn if if / else chain has duplicated conditions
                         -Wduplicated-branches # warn if if / else branches have duplicated code
                         -Wlogical-op # warn about logical operations being used where bitwise were
                                      # probably wanted
                         -Wnull-dereference # warn if a null dereference is detected
                         -Wuseless-cast # warn if you perform a cast to the same type
                         -Wdouble-promotion # warn if float is implicit promoted to double
                         -Wformat=2 # warn on security issues around functions that format output
                                    # (ie printf)
                         )
endif()

add_executable(OpenStudioLibTest main.cpp)
target_compile_features(OpenStudioLibTest PRIVATE cxx_std_17)
target_link_libraries(OpenStudioLibTest PRIVATE project_warnings openstudio::openstudiolib)

enable_testing()
add_test(openstudiolibtest OpenStudioLibTest)


