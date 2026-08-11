# Experiment - Linux Process Model

## Environment

- **Host OS**: Windows 11 (PowerShell)
- **Subsystem**: WSL2 (Windows Subsystem for Linux)
- **Distribution**: Ubuntu 24.04.4 LTS
- **Kernel Version**: Linux 6.18.35.2-microsoft-standard-WSL2
- **Architecture**: x86_64

> [!NOTE]
> **Environment Context**: WSL2 operates as a virtualized Linux utility VM hosted on Hyper-V. Process creation, scheduling, and procfs interfaces follow standard Linux kernel semantics, but host resource constraints are governed by the WSL2 virtual machine boundary.

---

## Experiment 1: Fork Demo

### Purpose
To observe `fork()` process creation, return values in parent and child contexts, and PID/PPID relationships.

### Source Code (`fork-demo.c`)

```c
#include <stdio.h>
#include <unistd.h>

int main()
{
    pid_t pid = fork();

    if (pid < 0)
    {
        perror("fork");
        return 1;
    }

    if (pid == 0)
    {
        printf("Child: PID=%d, PPID=%d\n", getpid(), getppid());
    }
    else
    {
        printf("Parent: PID=%d, Child PID=%d\n", getpid(), pid);
    }

    return 0;
}
```

### Commands Executed

```bash
nano fork-demo.c
gcc fork-demo.c -o fork-demo
ls -l fork-demo
./fork-demo
```

### Actual Execution Output

```text
Parent: PID=1511, Child PID=1512
Child: PID=1512, PPID=1511
```

### Analysis of Results
1. **Parent Process (PID 1511)**: `fork()` returned **1512**, which evaluated to the `else` branch, printing its own PID (1511) and the returned child PID (1512).
2. **Child Process (PID 1512)**: `fork()` returned **0**, evaluating to the `pid == 0` branch, printing its own PID (1512) and querying its parent PID (`getppid()`), which returned **1511**.
3. **Conclusion**: The parent-child relationship (`Parent PID 1511 -> Child PID 1512`) was experimentally verified.

---

## Experiment 2: File Name & Command Mistakes

### Attempt 1: Malformed Filename Argument

```bash
nano fork -demo.c
```

- **Result**: `Invalid operating directory: .c`
- **Root Cause**: The unescaped space split the argument into `fork` and `-demo.c`, which `nano` interpreted as directory options.

### Attempt 2: Incorrect File Extension

```bash
nano fork -demo.java
```

- **Result**: `Invalid operating directory: .java`
- **Root Cause**: Retained the space artifact and specified `.java` instead of C source extension `.c`.

### Corrected Source Creation Command

```bash
nano fork-demo.c
```

- **Result**: File created successfully.

### Attempt 3: Incorrect Executable Invocation

```bash
./fork.demo
```

- **Result**: `-bash: ./fork.demo: No such file or directory`
- **Root Cause**: The compiled output binary was named `fork-demo` (hyphen), whereas the invocation used `fork.demo` (dot).

### Corrected Execution Command

```bash
./fork-demo
```

- **Result**: Execution succeeded.

---

## Experiment 3: Failed Background Process Attempt

> [!WARNING]
> **Classification**: **FAILED ATTEMPT**

### Command Entered

```bash
stable 300 &
```

### Output

```text
[2] 1685
Command 'stable' not found
[2]- Exit 127 stable 300
```

*(Note: An earlier attempt also generated PID 1560 with the same malformed `stable` command).*

### Analysis
- `stable` was an accidental typo for `sleep`.
- The shell assigned job number `[2]` and PID `1685` to the subshell command line before attempting binary lookup.
- Because `stable` did not exist in `$PATH`, the process terminated immediately with exit code `127` (Command Not Found).
- **Lesson**: Shell job assignment and PID emission occur before binary execution validation. A assigned PID from a failed background command must not be mistaken for a running process.

---

## Experiment 4: Successful Background Process Creation

### Commands Executed

```bash
sleep 300 &
echo $!
```

### Actual Output

```text
[2] 1703
1703
```

### Interpretation
- `sleep 300 &` successfully launched the `sleep` command in the background.
- The shell assigned background job ID `[2]` and PID `1703`.
- `$!` expanded to `1703`, confirming the process ID of the background process.

### Experiment Status
- **COMPLETED**: Background process creation (`sleep 300 &`) and PID retrieval (`$!`).
- **NOT CONFIRMED**: Process attribute inspection (`ps -p 1703`), procfs inspection (`/proc/1703/status`), and manual process termination (`kill 1703`) were not executed in this trial.

---

## Experiment 5: Invalid PID Inspection

### Commands Executed

```bash
ps -p 1200 -o pid,ppid,state,%cpu,%mem,cmd
```

### Output

```text
PID    PPID S %CPU %MEM CMD
```

*(No process row returned).*

```bash
cat /proc/1200/status
```

### Output

```text
cat: /proc/1200/status: No such file or directory
```

### Analysis
- PID `1200` was a hardcoded example PID from reference material, not an active process on the current host.
- Because PID `1200` did not exist in the Linux kernel process table, `ps` returned an empty list and `/proc/1200/` was non-existent.
- **Lesson**: Process diagnostics must dynamically use actual PIDs returned by system commands (`$!`, `ps`, or `pgrep`) rather than static tutorial numbers.

---

## Experiment 6: Process Hierarchy Observation via `pstree`

### Command Executed

```bash
pstree -p
```

### Actual Observed Output (Relevant Subtree)

```text
systemd(1)─┬─agetty(215)
           ├─containerd(221)─┬─{containerd}(236)
           │                 ├─{containerd}(237)
           │                 ├─{containerd}(238)
           │                 ├─{containerd}(239)
           │                 ├─{containerd}(240)
           │                 ├─{containerd}(253)
           │                 ├─{containerd}(259)
           │                 └─{containerd}(275)
           ├─cron(166)
           ├─dbus-daemon(167)
           ├─dockerd(311)─┬─{dockerd}(314)
           │              ├─{dockerd}(315)
           │              ├─{dockerd}(316)
           │              ├─{dockerd}(317)
           │              ├─{dockerd}(318)
           │              ├─{dockerd}(319)
           │              ├─{dockerd}(323)
           │              ├─{dockerd}(324)
           │              └─{dockerd}(338)
           ├─init-systemd(Ub(2)─┬─SessionLeader(561)───Relay(563)(562)───bash(563)─┬─pstree(1699)
           │                    │                                                  └─top(889)
```

### Empirical Observations
1. **Root Process**: `systemd` runs as PID 1 at the root of the user-space process tree.
2. **Container Infrastructure**: `dockerd` (PID 311) and `containerd` (PID 221) run as system service processes.
3. **Interactive Shell Hierarchy**: The active shell `bash` (PID 563) spawned `pstree` (PID 1699) and `top` (PID 889) as child processes.
4. **Thread Representation**: `pstree` displays worker threads in curly braces (e.g., `{dockerd}(314)`). Thread scheduling will be evaluated in Day 3.

---

## Experiment 7: System Call Tracing Preparation (`strace`)

### Initial Invocation Attempt

```bash
strace -f -e trace=process ./fork-demo
```

- **Output**: `Command 'strace' not found`

### Package Installation Command

```bash
sudo apt install strace
```

- **Result**: Installation completed successfully.

### Status
- **`strace` Installation**: COMPLETED
- **System Call Tracing Execution**: NOT PERFORMED (tracing `./fork-demo` via `strace` remains a pending future experiment).

---

## Experiment Summary Table

| # | Experiment | Result | Concept Demonstrated | Status |
| :--- | :--- | :--- | :--- | :--- |
| **1** | `fork-demo` compilation/execution | Successful | `fork()`, PID 1511/1512, PPID return | Completed |
| **2** | `nano` filename CLI mistakes | Syntax Errors | CLI argument parsing & filename boundaries | Completed (Observation) |
| **3** | `./fork.demo` execution attempt | Command Failed | Executable binary naming precision | Completed (Observation) |
| **4** | `stable 300 &` execution attempt | Exit 127 | Shell job assignment vs binary resolution | Failed Attempt |
| **5** | `sleep 300 &` execution | Successful | Background job creation & PID 1703 retrieval | Completed |
| **6** | Static PID 1200 inspection | No Process | PID validity & procfs existence rules | Completed (Observation) |
| **7** | `pstree -p` tree inspection | Successful | OS process hierarchy & parent-child trees | Completed |
| **8** | `strace` tool installation | Installed | Diagnostic tooling preparation | Completed |
| **9** | `strace` process tracing | Pending | Low-level syscall interception | Pending (Future) |

---

## Mistakes and What They Taught

| Mistake | Root Cause | Lesson Learned |
| :--- | :--- | :--- |
| `nano fork -demo.c` | Unescaped space in argument. | Spaces separate CLI arguments; filenames containing hyphens must not contain spaces. |
| `nano fork -demo.java` | Malformed argument & wrong extension. | Source code files require exact extensions (`.c`) for compiler resolution. |
| `./fork.demo` | Typo in executable filename. | Executable filenames must match the exact output specified during compilation (`-o fork-demo`). |
| `stable 300 &` | Typo (`stable` instead of `sleep`). | The shell assigns job numbers and PIDs before validating binary existence; exit 127 indicates `Command Not Found`. |
| Inspecting PID 1200 | Using static tutorial PIDs. | Tutorial PIDs are arbitrary; diagnostic commands must target active PIDs returned by `$!` or `ps`. |
| Uninstalled `strace` | Tool not present in base image. | System diagnostic utility availability must be verified prior to test execution. |

---

## Observation vs. Interpretation

| Observed Fact | Empirical Evidence | Technical Interpretation |
| :--- | :--- | :--- |
| Parent output: `PID=1511, Child PID=1512` | `fork-demo` stdout | `fork()` returned the newly allocated child PID (1512) to the parent process context. |
| Child output: `PID=1512, PPID=1511` | `fork-demo` stdout | `fork()` returned 0 to the child process context; child queried PPID 1511. |
| `pstree` output showed `bash(563) ─── pstree(1699)` | `pstree -p` output | Interactive shell `bash` instantiated `pstree` as a child process via `fork()` + `exec()`. |
| `echo $!` returned `1703` after `sleep 300 &` | Shell output | `$!` evaluates to the PID of the most recently spawned background process (1703). |

---

## Research Discipline

- **Preliminary Engineering Observation**: Infrastructure-level reasoning requires awareness of actual process state, resource state, execution context, and process relationships rather than relying only on application-level intent.
- **Classification**: Preliminary observation
- **Evidence Base**: Day 1 and Day 2 Linux process observations in WSL2
- **Validation Required**: Formal literature review and future controlled experiments
- **Status**: **NOT a validated research gap.**
