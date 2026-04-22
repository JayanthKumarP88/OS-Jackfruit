# OS-Jackfruit: Lightweight Container Engine

### 1. Team Information
* **Jayanth Kumar P**: PES1UG24CS199
* **Rishit K**: PES1UG24CS207

---

### 2. Build, Load, and Run Instructions

**Step 1: Build the Project**
Compile the user-space engine, the kernel module, and the test workloads:
make

**Step 2: Load Kernel Module**
Insert the memory monitor module and verify the character device:
sudo insmod monitor.ko
ls -l /dev/container_monitor

**Step 3: Start Supervisor**
Launch the parent supervisor process in a dedicated terminal:
sudo ./engine supervisor ./rootfs-base

**Step 4: Launch Containers**
Create writable rootfs copies and start containers with memory limits:
cp -a ./rootfs-base ./rootfs-alpha
sudo ./engine start alpha ./rootfs-alpha "sh -c 'while true; do true; done'" --soft-mib 48 --hard-mib 80

**Step 5: Cleanup**
Stop the engine and unload the module to return the system to a clean state:
sudo pkill -9 engine
sudo rmmod monitor
make clean

---

### 3. Demo with Screenshots
*Click the filenames below to view the proof of execution.*

| File | Caption |
| :--- | :--- |
| [ss1.png](./screenshots/ss1.png) | Multi-container supervision: Two containers running under one supervisor process. |
| [ss2.png](./screenshots/ss2.png) | Metadata tracking: Output of ./engine ps showing PIDs, status, and resource limits. |
| [ss3.png](./screenshots/ss3.png) | Bounded-buffer logging: Evidence of log capture from container stdout to local files. |
| [ss4.png](./screenshots/ss4.png) | CLI and IPC: Supervisor responding to a stop command issued via Unix Domain Sockets. |
| [ss5.png](./screenshots/ss5.png) | Soft-limit warning: dmesg output showing a SOFT LIMIT warning triggered by RSS usage. |
| [ss6.png](./screenshots/ss6.png) | Hard-limit enforcement: dmesg showing HARD LIMIT breach and the resulting SIGKILL. |
| [ss7.png](./screenshots/ss7.png) | Scheduling experiment: top output showing the NI 0 process dominating the NI 19 process. |
| [ss8.png](./screenshots/ss8.png) | Clean teardown: ps aux output showing no lingering engine or container processes. |

---

### 4. Engineering Analysis

#### I. Isolation Mechanisms
Our runtime achieves isolation by leveraging Linux Namespaces. 
* PID Namespace: Virtualizes process IDs so a containerized process cannot see host processes.
* Mount Namespace & chroot: Restricts the container to its own directory, preventing access to host files. 
* UTS Namespace: Allows each container to have its own hostname.
Note: The host kernel is still shared. All containers use the same syscall interface and kernel data structures.

#### II. Supervisor and Process Lifecycle
A long-running parent supervisor is critical for reliability. It acts as the reaper (handling SIGCHLD) to prevent zombie processes. It maintains the source of truth for metadata and provides a persistent IPC endpoint for CLI communication.

#### III. IPC, Threads, and Synchronization
* Unix Domain Sockets: Used for CLI-to-Supervisor communication. We used Mutexes to ensure that only one CLI request updates metadata at a time.
* Bounded-Buffer Logging: Implemented a producer-consumer model for container output. We used Condition Variables to synchronize the logging threads and prevent race conditions.

#### IV. Memory Management and Enforcement
The kernel module monitors RSS (Resident Set Size), the actual physical RAM occupied by a process. 
* Limits: Soft limits act as a warning threshold; hard limits are absolute ceilings. 
* Kernel Enforcement: Enforcement must happen in kernel space because user-space processes can ignore signals. The LKM ensures the OS can forcibly reclaim resources via SIGKILL.

#### V. Scheduling Behavior
The experiment demonstrated the Completely Fair Scheduler (CFS). By altering Nice values, we manipulated the weight assigned to each process's time-slice. The scheduler prioritizes throughput for the higher-priority (lower nice) process.

---

### 5. Design Decisions and Tradeoffs

1. Unix Domain Sockets for IPC
   * Tradeoff: More complex than signals but allows structured data transfer.
   * Justification: Necessary for commands like ps to return metadata.

2. LKM-based Monitoring
   * Tradeoff: Risk of system-wide crash if bugs exist in the module.
   * Justification: Only way to get real-time RSS data without polling overhead.

3. Static Metadata Table
   * Tradeoff: Limits the absolute number of containers to a fixed size.
   * Justification: Avoids risky dynamic memory allocation in the supervisor's critical path.

---

### 6. Scheduler Experiment Results

When two CPU-intensive shell loops were pinned to a single core using taskset:

| Container PID | Nice Value | Observed %CPU Usage |
| :--- | :--- | :--- |
| 18527 | 0 (Normal) | 98.5% |
| 18533 | 19 (Nice) | 1.5% |

**Conclusion:** The Linux scheduler effectively prioritizes the normal process when resources are scarce. This validates that our engine successfully interfaces with OS scheduling goals of priority-based throughput.
