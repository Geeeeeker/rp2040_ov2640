cmake_minimum_required(VERSION 3.13)

# 设置 Pico SDK 自动获取，不使用本地子模块
set(PICO_SDK_FETCH_FROM_GIT ON)
set(PICO_SDK_FETCH_GIT_TAG "2.1.1")  # 指定稳定版本

# 从官方仓库获取 pico_sdk_import.cmake 并自动下载 SDK
include(FetchContent)
FetchContent_Declare(
    pico_sdk_import
    URL https://raw.githubusercontent.com/raspberrypi/pico-sdk/master/external/pico_sdk_import.cmake
    DOWNLOAD_NO_EXTRACT TRUE
)
FetchContent_Populate(pico_sdk_import)
include(${pico_sdk_import_SOURCE_DIR}/pico_sdk_import.cmake)

# 项目定义与 SDK 初始化
project(rp2040_ov2640 C CXX ASM)
pico_sdk_init()

# 创建可执行文件
add_executable(firmware
    ov2640.c
    main.c
)

# 生成 PIO 头文件
pico_generate_pio_header(firmware ${CMAKE_CURRENT_LIST_DIR}/image.pio)

# 链接库
target_link_libraries(firmware
    pico_stdlib
    hardware_dma
    hardware_i2c
    hardware_pio
    hardware_pwm
)

# 处理自定义引脚定义（从环境变量或 CMake 变量传入）
if(DEFINED ENV{CUSTOM_PIN_DEFINES})
    set(CUSTOM_PIN_DEFINES $ENV{CUSTOM_PIN_DEFINES})
endif()

if(NOT DEFINED CUSTOM_PIN_DEFINES)
    # 默认引脚映射（与 README 一致）
    set(CUSTOM_PIN_DEFINES
        PIN_RST=2
        PIN_XCLK=3
        PIN_SDA=4
        PIN_SCL=5
        PIN_D0=6
        PIN_D1=7
        PIN_D2=8
        PIN_D3=9
        PIN_D4=10
        PIN_D5=11
        PIN_D6=12
        PIN_D7=13
        PIN_PCLK=14
        PIN_HREF=15
        PIN_VSYNC=16
    )
endif()

foreach(def ${CUSTOM_PIN_DEFINES})
    string(REPLACE "=" ";" def_list ${def})
    list(GET def_list 0 key)
    list(GET def_list 1 value)
    target_compile_definitions(firmware PRIVATE ${key}=${value})
endforeach()

# 摄像头型号支持
if(DEFINED ENV{CAMERA_MODEL})
    set(CAMERA_MODEL $ENV{CAMERA_MODEL})
endif()
if(DEFINED CAMERA_MODEL)
    string(TOUPPER ${CAMERA_MODEL} CAMERA_MODEL_UPPER)
    target_compile_definitions(firmware PRIVATE CAMERA_MODEL_${CAMERA_MODEL_UPPER}=1)
else()
    target_compile_definitions(firmware PRIVATE CAMERA_MODEL_OV2640=1)
endif()

# 生成额外输出文件（.uf2）
pico_add_extra_outputs(firmware)

# 打印配置摘要
message(STATUS "PICO_SDK_PATH: ${PICO_SDK_PATH}")
message(STATUS "Camera model: ${CAMERA_MODEL}")
message(STATUS "Custom pin definitions: ${CUSTOM_PIN_DEFINES}")
