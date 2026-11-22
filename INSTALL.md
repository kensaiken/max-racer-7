# Installing Max Racer 7 (v0.0.3)

🇬🇧 **English** · 🇪🇸 [Español](INSTALACION.md)

> Alpha build in an early development stage. Expect bugs.

## 1. Requirements

- Windows 10 or 11, 64-bit.
- A graphics card with DirectX 12.
- [Microsoft Visual C++ Redistributable 2015-2022 x64](https://aka.ms/vs/17/release/vc_redist.x64.exe).
- Your own original copy of Ridge Racer 7 for PS3, **European version BCES00009**.

## 2. Download

1. Go to [Releases](https://github.com/kensaiken/max-racer-7/releases) and download `MaxRacer7-v0.0.3.zip`.
2. Extract it to a regular folder, for example `C:\Games\MaxRacer7`
   (avoid `Program Files`).

## 3. Prepare the game disc

Either of these works:

- An **ISO** of your disc (for example `RR7.iso`). The launcher mounts it for you.
- A **folder** with the disc contents. It must contain
  `PS3_GAME\USRDIR\RR7.DAT`.

## 4. Get the decrypted EBOOT.elf

The port needs the game's executable in decrypted form. It comes from **your**
disc; it is not distributed here.

With [RPCS3](https://rpcs3.net) (only needed for this step, not to play):

1. Install RPCS3 and the official PS3 firmware following its guide.
2. Menu **Utilities → Decrypt PS3 Binaries**.
3. Select `PS3_GAME\USRDIR\EBOOT.BIN` from your disc.
4. RPCS3 creates `EBOOT.elf` in the same folder.
5. Copy it into the port folder: `MaxRacer7\game\EBOOT.elf`.

The launcher checks the file's SHA-256 hash. The supported version is:

```
5faa5af92202c4054fd095029a0a06b15fb303e53dd8e2e11288fb726a4b03d5
```

If it does not match, it is most likely a different region or the file is
still encrypted (SELF).

## 5. Play

Double-click `Iniciar.bat`. If it cannot find the disc by itself, run it from
a console and pass the location with `-Disco`:

```bat
Iniciar.bat -Disco "D:\Games\RR7.iso"
Iniciar.bat -Disco "D:\Games\BCES00009"
Iniciar.bat -Disco G
```

The path is saved in `config.txt` for next time. The launcher's messages are
in Spanish for now.

### Controls

XInput (Xbox) controller recommended. Keyboard:

| Key | PS3 button |
|---|---|
| Arrows | D-pad |
| Z | ✕ (cross) |
| X | ○ (circle) |
| A | □ (square) |
| S | △ (triangle) |
| Q / W | L1 / R1 |
| 1 / 2 | L2 / R2 |
| Enter | START |
| Tab | SELECT |

## 6. Troubleshooting

| Symptom | Fix |
|---|---|
| `VCRUNTIME140.dll` missing | Install the Visual C++ Redistributable x64. |
| "No encuentro el disco del juego" (disc not found) | Pass your ISO or folder with `-Disco`. |
| EBOOT.elf mismatch warning | Use the European version BCES00009 and decrypt it with RPCS3. |
| Black screen or crash | Open an [issue](https://github.com/kensaiken/max-racer-7/issues) with `log_err.txt`. |
| Slow motion | Known in v0.0.3; performance work is in progress. |

## Support the project

[patreon.com/cw/ilesraider](https://www.patreon.com/cw/ilesraider/membership)
