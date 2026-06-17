# Qwen3.6 MTP Windows ROCm gfx1151 Split Package

Hardware target: this package is built for AMD Ryzen AI Max Series APUs with
RDNA 3.5 graphics (`gfx1151`). 

It is not a generic ROCm runtime for other AMD GPU architectures.

The deployable runtime trees are distributed as GitHub Release assets, not as
files committed to the git tree.

- [llama-b1294-windows-rocm-gfx1151-x64.zip](https://github.com/akqmffl/llama.cpp-lemonade-Runtime-for-qwen-3.6-MTP/releases/download/b1294/llama-b1294-windows-rocm-gfx1151-x64.zip):
  standalone llama.cpp runtime, settings, and scripts.
- [lemonade-b1294-windows-rocm-gfx1151-x64.zip](https://github.com/akqmffl/llama.cpp-lemonade-Runtime-for-qwen-3.6-MTP/releases/download/b1294/lemonade-b1294-windows-rocm-gfx1151-x64.zip):
  Lemonade runtime replacement/overlay, env presets, config
  snippets, and scripts.

## Layout

```text
README.md
.gitattributes
GitHub Release asset: llama-b1294-windows-rocm-gfx1151-x64.zip
  README.md
  runtime/   Full directly built llama.cpp runtime payload.
  settings/  Qwen3.6 Claude-Opus benchmark/runtime JSON settings.
  scripts/   Convenience launch scripts for llama-server.exe.

GitHub Release asset: lemonade-b1294-windows-rocm-gfx1151-x64.zip
  README.md
  runtime/full-replacement/  Full runtime folder for Lemonade copy/paste.
  runtime/minimal-overlay/   ABI-safe minimal overwrite set for an existing
                              matching ROCm Lemonade runtime.
  env/                       Feature-scoped env groups and combined presets.
  config/                    Lemonade llamacpp args and config snippets.
  scripts/                   Runtime/env/config apply helpers.
```

Download and extract only the ZIP archive you need. After extraction, the paths
shown below refer to the extracted `llama.cpp/` or `lemonade/` folder.

The `llama.cpp/runtime/` and `lemonade/runtime/full-replacement/` folders
preserve the standard Windows ROCm llama.cpp executable layout: server binary,
support DLLs, and the `hipblaslt/` and `rocblas/` runtime library folders stay
together.

The Lemonade `minimal-overlay/` folder intentionally contains only the changed
llama.cpp/ggml ABI set. Use it only when the target Lemonade runtime already has
the matching ROCm vendor DLLs plus `hipblaslt/` and `rocblas/` directories.

## Quick Start

The ZIP files are self-rooted: their contents start with `README.md`,
`runtime\`, `settings\`, and related folders, not with an extra top-level
wrapper directory. Create the destination folder first, then extract the ZIP
into it. Replace every placeholder path such as `C:\path\to\model.gguf` or
`C:\path\to\lemonade` with the actual path on the target PC.

### A. Standalone llama.cpp

Use this path when you want to run `llama-server.exe` directly, without
Lemonade.

From the directory where you downloaded the ZIP:

```cmd
mkdir llama.cpp
tar -xf llama-b1294-windows-rocm-gfx1151-x64.zip -C llama.cpp
cd llama.cpp
scripts\start-production-n2.cmd -ModelPath "C:\path\to\model.gguf" -BindHost 127.0.0.1 -Port 8080
```

This command uses:

```text
Script file:   llama.cpp\scripts\start-production-n2.cmd
Launcher file: llama.cpp\scripts\start-from-settings.ps1
Settings file: llama.cpp\settings\qwen36-claude-opus-production-n2-4096.json
Server file:   llama.cpp\runtime\llama-server.exe
```

The `start-from-settings.ps1` launcher reads the settings JSON, sets the
required process-level environment variables, builds the `llama-server.exe`
argument list, and starts the server. It does not permanently change Windows
user or system environment variables.

`-BindHost 127.0.0.1` and `-Port 8080` are local example defaults; change them
only when the target PC needs a different bind address or port.

To run without the helper scripts, set the same environment variables and call
the server directly:

```cmd
set ROCBLAS_DEFAULT_ATOMICS_MODE=0
set LLAMA_MTP_ENABLE_RS_SCRATCH=1
set LLAMA_MTP_DRAFT_COMPACT_ARGMAX=1
set LLAMA_MTP_GREEDY_COMPACT_VERIFY=1
set LLAMA_MTP_FAST_VERIFY_MAX_PROMPT_TOKENS=0
set LLAMA_MTP_SEPARATE_SCHED=1
set LLAMA_MTP_GDN_DIRECT_SNAPSHOT=1
set GGML_HIP_MTP_VERIFIER_GFX1151_ARGMAX=1
rem 4096 is the conservative default. 8192 or larger is also supported on this gfx1151 runtime after local throughput retesting.
set GGML_HIP_MTP_VERIFIER_GFX1151_ARGMAX_TILE=4096
set LLAMA_HOST=127.0.0.1
set LLAMA_PORT=8080

runtime\llama-server.exe ^
  --host %LLAMA_HOST% ^
  --port %LLAMA_PORT% ^
  --model "C:\path\to\model.gguf" ^
  --ctx-size 131072 ^
  --gpu-layers all ^
  --parallel 1 ^
  --no-mmap ^
  --flash-attn off ^
  --cache-type-k f16 ^
  --cache-type-v f16 ^
  --cache-ram 0 ^
  --spec-type mtp ^
  --spec-draft-n-max 2 ^
  --spec-draft-n-min 1 ^
  --spec-draft-p-min 0.75 ^
  --ubatch-size 512
```

Run the command from the extracted `llama.cpp` folder. The `--ctx-size 131072`
value is the 128K context production setting used for the recorded n=2 anchor;
reduce it if the target system does not have enough memory for that context.

`LLAMA_HOST=127.0.0.1` and `LLAMA_PORT=8080` are example local defaults. Change
`LLAMA_PORT` when another service already uses that port. Keep
`LLAMA_HOST=127.0.0.1` for local-only access; use another bind address only
when you intentionally expose the server outside the local machine.

### B. Lemonade Integration With Helper Scripts

Use this path when you want Lemonade to launch this runtime.

From the directory where you downloaded the ZIP:

```cmd
mkdir lemonade
tar -xf lemonade-b1294-windows-rocm-gfx1151-x64.zip -C lemonade
cd lemonade
```

1. Copy the runtime into the Lemonade llama.cpp runtime folder.

For an existing matching ROCm Lemonade runtime:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\scripts\apply-runtime.ps1 -Mode minimal -TargetRuntimeDir "C:\path\to\lemonade\bin\llamacpp\qwen36-mtp-gfx1151" -Backup
```

For a clean or complete replacement runtime:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\scripts\apply-runtime.ps1 -Mode full -TargetRuntimeDir "C:\path\to\lemonade\bin\llamacpp\qwen36-mtp-gfx1151" -Backup
```

2. Apply the production environment preset.

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\scripts\apply-env.ps1 -Preset .\env\presets\qwen36-production-n2-faoff\env.json -Target User
```

3. Apply the Lemonade `llamacpp` config values.

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\scripts\apply-config.ps1 -ConfigPath "C:\path\to\lemonade\config.json" -RocmBin "C:\path\to\lemonade\bin\llamacpp\qwen36-mtp-gfx1151\llama-server.exe"
```

Restart Lemonade after changing runtime files, environment variables, or
`config.json`.

### C. Manual Lemonade Setup Without Helper Scripts

Use this path if you prefer to edit Lemonade files and Windows environment
variables manually.

#### Runtime Files

Copy one of these folders into the Lemonade llama.cpp runtime folder:

```text
Source folder, minimal overlay:
lemonade\runtime\minimal-overlay\

Source folder, full replacement:
lemonade\runtime\full-replacement\

Target folder:
C:\path\to\lemonade\bin\llamacpp\qwen36-mtp-gfx1151\
```

The `qwen36-mtp-gfx1151` folder name is only a recommended local runtime folder
name. You may use a different folder name, but `rocm_bin` must point to the
matching `llama-server.exe` inside that same folder.

Use `minimal-overlay` only when the target folder already contains compatible
ROCm vendor DLLs plus the `hipblaslt\` and `rocblas\` runtime library folders.
Use `full-replacement` when in doubt.

#### Windows Environment Variables

Reference preset file:

```text
lemonade\env\presets\qwen36-production-n2-faoff\env.json
```

Set these variables in the Windows user environment, then restart Lemonade:

```powershell
[Environment]::SetEnvironmentVariable("ROCBLAS_DEFAULT_ATOMICS_MODE", "0", "User")
[Environment]::SetEnvironmentVariable("LLAMA_MTP_ENABLE_RS_SCRATCH", "1", "User")
[Environment]::SetEnvironmentVariable("LLAMA_MTP_DRAFT_COMPACT_ARGMAX", "1", "User")
[Environment]::SetEnvironmentVariable("LLAMA_MTP_GREEDY_COMPACT_VERIFY", "1", "User")
[Environment]::SetEnvironmentVariable("LLAMA_MTP_FAST_VERIFY_MAX_PROMPT_TOKENS", "0", "User")
[Environment]::SetEnvironmentVariable("LLAMA_MTP_SEPARATE_SCHED", "1", "User")
[Environment]::SetEnvironmentVariable("LLAMA_MTP_GDN_DIRECT_SNAPSHOT", "1", "User")
[Environment]::SetEnvironmentVariable("GGML_HIP_MTP_VERIFIER_GFX1151_ARGMAX", "1", "User")
[Environment]::SetEnvironmentVariable("GGML_HIP_MTP_VERIFIER_GFX1151_ARGMAX_TILE", "4096", "User")
```

`GGML_HIP_MTP_VERIFIER_GFX1151_ARGMAX_TILE=4096` is the conservative default
used by this package. Larger tile values such as `8192` and above are also
supported by this gfx1151 runtime; if you change the value, keep it as an
integer environment variable and retest throughput on the target machine.

#### Lemonade Config

Edit this file in the Lemonade install:

```text
C:\path\to\lemonade\config.json
```

Reference snippet file in this package:

```text
lemonade\config\lemonade-llamacpp-production-n2-snippet.json
```

Set the `llamacpp` object to the following values, replacing `rocm_bin` with the
actual target PC path:

```json
{
  "llamacpp": {
    "backend": "rocm",
    "prefer_system": true,
    "rocm_bin": "C:\\path\\to\\lemonade\\bin\\llamacpp\\qwen36-mtp-gfx1151\\llama-server.exe",
    "args": "--flash-attn off --parallel 1 --spec-type mtp --spec-draft-n-max 2 --spec-draft-n-min 1 --spec-draft-p-min 0.75 --cache-ram 0 --no-mmap --ubatch-size 512 --cache-type-k f16 --cache-type-v f16"
  }
}
```

The `args` value above is also stored in:

```text
lemonade\config\llamacpp-args-production-n2-faoff.txt
```

Configure the Qwen3.6 GGUF model path through Lemonade's normal model
discovery/configuration flow for the target PC. The GGUF model file is not
included in this runtime package.

## Primary Server

```text
llama.cpp\runtime\llama-server.exe
SHA-256: 7E70C443971F26DE6734444D7E459B9EF5136D9F96B480FDD4A3482190699284
```

## Recorded Benchmark Anchors

| Lane | Shape | Result |
| --- | --- | --- |
| n=2 production | 128K ctx, 4096 generated tokens, FA off | 15.131 tok/s, 92.286% acceptance |
| n=2 reasoning off | 128K ctx, 4096 generated tokens, FA off | 15.413 tok/s, 93.182% acceptance |
| n=3 chat stream | 8192 ctx, 100 generated tokens, FA off | 18.050 tok/s, 97.333% acceptance, exact |

Model GGUF files are not included. After extracting
`llama-b1294-windows-rocm-gfx1151-x64.zip`, use the `llama.cpp/scripts`
launchers with `-ModelPath "C:\path\to\model.gguf"`, or configure Lemonade to
point at the model location on the target PC.

## Upload Note

This package contains binary files larger than GitHub's normal 100 MB git
limit. Use Git LFS or upload the ZIP as a GitHub Release asset.

## Custom GPU Kernel Requests and Support

This package is specialized for AMD Ryzen AI Max Series APUs with RDNA 3.5
graphics (`gfx1151`). If you need a custom build or hardware-specific kernels
for another GPU architecture, contact:

```text
wnxor92@gmail.com
```

If this work is useful to you, donations help support continued development,
testing, packaging, and open-source optimization work for local LLM runtimes.

USDT(trx, Tron Network) address:

```text
TU4rcQsgScdvsL74maMzjVBUuKwgq4UrKr
```

![USDT donation QR code](assets/usdt-qr.png)

Before sending any funds, verify the address and the intended USDT network in
your wallet. Blockchain transactions are irreversible.
