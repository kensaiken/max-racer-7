# Informe técnico — Fase 1
## Ridge Racer 7 (PS3, BCES00009) → Windows nativo

**Fecha:** 16 de noviembre de 2026  
**Rol:** ingeniero principal del port  
**Alcance:** investigación del entorno, del ejecutable, de los recursos y de las
herramientas; viabilidad real de la recompilación estática.

> **ACTUALIZACIÓN (misma fecha):** el bloqueo por cifrado descrito en este
> informe se resolvió usando RPCS3 `--decrypt` sobre la copia del usuario. El
> ELF se cargó, se liftaron 14.834 funciones PPU y 42 imágenes SPU, y se localizó
> la TOC real de `RR7.DAT`. Los resultados y las cifras actualizadas están en
> [INFORME_FASE4A.md](INFORME_FASE4A.md). Las secciones que dicen “pendiente” o
> “bloqueado” reflejan el estado anterior al desbloqueo y deben leerse junto con
> ese informe.

---

## 0. Resumen ejecutivo

Se ha inspeccionado el disco real, se ha identificado su contenido, se han
analizado los formatos que no dependen de claves y se ha construido y probado un
primer ejecutable nativo de Windows. También se ha compilado la runtime completa
de **ps3recomp v0.19.0** en esta máquina.

**Resultado principal:** el port es técnicamente abordable, pero **no se puede
empezar a recompilar el juego todavía**. El `EBOOT.BIN` de esta copia es un SELF
retail cifrado (`key_revision 0x0001`, payload cifrado). Sin un `EBOOT.ELF`
descifrado de la copia legítima del usuario no es posible ejecutar el lifter de
PPU, inventariar imports ni extraer la tabla de contenidos de `RR7.DAT`.

**Lo que sí está hecho y probado:**

- Análisis completo del disco y de los 15 archivos de `USRDIR`.
- Parser propio de `PARAM.SFO`, SELF, `.nub` y PAMF, integrado en el ejecutable.
- Extracción de audio verificada con FFmpeg (ATRAC3plus 5.1 y PCM).
- Extracción de vídeo PAMF a MPEG-PS verificada con FFmpeg (MPEG-2 720p).
- Herramienta de localización de la TOC de `RR7.DAT` y extractor con RRLZ,
  validados con un ELF/DAT sintéticos (60 archivos, 10 contenedores).
- Shell nativo Win32 + D3D11 compilado y ejecutado en la RTX 5070.
- Runtime de ps3recomp compilada (160/160 tareas) con Clang/MinGW.

**Lo que no está hecho:** no se ha ejecutado el juego, ni una sola instrucción
del juego. No se ha validado la compatibilidad de ps3recomp con Ridge Racer 7.
Cualquier afirmación en contrario sería falsa.

---

## 1. Entorno y material analizado

### 1.1 Equipo

| Componente | Detectado |
|---|---|
| CPU | AMD Ryzen 7 7800X3D (objetivo declarado) |
| GPU | NVIDIA GeForce RTX 5070, 12 GB (confirmada por DXGI) |
| RAM | 32 GB DDR5 (objetivo declarado) |
| SO | Windows 11 x64 |

### 1.2 Herramientas

| Herramienta | Versión / ruta | Uso |
|---|---|---|
| Clang/LLVM | 22.1.8, target `x86_64-w64-windows-gnu` | compilación nativa |
| CMake | 3.31.12 | build system |
| Ninja | incluido en el toolchain | build |
| Python | 3.12.10 | pipeline de activos |
| Git | disponible | repositorios |
| ps3recomp | v0.19.0 (clon `rr7_work/ps3recomp`) | runtime y lifters |
| RRUnpacker | `Nenkai/RRUnpacker` (referencia MIT) | formato `.DAT` |
| vgmstream `nub.c` | referencia pública | formato `.nub` |
| FFmpeg | 9.0.2 essentials | validación de audio/vídeo |
| RPCS3 | **no instalado** | referencia / volcado del ELF |

No hay Visual Studio ni `cl` instalados. ps3recomp documenta Windows con
clang-cl + VS 2022; aun así, su runtime se ha compilado con Clang/MinGW (ver
§2.1), lo que constituye una vía adicional no oficial pero verificada.

### 1.3 Disco

Archivo: `C:\Users\User\Desktop\Port  Nativo\Ridge Racer 7 (Europe) (En,Ja,Fr,De,Es,It).iso`  
Tamaño: **8.254.324.736 bytes**. Montado como UDF en `G:\`.

| Archivo | Tamaño | SHA-256 |
|---|---:|---|
| `PS3_GAME/USRDIR/EBOOT.BIN` | 7.887.584 | `A92D3D1FBAFAD95CF29BF9D4BC7EDB7AE42BBB7D81C53EC1A7247B0FA675846B` |
| `PS3_GAME/PARAM.SFO` | 976 | `D52274470CEE281E6C668176D769C6ABFD47133145BDD8B0A19BD4391B7B3982` |

Contenido de `PS3_GAME`:

```
USRDIR/
  EBOOT.BIN          7.887.584
  RR7.DAT        5.350.496.256
  BGM.nub          615.863.504
  Game_DJ.nub      152.133.088
  Game_DJ_F.nub    134.287.488
  Game_DJ_G.nub    173.035.712
  Game_DJ_I.nub    171.146.320
  Game_DJ_S.nub    174.374.048
  auroravision.pam  59.576.320
  e3.pam           151.226.368
  ending.pam       614.176.768
  intro.pam        174.923.776
  menu.pam          49.606.656
  title.pam         73.951.232
  title2.pam        76.324.864
```

Total de `USRDIR`: **7,43 GiB**. No hay `.sprx` ni módulos de juego externos:
todo el código del título está en `EBOOT.BIN`.

### 1.4 Metadatos (`PARAM.SFO`)

| Clave | Valor |
|---|---|
| `TITLE` | `RIDGE RACER 7` |
| `TITLE_ID` | **BCES00009** |
| `VERSION` | `01.01` |
| `CATEGORY` | `DG` (disco) |
| `PS3_SYSTEM_VER` | `01.5000` |
| `RESOLUTION` | `63` (máscara de modos; incluye 1080p) |
| `SOUND_FORMAT` | `263` |
| `PARENTAL_LEVEL` | `2` |

La versión europea retail **no es** una de las que RRUnpacker soporta
(`NPUB30457` / `NPEB00513`, versiones PSN). Por eso la localización de la TOC se
ha implementado como herramienta propia y no se ha asumido ningún offset ajeno.

---

## 2. Herramientas: qué se puede aprovechar y qué no

### 2.1 ps3recomp

**Arquitectura verificada en el repositorio:**

- `tools/ppu_loader.py`: imagen, tabla OPD, TOC, imports de firmware.
- `tools/ppu_disasm.py` y `tools/ppu_lifter.py`: PowerPC64/VMX → C++.
- `tools/spu_disasm.py`, `spu_lifter.py`, `extract_spu_images.py`,
  `find_spu_functions.py`: ISA SPU, Local Store, DMA y canales.
- `runtime/`: contexto PPU/SPU, memoria virtual, syscalls LV2, hilos.
- `libs/`: HLE de cellGcmSys, cellSpurs, cellAudio, cellPad, cellFs,
  cellSysutil, cellVideoOut, cellResc, cellSync, códecs, etc.
- `libs/video/rsx_live_draw.c` + `rsx_d3d12_backend.c`: procesador de comandos
  RSX/NV4097 y backend **D3D12** que ejecuta los shaders del título.
- `templates/project/`: scaffold de un port por juego.

**Estado real (README del proyecto, v0.19.0):** los ports de referencia llegan a
menús y juego con títulos retail (Tornado Outbreak, The Simpsons Arcade, inFamous,
Killzone 3, Twisted Metal). La fila de Windows declara “runs a recompiled game”
para el árbol, pero **no es una promesa de compatibilidad con RR7**.

**Compilación en esta máquina:**

- Configuración CMake correcta.
- Primer build: **fallo** en `runtime/spu/spu_workload.c:1017`:
  `GetCurrentThreadStackLimits` no declarada. Causa: los headers MinGW solo la
  declaran con `_WIN32_WINNT >= 0x0602`, y el proyecto no define `_WIN32_WINNT`.
- Solución aplicada **sin parchear fuentes**: reconfigurar con
  `-DCMAKE_C_FLAGS="-D_WIN32_WINNT=0x0A00"` y el equivalente de C++.
- Resultado: **160/160 tareas**, `build/libps3recomp_runtime.a` de 2.875.548
  bytes. Avisos no bloqueantes en el backend D3D12 (`#pragma comment` ignorado,
  variables sin usar).

**Limitaciones relevantes para RR7:**

1. El lifter necesita el ELF descifrado. No hay atajo legal sin él.
2. `cellSpurs` está marcado como parcial; si RR7 usa SPURS de forma intensiva,
   es el mayor riesgo de la Fase 4.
3. `cellPamf` **no parsea correctamente PAMF 4.1**: su implementación asume
   `data_offset` en 0x08, `stream_table` en 0x10 y `num_streams` en 0x14. En
   RR7, 0x08 es el tamaño de cabecera en unidades de 2048, 0x0C el tamaño de
   datos y 0x10–0x1F reservado. Nuestro parser sigue la estructura real de
   RPCS3 (`PamfHeader`), y el bug queda documentado para corregirlo upstream.
4. `cellAdec`/`cellAtrac`/`cellVdec` son parciales y dependen de FFmpeg opcional.
   RR7 necesita MPEG-2 y ATRAC3plus; el camino viable es FFmpeg (LGPL, cargado
   en runtime, nunca vendorizado).
5. El build completo del juego requiere dividir la salida del lifter en varias
   traducciones (problema de escala ya señalado por el propio proyecto).

### 2.2 RRUnpacker

Repositorio MIT que documenta el `.DAT` de RR6/7/PSP/Vita. Aporta:

- Estructura de descriptores de archivo (20 bytes) y de contenedor (36 bytes).
- Algoritmo RRLZ y su uso por contenedor (incluido el alineamiento del stream
  comprimido al final del área de sectores).
- Confirmación de que **la TOC vive dentro del ejecutable**, no en el `.DAT`.

Limitación: solo trae offsets para las versiones PSN. Para BCES00009 no sirve
tal cual. Por eso se ha implementado `tools/rr7_dat.py`, que **localiza** la TOC
en el ELF por estructura y valida la relación entre ambas tablas, sin offsets
hardcodeados.

### 2.3 vgmstream

`src/meta/nub.c` documenta el contenedor Namco nuSound2 y cita explícitamente
Ridge Racer 7 (PS3) para los códecs WAV, VAG y AT3. Ha servido para fijar:

- Cabecera base: `version` 0x00020100, `data_start`, `header_start`, etc.
- Entrada de tono: codec en +0x0C, tamaño en +0x14, offset en +0x18,
  subcabecera en +0x1C.
- Subcabecera en +0xBC (relativa a la entrada del tono), no +0xC0.
- ATRAC3: el stream es un RIFF/WAVE completo dentro del área de datos.

### 2.4 RPCS3

No está instalado. Se usa como **referencia de comportamiento** y como vía
prevista para que el usuario obtenga el ELF descifrado de su copia. El ejecutable
final no dependerá de RPCS3.

### 2.5 Ghidra / IDA

ps3recomp incluye `ghidra_analyze.py`, `ghidra_names.py`, `ida_analyze.py`. No se
han usado porque no hay ELF descifrado. Serán útiles en la Fase 4 para nombrar
funciones, localizar la TOC y analizar el motor.

### 2.6 Herramientas propias creadas en esta fase

| Herramienta | Estado |
|---|---|
| `rr7native --probe` | probado con el disco real |
| parsers SFO/SELF/NUB/PAMF | probados con archivos reales |
| `rr7native --nub --extract` | PCM/VAG→WAV, ATRAC3→`.at3`; probado + FFmpeg |
| `tools/rr7_dat.py` | TOC + extracción + RRLZ; probado con datos sintéticos |
| `tools/pamf_extract.py` | PAMF→MPEG-PS; probado con FFmpeg |
| `tools/test_formats.py` | 3 pruebas, todas OK |

---

## 3. Arquitectura PS3 y uso real por Ridge Racer 7

### 3.1 Cell

- **PPU**: PowerPC64 big-endian, 2 hilos hardware, VMX/AltiVec de 128 bits.
  Es donde vive el juego, la IA, la física, el streaming y la emisión de
  comandos RSX.
- **SPU**: 6 utilizables, ISA propia, 256 KiB de Local Store cada una, DMA
  explícito y canales de mailbox. No son “hilos normales”: un port fiel debe
  preservar Local Store, DMA, señales y orden.
- **Memoria**: 256 MB XDR compartidos por PPU/SPU + 256 MB GDDR3 para RSX.
  Alineamientos de 128 bytes y transferencias explícitas.

### 3.2 RSX

NVIDIA RSX (base NV47/NV40): shaders NV4097, texturas, framebuffers, Z-buffer,
render-to-texture, comandos GCM. RR7 es un título de lanzamiento con:

- **1080p nativo a 60 fps** (confirmado por Digital Foundry y por la máscara de
  resolución del SFO; la máscara por sí sola no lo prueba, pero coincide).
- Iluminación horneada/estática y geometría de baja densidad, según el análisis
  técnico público de Digital Foundry. Esto reduce el riesgo del backend RSX.
- Coches de pocos polígonos, texturas de baja resolución, pero 60 fps estables.
- Reflejos, motion blur y efectos de pantalla presentes; habrá que validar su
  traducción.

**No se puede afirmar todavía** qué porcentaje del frame usa PPU, SPU o RSX, ni
qué shaders concretos emplea: eso exige el ELF y capturas de comandos RSX.

### 3.3 Audio y vídeo observados

- Música: **43 tonos ATRAC3plus 5.1 a 48 kHz** en `BGM.nub` (587,32 MiB de
  payload), con RIFF/WAVE completo por tono.
- Voces DJ: **714 clips PCM16BE mono a 44,1 kHz por idioma**, en cinco bancos
  (`Game_DJ`, `_F`, `_G`, `_I`, `_S`), ~128–166 MiB cada uno.
- Vídeo: 7 PAMF v4.1, MPEG-2 720p (excepto `auroravision` a 320×176), con
  ATRAC3plus 5.1 en `intro` (5 pistas), `e3` y `ending`.
- El área de datos PAMF es **MPEG-PS estándar** desde el offset de cabecera;
  FFmpeg lo demuxa y decodifica el vídeo sin modificaciones.

---

## 4. Análisis del ejecutable

`EBOOT.BIN` es un SELF con cabecera `SCE\0`:

| Campo | Valor |
|---|---|
| Versión SELF | 2 |
| `key_revision` | **0x0001 (retail, cifrado)** |
| `header_type` | 1 |
| Tamaño de cabecera | 0x980 |
| Tamaño ELF descifrado esperado | 7.885.152 |
| ELF embebido | ELF64, big-endian, `ET_EXEC`, machine 21 (PowerPC64) |
| Entry point | **0x4CB300** |
| Program headers | 8 |
| Section headers | 35 |

Segmentos:

| # | Tipo | Flags | File off | Vaddr | File sz | Mem sz | Cifrado |
|---|---|---|---:|---:|---:|---:|---|
| 0 | PT_LOAD | 0x5 | 0x0 | 0x10000 | 0x481860 | 0x481860 | sí |
| 1 | PT_LOAD | 0x6 | 0x490000 | 0x4A0000 | 0x4918C | 0x4918C | sí |
| 2 | PT_LOAD | 0x400004 | 0x4E0000 | 0x10000000 | 0x149600 | 0x149600 | sí |
| 3 | PT_LOAD | 0x600006 | 0x630000 | 0x10150000 | 0x13F478 | 0x50AA20 | sí |
| 4 | PT_LOAD | 0x6600006 | 0x76F478 | 0x0 | 0 | 0 | sí |
| 5 | PT_TLS | 0x4 | 0x4D9188 | 0x4E9188 | 4 | 0x1B8 | no |
| 6 | 0x60000001 | 0 | 0x481818 | 0x491818 | 0x20 | 0x20 | no |
| 7 | 0x60000002 | 0 | 0x481838 | 0x491838 | 0x28 | 0x28 | no |

El segmento 7 es `proc_prx_param`, donde vivirían los imports de firmware. Está
cifrado en el archivo, por lo que **no se puede inventariar imports** sin
descifrar.

`unfself.py --info` confirma: `key_revision : 0x0001 (RETAIL - encrypted)`. El
descompresor de ps3recomp rechaza explícitamente este caso. No se ha intentado
eludir la protección ni se han incluido claves.

**Conclusión:** el 100% del código del juego está en este ejecutable y está
cifrado. Sin el ELF descifrado, la Fase 4 no puede comenzar.

---

## 5. Recursos y formatos (verificado)

### 5.1 `RR7.DAT`

- 5.350.496.256 bytes.
- Sin cabecera ni índice utilizables al principio: empieza con ceros y datos de
  un recurso. La TOC **no está en el `.DAT`**.
- La TOC está en el ELF, con dos tablas consecutivas:
  - descriptores de archivo de 20 bytes;
  - descriptores de contenedor de 36 bytes.
- Cada contenedor ocupa `sector_offset * 0x800` y `sector_size * 0x800` bytes.
- Contenedores comprimidos usan RRLZ; el stream comprimido está alineado al
  **final** del área de sectores:
  `base + (0x800 - compressed_size % 0x800)`.
- En las versiones PSN hay 12.810–13.313 archivos y 2.088–2.139 contenedores.
  Para BCES00009 el número exacto se obtendrá al localizar la TOC en el ELF.

### 5.2 `.nub` (nuSound2)

Cabecera base (big-endian):

| Offset | Campo |
|---:|---|
| 0x00 | versión (0x00020100 en RR7) |
| 0x04 | 0 |
| 0x08 | id de banco |
| 0x0C | número de tonos |
| 0x10 | inicio de datos |
| 0x14 | fin de datos |
| 0x18 | inicio de tabla de offsets |
| 0x1C | fin de tabla |

Entrada de tono (v2.1; +0x04 respecto al offset de tabla):

| Offset | Campo | Ejemplo BGM |
|---:|---|---|
| 0x00 | tag (`at3\0`, `wav\0`) | `at3` |
| 0x0C | codec (1 PCM, 2 VAG, 3 ATRAC3) | 3 |
| 0x14 | tamaño del stream | 0x13D80B0 |
| 0x18 | offset del stream (relativo a `data_start`) | 0 |
| 0x1C | tamaño de subcabecera | 0x40 |
| 0xBC | subcabecera específica | — |

Verificado en el disco:

- `BGM.nub`: 43 tonos, todos ATRAC3plus 6 canales/48 kHz, 587,32 MiB.
- `Game_DJ.nub`: 714 tonos, PCM16BE mono/44,1 kHz, 145,02 MiB.
- Los demás `Game_DJ_*`: misma estructura, distinto idioma.

### 5.3 PAMF

Cabecera v4.1 (big-endian), verificada contra RPCS3 y contra los archivos:

| Offset | Campo | Ejemplo `title.pam` |
|---:|---|---|
| 0x00 | `PAMF` | — |
| 0x04 | versión ASCII `0041` | — |
| 0x08 | tamaño de cabecera / 2048 | 1 |
| 0x0C | tamaño de datos / 2048 | 0x8D0C |
| 0x50 | `seq_info.size` | 0x64 |
| 0x58 | PTS inicio (90 kHz) | 90000 |
| 0x5E | PTS fin | 2789697 |
| 0x62 | `mux_rate_bound` | 120000 |
| 0x6A | número total de streams | 1 |
| 0x82 | grupo | — |
| 0x88 | primer `PamfStreamHeader` (48 bytes) | — |

Streams verificados:

| Archivo | Duración | Vídeo | Audio |
|---|---:|---|---|
| `title.pam` | 30,00 s | MPEG-2 1280×720 | — |
| `title2.pam` | 30,00 s | MPEG-2 1280×720 | — |
| `menu.pam` | 34,97 s | MPEG-2 1280×720 | — |
| `intro.pam` | 53,45 s | MPEG-2 1280×720 | 5 × ATRAC3plus 5.1 |
| `e3.pam` | 68,87 s | MPEG-2 1280×720 | ATRAC3plus 5.1 |
| `ending.pam` | 302,90 s | MPEG-2 1280×720 | ATRAC3plus 5.1 |
| `auroravision.pam` | 465,46 s | MPEG-2 320×176 | — |

El payload desde `header_size` es MPEG-PS: FFprobe identifica
`mpeg2video (Main), yuv420p, bt709, progressive, 1280x720, 29.97 fps`. Las pistas
ATRAC3plus van por *private stream* 0xBD y FFmpeg no las expone sin un demuxer
PES específico; es una tarea concreta de la Fase 4.

---

## 6. Respuestas a las preguntas de la Fase 1

### 6.1 ¿Qué herramientas podemos aprovechar?

**Directamente:**

- **ps3recomp**: runtime, lifters PPU/SPU, HLE de sistema, backend RSX→D3D12 y
  scaffold de port. Compila en esta máquina.
- **FFmpeg**: decodificación MPEG-2 y ATRAC3plus; validado con los activos
  extraídos. Se cargará en runtime (LGPL), no se vendoriza.
- **CMake/Clang/Ninja**: build nativo reproducible.
- **Python**: pipeline de activos y localización de TOC.
- **RPCS3**: referencia de comportamiento y vía de volcado del ELF propio.

**Con modificaciones:**

- **ps3recomp**: `cellPamf` 4.1, `cellAdec`/`cellAtrac` y `cellSpurs` necesitan
  trabajo para RR7. La división de la salida del lifter en varias TU también.
- **RRUnpacker**: su lógica es correcta, pero sus offsets son de PSN; para
  BCES00009 se usa nuestro localizador estructural.

**Descartado por ahora:** Ghidra/IDA (sin ELF), herramientas de SPU de terceros
(sin imágenes que analizar).

### 6.2 ¿Qué partes del juego podemos recompilar?

- **PPU:** todo el `EBOOT.BIN` (8 segmentos, entry 0x4CB300) una vez descifrado.
  ps3recomp tiene cobertura de ISA alta (medida en Uncharted, no en RR7); hay que
  medirla con `lift_audit.py` y `ppu_disasm.py`. El juego no trae `.sprx`, así
  que no hay módulos de juego externos.
- **SPU:** no se sabe aún. Hay que extraer imágenes con
  `extract_spu_images.py` e identificarlas. Si son jobs SPURS, se usa
  `cellSpurs`; si son overlays raw, el runtime de SPU con `image_id`. No se
  asume nada.
- **Recursos:** se reutilizan tal cual (audio, vídeo, modelos, texturas) tras
  extraerlos del `.DAT`. No se sustituyen por versiones genéricas.
- **RSX:** los comandos y shaders están en el código; se traducen con el backend
  D3D12 de ps3recomp. Los datos de geometría/textura se cargan sin cambios.

### 6.3 ¿Cómo vamos a gestionar el código PPU?

1. El usuario aporta `EBOOT.ELF` descifrado.
2. `ppu_loader.py` genera imagen, tabla OPD, TOC, entry y `imports.json`.
3. `find_functions.py` completa límites; `ppu_lifter.py` genera C++.
4. `gen_hle_nids.py` genera la tabla NID→HLE.
5. Se enlaza con `libps3recomp_runtime.a` y se ejecuta el entry.
6. Se corrigen imports no implementados y se valida el arranque contra RPCS3.
7. Se divide la salida en varias TU para que el build sea manejable.
8. La memoria invitada sigue siendo big-endian de 256 MB; los accesos pasan por
   los helpers de ps3recomp. Los punteros OPD se resuelven a direcciones de
   código; las llamadas indirectas pasan por el registro de funciones.
9. La simulación se ejecuta a **60 Hz fijos**; el render se desacopla.

### 6.4 ¿Cómo vamos a gestionar el código SPU?

1. Inventario: extraer todas las imágenes SPU del ELF y clasificarlas
   (SPURS kernel/policy/job, overlays raw, blobs de audio).
2. Si son SPURS: implementar/validar `cellSpurs`, `cellSpursJq`, `cellSync` y la
   cola de trabajos; el scheduler de ps3recomp es parcial.
3. Si son raw: lift por imagen, Local Store como array de 256 KiB, DMA como
   copia asíncrona con las mismas garantías, canales como colas con orden.
4. Cada job debe ser determinista respecto al original; nada de sustituirlo por
   un hilo genérico.
5. Donde el lift no sea viable, usar el registro de fallback de ps3recomp con una
   implementación propia documentada, nunca un “stub silencioso”.
6. Instrumentar contadores y comparar con RPCS3 antes de dar por bueno un job.

### 6.5 ¿Cómo vamos a adaptar el renderizado RSX?

- Base: `cellGcmSys` + procesador de comandos RSX de ps3recomp + backend D3D12.
- RR7 es un título de 2006, 1080p60, con iluminación horneada y geometría
  ligera; el esfuerzo principal estará en:
  - traducción de shaders NV4097 (vertex/fragment);
  - render-to-texture (reflejos, blur, pantalla);
  - formatos de textura y swizzling RSX;
  - precisión de profundidad y sincronización CPU/GPU;
  - gamma/color (bt709 en vídeo, espacio de color del framebuffer).
- Validación: capturas de referencia en RPCS3 y en PS3; comparación por píxel
  en escenas fijas (menú, salida, primera curva).
- El shell actual usa D3D11 para presentar y diagnosticar; la capa RSX usará
  D3D12. En Fase 4 se unifica el dispositivo de presentación (o se comparte el
  swapchain) para evitar copias.
- El vídeo PAMF se decodifica con FFmpeg (MPEG-2) y se compone como textura; el
  audio ATRAC3plus se demuxa de los PES 0xBD y se decodifica con FFmpeg.

### 6.6 ¿Qué servicios de PS3 debemos reconstruir?

Ya presentes en ps3recomp (a validar con RR7): `cellGcmSys`, `cellSysutil`,
`cellFs`, `cellPad`, `cellAudio`, `cellVideoOut`, `cellResc`, `cellSync`,
`cellSpurs`, `cellGame`, `cellSaveData`, `sys_*`, `cellFont`.

Pendientes o parciales que RR7 probablemente necesita:

| Servicio | Estado actual | Trabajo |
|---|---|---|
| `cellPamf` | incorrecto para v4.1 | reescribir según estructura RPCS3 |
| `cellDmux`/`cellVdec` | parcial | MPEG-2 + PES; FFmpeg |
| `cellAdec`/`cellAtrac` | parcial | ATRAC3plus 5.1; FFmpeg |
| `cellSpurs` | parcial | scheduler, task sets, señales |
| `cellAudio` | completo | verificar 5.1/48 kHz y mezcla |
| `cellPad` | completo | XInput; añadir teclado y mapeo |
| `cellSaveData` | completo | validar formato de RR7 |
| `sys_net`/`sceNp` | completo/parcial | online original probablemente muerto; offline primero |

El inventario definitivo de imports sale de `imports.json` tras descifrar.

### 6.7 ¿Qué dificultades específicas presenta Ridge Racer 7?

1. **Cifrado del EBOOT** (bloqueo actual).
2. **TOC dentro del ejecutable** y versión retail no soportada por RRUnpacker.
3. **5,35 GB de datos**, ~13k archivos y ~2k contenedores; extracción y
   verificación requieren tiempo y espacio.
4. **60 Hz fijos y 1080p**: la lógica no debe depender del framerate; el render
   debe poder ir a 120/144 Hz sin alterar la simulación.
5. **Audio 5.1 ATRAC3plus** y 714 clips de voz por idioma; el motor de audio
   original mezcla y modula; no basta con reproducir archivos.
6. **Vídeo PAMF con múltiples pistas**: hay que demuxar PES 0xBD.
7. **SPU/SPURS desconocidos**: es el mayor riesgo técnico; el motor Namco puede
   delegar en SPU iluminación, partículas, post-proceso o físicas.
8. **Sin símbolos**: nombres de funciones, estructuras y la TOC hay que
   reconstruirlos.
9. **Online**: PSN de PS3 está discontinuado; el objetivo inicial es offline.
10. **Física y IA originales**: no se sustituyen por un sistema genérico; se
    recompilan y se validan contra RPCS3/PS3.
11. **Formato de save**: desconocido hasta ejecutar.
12. **Legal**: no se pueden distribuir claves ni activos.

### 6.8 ¿Cuál es el camino más realista para conseguir la primera carrera?

**Condición previa (usuario):** obtener `EBOOT.ELF` descifrado de su copia
(ver `docs/LEGAL.md`). Sin esto no hay carrera posible.

Ruta propuesta, en orden:

1. **TOC y activos** (días): `rr7_dat.py locate` sobre el ELF, extraer el `.DAT`,
   inventariar coches, circuitos, texturas y audio. Validar que los archivos
   extraídos son coherentes.
2. **Arranque PPU** (semanas): loader → lifter → enlace → primera llamada HLE.
   Iterar imports hasta llegar a menús. Medir cobertura de ISA.
3. **Primer frame RSX** (semanas): capturar comandos, activar D3D12, validar
   shaders y texturas en una pantalla estática.
4. **Menús y carga de carrera** (semanas): selección de coche/circuito, carga de
   datos, entrada de mando.
5. **Primera carrera jugable** (semanas/meses): física PPU, IA, colisiones,
   cámara, audio dinámico, HUD. Criterio de aceptación: salida, aceleración,
   frenada, derrape, nitro, rivales, vueltas y meta.
6. **Fidelidad y optimización** (meses): comparar con RPCS3/PS3, corregir
   diferencias, 1080p/60+ estable, audio 5.1, guardado, progresión.
7. **Distribuible** (semanas): `.exe`, configuración, carga de datos del usuario,
   documentación, instalador.

**Estimación honesta:** con el ELF disponible y sin un bloqueo grave de SPURS,
la primera carrera es un objetivo de **2 a 4 meses de trabajo enfocado**; el port
completo (menús, progresión, todos los circuitos/coches, audio, pulido) es de
**meses a un año**. No es un proyecto de una sesión. La estimación depende
críticamente del inventario de SPU y de la complejidad del motor Namco, que aún
no se ha medido.

---

## 7. Estado por categorías

| Elemento | Categoría | Nota |
|---|---|---|
| Disco, SFO, árbol de archivos | **INVESTIGADO** | datos verificados |
| SELF y cifrado | **INVESTIGADO** | key rev 1, sin claves |
| Formato NUB | **INVESTIGADO / IMPLEMENTADO / PROBADO** | extracción + FFmpeg |
| Formato PAMF | **INVESTIGADO / IMPLEMENTADO / PROBADO** | cabecera + MPEG-PS |
| TOC de RR7.DAT | **IDENTIFICADO / IMPLEMENTADO / PROBADO (sintético)** | falta ELF real |
| RRLZ | **IMPLEMENTADO / PROBADO** | test sintético |
| Shell nativo | **IMPLEMENTADO / COMPILADO / PROBADO** | D3D11, RTX 5070 |
| Runtime ps3recomp | **COMPILADO** | 160/160, MinGW |
| Lifter PPU sobre RR7 | **PENDIENTE** | bloqueado por cifrado |
| Imports/HLE de RR7 | **PENDIENTE** | bloqueado |
| SPU/SPURS de RR7 | **PENDIENTE** | bloqueado |
| RSX de RR7 | **PENDIENTE** | bloqueado |
| Física/IA | **PENDIENTE** | bloqueado |
| Primera carrera | **PENDIENTE** | bloqueado |

---

## 8. Bloqueo crítico y acción requerida

Para continuar, el usuario debe aportar **su propio** `EBOOT.ELF` descifrado,
obtenido de una copia legítima. Opciones:

1. **RPCS3** (recomendada por la guía de ps3recomp): activar el volcado de
   ejecutable en las opciones de depuración al cargar el juego y localizar el
   ELF resultante. No se debe usar RPCS3 como dependencia del ejecutable final.
2. **Consola propia con CFW**: volcar el ejecutable descifrado.
3. **Herramientas de descifrado con claves propias**: solo si el usuario las
   extrae de su consola; el proyecto no las incluye.

Colocar el archivo en:

```
RR7-Native/game/EBOOT.ELF
```

y ejecutar:

```powershell
python tools\rr7_dat.py locate --elf game\EBOOT.ELF --dat "G:\PS3_GAME\USRDIR\RR7.DAT" --json game\toc.json
```

Con eso se desbloquea la Fase 4.

---

## 9. Riesgos principales

| Riesgo | Probabilidad | Impacto | Mitigación |
|---|---|---|---|
| SPU/SPURS intensivo y no liftable | media | alto | inventario temprano, fallback documentado, comparar con RPCS3 |
| Shaders RSX no traducibles | media | alto | capturar corpus, usar backend D3D12, corregir por shader |
| Dependencia de 60 Hz en la lógica | alta | medio | acumulador fijo, render desacoplado, pruebas A/B |
| Falta de símbolos | alta | medio | Ghidra, patrones, TOC, comparación con RPCS3 |
| Audio ATRAC3plus 5.1 | media | medio | FFmpeg + demuxer PES propio |
| Rendimiento 1080p60 | baja/media | medio | hardware moderno; perfilado y cachés |
| Formato de save | media | bajo | ingeniería inversa al ejecutar |
| Legal | alta | alto | solo copia del usuario; sin claves ni activos |

---

## 10. Próximos pasos inmediatos

1. Usuario: aportar `EBOOT.ELF` descifrado.
2. Ejecutar `ppu_loader.py`, `find_functions.py`, `prx_analyzer.py` y publicar
   el inventario de imports/funciones.
3. Localizar la TOC real y extraer `RR7.DAT`; verificar coches/circuitos.
4. Lift de un subconjunto y arranque hasta la primera llamada HLE.
5. Corregir `cellPamf` 4.1 y añadir el demuxer PES de ATRAC3plus.
6. Medir SPU: extraer imágenes y decidir SPURS vs raw.
7. Primer frame RSX y comparación con RPCS3.

---

## Anexo A — Evidencias de ejecución

### A.1 Shell nativo

```
[INFO ] Adapter: NVIDIA GeForce RTX 5070
[INFO ] Feature level: 11.1
[INFO ] Resolution: 1280x720 vsync=1
[INFO ] Presented 60 frames in 354.88 ms (169.1 fps, uncapped selftest)
[INFO ] SELFTEST OK
EXIT=0
```

### A.2 Probe del disco

```
PARAM.SFO: 10 entries
  TITLE                = RIDGE RACER 7
  TITLE_ID             = BCES00009
EBOOT: G:\PS3_GAME\USRDIR\EBOOT.BIN (7.52 MiB)
  SELF v2, header type 1, key revision 0x0001
  embedded ELF: type=2 machine=21 entry=0x4CB300 (PowerPC64)
  payload: RETAIL ENCRYPTED SELF
USRDIR total: 7.43 GiB (6 .nub, 7 .pam)
```

### A.3 Audio

```
BGM.nub: v20100 tones=43 ATRAC3 ch=6 rate=48000
Game_DJ.nub: tones=714 PCM16BE ch=1 rate=44100
ffprobe tone_0000.at3 -> atrac3p 48000 Hz, 5.1, 384 kb/s, 7:13
ffprobe tone_0000.wav -> pcm_s16le 44100 Hz, 1 channel, 0.83 s
decode exit=0
```

### A.4 Vídeo

```
PAMF 0041 title.pam: 1280x720 MPEG-2, 30.00 s
PAMF 0041 intro.pam: 1280x720 MPEG-2 + 5 x ATRAC3plus 5.1, 53.45 s
ffprobe title.mpg -> mpeg2video (Main), yuv420p, bt709, 1280x720, 29.97 fps
```

### A.5 DAT/RRLZ

```
RRLZ: OK
TOC locator + extraction: OK
all format tests passed
(60 files, 10 containers, validation ratio 1.0)
```

### A.6 ps3recomp

```
[160/160] Linking CXX static library libps3recomp_runtime.a
```

---

## Anexo B — Referencias

- sp00nznet/ps3recomp, v0.19.0, MIT.
- Nenkai/RRUnpacker, MIT (formato `.DAT`, RRLZ).
- vgmstream, `src/meta/nub.c` (formato nuSound2).
- RPCS3, `cellPamf.h` / `cellPamf.cpp` (referencia de comportamiento PAMF).
- Digital Foundry, “Ridge Racer 7: The 1080p Dream” (28/06/2009).
- Sony PS3 Cell/RSX documentation (pública).
