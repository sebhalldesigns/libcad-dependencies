# libcad-dependencies
Dependency management for libcad

## Setup
- Clone repo with submodules
- For native build run `cmake -B build -DCMAKE_BUILD_TYPE=Release; cmake --build build --config Release;  cmake --build build --config Release --target package;`
- For WASM build run `emcmake cmake -B build_wasm -DCMAKE_BUILD_TYPE=Release; cmake --build build_wasm; cmake --build build_wasm --target package`