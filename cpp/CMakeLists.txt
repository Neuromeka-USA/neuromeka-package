cmake_minimum_required(VERSION 3.15)
project(IndyDCP3)

# C++ standard
set(CMAKE_C_STANDARD 17)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_C_STANDARD_REQUIRED True)
set(CMAKE_CXX_STANDARD_REQUIRED True)

# Build option
option(BUILD_PROTO "Build proto task" OFF)
option(USE_FETCH_CONTENT_GRPC "Use FetchContent for gRPC" ON)

if(USE_FETCH_CONTENT_GRPC)
    message(STATUS "Using FetchContent for gRPC v1.76.0")
    include(FetchContent)

    # Disable install to prevent permission denied errors
    set(gRPC_INSTALL OFF CACHE BOOL "" FORCE)
    set(protobuf_INSTALL OFF CACHE BOOL "" FORCE)
    set(gRPC_BUILD_TESTS OFF CACHE BOOL "" FORCE)
    set(ABSL_PROPAGATE_CXX_STD ON)

    FetchContent_Declare(
      gRPC
      GIT_REPOSITORY https://github.com/grpc/grpc
      GIT_TAG        v1.76.0
    )

    FetchContent_MakeAvailable(gRPC)

    # Non-namespaced targets are often created by FetchContent or we can use aliases
    # But usually FetchContent of gRPC makes targets like grpc++, libprotobuf available directly or via gRPC:: alias depending on version.
    # We will set our internal aliases.
    set(_PROTOBUF_LIBPROTOBUF libprotobuf)
    set(_REFLECTION grpc++_reflection)
    set(_GRPC_GRPCPP grpc++)
    
    # Abseil and utf8_range specific handling if needed. 
    # FetchContent of gRPC pulls in abseil-cpp.
    # We will assume standard absl targets are available.
    set(_ABSL_LOG absl::log)
    set(_UTF8_RANGE utf8_range::utf8_range)

else()
    if(WIN32)
        message(STATUS "Configuring for Windows using find_package")
        # Manual link cmake paths - keep user's original paths if they exist or revert to general find_package
        # Assuming user environment is set up.
        set(absl_DIR "E:/Example/install/lib/cmake/absl")
        set(utf8_range_DIR "E:/Example/install/lib/cmake/utf8_range")
        set(protobuf_DIR "E:/Example/install/cmake")
        set(gRPC_DIR "E:/Example/install/lib/cmake/grpc")
    else()
        message(STATUS "Configuring for Linux using find_package")
    endif()

    find_package(Protobuf CONFIG REQUIRED)
    message(STATUS "Using protobuf ${Protobuf_VERSION}")

    find_package(gRPC CONFIG REQUIRED)
    message(STATUS "Using gRPC ${gRPC_VERSION}")
    
    # Also need abseil if we are linking against it explicitly
    find_package(absl CONFIG REQUIRED)
    find_package(utf8_range CONFIG)

    # Namespaced targets from config files
    set(_PROTOBUF_LIBPROTOBUF protobuf::libprotobuf)
    set(_REFLECTION gRPC::grpc++_reflection)
    set(_GRPC_GRPCPP gRPC::grpc++)
    
    set(_ABSL_LOG absl::log)
    if(TARGET utf8_range::utf8_range)
        set(_UTF8_RANGE utf8_range::utf8_range)
    else()
        set(_UTF8_RANGE "") # Handle case where it might be missing or included differently
    endif()
endif()

# Include directories for source files
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/neuromeka_cpp)
# We will output generated protos to build directory or source directory. 
# Using CMAKE_CURRENT_BINARY_DIR is cleaner for generated files.
include_directories(${CMAKE_CURRENT_BINARY_DIR})
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/neuromeka_cpp/proto) # For existing headers if mixed

if(BUILD_PROTO)
    add_executable(build_proto src/build_proto.cpp)
    target_link_libraries(build_proto ${Protobuf_LIBRARIES} ${gRPC_LIBRARIES})

else()
    # Find all proto files in RECONSTRUCTED folder as source truth
    file(GLOB PROTO_FILES "${CMAKE_CURRENT_SOURCE_DIR}/neuromeka_cpp/reconstructed_protos/*.proto")

    # Generate sources
    # protobuf_generate_cpp is verified to be available when using protobuf config or fetchcontent
    protobuf_generate_cpp(PROTO_SRCS PROTO_HDRS ${PROTO_FILES})

    set(INDY_SOURCES
        neuromeka_cpp/indydcp3.cpp
        ${PROTO_SRCS}
    )

    add_executable(example_indydcp3 src/example_indydcp3.cpp ${INDY_SOURCES})
    target_link_libraries(example_indydcp3 
                        ${_REFLECTION}
                        ${_GRPC_GRPCPP}
                        ${_PROTOBUF_LIBPROTOBUF}
                        ${_ABSL_LOG}
                        ${_UTF8_RANGE}
                        )
endif()
