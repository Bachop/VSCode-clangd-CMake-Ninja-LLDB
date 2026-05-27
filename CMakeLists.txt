cmake_minimum_required(VERSION 3.10)
project(TestProject CXX)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(SOURCES
    main.cpp
)

add_executable(room2_exe ${SOURCES})