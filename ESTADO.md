# Estado del proyecto

Clasificación estricta. Un elemento solo pasa a **PROBADO** cuando se ha
ejecutado contra datos reales o una prueba automatizada reproducible.

---

## INVESTIGADO

| Elemento | Evidencia |
|---|---|
| Disco BCES00009 y árbol de archivos | `rr7native --probe G:\` |
| Metadatos PARAM.SFO | 10 entradas, título/ID/versión |
| SELF retail cifrado | cabecera SCE, key rev 1, 8 segmentos cifrados |
| Arquitectura PS3 (Cell/RSX) | documentación y estructura del ELF |
| Formato nuSound2 | vgmstream + extracción real |
| Formato PAMF 4.1 | cabecera real + RPCS3 |
| Formato RR7.DAT/TOC | RRUnpacker + estructura del ELF |
| RRLZ | documentación + implementación |
| Rendimiento original (1080p60) | Digital Foundry + SFO |
| ps3recomp | repositorio, docs y build local |

| Elemento | Evidencia |
|---|---|
| Descifrado del ELF | RPCS3 `--decrypt` sobre la copia del usuario; SHA-256 `5FAA5AF9…4B03D5` |
| Loader PPU | 14.067 funciones, entry `0x1022C`, TOC `0x4EEA88` |
| Imports | 262 en 23 librerías; 189/262 con handler HLE |
| TOC real de RR7.DAT | `0x630128`, 13.313 archivos, 2.139 contenedores, ratio 1.0 |
| Extracción DAT | `DB` (13 archivos) y `SoundConfig_Free` (2) con RRLZ |
| Imágenes SPU | 42, 1.005.116 bytes, liftadas por el scaffold |
| Lifter PPU | 14.834 funciones, 9 TUs, ~1,5M líneas |
| Build del port | configurado; compilación en curso |

## IDENTIFICADO

| Elemento | Detalle |
|---|---|
| Punto de entrada del juego | `0x4CB300` (OPD) / `0x1022C` (código) |
| TOC del ejecutable | `0x4EEA88` |
| TOC de archivos | `0x630128` |
| Segmento de imports | `proc_prx_param` (ya descifrado) |
| Bug `cellPamf` de ps3recomp | offsets 0x08/0x10/0x14 no son los de PAMF 4.1 |
| Códecs de audio | ATRAC3plus 5.1 y PCM16BE |
| Códecs de vídeo | MPEG-2 720p en MPEG-PS |
| Idiomas | 5 bancos de voces DJ |
| Ausencia de SPRX | todo el código en EBOOT |
| HLE pendiente crítico | `sysPrxForUser` (11), `_cellGcmInitBody`, `cellAtrac` (6) |

## IMPLEMENTADO

| Elemento | Archivo |
|---|---|
| Shell Win32 + D3D11 | `src/main.cpp`, `src/platform`, `src/gfx` |
| Configuración INI | `src/core/config.*` |
| Log y diagnóstico | `src/core/log.*` |
| Parser SFO | `src/assets/sfo.*` |
| Parser SELF | `src/assets/self.*` |
| Parser NUB + extracción | `src/assets/nub.*` |
| Parser PAMF | `src/assets/pamf.*` |
| Esqueleto DAT | `src/assets/dat.*` |
| TOC + extracción DAT | `tools/rr7_dat.py` |
| RRLZ | `tools/rrlz.py` |
| PAMF → MPEG-PS | `tools/pamf_extract.py` |
| Tests sintéticos | `tools/test_formats.py` |

## COMPILADO

| Artefacto | Resultado |
|---|---|
| `rr7native.exe` | 339.968 bytes, Clang 22/MinGW, sin errores |
| `libps3recomp_runtime.a` | 2.875.548 bytes, 160/160 tareas |
| Backend D3D12 de ps3recomp | compilado (con avisos no bloqueantes) |
| Lift PPU | 9 TUs, `clang++ -fsyntax-only` sin errores |
| Proyecto de port | configurado con 9 PPU + 42 SPU |
| `MyGameRecomp.exe` | 66.412.544 bytes, enlazado con D3D12/DXGI/D3DCompiler |

## PROBADO

| Prueba | Resultado |
|---|---|
| `--selftest --frames 60` | RTX 5070, FL 11.1, 60 frames, exit 0 |
| `--window --frames 60` | ventana real, 60 frames, exit 0 |
| `--probe G:\` | 7,43 GiB, 6 `.nub`, 7 `.pam`, SELF cifrado |
| Extracción BGM (2 tonos) | 33,75 MiB; FFprobe: atrac3p 5.1 48 kHz |
| Extracción DJ (3 tonos) | 273,63 KiB; FFprobe/FFmpeg: PCM 44,1 kHz mono |
| PAMF → MPEG-PS | FFprobe: mpeg2video 1280×720 29,97 |
| `tools/test_formats.py` | RRLZ, TOC, extracción 60 archivos: OK |
| `rr7_dat.py locate` sobre SELF | falla limpiamente: “not an ELF” |
| Primer arranque nativo | D3D12 + GCM 1080p + cellFont; se bloquea en `cellFont` |
| `sys_mempool_*` | 4 pools creados; `try_allocate` devuelve bloques válidos |
| `RR7_FONT_SEED=1` | sonda: elimina nulls de lectura; quedan escrituras NULL y espera del semáforo 4 |
| VFS / ISO montado | requisito: `PS3_VFS_ROOT=G:/`; sin montar, bucle de `open FAIL RR7.DAT` |
| Pipeline de carga/render | `Draw Thread`, `File Thread`, `Initialize Thread`; el loader lee `RR7.DAT` real hasta 41 MB |
| SPU `bnusCore` | hilo SPU liftado ejecutándose (`entry=0x90`) |
| Primeros frames D3D12 | `CLEAR_SURFACE` + `DRAW_ARRAYS`; PSO falla con `0x887A0005` (reason `INVALID_CALL`) |
| Debug layer D3D12 | no disponible (`0x887A002D`); sin admin para Graphics Tools |
| PSO FP | **resuelto**: el fallo era el scanout 1920×1080 sobre backbuffer 1280×720; PSOs listos |
| Geometría FIFO RR7 | anillo 64 KB EA `0x11100000` → IO `0x0000`; buffers enlazados en IO `0x00CB0000` y `0x010A0000` |
| Desvío del walker | **corregido**: 0 saltos a datos de vértices; resync acotado al `put` vivo |
| Backbuffer | negro; el juego se detiene en carga (lecturas del DAT paradas en 41 MB) |
| Siguiente bloqueo | estado de inicialización del juego: frame de 1 triángulo negro, sin texturas, vídeo en 720×480 |
| `stop 0x110` (SPU) | **corregido** en la ruta liftada: clave sin cola → `CELL_ESRCH`; ya no inunda el buzón |
| Perfilado | `PS3_SAMPLE` habilitado en el port; 94 % del tiempo en `ntdll` (hilos bloqueados, no girando) |
| Texto del frame | no hay glifos reales: `cellFontRenderCharGlyphImage` recibe `code=0` y superficie nula (sonda del hilo de fuentes) |
| Presentación | **verificada**: enlaza el RT 1280×720 como textura (unit 0, off `0x10000`) y dibuja el triángulo de blit |
| Contenido | los assets de carga (`Obj_Loading`/`Tex_Loading`, ~20 MB) ya están leídos pero nunca se dibujan |
| Loader | espera en `sys_cond_wait`; productor identificado (`func_0003632C` + resolutor de rutas `func_0027A0CC`) |
| Mapa por `lr` | `RR7_LRS=1`: el loader **alterna** bucle/resolución (no está bloqueado); el `File Thread` sigue ocioso |
| Lecturas del DAT | **corregido**: con `FS_READ_ALL=1` el total real llega a **45.955.072 B** (el log sólo imprimía 50 líneas) |
| Punto de parada | dentro de `Obj_RBTop`, antes de `Obj_OnlineRoomSearch`; sin EBADF ni lecturas cortas |
| Jobs del File Thread | dos tipos alternos (`0x139C0` lector, `0x140C8`); recuento coherente con las lecturas, sin livelock |
| Motores 2D | descartados: `GCM2D_TRACE` no registra ni una llamada (subcanales 2–6 = 0) |
| Callback de `sceNp` | ahora se notifica el estado actual (OFFLINE); RR7 no cambia por ello |
| Input | descartado: `PAD_AUTOPRESS` y `YDKJ_INJECT_PAD` (START+CROSS) no cambian nada |
| `put` del GCM | **congelado** en `0x00CB12F0`: el hilo de dibujo deja de construir el frame a medias |
| `current` del GCM | **congelado** en `0x111001A4`: el invitado tampoco reserva; no es sólo el walker |
| Draw Thread | en `func_000C5680` → `func_000E5998` (sprites, 2× `AddressToOffset`), sin flush |
| Referencia RPCS3 | arranca `BCES00009` pero aborta por **firmware incompleto**; no usable sin el del usuario |
| Firmware (usuario) | instalado en RPCS3 local (4.92); **RR7 llega a la pantalla de título a 74 fps** |
| Objetivo visual | `docs/referencia_titulo_rpcs3.png` (captura local, no distribuible) |
| Datos leídos | RPCS3: 77 MB hasta el título; nuestro port: 46 MB |
| `put` | **corregido**: el invitado SÍ lo publica en `0x20002000` (`GCM_CONTROL_GUEST_ADDR` usa `ppu_hle_inject_base`, no `0x0F80…`) |
| `0xCB12F0` | **no es overrun**: está dentro del buffer de 64 KB; la sonda `RING_OVERRUN_FIX` era errónea |
| Frames publicados | 35 (alternando buffers `0xCB0000` / `0x10A0000`) y luego deja de publicar |
| Hilo principal | en `lr=0xBE348`, la rama que **salta el render** esperando al Draw Thread |
| Reloj del frame | `sys_time_get_system_time`: se llama y avanza (1.200 llamadas/14 s), no es el problema |
| Draw Thread | dentro de `func_000C5680` traduciendo direcciones (~200/s, distintas) y sin cerrar el frame |
| **Ciclo real** | 284 llamadas idénticas `0xE7DC0(r3=0x107C4780, r4=0xC4)` desde `func_000E6B10+0x463`: espera al **recurso 196** |
| Mesa de recursos | `func_000E6B10` rellena 16 entradas (`0xC0..0xFC`) con objetos `0xEC2C8/0xEC548/0xEC878` |
| Bucle real | `func_000E6CBC+0x2B7`; cola `loc_000E6F4C` = `esperar(0x3B92B8)` + `recorrer(0x41ED2C)` en `goto` infinito (render loop) |
| Flush | `func_000BCAE4` se invoca desde `func_000BE0CC` vía el thunk virtual `func_000C474C` |
| Predicado | verificado en el binario: `0xE7DC0` = `li r3,1; blr` (método por defecto) |
| Objeto del bucle | `0x107C4780` está **a cero** (`RR7_MEMDUMP`); el recorrido no puede producir nada |
| TTY de RPCS3 | activados `sys_rsx`/`cellFs`/etc. a Trace: **no añaden trazas**; la comparación útil es de memoria |
| Mesa de 16 slots | `base[0xC0 + i*4]` está **toda a cero** (0xD0 bytes); el juego recorre una estructura vacía |
| Constructor | `func_000180F0` (construye la mesa en `obj+0xC0`) **nunca se llama**; su único OPD es `0x4CB938` |
| Hilo que gira | tid 11 (Initialize, func_0003833C): ~1,63 M mutex_lock/mutex_unlock por segundo |
| Espera | sys_cond_wait (107) llamado **1 vez** en total: el bucle salta la espera porque g->[4] = 1 |
| Salida del bucle | sólo si g->[0] == 0x10; vale 0 y nada lo escribe (verificado con WWATCH y contra el binario) |
| Defecto del toolkit | sys_cond_wait lee el timeout de gpr[4] (mutex) en vez de gpr[5]; **no aplicado** (RR7 dejaría de bloquear) |
| Carga | 69 lecturas de `RR7.DAT` = 45.955.072 B y despues **nada** (30 s sin avanzar); cero `lseek` |
| File Thread | recibe exactamente 69 trabajos (`func_000139C0`), iguales a las 69 lecturas: deja de recibir trabajo, no falla |
| Listener del bucle | **existe** (`g->[8] = 0x1042C020` -> `func_000366D4`); el bucle del frontend corre su actualizacion por frame |
| Correccion | los volcados que mostraban `g->[8] = 0` eran de ejecuciones sin ISO montado: afirmacion retirada |
| Bucle de frames | **vivo a ~33 Hz**: `t1 -> sem5 -> t4 (Draw) -> sem6 -> t1` (~255 ciclos en 9 s) |
| Frontend | bombea su lista de tareas cada frame (3..12 entradas); no esta parado |
| Peticion de fichero | objeto `0x10290640` termina la 69a lectura (`0x3D000` B) y queda ocioso: **nadie crea la 70** |
| Punto exacto a leer | callback de fin de peticion OPD `0x4CB788` -> `0x000156AC` -> `0x127F4` |
| Productor de lecturas | **tid 11 (frontend)** rellena la peticion con `func_00015148` (lr `0x173E8`); el File Thread la consume y marca la secuencia |
| Ultima peticion | `seq = 0x71`, `0x3D000` B (69a lectura); despues el productor **deja de pedir** |
| Estado terminal | `file[expect=0x3D000 pending=0 eof=0]`: la rama de fin es la **normal**, no un error (corrige �7.45) |
| Diagnostico | el loader **no** esta bloqueado: se queda sin demanda; falta la transicion a construir la escena |
| Cola del loader | `obj = *(*(toc-0x7F84)-0x8000)`; `head = tail = 56`, `flag = 0`: **vacia** |
| Carga de arranque | **completada**: 56 comandos -> 56 callbacks -> 69 lecturas, todas cerradas |
| Fence de GCM | label `0x80` (`0x20000800`) alterna 1/0 ~26 veces/s: el submit `func_000BCAE4` **si** se ejecuta |
| Emision de comandos | tras el arranque el juego **deja de generar**: `ctx.current` se queda en `0x111001A4` y `put` en `0x00CB12F0` |
| Sonda de anillo | `GCM_RING_FROM_IO=1` cambia **+2 draws** (12->14): **no** es la causa raiz (corregido en �7.56) |
| Presentacion | **funciona**: la ventana presenta (~31/s) y el color forzado se ve en pantalla |
| Frames con geometria | ~11 de ~310 presents llevan draws; despues solo frames vacios |
| Motor de dibujo / PAD_AUTOPRESS | probados: **sin efecto** (era varianza). Descartados como causa |
| Depurador | **operativo** (lldb + shim python + PYTHONHOME); breakpoints sobre build_dbg |
| Bloqueo unico | el frontend no pasa de su estado de arranque al de titulo |
| Correccion | no es un atasco de E/S: falta la **transicion** del frontend a construir la escena |
| Descartado | `vtbl=0` (sonda heredada de otro titulo), TTY del juego (no imprime), `+6834` en el objeto equivocado |
| Datos guardados al arrancar | `AutoLoad(dir='BLJS10001GAME')` -> `funcStat` del juego devuelve **-4** (sin datos); una sola vez, sin bucle ni espera |
| Entorno de pruebas | el ISO se desmonta solo entre tandas; sin `G:` el juego reintenta la apertura de `RR7.DAT` ~1.000.000 veces (hay que remontar antes de medir) |
| Causa primaria | la **parada de la carga** en 46 MB: el constructor es consumidor de ese arranque |
| Callback de anillo | nunca se invoca (`RECYCLE_DBG`: 0 llamadas); `put` vive en buffers propios del juego, no en el anillo de IO 0 |
| Fuentes | `cellFontFT` es stub; `FONT_TTF=<ruta>` añade rasterizado sustituto opt-in (no empaquetado) |

| cellPamf HLE | **corregido**: puntero sin traducir (0xC0000005 en el hilo del demuxer), `GetHeaderSize` ausente (NID 0xCA8181C1) y layout 4.1 real; 0 excepciones tras el cambio |
| cellPamf importado | 10 funciones; todas cubiertas (1256 handlers registrados, antes 1251) |
| Bucle de arranque | **reproducido**: `AutoLoad` -> NODATA -> "Cancelar carga de datos?" -> `AutoLoad`, 336 veces en 150 s |
| `SAVEDATA_NODATA_OK=1` | corta el bucle (1 dialogo) y devuelve al estado de escena vacia: 13 de 2.383 frames con draws |
| `MSGDIALOG_ANSWER` | **sin efecto** sobre este bucle: el reintento es del titulo ante NODATA |
| Video | el demuxer y `title.pam` solo se alcanzan con depurador; las 379.897 aperturas eran por `GetHeaderSize` sin implementar |

| Respuesta de guardado | **corregida** segun la referencia: NODATA del callback -> `ERROR_CBRESULT` (`SAVEDATA_NODATA=cbr`); corta el bucle, no desbloquea la escena |
| Hilo del frontend | **no bloqueado**: tid 11 ejecutando `func_0027A0CC` (tokenizador de rutas) desde el listener `func_000366D4` |
| Pump medido | `f0=0` en 210/210 muestras; `f8=0x1042C020` (listener presente); lista del loader vacia (`head=tail=0`) |
| Mesa de escena | 13 slots no nulos (`0x107C4780`...); el `put` del GCM se congela en `0x11DB0D54` |
| Siguiente objetivo | la entrada que el listener pasa a `func_0027A0CC`: que tarea es y por que no encola carga |

| Guardado | **funciona**: el titulo escribio y relee `BLJS10001GAME` (GAME 1 MB + ICON0/PIC1/PARAM.SFO); `LOAD complete` |
| Entrada de `func_0027A0CC` | **no es ASCII**: el blob (`0x101A4188`) es un diccionario de pares de 2 bytes; el indice del listener esta en `+0x30` |

| **Frontend** | **resuelto**: esperaba una pulsacion; con `PAD_AUTOPRESS=15000:0x0008` (START) **entra al titulo** |
| Titulo | `title.pam` abierto y leido; `PAMF v0041` con 1 stream, 73.949.184 B de datos |
| `cellDmuxQueryAttr` / `QueryEsAttr` | **implementadas** (NID 0xA2D4189B / 0x02170D1A): faltaban y corrompian los tamanos -> 0xC0000005 |
| Callbacks de `cellDmux` | **corregidos**: se llamaban como puntero de host (OPD `0xD0054F30` -> DEP); ahora `ps3_invoke_guest` |
| Dibujo en el titulo | 649 de 2.250 frames con **4 draws** (quad de video + UI); antes 11 frames en total |
| Video visible | **no**: `cellVdec` sin runtime de FFmpeg da cuadro negro y el demuxer entrega el buffer entero como un AU |

| Demuxer | **real**: parseo MPEG-PS (BA/BB/BF/PES); 32 AUs con PTS por bloque de 64 KB |
| `cellDmuxGetAu` | **corregido**: devuelve EA de `CellDmuxAuInfo[]` en memoria de invitado; antes un puntero de host |
| Filter ids | **corregido**: `CellCodecEsFilterId` son u32; se leian con offsets de byte y no casaban |
| Decodificador | el titulo abre `cellVdec` y lo cierra al instante (`StartSeq`->`EndSeq`->`Close`); 11 de 2.358 frames con draws |

| FFmpeg en el port | **activo**: `libavcodec 62.28.102 loaded` (headers FFmpeg 8.0 + DLLs compartidas de OBS) |
| Demuxer | **reensambla cuadros completos** desde los PES; un AU por cuadro (19-87 KB), PTS +3003 |
| Callbacks `cellDmux` | **corregidos**: Open/EnableEs reciben puntero a `CellDmuxCb`/`CellDmuxEsCb`; el mensaje va por puntero |
| `CellDmuxAuInfo` | **corregido**: 0x28 bytes, `auMaxSize`/`isRap`/`pts{upper,lower}` |
| Video | decodifica ~1000 AUs en 75 s **sin errores de libavcodec**; 4 draws por frame |
| Pendiente inmediato | la ventana sigue negra: falta el tramo `cellVdecGetPicture` -> textura RSX -> composicion |
| `RR7_NO_VP` | **contraproducente**: con el activo no se dibuja el video (11 frames en total) |

| Video decodificado | **si**: `picture 1280x720` y `GetPicture` con formato 3 (YUV420 planar), 0 errores |
| Bug del buffer | **corregido**: mover la cola a offset 0 machacaba los AUs publicados (`AVERROR_INVALIDDATA`) |
| Bloqueo actual | los PSO del camino **VP** fallan con `E_INVALIDARG` (0x80070057) -> ningun draw del video llega a pantalla |
| Por eso `RR7_NO_VP` | el parche antiguo desactivaba el unico camino que puede pintar el video |

| Camino VP | **desbloqueado**: PSO de respaldo solo-color cuando el PS de atlas no enlaza (el VS del video no emite TEXCOORD) |
| Fragment programs | **se compilan** ahora (`[FP] guest FP pipeline ready`) al llegar el draw al backend |
| Video | 399 `SetStream` / 352 `DecodeAu` / **134 `GetPicture`** en ~40 s: el titulo consume el video |
| Siguiente bloqueo | **cero `draw_arrays`** tras arrancar la pelicula y `DecodeFailed 83:82:75` del propio juego |

| **Primeros pixeles** | el quad del video se dibuja: rectangulo 720x480 con el shader del juego (antes todo plano) |
| Volcado de frames | `frame_010.png` 47.878 B con contenido (antes 4.349 B = plano) |
| Textura | **no enlazada**: `tex0=0x0 set=0` en el draw del video; el FP del juego la muestrea y sale color plano |

| Enlace de textura | **emitido pero perdido**: `[TEXM] unit=0 off=0x10000` y el draw sigue con `tex0=0 set=0` |
| Consecuencia | `d3d12_bind_texture` no se invoca en los draws del video -> `cur_texs` a cero -> el FP muestrea nada |
| Rectangulo rojo | es el quad del video a 720x480 con el shader del juego, sin textura |

| Boton del titulo | **CROSS** (0x4000), no START: con START la pelicula no arrancaba |
| Video | **decodificado limpio**: AUs con arena propia (4 slots de 256 KB) -> escena real en `dumps\vdec_frame4.png` |
| Publicacion | el titulo **no publica**: 0 flips, 0 blits NV3089, 0 escrituras de CPU al framebuffer, 11 draws |
| Sonda visible | `RR7_VDEC_TO_FB` + `GCM_GUEST_FB` presenta el cuadro desde un bufer del host |

| Build de release | **corregida**: se configuro sin FFmpeg (`PS3RECOMP_FFMPEG_RUNTIME=OFF`) -> tus pruebas no decodificaban video |
| Sonda de video | `[vdecprobe] push=1 w=1280 h=720` y `BMP=2,7 MB` en la release corregida |
| Presentador | `[GUESTFB] USING VDCE FRAME`: la ventana usa el cuadro decodificado desde el bufer del host |

| **Video del titulo** | **se decodifica el logo RIDGE** (`dumps\RR7_title_RIDGE.png`): demux+FFmpeg+conversion correctos |
| Presentacion en ventana | sigue blanca: `GUESTFB` usa el bufer del host pero la copia D3D12 no llega al swapchain |

| **IMAGEN EN VENTANA** | **SI**: se ve la pelicula de titulo (`RIDGE STATE - THE HOME OF DRIFT`), `dumps\RR7_TITLE_ON_SCREEN.png` |
| Causa del blanco | `CopyTextureRegion` 1920x1080 -> 1280x720 sin box = llamada invalida -> **dispositivo D3D12 eliminado** (todo el frame fallaba) |
| Presentador | escala en CPU al tamano del backbuffer y copia una caja valida (`[GUESTFB] copy 1280x720 -> backbuffer 1280x720`) |
| Captura de ventana | requiere `HWND_TOPMOST` + ocultar la consola; `SetForegroundWindow` no basta |
| Lanzador | `port\run_video_retry.cmd`: monta la ISO, CROSS, sonda ON, ventana abierta hasta cerrarla |

| Ticker vblank | vivo: 3598 ticks/60 s y **3300 entregas** del handler del juego |
| Trabajos SPU | **0 despachados** (`SPU_JOBEXEC=1` no imprime ni un DISPATCH) |
| Metodos RSX | histograma **vacio** en regimen estable: el juego no emite comandos de dibujo |
| Flips del juego | 0 (el `GetPicture=6` del log es el tope de lineas, no el numero real) |
| Bloqueo del render | hilo Draw espera `sem=5`; SPU `bnusCore` recibe `CELL_ESRCH` en `stop 0x110` (clave `nuAS` inexistente) |

| Calidad del video | **corregida**: la corrupcion por macrobloques era descartar AUs (`BUSY`); ahora se tira el cuadro mas antiguo y la imagen sale nitida (`dumps\RR7_CLEAN_FRAME.png`) |

| FIFO GCM | se drena bien: `getoff == put`, 13,3 MB de comandos consumidos |
| Circuito de eventos | `port6 -> q=7` **nunca envia**; el hilo update sondea q=7 cada 5,3 ms y el SPU gira esperando trabajo |
| Siguiente arreglo | cerrar el protocolo `bnusCore`: la cola `nuSA` debe crearla el hilo decoder al recibir la peticion por `q=4` |

| Decodificador MPEG-2 | repetir la cabecera baja los rechazos pero **corrompe la imagen** (gris con motas): **revertido**, se mantiene la ruta original (imagen limpia) |
| SPU `bnusCore` | **si trabaja**: recibe DMA 0x40 y devuelve 0x20 en bucle (no esta muerto) |

| **Muro del render** | **1342 tareas `cellSpurs` encoladas y 0 completadas**; el policy manager SPURS no corre (el kernel del titulo es `bnusCore`) |
| Siguiente paso | implementar/adaptar el protocolo de tareas de `bnusCore` para que la carga termine |

## PENDIENTE

| Elemento | Bloqueo |
|---|---|
| Primer arranque del port | build en curso |
| HLE `sysPrxForUser` / `_cellGcmInitBody` | implementar 13 NIDs |
| `cellPamf` / `cellAtrac` / `cellDmux` | 13 NIDs + demux PES |
| 42 imágenes SPU en ejecución | validar SPURS/raw y canales |
| ~2.230 instrucciones VMX/op31 | corregir en el lifter |
| Shaders y frames RSX | arranque + D3D12 |
| Formatos de modelos/texturas | extraer y documentar |
| Física, IA, colisiones | arranque |
| Audio dinámico del motor | arranque |
| Menús, progresión, guardado | arranque |
| Primera carrera | todo lo anterior |
| Distribución | primera carrera estable |

---

## Resumen

```
Fase 1  ████████████████████ 100%  (informe + evidencias)
Fase 2  ████████████████████ 100%  (plan)
Fase 3  ████████████████████ 100%  (shell compilado y probado)
Fase 4  ██████████████████░░  90%  (SPU audio + primeros frames; PSO D3D12 pendiente)
Fase 5  ░░░░░░░░░░░░░░░░░░░░   0%
Fase 6  ░░░░░░░░░░░░░░░░░░░░   0%
Fase 7  ░░░░░░░░░░░░░░░░░░░░   0%
```
