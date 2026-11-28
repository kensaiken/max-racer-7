# Instalación de Max Racer 7 (v0.0.5)

🇬🇧 [English](INSTALL.md) · 🇪🇸 **Español**

> Versión alfa en fase temprana de desarrollo. Espera fallos.

## 1. Requisitos

- Windows 10 u 11 de 64 bits.
- Tarjeta gráfica con DirectX 12.
- [Microsoft Visual C++ Redistributable 2015-2022 x64](https://aka.ms/vs/17/release/vc_redist.x64.exe).
- Tu copia original de Ridge Racer 7 para PS3, **versión europea BCES00009**.

## 2. Descargar

1. Ve a [Releases](https://github.com/kensaiken/max-racer-7/releases) y baja `MaxRacer7-v0.0.5.zip`.
2. Descomprímelo en una carpeta sin permisos especiales, por ejemplo `C:\Juegos\MaxRacer7`
   (evita `Archivos de programa`).

## 3. Preparar el disco del juego

Vale cualquiera de estas dos formas:

- **ISO** de tu disco (por ejemplo `RR7.iso`). El lanzador la monta solo.
- **Carpeta** con el contenido del disco. Dentro debe existir
  `PS3_GAME\USRDIR\RR7.DAT`.

## 4. Obtener el EBOOT.elf descifrado

El port necesita el ejecutable del juego descifrado. Sale de **tu** disco; no
se distribuye aquí.

Con [RPCS3](https://rpcs3.net) (no hace falta para jugar, solo para este paso):

1. Instala RPCS3 y el firmware oficial de PS3 siguiendo su guía.
2. Menú **Utilities → Decrypt PS3 Binaries**.
3. Elige `PS3_GAME\USRDIR\EBOOT.BIN` de tu disco.
4. RPCS3 crea `EBOOT.elf` en la misma carpeta.
5. Cópialo a la carpeta del port: `MaxRacer7\game\EBOOT.elf`.

El lanzador comprueba la huella SHA-256 del archivo. La versión soportada es:

```
5faa5af92202c4054fd095029a0a06b15fb303e53dd8e2e11288fb726a4b03d5
```

Si no coincide, probablemente es otra región o el archivo sigue cifrado
(SELF).

## 5. Jugar

Haz doble clic en `Iniciar.bat`. Si no encuentra el disco solo, ábrelo desde
una consola indicándolo:

```bat
Iniciar.bat -Disco "D:\Juegos\RR7.iso"
Iniciar.bat -Disco "D:\Juegos\BCES00009"
Iniciar.bat -Disco G
```

La ruta se guarda en `config.txt` para las siguientes veces.

### Controles

Mando XInput (Xbox) recomendado. Con teclado:

| Tecla | Botón PS3 |
|---|---|
| Flechas | Cruceta |
| Z | ✕ (cruz) |
| X | ○ (círculo) |
| A | □ (cuadrado) |
| S | △ (triángulo) |
| Q / W | L1 / R1 |
| 1 / 2 | L2 / R2 |
| Enter | START |
| Tab | SELECT |
| F11 / Alt+Enter | Pantalla completa |

## 6. Problemas frecuentes

| Síntoma | Solución |
|---|---|
| Falta `VCRUNTIME140.dll` | Instala el Visual C++ Redistributable x64. |
| "No encuentro el disco del juego" | Indica la ISO o la carpeta con `-Disco`. |
| Aviso de EBOOT.elf no coincide | Usa la versión europea BCES00009 y descífrala con RPCS3. |
| Pantalla negra o cierre | Abre un [issue](https://github.com/kensaiken/max-racer-7/issues) con `log_err.txt`. |
| Va a cámara lenta | Conocido en v0.0.5; se está trabajando en el rendimiento. |
| Se queda colgado en el logo | Pasa rara vez; cierra el juego y vuelve a abrirlo. |
| Ventana negra en OBS | Usa **Captura de ventana** en vez de Captura de juego. |

## Apoya el proyecto

[patreon.com/cw/ilesraider](https://www.patreon.com/cw/ilesraider/membership)
