# DevDiv/WebTools — BinSkim Profile

- **Repository**: `https://dev.azure.com/devdiv/DevDiv/_git/a952178f-8cdc-4680-a307-a715e07ff7da`
- **Org**: devdiv (DevDiv project)
- **Multiple official pipelines** — the SDL portal aggregates BinSkim findings across all of them:
  - [`WebTools-MicroBuild`](https://devdiv.visualstudio.com/DevDiv/_build/definition?definitionId=14855) — main build (daily)
  - [`WebTools-MicroBuild-Release`](https://devdiv.visualstudio.com/DevDiv/_build/definition?definitionId=25942) — insertion build (daily, scheduled)
  - [`WebTools-Functions-Feed-Update`](https://devdiv.visualstudio.com/DevDiv/_build/definition?definitionId=27480) — Functions feed update (weekly Tuesday, scheduled)
  - [`WebTools-MicroBuild-Validation`](https://devdiv.visualstudio.com/DevDiv/_build/definition?definitionId=27490) — validation (commit-triggered)
- **BinSkim config**: Pipeline-level via Guardian/1ES Pipeline Template
- **Scan targets vary by pipeline** — main build scans Release drop; Functions-Feed-Update scans decomposed VSIX/package contents

## Known findings (as of April 2025)

### Pipeline: WebTools-Functions-Feed-Update

This is the **primary source of active BinSkim findings**. It scans decomposed package contents (VSIX verification), including third-party native binaries from Azure Functions runtime packages.

**103 raw findings, 19 survive Guardian filtering:**

| Rule | Raw | Survives Guardian | Description |
|------|-----|-------------------|-------------|
| BA2008 | 12 | **Yes (12)** | EnableControlFlowGuard |
| BA2009 | 6 | **Yes (6)** | EnableAddressSpaceLayoutRandomization |
| BA2021 | 1 | **Yes (1)** | DoNotMarkWritableSectionsAsExecutable |
| BA2001 | 10 | No | EnableLoadConfigTable |
| BA2004 | 17 | No | EnableSecureSourceCodeHashing |
| BA2015 | 9 | No | EnableHighEntropyVirtualAddresses |
| BA2022 | 30 | No | SignSecurely |
| BA2027 | 18 | No | EnableSourceLink |

#### BA2009 — SDL-required violations (6 findings)

All BA2009 findings are on **third-party binaries** from Azure Functions runtime packages, not built by WebTools:

| Binary | Instances | Locations | Origin |
|--------|-----------|-----------|--------|
| `libMonoPosixHelper.dll` | 5 | `MicroBuild/Verify/in/decomp/{1/in-proc6, 1/in-proc8, 3, 4, 5}` | Mono runtime native helper (from Functions runtime) |
| `gozip.exe` | 1 | `MicroBuild/Verify/in/decomp/3` | Go-compiled binary (from Functions runtime) |

**Root cause**: `libMonoPosixHelper.dll` is a native Mono runtime component built without `/DYNAMICBASE`. `gozip.exe` is a Go binary — the Go compiler does not emit PE DYNAMICBASE by default on older versions. Both are **upstream/third-party** — WebTools consumes these via Azure Functions runtime packages and cannot directly fix them.

#### BA2008 and BA2021 (non-portal, survive Guardian)

12 BA2008 (Control Flow Guard) and 1 BA2021 (writable+executable sections) findings also survive Guardian filtering but are not currently highlighted on the SDL remediation portal for this repo. These are also on third-party binaries in the decomposed package contents.

### Pipeline: WebTools-MicroBuild (main build)

9 raw findings, **all filtered by Guardian** (none survive to Results.sarif):

| Rule | Count | Binaries |
|------|-------|----------|
| BA2004 | 4 | `Microsoft.SqlServer.BatchParser.dll` |
| BA2024 | 1 | VS ExtensionEngine/ExtensionManager DLLs |
| BA2025 | 1 | VS ExtensionEngine/ExtensionManager DLLs |
| BA2026 | 1 | VS ExtensionEngine/ExtensionManager DLLs |
| BA2027 | 2 | VS ExtensionEngine/ExtensionManager DLLs |

These are all on **foreign binaries** (SQL Server, VS Extension host) bundled in the drop, not WebTools-owned. Guardian filters them because these rules are not in the SDL-required set for this org, or because the binaries are recognized as third-party.

BinSkim only runs on the Release leg; the Debug leg has no BinSkim output.

### Pipelines: WebTools-MicroBuild-Release & WebTools-MicroBuild-Validation

These pipelines **do not currently run BinSkim**. Their SDL artifacts contain only antimalware, SARIF pattern matcher, and break tools — no `binskim/` folder.

The SDL remediation portal shows BA2009 + BA2016 findings attributed to these pipelines for two test binaries:

| Binary | Rules | Location |
|--------|-------|----------|
| `Interop.VxWebsiteExtensibility.dll` | BA2009, BA2016 | `test/ProjectSystem/WebSiteWap.IntegrationTests/TestSolution/WebSiteDteTests/DTETest/Bin` |
| `LibReferenced.dll` | BA2009, BA2016 | `test/ProjectSystem/WebSiteWap.IntegrationTests/TestSolution/WebSiteDteTests/assembly` |

These findings are **stale/cached** from older builds when BinSkim was enabled in these pipelines. Current builds do not reproduce them. The portal retains historical findings until a new clean scan replaces them.

- **BA2016** (MarkImageAsNXCompatible) is **not present in any current pipeline's raw SARIF** — the portal is showing cached results only.

## Quirks

- **Multi-pipeline aggregation**: The SDL portal aggregates findings across ALL official pipelines for a repository, not just the main build. Investigating a single pipeline is insufficient — you must check all pipelines listed on the portal.
- **SDL artifact naming varies**: Main build uses `drop_Primary_Release_sdl_analysis`, Functions uses `drop_UpdateFunctionsFeed_UpdateFeed_sdl_analysis`, Release uses `drop_Stage_N_Job_1_sdl_analysis`.
- **Stale portal findings**: The Release and Validation pipelines no longer run BinSkim, but the portal retains their historical findings. These will persist until either BinSkim is re-enabled and produces a clean scan, or the portal data is manually cleared.
- **Third-party binary dominance**: All active findings are on third-party binaries (Mono runtime, Go toolchain, SQL Server, VS Extension host). WebTools does not own or build any of the flagged binaries.
- **VSIX decomposition paths**: The Functions-Feed-Update pipeline decomposes VSIX packages for verification, creating paths like `MicroBuild/Verify/in/decomp/N/` where the same binary may appear multiple times under different decomposition roots.
