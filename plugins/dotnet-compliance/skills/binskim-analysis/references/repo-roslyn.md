# dotnet/roslyn — BinSkim Profile

- **Pipeline YAML**: `azure-pipelines-official.yml`
- **Official pipeline**: [`dotnet-roslyn-official`](https://dev.azure.com/dnceng/internal/_build/definition?definitionId=327) (dnceng/internal #327)
- **BinSkim config**: Pipeline-level via Guardian with `-ArtifactToolsList @("binskim")`
- **Scan target**: `VSSetup` drop (installer VSIXes) and `PackageArtifacts` (NuGet packages)
- **How to reproduce locally**:
  1. Build: `build.cmd -c Release`
  2. Pack: `build.cmd -c Release -pack`
  3. Scan NuGet packages and/or VSIX contents
- **Quirks**:
  - Publishes VSSetup/VSIX artifacts — these contain binaries that NuGet scanning misses
  - Has guardian suppression entries in pipeline YAML
  - BinSkim does NOT run in Roslyn's own official pipeline — it only runs when Roslyn is built as part of dotnet/dotnet (VMR), so the official pipeline has zero BinSkim output
  - Roslyn does NOT use NativeAOT; the language server uses ReadyToRun (`PublishReadyToRun=true`, `SelfContained=false`) and is packaged as a dotnet tool

## Known findings (as of April 2025)

### Windows PE binaries: clean
All Windows binaries (1,891 DLLs/EXEs across VSSetup and PackageArtifacts) pass all BinSkim rules with zero violations.

### Linux ELF binaries: SDL-required violations on musl targets only

The only SDL-required violations are **BA3003** (stack protector) and **BA3030** (Fortify Source), both on `libe_sqlite3.so` from the 3rd-party **SQLitePCLRaw** package (`SQLitePCLRaw.lib.e_sqlite3`, version set by `<SqliteVersion>` in `eng/Packages.props`). These violations are **musl-architecture-specific**:

| RID | BA3003 | BA3030 | Notes |
|-----|--------|--------|-------|
| `linux-x64` (glibc) | ✅ pass | ✅ pass | gcc enables stack protector by default |
| `linux-musl-x64` (Alpine) | ✅ pass | ❌ fail | musl libc does not support `_FORTIFY_SOURCE` |
| `linux-musl-arm64` (Alpine) | ❌ fail | ❌ fail | musl cross-compiler lacks default stack protector |

**Root cause**: The native SQLite binary is built in the [`ericsink/cb`](https://github.com/ericsink/cb) repo (`bld/cb.cs`). The musl builds use `musl-gcc` / `aarch64-linux-musl-cc` with minimal flags (`-shared -fPIC -O -DNDEBUG`), missing `-fstack-protector-strong` and `-D_FORTIFY_SOURCE=2`. However, `_FORTIFY_SOURCE` is fundamentally a glibc feature — musl libc does not implement it, so BA3030 on musl targets is **unfixable** without switching to a different libc. BA3003 on musl-arm64 may be fixable by passing `-fstack-protector-strong` explicitly to the cross-compiler.

**Impact**: Non-Alpine Linux and all Windows users have zero SDL-required violations. Only Alpine (`linux-musl-*`) deployments are affected.

Upgrading SQLitePCLRaw does not fix the musl violations — tested v2.1.6, v2.1.11, and the replacement package `SourceGear.sqlite3` v3.50.4.5; all have the same musl failures. No issues exist in the upstream repo about binary hardening.

### Non-SDL-required findings (informational)

- **BA3031** (SafeStack) on `Microsoft.CodeAnalysis.LanguageServer` ELF apphost (~77KB .NET shim) — this is a standard dotnet apphost, not NativeAOT; BA3031 is not SDL-required
- **BA3011** (BIND_NOW), **BA3004**, **BA3005** on `libe_sqlite3.so` — not SDL-required
