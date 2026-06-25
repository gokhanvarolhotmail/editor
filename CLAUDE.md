AGENTS.md

<!-- BEGIN managed:host-paths v2 — do not edit between markers; identical across Gokhan's repos -->
## Host-specific paths (multi-machine)

This repo is worked on from three of Gokhan's machines. The repo lives at a
different absolute path on each. **Detect the host at runtime and resolve roots
from the matching row — never hard-code one machine's drive letters.** In
PowerShell use `$env:COMPUTERNAME`; in bash / git-bash use `hostname`.

| Machine  | Hostname          | GitHub root              | Google Drive    |
| -------- | ----------------- | ------------------------ | --------------- |
| Notebook | `GV20260527`      | `C:\Users\gokha\GitHub`  | `G:\My Drive`   |
| Desktop  | `GVAROL202411`    | `E:\GitHub`              | `J:\My Drive`   |
| Desktop2 | `DESKTOP-HF2AV4G` | `C:\Users\verek\Github`  | — (none)        |

`C:\Temp`, `C:\Tim`, and `C:\Data` are the same path on all machines.

If `$env:COMPUTERNAME` matches none of these hostnames, stop and ask rather than guessing.
<!-- END managed:host-paths v2 -->
