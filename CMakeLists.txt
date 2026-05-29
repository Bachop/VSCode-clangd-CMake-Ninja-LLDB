cmake_minimum_required(VERSION 3.10)
project(room2 LANGUAGES C CXX)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(SOURCES
    ./src/main.cpp
)

add_executable(room2 ${SOURCES})

target_include_directories(room2 PRIVATE include)

