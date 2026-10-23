# Use REA on Windows and Ubuntu WSL

Run separate Windows and Linux MCP servers when an investigation needs both
Windows applications and Linux-only analysis tools. A second server offers a
different host environment; it does not make every operation portable.
See [installation](installation.md), [Windows Ghidra scope](windows-ghidra-p0.md)
and the individual provider guides for supported targets and limitations.

## Prepare each host

Install a [supported Node.js runtime](installation.md#package-runners-and-global-installation)
and npm on each host. A Windows Node executable is not a Linux Node installation.
In Ubuntu, check `command -v node`, `node --version` and `command -v ps`; Linux
owned-process workflows require a procps-compatible `ps` and appropriate process
ownership controls. Use a full compatible JDK for Ghidra and headless JADX.

REA's setup configures agents and their bundled workflow. It does not install
or upgrade Node, Java, Ghidra, or the optional tools below. Obtain them separately
from their official sources and verify release digests where available. Select
only the providers needed for the investigation:

| Analysis                         | Prerequisite and configuration                                                  | Guide                                                        |
| -------------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| Native binaries                  | Ghidra 12.1.x, full compatible JDK; `GHIDRA_INSTALL_DIR`, `JAVA_HOME`           | [Installation](installation.md#ghidra)                       |
| Android classes                  | Pinned headless JADX MCP 0.7.1 JAR; `REA_JADX_MCP_JAR`                          | [Android](android-analysis.md)                               |
| Android resources                | Apktool launcher; `REA_APKTOOL_COMMAND`                                         | [Resources](apktool-resource-analysis.md)                    |
| Android devices                  | Android platform tools; `REA_ADB_PATH`                                          | [ADB](adb-device-analysis.md)                                |
| WebAssembly                      | WABT 1.0.42 executables; `REA_WABT_BIN_DIRECTORY`                               | [WASM](wasm-artifacts.md)                                    |
| Offline ELF and recorded crashes | Exact pwntools/pyelftools/Unicorn profile; `REA_PWNTOOLS_PYTHON`                | [ELF](binary-diagnostics.md), [crashes](recorded-crashes.md) |
| Firmware on Linux                | Binwalk 3.1.x and/or Unblob 26.6.x; `REA_BINWALK_COMMAND`, `REA_UNBLOB_COMMAND` | [Firmware](firmware-analysis.md)                             |
| Saved packets on Linux           | TShark/capinfos 4.x and `prlimit`; `REA_TSHARK_COMMAND`, `REA_CAPINFOS_COMMAND` | [PCAP](pcap-analysis.md)                                     |
| Native mitmproxy files on Linux  | mitmdump 12.2.3; `REA_MITMDUMP_COMMAND`                                         | [Web captures](web-network-captures.md)                      |
| Recorded logic signals on Linux  | sigrok-cli 0.7.2 and `prlimit`; `REA_SIGROK_COMMAND`                            | [Signals](signal-recordings.md)                              |

Versions above describe the current REA provider profiles, not a promise that
the newest tool release will be accepted. Recheck the linked guide before
upgrading a provider. Commercial engines need independently obtained licenses.

## Configure Windows

Use PowerShell to set paths to the installations you selected, then preview
and apply setup. The following paths are examples; substitute your real paths:

```powershell
$env:GHIDRA_INSTALL_DIR = 'C:\tools\ghidra_12.1.4_PUBLIC'
$env:JAVA_HOME = 'C:\tools\jdk-21'
$env:REA_JADX_MCP_JAR = 'C:\tools\jadx-headless-mcp-0.7.1-all.jar'
npx rea-agents@6.7.0 setup --client codex --dry-run --json
npx rea-agents@6.7.0 setup --client codex
```

Review the plan and approve it. Setup backs up existing configuration and
preserves unrelated entries. For optional provider paths not captured by setup,
add their explicit environment keys to `[mcp_servers.rea.env]` in the Codex
configuration after making a backup. Shell variables in another terminal do
not change an already running MCP server's environment.

## Add the Ubuntu server

Install REA inside Ubuntu, using the selected Linux Node installation:

```sh
npm install --prefix "$HOME/.local/share/rea" rea-agents@6.7.0
```

Create a launcher at `~/.local/bin/rea-linux-mcp`. Use your actual absolute
Linux executable and provider paths. This example assumes Linux Node is
installed at `/usr/local/bin/node` and enables Ghidra; omit optional provider
variables for engines you have not installed:

```bash
#!/usr/bin/env bash
set -euo pipefail
export PATH="/usr/local/bin:/usr/bin:/bin"
export GHIDRA_INSTALL_DIR="$HOME/tools/ghidra_12.1.4_PUBLIC"
export JAVA_HOME="/usr/lib/jvm/java-21-openjdk-amd64"
# Add explicit provider keys from the table above when needed.
exec /usr/local/bin/node \
  "$HOME/.local/share/rea/node_modules/rea-agents/scripts/rea.mjs" "$@"
```

Keep Linux engines and Python environments on the Ubuntu filesystem when
possible. Verify downloaded bytes before copying them. If a valid JAR fails
REA's archive inspection across `/mnt/c` or `/mnt/d`, copy the unchanged file
to Linux storage, verify the copy's digest and recheck its configured path.

From Windows PowerShell, register that launcher under a distinct name. Replace
the distribution name and Linux username with your own values:

```powershell
codex mcp add rea_linux -- wsl.exe -d Ubuntu -- bash /home/YOUR_USER/.local/bin/rea-linux-mcp mcp
```

If cold WSL startup exceeds the client's default deadline, set explicit
timeouts in the existing `[mcp_servers.rea_linux]` table; do not create a second
table with the same name:

```toml
startup_timeout_sec = 90
tool_timeout_sec = 720
```

Choose the operation timeout for your workload. Provider startup deadlines
have separate controls described in the provider guides. Restart Codex after
configuration changes. Use the Windows server for Windows-native paths and
the Linux server for Linux paths: `D:\samples\app` becomes `/mnt/d/samples/app`
inside WSL. Each server has independent sessions and temporary workspaces.

## Verify the actual connections

In the same Windows environment used for setup:

```powershell
npx rea-agents@6.7.0 doctor --client codex --provider ghidra --provider jadx --skill --json
npx rea-agents@6.7.0 mcp doctor --json
wsl.exe -d Ubuntu -- bash /home/YOUR_USER/.local/bin/rea-linux-mcp doctor --provider ghidra --json
wsl.exe -d Ubuntu -- bash /home/YOUR_USER/.local/bin/rea-linux-mcp mcp doctor --json
```

Select only configured providers in `doctor`. The existing `mcp doctor` checks
the production stdio connection and catalog; it does not establish that every
provider or target will work. In a restarted Codex session, call `binary_session`
on each server to inspect the identity of the server the client actually loaded.
Then run a relevant analysis on a local test artifact you are authorized to
inspect. Keep resulting Evidence and diagnostic logs local.

Windows Ghidra retains its documented read-only and local-NTFS target limits.
WSL does not supply macOS APIs, a commercial license, an Android device, or a
browser CDP endpoint. Browser and device investigations require the explicit
target setup described in their guides.
