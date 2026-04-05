# CopilotScope — Developer Setup

## Prerequisites

### Windows (MSVC — recommended for production)

| Tool | Version | Notes |
|------|---------|-------|
| Visual Studio 2022 (Build Tools) | 17.x | Desktop development with C++ workload |
| Windows 11 SDK | 10.0.22621.0+ | Bundled with VS Build Tools |
| CMake | ≥ 3.25 | Bundled with VS or cmake.org |
| Node.js | 20 LTS | nodejs.org |
| npm | ≥ 10 | Bundled with Node.js |
| WiX Toolset 3.14 | 3.14.x | wixtoolset.org — **v3**, not v4 |
| pkg | Latest | `npm install -g @vercel/pkg` |
| Git | Any | — |

### Linux (MinGW-w64 cross-compile + native proxy/extension)

| Tool | Version | Notes |
|------|---------|-------|
| `g++-mingw-w64-x86-64` | 12+ | `sudo apt install g++-mingw-w64-x86-64` |
| `gcc-mingw-w64-x86-64` | 12+ | `sudo apt install gcc-mingw-w64-x86-64` |
| CMake | ≥ 3.25 | `sudo apt install cmake` |
| Node.js | 20 LTS | nodejs.org or `sudo apt install nodejs npm` |
| Wine (optional) | 9.0+ | For running/testing Windows binaries on Linux |

Optional:
- **VS Code** ≥ 1.85.0 — extension development
- **Xvfb** + **ImageMagick** — headless GUI testing on Linux

---

## Clone

```powershell
git clone https://github.com/The-Spectral-Operator/CopilotScope.git
cd CopilotScope
```

---

## Building the C++ Components

### Configure

```powershell
cmake -B build -G "Visual Studio 17 2022" -A x64 `
    -DCMAKE_BUILD_TYPE=Release `
    -DBUILD_TESTS=OFF
```

### Build daemon + GUI

```powershell
cmake --build build --config Release --parallel
```

Output:
- `build\Release\copilotscoped.exe` — daemon
- `build\Release\copilotscope.exe`  — GUI

### Build with tests

```powershell
cmake -B build_test -G "Visual Studio 17 2022" -A x64 -DBUILD_TESTS=ON
cmake --build build_test --config Debug --parallel
ctest --test-dir build_test -C Debug --output-on-failure -j 4
```

---

## Building on Linux (Cross-Compile)

### Quick start

```bash
chmod +x build_linux.sh
./build_linux.sh
```

This script:
1. Builds the Node.js proxy natively
2. Runs proxy tests (44 tests)
3. Builds and tests the VS Code extension (23 tests)
4. Cross-compiles the C++ daemon + GUI using MinGW-w64

### Manual cross-compile

```bash
mkdir build_mingw && cd build_mingw
cmake .. -DCMAKE_TOOLCHAIN_FILE=../cmake/mingw-w64-x86_64.cmake \
         -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
```

Output:
- `build_mingw/src/daemon/copilotscoped.exe` — PE32+ daemon (3.7 MB)
- `build_mingw/src/gui/copilotscope.exe` — PE32+ GUI (1005 KB)

### Testing under Wine (optional)

```bash
sudo apt install wine64 xvfb imagemagick xdotool
Xvfb :99 -screen 0 1920x1080x24 &
DISPLAY=:99 wine build_mingw/src/gui/copilotscope.exe
```

---

## Building the VS Code Extension

```powershell
cd src\extension
npm install
npm run compile
npm run lint
npm run package
```

To install locally:
```powershell
code --install-extension copilotscope-sidecar-1.0.0.vsix
```

### Extension dev loop

1. Open `src/extension/` in VS Code.
2. `F5` → Extension Development Host.
3. `Ctrl+Shift+P` → `CopilotScope: Open Monitor`.
4. TypeScript changes hot-reload if `npm run watch` is running.

---

## Building the HTTPS Proxy

### Dev mode

```powershell
cd src\proxy
npm install
node proxy.js --port 8877
```

### Standalone EXE

```powershell
cd src\proxy
npm install
npx pkg proxy.js --target node20-win-x64 --output ..\..\build\Release\cs_proxy.exe
```

---

## Running Tests

### Extension

```powershell
cd tests\extension
npm install
npm test
```

### Proxy

```powershell
cd tests\proxy
npm install
npm test
```

### C++ unit tests

```powershell
ctest --test-dir build_test -C Debug -R test_token_counter -V
ctest --test-dir build_test -C Debug -R test_price_calc    -V
ctest --test-dir build_test -C Debug -R test_anomaly_engine -V
ctest --test-dir build_test -C Debug -R test_pipe_server   -V
```

### Integration tests

Requires a built `build\Release\copilotscoped.exe` and `node.exe` on PATH.

```powershell
cd tests\integration
.\test_full_session.ps1 -Verbose
```

---

## Building the MSI Installer (WiX 3.14)

Build all binaries first, then:

```powershell
# Compile all WiX source files
candle.exe -dBuildDir=build -dSourceRoot=. `
    packaging\wix\Product.wxs `
    packaging\wix\Directories.wxs `
    packaging\wix\Components.wxs `
    -ext WixUIExtension `
    -out build\wix_obj\

# Link the MSI
light.exe build\wix_obj\*.wixobj `
    -ext WixUIExtension `
    -b . `
    -o build\CopilotScope-1.0.0.msi
```

Build the Burn bootstrapper:

```powershell
candle.exe -dBuildDir=build -dSourceRoot=. `
    packaging\wix\Bundle.wxs `
    -ext WixBalExtension `
    -out build\wix_obj\Bundle.wixobj

light.exe build\wix_obj\Bundle.wixobj `
    -ext WixBalExtension `
    -o build\CopilotScope-1.0.0-setup.exe
```

---

## Code Signing

```powershell
# EV cert by thumbprint:
.\packaging\sign\sign.ps1 `
    -CertThumbprint AABBCCDDEEFF001122334455 `
    -BuildDir .\build\Release

# PFX file:
.\packaging\sign\sign.ps1 `
    -PfxPath .\signing\cert.pfx `
    -BuildDir .\build\Release
```

---

## Project Structure

```
src\
  daemon\         C++20, Win32 service
  gui\            C++20, Win32 GUI (theme_manager.cpp handles all 6 themes)
  extension\      TypeScript, VS Code Extension API
  proxy\          Node.js 20, CommonJS
shared\           Cross-component protocol headers + pricing data
tests\
  daemon\         Google Test unit tests
  extension\      Jest unit tests
  proxy\          Jest unit tests
  integration\    PowerShell E2E harness
packaging\
  wix\            WiX 3.14 MSI + Burn bootstrapper
  sign\           Authenticode signing script
docs\             Documentation
third_party\
  sqlite3\        SQLite amalgamation
```

---

## SQLite Amalgamation

```powershell
mkdir third_party\sqlite3
copy sqlite-amalgamation-*\sqlite3.c third_party\sqlite3\
copy sqlite-amalgamation-*\sqlite3.h third_party\sqlite3\
```

---

## Common Build Errors

| Problem | Fix |
|---------|-----|
| `CMAKE_CXX_COMPILER not set` | Run from Developer Command Prompt |
| `LNK1561: entry point must be defined` | Verify `WIN32` is set on the EXE target |
| `error C2220: warning treated as error` | Remove `/WX` from third-party header paths |
| `node-forge` slow in tests | RSA keygen is CPU-bound; 5–10s expected |
| `npm ERR! peer dep missing` | `npm install --legacy-peer-deps` in `src/extension` |
| `candle: error CNDL0001` | Verify WiX 3.14 is on PATH, not WiX 4 |

---

## Coding Conventions

### C++ (Daemon + GUI)
- C++20, MSVC `/std:c++20`
- `namespace cs {}` — daemon; `namespace gui {}` — GUI
- RAII for all handles
- No exceptions in hot paths — return codes + optional out-params
- All public headers: `#pragma once`, no include guards

### TypeScript (Extension)
- `strict: true`
- No `any` except where VS Code API forces it
- `async`/`await` throughout

### Node.js (Proxy)
- CommonJS (`require`/`module.exports`)
- `'use strict'` top of every file
- Error-first callbacks for internal helpers; `async`/`await` for top-level
