# Max Racer 7 — port nativo (documentacion)

Registro tecnico del port nativo de **Max Racer 7** (titulo de carreras de PS3)
a PC, escrito en C/C++ con recompilacion estatica del ejecutable original.

## Estado actual

| Area | Estado |
|---|---|
| Arranque / ejecucion del codigo del juego | OK |
| Carga de assets y streaming | OK |
| Decodificacion de video (MPEG-PS + FFmpeg) | OK |
| Presentacion en ventana (D3D12) | OK — se ve el video de titulo |
| Tareas SPURS | OK — encoladas, arrancadas y completadas |
| Render 3D propio del juego | Bloqueado: falta la notificacion de fin de tarea |

## Documentos

- `INFORME_FASE4A.md` — bitacora tecnica completa (aprox. 100 secciones, con
  medidas, causas raiz y descartes).
- `ESTADO.md` — resumen de estado y pendientes.

## Hitos tecnicos destacados

1. Demux MPEG-PS propio + FFmpeg cargado en tiempo de ejecucion.
2. Presentador D3D12 con escalado al backbuffer (la copia sin `box` eliminaba
   el dispositivo, causa del "pantalla en blanco").
3. Reensamblado de AUs con cadena de referencias intacta (sin macrobloques).
4. Despachador nativo de tareas SPURS: sin la politica de Sony, las tareas se
   arrancan desde el hilo del scheduler; 1.700 arrancadas, 2.593 completadas.

## Aviso

Este repositorio contiene **solo documentacion tecnica**. No incluye codigo del
juego, binarios, imagenes, audio ni ningun otro material con derechos de autor.