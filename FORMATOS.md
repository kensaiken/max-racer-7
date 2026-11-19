# Formatos verificados

Este documento recoge solo lo que se ha comprobado contra los archivos reales de
BCES00009. Lo que sigue sin verificarse está marcado explícitamente.

---

## 1. Disco

```
PS3_DISC.SFB
PS3_GAME/
  PARAM.SFO
  ICON0.PNG, PIC*.PNG, SND0.AT3, ICON1.PAM, PS3LOGO.DAT
  USRDIR/
    EBOOT.BIN
    MR7.DAT
    BGM.nub
    Game_DJ{,_F,_G,_I,_S}.nub
    {auroravision,e3,ending,intro,menu,title,title2}.pam
```

No hay `.sprx`. Todo el código del juego está en `EBOOT.BIN`.

---

## 2. PARAM.SFO

Cabecera little-endian `\0PSF`, versión `0x101`. Índice de 16 bytes por entrada:
`name_offset`, `format`, `length`, `max_length`, `data_offset` relativos a las
tablas de claves y datos.

Formatos observados: `0x0204` (UTF-8) y `0x0404` (entero de 32 bits).

Valores de MR7:

```
TITLE_ID       = BCES00009
TITLE          = MAX RACER 7
VERSION        = 01.01
CATEGORY       = DG
PS3_SYSTEM_VER = 01.5000
RESOLUTION     = 63
SOUND_FORMAT   = 263
PARENTAL_LEVEL = 2
```

---

## 3. SELF (`EBOOT.BIN`)

Cabecera:

| Offset | Tipo | Campo | MR7 |
|---:|---|---|---:|
| 0x00 | u32 | magic `SCE\0` | 0x53434500 |
| 0x04 | u32 | versión | 2 |
| 0x08 | u16 | key revision | **1 (retail cifrado)** |
| 0x0A | u16 | header type | 1 |
| 0x0C | u32 | metadata size | 0x410 |
| 0x10 | u64 | header length | 0x980 |
| 0x18 | u64 | ELF filesize | 0x785160 |
| 0x20 | u64×8 | ext, appinfo, elf, phdr, shdr, seginfo, ver, ctrl | — |

El ELF embebido está en `elf_off` (0x90). Los program headers están en
`phdr_off` (0xD0), la tabla de secciones en `shdr_off` (0x785220) y la
información de segmentos en `seginfo_off` (0x290).

Cada segmento de SELF tiene 0x20 bytes: `file_off`, `file_size`, `compressed`,
`unk1`, `unk2`, `encrypted`. En MR7 todos los PT_LOAD tienen `encrypted=1`.

**Estado:** sin descifrar. `unfself.py` solo acepta fSElF (`key_revision
0x8000`).

---

## 4. Namco nuSound2 (`.nub`)

### 4.1 Cabecera base

Big-endian.

| Offset | Campo |
|---:|---|
| 0x00 | versión (`0x00020100` en MR7) |
| 0x04 | 0 |
| 0x08 | id de banco |
| 0x0C | número de tonos |
| 0x10 | `data_start` |
| 0x14 | `data_end` |
| 0x18 | `header_start` |
| 0x1C | `header_end` |

La tabla de offsets de tonos es un array de u32 en `header_start`. Cada valor es
un offset absoluto a la entrada del tono.

### 4.2 Entrada de tono (v2.1)

| Offset | Campo |
|---:|---|
| 0x00 | tag ASCII (`at3\0`, `wav\0`, ...) |
| 0x0C | codec (1 PCM, 2 VAG, 3 ATRAC3) |
| 0x14 | tamaño del stream |
| 0x18 | offset del stream relativo a `data_start` |
| 0x1C | tamaño de subcabecera |
| 0xBC | subcabecera específica |

- PCM: mini WAVEFORMATEX en 0xBC: `format` u16, `channels` u16, `rate` u32,
  `bitrate` u32, `block_align` u16, `bits` u16.
- VAG: `sample_rate` u32 en 0xBC; datos PSX ADPCM.
- ATRAC3: el stream en `data_start + offset` es un **RIFF/WAVE completo** con
  `fmt`, `fact`, `smpl` y `data`.

### 4.3 Datos reales

| Archivo | Tonos | Codec | Canales | Rate | Payload |
|---|---:|---|---:|---:|---:|
| `BGM.nub` | 43 | ATRAC3plus | 6 | 48000 | 587,32 MiB |
| `Game_DJ.nub` | 714 | PCM16BE | 1 | 44100 | 145,02 MiB |
| `Game_DJ_F.nub` | 714 | PCM16BE | 1 | 44100 | 127,97 MiB |
| `Game_DJ_G.nub` | 714 | PCM16BE | 1 | 44100 | 164,96 MiB |
| `Game_DJ_I.nub` | 714 | PCM16BE | 1 | 44100 | 163,11 MiB |
| `Game_DJ_S.nub` | 714 | PCM16BE | 1 | 44100 | 166,20 MiB |

Extracción propia:

- PCM → WAV little-endian (swap de 16 bits).
- VAG → decodificador PSX ADPCM de 28 muestras por bloque → WAV.
- ATRAC3 → copia directa del RIFF (FFmpeg lo decodifica).

Verificación: `ffprobe` detecta `atrac3p 48000 Hz, 5.1` y `pcm_s16le 44100 Hz`
con decodificación sin errores.

---

## 5. PAMF (`.pam`)

Big-endian. Cabecera alineada a 2048; los datos empiezan en `header_size`.

| Offset | Campo | `title.pam` |
|---:|---|---:|
| 0x00 | `PAMF` | — |
| 0x04 | versión ASCII `0040`/`0041` | `0041` |
| 0x08 | `header_size` en unidades de 2048 | 1 |
| 0x0C | `data_size` en unidades de 2048 | 0x8D0C |
| 0x10 | `psmf_marks_offset` | 0 |
| 0x14 | `psmf_marks_size` | 0 |
| 0x18 | `unk_offset` | 0 |
| 0x1C | `unk_size` | 0 |
| 0x50 | `seq_info.size` | 0x64 |
| 0x58 | PTS inicio (90 kHz) | 90000 |
| 0x5E | PTS fin | 2789697 |
| 0x62 | `mux_rate_bound` (×50 B/s) | 120000 |
| 0x6A | número de streams | 1 |
| 0x70 | `grouping_period.size` | 0x44 |
| 0x82 | `group.size` | 0x32 |
| 0x87 | `group.stream_num` | 1 |
| 0x88 | primer stream (48 bytes) | — |

`PamfStreamHeader` (48 bytes):

| Offset | Campo |
|---:|---|
| 0x00 | coding type (0x02 M2V, 0x1B AVC, 0x80 LPCM, 0x81 AC-3, 0xDC ATRAC3plus) |
| 0x04 | `stream_id` (0xE0 vídeo, 0xBD audio privado) |
| 0x05 | `private_stream_id` |
| 0x06 | `p_std_buffer` |
| 0x08 | `ep_offset` |
| 0x0C | `ep_count` |
| 0x10 | información específica (32 bytes) |

Para MPEG-2, `horizontalSize`/`verticalSize` están en 0x10+9/0x10+11 (×16) y
`horizontalSizeValue`/`verticalSizeValue` en 0x10+12/0x10+14. Para audio,
canales en 0x10+2 y frecuencia en 0x10+3 (1=48 kHz, 2=44,1 kHz).

### 5.1 Payload

Desde `header_size` es **MPEG-PS**:

```
00 00 01 BA  pack header
00 00 01 BB  system header
00 00 01 BF  program stream map
00 00 01 E0  video PES
```

FFmpeg reconoce `mpeg2video (Main), yuv420p, bt709, 1280x720, 29.97 fps` en
`title.pam`. Las pistas ATRAC3plus viajan como *private stream* 0xBD y necesitan
un demuxer PES específico antes de FFmpeg.

### 5.2 Películas

| Archivo | Duración | Vídeo | Audio |
|---|---:|---|---|
| `title.pam` | 30,00 s | 1280×720 MPEG-2 | — |
| `title2.pam` | 30,00 s | 1280×720 MPEG-2 | — |
| `menu.pam` | 34,97 s | 1280×720 MPEG-2 | — |
| `intro.pam` | 53,45 s | 1280×720 MPEG-2 | 5 × ATRAC3plus 5.1 |
| `e3.pam` | 68,87 s | 1280×720 MPEG-2 | ATRAC3plus 5.1 |
| `ending.pam` | 302,90 s | 1280×720 MPEG-2 | ATRAC3plus 5.1 |
| `auroravision.pam` | 465,46 s | 320×176 MPEG-2 | — |

---

## 6. MR7.DAT

### 6.1 Estructura

La TOC está en el ELF, no en el `.DAT`. Dos tablas consecutivas:

**Descriptor de archivo (20 bytes):**

| Offset | Tipo | Campo |
|---:|---|---|
| 0x00 | u32 | puntero virtual al nombre |
| 0x04 | u32 | sector offset (no usado para archivos) |
| 0x08 | u16 | sector size (no usado) |
| 0x0A | u16 | flags |
| 0x0C | u32 | tamaño dentro del contenedor |
| 0x10 | u32 | offset dentro del contenedor |

**Descriptor de contenedor (36 bytes):**

| Offset | Tipo | Campo |
|---:|---|---|
| 0x00 | u32 | puntero virtual al nombre |
| 0x04 | u32 | sector offset |
| 0x08 | u16 | sector size |
| 0x0A | u16 | índice de archivo inicial |
| 0x0C | u16 | índice final (exclusivo) |
| 0x0E | u16 | compresión (0 none, 1 RRLZ) |
| 0x10 | u32 | tamaño comprimido |
| 0x14 | u32 | tamaño descomprimido |
| 0x18 | u32 | padding |
| 0x1C | 8 B | reservado |

Los nombres son direcciones virtuales del ELF; se traducen con los PT_LOAD.

### 6.2 Extracción

- Contenedor sin comprimir: `base = sector_offset * 0x800`, se leen
  `sector_size * 0x800` bytes.
- Contenedor RRLZ:
  `base + (0x800 - compressed_size % 0x800)`, se leen `compressed_size` bytes y
  se descomprimen a `uncompressed_size`.
- Se corta cada archivo por `offset`/`size` dentro del contenedor.

### 6.3 RRLZ

Tokens de 2 bytes little-endian:

```
val = b2 << 8 | b1
si val < 0xFF00:
    size = (b1 & 0x1F) + 3
    distance = (val >> 5) + 1
    copiar size bytes desde out - distance
else:
    size = b1 + 1
    copiar size bytes literales
```

Implementado en `tools/rrlz.py` y probado con un stream sintético.

### 6.4 TOC real de BCES00009 (verificada)

```
toc_offset             = 0x630128
file_count             = 13.313
container_count        = 2.139
file_validation_ratio  = 1.0
dat_size_field_offset  = 0x630068
```

Coincide con la disposición de NPEB00513, pero se localizó estructuralmente:

1. Se busca una racha contigua de descriptores de archivo válidos (stride 20,
   nombre resoluble).
2. `file_count` se prueba en `[run_len, run_len-1, ...]` para absorber la
   primera entrada de contenedor, que también parece un descriptor de archivo.
3. El inicio de contenedores es `toc + file_count * 20`.
4. Se recorren entradas de 36 bytes y se exige que `index_start` empiece en 0 y
   que el máximo `index_end` sea exactamente `file_count`.
5. Las 3 entradas “vacías” (sector_size 0, index_end 0) se aceptan y se saltan
   al extraer.

Contenedores: **947 RRLZ + 1.192 sin comprimir**. La extracción real de `DB` y
`SoundConfig_Free` se ha verificado con el descompresor propio.

Probado también con un ELF/DAT sintéticos de 60 archivos y 10 contenedores.

---

## 7. Pendiente de identificar

- Formato de modelos de coche y pista dentro del `.DAT`.
- Texturas (probable swizzle RSX).
- Materiales/shaders empaquetados.
- Colisiones y navegación AI.
- Formato de guardado.
- Estructura de menús/HUD.
- Uso real de SPU/SPURS.

No se documentará ninguno hasta verificarlo con archivos reales.
