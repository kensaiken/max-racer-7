# Guia de depuracion (VS + build_dbg)

Objetivo de esta sesion de depuracion: responder **una** pregunta que las sondas de
memoria ya no pueden separar:

> La escena tiene objetos (13/16 slots) y el fence funciona, pero no se emite ni un
> comando de geometria. ¿No se llama al envio de cada objeto, o se llama y descarta
> todos?

## 1. Preparacion

- Build con simbolos: `build_dbg\MyGameRecomp.exe` + `MyGameRecomp.pdb`
  (configurado con `-O1 -g`; el Release de `build\` no se toca).
- Abrir en VS la carpeta del proyecto (`rm7_work\port`) o el `CMakeLists.txt`.
- **No** hace falta configurar VS para lanzar: usa `run_debug.cmd` y luego
  *Depurar > Asociar al proceso* (`Attach to Process`) sobre `MyGameRecomp.exe`.
  VS encontrara el `.pdb` en la misma carpeta.
- El ISO debe estar montado en `G:` (si no, el juego reintenta abrir `MR7.DAT`
  ~1.000.000 de veces y no llega al estado que queremos observar).

## 2. Breakpoints, en este orden

| # | Funcion (archivo liftado) | Que responde |
|---|---|---|
| 1 | `func_000BED40` (`ppu_recomp_001.cpp`) | ¿El hilo de dibujo entra al render en cada frame? |
| 2 | `func_000E244C` (`ppu_recomp_002.cpp`) | ¿Se llama al envio de la lista? |
| 3 | `func_000E2264` (`ppu_recomp_002.cpp`) | ¿Se construye alguna entrada de dibujo? |
| 4 | `func_000BE0CC` (`ppu_recomp_001.cpp`) | ¿Con que argumento entra el frame? (0 = rama de vblank) |

Sugerencia: ponerlos con accion **"Continue"** y *Hit Count* activado, para no parar
en cada frame; luego mirar el contador en la ventana *Breakpoints*.

## 3. Que mirar en cada uno

Todo el estado del invitado se lee desde la memoria del host: `vm_base + EA`, y los
enteros estan en **big-endian**, asi que en la ventana *Watch* hay que envolverlos:

```
_byteswap_ulong(*(unsigned int*)(vm_base + 0x10789DB0))
```

### En `func_000E2264` (la clave)

`ctx->gpr[4]` es el EA del descriptor del objeto. Añadir a *Watch*
(sustituyendo `EA` por el valor de `ctx->gpr[4]`):

```
_byteswap_ulong(*(unsigned int*)(vm_base + EA + 0x00))   // index/estado
_byteswap_ulong(*(unsigned int*)(vm_base + EA + 0x04))   // cuenta de vertices?
_byteswap_ulong(*(unsigned int*)(vm_base + EA + 0x08))   // formato
_byteswap_ulong(*(unsigned int*)(vm_base + EA + 0x0C))   // modo (comparado con 2)
   *(unsigned char*)(vm_base + EA + 0x10)                // FLAG de visibilidad (comparado con 1)
_byteswap_ulong(*(unsigned int*)(vm_base + EA + 0x20))
_byteswap_ulong(*(unsigned int*)(vm_base + EA + 0x24))
_byteswap_ulong(*(unsigned int*)(vm_base + EA + 0x28))
```

La funcion escribe el flag 1 en la salida solo si `EA+0x10 >= 1`; si es 0, la entrada
sale "vacia". **Si el flag es 0, ese es el motivo del frame negro** y el siguiente
paso es ver quien deberia ponerlo a 1.

### En `func_000BE0CC`

```
ctx->gpr[3]    // argumento: 0 -> rama vblank (0xBE320)
```

## 4. Que me tienes que pasar

1. ¿Se alcanzan los 4 breakpoints? (si/no y, si hay hit count, cuantas veces por
   segundo).
2. En el primero que se alcance en `func_000E2264`: el EA del descriptor y los 8
   valores de la tabla de arriba.
3. En `func_000BE0CC`: el valor de `ctx->gpr[3]`.

Con eso se decide en un paso si el problema es "no se llama" o "se llama y el flag de
visibilidad esta a cero", y en el segundo caso se busca quien lo escribe.
