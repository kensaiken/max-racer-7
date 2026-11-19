# Compilación

## 1. Toolchain usado

| Herramienta | Versión | Ruta |
|---|---|---|
| Clang/Clang++ | 22.1.8 | `%USERPROFILE%\.local\share\retcomm\toolchains\cmake-clang-v1\latest\bin` |
| CMake | 3.31.12 | ídem |
| Ninja | incluido | ídem |
| Python | 3.12.10 | `%LOCALAPPDATA%\Programs\Python\Python312` |
| Windows SDK headers | MinGW-w64 | toolchain |
| D3D11/DXGI/XInput | libs del toolchain | `libd3d11.a`, `libdxgi.a`, `libdxguid.a`, `libxinput.a` |

No se requiere Visual Studio para el shell. ps3recomp documenta clang-cl + VS
2022 para Windows; este proyecto ha validado además la vía Clang/MinGW.

## 2. Shell nativo

```powershell
cd C:\Users\User\Desktop\MR7-Native
.\build.ps1
```

Equivalente manual:

```powershell
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build --parallel
```

Salida: `build\maxracer7.exe`.

### Pruebas

```powershell
.\build\maxracer7.exe --selftest --frames 60
.\build\maxracer7.exe --window --frames 60
python run_tests.py
```

Resultados esperados:

```
Adapter: NVIDIA GeForce RTX 5070
Feature level: 11.1
Presented 60 frames ...
SELFTEST OK
```

## 3. ps3recomp

```powershell
cd C:\Users\User\Desktop\rm7_work\ps3recomp

python -m pip install -r requirements.txt

cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release `
      -DCMAKE_C_FLAGS="-D_WIN32_WINNT=0x0A00" `
      -DCMAKE_CXX_FLAGS="-D_WIN32_WINNT=0x0A00"

cmake --build build --parallel
```

### Incidencia y solución

Sin `_WIN32_WINNT`, MinGW no declara `GetCurrentThreadStackLimits` (Windows 8+)
y falla `runtime/spu/spu_workload.c:1017`. La solución es definir
`_WIN32_WINNT=0x0A00`; no se ha modificado el código de ps3recomp.

Salida: `build\libps3recomp_runtime.a` (2.875.548 bytes).

### Estado del build

```
[160/160] Linking CXX static library libps3recomp_runtime.a
```

Avisos no bloqueantes:

- `#pragma comment(lib, ...)` ignorado por MinGW (las libs se enlazan en CMake).
- variables sin usar en el backend D3D12.

## 4. Herramientas Python

No requieren instalación adicional para los tests:

```powershell
python tools\test_formats.py
```

FFmpeg es opcional y solo para validación externa:

```powershell
ffprobe out\dj\tone_0000.wav
ffprobe out\bgm\tone_0000.at3
```

## 5. Problemas frecuentes

| Síntoma | Causa / solución |
|---|---|
| `GetCurrentThreadStackLimits` no declarada | añadir `-D_WIN32_WINNT=0x0A00` |
| `CreateDXGIFactory` unresolved | enlazar `dxgi` y `dxguid` |
| `D3D11CreateDevice failed` | driver/GPU; el selftest informa el HRESULT |
| `Could not load configuration` | ejecutar desde la raíz o pasar `--config` |
| `not an ELF file` en `maxracer_dat.py` | el EBOOT sigue cifrado; hace falta el ELF |
| `KeyError` en tests | ejecutar `python run_tests.py` desde la raíz |

## 6. Distribución

Para una build distribuible:

```powershell
cmake -B build-release -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build-release --parallel
```

Copiar:

```
maxracer7.exe
config\rm7.ini
docs\
```

Los datos del juego **no** se incluyen; el usuario aporta su copia y el port
extrae `gamedata/` en el primer arranque.
