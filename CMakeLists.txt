cmake_minimum_required(VERSION 3.20)

# CORE PROJECT SETUP AND TOOLCHAIN DEFINITION

project("power_electronics_comm_validation" C CXX ASM)
enable_language(C CXX ASM)
message("Build type: " ${CMAKE_BUILD_TYPE})

# Define System and Compiler for Cross-Compilation (ARM-NONE-EABI)
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)

# Set the cross-compiler prefix (assuming 'arm-none-eabi-' is in your PATH)
set(TOOLCHAIN_PREFIX arm-none-eabi-)

# Set the C, CXX, and ASM compilers
set(CMAKE_C_COMPILER ${TOOLCHAIN_PREFIX}gcc)
set(CMAKE_CXX_COMPILER ${TOOLCHAIN_PREFIX}g++)
set(CMAKE_ASM_COMPILER ${TOOLCHAIN_PREFIX}gcc)

# Prevent CMake from running tests on the cross-compiler
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)


# RENODE TOGGLE OPTION

option(RENODE_SIMULATION "Build for Renode simulation (uses ELF/GDB)" ON)

if(RENODE_SIMULATION)
    message(STATUS "RENODE_SIMULATION is ON. Output will be an ELF file suitable for GDB/Renode.")
    set(LINKER_SPECS "--specs=rdimon.specs") # Use rdimon.specs for Renode semihosting/debugging
else()
    message(STATUS "RENODE_SIMULATION is OFF. Output is suitable for actual hardware flashing.")
    set(LINKER_SPECS "--specs=nosys.specs") # Use nosys.specs for bare-metal hardware
endif()


# MCU-SPECIFIC DEFINITIONS (STM32F401RE)

# Core MCU flags: Cortex-M4, Thumb, Hard-Float FPU
set(cpu_PARAMS 
    -mthumb
    -mcpu=cortex-m4
    -mfloat-abi=hard
    -mfpu=fpv4-sp-d16
)

# Project Symbols (Macros)
set(symbols_SYMB
    STM32F401xE       # Defines the specific chip variant for headers
    USE_HAL_DRIVER    # Enables HAL library support
    HSE_VALUE=8000000 # Default external clock (Nucleo board crystal)
)

# Linker script (ADJUST PATH)
set(linker_script_SRC 
    ${CMAKE_CURRENT_SOURCE_DIR}/STM32F401RETx_FLASH.ld 
)

# Include directories (ADJUST PATHS)
set(include_DIRS
    ${CMAKE_CURRENT_SOURCE_DIR}/Inc
    ${CMAKE_CURRENT_SOURCE_DIR}/Drivers/CMSIS/Include
    ${CMAKE_CURRENT_SOURCE_DIR}/Drivers/CMSIS/Device/ST/STM32F4xx/Include
    ${CMAKE_CURRENT_SOURCE_DIR}/Drivers/STM32F4xx_HAL_Driver/Inc
)

# Sources (ADJUST PATHS AND ADD ALL YOUR .c FILES)
set(sources_SRCS
    ${CMAKE_CURRENT_SOURCE_DIR}/Src/main.c
    ${CMAKE_CURRENT_SOURCE_DIR}/Src/stm32f4xx_it.c
    ${CMAKE_CURRENT_SOURCE_DIR}/Src/stm32f4xx_hal_msp.c
    
    # CRUCIAL SYSTEM FILES
    ${CMAKE_CURRENT_SOURCE_DIR}/Drivers/CMSIS/Device/ST/STM32F4xx/Source/Templates/gcc/startup_stm32f401xe.s
    ${CMAKE_CURRENT_SOURCE_DIR}/Drivers/CMSIS/Device/ST/STM32F4xx/Source/Templates/system_stm32f4xx.c
    
    # EXAMPLE HAL DRIVERS - MUST LIST ALL USED .c FILES
    # ${CMAKE_CURRENT_SOURCE_DIR}/Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal.c
    # ${CMAKE_CURRENT_SOURCE_DIR}/Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_gpio.c
    # ... include all other .c files used by your project
)

include("cmake/vscode_generated.cmake")

# EXECUTABLE & LINKING

set(PROJECT_ORIGINAL_NAME ${PROJECT_NAME}) # Keep the original name before adding extensions
add_executable(${PROJECT_NAME})
set_target_properties(${PROJECT_NAME} PROPERTIES OUTPUT_NAME ${PROJECT_ORIGINAL_NAME}.elf)

target_sources(${PROJECT_NAME} PUBLIC ${sources_SRCS})
target_include_directories(${PROJECT_NAME} PRIVATE ${include_DIRS})
target_compile_definitions(${PROJECT_NAME} PRIVATE ${symbols_SYMB} $<$<CONFIG:Debug>:DEBUG>)

# Compiler options
target_compile_options(${PROJECT_NAME} PRIVATE
    ${cpu_PARAMS}
    -Wall -Wextra -Wpedantic -Wno-unused-parameter
    $<$<CONFIG:Debug>:-O0 -g3 -ggdb>
    $<$<CONFIG:Release>:-Os>
)

# Linker options: uses the LINKER_SPECS variable set by the RENODE_SIMULATION option
target_link_options(${PROJECT_NAME} PRIVATE
    -T${linker_script_SRC}
    ${cpu_PARAMS}
    -Wl,-Map=${PROJECT_ORIGINAL_NAME}.map
    -u _printf_float 
    ${LINKER_SPECS}
    -Wl,--start-group
        -lc
        -lm
        -lnosys
    -Wl,--end-group
    -Wl,--print-memory-usage
)

# POST-BUILD COMMANDS (Generate .hex and .bin for physical flash)
#Create the other formats when flashing to the real hardware.
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
    COMMAND ${CMAKE_SIZE} $<TARGET_FILE:${PROJECT_NAME}>
    COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:${PROJECT_NAME}> ${PROJECT_ORIGINAL_NAME}.hex
    COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${PROJECT_NAME}> ${PROJECT_ORIGINAL_NAME}.bin
)
