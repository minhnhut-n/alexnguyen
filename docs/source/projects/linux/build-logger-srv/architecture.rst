.. _architecture:

==================================
0. System Architecture & Layout
==================================

Tổng quan
---------
Kiến trúc của hệ thống Logging Daemon trên nền tảng Linux được thiết kế theo mô hình client-server bất đồng bộ, phân tách rõ ràng giữa mã nguồn ứng dụng (Application Code) và tiến trình daemon nền (Logger Daemon). Hai thành phần này giao tiếp với nhau thông qua **Unix Domain Sockets**.
Version 1: https://github.com/minhnhut-n/helper-tool/tree/main/logging-srv/logger

1. Ứng dụng (Application Code / Client)
---------------------------------------
Thành phần ứng dụng đóng vai trò là bên phát sinh thông điệp (Producer), gửi các bản tin log đến tiến trình daemon mà không làm nghẽn luồng xử lý chính.

- **Public API cho Ứng dụng:**
  - `int logger_init(const char *socket_path);` — Thiết lập kết nối đến Unix Domain Socket của daemon.
  - `void log_message(int level, const char *format, ...);` — Gửi bản tin log bất đồng bộ hoặc đồng bộ từ ứng dụng đến daemon.
  - `void logger_close(void);` — Đóng kết nối socket khi ứng dụng kết thúc.

- **Quy trình và Tuần tự (Sequence Flow):**
  1. Ứng dụng gọi hàm khởi tạo `logger_init()` để kết nối tới đường dẫn socket cố định (ví dụ: `/tmp/logger.sock`).
  2. Khi cần ghi log, hàm `log_message()` định dạng chuỗi log và truyền tải dữ liệu qua socket.
  3. Quá trình xử lý được tối ưu hóa để đảm bảo luồng thực thi chính của ứng dụng không bị block.

2. Logger Daemon & Dịch vụ Hệ thống (System Service)
----------------------------------------------------
Tiến trình nền chuyên trách việc tiếp nhận kết nối, đưa dữ liệu vào bộ đệm vòng (Ring Buffer) và ghi xuống tệp tin bất đồng bộ.

- **Quản lý Dịch vụ Systemd (`logger.service`):**
  Daemon được quản lý trực tiếp bởi `systemd`, hỗ trợ tính năng tự động khởi động cùng hệ thống và tự động khôi phục nếu gặp sự cố (`Restart=on-failure`).

- **Cấu trúc Cây Thư mục Dự án (Directory Tree):**
  ::

     ~/app/logger/
     ├── CMakeLists.txt              # Cấu hình build chung cho CMake
     ├── daemon-code/
     │   ├── logger-daemon.c         # Mã nguồn chính của tiến trình daemon (Server)
     │   ├── ring_buffer.c / .h      # Quản lý bộ đệm vòng truyền dữ liệu
     │   └── logger-io.h             # Xử lý ghi file nền (File I/O background worker)
     ├── application-code/
     │   ├── app-code.c              # Mã nguồn ứng dụng mẫu gọi log
     │   └── logger-srv.c            # Thư viện client giao tiếp socket
     └── logger.service              # Cấu hình dịch vụ systemd