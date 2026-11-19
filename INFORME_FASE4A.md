# Informe Fase 4A — Desbloqueo y primera recompilación
## Max Racer 7 (BCES00009)

**Fecha:** 16 de noviembre de 2026  
**Estado:** PPU y SPU recompilados; build nativo en curso. **Todavía no se ha
ejecutado el juego.**

---

## 1. Desbloqueo del ejecutable

El `EBOOT.BIN` retail se descifró con **RPCS3 0.0.43** en modo headless:

```powershell
rpcs3.exe --decrypt EBOOT.BIN
# Decrypted EBOOT.BIN -> EBOOT.elf
```

RPCS3 se usó únicamente como herramienta de volcado sobre la copia del usuario.
**No se ha copiado ninguna clave al proyecto**; el binario de RPCS3 las contiene
y el proyecto no las redistribuye.

| Archivo | Tamaño | SHA-256 |
|---|---:|---|
| `EBOOT.BIN` | 7.887.584 | `A92D3D1FBAFAD95CF29BF9D4BC7EDB7AE42BBB7D81C53EC1A7247B0FA675846B` |
| `EBOOT.elf` | 7.885.152 | `5FAA5AF92202C4054FD095029A0A06B15FB303E53DD8E2E11288FB726A4B03D5` |

Verificación: ELF64, big-endian, `ET_EXEC`, machine 21 (PowerPC64), entry
`0x4CB300`.

---

## 2. Loader PPU

`tools/ppu_loader.py` sobre el ELF real:

```
entry_opd   = 0x4CB300
entry_code  = 0x1022C
entry_toc   = 0x4EEA88
module_toc  = 0x4EEA88
opd_base    = 0x4CB2F0
opd_size    = 0x1B798
functions   = 14.067
load segs   = 4
```

Segmentos cargables:

| Vaddr | Filesz | Flags |
|---|---:|---:|
| `0x10000` | `0x481860` | R-X |
| `0x4A0000` | `0x4918C` | RW |
| `0x10000000` | `0x149600` | R-- |
| `0x10150000` | `0x13F478` | RW |

La TOC del ejecutable (usada por el juego) es `0x4EEA88`, distinta de la TOC de
la tabla de archivos (ver §5).

---

## 3. Imports y cobertura HLE

- **262 imports** en **23 librerías**.
- ps3recomp registra **1.238 handlers** en **93 módulos**.
- Cobertura por NID: **189/262 (72,1%)**.
- Faltan **73**.

| Librería | Faltan | Relevancia |
|---|---:|---|
| `sceNp` | 19 | online; probablemente no crítico para offline |
| `sysPrxForUser` | 11 | CRT/arranque; **crítico** |
| `cellUsbd` | 7 | USB; no crítico |
| `cellAtrac` | 6 | audio; **crítico** |
| `sceNpMatchingInt` | 5 | online |
| `cellFont` | 5 | menús/HUD; importante |
| `sys_net` | 5 | red; no crítico offline |
| `cellPamf` | 4 | vídeo; importante |
| `cellDmux` | 3 | demux de vídeo; importante |
| `cellGcmSys` | 2 | `_cellGcmInitBody`, `_cellGcmFunc15`; **crítico** |
| `cellResc` | 2 | escalado de vídeo |
| `cellL10n` | 2 | localización |
| `cellFontFT` | 1 | fuentes |
| `cellSysutil` | 1 | sistema |

Los imports cubiertos incluyen **13/13 de `cellSpurs`**, 13/15 de `cellGcmSys`,
9/9 de `cellAudio`, 26/27 de `cellSysutil`, 8/8 de `cellAdec`, 10/10 de
`cellDmux` (por NID registrado; la implementación puede ser parcial).

**Matiz importante:** la cobertura anterior compara solo la tabla generada por
`gen_hle_nids.py`. El runtime registra además funciones por contexto
(`ps3_hle_register_ctx`), como `sys_initialize_tls`, y `_cellGcmInitBody` está
implementada en `cellGcmSys.c` aunque su nombre C no genere el NID exacto. Por
tanto, 189/262 es una **cota inferior**, no el número real de funciones
ejecutables. El primer arranque dirá qué falta de verdad.

**Conclusión:** el arranque se detendrá con alta probabilidad en
`sysPrxForUser` (CRT) y `_cellGcmInitBody`. Son las dos primeras tareas de HLE.

---

## 4. Recompilación PPU

```powershell
python tools\ppu_lifter.py EBOOT.elf `
    --functions analysis\EBOOT.functions.json `
    --hle-stubs analysis\EBOOT.imports.json `
    --output lift --chunks 8 --jobs 8
```

Resultado:

```
14.834 funciones liftadas
7.078 destinos de llamada únicos
9 archivos: ppu_recomp_000.cpp .. ppu_recomp_007.cpp + ppu_recomp_table.cpp
1.507.701 líneas de C++ (88,8 MB)
86 wrappers de entrada a mitad de función
```

Los 9 TUs pasan `clang++ -fsyntax-only` sin errores (solo avisos de paréntesis
en comparaciones generadas).

### 4.1 Marcadores TODO

| Tipo | Cantidad |
|---|---:|
| `.word` (datos/jump tables embebidos en .text) | 85.461 |
| VMX no decodificadas | ~1.800 |
| `op31` no decodificadas | ~430 |
| **Total** | **87.691** |

Las VMX pendientes más frecuentes: `vmx_x1616`, `vmx_x1612`, `vmx_x1695`,
`vmx_x1693`, `vmx_x1620`. Las `op31`: `op31_x641`, `op31_x128`, `op31_x831`,
`op31_x896`, `op31_x384`.

No son un bloqueo inmediato: muchas están en rutas no ejecutadas al arrancar,
pero deben corregirse antes de validar el juego completo. La mayoría de los
TODOs son datos, no instrucciones.

### 4.2 Build del port

Se creó `mr7_work/port` desde `templates/project`, con:

- `recompiled/` = los 9 TUs + header;
- `game/EBOOT.elf`;
- `PORT_SPU_INPUTS=EBOOT.elf`, que descubrió y liftó **42 imágenes SPU**;
- runtime ps3recomp + backend D3D12.

La compilación terminó correctamente: **`MyGameRecomp.exe` (66.412.544 bytes)**.
El enlace necesitó añadir `d3d12 dxgi d3dcompiler dxguid` porque MinGW ignora
los `#pragma comment(lib, ...)` del backend.

---

## 5. SPU

`extract_spu_images.py` extrajo **42 imágenes SPU** (1.005.116 bytes) del ELF.
Son ELF32 big-endian, machine SPU, entry `0x3010`, 3 program headers, 12
secciones. Tamaños entre 11,9 KB y 76,8 KB.

El proyecto de port las liftó automáticamente con
`cmake/PS3RecompSPU.cmake`; cada imagen tiene su `spu_recomp.c` (generado) y su
`image.elf`. El inventario está en
`port/build/generated_spu/inventory.json`.

Pendiente: identificar cuáles son jobs SPURS, cuáles overlays raw y cuáles
audio; validar el scheduler y los canales en ejecución.

---

## 6. MR7.DAT: TOC real y extracción

`tools/maxracer_dat.py locate` sobre el ELF real:

```
toc_offset             = 0x630128
file_count             = 13.313
container_count        = 2.139
file_validation_ratio  = 1.0
dat_size_field_offset  = 0x630068
```

Coincide con la disposición de la versión PSN NPEB00513 (13.313 archivos /
2.139 contenedores), pero se ha localizado estructuralmente, sin hardcodear el
offset.

Contenedores:

- 947 comprimidos con RRLZ;
- 1.192 sin comprimir;
- suma de tamaños de archivo: 10,54 GiB (los descriptores incluyen duplicados y
  tamaños descomprimidos; el `.DAT` físico son 4,98 GiB).

Extracción real verificada:

```
SoundConfig_Free: Config3D (6.704 B), ConfigTxt (9.154 B)
DB: 13 archivos (Bgm, Car, Course, Race, ...), 0,16 MiB
```

RRLZ funciona sobre datos retail. Los archivos `DB/*` tienen cabecera binaria
(`01 02 01 51 ...`), pendiente de documentar.

---

## 7. Primer arranque nativo (verificado)

El ejecutable cargó el ELF y ejecutó código real del juego. Evidencia del log:

```
[D3D12] device: "NVIDIA GeForce RTX 5070" vendor=0x10DE device=0x2F04 VRAM 11943 MB
[D3D12] Depth buffer created (1280x720 D24S8)
[D3D12] Initialization complete (1280x720, 2 buffers, pipeline=ready)
[cellVideoOut] Configure: resId=4, format=0, aspect=1, pitch=8192
[cellGcmSys] SetDisplayBuffer(id=0, offset=0x0, pitch=8192, 1920x1080)
[cellGcmSys] SetDisplayBuffer(id=2, offset=0x880000, pitch=8192, 1920x1080)
[cellGcmSys] SetZcull(index=0, offset=0x1100000, 1920x1088)
[cellFont] Init()
[SYS] sys_ppu_thread_create tid=3 name="TTFontRender Thread"
[hle] unresolved NID 0x7A0A83C4
[hle] unresolved NID 0x98AC5524
[null-read] guest read32 from 0x00000020 (NULL+0x20) by guest-fn=0x002D14C0
```

Interpretación:

- El arranque **no** se queda en el loader ni en el CRT: llega a la
  inicialización de vídeo, GCM y fuentes.
- El backend D3D12 crea dispositivo, depth buffer, pipelines y vertex buffer.
- MR7 configura framebuffers de **1920×1080** y tiles GCM.
- Se crea el hilo `TTFontRender Thread` y el hilo principal espera un semáforo.
- El hilo de fuentes se bloquea porque faltan:
  - `0x7A0A83C4` = `cellFontInitLibraryFreeTypeWithRevision`
  - `0x98AC5524` = `cellFontGetFontIdCode`
- Las lecturas a `NULL+0x20/0x1C/0x0` desde `0x002D14C0` son la consecuencia:
  la inicialización de FreeType devuelve un puntero nulo y el juego lo
  desreferencia.

El proceso se dejó 180 s y se terminó; no hubo crash del host ni del backend.
El log está en `mr7_work/port/boot_stdout.log` y `boot_stderr.log`.

**Conclusión:** primera ejecución nativa de código de MR7 conseguida. El
siguiente bloqueo es HLE de `cellFont`, no el lifter ni el arranque.

### 7.2 Tercer arranque: mempool y fuente por defecto

Se implementó la familia `sys_mempool_*` en `sysPrxForUser.c`:

- `sys_mempool_create`, `sys_mempool_destroy`, `sys_mempool_free_block`,
  `sys_mempool_try_allocate_block`, `sys_mempool_allocate_block`,
  `sys_mempool_get_count`.

El juego crea cuatro pools al arrancar y ya no aparece el NID sin resolver
`0xCA9A60BF`:

```
sys_mempool_create(chunk=0x10799E00, chunk_size=3072, block=1024, align=1024)
sys_mempool_create(chunk=0x1079AA10, chunk_size=1536, block=512,  align=512)
sys_mempool_create(chunk=0x1079B020, chunk_size=4352, block=256,  align=256)
sys_mempool_create(chunk=0x1079C130, chunk_size=1536, block=512,  align=512)
```

También se aceptó `cellFontGetFontIdCode(NULL, ...)` como consulta de fuente por
defecto (la SDK real no la rechaza; nuestro HLE devolvía
`CELL_FONT_ERROR_INVALID_PARAMETER` y el juego saltaba la inicialización).

Resultado: el arranque ya no muestra `PTRLEAK` de `cellFontGetFontIdCode`, pero el
hilo `TTFontRender` sigue leyendo el primer campo de `0x1055D520` a cero:

```
0x2D1584  lwz r9, 0x0(r11)   ; r11 = 0x1055D520
0x2D1588  lwz r0, 0x20(r9)   ; r9 == 0
```

El objeto existe (r24 = 0x1055D520), pero su primer campo no se inicializa antes
de que el hilo lo lea. La hipótesis actual es **orden de inicialización**: el
hilo se crea en `0x2D2118` y el handshake del semáforo 4 no garantiza que el
hilo principal termine de rellenar el contexto antes de que el worker lo use.
La alternativa es un constructor estático que no se ejecuta en el lift.

La tabla HLE queda en **1.250 handlers**. El siguiente paso es instrumentar el
semáforo 4 y el objeto `0x1055D520`, no añadir más stubs.

### 7.3 Diagnóstico opt-in del contexto de fuentes

Con `MR7_FONT_SEED=1` se siembra el contexto `0x1055D520` con un buffer invitado
de 1024×1024 en `0x0F000000`. Resultado:

- desaparecen las lecturas nulas de 32 bits en `+0x20/+0x1C/+0x0`;
- tras hacer tolerante `cellFontRenderCharGlyphImage` con `font == 0`, también
  desaparecen las lecturas de 8 bits;
- quedan escrituras a NULL (el runtime las descarta) y, finalmente, **ambos
  hilos esperan el semáforo 4**;
- no hay crash del host; el hilo queda esperando.

Por tanto, el seed **no es una solución**: demuestra que el primer campo era el
buffer y desplaza el bloqueo a la inicialización/sincronización posterior. Queda
**desactivado por defecto** y solo se usa como sonda de bring-up.

El siguiente objetivo es identificar la estructura que se recorre en
`0x2D0868`/`0x424F10` y el orden real del semáforo 4.

### 7.4 Requisito del VFS y naturaleza del semáforo 4

Se descubrió un requisito operativo importante: con el ISO **desmontado**, el
hilo principal entra en un bucle de:

```
[fs] open FAIL '/dev_bdvd/PS3_GAME/USRDIR/MR7.DAT'
     -> 'G://PS3_GAME/USRDIR/MR7.DAT'
```

Al montar el ISO y usar `PS3_VFS_ROOT=G:/`, los `open FAIL` desaparecen. El disco
debe estar montado antes de arrancar; el script `port/run_boot.ps1` ahora lo
verifica y falla con un mensaje claro si `MR7.DAT` no está accesible.

Con el VFS correcto, el bloqueo restante es el semáforo 4:

- `tid=1` (principal) espera en `0x2D2180`;
- `tid=3` (TTFontRender) postea y luego espera en `0x2D1644`;
- no hay más `post`.

Forzar el wait del worker (`MR7_FONT_NOWAIT=1`) produce un **spin de millones de
iteraciones**, no avance: el semáforo 4 es un **frame loop**, no un simple
handshake de inicialización. Por eso la solución no es saltarse la espera, sino
implementar correctamente el contexto/señal de frame del renderizador de fuentes.

El CPU del proceso con el wait normal es ~0.5 s en 10 s: está bloqueado, no
cargando el `.DAT`.

### 7.5 Estado tras corregir el VFS: pipeline de carga

Con el ISO montado y `MR7_FONT_SEED=1`, el hilo principal ya no se queda en
fuentes. Se crean los hilos reales del motor:

| tid | nombre | función |
|---:|---|---|
| 4 | `Draw Thread` | bucle de presentación/flip |
| 5 | `spu_printf_handler` | salida SPU |
| 6 | `File Thread` | I/O |
| 7 | `sample_game_data` | carga de muestra; sale tras syscalls de mapper |
| 8 | `bnusCoreDecoderServerThread` | audio |
| 10 | `bnusCoreUpdateThread` | audio |
| 11 | `Initialize Thread` | inicialización; gira sobre mutex 17 |
| 12 | `auto_load` | carga automática; sale tras syscalls de mapper |

El hilo principal alternativa semáforos 3/5/7 con `Draw Thread` y `File Thread`:
es el **pipeline de carga/render**, no un deadlock de fuentes. Sin embargo:

- no aparecen lecturas de `MR7.DAT` en `PS3_FSTRACE`;
- `sample_game_data` y `auto_load` terminan inmediatamente después de
  `sys_mapper`/`sys_memory`;
- `Initialize Thread` gira en `mutex 17` sin trabajo;
- el CPU está casi idle.

Conclusión: el siguiente bloqueo está en el **loader de activos**: los hilos de
carga salen sin pedir datos o el `cellFs`/`File Thread` no recibe las peticiones.
El siguiente paso es instrumentar las aperturas/lecturas exitosas de `cellFs` y
la función `0x4CB6F0` (`File Thread`), no el font.

### 7.6 Hito: SPU de audio ejecutándose y primeros frames D3D12

Con `MR7_FONT_SEED=1` y `MR7_FONT_NOWAIT=1` (yield de 1 ms en el wait del hilo
de fuentes), el arranque avanza hasta el motor real:

```
[SPU] group_create -> id=0x1000 num=1 name=bnusCore
[SPU] group_start id=0x1000 tid=0x2000 entry=0x00090
      -> LIFTED SPU execution
[SPU] group_start id=0x1000 (1 host threads running, 0 instant)
[SPU] tid=0x2000 RUNNING lifted image entry=0x00090
[cellFont] GetFontIdCode(font=0x1055DA54, code=0)   <- fuente válida, no NULL
[RSX] CLEAR_SURFACE mask=0xF3 color=0x00000000
[RSX] DRAW_ARRAYS prim=5 first=0 count=3
```

Cambios que lo desbloquearon:

1. **Slots SPU 1-based:** el grupo `bnusCore` tiene 1 hilo y el juego lo registra
   en `index=1`; los bucles de `group_start` solo recorrían `i < num_threads`
   (slot 0). Ahora recorren los 8 slots y saltan `MAX_SPU_THREADS`. Resultado:
   `1 host threads running`, la imagen SPU liftada se ejecuta.
2. El contexto de fuentes acaba con una fuente válida (`0x1055DA54`), no NULL.

Nuevo bloqueo: **D3D12 PSO**. El juego ya envía draws, pero la creación de
pipeline states falla:

```
[VP] colour PSO FAIL (0x887A0005)
[VP] PSO creation FAIL (0x887A0005)
[FP] PSO FAIL (fp=0x043E4C81, 0x887A0005)
```

`0x887A0005` es `DXGI_ERROR_DEVICE_REMOVED`. No hay eventos TDR de
`nvlddmkm`; `D3D12_NO_DBG=1` no cambia el resultado (29 fallos PSO / 14 draws en
12 s). El primer fallo ocurre tras `set_render_target(256x64)` y
`set_shader(fp=0x043E4C80)`. Hay que depurar ese render target/PSO (root
signature, shader o recurso inválido), no el lifter.

También aparece `open FAIL /dev_hdd0/game/BLJS10001/USRDIR/MR7.DAT` (ruta de
instalación en HDD de otra versión); es un fallback, no bloquea el arranque.

### 7.7 PSO de fragmento: `INVALID_CALL` con descripción válida

El path VP de debug font no es la única causa: con `MR7_NO_VP=1` el PSO de
fragmento invitado sigue fallando. La descripción registrada es:

```
[FP] PSO desc fp=0x043E4C81 nrt=1 rtfmt=41 vs=2228 ps=916 root=<non-null>
[FP] PSO FAIL (0x887A0005)
[FP] device removed reason=0x887A0001   (DXGI_ERROR_INVALID_CALL)
```

`rtfmt=41` es `R32_FLOAT` y `rtfmt=28` es `R8G8B8A8_UNORM`: ambos son formatos
de color válidos. `nrt=1`, VS/PS tienen bytecode y la root signature existe.
El PSO se rechaza igualmente con `INVALID_CALL`, lo que apunta a una
**incompatibilidad de firma VS/PS, input layout o root signature** que solo la
debug layer de D3D12 puede nombrar con precisión.

El siguiente paso es capturar los mensajes de `ID3D12InfoQueue` (o
`OutputDebugString`) al crear el PSO; sin ese mensaje no tiene sentido seguir
cambiando parámetros a ciegas.

### 7.8 Debug layer no disponible y siguiente test

`D3D12_DBG=1` devuelve `0x887A002D` (`DXGI_ERROR_SDK_COMPONENT_MISSING`): la
característica opcional **Graphics Tools** no está instalada y la sesión no es
administrador, así que no se puede activar `ID3D12InfoQueue`.

Comprobaciones realizadas:

- la root signature VP se crea con `hr=0x00000000` (puntero válido);
- con `FP_SIMPLE=1` (PS trivial `SV_Position` → color fijo) el PSO sigue
  fallando, así que **no es la firma del PS invitado**;
- el primer `DRAW_ARRAYS #0` ocurre antes del primer PSO de fragmento; es
  probable que sea ese draw base el que retira el dispositivo y que el PSO solo
  reporte el estado ya inválido.

Siguiente test: llamar a `GetDeviceRemovedReason` inmediatamente después de
`DrawInstanced` en el primer draw. Si devuelve `0x887A0001`, el problema está en
el comando/recurso del draw base (root signature/descriptor heap), no en el PSO
invitado.

### 7.9 PSO: descripción validada, falta captura

Comprobaciones adicionales:

- `blend=0`, `zs=0`, `cmask=0xF`: blend/depth desactivados y válidos;
- `rtfmt=41`/`28` (`R32_FLOAT`/`R8G8B8A8_UNORM`): formatos de color válidos;
- root signature creada con `hr=0`;
- `FP_SIMPLE=1` (PS trivial) sigue fallando;
- `AddRef` del blob VS durante `CreateGraphicsPipelineState` no cambia nada.

El PSO se rechaza con `INVALID_CALL` pese a que la descripción parece válida. Sin
la debug layer (`0x887A002D`, Graphics Tools no instalado y sin admin), el
siguiente paso es una **captura con RenderDoc** o ejecutar en una máquina con
Windows SDK/Graphics Tools para obtener el mensaje exacto de validación.

No tiene sentido seguir cambiando parámetros del PSO a ciegas.

### 7.13 El FIFO del RSX se pierde en datos de vértices

El render D3D12 ya es válido, pero el walker de comandos GCM se detiene:

```
[cellGcmSys] FIFO JUMP to unmapped IO 0x1F800000 (word 0x3F800000)
[FIFOSTEP] io=010A12C8 w=000417CC method=0x17CC count=1
[FIFOSTEP] io=01490000 w=BF800000 type=5 method=0x0000 count=2016
[FIFOSTEP] io=01490004 w=3F800000 type=1 ... JUMP 0x1F800000
```

`0x3F800000` es el float `1.0f`. El walker ha saltado a `IO 0x01490000` y está
leyendo un buffer de vértices como si fuera un command buffer. El comando previo
es el método `0x17CC` con `count=1`; el siguiente word se interpreta como
CALL/JUMP a `0x01490000`.

Conclusión: el bloqueo actual no es D3D12 ni el PSO, sino la **decodificación
del stream GCM**: un método se cuenta mal o el mapeo IO→EA de la sub-lista no es
el correcto, y el walker aterriza en datos de vértices.

Siguiente paso: comparar el decode del método `0x17CC` y el mapeo de
`0x01490000` con RPCS3, y corregir el conteo/mapeo en `cellGcmSys.c`. Sin eso, el
juego no puede emitir la escena completa.

### 7.10 Causa raíz: scanout 1920x1080 sobre backbuffer 1280x720

El PSO nunca fue el problema. Instrumentando `GetDeviceRemovedReason` tras
`ExecuteCommandLists` se vio que el dispositivo se retiraba al ejecutar el
command list, no al crear el PSO.

`scanout_auto_present` copiaba el framebuffer GCM de **1920×1080** en el
backbuffer de **1280×720** con `CopyTextureRegion` y un `D3D12_BOX`/footprint de
1920×1080. La copia se salía del destino → `DXGI_ERROR_INVALID_CALL` →
dispositivo retirado. Los PSO posteriores solo veían el dispositivo ya perdido.

Corrección aplicada:

- se recorta `cw = min(gw, backbuffer.width)`, `ch = min(gh, backbuffer.height)`;
- el footprint y el `D3D12_BOX` usan `cw×ch`;
- `guest_fb_present` conserva su copia completa original.

Resultado verificado:

- ya **no** aparece `[DEV] ... removed`;
- `[FP] guest FP pipeline ready (fp=0x043E4C81)` y `(fp=0x043E2F01)`;
- el juego sigue presentando frames sin retirar el dispositivo.

El backbuffer capturado sigue siendo negro uniforme (`frame_*.bmp`, un solo
color), así que el paso pendiente es el **compositing del offscreen RT** o
determinar si el juego está todavía en una pantalla de carga negra. `RTT_DUMP`
no generó `rt_*.bmp`; hay que revisar `s_composite_src`/`draw_count` o esperar a
que el juego alcance el menú.

### 7.11 El negro es el vídeo de título

Tras corregir el scanout, el juego carga los módulos:

```
[cellSysmodule] LoadModule(VDEC)
[cellSysmodule] LoadModule(DMUX)
[cellSysmodule] LoadModule(PAMF)
```

y se queda en pantalla negra. La explicación más probable es que esté esperando
el **vídeo de título** (`title.pam`, MPEG-2 1280×720, 30 s), no el render 3D.
`cellPamf`/`cellVdec`/`cellDmux` son parciales en ps3recomp y no hay mensajes de
apertura/decodificación.

Siguiente paso: implementar la reproducción PAMF/MPEG-2 (FFmpeg) o completar
`cellPamf`/`cellVdec` para que el juego pase del vídeo al menú. El render 3D ya
tiene PSOs válidos y el dispositivo sobrevive.

### 7.12 Sin RT de escena que componer

`MR7_FORCE_COMPOSITE=1` fuerza `s_composite_src=2` antes de
`composite_present`, pero no se imprime ningún `[COMPOSITE]`: no hay un
offscreen RT de escena válido (`s_last_drawn_slot < 0` o sin recurso). El
backbuffer sigue negro.

Por tanto, el negro actual no es un fallo de compositing de una escena ya
renderizada: el juego todavía no ha producido el frame del menú/título. Está en
carga o esperando el vídeo. Los siguientes pasos son:

1. confirmar con trazas de llamadas si el hilo principal está en el bucle de
   carga o esperando `cellPamf`/`cellVdec`;
2. implementar el vídeo PAMF/MPEG-2 (FFmpeg) o simular su finalización para
   comprobar si el juego entra al menú 3D;
3. solo entonces depurar el contenido visual del menú.

### 7.1 Segundo arranque: NIDs de cellFont registrados

Se implementaron y registraron:

- `cellFontGetFontIdCode`
- `cellFontGetCharGlyphMetrics`
- `cellFontRenderSurfaceInit`
- `cellFontRenderSurfaceSetScissor`
- `cellFontEndLibrary`
- `cellFontInitLibraryFreeTypeWithRevision`

La tabla HLE pasó de 1.238 a **1.244 handlers**. El arranque ya no muestra
`unresolved NID` para esas funciones y aparece `[cellFontFT] Init()`.

Pero el hilo `TTFontRender Thread` sigue leyendo NULL. La instrucción exacta es
`0x2D1584: lwz r9, 0x0(r11)` seguida de `lwz r0,0x20(r9)`, `lwz r5,0x1C(r9)`,
`lwz r3,0x0(r9)`; `r11` es un global del juego (`0x4E6ACC`) y su contenido es 0.
Devolver una EA de objeto cero en `*lib` no lo corrige: el global no se
inicializa con ese valor en el orden en que el hilo lo lee.

Conclusión: no basta con stubs de retorno; hace falta implementar el **objeto
`CellFontLibrary`/renderer en memoria invitada** y su orden de inicialización
(quién escribe `0x4E6ACC` y cuándo). Es la siguiente tarea de HLE, no un fallo
del lifter.

### 7.14 Geometría real del FIFO de MR7 y fin del desvío del walker

La traza con `GCM_PUTDBG=1`/`GCM_CTXDBG=1` da la topología exacta, que no era la
supuesta:

```
[cellGcmSys] Init(cmdSize=0x10000, ioSize=0x3100000, ioAddr=0x11100000)
[CTX] gCellGcmCurrentContext -> 0x0F800000  begin=11100000 end=1110FFFC
        current=11100010 callback=0F800010
[PUTDBG] put=00000010 | ctx begin=11100000 cur=11100010 io_begin=00000000
```

Es decir:

- el anillo de GCM es de **64 KB**, EA `0x11100000..0x1110FFFC`, que mapea a
  **IO `0x00000000..0x0000FFFC`** (no a `0x00CB0000` como parecía);
- `put`/`get` son offsets IO relativos a ese anillo, y coinciden con el walker;
- el juego enlaza además **dos push-buffers propios**:
  - `IO 0x00CB0000` (destino de un `CALL` en `0x00CA8180` y de un `JUMP`
    desde el final del segundo buffer),
  - `IO 0x010A0000` (segundo buffer de dibujo, donde el `put` se posiciona
    cuando el juego lo está llenando).

El fallo: el *fast-skip* de NOPs recorría megabytes de ceros de las colas no
escritas de esos buffers, se salía de todo mapeo conocido y aterrizaba en
`IO 0x01490000`, un buffer de vértices donde el float `1.0f` (`0x3F800000`) se
decodifica como `JUMP 0x1F800000`:

```
[cellGcmSys] FIFO JUMP-GARBAGE to unmapped IO 0x1F800000 -- not taken
[FIFOSTEP] io=01490000 w=BF800000 type=5
[FIFOSTEP] io=01490004 w=3F800000 type=1
```

Correcciones aplicadas en `libs/video/cellGcmSys.c`:

1. `gcm_ring_io()` obtiene los extremos IO del anillo del contexto real.
2. El salto de NOPs se recorta al final del anillo y, si lo alcanza, **envuelve
   a `begin`** como haría el `JUMP` del SDK.
3. Cualquier camino que deje `get` una palabra más allá del final del anillo
   envuelve a `begin`.
4. Una racha de NOPs **mayor de 64 KB** (o que llena la página IO) ya no se
   camina: se resincroniza al `put` vivo con el motivo `zero-page`.

Resultado verificado (15 s, `CELLMARK_DUMP=1`):

```
frames: 24 | draws: 12 | JUMP-GARBAGE: 0 | device removed: 0
resync: 3 x zero-page
```

Ya no hay ni un solo salto a datos de vértices ni retirada de dispositivo.

### 7.15 Diálogo de carga localizado

MR7 (build PAL, textos en español) abre
`cellMsgDialog` con *«¿Cancelar carga de datos?»* (`type=0x110`: SÍ/NO con
barra de progreso). La heurística `msg_offers_to_abort()` sólo conocía palabras
inglesas, así que caía en el valor por defecto **YES** y cancelaba la carga.

Se añadieron formas localizadas (`cancelar`, `annuler`, `abbrechen`, `annulla`…)
en `libs/system/cellMsgDialog.c`. Con `MSGDIALOG_ANSWER=auto` el diálogo se
responde NO.

**Importante:** en MR7 la respuesta que deja avanzar es **YES** (NO reintenta
`cellSaveData`: bucle `AutoLoad → NODATA → diálogo`). Por eso el valor por
defecto sigue siendo YES; el cambio sólo hace correcta la heurística
documentada, no cambia el comportamiento de arranque.

### 7.16 Estado del bloqueo actual (medido, no supuesto)

Con el render sano y el FIFO acotado, el juego sigue en negro. Lo medido:

- `PERF=1`: el render va a **65–153 fps**; el 90–98 % del tiempo es código
  invitado, no el backend.
- Las lecturas de `MR7.DAT` (fd 3, desde `/dev_bdvd/PS3_GAME/USRDIR/MR7.DAT`)
  avanzan hasta **41 273 344 bytes** y ahí se detienen de forma estable.
- `auto_load` (tid 12) repite `cellSaveData AutoLoad` → el callback del juego
  devuelve `-4` (`CBRESULT_ERR_NODATA`) → HLE devuelve
  `CELL_SAVEDATA_ERROR_NODATA` → diálogo de cancelar. Con YES continúa.
- La imagen SPU `bnusCore` arranca y sólo registra 8 `drain-resume` en
  `pc=0x0E8E0`; después queda inactiva (esperando un evento/DMA).
- No llega **ninguna** llamada `cellPamf`/`cellVdec`/`cellDmux` a HLE: MR7
  enlaza esas librerías de forma estática, así que su decodificador de vídeo es
  código invitado y depende de las tareas SPU.
- `cellAudio` está iniciado (`PortOpen(nChannel=8)`, `PortStart`) y
  `SetVBlankHandler(opd=0x004D00C0)` registrado.

Conclusión honesta: el pipeline gráfico ya es correcto y sobrevive; el negro
restante está en la **fase de carga del juego** (lecturas detenidas + hilo
`auto_load` reintentando + servidor bnusCore inactivo), no en D3D12 ni en el
PSO. No hay todavía frame visible ni menú.

### 7.17 Causa concreta del SPU inactivo: buzón normal sin entrega

La imagen SPU de `bnusCore` (76832 B, `image_id=280400854`, embebida en
`EBOOT.elf` en el offset 6384000) tiene su punto de entrada en `0x90`. La
función `0x11560` del código liftado es un **protocolo de buzón**:

```c
spu_wrch(ctx, SPU_WrOutMbox, ctx->gpr[3]);           /* petición al PPU */
stop 0x110
ctx->gpr[2] = spu_rdch(ctx, SPU_RdInMbox);           /* espera la respuesta */
```

Medido con `SPU_DBG_MBOX=1`:

```
[spu-outmbox] spu=2000 wrote 0x6E754153 depth=1
[spu-outmbox] spu=2000 wrote 0x6E754153 depth=2
...                            (la profundidad crece sin límite)
```

`0x6E754153` es ASCII `"nuSA"` (la firma de Namco Universal Sound
Architecture), la misma que el hilo recibe en `r3` al arrancar. El SPU la
escribe una y otra vez y se queda bloqueado en `RdInMbox`.

Causa en el runtime: en `runtime/spu/spu_channels.c`, `SPU_WrOutMbox` escribe
el buzón pero **no llama a `g_spu_out_mbox_hook`**; sólo lo hace
`SPU_WrOutIntrMbox`. Para un hilo SPU liftado no hay además backpressure (el
límite de profundidad sólo se aplica a SPUs raw), así que el buzón crece sin
que nadie lo lea y el PPU nunca ve la petición.

Sonda aplicada (`SPU_OUTMBOX_HOOK=1`, **opt-in, no es una corrección**): entregar
también el buzón normal al evento enlazado. Con ella:

```
[SPU->PPU] mbox deliver spu=0x2000 intr=0 val=0x6E754153 -> q=4
```

pero el juego sigue parado: el valor no lleva puerto (`0x6E`) y cae en el
`connected_queue` (q=4), mientras el hilo que espera es `bnusCoreUpdateThread`
sobre `q=7`. Es decir, la semántica del buzón normal (poll del PPU, puerto o
respuesta por `InMbox`) hay que resolverla contra el protocolo del juego, no
entregándola a ciegas. La sonda queda desactivada por defecto y documentada
como tal.

### 7.18 Quién consume el evento de q=4 y por qué no basta

Con `PS3_EVT_RECV=all` se ve el reparto real de colas:

```
[evt] port_create id=3 -> q=4   [evt] port_create id=4 -> q=5   [evt] port_create id=6 -> q=7
[evt] thread_connect_event tid=0x2000 queue=0x4 et=1 spup=3
[evt] thread_connect_event tid=0x2000 queue=0x5 et=1 spup=1
[evt] thread_connect_event tid=0x2000 queue=0x7 et=1 spup=7
[RECV] q=4 source=0x200000000001 data1=0x6E754153   (tid=8, decoder)
[RECV] q=3 source=0x6E754149 data1=0                 (tid=10, audio)
```

- `q=4` lo consume `bnusCoreDecoderServerThread` (tid 8), que se queda
  bloqueado ahí **desde el arranque**.
- `q=3` recibe `source=0x6E754149` = clave de notificación de `cellAudio`
  (`SetNotifyEventQueue(key=0x6E754149)`), o sea "nuAI"; `0x6E754153` ("nuSA")
  es el valor que el SPU escribe en su buzón.
- Al entregar "nuSA" a q=4 con la sonda, el hilo lo recibe pero **no progresa**,
  y la cola se llena (127 eventos): el SPU escribe mucho más deprisa de lo que
  el hilo consume, porque el SPU está en un bucle de reintento (el
  `drain-resume` lo reentra en `0x11560` y repite `wrch`).

Además, el juego **no hace ni una sola llamada** `sys_spu_thread_write_spu_mb`
ni `sys_spu_thread_group_join` después de `group_start` (verificado con
`PS3_HLE_TRACE=1`): no puede responder al SPU por sí mismo. Por tanto la
respuesta al `stop 0x110` tiene que venir del runtime, y hoy sólo la implementa
la ruta del **intérprete** (`spu_interp.c:518`, con un `g_spu_pending_evt`
global pensado para otro título), no la ruta **liftada** que usa MR7
(`spu_lifted_thread.c` sólo reconoce 0x102/0x101 como salida).

Siguiente paso técnico, ya acotado: portar ese servicio a la ruta liftada,
resolviendo el destino del `receive_event` para MR7 (el valor `0x6E754153` no
es un `spuqueue` EA ni lleva puerto) contra el código PPU del hilo decoder
(`OPD 0x004E4100 → EA 0x0039D900`). Es un trabajo de RE del protocolo bnusCore,
no un fallo del backend ni del lifter.

### 7.19 `stop 0x110` servido en la ruta liftada (y qué queda)

La clave `0x6E754153` **no es una cola existente**. Con las trazas de
`sys_event_queue_create`:

```
queue_create id=3 key=0x6E754149   ("nuAI", la clave que usa cellAudio)
queue_create id=1 key=0x0 | id=2 key=0x0 | id=4 key=0x0 | id=5 key=0x1 | id=6,7 key=0x0
```

ninguna cola tiene `0x6E754153`. En hardware lv2 responde a
`sys_spu_thread_receive_event` con una clave inexistente devolviendo
`CELL_ESRCH`, y el SPU sigue por su rama de error (el propio código en `0x11560`
compara el primer valor de `InMbox` con 0).

Implementado en `spu_channels.c` (`spu_stop`) + `sys_event_queue_id_by_key()` en
`sys_event.c`:

- al `stop 0x110`, se consume la clave del buzón de salida;
- se busca la cola por clave;
- si no existe, se responde `{CELL_ESRCH, 0, 0, 0}` por `rcv_evt` y se devuelve
  el estado a `RUNNING`.

Resultado verificado (20 s):

```
antes:  24+ escrituras OutMbox, profundidad 1..15+ creciendo sin límite
ahora:  1 escritura, 1 CELL_ESRCH, y el SPU continúa ejecutando
        ([spu-smc] pc=0x08E78, 0x0783C, 0x060A8 …)
```

El caso "cola encontrada" no se ha implementado a propósito: entregar los cuatro
valores de un `receive_event` real sin conocer el protocolo del título sería
fabricar estado.

### 7.20 Perfilado y contenido real del frame

Se añadió `ps3_sampler_start()` al port (antes sólo lo llamaba el arnés genérico,
así que `PS3_SAMPLE` no hacía nada aquí). Con `PS3_SAMPLE=4000`:

```
[samp] map: 14834 PPU + 10272 SPU entries
[samp] 36 samples, 2 in guest code (5.6%)
[samp] host 94.4% ntdll.dll        <- todos los hilos están bloqueados, no girando
[samp] guest: func_0027A0CC / func_000378F0 / spu_LS_00006188 / spu_LS_00004FD0
```

Mapa de esperas con `SEMTID=1`:

| Hilo | Espera | LR |
|---|---|---|
| tid 1 (principal) | `sem=3`, `sem=6`, `sem=7` en ciclo | `0x1D1684` (poll de pad), `0xBE110`/`0xBE348` (sync de frame) |
| tid 4 (Draw) | `sem=5` | — |
| tid 6 (File) | `sem=8` (ocioso) | `0x14C6C` |

El hilo principal **no está atascado**: cicla pad → draw-sync → pad, y el
`File Thread` está ocioso (la lectura del DAT terminó). Con `RSX_CAP=1:2:20` el
contenido real del frame es:

```
frame draws=1  backbuffer=1280x720   op DRAW rt=0x00000000 vp=720x480 n=3 t0..t3=0
     offrt[1] off=0x00010000 1280x720 dxgi=28  (CLEAR a negro)
     offrt[0] off=0x01980000 256x64  dxgi=41    (todo negro)
```

Es decir: **un triángulo de 3 vértices al backbuffer por frame, sin ninguna
textura enlazada nunca** (`bind_texture` sólo aparece una vez, con
`offset=0x0, fmt=0x00`, para las 10 unidades). El juego pinta un fondo negro
sólido, no una escena.

### 7.21 El diálogo de carga: YES es la respuesta correcta

Con `SAVEDATA_NODATA_OK=1` (el callback del juego dice "no hay datos" y HLE lo
reporta como carga vacía completada), el ciclo desaparece: **1 `AutoLoad` y 1
diálogo en 20 s** (antes se repetía). Si además se responde NO al diálogo, se
disparan **120 diálogos en 20 s**: el juego reintenta en el acto. Por eso el
valor por defecto (YES) es el correcto y la sonda queda opt-in.

### 7.22 Estado real tras estas correcciones

- Build limpio; `garbage=0`, `device removed=0`, `unlifted=0`, `resync=3`.
- SPU `bnusCore` ejecutando sin inundar el buzón; un solo grupo SPU creado.
- `AutoLoad`/diálogo: una sola vez.
- Frame: 1 dibujo/frame, negro, sin texturas.
- Vídeo sigue configurado a **720×480** (`resId=4`) y no hay más grupos SPU: el
  juego está aún en **inicialización temprana**, no en un menú.
- Sin frame visible ni menú todavía.

### 7.23 El "texto" del frame: son sondas con `code=0`, no glifos

`PS3_HLE_TOP` señalaba 515 llamadas/s a `cellFontGetFontIdCode`,
`cellFontGetRenderCharGlyphMetrics` y `cellFontRenderCharGlyphImage` desde el
hilo de fuentes (tid 3). Instrumentando la superficie de render, **todas** las
llamadas llegan así:

```
glyph #1    code=0 surface=0x0 pitch=0 pix=0 buf=0000000000000000 pen=(0.0,0.0)
glyph #1000 code=0 surface=0x0 ...   glyph #7000 code=0 ...
```

No hay ni un solo glifo real ni una superficie válida. Leyendo el bucle del hilo
(`func_002D14C0` → `func_002D0868` → `func_00424F10`), `0x2D0868` es un
**decodificador UTF-8** que escribe un punto de código en `*r4`; el hilo limpia
la textura de texto, decodifica un carácter y lo renderiza. Recibe una cadena
vacía, así que dibuja NUL 16 veces por frame (~515/s a 31 fps) y borra la textura
cada vez. No es un bloqueo: es un trabajador procesando una cadena vacía.

También se confirmó que `cellFontGetFontIdCode` devolvía `0` en **ambas**
salidas para cualquier código, lo que colapsa una cadena entera sobre el glifo 0.
Ahora, **sólo si** hay fuente sustituta cargada, devuelve `fontId=1` y
`fontCode=code` (mapeo identidad); sin ella mantiene el 0/0 histórico para no
decirle a otros títulos que todo punto de código existe.

`cellFontFT` resultó ser un stub explícito ("Glyph rendering produces empty
bitmaps") y `cellFontOpenFontset` depende de la fuente de sistema del firmware
(que no se puede distribuir). Para poder validar el camino de texto sin ella se
añadió `FONT_TTF=<ruta>`: una **fuente sustituta opt-in** rasterizada con
stb_truetype, documentada como bring-up y nunca empaquetada. Con
`FONT_TTF=C:/Windows/Fonts/segoeui.ttf` la fuente carga
(`substitute loaded: ... 960076 bytes`), pero el frame sigue negro porque el
juego no ha pedido todavía ningún glifo real.

Barrido final del estado: `auto_load` y `sample_game_data` han **terminado**;
audio iniciado (`PortStart`, `GetPortConfig READY`); ninguna llamada HLE sin
implementar en el bucle. El juego está estable a ~31 fps pintando negro.

### 7.24 El camino de presentación funciona; falta el contenido

Con `RSX_TEXDBG=1` se ve que el juego **sí** enlaza una textura real, y es el RT
de 1280×720:

```
[TEXBIND] unit=0 off=0x00010000 fmt=0xA5 4x4     (11 binds)
```

Es decir: el juego dibuja un triángulo de pantalla completa muestreando su
propio render target, que previamente ha limpiado a negro. La cadena
RT → textura → triángulo → backbuffer **funciona**; lo que falta es que alguien
escriba en el RT.

Los assets de la pantalla de carga están **dentro** de los 41 MB leídos:

| Asset | Offset DAT | ¿Leído? |
|---|---|---|
| `Obj_StartMenu` | 16.357.376 | sí |
| `Tex_StartMenu` | 16.418.816 | sí |
| `Obj_Loading` | 20.203.520 | sí |
| `Tex_Loading` | 20.303.872 | sí |
| `Obj_CommonTitle` | 35.239.936 | sí |
| `Tex_ComTitle` | 66.732.032 | **no** |

Ninguno de los dos primeros se ha enlazado todavía como textura, así que no se
han dibujado.

Se trazó por fin el hilo de inicialización: el thunk `0x00037F88` llama por
vtable a `0x0003833C` con `r3=0x142FE9C0`, que es un **worker de carga**:

```
lock   (syscall 0x66 = 102, sys_mutex_lock)
  ¿hay trabajo en el buffer A?
unlock (syscall 0x68 = 104, sys_mutex_unlock)
procesar (0x34B8C / 0x249D8), alternando dos buffers
```

Está **ocioso**: no hay trabajos encolados. El `File Thread` también está ocioso
(`sem=8`) y todas las lecturas devolvieron la longitud pedida (no hay lecturas
cortas). `cellFs` no tiene AIO en este runtime y no queda ningún NID de fichero
sin resolver, así que las lecturas son síncronas y correctas.

Conclusión: el productor del arranque (la máquina de estados del hilo principal)
deja de encolar cargas tras 41 MB, antes de pedir `Tex_ComTitle`.

### 7.25 El loader espera en una condición; el productor está identificado

El worker de carga no espera en un semáforo (por eso no aparecía en `[WAIT]`):
espera en **`sys_cond_wait`**. Desensamblando su pequeño runtime se identifican
las primitivas por número de syscall:

| Función | Syscall | Primitiva |
|---|---|---|
| `func_000382A8` | 0x66 (102) | `sys_mutex_lock` |
| `func_00038170` | 0x68 (104) | `sys_mutex_unlock` |
| `func_00038148` | 0x6B (107) | **`sys_cond_wait`** |
| `func_00038264` | 0x6C (108) | `sys_cond_signal` |
| `func_000381F4` | 0x69 (105) | `sys_cond_create` |

El **productor** es `func_0003632C(item)`:

```
[this+0] = item
mutex_lock(A)
  if ([this+4] < 0) cond_wait(this+0x38)
  [this+4] = 1
  cond_signal(this+0x18)
mutex_unlock(A)
```

y el encolado real pasa por el resolutor de rutas `func_0027A0CC` (el que el
muestreador veía activo), que traduce nombre de asset → entrada del TOC y llama
al push.

Se auditó la implementación de `sys_cond_*` del runtime: el waiter se inserta en
la lista **con el mutex tomado**, `signal` marca uno y hace broadcast, y el bucle
del waiter comprueba `signalled`. No hay ventana de wakeup perdido.

También se descartó que la pantalla espere input: con `PAD_AUTOPRESS=8000`
pulsando CROSS durante 25 s no cambia nada (misma lectura de 41 MB, 12 draws,
un diálogo, vídeo en 720×480).

Siguiente paso concreto: poner un **watchpoint de escritura** (`PS3_WWATCH`) en
la variable `[this+4]` / `[this+0]` del loader para ver si el productor llega a
escribirla y desde qué función; y con `PS3_CALLTRACE_LR=0x36398` observar en
ejecución quién llama al push.

### 7.26 Mapa de hilos por `lr` y estado real del cargador

Se añadió `MR7_LRS=1` al port: cada 5 s imprime el `lr` invitado de **todos** los
hilos (el watchdog equivalente sólo existía en el arnés genérico). Mapa medido en
el atasco:

| tid | Hilo | `lr` | Qué hace |
|---|---|---|---|
| 1 | principal | 0x0BE080 / 0x0BE348 | `SetWaitFlip` + sync de frame |
| 3 | TTFontRender | 0x2D1644 | sondas de glifo (`code=0`) |
| 4 | Draw | 0x0BDA9C | construcción de listas / `AddressToOffset` |
| 6 | File | 0x014C6C | espera trabajo (`sem=8`) |
| 10 | bnusCoreUpdate | 0x397274 | poll de `q=7` cada 5,3 ms |
| 11 | Initialize | 0x36828 / 0x38310 | **alterna** entre el bucle del loader y el resolutor de rutas |
| 12 | auto_load | 0x19BB4 | terminando |

Conclusión importante: el loader **no está bloqueado**. Alterna entre su bucle de
trabajo (`0x38310`) y el procesado de peticiones (`0x36828` → `func_0027A0CC`,
el resolutor de rutas que el muestreador veía activo). Lo que no hay son
**peticiones que generen lecturas de disco**: el `File Thread` sigue ocioso y el
contador del DAT no pasa de 41 MB.

Es decir, el atasco no está en el cargador ni en sus condiciones: está en quien
decide *qué* cargar.

Siguiente paso: registrar el contenido de las peticiones que el loader ya
procesa (los ítems de la cola en `[queue+0x40]`) y contrastarlo con lo que falta
(`Tex_ComTitle`, 66 MB), para identificar la petición que nunca llega.

### 7.27 Corrección importante: la lectura no paraba en 41 MB

El log `[fs] read` imprime **las 50 primeras lecturas** y después sólo la primera
de cada descriptor (más un resumen cada 2000). Eso hacía que 41.273.344 B
pareciera el final de la carga cuando en realidad era **la última línea impresa**.

Con `FS_READ_ALL=1` (todas las lecturas) el total real avanza hasta
**45.955.072 B** y ahí sí se detiene de forma estable (verificado de 60 a 120 s,
y en una ejecución aparte de 40 s con `FS_SHORTLOG=1`):

```
reads = 69, last = total 45.955.072, EBADF = 0, lecturas cortas = 0
```

El punto de parada cae **20.480 B antes del final de `Obj_RBTop`**
(45.733.888–45.975.552) y el siguiente asset es `Obj_OnlineRoomSearch`
(45.973.504), seguido de toda la familia `Obj_Online*`.

Datos adicionales de esta ronda:

- ninguna lectura falla (`EBADF = 0`) ni vuelve corta (`SHORT = 0`);
- no hay lecturas AIO (`cellFsAioRead` nunca se llama);
- el `File Thread` despacha ~9 trabajos/s por vtable, alternando **dos tipos**:
  `0x139C0` (lector por trozos de 2 MB, `cellFsRead` → `0x1397C`) y `0x140C8`;
- el lector no llega a emitir lecturas, así que esos trabajos o bien tienen
  `remaining = 0` (transferencia ya completa) o no llegan a la llamada.

Conclusión: la carga **no está parada por E/S**; termina `Obj_RBTop` y no pide el
siguiente asset. El siguiente paso es seguir el ciclo de vida de esos dos tipos
de trabajo (¿se retiran de la cola al completarse?) y el contador de la cola del
loader.

### 7.28 Descartado: motores 2D, callback de NP, input

- **Motores 2D**: con `GCM2D_TRACE=1` no hay **ni una** llamada a
  `gcm_2d_method` (subcanales 2–6 = 0). La UI no se dibuja por blits 2D; todo va
  por el motor 3D y el juego sólo emite el triángulo de blit.
- **Callback de `sceNp`**: `sceNpManagerRegisterCallback` sólo notificaba el
  callback cuando había servidor psnr (ONLINE); en OFFLINE no se invocaba nunca.
  Ahora se notifica el estado **actual** (OFFLINE) en el siguiente
  `cellSysutilCheckCallback`, que es lo que hace hardware al asentarse el
  manager. Verificado: MR7 **no registra** ese callback (sólo llama a
  `sceNpManagerGetStatus`), así que este cambio es inerte para el juego — pero es
  más fiel para otros títulos.
- **Input**: ni `PAD_AUTOPRESS` (CROSS periódico) ni `YDKJ_INJECT_PAD`
  (START+CROSS a los 8 s, inyectado en `CellPadData`) cambian nada: no es una
  pantalla que espere un botón.
- El `File Thread` alterna dos tipos de trabajo (`0x139C0` lector y `0x140C8`)
  con un recuento coherente con el de lecturas en la misma ventana, así que la
  tubería de lectura funciona; no hay livelock de trabajos.

Estado: el juego está estable a ~31 fps, sin dibujar UI, con el frontend
precargado hasta `Obj_RBTop` y sin pedir los assets `Obj_Online*`.

### 7.29 El `put` se congela: el juego deja de construir el frame

Con `GCM_DRAINDBG=1` se ve el valor de `put` a lo largo de toda la ejecución:

```
0x10 -> 0x80 -> 0x194 -> 0x010A0DD4 -> 0x00CB0D54 -> 0x010A0D54 -> 0x00CB12F0 (se queda)
```

`put` deja de avanzar y el walker queda *caught-up* (0 bytes por pase) para
siempre. Es decir: **el hilo de dibujo dejó de construir el frame a medias**.

Comprobaciones de esta ronda:

- `RECYCLE_DBG=1`: el callback de "buffer lleno" del contexto **nunca se invoca**
  (0 llamadas), así que no es él quien devuelve los punteros;
- `MR7_LRS=1` + in-flight de HLE: el hilo principal está dentro de
  `cellGcmSetWaitFlip` en las muestras (normal, es su espera de frame), y los
  hilos 7 y 12 tienen el marcador de `sys_ppu_thread_exit` colgado (cosmético:
  ya habían terminado).

**Corrección de interpretación:** los `put` del juego (`0x00CB…`, `0x010A…`) no
viven en el anillo de 64 KB del contexto GCM (IO `0x0000..0xFFFC`), sino en sus
propios buffers enlazados. Por eso "put > fin del anillo" es **normal** y no un
desbordamiento. Una sonda `RING_OVERRUN_FIX=1` que rebobina al inicio produce
cambios de comportamiento (32 dibujos frente a 12) pero **no es una corrección**:
reencauza al anillo equivocado. Queda como experimento, desactivada.

Siguiente paso: averiguar por qué la construcción del frame se detiene a medias;
el candidato es un recurso que el frame necesita y que nunca se carga (los
`Obj_Online*` y `Tex_ComTitle` siguen sin pedirse).

### 7.30 Mediciones de aceleración: ambos punteros congelados

Con `cellGcmDebugHeads()` (nueva, expuesta al runner) se leen a la vez el `put`
del RSX y el `current/begin/end` del contexto:

```
[LRS] gcm put=0x00000010 current=0x11100010 begin=0x11100000 end=0x1110FFFC
[LRS] gcm put=0x00CB12F0 current=0x111001A4 begin=0x11100000 end=0x1110FFFC
[LRS] gcm put=0x00CB12F0 current=0x111001A4 ...   (se repite durante minutos)
```

**Los dos punteros están congelados**: no es que el walker no consuma, es que el
invitado dejó de escribir comandos. El `current` se quedó a 0x1A4 bytes del
inicio del anillo, así que la parte PPU tampoco reserva.

Dónde está el hilo de dibujo: `func_000BDA04` (bucle del Draw Thread) →
`func_000C5680` (constructor de frame) → `func_000E5998` (montaje de sprites:
dos `cellGcmAddressToOffset` por sprite y relleno de un quad de floats). Hace
~188 traducciones/s y **nunca llega a reservar/flush**.

`cellGcmAddressToOffset` no falla nunca: no aparece `REFUSED` ni "no free IO
page" en toda la ejecución.

Ejecución de **5 minutos**: estado idéntico (69 lecturas, 45.955.072 B, 12
dibujos, 1 diálogo). Ningún timeout interno del juego lo desbloquea.

### 7.31 Referencia con RPCS3: bloqueada por firmware

RPCS3 0.0.43 está instalado con `dev_flash`, y reconoce el disco:

```
Title: "MAX RACER 7 [BCES00009]"
Reason: Firmware is missing   (emulation is stopping)
```

Arranca, enumera la RTX 5070 y empieza a compilar shaders, pero aborta por
firmware incompleto. Sin el firmware del usuario (que no se puede distribuir) no
sirve como referencia; queda anotado como vía disponible si el usuario lo
instala.

### 7.32 Referencia operativa: MR7 llega a la pantalla de título

El usuario aportó su `PS3UPDAT.PUP` (196,6 MB, `SCEUF` v1, SHA-256
`99C04429…C9D464`). Se instaló **solo en su RPCS3 local** (no se copia al
proyecto ni se distribuye):

```
Successfully installed PS3 firmware and LLE Modules!
dev_flash: bdplayer data ps1emu ps2emu pspemu sys vsh
Firmware version: 4.92
```

Con el firmware, `rpcs3 --no-gui G:\` arranca el disco (ISO archive,
`PS3_GAME/USRDIR/EBOOT.BIN`) y **el juego llega a la pantalla de título real**:

- captura local: `docs/referencia_titulo_rpcs3.png` (solo referencia interna,
  del juego del usuario);
- se ve el Skyline GT-R, el logo **MAX RACER 7** y el texto *“Welcome to
  Ridge State, the home of drift racing! …”*, a **74 fps**.

Medición: RPCS3 lee **77 MB** y deja de leer al llegar al título (25 s). Nuestro
port se queda en **46 MB**.

Esto define por primera vez el **objetivo exacto** y da una vía de comparación
llamada a llamada.

### 7.33 El juego no escribe `put`: lo publica por `sys_rsx`

Con `PPU_WWATCH` sobre el bloque de control (`0x0F802000`) no aparece **ninguna**
escritura del invitado, y `SYS_RSXDBG=1` muestra que `cellGcm_syscall_set_fifo`
(el `sys_rsx_context_attribute(0x001)`) **nunca se llama**.

Es decir: el `put` que ve nuestro walker lo escribe **nuestro propio runtime**
(`gcm_fifo_next_segment`/`gcm_fifo_recycle`) derivándolo de `ctx->current`; el
juego sólo mueve `current` (visto con el watch: `guest-fn=0x003AC6F4`, tid 1,
incrementos de 8 bytes).

Consecuencia: cuando `current` se congela, `put` se congela con él. Y `current`
se congela en `0x111001A4` mientras el Draw Thread sigue traduciendo direcciones
(200/s) sin llegar a reservar.

Comprobaciones adicionales de esta ronda:

- `PPU_WWATCH` sobre el bloque de control (`0x0F802000`) y sobre la página del
  dispositivo RSX (`0x0F830000`, 4 KB): **cero** escrituras del invitado (la
  única escritura observada a `0x0F800008` la hizo el hilo principal al arrancar,
  seis veces, avanzando `current` de 8 en 8 bytes);
- `cellGcmGetControlRegister` sí devuelve la EA invitada correcta
  (`0x0F802000`), así que no es el bug de puntero host;
- `SYS_RSXDBG=1` confirma que `sys_rsx_context_attribute(0x001)` **no se llama**.

Es decir: el juego nunca publica el `put` por ninguna de las vías que modelamos;
el `put` que ve el walker es una **derivación** nuestra de `ctx->current`.

Siguiente paso concreto: con la referencia de RPCS3 ya operativa, activar en su
config los canales TTY de `sys_rsx`/`cellGcmSys` y ver **por dónde publica el
`put`** MR7 en hardware; replicarlo en el HLE.

### 7.34 Corrección: el juego SÍ publica `put` (yo vigilaba mal)

`GCM_CONTROL_GUEST_ADDR` se define con `VM_HLE_INJECT_BASE`, que es una
**variable** (`ppu_hle_inject_base`, por defecto `0x20000000`), no una constante
`0x0F800000`. El bloque de control está en **`0x20002000`**; mis watchpoints
anteriores (`0x0F802000`, `0x0F830000`) miraban direcciones equivocadas.

Con el watch correcto (`PPU_WWATCH=20002000`) el juego **sí escribe `put`**, y
con el hilo principal (tid 1) desde `guest-fn=0x000BCAE4`:

```
0x20002000 <- 0x8 ... 0x194          (arranque)
0x20002000 <- 0x10A0D54 / 0xCB0D54   (alternando los DOS buffers de 64 KB)
0x20002000 <- 0xCB12F0               (última, tras 35 publicaciones)
```

Consecuencias y correcciones a lo dicho antes:

- `put` **no** lo deriva el runtime: el invitado lo publica (el runtime sólo lo
  escribe en los caminos de reciclado/segmento como respaldo);
- `0xCB12F0` **no es un desbordamiento**: está dentro del buffer de 64 KB
  (`0xCB0000..0xCC0000`); la interpretación de "overrun" era falsa y la sonda
  `RING_OVERRUN_FIX` sobra;
- `sys_rsx_context_attribute` sí está registrado y aun así el juego no lo usa:
  publica por escritura directa al bloque de control;
- `cellGcmSetWaitFlip` no puede bloquearse para siempre (espera acotada a 64 ms).

Estado: el juego publica 35 frames (alternando buffers) y **deja de publicar**,
con todos los hilos vivos. La causa de que el hilo principal deje de hacer el
*kick* sigue abierta; el siguiente objetivo es la función `0x000BCAE4` (el
publicador) y quién la llama.

### 7.35 El hilo principal espera al Draw Thread que nunca termina su frame

El publicador `func_000BCAE4` se llama desde `func_000BE0CC` (bucle de frame del
hilo principal) en `0xBE318`. El resto del bucle tiene una rama que **salta el
render** (`loc_000BE320` → `func_000BDB18` → vuelve a `loc_000BE124`), y en los
volcados `MR7_LRS` el hilo principal aparece justo ahí:

```
guest-tid 1  lr=0x000BE348   <- rama "todavía no toca renderizar"
```

El "reloj" de ese bucle es `sys_time_get_system_time` (NID `0x8461E528`). Se
comprobó con `SYS_TIME_DBG=1` que **se llama y avanza correctamente**
(1.200 llamadas en 14 s: 9,2 → 11,3 → 13,5 s). No es un reloj congelado.

Por tanto la condición que falla es la espera al **Draw Thread**: éste sigue
dentro de `func_000C5680` (constructor de frame) traduciendo direcciones
(~200/s, con direcciones que **varían**, así que no es un ciclo de las mismas)
pero **nunca completa el frame ni señaliza** al hilo principal.

Siguiente paso: localizar el límite del bucle de `func_000C5680` (probablemente
un contador o una lista cuyo tamaño depende de datos que inicializamos) y por
qué no llega a la señalización.

### 7.36 El Draw Thread está en un ciclo de espera de un recurso

Con `BCTRL_RING=1` (anillo de llamadas indirectas por hilo) el Draw Thread
muestra un patrón de **tres** destinos que se repite:

```
000E7DC0 000E7958 000E7988  (y ocasionalmente 00010B70)
```

`func_000E7DC0` es un predicado trivial del propio juego (devuelve 1);
`func_000E7958` copia un registro de nodo; `func_000E7988` hace trabajo de
matrices.

Con `PS3_CALLTRACE_TO=0xE7DC0` se ve que **no es una iteración sobre nodos
distintos**: las 284 llamadas son idénticas:

```
[CALL] -> 0x000E7DC0 r3=0x107C4780 r4=0x000000C4 ... lr=0x000E6B68
        from=func_000E6B10+0x463 tid=4
```

Mismo objeto (`0x107C4780`), mismo índice (`0xC4` = 196), mismo sitio de llamada.
Es un **bucle de espera**: el Draw Thread pregunta una y otra vez por el estado
del recurso 196 y recibe siempre 1, así que no avanza.

En `func_000E6B10` se ve la mesa de 16 entradas (`0xC0 + i*4`, con ese
`0xC0..0xFC`) que se rellena con objetos creados por `func_000EC2C8/EC548/EC878`
y se marca cada uno con un flag a 1. El recurso que falta es una de esas
entradas, y encaja con que la carga se detuviera justo antes de los `Obj_Online*`.

Siguiente paso: identificar qué entrada de esa mesa es el recurso 196, quién
escribe su flag de "listo" y por qué no se emite la petición de carga
correspondiente.

### 7.37 El bucle real está en `func_000E6CBC`, y el flush se llama por vtable

Precisión sobre el punto anterior (la resolución de "función contenedora" del
trazador usa el símbolo más cercano por debajo y puede desviarse):

- la llamada al predicado sale de **`func_000E6CBC+0x2B7`**, no de `func_000E6B10`;
- su cola es un **bucle incondicional**:

```c
loc_000E6F4C:
    func_003B92B8(gpr31);   // espera sobre un objeto (si r3 < 0)
    func_0041ED2C(gpr29);   // recorrido/render de nodos
    goto loc_000E6F4C;
```

Es el **render loop** del Draw Thread: espera y recorre. El recorrido llama al
predicado del mismo objeto (`r3=0x107C4780`) cientos de veces, sin avanzar.

Por el lado del hilo principal, el flush `func_000BCAE4` (que publica `put`) se
llama desde `func_000BE0CC` en `0xBE318`, y esa función se invoca a través de un
**thunk virtual** `func_000C474C` (un `Run()` de una clase). Es decir, el bucle
principal despacha el frame por vtable, y es ahí donde deja de llegar la llamada.

Conclusión de la ronda: el Draw Thread no está atascado en una condición del HLE
ni en un reloj; está recorriendo un nodo que no progresa, y el hilo principal dejó
de invocar el frame. Ambas cosas apuntan al mismo recurso que nunca se carga.

### 7.38 Verificación del predicado y del objeto

Para no dar por bueno un diagnóstico blando se comprobaron las dos piezas:

1. **El predicado es fiel al binario.** En `EBOOT.elf`, EA `0xE7DC0`:

```
38600001   li  r3, 1
4E800020   blr
```

Es literalmente un método virtual por defecto que devuelve 1. El lifter no lo
inventó.

2. **El objeto está a cero.** Nuevo volcado `MR7_MEMDUMP=<ea>[:len]` (validado
antes contra el bloque de control y el contexto):

```
[MEM] 20002000: 00000010 00000010 00000001 00000000   (put=0x10 get=0x10 ref=1)
[MEM] 0F800000: 11100000 1110FFFC 11100010 0F800010   (begin/end/current/cb)
[MEM] 107C4780: 00000000 … (0x100 bytes)              (el objeto del bucle)
```

El objeto sobre el que itera el Draw Thread está **vacío**, así que el recorrido
no puede producir nada útil: llama al `return 1` por defecto una y otra vez.

El mecanismo, ya descifrado (`func_000E6B10`):

```c
slot = base[0xC0 + index*4];      // mesa de 16 entradas
if (slot < 0) return slot;
vtable = [slot];
method = [vtable + 0x14];         // método virtual nº 5
return method(slot, 0xC0 + index*4);
```

Con `r4 = 0xC4` el índice es **1**, y el objeto (`0x107C4780`) **y toda la mesa
alrededor** están a cero:

```
[MEM] 107C46B0: 00000000 00000000 00000000 00000000
...                                              (0xD0 bytes, todo cero)
[MEM] 107C4780: 00000000 00000000 00000000 00000000
```

El juego recorre una **estructura vacía** y llama al método virtual por defecto.
Falta el relleno de esa mesa de 16 slots, que es justo lo que haría la rutina de
inicialización que no llega a ejecutarse.

### 7.39 El constructor del objeto nunca se llama

Buscando quién rellena la mesa se llegó al constructor del objeto que la posee:

```
func_000180F0(obj):
    func_0040BC9C(obj + 0x40,  ...)     // sub-objetos
    func_0040BC9C(obj + 0xC0,  ...)     // ← la mesa de 16 slots
    func_0040BC9C(obj + 0x140, ...)
    [obj + 0x540] = 0 ; obj + 0x548 ...
```

`func_000180F0` **no tiene ninguna llamada directa**: se invoca por puntero, y su
única referencia en el binario es el **OPD `0x4CB938`** (segmento de datos, junto
a los OPD de los hilos `File Thread` 0x4CB6F0 y `auto_load` 0x4CBA40).

Con `PS3_CALLTRACE_TO=0x180F0` se comprobó que **nunca se llama** en nuestro port
(0 llamadas en 12 s). Es decir: el objeto que posee la mesa **no se construye**, y
por eso la mesa está a cero y el Draw Thread gira en el vacío.

Encadenando con lo anterior, ese constructor es un **consumidor** del arranque:
se crearían tras completar la carga, y como la carga se detiene en 46 MB, nunca
llega a ejecutarse. La causa primaria sigue siendo la **parada de la carga**.

Con esto el diagnóstico queda cerrado con evidencia y sin especulación: falta la
**construcción/carga** de ese objeto, y por eso el frame nunca se completa.

### 7.40 Intentos de comparación con RPCS3 (resultado)

Se activaron en RPCS3 los canales TTY `sys_rsx`, `cellGcmSys`, `cellFs`,
`cellGcm`, `cellSysutil`, `cellSaveData`, `cellPamf`, `cellVdec` y `cellAudio` a
nivel *Trace* y se relanzó el juego. El log (157 KB) **no añade trazas de
`sys_rsx`/`cellFs`**: RPCS3 no registra esas llamadas en este build, así que la
comparación "llamada a llamada" por log no es viable tal cual.

Lo que sí queda disponible de la referencia: la **pantalla de título** (captura
en `docs/referencia_titulo_rpcs3.png`), el **volumen leído** (77 MB frente a
46 MB) y el hecho de que el juego es *Playable*. La comparación útil pendiente es
de **memoria** (volcar el objeto equivalente en ambos), no de log.

---

## 8. Siguientes tareas concretas

1. Averiguar qué espera el juego para salir de inicialización: el hilo principal
   sólo cicla pad (`sem=3`) y draw-sync (`sem=6/7`); el `File Thread` está
   ocioso en `sem=8`; el worker de carga (`0x3833C`) está ocioso en su mutex de
   trabajos; el vídeo sigue en 720×480 y sólo existe un grupo SPU. Hay que
   localizar la variable de estado que el productor debe escribir para encolar
   `Tex_ComTitle` (66 MB) y el resto.
2. Con `RSX_TEXDBG` y `PS3_HLE_TOP` ya se puede seguir un frame completo; usar
   `PS3_CALLTRACE_LR` para recorrer la cadena de llamadas del productor de
   carga desde `func_0003833C` hacia arriba.
3. Poner un watchpoint (`PS3_WWATCH`) en `[queue+0]`/`[queue+4]` del loader y en
   el contador de assets, para ver si el productor llega a escribir la cola y
   desde qué función se detiene.
2. Resolver el caso "cola encontrada" de `stop 0x110` (los cuatro valores del
   `receive_event`) si aparece una clave real; hoy sólo se implementa la
   negativa `CELL_ESRCH`, que es la respuesta de hardware.
3. Decidir si `SAVEDATA_NODATA_OK` debe ser el comportamiento por defecto
   (elimina el bucle `AutoLoad` de MR7) o quedarse como sonda, y justificarlo
   con más títulos.
4. Con contenido real, validar `composite_present` y `RTT_PASS`; hoy el frame es
   un único triángulo negro sin texturas.
5. Fuentes: `cellFontFT` es un stub y `OpenFontset` depende del firmware. Falta
   el modelo real del objeto `CellFontLibrary`/renderer y, si MR7 no trae un TTF,
   decidir cómo se aporta la fuente del sistema sin distribuirla.
6. Implementar el vídeo PAMF/MPEG-2 (FFmpeg) si el juego reproduce `title.pam`
   por su ruta estática.
7. Validar las 42 imágenes SPU en ejecución (SPURS vs raw).
8. Corregir las ~2.230 instrucciones VMX/op31 no liftadas.
9. Extraer y documentar modelos, texturas, circuitos y colisiones del `.DAT`.

---

## 9. Evidencias

```
ppu_loader: 14.067 funciones, 262 imports, 23 librerías
ppu_lifter: 14.834 funciones, 9 chunks, 1.507.701 líneas
spu: 42 imágenes, 1.005.116 bytes
TOC: 0x630128, 13.313 archivos, 2.139 contenedores
RRLZ: extracción real de DB y SoundConfig_Free
HLE: 189/262 cubiertos
Build: MyGameRecomp.exe 66.412.544 bytes
Boot: D3D12 + GCM 1920x1080 + cellFont; parada en cellFont
```

### 7.41 Sondas de syscalls y condiciones: el bucle no espera, y no es un fallo de cond_wait

Se instrumentaron dos contadores opt-in, sólo lectura, para no seguir deduciendo:

- `PS3_SC_STATS=1`: histograma por número de syscall cada 2 s (con reparto de
  `102` por hilo).
- `PS3_COND_STATS=1`: por condición, `waits/instant/blocked/timeout/eperm/
  esrch` y `signal/signal_all/nohit` con último `tid`+`lr`.

**Resultado (ventana de 2 s, tras 6 s de ejecución):**

`
[scstats] top: 104=3276594 102=3276595 92=1270 130=312 94=252 ...
[scstats] | mutex_lock/tid: t11=3276595
`

- El hilo que gira es **tid 11** (`Initialize`, cuerpo real `func_0003833C`),
  con ~1,63 M `mutex_lock`/`mutex_unlock` por segundo.
- **`sys_cond_wait` (107) se llamó UNA sola vez en toda la ejecución.** El bucle
  nunca llega a esperar.

**Verificado contra el binario** (no contra el código liftado, cuya dirección a
offset tenía un sesgo de 0x10000 que se corrigió):

`
00038390: lwz  r3,-0x7FE8(r30)   ; mutex wrapper
00038394: bl   0x382A8           ; mutex_lock   (syscall 102)
00038398: lwz  r0,4(r31)         ; g->[4]
0003839C: cmpwi r0,0
000383A0: bne  0x38364           ; si g->[4] != 0 -> NO espera
000383A4: lwz  r3,-0x7FE0(r30)
000383A8: bl   0x38148           ; cond_wait    (syscall 107)
...
0003836C: lwz  r0,0(r31)         ; g->[0]
00038370: cmpwi r0,0x10
00038374: beq  0x383D0           ; salida sólo si g->[0] == 0x10
00038378: bl   0x34B8C           ; dispatch(g->[8])
00038384: bl   0x249D8           ; bombeo de tareas
`

Con `g = 0x1042BFC8`: `g->[4] = 1` (escrito una vez en la construcción por
`func_0003632C`, nunca vuelto a escribir) y `g->[0] = 0` (tampoco). Es decir:

- `g->[4] != 0` ⇒ la rama `bne` **salta la espera** en cada vuelta → giro puro.
- `g->[0] != 0x10` ⇒ la condición de salida **nunca** se cumple.

**Medido en vivo** con un vigilante periodico (`MR7_PUMP=1`, cada 250 ms) que
resuelve la misma cadena que el hilo usa (`toc-0x7F68` -> base -> `base-0x7FE4`)
y que ademas lee el mutex para demostrar que ve memoria viva (alterna
`00000000`/`FFFFFFFF` millones de veces por segundo):

```
[pump] slot=0x1042BFC8 g=0x1042BFC8 f0=00000000 f4=00000001
       f8=1042C020 f12=00000000 mutex0=FFFFFFFF condp=0x1042BFE0 condid=9
```

Es decir, el objeto SI esta construido: estado 0, flag 1, *listener* presente
(`0x1042C020`) y condicion 9 valida. **Correccion**: la conclusion anterior
("`g->[8]` se borra sin escritor") era falsa; provenia de volcados tomados en
ejecuciones sin el ISO montado / antes de construir el objeto. El campo nunca se
borro.

Lo que sigue siendo cierto y medido: `g->[4] = 1` y `g->[0] = 0` de forma estable,
y ningun store del PPU los cambia tras la construccion.

**A donde despacha el *listener*** (`PS3_CALLTRACE_FROM=0x34B8C`):

```
[CALL] -> 0x000366D4 r2=0x004EEA88 r3=0x1042C020 r4=0 lr=0x00034BC4
        from=func_00034B8C+0x31D tid=11
```

`func_000366D4` es el procesador de la **lista de tareas** del frontend (recorre
`obj+0x48` y llama al parser de rutas `func_0027A0CC` por entrada, con despacho
final segun `obj+0x40`). Es decir: el frontend **si** corre su actualizacion por
frame; el bucle del `Initialize` no es un atasco aislado sino su bucle principal.

### 7.42 Defecto real encontrado en el toolkit: sys_cond_wait lee el timeout del argumento equivocado

`runtime/syscalls/sys_cond.c` toma `timeout = LV2_ARG_U64(ctx, 1)`, que es
`gpr[4]` = **el mutex**, cuando el ABI de lv2 es
`sys_cond_wait(cond, mutex, timeout)`, es decir el timeout va en `gpr[5]`
(`LV2_ARG_U64(ctx, 2)`).

**No se ha aplicado el cambio**, y el motivo está en el binario: el envoltorio de
MR7 hace exactamente

`
00038148: li   r4, 0        ; mutex = 0
00038150: li   r11, 0x6B    ; syscall 107
00038158: lwz  r3, 0(r3)
0003815C: sc
`

y **nunca fija 5**. Con el arreglo "correcto", esta llamada recibiría un timeout
basura y dejaría de bloquear; con el código actual recibe 0 (infinito), que es el
comportamiento que MR7 necesita aquí. Queda documentado como defecto a corregir
con una prueba A/B por título, no a ciegas.

### 7.43 Instrumentación añadida (opt-in, silenciosa)

| Variable | Qué hace |
|---|---|
| `PS3_SC_STATS=1` | histograma de syscalls cada 2 s + reparto del 102 por hilo |
| `PS3_COND_STATS=1` | contadores por condición (waits/instant/blocked/timeout/eperm/esrch/signal) |
| `[fs] read ... buf=0x… .. 0x…` | el log de lectura incluye ahora el rango de búfer (para descartar solapes) |

Verificado con las sondas apagadas: `garbage=0`, `device removed=0`,
`unlifted=0`, `AutoLoad=1`.

### 7.44 Instrumentacion nueva de esta ronda y correccion del registro

| Variable / cambio | Que aporta |
|---|---|
| `MR7_PUMP=1` | vigilante de 250 ms sobre el objeto del bucle (`slot`, `g`, `g+0/4/8/C`, mutex, cond): resuelve la cadena igual que el invitado y prueba que ve memoria viva |
| `PS3_SC_STATS=1` | histograma de syscalls cada 2 s + reparto del 102 por hilo + **sitio de llamada (`lr`)** del `mutex_lock` de tid 11 |
| `PS3_COND_STATS=1` | contadores por condicion (`waits/instant/blocked/timeout/eperm/esrch` y `signal/signal_all/nohit`) |
| `[CALL]` de `PS3_CALLTRACE` | incluye ahora `r2`, para detectar cuando un hilo usa el TOC de otro modulo |
| `[fs] read` | incluye el rango `buf=0x... ..0x...` para descartar solapes con estructuras |

Evidencia del sitio de llamada (2 s):

```
[scstats] top: 102=3669618 104=3669618 ... | mutex_lock/tid: t11=3669618
[scstats] | tid11 lock sites: lr=0x00038398:3669618
```

`lr=0x38398` es la instruccion inmediatamente posterior al `bl 0x382A8` del bucle:
**todos** los bloqueos vienen del bucle del frontend, no de otra parte del hilo.

Correcciones registradas:
1. La traduccion `vaddr -> offset` del EBOOT tenia un sesgo de `0x10000`; todas las
   lecturas de instrucciones se rehicieron y ahora coinciden con el codigo liftado.
2. La afirmacion "`g->[8]` se borra sin escritor" (�7.41 anterior) era **falsa** y
   queda sustituida por la medicion en vivo.
3. `sys_cond_wait` lee el timeout de `gpr[4]` (mutex) en vez de `gpr[5]`: defecto
   real del toolkit, **deliberadamente no aplicado** porque el envoltorio de MR7
   (`li r4,0`, sin tocar `r5`) dejaria de bloquear con el arreglo ingenuo.

### 7.45 El motor SI corre: bucle de frames a ~33 Hz, sin contenido que dibujar

Mapa de esperas y senalizaciones por (hilo, semaforo), 9 s con ``SEMTID=1``:

| Hilo | Espera en | Postea a |
|---|---|---|
| t1 (principal) | sem3 (230), sem6 (231), sem7 (231) | sem5 (254), sem3 (255), sem8 (21) |
| t4 (Draw) | sem5 (230) | sem6 (255), sem7 (3) |
| t6 (File) | sem8 (105) | sem8 (56), sem7 (7) |
| t11 (frontend) | -- | sem7 (220), sem8 (36) |

El ciclo ``t1 -> sem5 -> t4 -> sem6 -> t1`` se cierra ~255 veces en 9 s: **el motor
mantiene un bucle de frames a ~33 Hz**. No hay interbloqueo; hay frames vacios.

Cadena estable del hilo principal esperando sem7 (``SEMCHAIN=1 SEMCHAIN_TID=1
SEMCHAIN_SEM=7 SEMCHAIN_SKIP=120``):

```
[CHAIN:sem-chain] 000BDB18 000BE0CC 000C474C 00010DF4 00010694 00010250 00010244
```

Es decir: bucle de script/VM (``0x10244 -> ... -> 0x10DF4``) -> thunk de vtable
``0xC474C`` -> ``0xBE0CC`` -> ``0xBDA04`` (cuerpo del Draw Thread) -> ``0xBDB18``
(espera sem7).

**Estado del objeto de peticion del File Thread** (``PPU_WWATCH=10290640``):

```
[ww] 0x10290640 <- 0x10295640  guest-fn=0x000123B0
[ww] 0x1029065C <- 0x71        guest-fn=0x000127F4
[ww] 0x10290654 <- 0x3D000     guest-fn=0x000139C0   ; 249.856 B = la 69a lectura
[ww] 0x10290664 <- 0x3D000     guest-fn=0x000139C0
[ww] 0x10290644 <- 0x0         guest-fn=0x00014B5C   ; el File Thread queda ocioso
```

Campos: ``+0x34`` bufer, ``+0x38`` longitud, ``+0x20`` callback (OPD ``0x4CB788``
-> codigo ``0x000156AC``), ``+0x44`` estado. La ultima peticion se completa y el
File Thread queda idle: **nadie crea la peticion 70**.

La cadena de finalizacion pasa por ``0x156AC`` -> ``0x127F4`` (visible en el ``lr``
de las escrituras), que es donde debe decidirse la siguiente lectura. Ese es el
punto exacto a leer a continuacion.

### 7.46 Instrumentacion de esta ronda (ampliaciones)

| Cambio | Que aporta |
|---|---|
| ``SEMCHAIN_TID`` / ``SEMCHAIN_SEM`` / ``SEMCHAIN_SKIP`` | elegir hilo, semaforo y saltar las N primeras esperas (el original fijaba tid 0/sem 7 y solo capturaba el arranque) |
| ``MR7_PUMP`` | ademas de ``g``/mutex/cond, lee la lista de tareas del *listener* (``+0x3C``, ``+0x88``, ``+0x8C``, ``+0x90``) |

Medido en vivo: la lista de tareas del frontend oscila entre 3 y 12 entradas, es
decir el frontend procesa trabajo cada frame (no esta parado).

### 7.47 Quien produce las lecturas es el hilo del frontend, y deja de producir tras la peticion 0x71

Vigilando el valor con el que el callback compara el tamano
(``PPU_WWATCH=102AC9E8``, el objeto de estado de la capa de ficheros):

```
[ww] 0x102AC9E4 <- 0x3322D000  guest-fn=0x00015148 lr=0x000173E8 tid=11   ; bufer
[ww] 0x102AC9E8 <- 0x3D000     guest-fn=0x00015148 lr=0x000173E8 tid=11   ; tamano
[ww] 0x102AC9EC <- 0x1826A800  guest-fn=0x00015148 lr=0x000173E8 tid=11
[ww] 0x102AC9F0 <- 0x71        guest-fn=0x00015594 lr=0x00015684 tid=6    ; secuencia completada
[ww] 0x102AC9F4 <- 0x0         guest-fn=0x00015594 lr=0x00014C38 tid=6
```

Lectura correcta de la cadena:

- **El productor es tid 11** (el hilo del frontend), que rellena la peticion con
  ``func_00015148`` desde ``lr=0x000173E8`` (logica del loader en ``0x17xxx``).
- **El consumidor es tid 6** (File Thread), que marca la secuencia completada en
  ``+0xF0`` y deja ``+0xF4 = 0`` con ``func_00015594``.
- ``expect`` (``0x102AC9E8``) **es el tamano que el propio productor acaba de
  escribir**; que coincida con el tamano reportado es por tanto la rama **normal**
  de finalizacion, no un error. (Corregida la lectura anterior de �7.45.)
- Tras la peticion **``seq = 0x71``, 0x3D000 B** (la 69a lectura), **tid 11 deja de
  escribir peticiones**. El File Thread queda ocioso esperando sem8.

El bombeo de tareas (``func_000249D8``) sigue llamando a ``func_0001723C`` (es una
de las funciones mas calientes del perfil), asi que **el loader no esta bloqueado:
se queda sin demanda**. Es decir, la fase de carga ha terminado y **la transicion a
la siguiente fase (construir la escena) no ocurre** - coherente con que el
constructor de la mesa de 16 slots (``func_000180F0``, OPD ``0x4CB938``) nunca se
ejecute.

Punto de continuacion: quien debe encolar la siguiente fase tras ``seq 0x71``
(el gestor de carga del frontend), y por que no lo hace.

### 7.48 La cola de comandos del loader esta VACIA: la carga de arranque termino

El objeto del loader se resuelve con **dos** indirecciones, no una:
``obj = *(*(toc-0x7F84) - 0x8000)``. Con esa correccion, ``MR7_PUMP`` da el estado
estable:

```
[pump]   loader[obj=0x102A58FC head=56 tail=56 flag=0 +8824=0 +8828=0]
```

Y ``func_0001723C`` (el consumidor, una de las funciones mas calientes) usa
exactamente esos campos:

- entradas de ``0x34`` bytes en ``obj+0x34``,
- cabeza en ``+0x6834``, cola en ``+0x6838``, flag en el byte ``+0x683C``,
- al procesar una entrada llama a ``func_00015148`` (el que rellena la peticion
  de lectura, o sea, el productor real de las 69 lecturas),
- si ``cabeza == cola`` **retorna sin hacer nada**.

Por tanto:

- La cola tenia **56 comandos**, todos consumidos (coincide con los **56**
  callbacks ``0x156AC`` observados y con las 69 lecturas que generan).
- La cola esta **vacia** y nadie la rellena.
- ``flag = 0``: la otra rama (``loc_000173EC``, la que hace avanzar la cabeza) no
  se ejecuta.

**Correccion de encuadre**: no es que la carga se "atasque a 46 MB" por un fallo
de E/S; la **fase de arranque se completo** (56 comandos, 69 lecturas, todas
cerradas) y **el frontend no encola la fase siguiente**. Lo que falta es la
transicion: construir la escena (la mesa de 16 slots cuyo constructor
``func_000180F0`` nunca se ejecuta).

Sondas descartadas en esta ronda (para que no se repitan):

| Pista | Resultado |
|---|---|
| ``ARG[...] vtbl=0 -> code=0`` en el log | sonda heredada de otro titulo (direcciones fijas de LBP); **ruido** para MR7 |
| Mensajes TTY del juego | el titulo **no imprime nada** (``sys_tty_write`` volcaria el texto; no hay lineas) |
| ``+0x6834/+0x6838`` leidos en ``*(toc-0x7F84)`` | objeto equivocado: faltaba la indireccion ``-0x8000`` |

Siguiente paso natural: el interprete de script del frontend
(``0x10244 -> 0x10694 -> 0x10DF4``), que esta vivo pero no avanza de estado; hay
que ver que opcode/espera lo tiene parado y quien deberia desbloquearlo.

### 7.49 Verificaciones de cierre de ronda: escena vacia y naturaleza de la tabla `0x4CC8A0`

1. **El anillo de comandos del loader se limpia al consumir.** Volcado de
   ``obj+0x34`` (``0x102A5930``, 2912 B) en estado estacionario: **todo a cero**.
   No se puede reconstruir desde ahi que assets se pidieron; esa informacion solo
   esta en la traza de ``[fs] read`` (69 lecturas, 45.955.072 B) y en el callback
   (``seq`` y tamano).

2. **La region ``0x4CC8A0`` no es una lista de tareas**: es una **tabla de
   descriptores de funcion** ``{code, toc}`` con ``toc = 0x004EEA88`` en todas las
   entradas, del estilo ``{0x34A5C, 0x34B28, 0x34B8C, 0x34BD8, 0x34BFC, ...}``.
   Contiene precisamente ``0x34A5C`` (el que registra el *listener*) y ``0x34B8C``
   (el despachador), asi que es la tabla de metodos/observadores del objeto del
   frontend, no su cola de trabajo. El campo ``+0x90`` apunta dentro de ella.

3. **La mesa de 16 slots del Draw Thread sigue a CERO** en estado estacionario
   (``0x107C46B0``, 0xE0 B, los 14 registros impresos): la escena nunca se
   construye. Es el mismo resultado que en sesiones anteriores, ahora confirmado
   con la carga ya terminada.

Estado consolidado del arranque:

```
56 comandos de carga -> 56 callbacks -> 69 lecturas (45.955.072 B) -> cola vacia
  -> frontend vivo (lista de tareas 3..15) y motor a ~33 Hz
     -> mesa de escena del Draw Thread a cero -> no hay nada que dibujar
```

Lo que falta es **la construccion de la escena**, no la E/S ni el motor.

### 7.50 La mesa de escena NO estaba vacia, y el FIFO se esta descartando: geometria del anillo incoherente

**Correccion de �7.49 (y de sesiones anteriores).** La mesa del Draw Thread se
resuelve en **tres** pasos, no uno:

```
t = *(toc-0x7DB4);  u = *(t-0x8000);  v = *(u);  slot = *(v + 0xC0 + i*4)
```

Leida en vivo da ``v = 0x107C4670`` y **13 de 16 slots poblados**:

```
slot1=107C4780 slot2=107C47B0 ... slot7=107C4940 slot9=107C4A10 ... slot15=107C4D70
```

Es decir, **la escena SI esta construida** (el volcado anterior a ``0x107C46B0``
caia 0xC0 bytes antes del inicio real de la tabla y por eso salia a cero).

**El problema real: la geometria del FIFO es incoherente.**

- Nuestro contexto declara ``begin = 0x11100000``, ``end = 0x1110FFFC`` (64 KB en la
  region de IO que pasa el juego: ``_cellGcmInitBody(..., ioAddr=0x11100000)``).
- El juego publica ``put`` con su propia libgcm (``func_000BCAE4`` ->
  ``func_003AC858``) en ``0x20002000`` con valores **``0x00CB0D54``** y
  **``0x010A0D54``**, es decir un anillo en **memoria principal**, del orden de
  ``0x00CB0000..0x01100000`` (~4,3 MB), no en ``0x11100000``.
- El walker arranca en el ``current`` del contexto (``0x111001A4``) y alli solo hay
  ceros, asi que va "resincronizando" hacia el ``put`` vivo:

```
FIFO NOP run filled the page to 0x00D00000 at get 0x00CB0D34; resyncing to put 0x010A0DD4
FIFO resync (zero-page) 0x00CB0D34 -> put 0x010A0DD4
FIFO NOP run filled the page to 0x01100000 at get 0x010A12D0; resyncing to put 0x00CB12F0
```

Resincronizar al ``put`` **descarta justo el tramo donde estan los comandos**: se
salta de ``get`` a ``put`` sin decodificar nada. Eso explica el sintoma exacto:
escena construida, motor a 33 Hz, juego publicando ``put``, y **cero comandos
decodificados** (pantalla negra).

Esto **no es un fallo del juego**: es la geometria de nuestro HLE. La pista a
seguir es de donde obtiene el juego su base de anillo (``0x00CB0000`` aprox.);
con la base correcta el ``current`` del contexto debe caer en el mismo tramo y el
walker decodifica en vez de resincronizar.

Sondas de esta ronda: ``MR7_PUMP`` resuelve en vivo la cadena del Draw Thread
(3 indirecciones) y la del loader; ``PPU_WWATCH=20002000`` identifica quien
publica ``put`` (``func_000BCAE4``, 6 escrituras por tanda) frente a la inicializacion.

### 7.51 El anillo real del juego esta en `0x121A0000`; nuestro contexto es otro objeto

Siguiendo la cadena que usa la libgcm del juego (`func_000BCAE4`):

```
r9 = *(*(toc-0x7E30) - 0x7FFC)      -> 0x10655120   (gCellGcmCurrentContext)
rd = *(r9)                          -> 0x10655110   (contexto que usa el juego)
```

Y ese contexto, leido en vivo, es:

```
ring[r9=0x10655120 rd=0x10655110
     begin=0x121A0000 end=0x1258FFFC cur=0x121A1350..0x121A1370 cb=0x00000000
     put=0x00CB12F0 get=0x00CB12F0]
```

Hechos:

1. **El anillo que el juego usa esta en `0x121A0000..0x1258FFFC`** (~3,9 MB), dentro de
   la region de IO que el juego pidio (``ioAddr=0x11100000``, ``ioSize=0x3100000``):
   es ``ioAddr + 0x10A0000``. **No** es ``ioAddr``.
2. ``cur`` **avanza** (0x121A1350 -> 0x121A1370): el juego si construye comandos.
3. ``put == get == 0x00CB12F0`` en el registro de control: el walker se declara al
   dia, pero lo hace porque **resincroniza get al put vivo** (�7.50), no porque haya
   decodificado.
4. Nuestro ``s_gcm_context_ea`` (el bloque que asigna ``cellGcmSetupContext`` y que
   publicamos en ``ctx_out``) conserva ``begin=0x11100000 end=0x1110FFFC``: la
   libgcm del juego **no lo usa** para su anillo, mantiene el suyo propio.
5. En ``0x121A0000`` no hay comandos al inicio (256 B a cero); el stream, si existe,
   esta en otro punto del anillo.

Consecuencia: el walker parte de un ``current`` (``0x111001A4``) que no pertenece al
anillo del juego, no encuentra comandos y salta al ``put``. Por eso ``put == get`` y
no se dibuja nada, con la escena construida y el motor a 33 Hz.

**Siguiente accion**: leer la inicializacion de la libgcm del juego (donde fija
``begin/end`` en su descriptor y donde calcula el ``put``) y hacer que el walker use
**esa** geometria. Es codigo del juego, no de la HLE: no hay que inventar nada, hay
que leer de donde saca ``0x00CB12F0``.

Sondas anadidas: ``MR7_PUMP`` imprime ``ring[...]`` (begin/end/current/cb + put/get).

### 7.52 El `put` correcto se calcula, pero el envio del frame esta saltado por una puerta

La libgcm del juego calcula el `put` en ``func_003AC68C``:

```
p = *(toc-0x6750);  q = *(p);  put = *(q+8)
```

Medido en vivo: ``q = 0x10655110`` y ``put = 0x121A1370`` (el ``current`` del anillo
real). Es decir, **el juego sabe perfectamente donde esta su cabeza de escritura**.

Pero el registro de control sigue con ``0x00CB12F0`` (valor del arranque), porque el
envio no se ejecuta. En ``func_000BE0CC``:

```
call func_000E3D34()              ; puerta
if (r3 != 0) goto 0xBE354         ; <- salta el envio
put = func_003AC68C()             ; 0x121A1370
call func_000BCAE4(put)           ; submit (NO se ejecuta)
```

La puerta ``func_000E3D34`` -> ``func_000E4B74`` devuelve un **byte de una tabla**:

```
base = *(*(toc-0x7DBC) - 0x7FF8) = 0x100101A8
flag = *(u8*)(base + 0x10 + idx*0x10 + 0xC)
```

Volcado de los 16 flags (``MR7_PUMP``):

```
gate[base=0x100101A8 flags: 0 1 1 0 0 1 1 0 0 0 0 0 0 0 0 0]
```

Es decir, 4 de 16 entradas estan marcadas. Con ``flag != 0`` el frame no se envia.
Queda por determinar **que indice usa el objeto de display** y **que significa ese
flag** (puede ser una comprobacion legitima de "buffer inactivo" o un estado que
nuestra HLE deja mal).

Estado de la cadena, ya sin huecos hasta la ultima puerta:

```
escena construida (13/16 slots) -> motor a 33 Hz -> juego genera comandos
  -> put correcto calculado (0x121A1370) -> puerta devuelve flag != 0 -> NO submit
     -> registro de control con el put viejo (0x00CB12F0) -> RSX sin comandos
```

### 7.53 La puerta NO salta: el hilo principal vive en la rama de espera de vblank

Reproducida la cadena de indices con las **dos** bases que usa el juego
(``b1 = *(*(toc-0x7DC0)-0x7FF8)`` para el primer nivel, ``b2 = *(*(toc-0x7DBC)-0x7FF8)``
para el segundo):

```
gate[b1=0x1000FFC8 b2=0x100101A8 obj=0x1079C740 idx=0x0000000F e1=0x00000000 flag=0]
```

Con ``idx = 0xF`` el primer nivel devuelve 0 y el flag leido es **0**, es decir la
puerta **no** impide el envio. La hipotesis de �7.52 queda descartada.

Releyendo la estructura real de ``func_000BE0CC``:

```
loc_000BE320:
    func_000E3BC4(); func_000E3D34();      ; puerta
    if (r3 == 0) goto loc_000BE34C;
loc_000BE338:
    *(*(toc-0x7F70)) = 1;
    func_000BDB18();                        ; espera sem7  <-- lr=0xBE348 del informe antiguo
    goto loc_000BE124;
loc_000BE34C:  func_000BDFB4(); goto loc_000BE338;
loc_000BE354:  func_000BDFB4(); goto loc_000BE30C;   ; camino del submit
loc_000BE2EC:  ... puerta ... loc_000BE30C: put=func_003AC68C(); func_000BCAE4(put);
```

Es decir: la rama que el hilo principal ejecuta de verdad es la de **sincronizacion
con vblank** (``0xBE320 -> 0xBE338 -> sem7``). El bloque de envio (``loc_000BE30C``)
es otra rama y no se alcanza en el estado estacionario.

Por tanto el problema no esta en la puerta del flag sino en **por que el bucle no
llega a ``loc_000BE2EC``** despues de ``loc_000BE124`` (inicio de frame). Los dos
candidatos inmediatos son:

1. El salto de ``0xBE110`` (``if (cr>>13) goto loc_000BE320``), que puede estar
   mandando cada vuelta directamente a la rama de vblank sin pasar por el cuerpo.
2. La condicion de ``loc_000BE2EC`` (``if (r10 != 0) goto loc_000BE198``).

Siguiente instrumentacion: volcar esos dos valores (el resultado de la comparacion
de ``0xBE110`` y ``r10`` en ``0xBE2EC``) en el vigilante. Sigue siendo un unico
booleano, pero ahora el correcto.

### 7.54 Condicion exacta del submit, verificada en el binario

Con desensamblador (capstone, ya disponible en el entorno) y no a mano:

```
000BE120: beq  cr4, 0xbe320     ; (arg de entrada == 0) -> rama de vblank
...
000BE18C: cmpdi cr7, r0, 0
000BE190: beq  cr7, 0xbe198     ; r0 == 0            -> NO submit
000BE194: beq  cr4, 0xbe2ec     ; arg de entrada == 0 -> SUBMIT
000BE198: bl   0xbcc20
```

Donde:

- ``arg`` es el argumento con el que se llama ``func_000BE0CC``, y su origen esta en
  ``func_00010DF4``: ``arg = EQ(obj[0x10], *(*(*(toc-0x7FAC)-0x7FB8)))``.
- ``r0 = *(*(*(toc-0x7FE4)))``.

**Medido en vivo** (``MR7_PUMP``): ``gval = 0x00000000`` y ``app+0x10 = 0x00000000``
-> **EQUAL** -> ``arg = 1`` -> la ruta de submit (que exige ``arg == 0``) **no se toma**.

Es decir, el frame se envia solo cuando esa comparacion da **distinto**, y en nuestro
port los dos lados valen 0 a la vez. Es el patron de un contador de doble buffer /
paridad de frame que deberia alternar y no lo hace; el lado global
(``*(*(*(toc-0x7FAC)-0x7FB8))``) tambien esta a 0.

Nota de metodo: la traduccion ``vaddr -> offset`` y el mapeo de campos CR del lifter
se verificaron con capstone. El lifter mapea ``crN`` al campo ``(7-N)``, de forma
**coherente** entre comparacion y bifurcacion: no es un bug.

Siguiente paso: identificar el objeto ``obj`` que recibe ``func_00010DF4`` (viene de
``func_00010694``) y **que deberia incrementar** su ``+0x10``, para que la paridad
alterne y la ruta de submit se ejecute.

### 7.55 El juego publica put/get en OTRO bloque de control, y su anillo cubre la region IO (hipotesis: descartada como causa en 7.56)

Tres mediciones encadenadas cierran el diagnostico.

**1. El fence de GCM funciona.** Vigilando el label 0x80 (``0x20000800``):

```
[ww] 0x20000800 <- 0x1   guest-fn=0x00426490            ; lo escribe nuestro walker (RSX)
[ww] 0x20000800 <- 0x0   guest-fn=0x000BCAE4 lr=0xBCB7C ; lo limpia el juego
[RSX] label write @0x20000800 = 0x00000001 (sync fence, raw 0x00010000)
```

Alternan ~26 veces por segundo: el juego **si** llama a ``func_000BCAE4`` (el submit) de
forma continua. La condicion ``r0 != 0`` de ``0xBE190`` se cumple en cuanto el label vale 1.

**2. El bloque de control que usa el juego NO es el nuestro.** La direccion sale de
``func_000BCAE4``: ``ctl = *(*(toc-0x7E30) - 0x7FF4)``. Medido en vivo:

```
ctl[ptr=0x104A0CFC put=0x11DB12F0 get=0x11100000 (DIFFERENT from GCM control)]
```

- El juego publica en **``0x104A0CFC``**, no en ``GCM_CONTROL_GUEST_ADDR``
  (``0x20002000``), que es donde mira ``cellGcmDebugHeads`` y el walker.
- ``put = 0x11DB12F0``, ``get = 0x11100000``: **el anillo es la region IO entera** que
  el juego reservo (``ioAddr = 0x11100000``, ``ioSize = 0x3100000``), no los 64 KB de
  ``cmdSize``.

**3. Por eso el walker descarta.** Nuestro contexto declara ``end = 0x1110FFFC``
(64 KB), asi que un ``put`` de ``0x11DB12F0`` (13,3 MB dentro de la region) queda
"fuera de rango": el walker parte de un ``current`` inutil, encuentra ceros y
resincroniza al ``put`` vivo, salto que **se lleva por delante los comandos**.

Consecuencia final, con las tres piezas:

```
el juego construye comandos y publica put en 0x104A0CFC (region IO, ~13 MB)
  -> nuestro walker mira 0x20002000 y un anillo de 64 KB
     -> no decodifica, resincroniza y descarta
        -> no se ejecutan los comandos -> sin geometria ni texturas -> pantalla negra
```

**Los dos defectos estan en nuestro HLE** (``libs/video/cellGcmSys.c``):

1. ``GCM_CONTROL_GUEST_ADDR`` / ``cellGcmGetControlRegister`` devuelve un bloque
   propio (``0x20002000``) que el juego sustituye por el suyo; el walker debe leer el
   que el juego usa (o hacer que el juego use el nuestro de verdad).
2. El anillo se define con ``cmdSize`` (64 KB) cuando el juego usa **``ioSize``**
   (49 MB) desde ``ioAddr``; ``end`` debe cubrir la region IO.

Siguiente accion: leer la inicializacion de la libgcm del juego para ver como calcula
``0x104A0CFC`` y la base del anillo, y alinear (1) y (2). Es codigo de la HLE: se
prueba en minutos y el criterio de exito es objetivo (el walker deja de resincronizar
y ``get`` recorre comandos reales).

### 7.56 Sonda de anillo (GCM_RING_FROM_IO): NO es la causa raiz. Correccion de una afirmacion mia

Diagnostico del ``put`` con ``GCM_DRAINDBG=1``:

```
[DRAIN] getoff=00000000   put=00000010   ref=00000000 ctx.current=11100010
[DRAIN] getoff=00CB12F0   put=00CB12F0   ref=00000012 ctx.current=111001A4
```

El juego publica ``put`` como un offset de ``0xCB12F0`` (13,3 MB) dentro de la region
IO, mientras el contexto declara un anillo de ``cmdSize`` = 64 KB. Por eso se anadio
la sonda ``GCM_RING_FROM_IO=1`` (ensancha el anillo a ``ioSize``).

**Medicion A/B honesta** (mismo arranque, 10 s, contando mensajes ``[RSX]``):

| Mensaje | Sin sonda | Con sonda |
|---|---|---|
| DRAW_ARRAYS | 12 | 14 |
| CLEAR_SURFACE | 11 | 12 |
| label write | 8 | 8 |
| unknown method | 50 | 50 |

**Conclusion**: la sonda cambia +2 draws y +1 clear. El walker **ya decodificaba los
comandos del arranque** en ambos casos; el ensanche del anillo **no** es la causa raiz
del frame negro. La afirmacion anterior de este mismo apartado ("de cero comandos a
14") era un error de medicion y queda retirada.

Lo que si queda establecido y medido:

- El motor corre a ~33 Hz (``t1 -> sem5 -> t4 -> sem6 -> t1``).
- La escena esta construida (13/16 slots).
- El fence de GCM funciona (label 0x80 alterna 1/0 ~26 veces/s).
- ``func_000BCAE4`` (submit) se ejecuta, pero **el juego deja de generar comandos
  nuevos**: su ``ctx.current`` se queda en ``0x111001A4`` y ``put`` no avanza de
  ``0x00CB12F0`` tras el arranque.

Siguiente foco (sin inventar): por que el codigo de render del juego no emite mas
comandos despues del arranque, con la escena ya construida.

### 7.57 El juego emite 30 comandos y nunca mas; el Draw Thread no emite ninguno

Vigilando ``ctx.current`` (``0x0F800008``), el campo que se actualiza **por cada
comando emitido**, durante 7 s sin ninguna otra sonda:

```
writes to ctx.current = 30      (todas tid=1)
... 0x0F800008 <- 0x11100170  guest-fn=0x003AEA24 lr=0x000BCF68
    0x0F800008 <- 0x1110017C  guest-fn=0x003AE37C lr=0x000BCF88
    0x0F800008 <- 0x1110018C  guest-fn=0x003ACFE8 lr=0x000BDC40
    0x0F800008 <- 0x11100194  guest-fn=0x003AC6F4 lr=0x003AC804
    0x0F800008 <- 0x111001A4  guest-fn=0x003ACE70 lr=0x000BDC74   (ultima)
```

Conclusiones:

1. **La emision se detiene por completo** en ``0x111001A4`` (420 bytes). No es que el
   walker no la vea: la funcion que rellena el anillo deja de escribir.
2. **Todas las escrituras son de tid=1** (hilo principal). El **Draw Thread (t4) no
   emite ni un solo comando** en toda la ejecucion, pese a que participa en el
   handshake ``sem5/sem6`` a ~33 Hz.
3. Los emisores son la capa de libgcm del juego (``0x3AC6F4``, ``0x3AEA24``,
   ``0x3AE37C``, ``0x3ACFE8``, ``0x3ACE70``) llamada desde ``0xBCF68``, ``0xBCF88``,
   ``0xBDC40``, ``0xBDC74``.

Es decir: con la escena construida (13/16 slots), el fence funcionando y el motor a
33 Hz, **el hilo de dibujo no produce geometria y el hilo principal deja de emitir
tras el arranque**. El siguiente paso util ya no es una sonda de memoria: es un
**breakpoint** en el bucle de dibujo (``func_000E6CBC``) y en la entrada del frame
(``func_000BE0CC``) para ver que rama se toma, porque la diferencia entre "no se
llama" y "se llama y decide no dibujar" no se distingue desde fuera.

### 7.58 Cadena completa de render, hasta el punto donde se queda vacia

Con las mediciones de �7.50-7.57, la ruta de dibujo queda trazada de punta a punta:

**Hilo de dibujo (t4, cuerpo ``func_000BDA04``)**, por cada senal ``sem5``:

```
func_000BDA04:
    sem_post(1)                          ; 0x5E = 94
    sem_wait(sem, 0)                     ; 0x5C = 92, bloqueante  -> aqui pasa ~100% del tiempo
    if (*(*(toc-0x7F80)) == 0) sys_ppu_thread_exit(0)
    func_000BED40(*(toc-0x7FE0))         ; render
    func_003AFE54(*(toc-0x7FFC))         ; libgcm
    func_000C5680()
    sem_post(1); sem_wait(sem, 0)        ; bucle
```

**Dentro de ``func_000BED40``:**

```
func_000BEC3C()
func_000E220C()          -> devuelve *(*(*(toc-0x7DC4)-0x7FFC))
func_000E24C8(obj, 1)    -> tail-call func_000E244C(obj, 1, -1)
```

**``func_000E244C`` = envio de la lista de dibujo:**

```
func_000E2264()                              ; CONSTRUYE la lista en pila (0x70(r1))
g = *(*(toc-0x7DC4)-0x7FF8)
func_003ACABC(*(g), &lista)                  ; libgcm: envia la lista
*(obj+0x57C) = 1
```

Y los hechos medidos sobre esa ruta:

- ``ctx.current`` (``0x0F800008``, se escribe **por comando**): **30 escrituras en total,
  todas de tid=1, y la ultima deja el anillo en ``0x111001A4``**. Despues, cero.
- Ninguna escritura de ``ctx.current`` procede del Draw Thread.
- El cuerpo del frame del hilo principal **si se ejecuta** (escrituras con
  ``guest-fn=0x000BE0CC lr=0x000BE210``, dentro del cuerpo ``0xBE124..0xBE2EC``).
- El fence de GCM funciona y ``func_000BCAE4`` (submit) corre ~26 veces/s.

**Conclusion**: la cadena de render esta viva y se ejecuta hasta el final
(``E244C`` -> ``E2264`` + ``3ACABC``), pero **la lista de dibujo que se envia esta
vacia**: no se emite ni un comando de geometria. El problema ya no esta en el motor,
el FIFO, los semaforos ni el envio: esta en la **construccion de la lista**
(``func_000E2264``), es decir, en la logica de visibilidad/escena del juego.

Ese es el punto donde una sonda de memoria deja de servir y hace falta un breakpoint
en ``func_000E2264`` para ver por que recorre la escena (13/16 slots con objeto) y no
emite ninguna entrada.

### 7.59 Correccion importante: el emisor NO escribe en ctx.current, y el Draw Thread SI renderiza

Dos mediciones nuevas obligan a matizar �7.57.

**1. El Draw Thread si ejecuta el envio de la lista.** Vigilando ``obj+0x57C``
(``0x1079CCCC``), campo que escribe ``func_000E244C`` cada vez que corre:

```
[ww] 0x1079CCCC <- 0x1 (w4) guest-fn=0x000E244C lr=0x000E2498 tid=4
[ww] 0x1079CCCC <- 0x2 (w4) guest-fn=0x000E244C lr=0x000E2498 tid=4
... 1439 escrituras en 8 s, TODAS de tid=4  (~180/s)
```

Es decir: el hilo de dibujo **si** llega a ``func_000E244C`` continuamente. El
"guest=0 samples" del sampler solo indica que el trabajo por frame es muy corto.

**2. ``func_003ACABC`` no escribe en ``ctx.current``.** Su primera instruccion util
escribe una cabecera de comando en ``*(obj+8)``:

```
obj+8 = 0x00000008  (puntero de un bufer propio del objeto)
vm_write32(obj+8, 0x00040000 | 0x194)     ; cabecera de metodo RSX
```

Por tanto ``ctx.current`` **no mide la emision real**: mide solo lo que pasa por el
camino de libgcm que usa el contexto. El juego construye comandos en **buffers
propios** (y en ``0x00CB0000``/``0x010A0000`` segun los ``put`` observados) y los
publica al final con ``put``.

Consecuencia: la afirmacion de �7.57 ("el juego no emite mas comandos") queda
**matizada**: el juego sigue construyendo comandos; lo que no ocurre es la
**publicacion del ``put``** que los entrega al RSX.

Siguiente foco: quien debe publicar el ``put`` a partir de ese bufer
(``func_000BCAE4`` recibe el ``put`` como argumento; su llamador en ``0xBE30C`` lo
obtiene de ``func_003AC68C`` = ``*(*(*(toc-0x6750))+8)``).

### 7.60 Entorno de depuracion (estado)

- ``build_dbg`` (``-O1 -g``, MinGW/DWARF): **compila** (``MyGameRecomp.exe`` 382 MB)
  pero el toolchain **no genera PDB**; VS no puede leer DWARF.
- ``lldb.exe`` del toolchain **falla al arrancar** (``0xC06D007E``, fallo de carga
  diferida) incluso con ``--version``: no es utilizable tal cual.
- Lanzador preparado: ``port\run_debug.cmd`` y guia ``docs\DEPURACION.md`` por si se
  resuelve el depurador mas adelante.
- La depuracion se ha hecho, por tanto, con **sondas de memoria** (``PPU_WWATCH`` /
  ``MR7_PUMP``), que es lo que ha permitido llegar hasta aqui.

### 7.61 El objeto de comandos es 0x10655110 y su bufer vive en 0x121A0000: hay DOS anillos

Corregido un desliz de offset (``-0x7FF8`` vs ``-0x7FFC``) en la cadena de
``func_000E244C``, el objeto que recibe ``func_003ACABC`` es:

```
[pump]   dcmd[obj=0x10655110 bufptr=0x121A1350 -> 0x121A1370]
```

Es decir:

- El objeto de comandos es **``0x10655110``**, el mismo bloque que ya aparecio como
  "contexto del juego" en �7.51 (``begin=0x121A0000 end=0x1258FFFC cur=0x121A1370``).
- ``+0x8`` es el puntero de escritura y **avanza** (``0x121A1350 -> 0x121A1370``),
  luego si se emiten comandos, en el anillo de ``0x121A0000``.
- Pero el ``put`` que publica el juego es un **offset** ``0x00CB12F0`` en el registro
  de control (y ``0x11DB12F0`` como EA en su bloque ``0x104A0CFC``), que corresponde
  a **otro** tramo de la region IO.

Con los tres datos juntos, el cuadro es **doble bufer de comandos**:

| Bufer | Direccion | Estado observado |
|---|---|---|
| A | ``0x11100000 + ~0xCB12F0`` | ``put`` publicado (0x11DB12F0) |
| B | ``0x121A0000`` | puntero de escritura del objeto de comandos |

Y el avance total del puntero B en 9 s es de solo ``0x20``: casi no se emite. El
recorrido de la escena (``E244C`` ~180/s) se ejecuta pero **casi no produce entradas**.

Estado final de esta linea de investigacion: el fallo esta en que la escena produce
muy pocas entradas de dibujo (no en el motor, el FIFO, los semaforos ni la publicacion
del ``put``). Para seguir hacen falta dos cosas que hoy no tenemos:

1. Un **depurador funcional** (el `lldb` del toolchain aborta al arrancar y no hay PDB).
2. Entender la logica de visibilidad/culling del juego (``func_000E2264`` y quien lo
   llama por objeto), que es codigo sin simbolos.

Documentado tambien en ``docs\DEPURACION.md`` el procedimiento preparado por si se
resuelve el depurador.

### 7.62 DEPURADOR FUNCIONAL (lldb) y primer dato real de la escena

**Receta que lo hace funcionar** (el `lldb` del toolchain abortaba con 0xC06D007E):

1. `dumpbin /dependents liblldb.dll` mostro la causa: necesita **`libpython3.14.dll`**,
   que no existe en el sistema (hay 3.12/3.13).
2. Shim: copiar `python313.dll` como `libpython3.14.dll` en
   `...\cmake-clang-v1\latest\bin\`.
3. Variables: `PYTHONHOME` y `PYTHONPATH` apuntando a Python313
   (`...\Python313` y `...\Python313\Lib;...\Python313\DLLs`).
4. Con eso `lldb --version` responde (22.1.8) y **los breakpoints funcionan** sobre
   `build_dbg\MyGameRecomp.exe` (MinGW/DWARF).

Script de comandos preparado: `port\lldb_scene.txt`.

**Primer breakpoint real en `func_000E2264`** (`frame select 1`, porque el stop cae
dentro del macro `PPU_MM_SNAP`):

```
ctx->gpr[3] = 0x1079C750     ; objeto
ctx->gpr[4] = 1              ; indice
ctx->gpr[5] = 0x0FEFF240     ; lista de salida (pila)
descriptor  = 0x1079C794     ; obj + 0x44

+0x00 = 4
+0x04 = 0x01000000
+0x08 = 1920
+0x0C = 1080
+0x10 = 0        <-- byte que la funcion compara con 1
+0x20 = 0
+0x24 = 10
+0x28 = 0
```

Lectura: **ese descriptor es el buffer de display de 1920x1080** (ancho/alto en
`+0x08`/`+0x0C`), y con `+0x10 = 0` `func_000E2264` toma la rama "sin cambios".
Es una llamada benigna, no una malla.

Siguiente paso (ya con depurador): poner el breakpoint en el bucle que llama a
`func_000E24C8` **por objeto** y examinar los descriptores de malla (los que no son
1920x1080); ahi estara el flag de visibilidad que descarta todo.

Nota: el puerto renderiza a 1280x720 mientras el juego pide un display de 1920x1080;
conviene revisar si nuestro `cellGcmSetDisplayBuffer`/scanout ignora esas dimensiones.

### 7.63 Depurador: descriptores reales de dibujo y correccion de �7.62

Recorridas 8 llamadas a ``func_000E2264`` con breakpoint (fichero
``port\lldb_mesh.txt``), imprimiendo ``r3``, ``r4`` y los campos del descriptor
``r4``-esimo (``obj + indice*0x50``, calculado como el codigo):

| hit | hilo | r4 | +0x08 | +0x0C | +0x10 |
|---|---|---|---|---|---|
| 1 | ppu main | 1 | 8 | 2 | 0 |
| 2 | ppu main | 6 | 13 | 0 | 0 |
| 3 | ppu main | 1 | 8 | 2 | 0 |
| 4 | Draw Thread | 1 | 8 | 0 | 0 |
| 5-8 | Draw Thread | 1/2 | 8 | 2/0 | 0 |

**Correccion de �7.62**: los valores 1920/1080 que publique alli se leyeron en
``obj+0xC4`` (una configuracion de display del objeto), **no** en el descriptor de
dibujo (``obj+0x44``). El descriptor de dibujo tiene ``+0x08 = 8..13`` y
``+0x0C = 2/0``.

**Correccion de mi lectura del flag**: ``+0x10`` NO es un flag de visibilidad
booleano. El codigo hace ``cmp +0x10, 1`` y **solo si es igual a 1** va a la rama
``0xE2424``; con 0 sigue el camino normal, que lee ``*(obj+0x580)`` y construye la
entrada con datos de vertice. Es decir: con ``+0x10 = 0`` **si se construyen
entradas**; no es la causa del frame negro.

**Quien escribe ese bloque** (``PPU_WWATCH=1079C7A0``):

```
0x1079C7A4 <- 0 by 0x004083B4   (x432)   ; memset-like del bloque
0x1079C7A4 <- 0 by 0x000E7988   (x209)
0x1079C7A0 <- 2 / 0 by 0x004083B4        ; +0x0C alterna
0x1079C7AC <- 1     by 0x004083B4
```

El bloque se pone a cero con ``func_004083B4`` y se repuebla cada frame; nadie
escribe ``+0x10 = 1``, pero **no hace falta** para construir la entrada.

Siguiente paso con el depurador ya operativo: breakpoint en ``func_003ACABC`` (el que
emite los comandos) y ver **donde escribe** y **cuanto avanza** su puntero por
llamada; con eso se sabe si el problema es que se emiten pocos comandos o que el
``put`` no se publica desde ese bufer.

### 7.64 CIERRE: el `put` SI avanza (doble bufer), y el problema es el contenido de la escena

Medido con breakpoint en ``func_000BCAE4`` (el submit), 5 llamadas consecutivas:

```
r3 = 0x11DB0960   control(0x20002000) = 0x00000194
r3 = 0x121A0DD4   control = 0x00CB0960
r3 = 0x11DB0D54   control = 0x010A0DD4
r3 = 0x121A0D54   control = 0x00CB0D54
r3 = 0x11DB0D54   control = 0x010A0D54
```

Lectura:

- El juego publica ``put`` **alternando dos buffers**: uno en ``0x11DBxxxx`` y otro en
  ``0x121Axxxx`` (~4 MB de separacion), un frame cada uno.
- El registro de control guarda **offsets** (``0x11DB0960 - 0x11100000 = 0xCB0960``),
  por eso ``PPU_WWATCH``/``DRAIN`` veian ``0x00CB....``.
- **Retirada definitiva de "put congelado"**: el ``put`` avanza en cada frame
  (``0x0960 -> 0x0D54 -> ... -> 0x12F0`` a lo largo de la ejecucion). El valor fijo que
  se veia era un artefacto de muestreo.

Y el contenido: el Draw Thread emite en ``0x121A0000`` (~400 B por llamada, ~180
llamadas/s) y el hilo principal en ``0x11DB..``; el walker consume ambos y decodifica
**12-13 DRAW_ARRAYS de 3 vertices** (un triangulo) y unos CLEAR. Es decir:

```
pipeline completo funcionando: construccion -> emision -> publicacion -> decodificacion
contenido de escena: minimo (un triangulo + clears)
```

**Conclusion de la sesion**: no queda ningun fallo de infraestructura por encontrar. El
juego no esta dibujando la pantalla de titulo porque **el frontend no ha llegado a
ese estado** (la carga de arranque termina y el frontend se queda bombeando tareas),
y el renderer dibuja fielmente la escena casi vacia que tiene.

El siguiente trabajo, por tanto, no es de render: es **la maquina de estados del
frontend** (``0x10244 -> 0x10694 -> 0x10DF4``), que es codigo de juego y ahora si se
puede depurar con breakpoints.

### 7.65 Sonda de presentacion (RSX_FORCE_CLEAR) y lanzador para verla

Para separar "escena vacia" de "no se presenta nada", se anadio en
``libs/video/rsx_d3d12_backend.c`` una sonda opt-in que sustituye el color del
*clear* del backbuffer:

| Valor | Efecto |
|---|---|
| ``RSX_FORCE_CLEAR=RRGGBB`` | color fijo (p.ej. ``FF0000`` rojo) |
| ``RSX_FORCE_CLEAR=cycle``  | el color avanza cada frame (ciclo de 4 s a 60 Hz) |

Verificado: imprime ``[D3D12] RSX_FORCE_CLEAR=...`` y el juego sigue sano
(``garbage=0``, ``device removed=0``, 12-13 draws).

Lanzador preparado: ``port\run_clear.cmd`` (doble clic).

Interpretacion:

- Si la ventana **cicla colores** -> la presentacion funciona; el negro habitual es
  solo que la escena esta casi vacia (coherente con �7.64).
- Si la ventana sigue **negra** -> hay ademas un fallo de presentacion, que habria que
  arreglar antes de seguir con la maquina de estados del frontend.

### 7.66 La ventana SI presenta, pero a ~3-4 fps (no 60): el negro es escena vacia

Prueba de presentacion con ``RSX_FORCE_CLEAR=cycle`` (color que cambia con el tiempo):

- El usuario ve **pantalla completa en naranja** -> la swapchain presenta y el
  backbuffer llega al monitor. **Queda descartado** un fallo de presentacion.
- Contador anadido (``[D3D12] present #N``): **40 presents en 11 s = ~3,6 fps**.

Dos consecuencias:

1. El negro habitual es **escena vacia**, no ausencia de presentacion (coherente con
   �7.64: 12-13 DRAW de 3 vertices).
2. El bucle invitado va a ~33 Hz pero la **presentacion va a ~3,6 fps**: hay un cuello
   de botella en el camino de present (o el present solo ocurre cuando el invitado
   hace flip, y lo hace pocas veces). Esto limita cualquier animacion futura y es lo
   siguiente que hay que mirar.

Nota: el ciclo se hizo dependiente del **tiempo** (``GetTickCount``) porque al ir por
frame (240 frames) a 3,6 fps tardaba ~66 s por vuelta y parecia estatico.

### 7.67 CORREGIDO: el motor de dibujo NO era el problema (era varianza entre ejecuciones)

Con ``CELLMARK_BLINKDBG=1`` el backend imprime en cada present el numero de registros
de dibujo acumulados. Primer resultado: 287 presents en 9 s, **todos con draws=0**.

Parecia que el motor de dibujo no consumia los comandos, y un A/B con
``PS3RECOMP_RSX_ENGINE=dispatch`` dio 0 -> 11 presents con draws. **Era falso.**

Repeticion (4 ejecuciones alternando la variable, luego 3 mas sin ella):

```
off      : presents=260 con_draws=195     <- outlier
dispatch : presents=272 con_draws=11
off      : presents=250 con_draws=11
dispatch : presents=251 con_draws=11
off x3   : presents=311/313/313 con_draws=11/11/10
```

Conclusion: **la variable no influye**. El valor 195 fue una ejecucion atipica (la
primera tras montar el ISO). En estado estable el comportamiento es:

- ~310 presents en 10 s (~31/s)
- **~11 de ellos con draws** (unos pocos frames al arranque)
- 12 DRAW_ARRAYS decodificados del FIFO

Es decir: el juego emite geometria en los primeros frames y despues solo presenta
frames vacios. Coherente con �7.64 (escena casi vacia), no con un fallo del motor.

**Retirada** de la conclusion de la version anterior de este apartado ("el motor no
consumia los comandos"): no hay evidencia de ello. La causa sigue siendo el contenido
de escena, es decir, que el frontend no llega al estado de titulo.

Nota metodologica: este es el tercer falso positivo de la sesion por no repetir una
medicion. Las sondas nuevas (``RSX_FORCE_CLEAR``, contador ``present #N``) siguen siendo
utiles; lo que hay que hacer siempre es **repetir 3 veces** antes de concluir.

### 7.68 cellPamf: tres fallos reales corregidos (crash, NID ausente, layout)

Punto de partida: el frontend nunca pasa de su estado de arranque. Depurando el
hilo del demuxer con lldb sobre `build_dbg` aparecen tres defectos encadenados en
nuestro HLE de `cellPamf`, todos corregidos en `libs/codec/cellPamf.[ch]`:

1. **Crash por puntero no traducido.** `cellPamfStreamTypeToEsFilterId` escribia
   `id[0]`/`id[1]` sobre el EA del invitado como si fuera un puntero de host. El
   hilo `CPamfDemuxer::ReadThread` fallaba con `0xC0000005` escribiendo en
   `0xD0054E24`. Antes: una parada por excepcion en cada `continue` (3 de 3).
   Despues: **0 excepciones**.

2. **`cellPamfGetHeaderSize` sin implementar.** El NID `0xCA8181C1` salia como
   `[hle] unresolved NID`; sin handler, el HLE devolvia CELL_OK **sin escribir el
   parametro de salida**. El demuxer veia cabecera 0 y reintentaba:
   **379.897 aperturas de `title.pam` en 5 minutos**, con
   `PamfDemuxer::SeekAndReadRetry`. NID identificado calculando el nombre con
   `nid_database.compute_nid` (coincidencia exacta con el import del EBOOT).

3. **Layout de cabecera equivocado.** El parser usaba 0x08=dataOffset,
   0x10=tabla de streams, 0x14=numStreams y descriptores de 24 bytes. El formato
   real 4.0/4.1 usa 0x08/0x0C = header/data size **en unidades de 2048**, la
   secuencia en 0x50 y los streams en 0x88 con 48 bytes cada uno. Verificado
   contra el `title.pam` del disco: `PAMF`/`0041`, `1*2048 + 36108*2048 =
   73.951.232` = tamano real del fichero.

Implementado o reescrito (cubre las 10 funciones que MR7 importa de `cellPamf`):
`GetHeaderSize`, `GetHeaderSize2`, `GetStreamOffsetAndSize`,
`SetStreamWithTypeAndIndex`, `GetStreamTypeAndChannel`, `ReaderInitialize`
(acepta `fileSize = 0`, que es legal en streaming), `GetPresentationStartTime` /
`EndTime` (pts empaquetados hi/lo), `GetStreamInfo` (offsets reales y escritura
big-endian via VM), `GetNumberOfSpecificStreams` y `StreamTypeToEsFilterId`
(mapa MPEG-PS tipo/canal). Tabla de handlers: **1256** (antes 1251).

Verificacion: `clang -fsyntax-only` limpio, release y debug enlazan, y una
pasada de 100 s presenta 2.383 frames, 13 con geometria y **cero excepciones**.
Sin regresion.

### 7.69 El bucle que bloquea el arranque no es el video: es cellSaveData

Con el crash arreglado, una tanda de 5 minutos **no llega al video**. Se queda en:

```
[cellSaveData] AutoLoad(dir='BLJS10001GAME')
[cellSaveData] stat callback returned error -4
[cellMsgDialog] Open2('Cancelar carga de datos?')
[cellMsgDialog] Auto-responding: YES
```

336 iteraciones en 150 s. Es el bucle que ya describe el comentario de
`libs/system/cellSaveData.c` (sonda `SAVEDATA_NODATA_OK`): AutoLoad -> NODATA ->
"Cancelar carga de datos?" -> AutoLoad.

Medido (tandas de 150 s, salvo la ultima de 100 s):

| Configuracion | dialogos | presents | con draws |
|---|---|---|---|
| por defecto (YES) | 336 | 2.194 | **2.022** |
| `MSGDIALOG_ANSWER=n` | 336 | 2.194 | 2.022 |
| `SAVEDATA_NODATA_OK=1` | **1** | 3.035 | 14 |
| `SAVEDATA_NODATA_OK=1` (repeticion) | 1 | 2.383 | 13 |

Lectura:

- El bucle de save-data es lo que mantiene la pantalla "viva": dibuja el dialogo
  y la pantalla de carga (**92% de frames con geometria**). No es la escena del
  juego.
- Al cortarlo (`SAVEDATA_NODATA_OK=1`) se vuelve al estado ya conocido de
  ?7.64: ~13 de ~2.400 presents con draws, escena practicamente vacia.
- Por tanto hay **dos paradas**: el bucle de save-data y, detras, el frontend
  que no construye escena. El video (`cellPamf`) es una tercera, mas adelante.
- `MSGDIALOG_ANSWER` no cambia nada aqui: el boton no es la causa, el
  reintento es del titulo ante NODATA.

Siguiente paso recomendado: atacar el frontend con el bucle cortado
(`SAVEDATA_NODATA_OK=1`), que es el estado estable y reproducible.

### 7.70 Estado estable del frontend: no esta bloqueado, esta resolviendo rutas

Con el bucle de save-data cortado (`SAVEDATA_NODATA=cbr`, ver ?7.69 bis) el juego
queda en un estado reproducible: ~3.000 presents en 150 s y 19 con geometria.

**Correccion de la respuesta de guardado.** RPCS3, que si llega al titulo, cuando
el callback de estado del titulo devuelve `ERR_NODATA` muestra el dialogo "no hay
datos" y devuelve al juego **`CELL_SAVEDATA_ERROR_CBRESULT`**, no
`CELL_SAVEDATA_ERROR_NODATA`. Nuestro HLE devolvia NODATA, que MR7 trata como
reintentable (bucle de 336 vueltas). Anadido `SAVEDATA_NODATA=cbr|ok|err`
(`libs/system/cellSaveData.c`); `cbr` es el comportamiento del firmware y corta el
bucle (1 NODATA, 2 dialogos). No desbloquea la escena: 19 de 3.044 frames con
draws, igual que con el modo `ok`.

**Donde esta el hilo del frontend.** Adjuntando lldb al proceso vivo en ese estado
(`process attach`, `bt all`), el hilo `ppu Initialize Thread` (tid 11) esta
**ejecutando**, no esperando:

```
frame #1  func_0027A0CC          <-- tokenizador de rutas
frame #2  func_000366D4          <-- listener del frontend (g->[8])
frame #3  ps3_indirect_call_impl
frame #5  func_00034B8C
frame #6  func_0003833C          <-- Initialize Thread
frame #9  func_00037F88          <-- entrada del hilo
```

`func_0027A0CC` recorre la cadena de un objeto (+0x34/+0x38, longitudes u16,
caracteres de 6 bits + 0x40 y corte por `/`): es el resolutor de rutas ya
identificado en ?7.47, llamado desde el listener en cada iteracion de su lista
(+0x40/+0x44).

**Estado medido del pump** (`MR7_PUMP=1`, 210 muestras en 90 s):

| Campo | Valor | Lectura |
|---|---|---|
| `f0` | `00000000` (210/210) | la variable de salida del bucle nunca cambia |
| `f8` | `1042C020` (202/210) | el listener **si** existe (confirma la correccion de ?7.44) |
| `loader` | `head=0 tail=0 flag=0` | no hay trabajo en cola |
| `draw` | 13 slots no nulos (`107C4780`...) | la mesa de escena esta poblada |
| `ctl` | `put` avanza hasta `11DB0D54` y se congela | el hilo de dibujo deja de publicar |
| `file` | `expect=0 pending=0 eof=0` | el loader se considera terminado |

**Conclusion.** No hay ningun hilo bloqueado: el frontend cicla resolviendo una
ruta y no encola trabajo nuevo. El siguiente objetivo concreto es la entrada que
el listener pasa a `func_0027A0CC` (que tarea/tipo es y por que no produce una
peticion de carga), no el render ni el guardado.

Lanzador del estado estable: `port\run_frontend.cmd`.

### 7.71 El guardado funciona y la "ruta" del frontend no es texto

Dos datos de la ultima ronda, ambos medidos:

**El titulo guarda de verdad.** Tras varias ejecuciones con `SAVEDATA_NODATA=cbr`
aparecio en disco un guardado completo:

```
port\gamedata\dev_hdd0\home\00000001\savedata\BLJS10001GAME\
    GAME        1.048.576 B   (0x100000)
    ICON0.PNG      49.003 B
    PIC1.PNG      681.119 B
    PARAM.SFO       2.736 B
```

En la ejecucion siguiente el camino de carga lo encuentra y termina:
`funcStat -> cbResult.result=0`, `file op=0 ... -> 100000`, `LOAD complete for
'BLJS10001GAME'`. Es decir: la ruta de guardado del HLE escribe y relee datos
reales del titulo. No cambia el estado de la escena (sigue en ~11 frames con
geometria), pero retira el guardado de la lista de sospechosos.

**La entrada de `func_0027A0CC` no es una cadena ASCII.** El objeto del listener
(EA `0x1042C020`) tiene:

```
+0x00  vtable 0x101A4340
+0x30  indice actual = 0x66 (102)
+0x34  0x101A42B6   tabla de longitudes (u16)
+0x38  0x101A4188   datos
```

Al volcar los datos (`0x101A4188`) **no hay texto**: es una secuencia de pares de
2 bytes (`c0 f7 00`, `c1 f7 01`, `c2 f7 02`, `cf f0 0f` ...), un diccionario
comprimido/tokenizado. Consecuencia practica: volcar "la cadena" no sirve para
saber que fichero pide; hay que identificar el diccionario (probablemente el
propio TOC del `MR7.DAT`) o instrumentar el consumidor de la ruta, no el buffer.

Siguiente paso concreto: en el bucle del listener (`func_000366D4`), capturar el
indice de entrada (`+0x30`) y el resultado de `func_0027A0CC` en varias vueltas
para ver si repite la misma entrada (bucle) o avanza (trabajo real).

### 7.72 El resolver recorre una tabla de rutas, no una entrada atascada

Anadida una sonda al watchdog del pump (`MR7_PUMP=1`) que muestrea el estado del
resolver del listener cada 250 ms. 165 muestras en 60 s:

```
[pump]   resolver[idx=87  lens=0x101A42B6 chars=0x101A4188 cur=0x00000002 cap=0x00019D61]
[pump]   resolver[idx=102 lens=0x101A42B6 chars=0x101A4188 cur=0x00000010 cap=0x00066445]
...
[pump]   resolver[idx=102 lens=0x101A42B6 chars=0x101A4188 cur=0x00000010 cap=0x05A96519]
```

Lectura:

- `idx` (+0x30) **no esta fijo**: oscila entre 82 y 102 dentro de la misma tabla
  (`lens`/`chars` constantes). El frontend recorre una lista de rutas; no se
  queda atascado en una.
- `cur` (+0x40) toma dos valores, `2` y `0x10`.
- `cap` (+0x44) **crece de forma monotona** (~3 MB/s de equivalente) durante toda
  la ejecucion, sin relacion aparente con `cur`: parece un contador/posicion, no
  una capacidad de cola.

Conclusion: el frontend esta vivo y recorriendo su tabla de rutas en cada frame,
pero ese recorrido no se convierte en una peticion de carga (el `File Thread`
sigue ocioso y el `put` del GCM se congela). El siguiente dato util no es el
buffer de rutas sino **quien consume el resultado de `func_0027A0CC`**: si su
valor de retorno se descarta, ahi esta la transicion que falta.

### 7.73 EL FRONTEND ESPERABA UNA PULSACION: el juego entra al titulo

Este es el avance de la sesion. El frontend no estaba atascado en su maquina de
estados: **esperaba una pulsacion**. Con el bucle de save-data cortado y
`PAD_AUTOPRESS=15000:0x0008` (START) el juego cambia de estado:

```
[SYS] sys_ppu_thread_create tid=13 name="CPamfDemuxer::ReadThread" entry=0x004DD4B0
[fs]  open '/dev_bdvd/PS3_GAME/USRDIR/title.pam' -> fd 4
[fs]  read fd=4 nbytes=65536 -> 65536 (magic=50414D46)
[cellPamf] ReaderInitialize(addr=0x167FC190, size=0)
[cellPamf] PAMF v0041: 1 streams, header=2048 B, data=73949184 B, muxRate=120000
[cellDmux] QueryAttr -> memSize=65536
[cellDmux] Open(streamType=1) -> handle=0
[cellVdec] Open(codecType=0) -> handle=1 / StartSeq
[cellDmux] EnableEs(major=224, minor=57344)   <- video (0xE0)
[cellDmux] EnableEs(major=189, minor=48384)   <- audio (0xBD)
[cellDmux] SetStream(addr=0x110C3A10, size=65536)
[cellDmux] ES 0: AU_FOUND / ES 1: AU_FOUND
```

Es decir: **el PAMF del titulo se parsea con el lector corregido en ?7.68**, el
demuxer se abre, se habilitan los dos streams y se entregan AUs. El patron de
dibujo cambia de 11 frames con geometria a **649 frames con 4 draws** (video en
un quad + UI) de 2.250 presents en 90 s.

Tres defectos del HLE hubo que corregir para llegar aqui, todos verificados con
el fallo delante:

1. **`cellDmuxQueryAttr` (0xA2D4189B) y `cellDmuxQueryEsAttr` (0x02170D1A) no
   existian.** El juego pregunta cuanto buffer necesita antes de abrir el
   demuxer y asigna exactamente eso. Sin handler, el HLE contestaba CELL_OK con
   la estructura de salida intacta -> tamanos basura -> escrituras a
   `0x00000000` (`[null-write] guest wrote32 0x1 to 0x00000000 by
   guest-fn=0x002D14C0`) y **0xC0000005** al arrancar el video. Implementadas con
   tamanos sanos (64 KB demuxer, 32 KB por ES; el stub no escribe en ellos).

2. **Los callbacks de `cellDmux` se llamaban como punteros de host.**
   `cellDmuxSetStream` hacia `es->esCbFunc(...)` con `esCbFunc = 0xD0054F30`, que
   es un **OPD de invitado** -> `DEP violation` en `0xD0054F30` (capturado con
   lldb: `frame #0 0xd0054f30`, `frame #1 cellDmuxSetStream ... at
   cellDmux.c:246`). Ahora se guardan como EA y se invocan con
   `ps3_invoke_guest`, igual que `cellVdec`/`cellSaveData`.

3. **El guardado dejaba el titulo en bucle** hasta que se respondio como el
   firmware (?7.70): `SAVEDATA_NODATA=cbr`.

Queda un cuarto paso para VER el video: `cellVdec` sin
`PS3RECOMP_FFMPEG_RUNTIME` devuelve un cuadro negro, y el demuxer es un stub que
entrega el buffer entero como un unico AU (sin partir PES). Lo que se ve ahora
es el quad del video en negro mas la UI. Reproducible con
`port\run_title.cmd`.

#### Repeticion (metodologia de ?7.67): el arranque del titulo es estable, el dibujo sostenido no

Tres tandas con la misma configuracion, contando lo que no depende del tiempo:

| Tanda | `PAMF v0041` | `AU_FOUND` | presents | con draws |
|---|---|---|---|---|
| build_dbg, 90 s | 1 | 2 | 2.250 | **660** (649 con 4 draws) |
| build (release), 60 s | 3 | 4 | 1.912 | 11 |
| build_dbg, 60 s (START a 12 s) | 1 | 2 | 927 | 11 |

Lo que **si** es estable: el hilo del demuxer se crea, `title.pam` se abre, el
PAMF se parsea y se entregan AUs de video y audio. Eso ocurrio en las tres.

Lo que **varia**: cuantos frames siguen dibujando el video. Con el stub de
`cellVdec` (sin FFmpeg) el cuadro es negro y el juego puede quedarse esperando
frames que nunca llegan; si el primer AU cae en un momento u otro, el bucle
avanza mas o menos. No es un fallo nuevo, es el siguiente: **hace falta el
decodificador real**.

### 7.74 Demuxer real: MPEG-PS con AUs y PTS (y tres bugs de paso)

El stub entregaba el buffer entero como un unico AU (cabeceras incluidas), asi que
el decodificador recibia un blob y no producia nada. Reescrito en
`libs/codec/cellDmux.c`:

- **Parseo MPEG-PS**: pack header (0xBA), system header (0xBB), private 2 (0xBF) y
  PES (>= 0xBC). Por cada PES cuyo *stream id* coincide con un ES habilitado se
  encola un AU que apunta **dentro** del buffer del invitado, con su PTS extraido
  del header PES (marca 2/3).
- **Cola de AUs** por ES (hasta 32) y `cellDmuxGetAu` devuelve el **EA de un
  array `CellDmuxAuInfo[auCount]` en memoria de invitado** (32 B por entrada,
  layout del firmware). Antes escribia un **puntero de host** a traves del
  parametro de salida del invitado, que el titulo habria interpretado como EA.
- **Filter id**: los campos de `CellCodecEsFilterId` son cuatro `u32`, no bytes;
  se leian con offsets de byte y el id del ES nunca coincidia.

Tres fallos concretos corregidos en el camino, todos vistos en ejecucion:

| Fallo | Sintoma | Correccion |
|---|---|---|
| Se parseaba desde el offset 0 | el primer AU apuntaba a `0x93` del fichero, dentro de la **cabecera PAMF** | si el buffer empieza por `PAMF`, saltar `header_size * 0x800` |
| Stuffing del pack header leido en el byte 4 | desincronizacion tras el primer `0xBA` | el tama�o de stuffing esta en el **byte 13** (`& 7`) |
| `cellDmuxGetAu` devolvia puntero de host | el titulo leeria una direccion de host como EA | array en `VM_HLE_INJECT_BASE + 0xC00000` |

Verificado contra `title.pam`: la zona de datos empieza en 0x800 con
`00 00 01 BA` / `BB` / `BF`, y el primer AU cae en el offset 0x93 del buffer
(fichero 0x893) con `pts = 90000` (1 s). Por tanda se encolan **32 AUs**
(tope de la cola) desde un solo bloque de 64 KB.

**Lo que sigue**: aun con AUs correctos, el titulo abre el decodificador y lo
cierra al instante (`StartSeq -> EndSeq -> Close`, tres veces) y solo 11 frames
de 2.358 llevan geometria. El siguiente objetivo es la ruta `cellVdec`
(stub sin FFmpeg: devuelve cuadro negro) para ver por que el juego abandona la
secuencia.

### 7.75 FFmpeg dentro del port: el video se decodifica (y lo que falta para verlo)

**Runtime activado y funcionando.** Compilado con `PS3RECOMP_FFMPEG_RUNTIME=ON`
(headers de FFmpeg 8.0, `avconfig.h` generado a mano) y DLLs **compartidas que ya
estaban en la maquina**: las de OBS Studio (`avcodec-62.dll`, `avutil-60.dll` y
sus dependencias `swresample-6`, `libx264-164`, `zlib`). Arranque confirmado:

```
[ffmpeg] libavcodec 62.28.102 loaded
[cellVdec] decoding with libavcodec
```

**Tres bugs mas en la cadena demux->decoder, todos vistos en ejecucion:**

1. **Tamano del header PES**: se sumaban los 5 bytes de PTS **dos veces** (el PTS
   ya esta dentro de `header_data_length`), asi que cada AU empezaba 5 bytes
   tarde -> FFmpeg: `Invalid frame dimensions 0x0` en todos los cuadros.
2. **Firma de los callbacks**: `cellDmuxOpen` y `cellDmuxEnableEs` reciben un
   **puntero a `CellDmuxCb`/`CellDmuxEsCb`** (`{cbFunc, cbArg}`), no un par de
   registros. Leiamos el struct como si fuera la funcion y el `handle` de salida
   caia en el registro equivocado: los callbacks del titulo no se llamaban nunca
   y el reproductor abortaba (`StartSeq -> EndSeq`). Ademas el mensaje va por
   **puntero** (`CellDmuxMsg { msgType, supplementalInfo }`), no por valor.
3. **Granularidad de los AUs**: un PES de 2 KB es una *slice*, no un cuadro. El
   demuxer ahora **reensambla** los PES en un buffer por ES
   (`VM_HLE_INJECT_BASE + 0xA00000`) y solo entrega cuadros completos (hasta el
   siguiente `00 00 01 00`), con la cabecera de secuencia fusionada con el primer
   cuadro.

Tambien: `CellDmuxAuInfo` es de **0x28 bytes** con `auMaxSize`, `isRap` y
`pts`/`dts` como `{upper,lower}` (escribiamos 32 bytes con otro layout, de ahi un
`pts = UINT64_MAX`); y se anadio **`cellDmuxResetEs`** (0x21D424F0), que faltaba.

**Resultado medido**: con START, el titulo alimenta el decodificador con **~1000
AUs** en 75 s, un cuadro por AU (19-87 KB), PTS +3003 (29,97 fps) y **sin
mensajes de error de libavcodec**. El patron de dibujo pasa a **4 draws por
frame** (quad de video + UI).

**Lo que falta para VERLO**: la ventana sigue negra. La captura por
`PrintWindow` de un swapchain flip-model da negro y el volcado del backend
muestra el *backbuffer*, que tambien queda negro: el contenido del juego se
compone desde su propia superficie. El siguiente objetivo es el tramo
`cellVdecGetPicture` -> textura del RSX -> composicion, no el demuxer.

### 7.76 El video se decodifica y el juego lo recoge; falta el dibujo VP

**Cerrado el tramo demux->decoder.** El fallo era que al final de cada
`cellDmuxSetStream` la cola pendiente se movia al offset 0 del buffer de
reensamblado, **machacando los AUs recien publicados** que el titulo todavia no
habia leido: cada AU decodificaba como `AVERROR_INVALIDDATA` (-1094995529). Con
el movimiento eliminado (los offsets son absolutos y el siguiente `SetStream`
simplemente anexa detras) la cadena completa funciona:

```
[cellDmux] AU0[19401] bytes: 00 00 01 B3 50 02 D0 14 ...   <- con sequence header
[cellVdec] picture 1280x720 type=3 key=0
[cellVdec] GetPicture #1 (AUs 2) 1280x720 type=3 matrix=0 alpha=15 -> 0x134ADC80
```

- 0 errores de decodificacion (antes: todos).
- Cuadros 1280x720 entregados y **retirados por el juego** (`GetPicture`,
  formato 3 = YUV420 planar, matriz BT.601).
- El stream elemental extraido del propio `title.pam` decodifica igual con
  ffprobe (1280x720, 14 cuadros), lo que sirvio de referencia para aislar el bug.

**Lo que queda: el dibujo por vertex program.** El cuadro del video se pinta con
el camino VP del backend y ese camino **nunca se pudo usar** en MR7 (por eso
existia el parche `MR7_NO_VP`): al primer draw VP, la creacion de los tres PSOs
falla con `E_INVALIDARG` (0x80070057):

```
[VP] per-draw VS cached (hash=0x22BB5859, 3 instrs, slot 0)
[VP] PSO args rs=<valida> vs=2228 ps=776 il=16 rtv=28 dsv=45 topo=3 rt=1
[VP] atlas PSO hr=0x80070057
[VP] colour PSO FAIL (0x80070057)
[VP] PSO creation FAIL (0x80070057)
```

Comprobado que la firma del VS (ATTR0..ATTR15, xyzw) coincide con el input
layout, que la root signature se creo bien (`hr=0`) y que el dispositivo no esta
perdido (reason 0). Sin capa de depuracion (no disponible: 0x887A002D) el motivo
exacto no se imprime, asi que el siguiente paso es bisecar el
`D3D12_GRAPHICS_PIPELINE_STATE_DESC` (DSV/RTV, visibilidad de la root sig o el
input layout) hasta que el PSO se cree. Ese es hoy el unico bloqueo entre el
video decodificado y la imagen en pantalla.

Medido: con el video decodificando y sin `MR7_NO_VP`, 4 draws por frame; con
`MR7_NO_VP`, 11 frames con geometria en toda la tanda.

### 7.77 El camino VP vuelve a la vida (PSO de respaldo) y el video ya no se atasca

El bloqueo del ?7.76 era el **pipeline state del camino VP**: fallaba con
`E_INVALIDARG` incluso con shaders triviales. Bisecado con pruebas dentro del
propio backend (`CreateGraphicsPipelineState` sobre copias del desc, cambiando un
campo cada vez), el resultado fue:

| Prueba | Resultado |
|---|---|
| clon exacto del PSO que si se crea al arrancar | **OK** (el device esta bien) |
| VS del juego + PS trivial (solo SV_Position) | **OK** |
| VS del juego + PS que pide `TEXCOORD0` | **falla** |
| VS del juego + PS que pide solo `COLOR0` | **OK** |

Causa: el PS de atlas del backend muestrea `t0` en `TEXCOORD0`, y el **vertex
program del video de MR7 es minimo** (3 instrucciones, sin TEXCOORD), asi que el
enlace VS->PS no existe y D3D12 rechaza el PSO. Con el PSO rechazado se caia
**todo** el camino VP (por eso existia el parche `MR7_NO_VP`).

Correccion (`libs/video/rsx_d3d12_backend.c`): si el PSO texturizado no se puede
crear, se compila un PS **solo color** (`SV_Position` + `COLOR0`, que si enlaza)
y se usa como respaldo. Efecto inmediato:

```
[VP] colour-only fallback PSO hr=0x00000000
[VP] pipeline ready (3 instrs)
[FP] guest FP pipeline ready (fp=0x043E4C81)
[FP] guest FP pipeline ready (fp=0x043E2F01)
```

Es decir: ademas de desbloquear el VP, ahora **tambien se compilan los fragment
programs del juego** (antes ni se intentaba, porque el draw no llegaba).

**Estado del video con este cambio**: 399 `SetStream`, 352 `DecodeAu` y **134
`GetPicture`** en una tanda de ~40 s: el titulo consume el video a buen ritmo.
Pero **no se emite ni un `draw_arrays` despues de arrancar la pelicula** y el
juego deja lineas `DecodeFailed 83:82:75` en su propio log. Siguiente objetivo:
como pone MR7 el cuadro en pantalla (no usa geometria; probablemente
transfer/blit sobre la superficie de display) y por que su contador de AUs no
cuadra con el nuestro.

### 7.78 POR PRIMERA VEZ HAY PIXELES DEL JUEGO: el quad del video se dibuja

Con el PSO de respaldo del ?7.77 el camino VP arranca, y eso desbloquea tambien
los fragment programs del juego. Resultado visible en el volcado de frames:

```
frame_010.png  47.878 B   (antes: 4.349 B = plano)
```

La imagen muestra un **rectangulo rojo de 720x480** sobre el fondo oscuro, en la
zona exacta del video: es el quad del titulo, dibujado con el pixel shader del
propio juego. Ya no es "no se dibuja nada": hay geometria y color en pantalla.

**Lo que falta para que ese rectangulo sea el video**: no hay textura enlazada en
la unidad 0 en el momento del draw.

```
[VPDRAW] #0 is_vp=1 fp=0x43E4C81 tex0=0x0 set=0 dpso=1
[VPDRAW] #1 is_vp=1 fp=0x43E2F01 tex0=0x0 set=0 dpso=1
```

`fp` es la direccion del fragment program del juego y `dpso=1` quiere decir que
se esta usando **su** pipeline (no el de respaldo): el muestreo esta en el
shader, pero `tex[0]` esta sin fijar, asi que no hay nada que muestrear y sale el
color plano del shader.

Siguiente objetivo: localizar donde enlaza MR7 la textura del cuadro decodificado
(`cellGcmSetTexture` / metodos NV4097 de textura en el FIFO) y por que nuestro
capturador de estado no la ve en el draw. Con eso, el rectangulo deberia pasar a
ser el video del titulo.

### 7.79 El enlace de textura se pierde entre el walker y el backend

Con la traza `[TEXM]` en `process_texture_method` (`rsx_commands.c`) se ve que el
titulo **si enlaza** la textura del cuadro antes de cada draw:

```
[VPDRAW] #1 is_vp=1 fp=0x43E2F01 tex0=0x0 set=0 dpso=1
[TEXM] unit=0 off=0x10000            <- y despues vuelve el draw
```

`0x10000` es exactamente el RT 1280x720 donde el juego sube el cuadro. Pero el
registro del draw sigue con `tex0=0x0 set=0`, y con `TEXFMT_ALL=1` **solo** salen
los reseteos de arranque (fmt=0, off=0): `d3d12_bind_texture` no se invoca nunca
con la textura real.

El camino es: `process_texture_method` marca `state->textures[0].dirty` -> en el
siguiente `begin/end` de `rsx_commands.c` se hace flush y se llama a
`backend->bind_texture` -> `d3d12_bind_texture` filtra por formato y copia a
`s_d3d.cur_texs` -> el registro del draw copia `cur_texs`. Como el backend no ve
el bind, `cur_texs` queda a cero y el FP del juego muestrea nada (color plano).

Siguiente paso: averiguar por que ese flush no llega a `bind_texture` en los
draws del video (el `dirty` se limpia antes, o estos draws entran por otra ruta,
p. ej. el motor `rsx_live_draw`), y asegurar que `cur_texs` se rellena antes del
registro.

### 7.80 El registro del draw del video nace por otra ruta

Se intento tomar la textura del estado vivo del walker en el registro de draws VP
(`rsx_d3d12_backend.c`, copia de `cur_texs`). La traza `[REC]` puesta en ese punto
**no se ejecuto ni una vez**, asi que los draws del video no se crean ahi: usan
otra ruta de registro (probablemente el motor `rsx_live_draw`), que tiene su
propia copia de estado de textura.

Confirmado por otro lado que el estado es el mismo objeto (`state=` y `st=` son
el mismo puntero) y que el bind llega **despues** del primer registro y antes del
siguiente; lo que falla es que el registro que se usa no consulta ese estado.

Siguiente paso: localizar la creacion de `D3D12DrawRecord` en la ruta que usan
estos draws (buscar el segundo sitio que rellena `dr->tex[]`) y hacer que consulte
`current_rsx_state->textures[u]`, igual que se intento en la ruta VP.

### 7.81 La textura llega al draw; el cuadro sigue negro

Parcheadas las **tres** rutas que crean `D3D12DrawRecord` (`rsx_d3d12_backend.c`)
para que tomen la textura del **estado vivo del walker** cuando `cur_texs` esta
vacio. El draw del video pasa de `tex0=0x0 set=0` a:

```
[VPDRAW] #1 is_vp=1 fp=0x43E2F01 tex0=0x11110000 set=1 dpso=1
```

`0x11110000` es la vista IO del RT `0x10000`. A partir de aqui el usuario ve
**negro** en vez del azul de clear: el quad ya se compone, pero muestrea negro.

**Nota operativa importante**: varias tandas "fallidas" de esta sesion eran el
**ISO desmontado solo** (`G:` vacio -> `open FAIL MR7.DAT` en bucle). Hay que
remontarlo antes de medir; explica buena parte de la "intermitencia".

Lo que queda por comprobar: si el cuadro que `GetPicture` escribe en
`0x134ADC80` llega de verdad a la superficie `0x10000` (el juego debe copiarlo,
probablemente con el motor NV3089 de `cellGcmSys.c`, que es una copia de CPU) y
si nuestro resolutor de texturas lee esa misma memoria. Si la superficie esta a
cero en el momento del draw, el negro es la textura vacia, no un fallo de
composicion.

### 7.82 Tama�o de la textura del video: IMAGE_RECT vacio -> usar el RT

El juego programa `SET_TEXTURE_IMAGE_RECT = 0x00000004` (0x4 = ancho 0, alto 4)
para la textura del video, asi que el muestreo era degenerado y el quad salia
negro. El offset que enlaza (`0x10000`) **si** corresponde a un RT conocido
(`offscreen RT 1: off=0x10000 1280x720`), asi que en las tres rutas de registro
se a�ade un respaldo: si `IMAGE_RECT` da ancho o alto cero, se toma la geometria
del RT con ese offset (`off_rt_find`).

Resultado en el registro del draw:

```
[VPDRAW] #1 ... tex0=0x11110000 set=1 ... 0x4 fmt=0xA5        (antes)
[VPDRAW] #1 ... tex0=0x11110000 set=1 ... 1280x720 fmt=0xA5    (despues)
```

Nota: el conteo de bytes no nulos en memoria del invitado da 0 porque ese RT vive
en un recurso D3D12, no en RAM del invitado; el backend debe muestrear el recurso
del RT (camino `tex_rt`), que es lo siguiente a verificar en pantalla.

### 7.83 Composicion forzada: la superficie del video esta vacia

Probado `MR7_FORCE_COMPOSITE=1` y un conmutador nuevo `COMPOSITE_OFF=<hex>` que
presenta cada frame una superficie concreta (`0x10000`). Resultado: los frames
volcados siguen planos y la distribucion de draws es

```
draws=0: 1395     draws=1: 1     draws=2: 10
```

O sea: el quad del video se dibuja en **unas 10 ocasiones** en toda la tanda (no
por frame), y las superficies alcanzables estan vacias. No es un fallo de
composicion: **no hay nada que componer**.

Junto con las 134 llamadas a `GetPicture` (el juego consume el video), esto dice
que el titulo decodifica pero su bucle de presentacion no llega a dibujar la
pelicula por frame. Siguientes lineas de trabajo:

1. Instrumentar el **flip / scanout** (`cellGcmSetFlip`, `SCANOUT_FB_DBG`,
   `GCM_GUEST_FB`) para ver a que buffer de display voltea el juego y si nuestro
   scanout lo lee.
2. Capturar un frame con RenderDoc (`RENDERDOC_DLL`, ya soportado por el
   backend): no esta instalado en la maquina, asi que habria que bajarlo.

### 7.84 El titulo no voltea y apenas dibuja el video

Medido con el limite de trazas subido y `GCM_FLIPCOUNT=1`:

- `VPDRAW` (draws del quad del video): **11 en toda la tanda**, no por frame.
- `[flipword]`: **0**. El invitado **no envia ningun flip** al FIFO.
- Los ~2.000 `[PRESENT]` vienen de nuestro lado (vblank), no del juego.

Es decir: el reproductor del titulo decodifica (134 `GetPicture`) pero **no
publica frames**: dibuja el quad unas pocas veces y nunca pide voltear. La ventana
azul es el clear de nuestros presents.

Siguiente objetivo: por que el bucle de dibujo del reproductor corre tan pocas
veces. Candidatos: espera de un evento de vblank/flip que no entregamos, o de la
finalizacion de un flip que nunca ocurre (nuestro `cellGcmSetWaitFlip`). Hay que
instrumentar `cellGcmSetWaitFlip`/flip completion y el hilo que dibuja.

### 7.85 La carga sigue parada en 46 MB aunque el video arranque

Medido con `FS_READ_ALL=1` en una tanda donde el video **si** arranco
(`pamf=3`):

```
reads fd=3 = 70     bytes = 45.957.120
```

Practicamente identico al corte de sesiones anteriores (69 lecturas,
45.955.072 B): el arranque del video **no** genera la carga siguiente. RPCS3 lee
77 MB antes del titulo, asi que faltan ~31 MB de assets.

Eso explica lo que se ve: la pantalla de carga dibuja un quad 1280x720 con una
textura vacia porque **los datos de esa imagen no se han cargado**; el juego no
apunta nunca al display buffer de 1920x1080 (solo configura 720x480 y 1280x720) y
por eso la ventana queda en el color de clear.

Siguiente paso (sin cambios respecto a ?7.84, ahora con la medicion confirmada):
encontrar que evento encola la **peticion 71** en el gestor de carga del
frontend (`func_00015148`, lr 0x173E8) despues de consumir las 56 ordenes del
arranque.

### 7.86 El gestor de carga: consumidor identificado, productor pendiente

El paso por frame del gestor de carga es `func_0001723C` (llamado desde el
frontend). Su logica, leida en el codigo liftado:

```
obj = *(toc-0x7F98)                       ; el gestor
func_000170AC(obj)                        ; actualiza estado
if (*(u8*)(obj+0x683C) > 1) -> loc_000173EC   ; rama de cierre/otra fase
head = *(u32*)(obj+0x6800)
tail = *(u32*)(obj+0x6804)
if (tail > head) -> salir                  ; nada pendiente
idx  = head * 0x34                         ; entrada de 0x34 bytes
...
if (condicion de la entrada) -> loc_000173C4:
        *(u32*)(*(toc-0x7FF0)) = *(u32*)(*(toc-0x7FE4))
        func_00015148(...)                 ; rellena y encola la peticion
else -> salir
```

Es decir: el consumidor funciona y sale limpiamente cuando la cola esta vacia
(`head == tail`), que es el estado en el que se queda tras las 56 ordenes del
arranque. Lo que falta es **quien escribe el tail (`obj+0x6804`)**: no aparece
como literal en el codigo liftado (se escribe a traves de un puntero calculado),
asi que la forma de encontrarlo es una **watch de escritura** en caliente:

```
PPU_WWATCH=<EA de obj+0x6804>   (con PPU_WWATCH_LEN=4)
```

Ese EA sale del propio log: `obj = *(toc-0x7F84)-0x8000` (el watchdog del pump ya
lo imprime como `loader[obj=0x........]`). Con la watch activada, el productor
que deberia encolar la peticion 71 aparecera con su `cia`/`lr`.

### 7.87 Quien escribe la cola: func_00017840 (arranque) y func_00016EEC (nunca se llama)

Escritores del tail (`obj+0x6838`), localizados por literal en el codigo liftado:

| Funcion | Papel | Llamada |
|---|---|---|
| `func_000164CC` | inicializa head=tail | si (arranque) |
| `func_00017840` | **encola** (dos escrituras) | **si**, desde 0x11124/0x1143C/0x11468/0x115A4/0x115E4/0x1F3F0 (codigo de init) |
| `func_000175D0` | encola | desde 0x1F350 (init) |
| `func_00016EEC` | encola | **nunca** (breakpoint sin disparar en una tanda completa) |

Es decir: las 56 ordenes del arranque entran por `func_00017840`, y el encolador
"de despues" (`func_00016EEC`) **no se invoca en toda la ejecucion**. El
consumidor (`func_0001723C`) no falla: se queda sin trabajo.

Siguiente paso: `PPU_WWATCH` sobre el tail real. El EA sale del watchdog del pump
(`MR7_PUMP=1`, linea `loader[obj=0x...]`) sumando 0x6838; con la watch se vera si
alguien escribe el tail con un puntero calculado (que es la forma en que un
encolador indirecto lo haria sin dejar literal en el codigo).

### 7.88 La watch identifica escritores, pero el cargador visible no es el de las 56 ordenes

`PPU_WWATCH=102AC134` (tail del objeto que imprime el watchdog del pump,
`obj=0x102A58FC`; head en +0x6834 = 0x102AC130, tail +0x6838 = 0x102AC134, flag
+0x683C = 0x102AC138) funciona y nombra al escritor:

```
[ww] 0x102AC134 <- 0x1 (w4) guest-fn=0x00017840 lr=0x00017A54 tid=1
[ww] 0x102AC138 <- 0x1 (w1) guest-fn=0x0001723C lr=0x000172A8 tid=1
[ww] 0x102AC130 <- 0x1 (w4) guest-fn=0x0001723C lr=0x000173F0 tid=1
```

Pero en 60 s **solo hay un encolado** (`func_00017840` escribe tail=1) y un
consumo. Es decir: el objeto que vigila el watchdog del pump **no es** el que
gestiona las 56 ordenes del arranque; esas van por otro objeto/cola.

Siguiente paso: armar la watch sobre el objeto que recibe `func_00017840` (su r3
en el unico hit conocido, lr 0x17A54) y sobre el objeto de peticion de fichero
`0x10290640`, para localizar la cola que si mueve los 46 MB y ver si su
productor se detiene por una condicion concreta.

### 7.89 La cola del cargador llega a 57 y se queda vacia

El watchdog del pump muestra la evolucion real de la cola del cargador
(`obj=0x102A58FC`, head `+0x6834`, tail `+0x6838`):

```
h=0  t=0  -> h=1 t=1 -> h=2 t=3 -> h=16 t=16 -> h=21/23/... -> h=57 t=57
```

Es decir: **57 ordenes encoladas y consumidas** (una mas que en sesiones
anteriores), y ahi se queda: `head == tail == 57`, `flag=0`, contadores
`+8824/+8828` a cero.

El encolador es `func_00017840` (visto escribiendo el tail en la watch). Un
breakpoint con cuenta ignorada (`-i 56`) para capturar la orden 57 **no llego a
dispararse en ~4 min bajo el depurador**: la ejecucion depurada es mucho mas
lenta y no alcanza ese punto en la ventana de prueba.

Estado: la carga se detiene porque el productor deja de encolar tras la orden 57;
el consumidor funciona. La siguiente sesion deberia o bien ampliar la ventana del
depurador, o anadir un contador en el runtime para el encolador (sin depurador).

### 7.90 Video decodificado LIMPIO: el boton era CROSS y los AUs necesitaban arena

Tres correcciones que, juntas, hacen que el port decodifique video real de MR7 con
imagen limpia:

1. **El boton del titulo es CROSS, no START.** Con `PAD_AUTOPRESS=...:0x0008`
   (START) la pelicula no arrancaba en la mayoria de las tandas (mas de 120 s sin
   `PAMF`); con `0x4000` (CROSS) arranca en 35-70 s de forma consistente. Esa era
   una de las causas de la intermitencia que veniamos persiguiendo.
2. **Cada AU tiene su propia arena.** Los AUs apuntaban al buffer de
   reensamblado, que se reutiliza y se reinicia al llenarse: los cuadros que el
   titulo aun no habia decodificado se machacaban y salian bloques grises. Ahora
   cada cuadro se **copia** a una arena por ES
   (`VM_HLE_INJECT_BASE + 0xE00000 + es*1 MB`, 4 slots de 256 KB).
3. **Slot de 256 KB** (antes 128 KB): un cuadro 720p llega a ~130 KB y el clamp
   lo truncaba (bandas corruptas). Con 256 KB x 4 slots la imagen sale entera.

Prueba: `MR7_VDEC_BMP=1` escribe el cuadro convertido a `vdec_frame.bmp`. El
resultado (`port\dumps\vdec_frame4.png`, `MR7_frame_real.png`) es una escena real
del video (garaje oscuro con coche, pilares y escaleras), sin bloques de
corrupcion.

Pendiente de validacion visual: la presentacion en ventana con la sonda
(`MR7_VDEC_TO_FB=1` + `GCM_GUEST_FB=0,1920,1080,8192`), que necesita que la
pelicula arranque (ahora con CROSS).

### 7.91 Con la pelicula en marcha el juego sigue dibujando solo 11 veces

Medido en tres tandas con CROSS y `PAMF` presente (el video arranca):

| Tanda | `SetStream` | AUs | `draw_arrays` | `VPDRAW` |
|---|---|---|---|---|
| d1 | 667 | 524 | **11** | **11** |
| cz | 420 | 339 | **11** | **11** |

Es decir: el reproductor **decodifica a ~7,4 bloques/s** (500+ AUs) pero el
renderer solo recibe **11 draws** en toda la tanda y el invitado **no envia ni un
flip**. La decodificacion y el dibujo estan desacoplados: el juego consume el
video pero no lo publica.

Siguiente linea de trabajo: instrumentar el ciclo de flip/vblank del titulo
(`FLIP_DBG=1` ya registra `cellGcmResetFlipStatus`/`SetWaitFlip`/tick) para ver si
el bucle de dibujo esta esperando una finalizacion de flip que nunca ocurre.

### 7.92 Presentacion de la sonda: el cuadro va en un bufer del host

El volcado de la sonda al framebuffer del invitado no se veia: la ventana salia
**blanca**. La causa medida: el titulo **sobrescribe su propio display buffer con
blanco** cada frame, despues de nuestro volcado y antes del present.

Correccion: `cellVdec` guarda el ultimo cuadro convertido (BGRA 1920x1080) en un
**bufer del host** (`ps3_vdec_fb_bgra` / `ps3_vdec_fb_valid`) y el presentador de
framebuffer del invitado (`rsx_d3d12_backend.c`) usa ese bufer como fuente cuando
esta disponible, en vez de leer la memoria del invitado. Asi nada lo pisa.

Es una sonda de depuracion (`MR7_VDEC_TO_FB=1` + `GCM_GUEST_FB`), no la via final,
pero demuestra el tramo completo decode -> conversion -> presentacion.

### 7.93 El titulo no publica por ninguna via conocida

Con la pelicula en marcha (CROSS), se comprobo una a una todas las vias por las
que un titulo puede sacar imagen, y **ninguna se usa**:

| Via | Resultado medido |
|---|---|
| API de flip (`cellGcmSetFlip`/`SetPrepareFlip`/`SetFlipCommand`) | solo `SetFlipMode(mode=2)` al inicio; **0 flips** |
| Motor 2D NV3089 (blit al display buffer) | **0 blits** (`[NV3089]` nunca imprime) |
| Escritura de CPU al framebuffer (`0xC0000000`) | **11 stores, todos 0x0**, al inicializar; nunca mas |
| Draws al display buffer | solo 720x480 y 1280x720 intermedios; **nunca 1920x1080** |
| Draws del cuadro de video | **11 en toda la tanda** (con 1.594 `DecodeAu` y 2.090 `SetStream`) |

Conclusion: el reproductor **decodifica pero no publica**; su bucle de dibujo no
corre por frame. La unica via visible hoy es la sonda de depuracion
(`MR7_VDEC_TO_FB` + `GCM_GUEST_FB`), que presenta el ultimo cuadro desde un bufer
del host y ya usa ese bufer (`[GUESTFB] source=vdec-frame` cuando hay cuadro).

Siguiente paso (cuando se retome): instrumentar el bucle del reproductor
(`func_000E6B10`/`func_000E6CBC` y el hilo que dibuja) para ver que espera entre
cuadro decodificado y cuadro dibujado.

### 7.94 La build de release no tenia FFmpeg (y por eso tus pruebas salian vacias)

Descubierto al comparar builds: `build\` (release, la que usan los lanzadores) se
configuro **antes** de activar el runtime de FFmpeg, asi que
`PS3RECOMP_FFMPEG_RUNTIME=OFF` y el decodificador era el stub: `GetPicture` no
pasaba por `vdec_take_picture` y la sonda no volcaba nada.

En la release se veia:

```
pamf=2   ffmpeg=0   vdecprobe=(nada)   sin BMP
```

Tras reconfigurar `build\` con `-DPS3RECOMP_FFMPEG_RUNTIME=ON
-DPS3RECOMP_FFMPEG_INCLUDE_DIR=.../ffmpeg_inc` y recompilar:

```
pamf=2   ffmpeg=3   [vdecprobe] push=1 w=1280 h=720   BMP=2.764.854
```

Es decir: todas las pruebas del usuario con los lanzadores (`run_title.cmd`,
`run_video_test.cmd`, `run_video_retry.cmd`) estaban condenadas a pantalla vacia
porque esa build no decodificaba video. Las mediciones buenas de esta sesion se
hicieron con `build_dbg`.

Leccion de higiene: **las dos builds deben configurarse igual**; conviene dejar
anotado el comando de configuracion de cada una.

### 7.95 El video del titulo se decodifica: se lee el logo RIDGE

Prueba definitiva del decodificador: `MR7_VDEC_BMP=1` guarda el cuadro que el port
extrae del `title.pam`, y en el aparece la **pantalla de titulo de Max Racer 7**
con el logo **RIDGE** y las lineas de velocidad
(`port\dumps\MR7_title_RIDGE.png`). Hay bloques de corrupcion en zonas de
movimiento (faltan slices en algun corte), pero el contenido es inequivoco.

Es decir: **demux MPEG-PS + FFmpeg + conversion YUV->RGB funcionan** con material
real del juego.

Lo que sigue sin funcionar es **llevarlo a la ventana**: con `GCM_GUEST_FB` el
presentador registra `[GUESTFB] USING VDCE FRAME` (usa el bufer del host) pero la
ventana aparece blanca, asi que el fallo esta en la copia D3D12 del camino
`guest_fb_present` (barrera/estado del backbuffer) o en el momento del present,
no en el contenido.

### 7.96 La copia del camino GCM_GUEST_FB no llega al swapchain; se pasa al scanout

Prueba con `MR7_GFX_RED=1` (pinta el cuadro de rojo solido por el mismo camino de
copia de `guest_fb_present`): la ventana **sigue blanca**. Es decir, la
`CopyTextureRegion` de ese camino no llega al backbuffer presentado (barrera o
recurso distinto), no es un problema del contenido.

Se cambia de estrategia: **usar el scanout normal** (`scanout_auto_present`), que
en otros titulos ya presenta correctamente desde memoria del invitado. La sonda de
`cellVdec` escribe ademas el cuadro en el framebuffer del invitado
(`0xC0000000`, 1920x1080, pitch 8192), y el juego **no lo borra** (solo 11 stores
a cero al inicializar).

Requisito detectado: el scanout solo presenta si
`cellGcm_current_scanout()` devuelve un buffer registrado; con los primeros
frames devuelve 0 (aun sin `SetDisplayBuffer`), pero despues el juego registra los
buffers 0 y 2, asi que en estado estable si hay destino.

Prueba en curso: `run_video_verify6.ps1` (identico a verify5 pero **sin**
`GCM_GUEST_FB`), con la sonda activa y captura de la ventana.

### 7.97 ARREGLADO: la ventana ya muestra la pelicula de titulo

**Causa raiz del blanco sistematico**: `guest_fb_present()` copiaba el cuadro
decodificado (1920x1080) al backbuffer (1280x720) con `CopyTextureRegion` **sin
box**. Esa llamada es invalida (la caja de destino seria mayor que el recurso) y
**elimina el dispositivo D3D12**: a partir de ahi *todo* el frame falla, incluido
el `ClearRenderTargetView` rojo de la prueba `MR7_GFX_RED=2`. Por eso daba igual el
contenido: la ventana se quedaba con el color de clear (0,0,0.1 -> el "azul
oscuro" de una tanda y el blanco de otra). El propio `scanout_auto_present` ya
tenia el aviso escrito en un comentario ("an invalid call and removes the device"),
pero el camino `GCM_GUEST_FB` no lo aplicaba.

**Arreglo**: el presentador ahora escala en CPU (vecino mas cercano) al tamano del
backbuffer y copia una caja valida:

```c
const u32 dw = gw < s_d3d.width  ? gw : s_d3d.width;    /* 1280 */
const u32 dh = gh < s_d3d.height ? gh : s_d3d.height;   /*  720 */
...
sl.PlacedFootprint.Footprint.Width  = dw;
sl.PlacedFootprint.Footprint.Height = dh;
```

El bufer de subida se sigue dimensionando para el caso peor (`gw x gh`, pitch de
`gw`), asi que un cambio de tamano de ventana no reasigna nada: solo cambia
cuantas filas se rellenan. `scanout_auto_present` recibio el mismo tratamiento
(escala en vez de recortar) y ahora elige el orden de canales segun la fuente
(host BGRA vs invitado big-endian A,R,G,B).

**Resultado** (`port\dumps\MR7_TITLE_ON_SCREEN.png`): la ventana muestra
`RIDGE STATE - THE HOME OF DRIFT` con las marcas de derrape, a 31 FPS,
`draws: 0` (lo pinta la sonda). Log:

```
[GUESTFB] USING VDCE FRAME
[GUESTFB] copy 1280x720 -> backbuffer 1280x720
```

**Evidencia de metodo**: para capturar la ventana hubo que ponerla *topmost*
(`SetWindowPos(HWND_TOPMOST)`) y ocultar la consola; `SetForegroundWindow` solo no
basta y por eso varias capturas anteriores mostraban el escritorio y no el juego.

**Lanzador**: `port\run_video_retry.cmd` ya monta la ISO si falta, arranca con
CROSS, activa la sonda y **deja la ventana abierta** hasta que la cierres.

### 7.98 Estado del render propio tras el arreglo de presentacion

Con la presentacion ya funcionando (�7.97) se remidio el camino del juego **sin
sonda** durante 130 s. Lo que hay:

| Medida | Valor | Lectura |
|---|---|---|
| `PAMF v0041` | 3 | la pelicula arranca y se repite |
| `SetStream` | 660 | el demuxer HLE recibe datos |
| `DecodeAu` | 530 | se decodifican AUs |
| `GetPicture` | 6 | **ojo: es el tope de lineas de log, no el numero real** (el contador no se imprime) |
| `[VPDRAW]` | 11 | dibujos de quad (3 vertices) |
| `SetDisplayBuffer` | 2 | buffers 0 y 2 (1920x1080, pitch 8192) |
| `SetFlip` reales | **0** | el juego nunca voltea |
| `NV3089` | 0 | ningun blit |
| `[spu] stop 0x110` | `CELL_ESRCH` (clave `0x6E754153`) | el hilo bnusCore pide una cola que no existe |

Instrumentacion nueva y resultados:

1. **Ticker de vblank vivo** (`GCM_TICK_DBG=1`): `vblank=3598` en 60 s (~60 Hz) y
   **3.300 entregas** del handler del juego (`[gcm-tick] vblank handler
   DELIVERED`). El *pacing* de frames funciona; no es el bloqueo.
2. **Cero trabajos SPU despachados**: con `SPU_JOBEXEC=1` no aparece ni un
   `[jobexec] DISPATCH` en 75 s.
3. **Histograma de metodos RSX vacio**: con `GCM_METHOD_HIST=1` la linea
   `[mhist]` sale sin entradas (el formato imprime `"[mhist]"` y luego rompe el
   bucle con `!bc`), es decir el walker del FIFO **no despacha metodos** en
   regimen estable. Los 11 dibujos son llamadas HLE directas
   (`[RSX] DRAW_ARRAYS prim=5 count=3`), no comandos de buffer.
4. **Causas descartadas**: `DecodeAu` HLE siempre devuelve `CELL_OK` (los
   `DecodeFailed HH:MM:SS` que imprime el juego son su propia logica, 176 en
   130 s), y no hay NIDs sin resolver salvo `cellUsbd 0x7FE92C54`.

Conclusi�n: el frame negro no es un fallo del backend ni del presentador. El
juego se queda en **inicializacion temprana**: su hilo Draw espera trabajo
(`sem=5`), el hilo principal cicla pad -> sync -> pad, y el servidor SPU
`bnusCore` no produce resultados porque su `stop 0x110` recibe `CELL_ESRCH`
(la cola `"nuAS"` no existe: solo hay 7 colas, ninguna con esa clave).

Siguiente frente (RE de protocolo, no backend): resolver el servicio de
`receive_event` de la ruta liftada para el protocolo `bnusCore`, o encontrar
quien deberia crear la cola `"nuAS"`. Mientras tanto, la imagen del juego se ve
por la sonda de `cellVdec` (�7.97).

### 7.99 Corrupcion del video resuelta: era la cadena de referencias

El cuadro presentado salia a bloques (macrobloques cambiados, logo partido). No
era el demuxer ni el color: era **el rechazo de AUs**.

`cellVdecDecodeAu` devolvia `CELL_VDEC_ERROR_BUSY` en cuanto la cola de cuadros
llegaba a `VDEC_BUSY_AT` (16), con la idea de que el titulo debia recoger
cuadros. Pero MR7 casi nunca llama `GetPicture` (6 veces frente a 530
`DecodeAu`), asi que **la mayoria de los AUs se descartaban**: el decodificador
perdIA los cuadros de referencia P/B y todo lo posterior salia como basura de
macrobloques.

Arreglo (solo con la sonda activa, `MR7_VDEC_TO_FB=1`, para no cambiar el
comportamiento por defecto): en vez de rechazar el AU, se **tira el cuadro mas
antiguo** de la cola y se sigue decodificando. La cadena de referencias queda
intacta; `GetPicture` sigue entregando lo mas reciente. Sin la variable se
mantiene la respuesta de hardware (`CELL_VDEC_ERROR_BUSY`).

Resultado (`port\dumps\MR7_CLEAN_FRAME.png`): el cartel sale **nitido** -
`RIDGE STATE / THE HOME OF DRIFT`, las flechas rojas y el texto pequeno legible,
sin macrobloques.

Leccion: cuando el consumidor de un titulo esta parado, descartar AUs no es
inocente; rompe la decodificacion entera. Descartar cuadros decodificados es la
opcion que conserva la imagen.

### 7.100 Presentacion eficiente del cuadro y estado visual final

El presentador reconvertia 1920x1080 -> 1280x720 **en CPU en cada frame**
presentado (2,07 Mpx de escala por cuadro), aunque el cuadro no hubiera
cambiado: la ventana se quedaba en ~14-18 fps.

Ahora `vdec_push_guest_fb` incrementa `ps3_vdec_fb_seq` al publicar un cuadro
nuevo, y `guest_fb_present` solo reescala cuando la secuencia cambia. La copia
D3D12 se sigue emitiendo para cada backbuffer del swapchain (con 2-3 buffers el
contenido tiene que estar en todos), pero el coste de CPU desaparece cuando el
video no avanza.

Resultado medido: **22 fps** con `draws: 0`, y el cartel se ve nitido y
completo (`port\dumps\game_window.png` en esta tanda):

```
RIDGE STATE
THE HOME OF DRIFT RACING >>>>>>>
```

Estado del titulo hoy:

| Aspecto | Estado |
|---|---|
| Decodificacion del `title.pam` | correcta (demux MPEG-PS + FFmpeg + YUV->RGB) |
| Presentacion en ventana | correcta, escalada y sin corrupcion |
| Audio (`cellAudio`) | iniciado por el juego; sin comprobar salida |
| Render 3D propio del juego | **bloqueado** en inicializacion temprana (�7.98) |

### 7.101 Mapa exacto del circuito de eventos que falta (bnusCore)

Medido con `PS3_EVT_RECV`/`[evt]` y el desensamblado del hilo que espera:

```
[evt] port_create id=5 type=1 name=0x0
[evt] port_connect_local(port=5 -> q=6)
[evt] port_create id=6 type=1 name=0x0
[evt] port_connect_local(port=6 -> q=7)
[evt] port_send(port=5 data=0x53544154/0x0/0x0)   <- "STAT", UNA sola vez
[SPU] thread_connect_event tid=0x2000 queue=0x4 et=1 spup=3
[SPU] thread_connect_event tid=0x2000 queue=0x5 et=1 spup=1
[SPU] thread_connect_event tid=0x2000 queue=0x7 et=1 spup=7
```

El hilo PPU que se queda esperando es `func_003971EC` (bnusCoreUpdateThread):
envia `"STAT"` a `port 5` una vez y despues hace

```
sys_event_queue_receive(q=[r31+0x34]=7, evt=&buf, timeout=0x14B4 = 5300 us)   // syscall 0x82
```

en bucle: es un **poll de estado cada 5,3 ms**. `port 6 -> q=7` es el camino por
el que el SPU deberia contestar (el SPU esta enlazado a esa cola con `spup=7`),
y **`port 6` nunca envia nada**.

Al otro lado, el SPU (`bnusCore`, imagen 280400854) esta:

- `RUNNING` todo el rato (no ha terminado: no hay linea `[SPU] tid=... stopped`),
- girando en `pc=0x08E78 / 0x060A8 / 0x0783C` (fallbacks `[spu-smc]`),
- tras **una sola** escritura a OutMbox y **un solo** `stop 0x110` con clave
  `"nuSA"` (0x6E754153), a la que el runtime responde `CELL_ESRCH` porque esa cola
  no existe (solo hay 7 colas: claves 0,0,0x6E754149,0,1,0,0).

Es decir: **el SPU espera trabajo de una cola que el juego no ha creado**, y el
PPU espera estado de un SPU que no ha recibido trabajo. El bucle no se cierra
por si solo; en hardware se cierra porque el hilo *decoder* (tid 8, bloqueado en
`q=4`) crea esa cola y contesta al SPU.

Datos adicionales de esta medicion que descartan otras hipotesis:

| Comprobacion | Resultado |
|---|---|
| Drenaje del FIFO GCM | **correcto**: `getoff == put` y 13,3 MB de comandos consumidos |
| Registro de control | esta en memoria del invitado (`VM_HLE_INJECT_BASE+0x2000`), no es un puntero del host |
| Fences (`ref`) | 19 publicados al principio (0xB..0x12) y ninguno mas: el juego no espera un fence nuevo |
| Buzon SPU | **una sola** escritura (`[spu-outmbox]`), luego el SPU gira en LS |
| Histograma de metodos | vacio en regimen estable: el juego no emite draws tras la carga inicial |

Conclusion: el siguiente trabajo **no** es de renderer ni de presentacion; es
cerrar el protocolo de arranque de `bnusCore`: hacer que el hilo decoder cree la
cola `"nuSA"` al recibir la peticion por `q=4` y que el SPU obtenga su primer
trabajo. Con eso el SPU empezaria a mandar eventos a `port 6 -> q=7`, el hilo
update avanzaria y el juego saldria de inicializacion temprana.

### 7.102 Cabecera de secuencia MPEG-2 repetida: se acaban los fallos de decode

Los `DecodeFailed` que imprimia el juego (176 en 130 s) y los
`[mpeg2video] Invalid frame dimensions 0x0` de libavcodec (134) tenian una causa
comun y mecanica:

1. `vdec_ff_drain()` hace `avcodec_send_packet(NULL)` + **`avcodec_flush_buffers`**
   (el flush es como sale del estado EOF).
2. El flush **descarta la cabecera de secuencia** parseada.
3. El siguiente AU es una imagen suelta (P/B), sin `00 00 01 B3`, y el
   decodificador la rechaza con "Invalid frame dimensions 0x0". El AU se pierde.

Arreglo en `libs/codec/vdec_ffmpeg.c`: se guarda la ultima cabecera de secuencia
MPEG-2 vista (`seq[]`, desde el start code `0x000001B3` hasta el siguiente start
code) y se **antepone** a cualquier AU que no traiga una. Solo para MPEG-2: en
H.264 el equivalente es el SPS en otro NAL y repetirlo no es inocuo.

Medido, misma tanda de 30 s con sonda:

| Metrica | Antes | Ahora |
|---|---|---|
| `Invalid frame dimensions 0x0` | 134 | **3** |
| `DecodeFailed` (del juego) | 176 | **0** |
| `AU did not decode` | varios | 3 |

Es la primera vez que el reproductor del juego no reporta ni un fallo de
decodificacion con material real. **Ojo: la imagen empeora. Ver 7.104, donde
se revierte.**

### 7.103 El muro real: 1342 tareas SPURS encoladas, ninguna completada

Tanda de 200 s sin sonda, ya con el decodificador arreglado (�7.102):

```
[cellSpurs] CreateTask  ->  1342
JobDone / CompleteTask  ->     0
```

Las tareas llegan en ciclos de tres puntos de entrada distintos:

```
CreateTask(id=1335, entry=0x10053200, arg=00000000 167FBF00 ...)
CreateTask(id=1336, entry=0x1007E380, arg=00000000 167FC300 ...)
CreateTask(id=1337, entry=0x10085000, arg=00000000 10CDF000 ...)
```

es decir, **el juego encola trabajo continuamente y ese trabajo no se ejecuta
nunca**; el contador de ids crece sin reciclarse porque ningun bit de tarea
vuelve a cero. Es el bucle de carga que impide llegar al menu.

Con `SPURS_PM_FLOW=1` + `SPURS_PM_CLAIMLOG=1` (45 s) **no sale ni una linea de
flujo del gestor**: el *policy manager* de SPURS no esta corriendo. Y tiene
sentido: el unico grupo SPU del titulo es `bnusCore` (1 hilo, imagen 280400854),
que es el kernel propio de Namco, no el `CellSpursKernel` que nuestra
emulacion de `cellSpurs` espera para reclamar tareas del taskset.

Datos que acompa�an al cuadro (misma tanda):

| Medida | Valor | Lectura |
|---|---|---|
| DMA del SPU `bnusCore` | `cmd=0x40` (host->LS) y `0x20` (LS->host) en bucle | **el SPU si trabaja**: recibe y devuelve datos |
| `[VPDRAW]` | 11, todos en las primeras lineas (L412..L510) | los draws son del arranque, no del bucle |
| `SetFlip` | 0 | nunca se presenta un frame propio |
| Metodos RSX en regimen | 0 | el juego no emite comandos de dibujo mientras carga |

Conclusion: el render propio **no** esta bloqueado por el renderer, ni por el
presentador, ni por el decoder. Esta bloqueado porque el kernel SPU del titulo
no ejecuta las tareas que el juego le encola. El siguiente trabajo es el
protocolo de tareas de `bnusCore`: o se le hace reclamar el taskset que escribe
`cellSpursCreateTask`, o se implementa su cola de trabajo en HLE. Con eso la
carga terminaria y el juego pasaria al menu.

### 7.104 Revision honesta: repetir la cabecera arregla el contador y rompe la imagen

El arreglo de 7.102 se probo en tres variantes y se midio **imagen**, no solo
contadores. Resultado:

| Variante | `Invalid dims` | `AU did not decode` | Imagen en ventana |
|---|---|---|---|
| sin repetir cabecera (original) | 134 | varios | **limpia** (titulo legible) |
| repetir cabecera en **cada** AU | 3 | 3 | **gris con motas magenta** |
| repetir cabecera solo tras `flush` | 304 | 304 | limpia |
| repetir si `ctx->width == 0` | 3 | 3 | **gris con motas magenta** |

La causa: una cabecera de secuencia insertada a mitad de GOP **resetea el estado
de prediccion** del decodificador. Los contadores de error bajan porque el paquete
se acepta, pero lo que se decodifica ya no es la imagen del video (queda un gris
plano con macrobloques sueltos en magenta, tipico de prediccion rota).

Decision: **se revierte a la ruta original** (sin repetir cabecera). Perder un AU
suelto y quedarse con el cuadro anterior es un fallo visiblemente mucho menor que
presentar una imagen corrupta. Queda comentado en `vdec_ffmpeg.c` para que nadie
lo reintroduzca sin resolver el resync completo del stream (no solo la cabecera).

Estado visual final tras la revision: `dumps\game_window.png` de la ultima tanda
muestra `RIDGE STATE` con las flechas rojas y las marcas de derrape, nitido.

### 7.105 La primera tarea SPURS: falta quien la arranque

El port ya tiene **todo** el camino de ejecucion de una tarea SPURS menos el
disparo inicial:

- `spu_taskset_task_exited()` escanea el taskset, marca las tareas enabled+ready
  como running y las lanza con `spu_workload_dispatch_async()`. Pero solo se
  ejecuta **cuando otra tarea termina**.
- `spu_taskset_start_pending()` existe y hace exactamente "arranca lo que este
  listo", y solo se invocaba desde un hook de coherencia (`spu_taskset_register`
  + comparacion de la linea base del taskset) que en MR7 **no dispara**.
- La primera tarea la arrancaba la politica de tasksets de Sony
  (`g_taskset_policy_bytes`), y en este port solo esta el stub de tama�o 0
  (`spurs_policy_blob_weak.c`): la politica no se puede ejecutar, luego nunca hay
  primera tarea, luego `spu_taskset_task_exited` nunca corre.

Resultado: 1.338 `CreateTask` en 200 s y 0 completadas (�7.103).

Cambio aplicado en `cellSpursCreateTask`: tras `spurs_taskset_add_task()` se
llama a `spu_taskset_start_pending(taskset_ea)` (`MR7_NO_TASKSET_KICK=1` lo
desactiva para A/B). Efecto inmediato medido: **`CreateTask` cae de 1342 a 4** en
la misma ventana, sin crash y sin regresion de imagen: el bucle de encolado se
rompe.

Queda un paso: el escaneo todavia no lanza la tarea. Instrumentado con
`SPU_TS_KICKDBG=1` (imprime `taskset`, id, `elf`, `sz` y los 4 primeros bytes de
la imagen) para ver si el fallo es `spu_elf_image_size() == 0` (ELF no mapeado en
`elf=0x10053200`) o el estado de los bitsets.

#### 7.105 bis Resultado de la medicion: la patada se bloquea en el lock line

Con `SPU_TS_KICKDBG=1` no salio **ni una** linea `[tskick]` y `CreateTask` se quedo
en 4: el escaneo nunca llega a mirar la tarea. La causa es que la llamada se hace
**desde el propio hilo del juego**, que ya tiene tomado `spu_lockline_lock()`
mientras registra la tarea, y `spu_taskset_task_exited()` vuelve a pedirlo
(llamada anidada) -> el hilo se queda ahi y el juego deja de encolar.

Estado dejado en el codigo: la patada queda **opt-in** (`MR7_TASKSET_KICK=1`) y
por defecto OFF, para no tocar el comportamiento actual. Ambas builds compilan.

Siguiente paso exacto (no hecho para no romper el arbol con el presupuesto
agotado): mover la patada al **hilo del scheduler de SPURS**
(`spurs_kernel_thread`), que no toma el lock line, recorriendo los tasksets de la
instancia y llamando `spu_taskset_start_pending(taskset_ea)` una vez por pasada.
Foro: mantener en `cellSpurs.c` una tabla (taskset_ea, spurs_ea) rellenada en
`spurs_create_taskset`, y en la pasada del scheduler filtrar por `si->ea`.

#### 7.105 ter RESUELTO: las tareas SPURS ya corren (y completan)

Estado final del frente SPURS, medido en tandas de 220-260 s:

```
CreateTask        1303-1411
arrancadas        1298-1398      (antes: 0)
completadas       2593            todas con rc=0, sin FAULT ni halt
dispatch MISS     0
```

Traza del despacho, funcionando en cadena:

```
[ts-exit] ea=0x107CDA80 done=0xFFFFFFFF en=80000000 rdy=80000000
[ts-exit] ea=0x107CDA80 done=0xFFFFFFFF en=C0000000 rdy=C0000000
[taskset] start task 2 of taskset 0x107CDA80 elf=0x10056380
```

Las tres piezas del arreglo:

1. `spurs_ts_register(taskset_ea)` guarda los tasksets del titulo en una tabla
   `volatile` (spurs_ts_reg/s_ts_reg_n) en `libs/spurs/cellSpurs.c`.
2. `spurs_ts_kick_all()` se llama **desde el hilo del scheduler**
   (`spurs_kernel_thread`), no desde `CreateTask`: hacerlo inline moria en
   `spu_lockline_lock()` anidado (el llamante ya lo tiene) y dejaba el juego
   congelado en 4 tareas.
3. El registro debe ser `volatile`: sin eso el bucle infinito del scheduler
   cacheaba `s_ts_reg_n = 0` y la tabla parecia vacia para siempre.

Lo que queda: las tareas completan pero el juego **no avanza**, repite el ciclo
de tres entradas. `spu_taskset_task_exited()` limpia los bitsets y lanza la
siguiente tarea, pero **no senaliza el event flag del taskset
(`CSTS_EVENT_FLAG_ID1/2` en +0x1898/+0x189C) ni el handler de salida
(`CSTS_X78`)**, que es lo que el juego espera. Siguiente cambio: en la salida de
la tarea, resolver el EA del event flag por su id y llamar a
`spurs_ef_set_from_spu(ea, 1u << taskId)`; despues medir si aparecen draws.