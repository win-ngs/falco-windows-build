# falco-windows-build

This repository provides pre-compiled Windows binaries of [Falco](https://github.com/smithlabcode/falco) 1.3.0.

These are unofficial Windows builds. They are not produced, endorsed, or
supported by the upstream [Falco](https://github.com/smithlabcode/falco) project.

Falco is a command-line quality control tool for sequencing read files such as FASTQ files. The upstream project is built mainly in Unix-like environments. This repository packages `falco.exe` so Windows users can download and run it more easily.

The release ZIP files include `falco.exe` and the MSYS2 DLLs required at runtime. You do not need to add MSYS2 to your Windows `PATH` just to run the downloaded binaries.

## Download

Download the Windows binaries from this release page:

https://github.com/tus-kondolab/falco-windows-build/releases/tag/v1.3.0-windows

Direct downloads:

- [`falco-1.3.0-msys.zip`](https://github.com/tus-kondolab/falco-windows-build/releases/download/v1.3.0-windows/falco-1.3.0-msys.zip)
- [`falco-1.3.0-ucrt64-hts.zip`](https://github.com/tus-kondolab/falco-windows-build/releases/download/v1.3.0-windows/falco-1.3.0-ucrt64-hts.zip)

## Which Build Should I Use?

If you only need to analyze FASTQ or gzip-compressed FASTQ files, start with `falco-1.3.0-msys.zip`.

If you need htslib-backed input support, for example for BAM/HTS-related workflows, use `falco-1.3.0-ucrt64-hts.zip`.

| ZIP file | Build environment | htslib | Recommended use |
|---|---|---:|---|
| `falco-1.3.0-msys.zip` | MSYS2-MSYS | No | Simple FASTQ / gzip FASTQ use |
| `falco-1.3.0-ucrt64-hts.zip` | MSYS2-UCRT64 | Yes | Workflows that need htslib support |

## How To Run Falco On Windows

1. Download one of the ZIP files from the release page.
2. Extract the ZIP file.
3. Open PowerShell or Command Prompt.
4. Change into the extracted folder.
5. Run `falco.exe`.

PowerShell example:

```powershell
cd C:\path\to\falco-1.3.0-msys
.\falco.exe --version
.\falco.exe --help
```

Analyze a FASTQ file:

```powershell
.\falco.exe C:\path\to\sample.fastq.gz
```

Write output files into a specific directory:

```powershell
.\falco.exe --outdir C:\path\to\output C:\path\to\sample.fastq.gz
```

Keep the included DLL files in the same folder as `falco.exe`. If you move only `falco.exe` without the DLLs, Windows may not be able to start it.

## What Is Included In The ZIP Files?

Each ZIP file contains:

- `falco.exe`
- Runtime DLLs reported by `ldd`

The ZIP files do not include Windows system DLLs such as `KERNEL32.DLL`, `KERNELBASE.dll`, `ucrtbase.dll`, or similar DLLs provided by Windows itself.

Third-party runtime DLL license information is summarized in [THIRD_PARTY_LICENSES.txt](THIRD_PARTY_LICENSES.txt).

## Repository Layout

| Path | Purpose |
|---|---|
| `falco-1.3.0/` | Upstream Falco 1.3.0 source tree used for the MSYS2-MSYS build |
| `falco-1.3.0-ucrt64-patch/` | UCRT64 + htslib source tree with small Windows portability fixes |
| `falco-1.3.0-msys/` | Local release staging directory for the MSYS2-MSYS binary; ignored by Git |
| `falco-1.3.0-ucrt64-hts/` | Local release staging directory for the UCRT64 + htslib binary; ignored by Git |

## Building From Source

You only need this section if you want to rebuild the binaries yourself.

Install MSYS2 first. This repository uses two MSYS2 shell environments:

- MSYS2-MSYS shell
- MSYS2-UCRT64 shell

### Build Without htslib

Use the MSYS2-MSYS shell.

```sh
cd /c/path/to/falco-windows-build/falco-1.3.0
./configure CXXFLAGS="-O3 -Wall"
make all
```

The executable is created at:

```text
falco-1.3.0/falco.exe
```

For a redistributable folder, copy `falco.exe` and the MSYS2 DLLs reported by:

```sh
ldd ./falco.exe
```

### Build With htslib

Use the MSYS2-UCRT64 shell.

Install htslib if it is not already installed:

```sh
pacman -S mingw-w64-ucrt-x86_64-htslib
```

Then build:

```sh
cd /c/path/to/falco-windows-build/falco-1.3.0-ucrt64-patch
./configure CXXFLAGS="-O3 -Wall" --enable-hts
make HAVE_HTSLIB=1 all
```

The executable is created at:

```text
falco-1.3.0-ucrt64-patch/falco.exe
```

For a redistributable folder, copy `falco.exe` and the UCRT64 DLLs reported by:

```sh
ldd ./falco.exe
```

## Why The UCRT64 + htslib Build Needs Patches

The `falco-1.3.0-msys.zip` binary is built from the upstream Falco 1.3.0 source tree without source changes.

The `falco-1.3.0-ucrt64-hts.zip` binary is built with MSYS2-UCRT64 and htslib enabled. That combination required small source changes because Windows/UCRT64 differs from the assumptions made by the upstream source.

The main issues were:

- On Windows/UCRT64, `size_t` and `unsigned long` are different types. This can break overload resolution and template type deduction in places that compile on other platforms.
- UCRT64 does not provide the POSIX-style `mkdir(path, mode)` function signature used by the upstream source.

## UCRT64 Patch List

| File | Change | Reason |
|---|---|---|
| `falco-1.3.0-ucrt64-patch/src/FalcoConfig.hpp` | Changed `size_t read_step` and `size_t threads` to `unsigned long` | Matches the existing `OptionParser::add_opt(..., unsigned long&)` overload |
| `falco-1.3.0-ucrt64-patch/src/falco.cpp` | Replaced `mkdir(path, mode)` with `std::filesystem::create_directories()` | UCRT64's `mkdir()` does not take the POSIX mode argument |
| `falco-1.3.0-ucrt64-patch/src/Module.cpp` | Replaced `std::max(1ul, ...)` with `std::max(std::size_t{1}, ...)` | Avoids `unsigned long` vs `std::size_t` type mismatch errors |
| `falco-1.3.0-ucrt64-patch/src/OptionParser.cpp` | Replaced `std::min(2ul, options.size())` with `std::min(std::size_t{2}, options.size())` | Avoids `unsigned long` vs `std::size_t` type mismatch errors |

Each changed source location includes an English comment explaining the change and keeps the previous form as a commented-out reference.

## License

Falco is distributed under the GNU General Public License version 3.

This repository follows the upstream GPL-3.0 license. See [LICENSE](LICENSE).

Third-party DLL license information is summarized in [THIRD_PARTY_LICENSES.txt](THIRD_PARTY_LICENSES.txt).
