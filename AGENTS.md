# AGENTS.md

## Cursor Cloud specific instructions

### What this repo is
`MemProcFS-Analyzer` is a **Windows-only PowerShell DFIR tool** that orchestrates
[MemProcFS](https://github.com/ufrisk/MemProcFS) plus many bundled CLI parsers to
analyze Windows memory dumps. The product itself (WinForms GUI, Dokany-mounted
memory image, `.NET 9` EZTools, MemProcFS Windows binaries, ClamAV, optional
Elasticsearch/Kibana) **cannot run end-to-end on this Linux cloud VM** — it
requires Windows, Administrator rights, and a memory dump. See `README.md` for
the real Windows setup (`Updater.ps1`, Dokany, `.NET 9`, IPinfo token, etc.).

The source that developers actually edit here is PowerShell:
`MemProcFS-Analyzer.ps1`, `Updater.ps1`, `Scripts/Get-ProcessTree/*.ps1`,
`Scripts/ProcessesAndModules-Extended_Info.ps1`. There is one cross-platform
bundled tool, `Scripts/1768/1768.py` (Didier Stevens' Cobalt Strike beacon
config analyzer), which does run on Linux.

### What is pre-installed (via VM snapshot, do NOT reinstall manually)
- **PowerShell 7.x (`pwsh`)** — installed from the Microsoft apt repo. This is the
  runtime used to lint/parse the `.ps1` scripts.
- **PSScriptAnalyzer** PowerShell module (CurrentUser scope) — the linter.
- **Python 3.12** with `pefile`/`peutils` — required by `Scripts/1768/1768.py`.

### Lint (PowerShell scripts)
```bash
pwsh -NoProfile -Command "Get-ChildItem -Recurse -Include *.ps1 -File | Where-Object { \$_.FullName -notlike '*/Tools/*' } | ForEach-Object { Invoke-ScriptAnalyzer -Path \$_.FullName -Severity Error,Warning } | Group-Object Severity | Format-Table"
```
Note: the existing scripts already produce warnings and a few `Error`-severity
findings (`PSAvoidUsingComputerNameHardcoded`) on the current tree. These are
**pre-existing** — treat them as the baseline, not something introduced by your
changes.

### "Build" (there is no compile step — validate scripts parse)
PowerShell is interpreted; the closest thing to a build is a parser check:
```bash
pwsh -NoProfile -Command "\$e=\$null; [System.Management.Automation.Language.Parser]::ParseFile((Resolve-Path ./MemProcFS-Analyzer.ps1),[ref]\$null,[ref]\$e); \$e"
```
No output = parses clean. Do the same for `Updater.ps1` and the `Scripts/*.ps1`.

### Run / test (only cross-platform component runnable on Linux)
Only `Scripts/1768/1768.py` runs here. Analyze a Cobalt Strike beacon/shellcode
blob (`-r` = raw scan, `-S` = drop configs that fail the sanity check):
```bash
python3 Scripts/1768/1768.py -r -S <sample.bin>
```
Everything else (`MemProcFS-Analyzer.ps1`, `Updater.ps1`, EZTools, ClamAV, ELK)
needs a Windows host with Dokany + MemProcFS and a memory image; it is expected
to be **impossible to execute here**. Do not spend time trying to run the GUI
pipeline on Linux — validate PowerShell changes with lint + parse instead.

### Gotchas
- Skip the `Tools/` directory when linting/parsing; it holds bundled binaries and
  batch files (`.reb`), not first-party PowerShell to analyze.
- `Updater.ps1` downloads Windows binaries into `Tools/` and installs the
  `ImportExcel` PS module; running it on Linux is neither needed nor supported.
- The `MemProcFS-Analyzer.ps1` file is very large (~720 KB / ~70k tokens);
  parsing/linting it takes a few seconds — that's normal.
