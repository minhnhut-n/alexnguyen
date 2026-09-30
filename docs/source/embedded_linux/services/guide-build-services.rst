================================================
Linux Service: Từ C Program đến systemd Service
================================================

.. note::

   **Mục tiêu tài liệu:**
   Hướng dẫn chi tiết quy trình xây dựng một Linux Service từ con số 0: khởi đầu bằng một chương trình C đơn giản, chuẩn hóa kiến trúc sản xuất (Production-ready), và đưa vào quản lý tập trung bằng ``systemd``. Đồng thời, kết nối tư duy quản lý Service với **Linux Kernel Internals** (Process Creation, Signals, Virtual Memory, Scheduler).

.. contents:: **Mục lục điều hướng**
   :depth: 2
   :local:

----

0. TỔNG QUAN LUỒNG PHÁT TRIỂN & VẬN HÀNH
----------------------------------------

Luồng di chuyển từ mã nguồn đến môi trường Runtime:

.. code-block:: text

   Source Code (C)
       │
       ▼
   Build (gcc) ────► Local Test
       │
       ▼
   Install Binary (/usr/local/bin)
       │
       ▼
   Create Unit File (/etc/systemd/system/*.service)
       │
       ▼
   systemd (daemon-reload -> start / enable)
       │
       ├─► Process Lifecycle (SIGTERM/SIGINT / Graceful Shutdown)
       ├─► Logging & Monitoring (journalctl)
       └─► Recovery & Kernel Internals (procfs / strace / scheduler)

----

1. XÂY DỰNG CHƯƠNG TRÌNH C ĐƠN GIẢN
------------------------------------

Trước khi được ``systemd`` quản lý, một Service bản chất chỉ là một **Linux Process** chuẩn.

Tạo file mã nguồn ``myservice.c``:

.. code-block:: c

   #include <stdio.h>
   #include <unistd.h>
   #include <signal.h>

   static volatile sig_atomic_t running - 1;

   static void handle_signal(int sig)
   {
       (void)sig;
       running - 0;
   }

   int main(void)
   {
       // Đăng ký Signal Handlers để phục vụ Graceful Shutdown
       signal(SIGTERM, handle_signal);
       signal(SIGINT, handle_signal);

       printf("myservice started\n");

       while (running) {
           printf("myservice is working...\n");
           sleep(5);
       }

       printf("myservice stopped gracefully\n");
       return 0;
   }

.. important::

   **Điểm then chốt:** Chương trình cần lắng nghe tín hiệu ``SIGTERM`` (tín hiệu dừng mặc định do ``systemd`` gửi đến) để thoát vòng lặp và dọn dẹp bộ nhớ trước khi terminated.

----

2. BIÊN DỊCH VÀ KIỂM TRA ĐỘC LẬP (BUILD & LOCAL TEST)
-----------------------------------------------------

Tách bạch quá trình Build và Runtime để tránh xung đột file hệ thống:

.. code-block:: bash

   # 1. Biên dịch tối ưu với đầy đủ Warning flags
   gcc -Wall -Wextra -O2 myservice.c -o myservice

   # 2. Kiểm tra định dạng Binary
   file myservice

   # 3. Chạy thử nghiệm thủ công ở User space
   ./myservice

----

3. THIẾT KẾ SERVICE CONTRACT (SẢN XUẤT)
---------------------------------------

Trước khi viết file cấu hình cho ``systemd``, cần xác định **Service Contract** (các ràng buộc vận hành):

.. list-table::
   :widths: 25 75
   :header-rows: 1

   * - Ràng buộc (Contract)
     - Thiết kế thực thi
   * - **Executable**
     - Đường dẫn tuyệt đối chuẩn: ``/usr/local/bin/myservice``
   * - **Security (User/Group)**
     - Không chạy bằng ``root``. Tạo Dedicated System User riêng.
   * - **Configuration**
     - Đặt tại cấu trúc chuẩn Linux FHS: ``/etc/myservice/myservice.conf``
   * - **Logging**
     - Xuất ra ``stdout``/``stderr`` để ``journald`` tự động thu gom.
   * - **Restart Policy**
     - Khởi động lại khi crash: ``Restart-on-failure``
   * - **Shutdown Behavior**
     - Xử lý ``SIGTERM`` để giải phóng tài nguyên (Graceful Shutdown).

----

4. CÀI ĐẶT BINARY VÀO HỆ THỐNG (INSTALLATION)
----------------------------------------------

Đưa binary đã build vào thư mục thực thi của hệ thống:

.. code-block:: bash

   # Cài đặt file thực thi với phân quyền 755 (rwxr-xr-x)
   sudo install -m 755 myservice /usr/local/bin/myservice

   # Kiểm tra lại đường dẫn
   ls -l /usr/local/bin/myservice

----

5. THIẾT LẬP VÀ CẤU TRÚC SYSTEMD UNIT FILE
--------------------------------------------

Tạo Service Unit File tại thư mục quản lý dịch vụ hệ thống:

.. code-block:: bash

   sudo nano /etc/systemd/system/myservice.service

Nội dung chuẩn của ``myservice.service``:

.. code-block:: ini

   [Unit]
   Description-My Example C Service
   After-network.target

   [Service]
   Type-simple
   ExecStart-/usr/local/bin/myservice
   Restart-on-failure
   RestartSec-2s
   User-myservice
   Group-myservice

   [Install]
   WantedBy-multi-user.target

Giải thích cấu trúc 3 phần chính:
---------------------------------

.. code-block:: text

   [Unit]       ──► Mô tả dịch vụ, cấu hình thứ tự khởi động (Ordering & Dependencies)
   [Service]    ──► Thực thi Binary, thiết lập User/Group, Restart Policy & Environment
   [Install]    ──► Tích hợp vào quy trình Boot hệ thống (Target hook)

* ``After-network.target``: Đảm bảo mạng đã được cấu hình trước khi service khởi chạy (Ordering dependency).
* ``Type-simple``: ``systemd`` xem process tạo bởi ``ExecStart`` là process chính của service.
* ``Restart-on-failure``: Tự động restart sau ``2s`` nếu process kết thúc với exit status khác 0 hoặc bị killed bất ngờ.

----

6. TẠO SYSTEM USER ĐẢM BẢO AN NINH (PRIVILEGE SEPARATION)
---------------------------------------------------------

Để tuân thủ nguyên tắc quyền tối thiểu (Least Privilege), tạo system user riêng không có thư mục home và shell đăng nhập:

.. code-block:: bash

   sudo useradd --system --no-create-home --shell /bin/false myservice

----

7. QUẢN LÝ LIFECYCLE SERVICE TRÊN SYSTEMD
-----------------------------------------

Sau khi chỉnh sửa hoặc tạo mới Unit file, bắt buộc phải báo cho ``systemd`` nạp lại cấu hình:

.. code-block:: bash

   # 1. Reload cấu hình systemd Manager
   sudo systemctl daemon-reload

   # 2. Khởi chạy Service
   sudo systemctl start myservice

   # 3. Kiểm tra trạng thái thực thi
   systemctl status myservice

   # 4. Tạm dừng Service (Tạo ra tín hiệu SIGTERM gửi tới Process)
   sudo systemctl stop myservice

   # 5. Tái khởi động Service
   sudo systemctl restart myservice

   # 6. Đăng ký tự động chạy khi Boot hệ thống
   sudo systemctl enable myservice

   # Tip: Kích hoạt + Chạy ngay lập tức bằng 1 lệnh
   sudo systemctl enable --now myservice

----

8. MONITORING & LOGGING VỚI JOURNALCTL
---------------------------------------

``systemd`` thu gom trực tiếp ``stdout``/``stderr`` của Service thông qua **Journald**:

.. code-block:: bash

   # Xem toàn bộ Log của service
   journalctl -u myservice

   # Theo dõi Log thời gian thực (Real-time output stream)
   journalctl -u myservice -f

   # Chỉ xem Log sinh ra từ lần Boot hiện tại
   journalctl -u myservice -b

----

9. QUY TRÌNH DEBUG SERVICE CHUYÊN SÂU
--------------------------------------

Sử dụng sơ đồ cây quyết định để khoanh vùng sự cố:

.. code-block:: text

   systemctl status myservice
            │
            ├─► Check Process Alive / Exit Code
            ▼
   journalctl -u myservice -e
            │
            ├─► Analyze Application Runtime Error
            ▼
   systemctl cat myservice
            │
            ├─► Verify ExecStart path & Environment Variables
            ▼
   ps aux | grep myservice
            │
            ├─► Verify PID & Running User/Group Credentials
            ▼
   ls -l /proc/<PID>/
            └─► Check File Descriptors, Cwd, and Memory Maps

Các câu lệnh kiểm tra hữu ích:

.. code-block:: bash

   # Hiển thị cấu hình file Unit thực tế đang chạy
   systemctl cat myservice

   # Xem toàn bộ thuộc tính chi tiết từ systemd DBus Object
   systemctl show myservice

   # Kiểm tra danh sách phụ thuộc (Dependencies)
   systemctl list-dependencies myservice

10. LIÊN KẾT VỚI LINUX KERNEL INTERNALS
----------------------------------------

Một ``systemd service`` không phải là một loại process đặc biệt trong mắt Kernel, mà chỉ là một **User Space Abstraction**.

.. code-block:: text

   +-------------------------------------------------------+
   |                     systemd                           |
   |   (Service Manager / Unit Dependencies / Logging)     |
   +---------------------------┬---------------------------+
                               │
                               ▼
   +-------------------------------------------------------+
   |                    User Space                         |
   |              /usr/local/bin/myservice                 |
   +---------------------------┬---------------------------+
                               │ System Calls (write, sleep)
                               ▼
   +-------------------------------------------------------+
   |                   Linux Kernel                        |
   |  ├─ task_struct (Process Control Block)               |
   |  ├─ Scheduler (CFS / EEVDF - CPU Allocation)           |
   |  ├─ Virtual Memory Management (Page Tables, MMAP)      |
   |  ├─ Signal Handling (SIGTERM delivery)                 |
   |  └─ VFS / Procfs (/proc/<PID>)                         |
   +-------------------------------------------------------+

Phân tích góc nhìn Kernel qua Syscalls (strace):
-------------------------------------------------

Theo dõi các System Calls mà Service gửi xuống Kernel:

.. code-block:: bash

   strace -p <PID>

Các system call cốt lõi tương ứng với hoạt động của Service:

* ``write(1, "myservice is working...\n", ...)``: In log ra stdout.
* ``nanosleep()`` / ``clock_nanosleep()``: Tạo trạng thái ngủ (Sleep state) đưa Task về ``TASK_INTERRUPTIBLE``.
* ``rt_sigaction()``: Đăng ký Signal Handler với Kernel.
* ``exit_group()``: Thoát Process an toàn khi xử lý xong tín hiệu.

----

11. TỔNG KẾT MENTAL MODEL & MÃ CÂU LỆNH TẮT
-------------------------------------------

Mental Model cần ghi nhớ:

.. code-block:: text

   systemd  ──manages──►  Process  ──executes──►  Binary  ──requests──► System Calls ──► Kernel

Bảng tra cứu nhanh các lệnh thao tác (Cheatsheet):
--------------------------------------------------

.. code-block:: bash

   # Build & Install
   gcc -Wall -Wextra -O2 myservice.c -o myservice
   sudo install -m 755 myservice /usr/local/bin/myservice

   # Systemd Management
   sudo systemctl daemon-reload
   sudo systemctl enable --now myservice
   sudo systemctl restart myservice
   systemctl status myservice

   # Inspection & Diagnostics
   journalctl -u myservice -f -b
   systemctl cat myservice
   strace -p $(pidof myservice)