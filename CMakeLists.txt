cmake_minimum_required(VERSION 3.10)
project(HDR_PLUS)

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

SET(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR})
include("macro.cmake")
set(CMAKE_CXX_STANDARD 11)

# Define dependencies
link_libtiff()
find_library(LIBRAW_LIBRARY NAMES raw raw_r)
find_package(ZLIB REQUIRED)
find_package(Halide REQUIRED)

# Files 
set(src_files
    src/InputSource.cpp
    src/Burst.cpp
    src/LibRaw2DngConverter.cpp)

include_directories(${CMAKE_CURRENT_SOURCE_DIR} 
                    ${HALIDE_DISTRIB_DIR}/include 
                    ${HALIDE_DISTRIB_DIR}/tools 
                    ${RAW2DNG_INCLUDE_DIRS})

include_directories(${CMAKE_BINARY_DIR}/genfiles)

# --- Generators --- #
# Generator: hdrplus_pipeline
add_executable(hdrplus_pipeline.generator src/hdrplus_pipeline_generator.cpp 
                                         src/align.cpp 
                                         src/merge.cpp 
                                         src/finish.cpp 
                                         src/util.cpp)
target_link_libraries(hdrplus_pipeline.generator PRIVATE Halide::Generator)
target_include_directories(hdrplus_pipeline.generator PRIVATE "include/")
add_halide_library(hdrplus_pipeline FROM hdrplus_pipeline.generator 
                    AUTOSCHEDULER ${SCHEDULER_TYPE}
                    PARAMS auto_schedule=false)

# Generator: align_and_merge
add_executable(align_and_merge.generator src/align_and_merge_generator.cpp 
                                         src/align.cpp 
                                         src/merge.cpp 
                                         src/util.cpp)
target_link_libraries(align_and_merge.generator PRIVATE Halide::Generator)
target_include_directories(align_and_merge.generator PRIVATE "include/")
add_halide_library(align_and_merge FROM align_and_merge.generator 
                    AUTOSCHEDULER ${SCHEDULER_TYPE}
                    PARAMS auto_schedule=false)


# --- Executables --- #
add_executable(hdrplus bin/HDRPlus.cpp ${src_files})
add_dependencies(hdrplus hdrplus_pipeline)
target_link_libraries(hdrplus 
                      hdrplus_pipeline 
                      Halide::ImageIO
                      Halide::Halide
                      ${LIBRAW_LIBRARY} 
                      ${TIFF_LIBRARIES} 
                      ${TIFFXX_LIBRARY}
                      )

add_executable(stack_frames bin/stack_frames.cpp ${src_files})
add_dependencies(stack_frames align_and_merge)
target_include_directories(stack_frames PRIVATE src/ include/)
target_link_libraries(stack_frames PRIVATE align_and_merge 
                                           Halide::ImageIO
                                           Halide::Halide
                                           ${LIBRAW_LIBRARY} 
                                           ${TIFF_LIBRARIES} 
                                           ${TIFFXX_LIBRARY}
                                        )
