cmake_minimum_required(VERSION 3.13)

# ============================================
# 1. 导入 Raspberry Pi Pico SDK
# ============================================
# 如果未定义 PICO_SDK_PATH，尝试从环境变量或默认相对路径查找
if(NOT DEFINED PICO_SDK_PATH)
    # 优先使用环境变量
    if(DEFINED ENV{PICO_SDK_PATH})
        set(PICO_SDK_PATH $ENV{PICO_SDK_PATH})
    else()
        # 否则假设 pico-sdk 子模块位于仓库根目录下的 pico-sdk 文件夹
        set(PICO_SDK_PATH ${CMAKE_CURRENT_LIST_DIR}/pico-sdk)
    endif()
endif()

# 导入 SDK 初始化脚本
include(${PICO_SDK_PATH}/external/pico_sdk_import.cmake)

# ============================================
# 2. 项目定义与 SDK 初始化
# ============================================
project(rp2040_ov2640 C CXX ASM)
pico_sdk_init()

# ============================================
# 3. 创建可执行文件
# ============================================
add_executable(firmware
    ov2640.c
    main.c
)

# 生成 PIO 汇编头文件（用于图像数据捕捉）
pico_generate_pio_header(firmware ${CMAKE_CURRENT_LIST_DIR}/image.pio)

# ============================================
# 4. 链接必要的硬件库
# ============================================
target_link_libraries(firmware
    pico_stdlib          # 标准库（包含 printf、gpio 等）
    hardware_dma         # DMA 支持
    hardware_i2c         # I2C 用于摄像头寄存器配置
    hardware_pio         # PIO 用于高效数据捕捉
    hardware_pwm         # PWM 用于 XCLK 时钟生成（可选）
)

# ============================================
# 5. 处理自定义引脚宏定义
# ============================================
# 可通过以下方式传入引脚配置：
#   - 命令行：cmake -DCUSTOM_PIN_DEFINES="PIN_SDA=4;PIN_SCL=5;..."
#   - 环境变量：CUSTOM_PIN_DEFINES（GitHub Actions 中使用）
#   - 如果都没有，则使用默认接线（与 README 一致）
if(DEFINED ENV{CUSTOM_PIN_DEFINES})
    set(CUSTOM_PIN_DEFINES $ENV{CUSTOM_PIN_DEFINES})
endif()

if(NOT DEFINED CUSTOM_PIN_DEFINES)
    # 默认引脚映射（基于 README 中的接线表）
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

# 将形如 "KEY1=VAL1;KEY2=VAL2" 的列表转换为编译器定义
foreach(def ${CUSTOM_PIN_DEFINES})
    # 分割 KEY=VALUE
    string(REPLACE "=" ";" def_list ${def})
    list(GET def_list 0 key)
    list(GET def_list 1 value)
    target_compile_definitions(firmware PRIVATE ${key}=${value})
endforeach()

# ============================================
# 6. 摄像头型号支持（可选扩展）
# ============================================
if(DEFINED ENV{CAMERA_MODEL})
    set(CAMERA_MODEL $ENV{CAMERA_MODEL})
endif()
if(DEFINED CAMERA_MODEL)
    string(TOUPPER ${CAMERA_MODEL} CAMERA_MODEL_UPPER)
    target_compile_definitions(firmware PRIVATE CAMERA_MODEL_${CAMERA_MODEL_UPPER}=1)
else()
    # 默认使用 OV2640
    target_compile_definitions(firmware PRIVATE CAMERA_MODEL_OV2640=1)
endif()

# ============================================
# 7. 生成额外输出文件（.uf2, .bin, .hex）
# ============================================
pico_add_extra_outputs(firmware)

# ============================================
# 8. 可选：启用 USB 输出（禁用 UART）
# ============================================
# 如果你希望 printf 通过 USB CDC 输出而不是物理 UART，取消下面的注释
# pico_enable_stdio_usb(firmware 1)
# pico_enable_stdio_uart(firmware 0)

# ============================================
# 9. 打印配置摘要（便于调试）
# ============================================
message(STATUS "PICO_SDK_PATH: ${PICO_SDK_PATH}")
message(STATUS "Camera model: ${CAMERA_MODEL}")
message(STATUS "Custom pin definitions: ${CUSTOM_PIN_DEFINES}")
