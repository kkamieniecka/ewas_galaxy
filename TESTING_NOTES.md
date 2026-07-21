# Local `planemo test` findings (2026-07-20/21)

Notes from debugging a local `planemo test` run of `minfi_analysis.xml` under WSL2 Ubuntu on Windows. Kept here so the two issues found aren't re-discovered from scratch next time.

## 1. Manifest/annotation packages fail to load ("cannot load manifest package")

**Symptom:**

```
Loading required package: IlluminaHumanMethylation450kmanifest
Error in getManifest(object) :
  cannot load manifest package IlluminaHumanMethylation450kmanifest
Calls: preprocessRaw ... getManifestInfo -> getProbeInfo -> getManifest -> getManifest
Execution halted
```

**Root cause:** `macros.xml` pins old bioconda packages (`r-base=3.6.2`, `bioconductor-minfi=1.32.0`, and the 27k/450k/EPIC manifest + annotation packages). Those data packages ship as a raw R source tarball that only gets installed via a conda `post-link.sh` hook (`installBiocDataPackage.sh`, which downloads the tarball from bioconductor.org and runs `R CMD INSTALL`).

Planemo's local dependency resolution satisfies the tool's requirements by pulling a pre-built "mulled" combined environment (`mulled-v1-<hash>`) rather than running `conda create` package-by-package. When that mulled bundle is unpacked, the post-link scripts are never executed. Compiled packages (`minfi`, `rtracklayer`, etc.) work fine since they don't need post-link steps, but every `IlluminaHumanMethylation*manifest` / `*anno.*` package is left as an unextracted tarball under `share/`, never registered in R's `lib/R/library`.

**Fix applied (diagnostic environment only, not a tool-code change):** manually ran each of the 6 post-link scripts against the existing mulled env:

```bash
PREFIX=/path/to/miniconda3/envs/mulled-v1-<hash>
export PREFIX PATH="$PREFIX/bin:/usr/bin:/bin"
for pkg in bioconductor-illuminahumanmethylation27kmanifest \
           bioconductor-illuminahumanmethylation27kanno.ilmn12.hg19 \
           bioconductor-illuminahumanmethylation450kmanifest \
           bioconductor-illuminahumanmethylation450kanno.ilmn12.hg19 \
           bioconductor-illuminahumanmethylationepicmanifest \
           bioconductor-illuminahumanmethylationepicanno.ilm10b4.hg19; do
  bash "$PREFIX/bin/.$pkg-post-link.sh"
done
```

This downloads each package tarball from `bioconductor.org` and runs `R CMD INSTALL` — after this, `minfi_analysis`'s R script correctly loads all manifest/annotation packages and proceeds into `preprocessRaw`/`bumphunter`. Confirmed working twice, in two independent test runs.

**Implication for CI:** this may or may not reproduce on Travis — it depends on whether Travis's dependency resolution path also falls back to unpacking a pre-built mulled bundle, or actually runs `conda create` (which executes post-link scripts normally). Worth checking Travis logs for the same "cannot load manifest package" error if CI ever fails there.

## 2. `bumphunter`'s "Finding regions" step never completes locally (memory ceiling)

**Symptom:** after the manifest fix, the R script proceeds past `preprocessRaw`/QC into `bumphunter`, prints `[bumphunterEngine] Finding regions.`, and then runs indefinitely (confirmed 20+ minutes with zero further output, zero output files, no crash) despite `B=0` (no permutations) which should make this step take seconds.

**Root cause:** this specific dev machine has only 6.9GB total RAM, with the WSL2 VM capped at 5.3GB (`.wslconfig`: `memory=5632MB`). Running Galaxy's local job-runner stack (gunicorn + 2 celery processes) simultaneously with R loading the EPIC/450k annotation packages and scanning genome-wide probe positions pushes memory to the ceiling; the process survives (no OOM kill observed in dmesg/journalctl) but swaps so heavily that it doesn't practically finish.

**Reproduced twice** (two independent full test runs) at the exact same point, confirming this is a deterministic hardware/VM-memory limitation on this machine, not an intermittent fluke.

**Things that helped but weren't sufficient on their own:**
- Killing orphaned Galaxy/celery/gunicorn/node processes left running from earlier `planemo test` invocations (each full stack persists ~700MB-1GB after the parent `planemo test` process is killed, since `supervisord` daemonizes them — `pkill -f 'planemo test'` does **not** clean these up; kill `supervisord`, `celery`, `gunicorn` explicitly, or better, kill by explicit PID since `pkill -f` proved unreliable in this WSL setup)
- Patching planemo's local install to skip the `gx-it-proxy` (interactive tools) node.js process it launches for every test run — real but small savings
- Attempting to disable `enable_celery_tasks` in planemo's generated `galaxy.yml`: **does not work** — gravity/supervisord starts the celery worker/beat processes regardless of this flag; it only affects whether Galaxy internally *dispatches* tasks to celery, not whether the process runs at all

**Conclusion:** getting a real green `planemo test` run for `minfi_analysis` needs either a machine/VM with more RAM (8GB+ free recommended) or reliance on CI, which won't share this constraint. Further local retries on this machine are not expected to succeed without a memory increase.

## Environment gotchas hit while doing this (WSL2 + Git Bash on Windows)

- `wsl.exe -d Ubuntu -- bash -lc "..."` calls with `/mnt/c/...` path arguments get mangled by Git Bash's MSYS path auto-conversion unless prefixed with `MSYS_NO_PATHCONV=1`.
- Inline shell variable assignment + read within a single `bash -lc "VAR=x; echo $VAR"` string is unreliable through this particular Bash-tool → `wsl.exe` invocation path (assignment silently doesn't persist) — write a real script file and execute that instead, or use fully literal values.
- The WSL2 VM's `/tmp` is not guaranteed to survive a VM restart (observed during a low-disk-space incident) — use `--no_cleanup` and point `--job_output_files`/`--test_output_json` at a path on the Windows-mounted filesystem if you need results to survive an interruption.
