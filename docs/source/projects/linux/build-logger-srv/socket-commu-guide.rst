.. _socket-commu-guide:

============================
3 Socket Communication Guide
============================

.. rst-class:: lead

   Client (ứng dụng) và Daemon (logger) giao tiếp với nhau qua **Unix Domain Socket** — một kênh IPC (Inter-Process Communication) nội bộ trên cùng một máy, sử dụng **đường dẫn file** (``/tmp/logger.sock``) làm điểm định danh thay cho địa chỉ IP và port. Tài liệu này mô tả chi tiết kiến trúc, vòng đời kết nối, cấu trúc dữ liệu địa chỉ và cơ chế xử lý dữ liệu dòng.

.. contents:: Nội dung bài viết
   :depth: 2
   :local:

.. grid:: 1 2 2 4
   :gutter: 3

   .. grid-item-card:: 📍 ``AF_UNIX``
      :text-align: center
      :class-card: sd-shadow-sm

      Họ địa chỉ *local*, định danh bằng **đường dẫn file** — không cần IP/port.

   .. grid-item-card:: 🔌 ``SOCK_STREAM``
      :text-align: center
      :class-card: sd-shadow-sm

      Luồng byte **có thứ tự, tin cậy**, tương tự TCP — không mất gói, không lặp.

   .. grid-item-card:: 🧵 Nhiều thread, 1 socket
      :text-align: center
      :class-card: sd-shadow-sm

      Nhiều ``pthread`` cùng gọi ``logger()`` chia sẻ **một** file descriptor.

   .. grid-item-card:: 🚀 Không block hot-path
      :text-align: center
      :class-card: sd-shadow-sm

      Gửi log chỉ là ``write()``; việc nặng (ghi đĩa) do daemon xử lý ở nền.


.. _socket-big-picture:

1. Bức tranh tổng thể
=====================

Hệ thống bao gồm hai tiến trình độc lập với vòng đời socket riêng biệt. Phía **Client** chủ động kết nối; phía **Server (daemon)** lắng nghe và chấp nhận kết nối.

.. code-block:: text
   :caption: Mô hình tương tác Client–Server qua Unix Domain Socket

   ┌────────────────────────┐                        ┌────────────────────────┐
   │   Ứng dụng (CLIENT)    │                        │  Logger Daemon (SERVER)│
   │   logger-srv.c         │                        │  logger-daemon.c       │
   └───────────┬────────────┘                        └───────────┬────────────┘
               │                                                 │
     socket()  │ ① tạo endpoint                                 │ ① unlink() dọn socket cũ
               │ ② connect(addr) ─────────────┐   ┌─────────────│ ② socket()
               │                              │   │             │ ③ bind(addr) gắn đường dẫn
               │                              ▼   ▼             │ ④ listen() mở hàng chờ
               │                        ╱─────────────╮         │ ⑤ accept() nhận client
               │                       │   Kênh ảo   │         │
     write()   │ ③ write(log) ────────▶│  /tmp/      │────────▶│ ⑥ read() → ring_buffer
               │                       │  logger.sock│         │    (worker ghi file nền)
     close()   │ ④ close() ────────────▶╲─────────────╱         │
               ▼                                                 ▼

.. note::
   **Vì sao dùng Unix Domain Socket thay vì TCP Loopback?**
   Đối với giao tiếp trên cùng một máy host, ``AF_UNIX`` đạt hiệu năng cao hơn đáng kể nhờ bỏ qua các tầng xử lý của TCP/IP stack (checksum, routing, loopback interface). Đồng thời, nó giúp tiết kiệm tài nguyên port và cho phép phân quyền truy cập thông qua cơ chế phân quyền file (file permissions).

.. seealso::
   Dữ liệu sau khi daemon tiếp nhận sẽ được chuyển qua :ref:`ring-buffer-impl` trước khi worker thread ghi xuống đĩa. Socket đóng vai trò là tầng vận chuyển (transport layer), còn ring buffer đóng vai trò tầng điều phối dữ liệu (data buffering layer).


.. _socket-client-lifecycle:

2. Vòng đời socket phía Client (ứng dụng)
===========================================

Client đóng vai trò là bên **chủ động (active endpoint)**.

Tuần tự System Call
-------------------

.. code-block:: text
   :caption: Trình tự các lời gọi hệ thống ở phía Client

   socket()  ──▶  connect()  ──▶  write() ... (gửi log) ...  ──▶  close()
   (Tạo fd)       (Kết nối)       (Truyền dữ liệu)               (Giải phóng)

Mã nguồn triển khai
-------------------

.. code-block:: c
   :caption: logger-srv.c — logger_init(): Khởi tạo endpoint và kết nối tới daemon

   int logger_init(const char *socket_path) {
       g_sock_fd = socket(AF_UNIX, SOCK_STREAM, 0);   // ① Tạo socket descriptor
       if (g_sock_fd < 0) return -1;

       struct sockaddr_un addr;
       memset(&addr, 0, sizeof(addr));                // ② Xoá bộ nhớ đệm struct
       addr.sun_family = AF_UNIX;                     //    Cấu hình họ AF_UNIX
       strncpy(addr.sun_path, socket_path,            //    Thiết lập đường dẫn socket
               sizeof(addr.sun_path) - 1);

       if (connect(g_sock_fd,                         // ③ Thực hiện kết nối tới daemon
                   (struct sockaddr *)&addr,
                   sizeof(addr)) < 0) {
           close(g_sock_fd);                          //    Giải phóng fd nếu lỗi
           g_sock_fd = -1;
           return -1;
       }
       return 0;                                      //    Khởi tạo thành công
   }

Đặc điểm hoạt động
------------------

.. grid:: 1
   :gutter: 3

   .. grid-item-card:: 📤 ``write()`` — Truyền dữ liệu
      :class-card: sd-shadow-sm

      Hàm ``logger()`` định dạng chuỗi theo dạng trình bày ``"<level>: <msg>\n"`` và gọi ``write()``. Vì nhiều thread dùng chung ``g_sock_fd``, việc đóng gói tin trong **một lời gọi ``write()`` duy nhất** giúp giảm nguy cơ xen kẽ dữ liệu.

      .. code-block:: c

         char buffer[512];
         int len = snprintf(buffer, sizeof(buffer), "%d: %s\n", level, message);
         write(g_sock_fd, buffer, len);

   .. grid-item-card:: 🔕 Không ``read()`` ở Client
      :class-card: sd-shadow-sm

      Giao tiếp vận hành theo mô hình **một chiều (unidirectional)**: client gửi log và không chờ phản hồi từ daemon. Cơ chế này giữ cho lời gọi ``logger()`` đạt tốc độ phản hồi cao, không gây cản trở (non-blocking) luồng nghiệp vụ chính.

.. tip::
   Thao tác ``memset(&addr, 0, sizeof(addr))`` giúp xóa sạch phần **padding** và dữ liệu dư thừa trong vùng đệm ``sun_path``, đảm bảo địa chỉ không bị ảnh hưởng bởi byte rác khiến hàm ``connect()`` thất bại.


.. _socket-server-lifecycle:

3. Vòng đời socket phía Server (daemon)
=======================================

Server đóng vai trò là bên **bị động (passive endpoint)**: thiết lập điểm kết nối cố định và lắng nghe các yêu cầu từ client.

Tuần tự System Call
-------------------

.. code-block:: text
   :caption: Trình tự các lời gọi hệ thống ở phía Server

   unlink() ─▶ socket() ─▶ bind() ─────────▶ listen() ──────▶ accept() ──▶ read()
   (Dọn file)  (Tạo fd)   (Gắn đường dẫn) (Mở hàng chờ)    (Nhận kết   (Đọc dữ
                                                            nối)        liệu)
                                                              │           │
                                                              └── Vòng lặp ┘

Mã nguồn triển khai
-------------------

.. code-block:: c
   :caption: logger-daemon.c — Thiết lập endpoint và quản lý vòng lặp nhận kết nối

   // ① Dọn dẹp file socket cũ nếu còn tồn tại từ lần chạy trước
   unlink(SOCK_PATH);

   // ② Tạo socket kiểu byte stream trong họ AF_UNIX
   int server_fd = socket(AF_UNIX, SOCK_STREAM, 0);
   if (server_fd < 0) { perror("Socket error"); return -1; }

   // ③ Cấu hình thông tin địa chỉ và gán (bind) với đường dẫn file
   struct sockaddr_un addr;
   memset(&addr, 0, sizeof(addr));
   addr.sun_family = AF_UNIX;
   strncpy(addr.sun_path, SOCK_PATH, sizeof(addr.sun_path) - 1);

   // ④ Bind địa chỉ và kích hoạt chế độ lắng nghe (backlog = 5)
   if (bind(server_fd, (struct sockaddr *)&addr, sizeof(addr)) < 0 ||
       listen(server_fd, 5) < 0) {
       perror("Bind/Listen error");
       close(server_fd);
       return -1;
   }

   // ⑤ Vòng lặp chấp nhận và xử lý kết nối từ client
   while (1) {
       int client_fd = accept(server_fd, NULL, NULL);  // Chấp nhận kết nối
       if (client_fd < 0) continue;

       // ⑥ Đọc liên tục cho đến khi client ngắt kết nối
       while ((n = read(client_fd, buffer, sizeof(buffer))) > 0) {
           /* Tách dòng, parse dạng "<level>: <msg>" → ring_buffer_push() */
       }
       close(client_fd);                               // Giải phóng client fd
   }

Lưu ý quan trọng khi triển khai
-------------------------------

.. grid:: 1
   :gutter: 3

   .. grid-item-card:: 🧹 Vai trò của ``unlink()``
      :class-card: sd-shadow-sm

      Mỗi khi gọi ``bind()``, Kernel sẽ tạo một **file socket đặc biệt** tại đường dẫn ``sun_path``. Nếu tiến trình trước đó kết thúc không hoàn chỉnh, file này vẫn tồn tại trên hệ thống file, làm cho lần gọi ``bind()`` tiếp theo gặp lỗi *"Address already in use"*. Việc gọi ``unlink()`` trước khi bind đảm bảo môi trường khởi tạo luôn sạch.

.. warning::
   **Phân biệt các cơ chế hàng chờ:**

   * **Backlog trong ``listen(fd, 5)``:** Quản lý hàng chờ các yêu cầu kết nối chưa hoàn tất quá trình chấp nhận ở tầng Kernel.
   * **Vòng lặp ``accept()``:** Xử lý **tuần tự** từng kết nối client. Một client phải hoàn tất truyền dữ liệu và đóng kết nối thì server mới tiếp tục nhận client kế tiếp.

.. note::
   Luồng xử lý ``accept()`` và ``read()`` hoạt động trên một luồng chính. Luồng ghi đĩa (Worker Thread) được khởi tạo riêng biệt bằng ``pthread_create()`` và trao đổi dữ liệu qua ring buffer, tránh việc I/O đĩa làm nghẽn tiến trình nhận socket.


.. _socket-struct-casting:

4. Cấu trúc địa chỉ và cơ chế ép kiểu
======================================

Các API socket chuẩn như ``bind()``, ``connect()``, và ``accept()`` nhận tham số địa chỉ dạng con trỏ chung **``struct sockaddr *``**. Tuy nhiên, với họ Unix Domain Socket, dữ liệu địa chỉ được định nghĩa chi tiết qua **``struct sockaddr_un``**.

So sánh cấu trúc dữ liệu
------------------------

.. grid:: 1
   :gutter: 3

   .. grid-item-card:: 🧬 ``struct sockaddr`` (Kiểu chung)
      :class-card: sd-shadow-sm

      Cấu trúc tổng quát dùng làm interface cho tất cả các họ socket.

      .. code-block:: c

         struct sockaddr {
             sa_family_t sa_family;   // Họ địa chỉ (2 bytes)
             char        sa_data[14]; // Mảng dữ liệu địa chỉ thô
         };

   .. grid-item-card:: 📇 ``struct sockaddr_un`` (Kiểu UNIX)
      :class-card: sd-shadow-sm

      Cấu trúc chuyên biệt cho họ ``AF_UNIX``, mở rộng vùng lưu trữ địa chỉ thành đường dẫn file.

      .. code-block:: c

         struct sockaddr_un {
             sa_family_t sun_family;    // Họ địa chỉ (= AF_UNIX)
             char        sun_path[108]; // Đường dẫn socket file
         };

Bố cục bộ nhớ và cơ chế Ép kiểu
-------------------------------

Cả hai cấu trúc đều có thành viên đầu tiên chứa thông tin họ địa chỉ (``sa_family`` / ``sun_family``). Nhờ tính tương thích về bố cục bộ nhớ ở phần đầu, con trỏ kiểu ``struct sockaddr_un *`` có thể ép kiểu an toàn về ``struct sockaddr *``.

.. code-block:: text
   :caption: Bố cục bộ nhớ tương thích giữa sockaddr_un và sockaddr

   struct sockaddr_un (Chuyên biệt, ~110 bytes)   struct sockaddr (Tổng quát, 16 bytes)
   ┌──────────────────────┬────────────────┐     ┌───────────────┬──────────────┐
   │ sun_family (2 bytes) │ sun_path[108]  │     │ sa_family     │ sa_data[14]  │
   └──────────────────────┴────────────────┘     └───────────────┴──────────────┘
   └────────── Phần đầu trùng khớp ─────────┘
                          │
                          ▼  Ép kiểu: (struct sockaddr *)&addr
        API đọc thông tin họ địa chỉ (AF_UNIX) trước,
        sau đó đọc tiếp vùng dữ liệu theo kích thước truyền vào (sizeof)

Mẫu mã nguồn áp dụng
--------------------

.. code-block:: c

   struct sockaddr_un addr;
   memset(&addr, 0, sizeof(addr));                  // 1. Khởi tạo cấu trúc cụ thể
   addr.sun_family = AF_UNIX;                       //    Gán họ địa chỉ
   strncpy(addr.sun_path, SOCK_PATH,                //    Gán đường dẫn
           sizeof(addr.sun_path) - 1);

   // 2. Ép kiểu con trỏ sang (struct sockaddr *) và truyền độ dài sizeof(addr)
   bind(server_fd, (struct sockaddr *)&addr, sizeof(addr));
   connect(g_sock_fd, (struct sockaddr *)&addr, sizeof(addr));

.. warning::
   **Lỗi sai thường gặp về kích thước:**
   Truyền ``sizeof(struct sockaddr)`` (16 bytes) thay vì ``sizeof(struct sockaddr_un)`` sẽ làm Kernel chỉ đọc 16 bytes đầu tiên. Điều này dẫn đến việc chuỗi ``sun_path`` bị cắt ngắn, gây lỗi ``bind`` hoặc ``connect`` thất bại. Luôn truyền tham số độ dài là ``sizeof(addr)`` hoặc ``sizeof(struct sockaddr_un)``.

Bảng tra cứu cấu trúc
---------------------

.. list-table::
   :header-rows: 1
   :widths: 25 37 38

   * - Khía cạnh
     - ``struct sockaddr``
     - ``struct sockaddr_un``
   * - **Phạm vi sử dụng**
     - Interface chung cho các API socket
     - Cấu trúc dữ liệu riêng cho ``AF_UNIX``
   * - **Trường họ địa chỉ**
     - ``sa_family``
     - ``sun_family`` (Giá trị: ``AF_UNIX``)
   * - **Trường địa chỉ**
     - ``sa_data[14]`` (Dữ liệu thô)
     - ``sun_path[108]`` (Đường dẫn file)
   * - **Kích thước bộ nhớ**
     - 16 bytes
     - ~110 bytes
   * - **Mục đích**
     - Nhận tham số trong ``bind``, ``connect``, ``accept``
     - Khởi tạo dữ liệu trước khi ép kiểu


.. _socket-message-framing:

5. Đóng gói bản tin (Message Framing)
=====================================

Kiểu socket ``SOCK_STREAM`` cung cấp một **luồng byte liên tục (byte stream)** và không tự duy trì ranh giới giữa các lần gọi ``write()``. Do đó, ứng dụng phải triển khai cơ chế phân tách bản tin (message framing). Trong hệ thống này, các bản tin được phân tách bằng ký tự xuống dòng (``\n``).

Cơ chế xử lý hai phía
---------------------

.. grid:: 1
   :gutter: 3

   .. grid-item-card:: ✍️ Phía gửi (Client)
      :class-card: sd-shadow-sm

      Mỗi bản tin log được đóng gói hoàn chỉnh với ký tự kết thúc ``\n`` trước khi truyền qua socket:

      .. code-block:: c

         snprintf(buffer, sizeof(buffer), "%d: %s\n", level, message);
         write(g_sock_fd, buffer, len);

   .. grid-item-card:: 📖 Phía nhận (Server)
      :class-card: sd-shadow-sm

      Daemon xử lý dữ liệu đọc được bằng cách tích lũy byte vào bộ đệm dòng cho đến khi gặp ký tự ``\n``:

      .. code-block:: c

         if (sscanf(line, "%d: %255[^\r\n]", &level, msg) == 2)
             ring_buffer_push(&g_ringbuf, level, msg);

Biểu diễn luồng dữ liệu Stream
------------------------------

.. code-block:: text
   :caption: Tách bản tin từ luồng byte stream phân mảnh

   Client gửi:      "2: task A\n"          "2: task B\n"
                        │ write()              │ write()
                        ▼                      ▼
   ═══════════════════ LUỒNG BYTE STREAM LIÊN TỤC ═══════════════════
                        │                      │
   Server nhận:  [ read() nhận: "2: tas" ]  [ read() nhận: "k A\n2: task B\n" ]
                        │                      │
                        ▼                      ▼
                 Tích lũy vào bộ đệm     Gặp '\n' → Tách thành 2 bản tin:
                 (Chưa xử lý)            "2: task A" và "2: task B" → Push

.. tip::
   Bộ đệm dòng (``line[]`` + ``line_length``) phía daemon đảm bảo dữ liệu được khôi phục chính xác bất kể số lượng byte trả về trong mỗi lời gọi ``read()``.

.. warning::
   Nếu một dòng log vượt quá kích thước tối đa của bộ đệm (``512`` bytes), daemon sẽ bật cờ ``discarding_line`` để hủy phần dữ liệu dư thừa cho đến ký tự ``\n`` tiếp theo, tránh sự cố tràn bộ nhớ đệm (buffer overflow).


.. _socket-multithreading:

6. Chia sẻ Socket giữa nhiều Thread
===================================

Trong kiến trúc client, nhiều ``pthread`` có thể cùng gọi hàm ``logger()`` và chia sẻ chung một file descriptor (``g_sock_fd``).

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: ✅ An toàn ở cấp độ Kernel
      :class-card: sd-shadow-sm

      Lời gọi hệ thống ``write()`` mang tính nguyên tử (atomic operation) ở mức Kernel đối với dung lượng dữ liệu nhỏ hơn ngưỡng đệm socket. Dữ liệu của một lời gọi ``write()`` sẽ không bị ghi đè chéo byte bởi lời gọi khác.

   .. grid-item-card:: ⚠️ Thứ tự bản tin
      :class-card: sd-shadow-sm

      Nếu hai thread gọi ``write()`` đồng thời, thứ tự các dòng log xuất hiện trên socket phụ thuộc vào tiến trình lập lịch của Kernel.

   .. grid-item-card:: 🧯 Biện pháp giảm thiểu
      :class-card: sd-shadow-sm

      Gộp toàn bộ chuỗi log vào một lời gọi ``write()`` duy nhất giúp đảm bảo tính toàn vẹn của từng dòng log. Trường hợp cần đảm bảo thứ tự tuyệt đối, có thể bổ sung cơ chế đồng bộ (như ``pthread_mutex``) xung quanh hàm ghi log.

   .. grid-item-card:: 🧩 Đánh giá thiết kế
      :class-card: sd-shadow-sm

      Đối với tác vụ ghi log hệ thống, việc các dòng log từ những thread khác nhau xuất hiện đan xen theo thời gian thực là hành vi bình thường và có thể chấp nhận được.


.. _socket-summary:

7. Tổng kết
===========

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - Khái niệm
     - Tóm tắt kỹ thuật
   * - **Họ địa chỉ**
     - ``AF_UNIX``: Định danh qua đường dẫn file, tối ưu hiệu năng IPC nội bộ.
   * - **Kiểu Socket**
     - ``SOCK_STREAM``: Luồng byte tin cậy, bảo đảm thứ tự dữ liệu.
   * - **Vòng đời Client**
     - ``socket()`` → ``connect()`` → ``write()`` → ``close()``
   * - **Vòng đời Server**
     - ``unlink()`` → ``socket()`` → ``bind()`` → ``listen()`` → ``accept()`` → ``read()``
   * - **Ép kiểu cấu trúc**
     - Cast từ ``struct sockaddr_un *`` sang ``struct sockaddr *`` nhờ tương thích layout bộ nhớ.
   * - **Tham số kích thước**
     - Luôn sử dụng ``sizeof(struct sockaddr_un)`` trong các hàm giao tiếp socket.
   * - **Đóng gói dữ liệu**
     - Sử dụng ký tự xuống dòng (``\n``) làm ranh giới phân tách bản tin trên luồng byte.
   * - **Mô hình I/O**
     - Client thực hiện write bất đồng bộ tương đối; Server tiếp nhận dữ liệu và đẩy vào Ring Buffer xử lý nền.

.. seealso::
   * :ref:`ring-buffer-impl` — Chi tiết cấu trúc dữ liệu vòng đệm tiếp nhận dữ liệu từ socket server.
   * :ref:`atomic-ops-label` — Cơ chế đồng bộ không khóa (lock-free) ứng dụng trong hệ thống.