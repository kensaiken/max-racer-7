# Legal, descifrado y distribución

## 1. Principios

- Este repositorio **no contiene claves de Sony, firmware, código del juego ni
  activos con copyright**.
- El usuario debe poseer una copia legítima de Ridge Racer 7 (BCES00009).
- El proyecto no elude protecciones por sí mismo: trabaja sobre un ejecutable ya
  descifrado por el usuario con su propia copia.
- El ejecutable final no incluirá RPCS3 ni dependerá de él.

## 2. Por qué hace falta un ELF descifrado

El `EBOOT.BIN` del disco es un SELF retail (`key_revision 0x0001`) con los
segmentos cifrados. La recompilación estática necesita las instrucciones
PowerPC y la tabla de imports, que están dentro del payload cifrado. Sin el ELF:

- no se puede ejecutar el lifter;
- no se pueden inventariar imports ni funciones;
- no se puede localizar la TOC de `RR7.DAT`.

No se ha intentado romper el cifrado ni se han incluido claves.

## 3. Cómo obtener el ELF (copia propia)

### Opción A — RPCS3 (referencia recomendada por ps3recomp)

1. Instalar RPCS3 y configurar el firmware que el usuario ya posee.
2. Cargar el juego desde el disco propio.
3. Activar el volcado de ejecutable en las opciones de depuración (la guía de
   ps3recomp lo denomina “Dump Executable”).
4. Localizar el ELF descifrado resultante.

RPCS3 se usa solo como herramienta de referencia/volcado. El port final no lo
necesita.

### Opción B — Consola propia

Volcar el ejecutable descifrado desde una PS3 con CFW del usuario. Requiere
conocimientos y responsabilidad sobre la propia consola.

### Opción C — Claves propias

Herramientas como `scetool` pueden descifrar un SELF si el usuario aporta las
claves de su consola. El proyecto no las incluye ni las solicita.

### Colocación

```
RR7-Native/
  game/
    EBOOT.ELF
```

Comprobar:

```powershell
python tools\rr7_dat.py locate --elf game\EBOOT.ELF --dat "G:\PS3_GAME\USRDIR\RR7.DAT" --json game\toc.json
```

Si el archivo sigue siendo un SELF, la herramienta falla con “not an ELF file”.

## 4. Distribución

### Permitido

- Distribuir el código fuente y los binarios compilados del port (MIT).
- Distribuir herramientas propias y documentación.
- Distribuir shaders propios, configuración e iconos originales.

### No permitido

- Distribuir `EBOOT.BIN`, `EBOOT.ELF`, `RR7.DAT`, `.nub`, `.pam`, texturas,
  modelos, música ni voces.
- Incluir claves, firmware o código propietario de Sony/Namco.
- Empaquetar los activos extraídos en el instalador.

### Instalación del usuario

El instalador debe detectar la copia del usuario (ISO montada o carpeta
`PS3_GAME`), extraer los datos a `gamedata/` y verificar el manifest. Nunca debe
descargar activos de Internet.

## 5. Atribuciones

| Proyecto | Uso | Licencia |
|---|---|---|
| [sp00nznet/ps3recomp](https://github.com/sp00nznet/ps3recomp) | runtime, lifters, HLE, backend RSX | MIT |
| [Nenkai/RRUnpacker](https://github.com/Nenkai/RRUnpacker) | referencia de formato `.DAT`/RRLZ | MIT |
| [vgmstream](https://github.com/losnoco/vgmstream) | referencia de formato `.nub` | permisiva (ver repo) |
| [RPCS3](https://github.com/RPCS3/rpcs3) | referencia de comportamiento PAMF | GPLv2 (no se copia código) |
| [FFmpeg](https://ffmpeg.org) | decodificación MPEG-2/ATRAC3plus | LGPL/GPL según build; se carga en runtime |
| Digital Foundry | análisis técnico 1080p60 | cita editorial |

`ps3recomp` declara explícitamente que no contiene código GPL de RPCS3; este
proyecto mantiene la misma política.

## 6. Marcas

“Ridge Racer”, “Ridge Racer 7”, “PlayStation” y “Cell” son marcas de sus
respectivos propietarios. Este proyecto no está afiliado a Sony, Bandai Namco ni
NVIDIA.
