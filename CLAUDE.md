AGENTS.md

<!-- BEGIN managed:host-paths v1 — do not edit between markers; identical across Gokhan's repos -->
## Host-specific paths (multi-machine)

This repo is worked on from two of Gokhan's machines. The repo lives at a
different absolute path on each. **Detect the host at runtime and resolve roots
from the matching row — never hard-code one machine's drive letters.** In
PowerShell use `$env:COMPUTERNAME`; in bash / git-bash use `hostname`.

| Machine  | Hostname       | GitHub root              | Google Drive    |
| -------- | -------------- | ------------------------ | --------------- |
| Notebook | `GV20260527`   | `C:\Users\gokha\GitHub`  | `G:\My Drive`   |
| Desktop  | `GVAROL202411` | `E:\GitHub`              | `J:\My Drive`   |

If `$env:COMPUTERNAME` matches neither hostname, stop and ask rather than guessing.
<!-- END managed:host-paths v1 -->
