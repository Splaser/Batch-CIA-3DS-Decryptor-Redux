<h3 align="center">Batch CIA 3DS Decryptor Redux - CXI Auto Fork</h3>
<p align="center">
  <a href="https://github.com/xxmichibxx/Batch-CIA-3DS-Decryptor-Redux">
    <img src="https://i.imgur.com/tm9OXKI.png" alt="Logo" width="100" height="100">
  </a>
</p>
<h3 align="center">Automatic CIA to CXI conversion and Azahar/Citra Update/DLC loose-layout extraction</h3>
<hr>

This fork is based on **Batch CIA 3DS Decryptor Redux** by xxmichibxx, which itself is a rewritten version of the original Batch CIA 3DS Decryptor by matiffeder.

Original thread:  
https://gbatemp.net/threads/batch-cia-3ds-decryptor-a-simple-batch-file-to-decrypt-cia-3ds.512385/

Upstream project:  
https://github.com/xxmichibxx/Batch-CIA-3DS-Decryptor-Redux

## What this fork adds

This fork adds a PowerShell-based conversion workflow for emulator use.

Main goals:

- Convert game CIA files directly into launchable `.cxi`
- Automatically distinguish game CIA from Update/DLC CIA
- Preserve useful title metadata in logs and CSV reports
- Generate Azahar/Citra-compatible loose SD layouts for Update/DLC packages when normal CIA installation fails

## Features

### Game CIA

For normal game CIA files, the script:

- Decrypts the CIA using the bundled Redux toolset
- Extracts NCCH contents
- Detects the main executable CXI-like content
- Writes the result to:

```text
_cxi_out/cxi/<game>.cxi
```

The generated `.cxi` can usually be launched directly in Azahar/Citra.

### Update and DLC CIA

Update/DLC packages are detected as install-only titles.

Title family mapping:

```text
Game:   00040000XXXXXXXX
Update: 0004000eXXXXXXXX
DLC:    0004008cXXXXXXXX
```

For example:

```text
Install TitleId:        0004008c00078a00
Expected Base TitleId:  0004000000078a00
```

The script first attempts a best-effort decrypted install CIA rebuild.  
If `makerom` rejects the decrypted NCCH content, the script falls back to an Azahar/Citra loose SD layout.

### Azahar/Citra loose SD layout

When CIA installation does not work, this fork can generate loose `.app` files under:

```text
_cxi_out/sd_install/title/
```

Example output:

```text
_cxi_out/sd_install/title/0004000e/001acb00/content/00000000.app
_cxi_out/sd_install/title/0004000e/001acb00/content/00000001.app

_cxi_out/sd_install/title/0004008c/00078a00/content/00000000.app
_cxi_out/sd_install/title/0004008c/00078a00/content/00000001.app
...
```

Copy the generated `title` folder into your Azahar/Citra virtual SD path:

```text
Nintendo 3DS/
  00000000000000000000000000000000/
    00000000000000000000000000000000/
      title/
```

## Important note about `.app` names

Real CIA metadata uses ContentId values, but Azahar direct-CXI loose lookup expects index-based file names.

So this fork writes loose `.app` files using ContentInfo index order:

```text
ContentInfo index 0x0000 -> 00000000.app
ContentInfo index 0x0001 -> 00000001.app
ContentInfo index 0x0002 -> 00000002.app
```

This matters because some update CIAs may contain mappings like:

```text
ContentInfo 0x0000 / ContentId 00000003
ContentInfo 0x0001 / ContentId 00000002
```

For Azahar loose loading, those should still become:

```text
00000000.app
00000001.app
```

The script also writes a mapping CSV:

```text
_cxi_out/sd_install/loose_map_<titleid>.csv
```

## Output layout

```text
_cxi_out/
  cxi/
    <game>.cxi

  cci/
    <game>.cci

  cia_install/
    <title> [0004000eXXXXXXXX].decrypted.cia
    <title> [0004008cXXXXXXXX].decrypted.cia

  sd_install/
    title/
      0004000e/
        XXXXXXXX/
          content/
            00000000.app
            00000001.app

      0004008c/
        XXXXXXXX/
          content/
            00000000.app
            00000001.app
            ...

    loose_map_<titleid>.csv

  logs/
    report_YYYYMMDD_HHMMSS.csv
    work_xxx/
```

## Usage

Put your `.cia` files in the project root folder, then run:

```powershell
powershell -ExecutionPolicy Bypass -File .\Convert-CIA-To-CXI.ps1 -Force
```

By default, the script scans the current folder for `.cia` files and writes output to:

```text
.\_cxi_out
```

### Common commands

Scan current folder:

```powershell
powershell -ExecutionPolicy Bypass -File .\Convert-CIA-To-CXI.ps1 -Force
```

Scan recursively:

```powershell
powershell -ExecutionPolicy Bypass -File .\Convert-CIA-To-CXI.ps1 -Recurse -Force
```

Use custom input/output folders:

```powershell
powershell -ExecutionPolicy Bypass -File .\Convert-CIA-To-CXI.ps1 -Root "E:\3DS" -OutDir "E:\3DS\_cxi_out" -Force
```

Keep work files and logs for debugging:

```powershell
powershell -ExecutionPolicy Bypass -File .\Convert-CIA-To-CXI.ps1 -Force -KeepWork
```

Generate both CXI and CCI where possible:

```powershell
powershell -ExecutionPolicy Bypass -File .\Convert-CIA-To-CXI.ps1 -Mode Both -Force
```

## Report CSV

Every run writes a report:

```text
_cxi_out/logs/report_YYYYMMDD_HHMMSS.csv
```

Useful columns include:

```text
file
status
issues
method
auto_type
install_title_id
expected_base_title_id
title_id
program_id
cxi_path
cci_path
install_cia_path
loose_install_path
log_dir
```

## Troubleshooting

### Azahar says the ROM is encrypted

Use the generated `.cxi` from:

```text
_cxi_out/cxi/
```

Do not launch the original encrypted CIA.

### Update/DLC CIA cannot be installed

Some decrypted Update/DLC CIAs may still be rejected by Azahar's CIA installer or by `makerom`.

Use the generated loose SD layout instead:

```text
_cxi_out/sd_install/title/
```

Copy this `title` folder into Azahar/Citra's virtual SD directory.

### Update does not apply

Check the Azahar log. If it says something like:

```text
Failed to open .../title/0004000e/XXXXXXXX/content/00000000.app
```

make sure the generated loose layout was copied to the correct virtual SD path.

### DLC does not appear

Check that the DLC title id matches the base game:

```text
Base game: 00040000XXXXXXXX
DLC:       0004008cXXXXXXXX
```

The last 8 hex digits must match.

### Update/DLC title mismatch

For update titles:

```text
Update: 0004000eXXXXXXXX
Base:   00040000XXXXXXXX
```

For DLC titles:

```text
DLC:    0004008cXXXXXXXX
Base:   00040000XXXXXXXX
```

If the expected base title id does not match your game, the update/DLC package is for a different title.

## Requirements

- Windows 7 SP1 x64 or newer
- PowerShell
- Visual C++ Redistributable for Visual Studio 2015
- Bundled Redux toolset:
  - `decrypt.exe`
  - `ctrtool.exe`
  - `makerom.exe`
  - `seeddb.bin`

## Notes

- This fork is mainly intended for emulator-oriented personal workflows.
- The original batch file is still useful for standard decryption tasks.
- This PowerShell workflow is focused on automatic CXI extraction and Azahar/Citra Update/DLC loose layout generation.
- TWL/DSi CIAs are not useful for current 3DS emulators. Use a DSi emulator such as melonDS for TWL titles.
- Do not commit real ROM/CIA/CXI/CCI output files to the repository.

## Recommended `.gitignore`

```gitignore
_cxi_out/
*.cia
*.3ds
*.cci
*.cxi
bin/tmp.*
bin/__cia_build_*/
__cxi_stage*
```

## Credits

- `Batch CIA 3DS Decryptor Redux` - [xxmichibxx](https://github.com/xxmichibxx/Batch-CIA-3DS-Decryptor-Redux)
- `Batch CIA 3DS Decryptor` - [matiffeder](https://github.com/matiffeder/3DS-stuff)
- `CTRTool.exe / makerom.exe` - [3DSGuy](https://github.com/3DSGuy/Project_CTR)
- `seeddb.bin` - [ihaveamac](https://github.com/ihaveamac/3DS-rom-tools/tree/master/seeddb)
- `decrypt.exe` - [davidmorom](https://github.com/davidmorom)
