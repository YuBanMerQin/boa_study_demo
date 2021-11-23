cmake_minimum_required(VERSION 3.6)
project(src C)

set(CMAKE_C_STANDARD 11)
set(EXECUTABLE_OUTPUT_PATH )

include_directories(include)
aux_source_directory(source SOURCES)

add_executable(src ${SOURCES})
