.. _socket-commu-guide:


================================
3. Socket Communication Guide
================================

.. rst-class:: lead

   Client (ứng dụng) và Daemon (logger) nói chuyện với nhau qua **Unix Domain
   Socket** — một kênh giao tiếp nội bộ trên cùng một máy, dùng **đường dẫn file**
   (``/tmp/logger.sock``) thay cho địa chỉ IP. Bài viết này trình bày *tuần tự
   từng bước mở/đóng socket* ở cả hai phía và làm rõ các ``struct`` địa chỉ cùng
   cơ chế **ép kiểu** giữa chúng.

.. grid:: 1 2 2 4
   :gutter: 3

   .. grid-item-card:: 📍 ``AF_UNIX``
      :text-align: center
      :class-card: sd-shadow-sm

      Họ địa chỉ *local*, định danh bằng **đường dẫn file** — không cần IP/port.

   .. grid-item-card:: 🔌 ``SOCK_STREAM``
      :text-align: center
      :class-card: sd-shadow-sm

      Luồng byte **có thứ tự, tin cậy**, giống TCP — không mất gói, không lặp.

   .. grid-item-card:: 🧵 Nhiều thread, 1 socket
      :text-align: center
      :class-card: sd-shadow-sm

      Nhiều ``pthread`` cùng gọi ``logger()`` chia sẻ **một** file descriptor.

   .. grid-item-card:: 🚀 Không block hot-path
      :text-align: center
      :class-card: sd-shadow-sm

      Gửi log chỉ là ``write()``; việc nặng (ghi file) do daemon lo ở nền.

----

.. rubric:: 1. Bức tranh tổng thể — ai nói chuyện với ai

Hệ thống gồm hai phía, mỗi phía có một vòng đời socket riêng. Phía **Client**
chủ động kết nối; phía **Server (daemon)** lắng nghe và chấp nhận.

.. code-block:: text
   :caption: Mô hình client–server qua Unix Domain Socket

   ┌────────────────────────┐                        ┌────────────────────────┐
   │   Ứng dụng (CLIENT)     │                        │  Logger Daemon (SERVER) │
   │   logger-srv.c         │                        │  logger-daemon.c       │
   └───────────┬────────────┘                        └───────────┬────────────┘
               │                                                 │
     socket()  │ ① tạo endpoint                                 │ ① unlink() dọn socket cũ
               │ ② connect(addr) ─────────────┐   ┌─────────────│ ② socket()
               │                              │   │             │ ③ bind(addr)  gắn đường dẫn
               │                              ▼   ▼             │ ④ listen()    mở hàng chờ
               │                         ╱─────────────╮        │ ⑤ accept()    nhận client
               │                        │  Kênh ảo     │        │
               │ ③ write(log) ─────────▶│  /tmp/       │───────▶│ ⑥ read() → ring_buffer
               │                        │  logger.sock │        │     (worker ghi file nền)
               │ ④ close() ────────────▶╲─────────────╱        │
               │                                                 │
               ▼                                                 ▼

.. note::
   **Vì sao dùng Unix Domain Socket thay vì TCP loopback?** Với giao tiếp *cùng
   máy*, ``AF_UNIX`` **nhanh hơn** (bỏ qua stack TCP/IP), **không tốn port**, và
   phân quyền được bằng quyền truy cập file của chính đường dẫn socket.

.. seealso::
   Dữ liệu sau khi vào daemon được đưa vào :ref:`ring-buffer-impl` trước khi
   worker ghi xuống đĩa — đó là lớp *điều phối dữ liệu*, còn socket chỉ là *đường vận chuyển*.


----

.. rubric:: 2. Vòng đời socket phía Client (ứng dụng)

Client là bên **chủ động**. Chuỗi bước ngắn gọn:

.. code-block:: text
   :caption: Thứ tự gọi hệ thống ở client (dấu ? = bước tuỳ chọn)

   socket()  ──▶  connect()  ──▶  write() …(gửi nhiều lần)…  ──▶  close()
   (tạo fd)      (bắt tay)       (đẩy dữ liệu log)                (giải phóng)

.. code-block:: c
   :caption: logger-srv.c — logger_init(): tạo endpoint và kết nối tới daemon

   int logger_init(const char *socket_path) {
       g_sock_fd = socket(AF_UNIX, SOCK_STREAM, 0);   // ① tạo socket
       if (g_sock_fd < 0) return -1;

       struct sockaddr_un addr;
       memset(&addr, 0, sizeof(addr));                // ② zero-hoá địa chỉ
       addr.sun_family = AF_UNIX;                     //    chọn họ AF_UNIX
       strncpy(addr.sun_path, socket_path,            //    gán đường dẫn
               sizeof(addr.sun_path) - 1);

       if (connect(g_sock_fd,                         // ③ kết nối tới daemon
                   (struct sockaddr *)&addr,
                   sizeof(addr)) < 0) {
           close(g_sock_fd);                          //    thất bại → dọn fd
           g_sock_fd = -1;
           return -1;
       }
       return 0;                                      //    sẵn sàng gửi log
   }

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: 📤 ``write()`` — đẩy log đi
      :class-card: sd-shadow-sm

      ``logger()`` định dạng ``"<level>: <msg>\n"`` rồi ``write()`` lên socket.
      Vì nhiều thread dùng chung ``g_sock_fd``, mỗi lần gửi nên gom **một
      ``write()`` duy nhất** để giảm nguy cơ xen kẽ (xem mục 6).

      .. code-block:: c

         char buffer[512];
         int len = snprintf(buffer, sizeof(buffer), "%d: %s\n", level, message);
         write(g_sock_fd, buffer, len);

   .. grid-item-card:: 🔕 Không ``read()`` ở client
      :class-card: sd-shadow-sm

      Kênh này **một chiều**: client chỉ *gửi*, không chờ phản hồi. Nhờ vậy
      ``logger()`` gần như tức thời, **không block** luồng nghiệp vụ của ứng dụng.

.. tip::
   ``memset(&addr, 0, sizeof(addr))`` trước khi gán không phải "trang trí". Nó
   zero-hoá phần **padding** và **đuôi** ``sun_path`` chưa dùng, tránh địa chỉ bị
   nhiễu byte rác gây ``connect`` thất bại khó hiểu.



----

.. rubric:: 3. Vòng đời socket phía Server (daemon)

Server là bên **bị động**: nó tạo một "điểm hẹn" cố định, rồi ngồi chờ client
tới. Chuỗi bước dài hơn client vì có thêm *bind* và *listen*:

.. code-block:: text
   :caption: Thứ tự gọi hệ thống ở server

   unlink()  ─▶  socket()  ─▶  bind()  ─▶  listen()  ─▶  accept()  ─▶  read()
   (dọn cũ)     (tạo fd)      (gắn        (mở hàng      (nhận         (nhận
       ?                        đường dẫn)   chờ)         client)       dữ liệu)
                                                                          │
                                                        vòng lặp: lại accept()
                                                        cho client kế tiếp

.. code-block:: c
   :caption: logger-daemon.c — dựng endpoint, lắng nghe và nhận kết nối

   // ① Dọn socket cũ lại từ lần chạy trước (tránh lỗi "Address already in use")
   unlink(SOCK_PATH);

   // ② Tạo socket kiểu stream trong họ AF_UNIX
   int server_fd = socket(AF_UNIX, SOCK_STREAM, 0);
   if (server_fd < 0) { perror("Socket error"); return -1; }

   // ③ Chuẩn bị địa chỉ và gắn (bind) với đường dẫn file
   struct sockaddr_un addr;
   memset(&addr, 0, sizeof(addr));
   addr.sun_family = AF_UNIX;
   strncpy(addr.sun_path, SOCK_PATH, sizeof(addr.sun_path) - 1);

   // ④ bind() + listen() — mở "cửa" và bật hàng chờ tối đa 5 kết nối
   if (bind(server_fd, (struct sockaddr *)&addr, sizeof(addr)) < 0 ||
       listen(server_fd, 5) < 0) {
       perror("Bind/Listen error"); close(server_fd); return -1;
   }

   // ⑤ Vòng lặp chấp nhận client
   while (1) {
       int client_fd = accept(server_fd, NULL, NULL);  // nhận 1 client
       if (client_fd < 0) continue;

       // ⑥ Đọc dữ liệu cho tới khi client đóng kết nối
       while ((n = read(client_fd, buffer, sizeof(buffer))) > 0) {
           /* tách từng dòng, parse "<level>: <msg>" → ring_buffer_push() */
       }
       close(client_fd);                                // client rời → đóng fd riêng
   }

.. grid:: 1
   :gutter: 3

   .. grid-item-card:: 🧹 Vì sao phải ``unlink()`` trước?
      :class-card: sd-shadow-sm

      Mỗi lần ``bind()``, kernel tạo một **file đặc biệt** tại ``sun_path``. Nếu
      tiến trình trước chết đột ngột, file đó còn sót lại → lần chạy sau ``bind()``
      sẽ báo lỗi *"Address already in use"*. ``unlink()`` xoá file cũ để **luôn**
      khởi động sạch.

.. warning::
   Hai hàng đợi cần phân biệt rõ:

   * ``listen(fd, 5)`` — **backlog**: hàng chờ các kết nối *chưa* được ``accept``
     (do TCP/stream three-way handshake).
   * Vòng ``while(1) { accept() … }`` — server **tuần tự** phục vụ từng client.
     Client đang được phục vụ xong (``read`` trả 0) thì client kế tiếp mới vào.

.. note::
   Trong mã hiện tại, ``accept()`` và toàn bộ ``read()`` chạy trên **một luồng
   duy nhất**. Worker ghi file (``pthread_create(&worker, …)``) là luồng *riêng*,
   nhận dữ liệu qua ring buffer — nhờ vậy việc ghi đĩa chậm **không làm nghẽn**
   việc nhận socket.



----

.. rubric:: 4. Các ``struct`` địa chỉ & mối liên hệ ép kiểu

Đây là chỗ "gây hoang mang" nhất khi mới học socket. Các API như ``bind()`` /
``connect()`` / ``accept()`` đều nhận con trỏ **``struct sockaddr *``** — kiểu
*tổng quát*. Nhưng ta lại xây địa chỉ cụ thể bằng **``struct sockaddr_un``**. Vậy
hai kiểu này liên hệ nhau thế nào?

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: 🧬 ``struct sockaddr`` — kiểu chung
      :class-card: sd-shadow-sm

      Kiểu *tổng quát*, dùng cho **mọi** họ địa chỉ. Chỉ có họ address và một
      vùng dữ liệu thô.

      .. code-block:: c

         struct sockaddr {
             sa_family_t sa_family;  // họ địa chỉ (2 byte)
             char        sa_data[14]; // địa chỉ dạng thô
         };

   .. grid-item-card:: 📇 ``struct sockaddr_un`` — kiểu Unix
      :class-card: sd-shadow-sm

      Biến thể *chuyên biệt* cho ``AF_UNIX``, thay vùng ``sa_data`` thô bằng
      **đường dẫn file** dễ đọc.

      .. code-block:: c

         struct sockaddr_un {
             sa_family_t sun_family;   // = AF_UNIX
             char        sun_path[108]; // đường dẫn socket
         };

.. important::
   **Điểm mấu chốt:** cả hai struct đều **bắt đầu** bằng một thành viên cùng kiểu
   là *address family* (``sa_family`` / ``sun_family``). Nhờ phần đầu **khớp nhau
   về bố cục bộ nhớ**, ta có thể ép kiểu an toàn từ ``sockaddr_un *`` sang
   ``sockaddr *``.

.. code-block:: text
   :caption: Bố cục bộ nhớ — phần đầu khớp nhau nên ép kiểu được

   struct sockaddr_un (cụ thể, ~110 byte)     struct sockaddr (chung, 16 byte)
   ┌──────────────────────┬────────────────┐  ┌───────────────┬──────────────┐
   │ sun_family (2 byte)  │ sun_path[108]  │  │ sa_family     │ sa_data[14]  │
   └──────────────────────┴────────────────┘  └───────────────┴──────────────┘
   └──────────── phần đầu KHỚP nhau ─────────┘
                     │
                     ▼  ép kiểu:  (struct sockaddr *)&addr
        API chỉ đọc: family trước → đó là AF_UNIX
        rồi đọc tiếp phần còn lại theo đúng kích thước ta TRUYỀN VÀO (sizeof)

.. dropdown:: Ép kiểu trong thực tế viết thế nào?
   :color: primary
   :open:

   Cả client lẫn server đều dùng cùng một khuôn:

   .. code-block:: c

      struct sockaddr_un addr;
      memset(&addr, 0, sizeof(addr));      // 1. dựng struct CỤ THỂ
      addr.sun_family = AF_UNIX;           //    điền họ địa chỉ
      strncpy(addr.sun_path, SOCK_PATH,    //    điền đường dẫn
              sizeof(addr.sun_path) - 1);

      // 2. ép về sockaddr* cho API chung, TRUYỀN sizeof(struct sockaddr_un)
      bind(server_fd, (struct sockaddr *)&addr, sizeof(addr));
      connect(g_sock_fd, (struct sockaddr *)&addr, sizeof(addr));

   * ``(struct sockaddr *)&addr`` — "hãy coi khối nhớ này theo kiểu chung".
   * ``sizeof(addr)`` — **bắt buộc** là kích thước struct *cụ thể*
     (``sizeof(struct sockaddr_un)``), để kernel biết đọc đủ byte của ``sun_path``.

.. warning::
   **Sai lầm kinh điển:** truyền ``sizeof(struct sockaddr)`` (16 byte) thay vì
   ``sizeof(struct sockaddr_un)``. Khi đó kernel chỉ đọc 16 byte → cắt ngang
   ``sun_path`` → ``bind``/``connect`` thất bại hoặc nối sai đường dẫn. Hãy nhớ:
   *luôn* ``sizeof`` của biến ``sockaddr_un`` mà bạn vừa điền.

.. rubric:: Bảng tra nhanh — hai struct

.. list-table::
   :header-rows: 1
   :widths: 24 38 38

   * - Khía cạnh
     - ``struct sockaddr``
     - ``struct sockaddr_un``
   * - Vai trò
     - Kiểu **chung** cho mọi họ (IPv4/IPv6/Unix…)
     - Kiểu **riêng** cho ``AF_UNIX``
   * - Trường định danh
     - ``sa_family``
     - ``sun_family`` (giá trị ``AF_UNIX``)
   * - Trường địa chỉ
     - ``sa_data[14]`` (byte thô)
     - ``sun_path[108]`` (đường dẫn file)
   * - Kích thước
     - 16 byte
     - ~110 byte
   * - Ai nhận struct này
     - Các API: ``bind``, ``connect``, ``accept``
     - **Ta** dựng và điền trước khi ép kiểu



----

.. rubric:: 5. Giao thức trên dây (message framing)

``SOCK_STREAM`` là **luồng byte liên tục**, không có ranh giới "gói tin". Nếu
client gửi 3 lần, phía nhận có thể nhận dồn thành 1 lần, hoặc 1 lần bị chẻ làm
đôi. Vì vậy hai bên phải **tự thỏa thuận** cách phân tách bản tin — ở đây là
**một dòng một log**, kết thúc bằng ký tự ``\n``.

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: ✍️ Phía gửi — thêm ``\n``
      :class-card: sd-shadow-sm

      Mỗi log được định dạng thành **một dòng** kết thúc bởi ``\n``:

      .. code-block:: c

         snprintf(buffer, sizeof(buffer), "%d: %s\n", level, message);
         write(g_sock_fd, buffer, len);

   .. grid-item-card:: 📖 Phía nhận — tách theo ``\n``
      :class-card: sd-shadow-sm

      Daemon quét từng byte, **gom cho đủ một dòng** (tới ``\n``) rồi mới parse:

      .. code-block:: c

         if (sscanf(line, "%d: %255[^\r\n]", &level, msg) == 2)
             ring_buffer_push(&g_ringbuf, level, msg);

.. code-block:: text
   :caption: Vấn đề "stream không có biên" — buffer có thể đến rời rạc

   Client gửi:      "2: nhut job1\n"      "2: nhut job2\n"
                        │ write()              │ write()
                        ▼                      ▼
   ═══════════════ ĐƯỜNG ỐNG STREAM (dòng byte liên tục) ═══════════════
                        │                      │
   Server nhận:   [ read() trả:  "2: nhut "  ]   [ read() trả: "job1\n2: nhut job2\n" ]
                        │                                   │
                        ▼                                   ▼
                  chưa có '\n' → giữ lại          có '\n' → tách thành 2 dòng hợp lệ
                  đợi read() kế tiếp             "nhut job1" + "nhut job2" → push

.. tip::
   Nhờ bộ **đệm theo dòng** (``line[]`` + ``line_length``) phía daemon, việc
   ``read()`` trả về *rời rạc thế nào cũng không sao*: nó cứ nạp dồn vào ``line``
   cho tới khi gặp ``\n`` mới xử lý. Đây chính là kỹ thuật **framing** — biến
   luồn byte thô thành các bản tin có nghĩa.

.. warning::
   Nếu một dòng **dài hơn** vùng đệm (``512`` byte), daemon kích hoạt cờ
   ``discarding_line`` để **bỏ qua** phần thừa thay vì ghi đè tràn bộ nhớ —
   một biện pháp phòng thủ cần thiết trước log bất thường từ client.



----

.. rubric:: 6. Nhiều thread dùng chung một socket

Trong ``app-code.c``, **3 pthread** cùng gọi ``logger()`` — tất cả chia sẻ **một**
``g_sock_fd``. Điều này hợp lệ về mặt kỹ thuật, nhưng cần lưu ý:

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: ✅ An toàn ở mức kernel
      :class-card: sd-shadow-sm

      Mỗi lần ``write()`` là một **system call nguyên tử** đối với kernel — các
      byte của *một* lần gọi sẽ vào socket không bị trộn lẫn với lời gọi khác.

   .. grid-item-card:: ⚠️ Nguy cơ xen kẽ dòng
      :class-card: sd-shadow-sm

      Hai thread gọi ``write()`` gần như đồng thời → hai dòng log có thể **đảo
      thứ tự**. Dòng vẫn *nguyên vẹn*, nhưng *thứ tự* giữa các thread không được
      đảm bảo.

   .. grid-item-card:: 🧯 Giảm thiểu
      :class-card: sd-shadow-sm

      Giữ mỗi bản tin trong **một ``write()`` duy nhất** (đã làm) để tối đa hoá
      tính nguyên tử. Nếu cần thứ tự tuyệt đối → thêm ``mutex`` quanh ``logger()``.

   .. grid-item-card:: 🧩 Về lâu dài
      :class-card: sd-shadow-sm

      Với log *mức độ debug* (như dự án này), việc đảo thứ tự giữa các thread
      thường **chấp nhận được**. Đừng tối ưu sớm (premature optimization) nếu
      chưa thấy vấn đề thực tế.

.. rubric:: Tổng kết

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Chủ đề
     - Điểm cần nhớ
   * - Họ địa chỉ
     - ``AF_UNIX`` — định danh bằng **đường dẫn file**, nhanh & không tốn port
   * - Kiểu socket
     - ``SOCK_STREAM`` — luồng byte **có thứ tự, tin cậy**
   * - Client
     - ``socket()`` → ``connect()`` → ``write()``… → ``close()``
   * - Server
     - ``unlink()?`` → ``socket()`` → ``bind()`` → ``listen()`` → ``accept()`` → ``read()``
   * - Ép kiểu
     - ``sockaddr_un *`` → ``sockaddr *`` an toàn vì **phần đầu khớp bố cục**
   * - Kích thước
     - Luôn truyền ``sizeof(struct sockaddr_un)``, **không** phải ``sizeof(sockaddr)``
   * - Phân mảnh bản tin
     - Thống nhất theo **dòng (``\n``)**; phía nhận đệm cho đủ dòng rồi parse
   * - Không block
     - Client chỉ ``write()``; việc nặng (ghi file) do daemon lo ở luồng nền

.. seealso::
   * :ref:`ring-buffer-impl` — nơi dữ liệu socket được đưa vào sau khi daemon parse.
   * :ref:`atomic-operations` — cơ chế CAS đằng sau ring buffer mà socket feeder đổ vào.
   * Kiến trúc tổng thể của client/daemon xem lại ở mục *0. System Architecture*.


