======================
Logging service part 1
======================

.. contents:: Table of Contents
   :depth: 2
   :local:

Executive Summary
=================

In real-time and embedded systems, synchronous I/O operations—such as calling ``printf()`` directly to serial ports (UART), filesystems, or socket endpoints—introduce unacceptable latency spikes, memory fragmentation, and race conditions (commonly referred to as *heisenbugs*).

To achieve microsecond-level predictability, system architects decouple the **Log Producer** (the latency-critical application thread) from the **Log Consumer** (an asynchronous background daemon).

.. note::
   The primary goal of a logging framework is to offload disk, flash, and network I/O operations away from the main execution thread to an asynchronous background worker while keeping memory allocations lock-free.

The Performance Problem: Synchronous vs. Asynchronous
=====================================================

Synchronous Logging Bottleneck
------------------------------

When an application calls standard output functions like ``printf()`` or ``fprintf()`` directly:

1. **Format Overhead:** The CPU spends cycles parsing string format specifiers (e.g., ``%s``, ``%d``, ``%f``).
2. **System Calls:** Context switching occurs from User Mode to Kernel Mode via I/O syscalls (``write``, ``ioctl``).
3. **Blocking Hardware:** Execution pauses while waiting for physical devices (UART bit shifts, flash erase cycles, or socket TCP ACKs).

.. warning::
   Writing a single 80-character line over a standard 115200 baud UART port blocks the calling thread for approximately **7 milliseconds**. In a system running a 1 kHz control loop, this delay causes massive frame drops and system timing failures.

Asynchronous Logging Architecture
---------------------------------

Modern architectures resolve this by moving raw data into a lock-free ring buffer residing in shared memory or a fast IPC queue.

::

   +-------------------------------------------------------------------+
   |                    Main Service (Producer)                        |
   |                                                                   |
   |   [Hot Path Code]                                                 |
   |          |                                                        |
   |          v                                                        |
   |   +---------------------------------------------------------+     |
   |   | Non-blocking Lock-Free Ring Buffer / Shared Memory      |     |
   |   +---------------------------------------------------------+     |
   +---------------------------------|---------------------------------+
                                     |
                                     | IPC Pipe / Shared Memory / Unix Socket
                                     v
   +-------------------------------------------------------------------+
   |                   Background Daemon (Consumer)                    |
   |                                                                   |
   |   +---------------------------------------------------------+     |
   |   | Dequeue Buffer & Format Serialization                   |     |
   |   +---------------------------------------------------------+     |
   |          |                                                        |
   |          +------------------+-------------------+                 |
   |          |                  |                   |                 |
   |          v                  v                   v                 |
   |   [Disk / Flash]      [Serial / UART]     [Network / Syslog]      |
   +-------------------------------------------------------------------+

Key Design Benefits
-------------------

* **Microsecond Latency:** Thread execution overhead drops from milliseconds to **nanoseconds**.
* **Zero Allocation:** Memory is allocated up front during startup; no ``malloc()`` calls occur on the critical path.
* **Deterministic Execution:** The main task execution duration remains consistent regardless of log verbosity.

Log Level Categorization: INFO vs. DEBUG
========================================

Log levels balance operational observability against processor utilization. Two core levels serve distinct phases of the software lifecycle:

.. list-table:: Comprehensive Log Level Comparison
   :widths: 15 35 25 25
   :header-rows: 1

   * - Log Level
     - Primary Purpose
     - Target Environments
     - Performance Impact
   * - **INFO**
     - Tracks major system lifecycle state changes, critical hardware events, and unrecoverable errors.
     - Production, Field Testing, Release Candidate Builds.
     - **Extremely Low.** Kept under strict bandwidth budgets to ensure zero disturbance in production.
   * - **DEBUG**
     - Fine-grained register dumps, hardware state transitions, and function entry/exit tracing.
     - Development Labs, Board Bring-Up, Automated Test Benches.
     - **High.** Disabled in production to save CPU bandwidth and storage space.

Implementation Patterns
=======================

Pattern 1: Compile-Time Log Stripping (Zero Overhead)
------------------------------------------------------

To eliminate CPU instruction overhead and reduce final binary code size in production, macro definitions completely strip ``DEBUG`` calls at compile time.

.. code-block:: c

   #include <stdio.h>
   #include <stdint.h>

   #define LOG_LEVEL_NONE  0
   #define LOG_LEVEL_INFO  1
   #define LOG_LEVEL_DEBUG 2

   // Configure build level via CFLAGS / CMake (Defaults to INFO)
   #ifndef BUILD_LOG_LEVEL
   #define BUILD_LOG_LEVEL LOG_LEVEL_INFO
   #endif

   // Internal logger bridge function
   void log_write(uint8_t level, const char *fmt, ...);

   // Always compiled into release builds
   #define LOG_INFO(fmt, ...) \
       log_write(LOG_LEVEL_INFO, "[INFO] " fmt "\n", ##__VA_ARGS__)

   // Stripped out completely during compilation if BUILD_LOG_LEVEL < LOG_LEVEL_DEBUG
   #if (BUILD_LOG_LEVEL >= LOG_LEVEL_DEBUG)
       #define LOG_DEBUG(fmt, ...) \
           log_write(LOG_LEVEL_DEBUG, "[DEBUG] " fmt "\n", ##__VA_ARGS__)
   #else
       #define LOG_DEBUG(fmt, ...) ((void)0) // Compiles to zero instructions
   #endif

Pattern 2: Dynamic Runtime Filtering
------------------------------------

When changing log verbosity without recompiling binary files is necessary, maintain an atomic verbosity variable that can be toggled live via operating system signals (e.g., ``SIGUSR1``).

.. code-block:: c

   #include <stdio.h>
   #include <stdatomic.h>
   #include <signal.h>

   typedef enum {
       LOG_LVL_INFO  = 1,
       LOG_LVL_DEBUG = 2
   } log_level_t;

   // Global atomic level variable
   static _Atomic log_level_t g_runtime_log_level = LOG_LVL_INFO;

   // Signal handler to toggle logging verbosity dynamically
   void handle_sigusr1(int sig) {
       (void)sig;
       if (atomic_load(&g_runtime_log_level) == LOG_LVL_INFO) {
           atomic_store(&g_runtime_log_level, LOG_LVL_DEBUG);
           printf("Log level changed to: DEBUG\n");
       } else {
           atomic_store(&g_runtime_log_level, LOG_LVL_INFO);
           printf("Log level changed to: INFO\n");
       }
   }

   #define DYNAMIC_LOG(level, fmt, ...) do { \
       if (atomic_load(&g_runtime_log_level) >= level) { \
           send_to_background_daemon(level, fmt, ##__VA_ARGS__); \
       } \
   } while(0)

Pattern 3: Lock-Free Single-Producer Single-Consumer (SPSC) Ring Buffer
------------------------------------------------------------------------

The most performant logging pattern uses a lock-free circular ring buffer with atomic head and tail pointers.

.. code-block:: c

   #include <stdint.h>
   #include <stdbool.h>
   #include <stdatomic.h>
   #include <string.h>

   #define RING_BUFFER_SIZE 1024 // Must be a power of 2
   #define MAX_LOG_LINE_LEN 128

   typedef struct {
       char message[MAX_LOG_LINE_LEN];
       uint8_t level;
   } log_entry_t;

   typedef struct {
       log_entry_t entries[RING_BUFFER_SIZE];
       _Atomic uint32_t head;
       _Atomic uint32_t tail;
   } spsc_ring_buffer_t;

   static spsc_ring_buffer_t g_log_ring;

   // Called by Producer (Main Service Thread)
   bool spsc_log_enqueue(uint8_t level, const char *msg) {
       uint32_t head = atomic_load_explicit(&g_log_ring.head, memory_order_relaxed);
       uint32_t tail = atomic_load_explicit(&g_log_ring.tail, memory_order_acquire);

       // Check if buffer is full
       if ((head - tail) >= RING_BUFFER_SIZE) {
           return false; // Buffer overflow, drop message or increment error counter
       }

       uint32_t index = head & (RING_BUFFER_SIZE - 1);
       g_log_ring.entries[index].level = level;
       strncpy(g_log_ring.entries[index].message, msg, MAX_LOG_LINE_LEN - 1);
       g_log_ring.entries[index].message[MAX_LOG_LINE_LEN - 1] = '\0';

       atomic_store_explicit(&g_log_ring.head, head + 1, memory_order_release);
       return true;
   }

   // Called by Consumer (Background Daemon Thread)
   bool spsc_log_dequeue(log_entry_t *out_entry) {
       uint32_t head = atomic_load_explicit(&g_log_ring.head, memory_order_acquire);
       uint32_t tail = atomic_load_explicit(&g_log_ring.tail, memory_order_relaxed);

       if (tail == head) {
           return false; // Buffer empty
       }

       uint32_t index = tail & (RING_BUFFER_SIZE - 1);
       *out_entry = g_log_ring.entries[index];

       atomic_store_explicit(&g_log_ring.tail, tail + 1, memory_order_release);
       return true;
   }

Production Deployment Architectures
===================================

1. Linux ``systemd-journald`` / Unix Domain Sockets
----------------------------------------------------
Standard Linux daemons connect to `/dev/log` or a Unix datagram socket. The OS kernel routes datagrams into `journald` buffers, handling disk persistence and log rotation out-of-band.

2. Shared Memory IPC (Shm / POSIX IPC)
---------------------------------------
Commonly used in Linux Board Support Packages (BSPs) and Automotive Platforms (AUTOSAR/Adaptive Linux). Main processes attach to shared RAM memory blocks created via ``shm_open()``, while a dedicated low-priority daemon drains and writes logs to storage.

3. Deferred Binary Tracing (Piggybacking)
-----------------------------------------
For extreme constraints, instead of sending human-readable ASCII strings (e.g., `"Sensor ID 5 failed with code 0x12"`), the producer pushes raw numbers: `[TIMESTAMP, LOG_ID=42, ARG1=5, ARG2=0x12]`. The background daemon parses an off-line string database to reconstruct human-readable logs later.