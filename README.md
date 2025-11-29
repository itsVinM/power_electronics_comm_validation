# 🚀 Build Instructions

This repository supports two build modes using the `RENODE_SIMULATION` CMake option: one for hardware simulation (Renode) and one for programming the physical NUCLEO-F401RE board.

### To build with Renode (board & sensors emulation)

This uses the `rdimon.specs` linker, which includes a semi-hosting interface optimal for GDB debugging within the Renode emulator.

```bash
cmake -B build
cmake --build build
```


### To build for real microcontroller - disabling Renode
```bash
cmake -B buid -DRENODE_SIMULATION=OFF
cmake --build build
```
# Generates: power_electronics_comm_validation.elf (using nosys.specs for hardware)

### Repo structure
```bash
tree -L 3 -I 'build' 
```

```bash
.
├── CMakeLists.txt
├── CMakePresets.json
├── Inc
├── README.md
├── Src
│   ├── main.c
│   ├── startup_stm32f401xx.S
│   ├── syscall.c
│   └── sysmem.c
├── board_emu
│   └── nucleo-f401re.resc
├── cmake
│   ├── gnu-tools-for-stm32.cmake
│   └── vscode_generated.cmake
└── stm32f401xe_flash.ld
```
