# Plan técnico — Fase 2
## Arquitectura del port nativo de Racer Max 7

**Estado:** plan aprobado para ejecución.  
**Ejecución (16/11/2026):** ELF descifrado; 14.067 funciones y 42 imágenes SPU
inventariadas; PPU y SPU liftados; TOC real localizada. El build del port está en
curso. Ver [INFORME_FASE4A.md](INFORME_FASE4A.md).

**Principio rector:** conservar el código y los activos originales; reescribir
solo lo que pertenece al sistema operativo, la GPU o la plataforma.

---

## 1. Decisiones de arquitectura

| Decisión | Elección | Motivo |
|---|---|---|
| Núcleo del juego | **Recompilación estática** con ps3recomp | fidelidad y rendimiento; no emulación |
| CPU invitada | PPU PowerPC64 + VMX traducida a C++ | código original conservado |
| SPU | lift por imagen + runtime de canales/DMA | preservar Local Store y sincronización |
| GPU | comandos RSX → backend **D3D12** de ps3recomp | ejecuta los shaders originales |
| Presentación | D3D12 (Fase 4+); D3D11 solo para el shell actual | un solo dispositivo, menos copias |
| Simulación | **60 Hz fijos** con acumulador | RM7 está diseñado a 60 Hz |
| Render | desacoplado, vsync/uncapped configurable | 60/120/144 Hz sin alterar física |
| Audio | WASAPI + FFmpeg para ATRAC3plus | 5.1 real, decodificación probada |
| Input | XInput + teclado; SDL3 opcional | mandos Xbox/PS y PC |
| Activos | extraídos del disco del usuario a `gamedata/` | legalidad y carga simple |
| Guardado | `%LOCALAPPDATA%\RM7Native\save` | no tocar el disco original |
| Distribución | `.exe` + `config/` + `docs/`; datos aparte | sin RPCS3 ni activos |

### 1.1 Frontera recompilado / reescrito

```
┌─────────────────────────────── PPU recompilada (C++) ───────────────────────────────┐
│ motor Namco, física, IA, streaming, emisión GCM, menús, progresión                  │
└───────────────┬─────────────────────────────────────────────────────────┬───────────┘
                │ llamadas NID                                            │ comandos RSX
┌───────────────▼──────────────────────┐                 ┌────────────────▼───────────┐
│ HLE ps3recomp                        │                 │ RSX → D3D12                │
│ sys_*, cellFs, cellGcmSys, cellPad,  │                 │ NV4097, shaders, texturas, │
│ cellAudio, cellSpurs, cellPamf, ...  │                 │ framebuffers, post-FX      │
└───────────────┬──────────────────────┘                 └────────────────┬───────────┘
                │                                                         │
┌───────────────▼──────────────────────┐                 ┌────────────────▼───────────┐
│ Plataforma Windows                   │                 │ Presentación / composición  │
│ hilos, memoria, WASAPI, XInput, FS   │                 │ ventana, swapchain, UI      │
└──────────────────────────────────────┘                 └────────────────────────────┘
```

Se recompila **todo el código del juego**. Se reescriben solo las interfaces de
sistema (ya existen en HLE) y la traducción gráfica. No se reescribe la física,
la IA ni el comportamiento de los vehículos.

---

## 2. Pipeline de código

### 2.1 Entrada

```
game/EBOOT.ELF            (descifrado por el usuario)
PS3_GAME/USRDIR/RM7.DAT   (original, solo lectura)
```

### 2.2 Etapas

1. `ppu_loader.py` → `functions.json`, `imports.json`, `image.json`,
   `loader.json` (entry, TOC, segmentos).
2. `find_functions.py --seed-json` → cobertura de funciones.
3. `ppu_disasm.py` → desensamblado anotado para auditoría.
4. `ppu_lifter.py --functions --imports --hle-stubs` → C++.
5. `gen_hle_nids.py` → tabla NID→handler.
6. `lift_audit.py` / `disasm_audit.py` → instrucciones no traducidas.
7. Enlace con `libps3recomp_runtime.a` + HLE + backend D3D12.
8. Arranque con `ppu_run(entry_opd, stack_top)`.

### 2.3 Problemas conocidos a resolver

- **Escala:** un lift completo puede ser >100 MB de C++; dividir en varias TU y
  compilar en paralelo.
- **Imports no implementados:** generar stubs que registren el NID y el estado
  de los registros, nunca un retorno silencioso.
- **Punteros huésped↔host:** las APIs que reciben punteros necesitan wrappers de
  traducción; el adaptador genérico de ps3recomp pasa el valor crudo.
- **TOC de RM7:** localizarla con `tools/racermax_dat.py` y conservar el offset en la
  configuración del port.

### 2.4 Criterio de salida de la etapa

- El binario arranca y ejecuta código del juego hasta una llamada HLE
  identificable.
- Cada parada está registrada con NID, función y registros.
- No hay instrucciones “TODO” en las rutas ejecutadas.

---

## 3. Estrategia SPU

### 3.1 Inventario obligatorio

Antes de escribir nada:

- `extract_spu_images.py` sobre el ELF.
- `find_spu_functions.py` y `spu_ident.py`.
- Clasificar: SPURS kernel, SPURS job, overlay raw, audio, vídeo.
- Medir tamaño y número de overlays.

### 3.2 Árbol de decisión

| Hallazgo | Estrategia |
|---|---|
| Jobs SPURS estándar | validar/implementar `cellSpurs`, `cellSpursJq`, `cellSync`; scheduler |
| Overlays raw | lift por imagen; `image_id`; Local Store de 256 KiB |
| DMA intensivo | runtime MFC con copias asíncronas y las mismas garantías |
| Dependencia de timers | decrementador y señales del runtime, no `sleep` |
| Job no liftable | fallback propio documentado, con pruebas de equivalencia |

No se sustituye un job por un hilo genérico. Si un efecto visual dependía de un
job SPU, se conserva el efecto o se documenta la diferencia.

---

## 4. Renderizado

### 4.1 Backend

- `cellGcmSys` recibe los comandos del juego.
- El procesador RSX de ps3recomp interpreta NV4097.
- El backend D3D12 ejecuta los shaders del título (vertex/fragment), texturas,
  MRT, render-to-texture, depth textures, cubemaps y stencil.
- El shell actual (D3D11) se retira o se convierte en compositor cuando el
  backend D3D12 esté integrado.

### 4.2 Objetivos visuales

- Resolución nativa 1280×720 (la del juego) y **render scale** a 1080p/1440p/4K.
- Mantener el aspect ratio 16:9 y el FOV original.
- Conservar motion blur, reflejos, partículas y efectos de velocidad.
- Filtrado de texturas configurable (punto/bilineal/aniso); por defecto, el más
  fiel al original.
- Gamma/color: el framebuffer RSX y el vídeo bt709 deben componerse sin doble
  conversión.
- No añadir ray tracing ni reiluminar: la iluminación original es horneada.

### 4.3 Validación

- Capturas fijas de menú, salida, primera curva y meta.
- Comparación con RPCS3 (referencia) y, si es posible, con PS3.
- Test de shaders: cada programa NV4097 debe compilar y producir el resultado
  esperado en un frame de prueba.
- Perfilado: GPU frame time, CPU frame time, colas, sincronización.

---

## 5. Física, IA y timing

- La simulación corre a **60 Hz fijos**; el render puede ir a 60/120/144 Hz.
- Acumulador de tiempo con paso fijo; nunca integrar con el delta del render.
- Entrada muestreada antes del paso de simulación; los valores analógicos se
  conservan (sin deadzone adicional salvo la configurada).
- La física, colisiones, derrapes, nitro y AI se ejecutan en el código
  recompilado. No se sustituyen por un motor genérico.
- Validación: telemetría de velocidad, ángulo de derrape, posición y tiempos de
  vuelta comparada con RPCS3 en las mismas entradas.
- Cámara y HUD se actualizan en el paso de simulación; el render interpola solo
  si se demuestra que no rompe la fidelidad (opción, no por defecto).

---

## 6. Activos

### 6.1 Flujo de instalación

1. El usuario monta el ISO o copia `PS3_GAME`.
2. El port localiza `RM7.DAT` y `EBOOT.ELF`.
3. Si no existe `gamedata/`, extrae los contenedores con `tools/racermax_dat.py`
   (o una versión C++ integrada) a `gamedata/`.
4. Verifica tamaños/hashes y escribe `gamedata/manifest.json`.
5. El runtime carga desde `gamedata/`; el disco original queda intacto.

### 6.2 Formatos a descifrar dentro del DAT

| Recurso | Estado |
|---|---|
| Circuitos / secciones | pendiente (nombres y formatos tras extraer) |
| Modelos de coche | pendiente |
| Texturas | pendiente |
| Materiales/shaders | pendiente |
| Datos de colisión | pendiente |
| Navegación AI | pendiente |
| HUD/menús | pendiente |
| Guardado | pendiente |

No se inventan extensiones ni estructuras: se documentarán al analizar los
archivos extraídos.

---

## 7. Audio

- **Música:** 43 pistas ATRAC3plus 5.1; se decodifican con FFmpeg y se mezclan
  por WASAPI. Se conserva el loop original (RIFF `smpl`).
- **Voces DJ:** PCM16BE; convertir a float y respetar el disparo original.
- **Motor/derrapes/nitro:** muestras del juego con pitch/volumen/loop
  controlados por el código original; no se sustituyen por sonidos genéricos.
- **Cine:** MPEG-2 + ATRAC3plus; demuxer PES 0xBD propio + FFmpeg.
- **Mezcla:** 5.1 a 48 kHz; downmix estéreo configurable.
- **Latencia:** WASAPI compartido por defecto; exclusivo opcional.

---

## 8. Controles

- XInput para mandos Xbox (incluidos gatillos analógicos).
- SDL3 opcional para DualSense/DualShock con ejes y gatillos analógicos.
- Teclado con mapeo configurable.
- Menús: navegación por D-pad/stick + botones; pausa; cambio de cámara.
- Se conserva la sensibilidad analógica original; la configuración solo escala
  si el usuario lo pide.

---

## 9. Estructura del proyecto final

```
RM7-Native/
  racermax7.exe
  config/rm7.ini
  data/                  (shaders propios, fuentes, iconos)
  gamedata/              (extraído por el usuario; no se distribuye)
  save/                  (en %LOCALAPPDATA%)
  docs/
  tools/
```

No se requiere RPCS3 ni herramientas de desarrollo para ejecutar una versión
compilada.

---

## 10. Hoja de ruta y criterios de aceptación

| Etapa | Objetivo | Criterio de aceptación | Estimación |
|---|---|---|---|
| 4.1 | TOC + extracción | `toc.json` válido; activos extraídos y verificados | 2–5 días |
| 4.2 | Arranque PPU | ejecuta el entry y llega a la primera llamada HLE | 1–3 semanas |
| 4.3 | Cobertura HLE | menús/servicios necesarios sin stubs silenciosos | 2–4 semanas |
| 4.4 | Primer frame RSX | imagen correcta de un frame estático | 2–4 semanas |
| 4.5 | Carga de activos | coche/circuito cargados desde `gamedata/` | 2–4 semanas |
| 4.6 | Audio | música 5.1 + voces + motor dinámico | 1–3 semanas |
| 4.7 | Input | mando y teclado con mapeo y menús | 1 semana |
| 4.8 | Menús | navegación y selección completas | 2–4 semanas |
| 4.9 | Primera carrera | salida, conducción, derrape, nitro, rivales, meta | 3–6 semanas |
| 4.10 | Fidelidad | comparación con RPCS3 y corrección de diferencias | 4–8 semanas |
| 5 | Optimización | 1080p60 estable; 1440p/4K opcional | 3–6 semanas |
| 6 | Distribución | instalador, docs, configuración, pruebas en otro PC | 2–3 semanas |

**Total orientativo:** 4–8 meses hasta la primera carrera; 9–15 meses hasta un
port completo y pulido, condicionado al SPU y a los formatos internos.

Estas cifras son estimaciones de ingeniería, no compromisos. Se revisarán tras
el inventario real del ELF.

---

## 11. Riesgos y mitigaciones

| Riesgo | Mitigación |
|---|---|
| SPURS parcial | priorizar inventario; contribuir fixes upstream; fallback documentado |
| Shaders no traducibles | corpus de shaders; corrección por programa; pruebas de frame |
| Escala del lifter | TU múltiples; compilación incremental; caché |
| Diferencias de física | telemetría y comparación con RPCS3; no tocar el código original |
| Audio 5.1 | FFmpeg + WASAPI; pruebas de canal y latencia |
| Rendimiento | perfilado, culling original, render scale, no reescribir la lógica |
| Legal | datos del usuario; sin claves; sin activos en el repo |

---

## 12. Primer entregable tras el desbloqueo

Un informe con:

1. Número de funciones, imports y librerías.
2. Cobertura de ISA y lista de instrucciones no liftadas.
3. Imágenes SPU y clasificación.
4. TOC real y muestra de activos extraídos.
5. Primer arranque hasta la primera llamada HLE, con log.
6. Riesgos revisados y estimación actualizada.
