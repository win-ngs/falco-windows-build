# Falco for Windows: Unofficial Community Builds

![Falco for Windows banner](assets/falco-for-windows-banner.jpg)

This repository provides unofficial pre-compiled Windows binaries for
[Falco](https://github.com/smithlabcode/falco) 1.3.0.

Falco is a command-line quality control tool for next-generation sequencing
read files. The upstream Falco project is primarily built for Unix-like
environments. This repository distributes ready-to-run `falco.exe` packages for
Windows users.

These builds are not produced, endorsed, or supported by the upstream Falco
project. For Falco itself, see the upstream repository:

https://github.com/smithlabcode/falco

## Download

Download the Windows ZIP files from the release page:

https://github.com/win-ngs/falco-windows-build/releases/tag/v1.3.0-windows

| File | Recommended for |
|---|---|
| [`falco-1.3.0-msys.zip`](https://github.com/win-ngs/falco-windows-build/releases/download/v1.3.0-windows/falco-1.3.0-msys.zip) | Most users processing FASTQ or gzip-compressed FASTQ files |
| [`falco-1.3.0-ucrt64-hts.zip`](https://github.com/win-ngs/falco-windows-build/releases/download/v1.3.0-windows/falco-1.3.0-ucrt64-hts.zip) | Users who need BAM input support through htslib |

If you are unsure which one to use, start with `falco-1.3.0-msys.zip`.

## How to Use

1. Download one of the ZIP files.
2. Extract the ZIP file.
3. Open PowerShell.
4. Move into the extracted folder.
5. Run `falco.exe`.

Example:

```powershell
cd C:\Users\you\Downloads\falco-1.3.0-msys
.\falco.exe --version
.\falco.exe --help
```

Analyze a FASTQ file:

```powershell
.\falco.exe C:\data\sample.fastq.gz
```

Write output files to a specific folder:

```powershell
.\falco.exe --outdir C:\data\falco-output C:\data\sample.fastq.gz
```

Keep the extracted files together. Do not move only `falco.exe` to another
folder, because the other files in the ZIP are needed for the program to start.

## Files in the ZIP

After extracting a ZIP file, you will see `falco.exe` and several `.dll` files.

| File type | What it is | What you should do |
|---|---|---|
| `falco.exe` | The Falco program | Run this file from PowerShell or Command Prompt |
| `*.dll` files | Support files needed by `falco.exe` | Keep them in the same folder as `falco.exe` |

You do not need to open the `.dll` files. They are included so that Windows can
start `falco.exe`. If they are deleted or moved away from `falco.exe`, Falco may
fail to start.

There is no installer. To remove this Windows build, delete the extracted
folder.

## Build Differences

This repository provides two Windows builds.

| Build | Source tree | Notes |
|---|---|---|
| `falco-1.3.0-msys.zip` | `falco-1.3.0-msys-patch/` | Built in MSYS2-MSYS without htslib; includes only the shared Windows filename fix described below |
| `falco-1.3.0-ucrt64-hts.zip` | `falco-1.3.0-ucrt64-patch/` | Built in MSYS2-UCRT64 with htslib enabled; includes the same filename fix plus small UCRT64 compatibility fixes |

We recommend the non-htslib build for ordinary FASTQ work. It is the more conservative
option: apart from the shared Windows filename fix, it does not include the
additional UCRT64 source-code compatibility changes. Use the htslib build
only when you need BAM input support through htslib.

## Building from Source

You do not need to build Falco yourself if you only want to use the released
Windows binaries. This section is for maintainers or users who want to recreate
the builds.

Install [MSYS2](https://www.msys2.org/) first. Run the package installation
commands below in the MSYS2 shell named for each build. If `pacman` asks you to
select packages from a group, pressing Enter accepts the default selection.

### Build without htslib

Use the MSYS2-MSYS shell.

Install the basic build tools and zlib development files:

```sh
pacman -S --needed base-devel gcc zlib-devel
```

Build Falco:

```sh
cd /c/path/to/falco-windows-build/falco-1.3.0-msys-patch
./configure CXXFLAGS="-O3 -Wall"
make all
```

The executable is created as:

```text
falco-1.3.0-msys-patch/falco.exe
```

### Build with htslib

Use the MSYS2-UCRT64 shell.

Install the basic build tools, UCRT64 compiler, zlib, and htslib:

```sh
pacman -S --needed base-devel mingw-w64-ucrt-x86_64-gcc mingw-w64-ucrt-x86_64-zlib mingw-w64-ucrt-x86_64-htslib
```

Build Falco:

```sh
cd /c/path/to/falco-windows-build/falco-1.3.0-ucrt64-patch
./configure CXXFLAGS="-O3 -Wall" --enable-hts
make HAVE_HTSLIB=1 all
```

The executable is created as:

```text
falco-1.3.0-ucrt64-patch/falco.exe
```

## Filename fix for Windows compatibility

Both source trees contain one filename change from upstream Falco 1.3.0:

| Source trees | Change | Reason |
|---|---|---|
| `falco-1.3.0-msys-patch/` and `falco-1.3.0-ucrt64-patch/` | Renamed `src/aux.hpp` to `src/falco_aux.hpp`, then updated the include and build-file references in `src/FalcoConfig.hpp`, `Makefile.am`, and `Makefile.in` | Windows treats `AUX` as a reserved device name, so `aux.hpp` is not a safe filename on Windows |

## UCRT64 Compatibility Patch

The htslib-enabled build uses MSYS2-UCRT64. The upstream Falco 1.3.0 source did
not compile there unchanged, even after the shared filename fix, so
`falco-1.3.0-ucrt64-patch/` contains additional compatibility changes.

These extra changes are limited to Windows/UCRT64 build issues:

| File | Change | Reason |
|---|---|---|
| `src/FalcoConfig.hpp` | Changed `read_step` and `threads` from `size_t` to `unsigned long` | Fixes the UCRT64 compile error where `OptionParser::add_opt()` has no overload that accepts `size_t&` |
| `src/falco.cpp` | Replaced `mkdir(path, mode)` with `std::filesystem::create_directories()` | UCRT64 does not provide the POSIX `mkdir(path, mode)` signature |
| `src/Module.cpp` | Replaced `std::max(1ul, ...)` with `std::max(std::size_t{1}, ...)` | Avoids a type mismatch between `unsigned long` and `std::size_t` |
| `src/OptionParser.cpp` | Replaced `std::min(2ul, options.size())` with `std::min(std::size_t{2}, options.size())` | Avoids a type mismatch between `unsigned long` and `std::size_t` |

The modified source locations include comments explaining the change and keep
the previous form as a commented-out reference.

## License

Falco is distributed under the GNU General Public License version 3. This
repository follows the same GPL-3.0 license. See [LICENSE](LICENSE).

Third-party license information for bundled components is summarized in
[THIRD_PARTY_LICENSES.txt](THIRD_PARTY_LICENSES.txt).
