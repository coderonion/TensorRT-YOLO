# 设置项目
cmake_minimum_required(VERSION 3.12)
cmake_policy(SET CMP0091 NEW)
project(pose LANGUAGES CXX CUDA)

# 设置 C++ 标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 生成编译数据库，便于代码分析工具使用
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# 添加依赖项
find_package(OpenCV REQUIRED)
if(CMAKE_VERSION VERSION_GREATER_EQUAL 3.18)
    find_package(CUDAToolkit REQUIRED)
    set(CUDA_INCLUDE_DIRS ${CUDAToolkit_INCLUDE_DIRS})
    set(CUDA_LIBRARIES ${CUDAToolkit_LIBRARIES})
else()
    find_package(CUDA REQUIRED)
endif()
set(CMAKE_CUDA_ARCHITECTURES 89;86;75;70;61)  # CUDA 架构版本 8.9, 8.6, 7.5, 7.0, 6.1

# 添加编译选项
option(TENSORRT_PATH "TensorRT Path. Example: /usr/local/tensorrt" "")
if(NOT DEFINED TENSORRT_PATH)
    message(FATAL_ERROR "TensorRT path is not set. Please specify the TensorRT path.")
endif()

option(DEPLOY_PATH "TensorRT-YOLO Project Path." "")
if(NOT DEFINED DEPLOY_PATH)
    message(FATAL_ERROR "TensorRT-YOLO project path is not set. Please specify the TensorRT-YOLO Project path.")
endif()

# 添加可执行文件
add_executable(pose "")

# 包含头文件目录
target_include_directories(pose PRIVATE
    ${CUDA_INCLUDE_DIRS}
    ${TENSORRT_PATH}/include
    ${OpenCV_INCLUDE_DIRS}
    ${DEPLOY_PATH}
)

# 链接库目录
target_link_directories(pose PRIVATE
    ${TENSORRT_PATH}/lib
    ${DEPLOY_PATH}/lib
)

# 私有源文件
target_sources(pose PRIVATE pose.cpp)

# 私有编译定义
target_compile_definitions(pose PRIVATE ${CUDA_DEFINITIONS})

# 私有链接库
target_link_libraries(pose PRIVATE
    ${CUDA_cudart_LIBRARY}
    ${OpenCV_LIBS}
    deploy
)

# 根据平台选择不同的 TensorRT 库
if(MSVC AND EXISTS ${TENSORRT_PATH}/lib/nvinfer_10.dll)
    target_link_libraries(pose PRIVATE nvinfer_10 nvinfer_plugin_10 nvonnxparser_10)
else()
    target_link_libraries(pose PRIVATE nvinfer nvinfer_plugin nvonnxparser)
endif()

# 设置输出目录
set_target_properties(pose PROPERTIES OUTPUT_NAME "pose")
if(MSVC)
    set_target_properties(pose PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY_RELEASE "${PROJECT_SOURCE_DIR}/bin"
        RUNTIME_OUTPUT_DIRECTORY_DEBUG "${PROJECT_SOURCE_DIR}/bin"
    )
else()
    set_target_properties(pose PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY "${PROJECT_SOURCE_DIR}/bin"
    )
endif()

# 编译选项
if(MSVC)
    target_compile_options(pose PRIVATE
        $<$<CONFIG:Release>:-O2>
    )
    set_property(TARGET pose PROPERTY MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
else()
    target_compile_options(pose PRIVATE
        $<$<COMPILE_LANGUAGE:CXX>:-O3 -flto=auto>
    )
    target_link_options(pose PRIVATE
        $<$<COMPILE_LANGUAGE:CXX>:-O3 -flto=auto>
    )
endif()