Daemon & Service trong Linux
============================

.. rubric:: Giới thiệu

**Daemon** và **Service** là hai khái niệm cốt lõi trong Linux — mọi tác vụ chạy nền,
dịch vụ mạng và quản trị hệ thống đều xoay quanh chúng.

* Hiểu **Daemon** = hiểu *tiến trình chạy ngầm* thực thi công việc.
* Hiểu **Service** = hiểu *lớp quản lý* mà Init system (``systemd``) cung cấp cho người dùng.

.. code-block:: text

   User / Admin ========> Service (systemctl) ========> Daemon (process chạy nền)
                   quản lý theo tên            thực thi I/O, mạng, hardware

---

.. rubric:: 1. Daemon là gì?

**Daemon** (đọc là /'di:mən/) là một tiến trình chạy ngầm (background process),
không gắn với bất kỳ terminal / TTY nào, sống xuyên suốt phiên làm việc.
Chỉ dừng khi:

- Tắt máy, hoặc
- Bị chủ động tìm và ngắt (``kill``, ``systemctl stop``).

**Đặc điểm nhận diện:**

- Không cần bàn phím hay terminal — ``ps`` thấy cột ``TTY`` là ``?``.
- Tên thường kết thúc bằng hậu tố ``d`` — viết tắt của *daemon*:
  ``sshd``, ``httpd``, ``dockerd``, ``crond``, ``systemd``, ``journald``.

**Vai trò:** lắng nghe và phản hồi yêu cầu từ mạng hoặc hệ thống.

.. list-table::
   :header-rows: 1

   * - Daemon
     - Lắng nghe cái gì?
     - Làm gì khi có sự kiện?
   * - ``sshd``
     - Port 22 / kết nối SSH
     - Fork phiên shell bảo mật cho client
   * - ``crond``
     - Thời gian hệ thống
     - Thực thi Cron job đúng lịch
   * - ``dockerd``
     - REST API / Unix socket ``/var/run/docker.sock``
     - Quản lý container lifecycle
   * - ``journald`` / ``rsyslogd``
     - Socket log ``/dev/log``
     - Thu thập và ghi log ra đĩa

.. note::

   Kiểm tra nhanh một daemon có detached khỏi terminal không:

   .. code-block:: bash

      ps -o pid,tty,stat,cmd -p $(pidof crond)
      # TTY = ?  =>  daemon chuẩn, không giữ terminal

---

.. rubric:: 2. Service là gì?

**Service** (Dịch vụ) là khái niệm ở tầng cao hơn — là *tính năng* mà hệ thống
cung cấp cho người dùng / admin, còn **Daemon** là *tiến trình* thực thi tính năng đó.

.. code-block:: text

   +-------------------------------------------------------------+
   |                 High-Level Application                      |
   |       (UI, Web App, Business Logic)                         |
   +-------------------------------------------------------------+
                               |
                               | Gọi API / Interface (IPC)
                               v
   +-------------------------------------------------------------+
   |                    SERVICE INTERFACE                        |
   |  (D-Bus, REST API, Unix Domain Socket, unit file)          |
   +-------------------------------------------------------------+
                               |
                               | Routing & Execution
                               v
   +-------------------------------------------------------------+
   |                   DAEMON (Background)                       |
   |   (Tiến trình C/C++ thực thi: I/O, Hardware, RAM)          |
   +-------------------------------------------------------------+

- **Daemon:** thực thể tiến trình cụ thể đang chạy (góc nhìn kernel / code).
- **Service:** tính năng / dịch vụ mà daemon đó cung cấp (góc nhìn quản trị / user).
- **Ví dụ kinh điển:** *SSH Service* (tính năng truy cập từ xa) được duy trì bởi
  daemon ``sshd`` (tiến trình lắng nghe port 22).

---

.. rubric:: 3. Hệ thống quản lý Daemon và Service (Init System)

Linux cần một *Init system* để quản lý vòng đời daemon:
khởi động, dừng, restart, tự chạy lại khi crash, tự chạy khi boot.

**3.1. systemd — phổ biến nhất hiện nay**

Hầu hết distro hiện đại (Ubuntu, Debian, Fedora, RHEL, CentOS) đều dùng
``systemd`` (PID 1) làm init system mặc định.

- **Unit file:** mỗi service là một file ``.service`` trong
  ``/lib/systemd/system/`` hoặc ``/etc/systemd/system/``.
- **Công cụ quản lý:** ``systemctl``.

.. code-block:: bash

   systemctl status sshd        # xem trạng thái + log gần nhất
   systemctl start sshd         # khởi chạy ngay
   systemctl stop sshd          # dừng ngay
   systemctl restart sshd       # khởi động lại
   systemctl enable sshd        # tự chạy khi boot
   systemctl disable sshd       # bỏ tự chạy khi boot
   systemctl is-enabled sshd    # kiểm tra có enable chưa?
   journalctl -u sshd -f        # theo dõi log realtime của service

.. code-block:: ini

   # Ví dụ tối giản: /etc/systemd/system/my_logger.service
   [Unit]
   Description=My Logging Daemon
   After=network.target

   [Service]
   ExecStart=/usr/local/bin/my_logger
   Restart=always
   RestartSec=2

   [Install]
   WantedBy=multi-user.target

.. tip::

   ``enable`` chỉ tạo symlink trong ``*.wants/`` — không chạy ngay.
   Muốn vừa enable vừa chạy ngay: ``systemctl enable --now <service>``.

**3.2. SysVinit / Upstart — hệ thống cũ**

Trên Linux đời cũ, service là các shell script trong ``/etc/init.d/``,
điều khiển bằng lệnh ``service``:

.. code-block:: bash

   service nginx start
   service nginx stop
   service nginx restart
   ls /etc/init.d/   # danh sách service kiểu SysVinit

.. list-table:: systemd vs SysVinit
   :header-rows: 1

   * - Tiêu chí
     - systemd (hiện đại)
     - SysVinit (cũ)
   * - Định nghĩa service
     - File ``.service`` khai báo (ini format)
     - Shell script trong ``/etc/init.d/``
   * - Lệnh quản lý
     - ``systemctl start/stop/enable``
     - ``service <name> start/stop``
   * - Tự restart khi crash
     - Có — ``Restart=always``
     - Không — phải tự viết watchdog
   * - Log
     - Tập trung via ``journalctl``
     - Rải rác trong ``/var/log/``
   * - Khởi động song song
     - Có — boot nhanh
     - Tuần tự — boot chậm

---

.. rubric:: 4. Các Daemon phổ biến trong Linux

.. list-table::
   :header-rows: 1

   * - Daemon (PID)
     - Vai trò
     - Ghi chú Embedded
   * - ``systemd`` (PID 1)
     - Tiến trình cha của mọi process, init hệ thống
     - Trên embedded nhỏ (Buildroot/BusyBox) có thể thay bằng ``init`` / ``openrc``
   * - ``sshd``
     - Remote shell bảo mật qua SSH
     - Thường tắt trên sản phẩm release để tiết kiệm RAM + an toàn
   * - ``crond``
     - Lập lịch Cron job theo thời gian
     - Thay bằng ``systemd.timer`` gọn nhẹ hơn
   * - ``journald`` / ``rsyslogd``
     - Thu thập log hệ thống và ứng dụng
     - Embedded NOR/NAND flash: cân nhắc ghi RAM (tmpfs) để chống mòn flash
   * - ``systemd-networkd`` / ``NetworkManager``
     - Quản lý Wi-Fi, Ethernet, IP/DHCP
     - Board embedded hay dùng ``systemd-networkd`` + ``wpa_supplicant``
   * - ``dockerd``
     - Quản lý container Docker
     - Chỉ có trên gateway / edge device RAM lớn
   * - ``udevd``
     - Quản lý hotplug device trong ``/dev``
     - Cực kỳ quan trọng với driver / sensor cắm rút nóng

---

.. rubric:: 5. So sánh chi tiết Daemon và Service

.. list-table::
   :header-rows: 1
   :widths: 20 40 40

   * - Tiêu chí
     - Daemon
     - Service
   * - Bản chất
     - Tiến trình phần mềm chạy ngầm (Process)
     - Chức năng / dịch vụ cung cấp cho hệ thống
   * - Định danh
     - Thường tận cùng ``d``: ``httpd``, ``mysqld``
     - Đặt theo tính năng: ``apache2``, ``mysql``, ``ssh``
   * - Góc nhìn
     - Kỹ thuật — OS / tiến trình
     - Quản trị — admin / người dùng cuối
   * - Tương tác
     - Trực tiếp qua PID / signal: ``kill <PID>``
     - Qua init system: ``systemctl start ssh``
   * - Vòng đời
     - ``fork()`` -> ``setsid()`` -> chạy nền
     - ``enable`` -> ``start`` -> ``restart`` -> ``stop``

.. code-block:: bash

   # Daemon view: nhìn theo PID
   pidof sshd          # lấy PID của daemon
   ps -ef | grep sshd  # cây tiến trình
   kill -HUP <PID>     # gửi signal reload trực tiếp

   # Service view: nhìn theo tên dịch vụ
   systemctl status ssh
   systemctl restart ssh

---

.. rubric:: 6. Tóm tắt nhanh

- ``Daemon`` = **process chạy nền**, không TTY, tên hay tận cùng ``d``.
- ``Service`` = **lớp quản lý** bọc quanh daemon qua ``systemd`` + ``systemctl``.
- Nhớ cặp lệnh: ``systemctl enable --now`` để vừa boot-auto vừa chạy ngay,
  ``journalctl -u <service> -f`` để debug log.
- Trên Embedded Linux: càng ít daemon càng tốt — mỗi daemon là RAM + flash + pin.

---

.. rubric:: Tài liệu tham khảo

- `systemd.service — man page <https://man7.org/linux/man-pages/man5/systemd.service.5.html>`_
- `systemctl — man page <https://man7.org/linux/man-pages/man1/systemctl.1.html>`_
- `daemon — man page <https://man7.org/linux/man-pages/man7/daemon.7.html>`_
