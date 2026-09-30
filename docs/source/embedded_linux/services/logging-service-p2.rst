======================
Logging service part 2
======================

.. contents:: Mục Lục Chi Tiết
   :depth: 2
   :local:

1. Thiết Kế Logging Service Tối Ưu Hiệu Năng Trên Linux
-------------------------------------------------------

1.1. Tổng Quan & Đặt Vấn Đề
---------------------------

Trong các hệ thống Linux nhúng và thời gian thực (Real-time/Embedded Systems), các thao tác I/O đồng bộ (Synchronous I/O)—chẳng hạn như gọi ``printf()`` trực tiếp ra cổng Serial/UART, ghi file đĩa, hay gửi qua Socket—gây ra mức overhead tài nguyên không thể chấp nhận được[cite: 1]. Tắc nghẽn này tạo ra các hiện tượng[cite: 1]:

* **Sụt giảm hiệu năng nghiêm trọng:** Tốn thời gian chờ CPU giải quyết I/O[cite: 1].
* **Latency Spikes:** Gây trễ (delay) luồng thực thi chính[cite: 1].
* **Heisenbugs:** Lỗi ẩn hiện bất thường do hành vi nghẽn làm thay đổi thời gian thực thi của luồng[cite: 1].

Để đạt được tốc độ thực thi cấp độ Nanosecond, kiến trúc phần mềm tiêu chuẩn cần **tách biệt triệt để** giữa **Log Producer** (ứng dụng chính trên Hot Path) và **Log Consumer** (Background Daemon chạy ẩn)[cite: 1].

1.2. Phân Tích So Sánh: Log Đồng Bộ vs. Log Bất Đồng Bộ
-------------------------------------------------------

1.2.1. Nút Tắc Nghẽn Của Log Đồng Bộ (Synchronous Logging)
-----------------------------------------------------------

Khi ứng dụng chính gọi trực tiếp các hàm xuất dữ liệu như ``printf()`` hoặc ``fprintf()`` [cite: 1]:

* **Phân tích định dạng chuỗi (String Formatting):** CPU mất nhiều chu kỳ (cycles) để phân tích các cú pháp định dạng (ví dụ: ``%s``, ``%d``, ``%f``)[cite: 1].
* **Chuyển ngữ cảnh (Context Switch):** Phát sinh System Call (``write``, ``ioctl``) để chuyển đổi từ User Mode sang Kernel Mode.
* **Nghẽn phần cứng (Blocking I/O):** Luồng chính bị dừng hoàn toàn để chờ các thiết bị phần cứng vật lý xử lý dữ liệu (dịch bit trên UART, chu kỳ xóa/ghi Flash, hoặc chờ ACK từ TCP Network)[cite: 1].

.. warning::
   Một lệnh ``printf()`` ghi chuỗi 80 ký tự ra cổng UART ở tốc độ baud standard ``115200`` sẽ làm block luồng chính khoảng **7 milliseconds** [cite: 1]. Trong các ứng dụng chạy vòng lặp điều khiển 1 kHz (1 ms/vòng), độ trễ này ngay lập tức làm sụp đổ toàn bộ hệ thống.

1.2.2. Mô Hình Log Bất Đồng Bộ (Asynchronous Logging Architecture)
-------------------------------------------------------------------

Kiến trúc logging tối ưu giải quyết vấn đề bằng cách chỉ đẩy dữ liệu thô (raw struct/binary) vào một vùng nhớ đệm vòng tròn không dùng khóa (**Lock-Free Ring Buffer**) nằm trong bộ nhớ chia sẻ (**POSIX Shared Memory**)[cite: 1].

::

   +-------------------------------------------------------------------+
   |                    Main Application (Producer)                    |
   |                                                                   |
   |   [Hot Path Code]                                                 |
   |          | (Ghi Binary Struct / Lock-Free - tốn vài Nanoseconds)  |
   |          v                                                        |
   |   +---------------------------------------------------------+     |
   |   | SPSC Lock-Free Ring Buffer / POSIX Shared Memory        |     |
   |   +---------------------------------------------------------+     |
   +---------------------------------|---------------------------------+
                                     |
                                     | IPC / POSIX Shared Memory (`shm_open`)[cite: 1]
                                     v
   +-------------------------------------------------------------------+
   |                   Background Logging Daemon (Consumer)            |
   |                                                                   |
   |   +---------------------------------------------------------+     |
   |   | Dequeue Buffer -> Deferred Format String -> Write I/O   |     |
   |   +---------------------------------------------------------+     |
   |          |                                                        |
   |          +------------------+-------------------+                 |
   |          v                  v                   v                 |
   |   [Flash / File Disk] [Syslog / Journald]   [Serial / UART]       |[cite: 1]
   +-------------------------------------------------------------------+

1.2.3. Lợi Ích Cốt Lõi
----------------------

* **Zero-Latency:** Thời gian xử lý log ở Hot Path giảm từ vài millisecond xuống **vài nanosecond** [cite: 1].
* **Zero Dynamic Memory Allocation:** Không gọi ``malloc()`` hay ``free()`` trên luồng thực thi chính, loại bỏ nguy cơ phân mảnh RAM và Rác bộ nhớ[cite: 1].
* **Triệt tiêu nghẽn I/O:** Mọi tác vụ ghi đĩa, định dạng chuỗi, xuất Serial đều được đẩy cho Background Daemon có mức ưu tiên CPU thấp hơn đảm nhận[cite: 1].

1.3. Cấu Hình Phân Cấp Log Level (INFO vs. DEBUG)
-------------------------------------------------

Các cấp độ log giúp cân bằng giữa khả năng giám sát hệ thống và mức độ tiêu tốn tài nguyên[cite: 1].

.. list-table:: So Sánh Các Cấp Độ Log Chính
   :widths: 15 35 25 25
   :header-rows: 1

   * - Cấp Độ Log
     - Mục Đích Sử Dụng
     - Môi Trường Áp Dụng
     - Ảnh Hưởng Hiệu Năng
   * - **INFO**
     - Ghi nhận chuyển giao trạng thái quan trọng, sự kiện phần cứng, lỗi nghiêm trọng[cite: 1].
     - Production, Field Testing, Release Candidates[cite: 1].
     - **Cực kỳ thấp.** Được giữ ở mức tối thiểu để chạy an toàn trên bản thương mại[cite: 1].
   * - **DEBUG**
     - Trích xuất thanh ghi, luồng hàm, dữ liệu cảm biến thô phục vụ tìm lỗi[cite: 1].
     - Development Labs, Board Bring-Up, Test Benches[cite: 1].
     - **Cao.** Cần loại bỏ hoàn toàn khỏi bản Release để tiết kiệm CPU và ROM/Flash[cite: 1].

1.4. Chi Tiết Triển Khai Lock-Free Ring Buffer Bằng C
-----------------------------------------------------

Dưới đây là mã nguồn C tiêu chuẩn triển khai mô hình **Single-Producer Single-Consumer (SPSC)** tối ưu tối đa tài nguyên hệ thống[cite: 1].

1.4.1. Khai Báo Cấu Trúc Dữ Liệu (`logger_common.h`)
-----------------------------------------------------

.. code-block:: c

   #ifndef LOGGER_COMMON_H
   #define LOGGER_COMMON_H

   #include <stdint.h>
   #include <stdbool.h>
   #include <stdatomic.h>

   // Kích thước buffer BẮT BUỘC phải là lũy thừa của 2 (Power of 2)[cite: 1]
   #define RING_BUFFER_SIZE 1024 
   #define MAX_LOG_MSG_LEN  128

   typedef enum {
       LOG_LVL_INFO  = 1,
       LOG_LVL_DEBUG = 2
   } log_level_t;

   typedef struct {
       uint64_t timestamp;
       uint8_t level;
       char message[MAX_LOG_MSG_LEN];
   } log_entry_t;

   typedef struct {
       log_entry_t entries[RING_BUFFER_SIZE];
       _Atomic uint32_t head;
       _Atomic uint32_t tail;
   } spsc_ring_buffer_t;

   #endif

1.4.2. Luồng Ghi Log Phía Producer (`app_producer.c`)
------------------------------------------------------

.. code-block:: c

   #include <stdio.h>
   #include <string.h>
   #include <time.h>
   #include "logger_common.h"

   #define BUILD_LOG_LEVEL LOG_LVL_INFO

   static spsc_ring_buffer_t g_log_ring;

   bool log_enqueue(uint8_t level, const char *msg) {
       uint32_t head = atomic_load_explicit(&g_log_ring.head, memory_order_relaxed);
       uint32_t tail = atomic_load_explicit(&g_log_ring.tail, memory_order_acquire);

       // Kiểm tra buffer đầy[cite: 1]
       if ((head - tail) >= RING_BUFFER_SIZE) {
           return false; // Tránh block, chấp nhận drop log nếu tràn đệm[cite: 1]
       }

       // Ánh xạ index bằng phép toán BITWISE AND thay cho phép chia lấy dư (%)[cite: 1]
       uint32_t index = head & (RING_BUFFER_SIZE - 1);
       g_log_ring.entries[index].level = level;
       g_log_ring.entries[index].timestamp = (uint64_t)time(NULL);
       
       strncpy(g_log_ring.entries[index].message, msg, MAX_LOG_MSG_LEN - 1);
       g_log_ring.entries[index].message[MAX_LOG_MSG_LEN - 1] = '\0';

       // Cập nhật head với cờ Release
       atomic_store_explicit(&g_log_ring.head, head + 1, memory_order_release);
       return true;
   }

   #define LOG_INFO(fmt, ...) log_enqueue(LOG_LVL_INFO, fmt)

   // Compile-time stripping: Loại bỏ hoàn toàn DEBUG log ở bản Release[cite: 1]
   #if (BUILD_LOG_LEVEL >= LOG_LVL_DEBUG)
       #define LOG_DEBUG(fmt, ...) log_enqueue(LOG_LVL_DEBUG, fmt)
   #else
       #define LOG_DEBUG(fmt, ...) ((void)0)
   #endif

1.4.3. Luồng Đọc Log Phía Background Daemon (`logging_daemon.c`)
----------------------------------------------------------------

.. code-block:: c

   #include <stdio.h>
   #include <unistd.h>
   #include "logger_common.h"

   extern spsc_ring_buffer_t g_log_ring;

   void run_logging_daemon(void) {
       log_entry_t entry;
       
       while (1) {
           uint32_t head = atomic_load_explicit(&g_log_ring.head, memory_order_acquire);
           uint32_t tail = atomic_load_explicit(&g_log_ring.tail, memory_order_relaxed);

           if (tail == head) {
               usleep(10000); // Sleep 10ms nếu buffer rỗng để tiết kiệm CPU
               continue;
           }

           uint32_t index = tail & (RING_BUFFER_SIZE - 1);
           entry = g_log_ring.entries[index];

           // Giải phóng ô nhớ sau khi lấy dữ liệu
           atomic_store_explicit(&g_log_ring.tail, tail + 1, memory_order_release);

           // Thực hiện I/O nặng ở Background Thread[cite: 1]
           printf("[%LU] [%s] %s\n", 
                  entry.timestamp, 
                  entry.level == LOG_LVL_INFO ? "INFO" : "DEBUG", 
                  entry.message);
       }
   }

1.5. Giải Thích Kỹ Thuật Chuyên Sâu (Deep-Dive Analysis)
--------------------------------------------------------

1.5.1. Bản Chất Của Biến `uint32_t` Trong Ring Buffer
------------------------------------------------------

Mặc dù bộ nhớ đệm chỉ chứa 1024 phần tử, biến ``head`` và ``tail`` bắt buộc phải dùng kiểu số nguyên không dấu 32-bit (``uint32_t``) vì các lý do toán học:

* **Tăng liên tục không đứt đoạn:** ``head`` và ``tail`` đại diện cho **tổng số giao dịch ghi/đọc kể từ khi chạy hệ thống**, chúng tăng tự do $0, 1, 2, \dots$ lên tới $4.294.967.295$.
* **Tận dụng Integer Overflow (Tràn số an toàn):** Khi đạt giá trị tối đa, ``uint32_t`` sẽ tự động xoay vòng về $0$. Phép trừ số nguyên không dấu ``(uint32_t)(head - tail)`` luôn luôn trả về số lượng phần tử thực tế đang chứa trong buffer ngay cả khi ``head`` đã bị tràn số quay về $0$ còn ``tail`` chưa tràn.
* **Phân biệt Buffer Đầy và Rỗng:**
  * **Rỗng:** ``head == tail`` (Số phần tử = $0$)[cite: 1].
  * **Đầy:** ``(head - tail) == 1024`` (Số phần tử = Capacity)[cite: 1].

1.5.2. Kỹ Thuật Bitwise Indexing (`head & (SIZE - 1)`)
------------------------------------------------------

Thay vì dùng phép chia lấy dư tốn nhiều chu kỳ CPU[cite: 1]:

.. code-block:: c

   // KHÔNG TỐI ƯU (Tốn nhiều CPU instruction)
   index = head % 1024; 

Hệ thống tận dụng tính chất của nhị phân với các buffer có kích thước là lũy thừa của 2 ($N = 2^k$)[cite: 1]:

.. code-block:: c

   // TỐI ƯU (Chạy trong 1 chu kỳ CPU)
   index = head & (1024 - 1); // Tương đương: head & 1023

1.5.3. Phân Tích Các Cờ Memory Order (Thứ Tự Sắp Xếp Bộ Nhớ)
------------------------------------------------------------

Các CPU hiện đại và Compiler luôn thực hiện cơ chế tự động **đảo thứ tự lệnh (Out-of-Order Execution)** để tăng tốc. Nếu không dùng các Rào Chắn Bộ Nhớ (Memory Barriers), dữ liệu sẽ bị hỏng do Producer báo hiệu cho Consumer trước khi chuỗi log thực sự được chép xong.

.. list-table:: Chi Tiết Về Các Memory Order Được Sử Dụng
   :widths: 25 25 50
   :header-rows: 1

   * - Cờ Memory Order
     - Vị Trí Sử Dụng
     - Ý Nghĩa Kỹ Thuật
   * - ``memory_order_relaxed``
     - Đọc biến của chính Thread đó (VD: Producer đọc ``head``).
     - Không thiết lập barrier, cho tốc độ truy cập CPU tối đa. Chỉ đảm bảo biến không bị đọc dở chừng.
   * - ``memory_order_acquire``
     - Đọc biến do Thread khác cập nhật (VD: Producer đọc ``tail``).
     - **Rào chắn nhìn về trước:** Đảm bảo mọi lệnh đọc/ghi bên dưới không bao giờ bị CPU nhảy lên trên lệnh này.
   * - ``memory_order_release``
     - Cập nhật chỉ số sau khi hoàn tất I/O (VD: ``atomic_store`` ``head``/``tail``).
     - **Rào chắn nhìn về sau:** Đảm bảo toàn bộ thao tác chép chuỗi/dữ liệu vào mảng BẮT BUỘC phải hoàn thành 100% trước khi phát tín hiệu store[cite: 1].

1.5.4. Thao Tác Ghi `+1` Và Nguyên Lý `atomic_store_explicit`
-------------------------------------------------------------

* **Lý do ghi `+1`:** Không phải là dịch chuyển con trỏ mảng nhảy $1$ ô trực tiếp trên RAM, mà là **tăng biến đếm giao dịch tổng** [cite: 1]. Vị trí ô nhớ thực tế luôn được ánh xạ tự động qua phép toán bitwise ``head & (SIZE - 1)`` [cite: 1].
* **Lý do dùng Atomic Store (`atomic_store_explicit`):**
  1. *Ghi nguyên tử (Atomicity):* Đảm bảo thao tác ghi 32-bit diễn ra trong đúng 1 chu kỳ ghi bus của CPU, ngăn chặn tuyệt đối việc đọc phải giá trị rác.
  2. *Kích hoạt Barrier:* Gắn kèm cờ Release ép CPU ghi xong dữ liệu vào mảng ``entries[index]`` rồi mới được phát tín hiệu tăng biến đếm ``head + 1`` [cite: 1].

1.6. Tích Hợp Native Hệ Thống Linux
------------------------------------

* **Unix Domain Datagram Socket (`/dev/log`):** Sử dụng socket non-blocking kiểu ``SOCK_DGRAM``. Các ứng dụng đẩy log vào socket, Kernel sẽ nhận dữ liệu asynchronously và chuyển cho ``systemd-journald`` xử lý lưu trữ[cite: 1].
* **Systemd Service Unbuffered Redirection:** Cấu hình ứng dụng ghi out đệm (unbuffered stdout: ``setvbuf(stdout, NULL, _IONBF, 0)``)[cite: 1] và khai báo trong file Service Systemd[cite: 1]:

.. code-block:: ini

   [Service]
   ExecStart=/usr/bin/my_app
   StandardOutput=journal
   StandardError=journal


2. Cơ Chế Điều Phối Năng Lượng Trong Linux Kernel Scheduler
-----------------------------------------------------------

2.1. Phân Tích Các Cơ Chế Helper Scheduler
-------------------------------------------

Capacity Aware Scheduling (CAS) và Energy Aware Scheduling (EAS) **không phải** là các thuật toán/bộ lập lịch (scheduler) độc lập riêng biệt[cite: 1, 2]. Chúng chỉ là các **cơ chế hỗ trợ (helper mechanisms / features / extension frameworks)** được tích hợp thẳng vào bộ lập lịch mặc định của Linux Kernel (CFS hoặc EEVDF)[cite: 1, 2].

2.1.1. Capacity Aware Scheduling (CAS)
---------------------------------------

* **Khái niệm:** Cơ chế điều phối tiến trình dựa trên **năng lực xử lý (capacity)** của từng lõi CPU trong hệ thống[cite: 1, 2].
* **Nguyên lý hoạt động:**
  * Báo hiệu cho Scheduler biết mỗi CPU có hiệu năng tối đa là bao nhiêu (ví dụ: CPU lớn có capacity 1024, CPU nhỏ có capacity thấp hơn)[cite: 1, 2].
  * Lựa chọn CPU phù hợp cho tiến trình dựa trên mức độ tải (workload) hiện tại, đảm bảo tiến trình nặng được chạy trên lõi đủ mạnh[cite: 1, 2].
* **Mục tiêu:** Tối ưu hóa **hiệu năng (Performance)** và khả năng xử lý toàn hệ thống[cite: 1, 2].

2.1.2. Energy Aware Scheduling (EAS)
-------------------------------------

* **Khái niệm:** Cơ chế điều phối tích hợp mô hình năng lượng (**Energy Model - EM**) nhằm tối ưu hóa giữa hiệu năng và **mức tiêu thụ điện năng** [cite: 1, 2].
* **Nguyên lý hoạt động:**
  * Thường dùng trên các kiến trúc phần cứng bất đối xứng như **ARM big.LITTLE** hoặc **DynamIQ** [cite: 1, 2].
  * Dự toán mức tiêu thụ năng lượng toàn hệ thống cho từng phương án xếp lịch khi tiến trình thức dậy[cite: 1, 2].
  * Chọn CPU giúp **tiêu tốn ít năng lượng nhất** mà vẫn đáp ứng đủ mức hiệu năng/deadline yêu cầu[cite: 1, 2].
* **Mục tiêu:** Tối ưu hóa **thời lượng pin (Energy Efficiency)** và giảm nhiệt độ thiết bị[cite: 1, 2].

2.2. So Sánh CAS vs. EAS
------------------------

.. list-table:: Bảng So Sánh Chi Tiết CAS Và EAS
   :widths: 20 40 40
   :header-rows: 1

   * - Tiêu Chí
     - Capacity Aware Scheduling (CAS)
     - Energy Aware Scheduling (EAS)
   * - **Trọng tâm chính**
     - Hiệu năng xử lý (Performance)[cite: 1, 2]
     - Tiết kiệm năng lượng (Energy Efficiency)[cite: 1, 2]
   * - **Yếu tố quyết định**
     - Khả năng tính toán tối đa của CPU[cite: 1, 2]
     - Mô hình tiêu thụ điện (Energy Model)[cite: 1, 2]
   * - **Thiết bị phổ biến**
     - Máy chủ, PC, hệ thống đa lõi hỗn hợp[cite: 1, 2]
     - Điện thoại thông minh, thiết bị chạy pin, Linux Embedded[cite: 1, 2]

2.3. Vị Trí Vận Hành Trong Linux Kernel
----------------------------------------

* **Bản chất Core Scheduler:** Linux chỉ có một core scheduler chính quản lý tiến trình và hoán đổi vị trí thực thi[cite: 1, 2].
* **Giai đoạn Can thiệp:** CAS và EAS hoạt động chủ yếu ở giai đoạn **Task Placement (Tìm CPU phù hợp nhất để đặt tiến trình vào chạy)**, cụ thể là trong hàm ``select_task_rq()`` [cite: 1, 2].
* **Luồng xử lý:** Khi một tiến trình thức dậy, CAS/EAS helper code can thiệp để tính toán và trả về **ID của CPU tốt nhất** [cite: 1, 2]. Việc quản lý hàng đợi (runqueue) hay timeslice sau đó vẫn do bộ lập lịch chính (CFS/EEVDF/RT) đảm nhiệm[cite: 1, 2].


3. Khung Phân Tích Chuyên Sâu Linux Kernel Module (LKM)
-------------------------------------------------------

Để phân tích và hiểu một Linux Kernel Module (LKM) một cách toàn diện (tương tự thuật ngữ "cấu hình dẫn động dư" trong robot học), các kỹ sư hệ thống phân loại qua 6 khía cạnh cốt lõi[cite: 1, 2].

3.1. Các Khía Cạnh Cốt Lõi Của Một Kernel Module
-------------------------------------------------

3.1.1. Phân Loại Module (Module Type / Role)
---------------------------------------------

* **Character Device Driver (Char Driver):** Quản lý truyền nhận dữ liệu chuỗi byte nối tiếp (UART, I2C, SPI, cảm biến)[cite: 1, 2].
* **Block Device Driver:** Quản lý thiết bị lưu trữ theo từng khối cố định (SSD, NVMe, eMMC, SD Card)[cite: 1, 2].
* **Network Device Driver:** Quản lý giao diện mạng (Ethernet, Wi-Fi, CAN bus) gửi/nhận dữ liệu dạng gói (packets)[cite: 1, 2].
* **System / Feature Module:** Mở rộng tính năng Kernel (mô-đun tường lửa Netfilter, thuật toán mã hóa, file system)[cite: 1, 2].

3.1.2. Vòng Đời Module (Lifecycle & Entry Points)
--------------------------------------------------

* **`module_init()` (Initialization):** Hàm khởi tạo (cấp phát tài nguyên, đăng ký phần cứng, cấp phát bộ nhớ)[cite: 1, 2].
* **`module_exit()` (Cleanup):** Hàm giải phóng tài nguyên khi dùng lệnh ``rmmod`` để tránh rò rỉ bộ nhớ[cite: 1, 2].
* **Module Metadata:** Khai báo thông tin như ``MODULE_LICENSE("GPL")``, ``MODULE_AUTHOR()``, ``MODULE_VERSION()`` [cite: 1, 2].

3.1.3. Giao Diện Giao Tiếp Với User Space (User Space Interfaces)
------------------------------------------------------------------

* **System Call File Operations (`struct file_operations`):** Các hàm callback như ``open()``, ``read()``, ``write()``, ``close()`` [cite: 1, 2].
* **`ioctl` (Input/Output Control):** Gửi các lệnh điều khiển đặc thù hoặc dữ liệu cấu hình phức tạp[cite: 1, 2].
* **sysfs (`/sys/`):** Cung cấp giao diện dạng cây thư mục để đọc/ghi các thuộc tính (attributes) thiết bị[cite: 1, 2].
* **procfs (`/proc/`):** Xuất thông tin trạng thái hoặc thống kê của module[cite: 1, 2].
* **debugfs:** Dùng riêng cho việc kiểm thử và debug[cite: 1, 2].

3.1.4. Cơ Chế Tương Tác Với Phần Cứng (Hardware Abstraction & Interruption)
----------------------------------------------------------------------------

* **MMIO / I/O Port Mapping:** Ánh xạ các thanh ghi phần cứng vào không gian địa chỉ Kernel (dùng ``ioremap()``)[cite: 1, 2].
* **Interrupt Handling (Xử lý ngắt):**
  * *Top Half (Hard IRQ):* Trình xử lý ngắt khẩn cấp, phản hồi ngay lập tức[cite: 1, 2].
  * *Bottom Half:* Hoãn các tác vụ tốn thời gian sang giai đoạn sau thông qua **Tasklets**, **Workqueues**, hoặc **SoftIRQs** [cite: 1, 2].
* **DMA (Direct Memory Access):** Truyền dữ liệu tốc độ cao trực tiếp giữa thiết bị và RAM mà không thông qua CPU[cite: 1, 2].

3.1.5. Quản Lý Bộ Nhớ & Đồng Bộ Hóa (Memory & Concurrency)
-----------------------------------------------------------

* **Cấp phát bộ nhớ:** Dùng ``kmalloc()`` (bộ nhớ liên tục vật lý) hoặc ``vmalloc()`` (bộ nhớ liên tục ảo)[cite: 1, 2].
* **Đồng bộ hóa (Synchronization Locks):** Sử dụng ``spinlock`` (cho ngữ cảnh ngắt/xử lý nhanh), ``mutex``, ``semaphore`` hoặc ``atomic operations`` để tránh Race Condition[cite: 1, 2].

3.1.6. Mô Hình Thiết Bị (Linux Device Model)
---------------------------------------------

* **Platform Driver & Platform Device:** Mô hình tách biệt giữa mã nguồn điều khiển (Driver) và thông tin cấu hình phần cứng (Device)[cite: 1, 2].
* **Device Tree (DTS / DTB):** Cấu trúc cây mô tả phần cứng được load lúc boot, giúp driver linh hoạt mà không cần hard-code địa chỉ vào file source code[cite: 1, 2].

3.2. So Sánh Góc Nhìn: Robot Học vs. Linux Kernel Module
--------------------------------------------------------

.. list-table:: So Sánh Tư Duy Thiết Kế Hệ Thống
   :widths: 20 40 40
   :header-rows: 1

   * - Khía Cạnh
     - Robot Học (VD: Cấu Hình Dẫn Động Dư)
     - Linux Kernel Module
   * - **Bản chất**
     - Mô tả đặc tính động học / phần cơ khí[cite: 1, 2]
     - Mô tả cách phần mềm giao tiếp phần cứng & hệ điều hành[cite: 1, 2]
   * - **Thành phần chính**
     - Số bậc tự do (DOF), khớp, động cơ, cảm biến[cite: 1, 2]
     - File operations, Interrupt handler, Memory allocation[cite: 1, 2]
   * - **Giao tiếp**
     - Động học thuận/ngược (Forward/Inverse Kinematics)[cite: 1, 2]
     - System calls (read, write, ioctl), sysfs[cite: 1, 2]
   * - **Mục tiêu**
     - Tối ưu hóa quỹ đạo, khả năng chịu lỗi, linh hoạt[cite: 1, 2]
     - Quản lý phần cứng an toàn, hiệu năng cao, không crash[cite: 1, 2]