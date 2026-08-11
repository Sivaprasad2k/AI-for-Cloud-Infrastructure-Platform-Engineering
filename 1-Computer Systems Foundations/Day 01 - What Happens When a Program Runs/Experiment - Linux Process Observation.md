# Experiment - Linux Process Observation

## Environment

- **Host OS**: Windows 11 (PowerShell)
- **Subsystem**: WSL2 (Windows Subsystem for Linux)
- **Distribution**: Ubuntu 24.04.4 LTS
- **Kernel Version**: Linux 6.18.35.2-microsoft-standard-WSL2
- **Architecture**: x86_64

---

## Commands Executed & Observations

### 1. `ps`

```bash
ps
```

**Observation**: `ps` without options displayed a limited process selection based on the current user/session and terminal context (specifically the interactive `bash` shell and the snapshot `ps` command itself).

### 2. `ps aux`

```bash
ps aux
```

**Observation**: `ps aux` provided a broad system-wide process listing for the WSL environment. Key observed system processes included:
- **PID 1**: `/sbin/init` (the init process/systemd daemon supervising user space).
- Various `systemd` background services.
- `containerd` daemon process.
- `dockerd` (Docker engine daemon process).
- Interactive `bash` shell session.
- User-level `systemd` manager instance.

*Note: Container infrastructure was already running as Linux processes in the WSL environment, serving as a bridge to future container lessons.*

### 3. `ps aux | grep java`

```bash
ps aux | grep java
```

**Observation**: No Java application process was found running in the environment. The output returned only the short-lived `grep java` process itself (assigned PID 773 at execution time).

### 4. `ps -p 773 -o pid,ppid,stat,%cpu,%mem,cmd`

```bash
ps -p 773 -o pid,ppid,stat,%cpu,%mem,cmd
```

**Observation**: Output failed to show process details for PID 773. The `grep` process had already terminated, so PID 773 no longer referred to a live process. The PID may eventually be reused by the kernel.

### 5. `cat/proc/ 773/status`

```bash
cat/proc/ 773/status
```

**Observation**: Returned a command syntax error (`bash: cat/proc/: No such file or directory`). The command contained two syntax flaws: missing space after `cat` and an erroneous space after `/proc/`. Furthermore, even with corrected syntax (`cat /proc/773/status`), PID 773 had already exited and its procfs entry had disappeared.

### 6. `top`

```bash
top
```

**Observation**: Interactive real-time process viewer reported system summary metrics:
- **Tasks**: 25 total tasks (1 running, 24 sleeping, 0 stopped, 0 zombie).
- **CPU State**: Approximately 100% CPU idle at the observation moment.
- **Memory State**: Approximately 5.9 GiB total system memory visible inside WSL2.
- **Swap Memory**: Approximately 2.0 GiB swap memory configured and 0 KiB used.

---

## Summary of Observations

| Observation | Evidence | Concept Demonstrated |
| :--- | :--- | :--- |
| Limited default `ps` selection | `ps` showed only `bash` and `ps` | Process selection rules based on session/terminal context. |
| Background daemons & container engine | `ps aux` listed `/sbin/init`, `dockerd`, `containerd` | OS init process hierarchy, system daemons, container host processes. |
| No active Java runtime | `ps aux \| grep java` returned only `grep` | Process existence vs search command artifacts. |
| Ephemeral process lifecycle | PID 773 was no longer a live process | Short-lived process lifecycle and rapid termination. |
| Syntax error on `/proc` access | `cat/proc/ 773/status` threw command error | Precision requirement for CLI syntax and pseudo-filesystem paths. |
| Idle system state | `top` reported 1 running, 24 sleeping, ~100% idle CPU | Process states (Running vs Sleeping) and CPU scheduling idle time. |
| Memory visibility | `top` reported 5.9 GiB RAM / 2.0 GiB Swap | Resource boundary allocation in WSL2 Linux virtual environment. |

---

## What Went Wrong

1. **Attempting to inspect an ephemeral process**: Attempting to run detailed inspection (`ps -p 773` and `cat /proc/773/status`) on the PID returned by `grep java`. Because `grep` completes in milliseconds, the process terminated and PID 773 no longer referred to a live process before inspection could take place.
2. **Incorrect Command Syntax**: The command `cat/proc/ 773/status` was entered with formatting errors:
   - Missing space between executable `cat` and argument `/proc/`.
   - Extra space between `/proc/` and `773/status`.

Correct syntax structure for inspecting process status via procfs:
```bash
cat /proc/<PID>/status
```

---

## What We Learned From the Mistakes

- **Short-Lived vs Long-Lived Processes**: Short-lived utility commands (`grep`, `ls`, `ps`) exist for milliseconds, making them unsuitable targets for manual step-by-step process state inspection. Stable or long-running processes (daemons, background sleep processes, or application servers) must be used when inspecting `/proc` attributes.
- **Procfs Mechanics**: `/proc` is not a physical directory on disk; it is a virtual pseudo-filesystem synthesized on the fly by the Linux kernel. When a process terminates, its corresponding virtual directory `/proc/<PID>` is instantly removed by the kernel.

---

## Correct Follow-up Experiment

> [!IMPORTANT]
> **Status**: **NOT YET EXECUTED**. The following experiment design is documented for future validation.

To properly inspect a process lifecycle and procfs entries without ephemeral termination, execute a long-running background process:

```bash
# 1. Start a long-running background process (5-minute sleep)
sleep 300 &

# 2. Capture the Process ID (PID) of the last background job
echo $!

# 3. Inspect specific process attributes using ps
ps -p <PID> -o pid,ppid,stat,%cpu,%mem,cmd

# 4. Read process status from the proc pseudo-filesystem
cat /proc/<PID>/status

# 5. List open file descriptors and entries in proc directory
ls -l /proc/<PID>/

# 6. Terminate the background process cleanly
kill <PID>
```

---

## Research Observation

- **Preliminary Observation**: Infrastructure automation cannot safely reason from application-level intent alone. It must account for observable system state, resource constraints, and execution context.
- **Evidence**: Day 1 Linux process observation.
- **Validation Required**: Literature review and future empirical experiments.
- **Status**: **NOT a validated research gap.**
