# libcad-dependencies
Dependency management for libcad

## Setup
- Clone repo with submodules
- For native build run `cmake -B build; cmake --build build`
- For WASM build run `emcmake cmake -B build_wasm; cmake --build build_wasm; cmake --build build_wasm --target package`