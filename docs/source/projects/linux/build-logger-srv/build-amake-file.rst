.. _build-amake-file:

==========================================
Viết Makefile để biên dịch chương trình
==========================================

.. rst-class:: lead

   **Makefile** là "công thức" cho lệnh ``make`` biết cách biên dịch chương trình:
   file nào cần build, phụ thuộc vào nhau ra sao, dùng compiler và cờ (flags) gì.
   Bài viết này lột xác **Makefile thật** của dự án *Build Logger* để bạn vừa hiểu
   **cú pháp**, vừa nắm **ý nghĩa từng ký tự** và **cấu trúc tổng thể** của nó.

.. grid:: 1 2 2 4
   :gutter: 3

   .. grid-item-card:: 🧾 Biến (Variables)
      :text-align: center
      :class-card: sd-shadow-sm

      ``CC``, ``CFLAGS``, ``*_SRCS``, ``*_OBJS`` — đặt một lần, dùng nhiều nơi.

   .. grid-item-card:: 🎯 Target & Phụ thuộc
      :text-align: center
      :class-card: sd-shadow-sm

      ``target: deps`` — mô tả *làm gì* và *cần gì trước đã*.

   .. grid-item-card:: 🔡 Biến tự động
      :text-align: center
      :class-card: sd-shadow-sm

      ``$@`` ``$^`` ``$<`` — rút gọn cho target và dependencies.

   .. grid-item-card:: 🧩 Pattern rule
      :text-align: center
      :class-card: sd-shadow-sm

      ``%.o: %.c`` — một quy tắc cho *mọi* file ``.c`` → ``.o``.

----

.. rubric:: 1. Makefile là gì & vì sao cần?

Khi project có nhiều file ``.c``, gõ tay ``gcc`` cho từng file rất chán và dễ sai.
``make`` đọc **Makefile**, tự tính xem file nào **đã đổi** (cần build lại) và file
nào **không cần** (bỏ qua) — gọi là **incremental build**.

.. code-block:: text
   :caption: Không Makefile vs. có Makefile

   ❌ Thủ công (mỗi lần đổi 1 file là compile lại tay):
      gcc -Wall -pthread -c app-code.c -o app-code.o
      gcc -Wall -pthread -c logger-srv.c -o logger-srv.o
      gcc -Wall -pthread -o app app-code.o logger-srv.o

   ✅ Có Makefile (một lệnh duy nhất):
      $ make          # tự build, chỉ build lại phần đã đổi
      $ make clean    # dọn sạch file tạm

.. note::
   Trong dự án này, mỗi thư mục (``application-code/`` và ``daemon-code/``) có
   **một Makefile riêng** — vì chúng là **hai chương trình độc lập**: ``app``
   (client) và ``logger_daemon`` (server).

.. seealso::
   * :ref:`architecture` — vị trí ``application-code/`` và ``daemon-code/`` trong cây dự án.
   * :ref:`ring-buffer-impl` — ``ring_buffer.c`` được build cùng daemon qua Makefile.


----

.. rubric:: 2. Giải phẫu Makefile của Daemon

Đây là Makefile **thật** trong ``daemon-code/Makefile``. Ta đọc từng khối:

.. code-block:: make
   :caption: daemon-code/Makefile — build logger_daemon từ 2 file nguồn

   # Environment declaration first
   CC = gcc
   CFLAGS = -std=gnu11 -Wall -Wextra -pthread -I.

   # next, output file name are specified. <name>.o
   DAEMON_SRCS = logger-daemon.c ring_buffer.c
   DAEMON_OBJS = $(DAEMON_SRCS:.c=.o)

   # the output daemon executable name
   DAEMON_TARGET = logger_daemon

   all: $(DAEMON_TARGET)

   # daemon compile,
   # "$@" represent for target, is left-side ":"
   # "$^" represent for depen.., is right-side ":"
   $(DAEMON_TARGET): $(DAEMON_OBJS)
   	$(CC) $(CFLAGS) -o $@ $^

   # compile .c to .o file
   %.o: %.c
   	$(CC) $(CFLAGS) -c $< -o $@

   # clean up
   clean:
   	rm -f *.o $(DAEMON_TARGET)

   .PHONY: all clean

.. rubric:: Khối 1 — Khai báo biến (Variables)

.. list-table:: Các biến trong Makefile và ý nghĩa
   :header-rows: 1
   :widths: 20 40 40

   * - Biến
     - Giá trị
     - Ý nghĩa
   * - ``CC``
     - ``gcc``
     - Trình biên dịch C cần dùng
   * - ``CFLAGS``
     - ``-std=gnu11 -Wall -Wextra -pthread -I.``
     - Cờ biên dịch (xem bảng dưới)
   * - ``DAEMON_SRCS``
     - ``logger-daemon.c ring_buffer.c``
     - Danh sách file nguồn ``.c``
   * - ``DAEMON_OBJS``
     - ``logger-daemon.o ring_buffer.o``
     - Danh sách file object — *sinh tự động* từ ``SRCS``
   * - ``DAEMON_TARGET``
     - ``logger_daemon``
     - Tên file thực thi đầu ra

.. tip::
   Dòng ``DAEMON_OBJS = $(DAEMON_SRCS:.c=.o)`` dùng **substitution reference**:
   nó thay mọi hậu tố ``.c`` bằng ``.o``. Thêm/bớt file ``.c``? Chỉ cần sửa
   ``DAEMON_SRCS`` — ``OBJS`` tự cập nhật, khỏi sửa hai chỗ.

.. rubric:: Khối 2 — Cờ biên dịch (CFLAGS)

.. list-table:: Giải nghĩa từng cờ trong CFLAGS
   :header-rows: 1
   :widths: 22 78

   * - Cờ
     - Tác dụng
   * - ``-std=gnu11``
     - Dùng chuẩn **C11** (bắt buộc cho ``<stdatomic.h>``) + phần mở rộng GNU
   * - ``-Wall``
     - Bật *hầu hết* cảnh báo (warning) phổ biến
   * - ``-Wextra``
     - Bật thêm cảnh báo "khó tính" hơn nữa
   * - ``-pthread``
     - Liên kết thư viện **pthread** (đa luồng) — cần cho worker/daemon
   * - ``-I.``
     - Thêm thư mục hiện tại vào đường dẫn tìm **header** (``.h``)

.. important::
   ``-std=gnu11`` là **bắt buộc** ở daemon, vì ``ring_buffer.h`` dùng
   ``<stdatomic.h>`` — tính năng **C11** (xem :ref:`atomic-operations`). Thiếu cờ
   này, compiler sẽ báo lỗi không nhận diện được ``_Atomic``.



----

.. rubric:: 3. Rule (quy tắc), ký tự đặc biệt & pattern

Một **rule** (quy tắc) trong Makefile có dạng tổng quát:

.. code-block:: text
   :caption: Cấu trúc một rule — target : prerequisites ⇥ recipe

   target … : prerequisites …      ◄── dòng "quy tắc" (bắt buộc bẳng TAB trước recipe)
   ⎋⎋⎋⎋⎋⎋ recipe …                  ⇥ recipe = các lệnh shell cần chạy
   ⎋⎋⎋⎋⎋⎋ recipe …

   * target         : file (hoặc hành động) cần tạo ra
   * prerequisites  : file cần có TRƯỚC khi chạy recipe
   * recipe         : lệnh shell (mỗi dòng phải bắt đầu bằng ký tự TAB)

.. warning::
   **Cái bẫy số 1 của Makefile:** mỗi dòng lệnh trong recipe **phải** bắt đầu bằng
   ký tự **TAB** (``\t``), *không* phải dấu cách. Dùng space → make báo lỗi
   khó hiểu *"missing separator"*. Đây là lỗi gặp nhất khi mới học.

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: 🎯 Rule liên kết (link)
      :class-card: sd-shadow-sm

      Gom các ``.o`` thành file thực thi:

      .. code-block:: make

         $(DAEMON_TARGET): $(DAEMON_OBJS)
         	$(CC) $(CFLAGS) -o $@ $^
         # ⇧ logger_daemon : *.o  →  gcc … -o logger_daemon *.o

   .. grid-item-card:: 🧩 Pattern rule (biên dịch)
      :class-card: sd-shadow-sm

      Một quy tắc cho *mọi* cặp ``.c`` → ``.o``:

      .. code-block:: make

         %.o: %.c
         	$(CC) $(CFLAGS) -c $< -o $@
         # % là "ký tự đại diện" (wildcard) khớp mọi tên file

.. rubric:: Bảng tra — biến tự động (automatic variables)

Đây là các ký hiệu ``$…`` mà make tự điền, giúp recipe ngắn gọn:

.. list-table:: Các biến tự động và ý nghĩa
   :header-rows: 1
   :widths: 14 40 46

   * - Ký hiệu
     - Ý nghĩa
     - Ví dụ trong dự án
   * - ``$@``
     - **Target** — vế trái dấu ``:``
     - ``logger_daemon`` (hoặc ``x.o``)
   * - ``$^``
     - **Tất cả prerequisites** — vế phải ``:``
     - ``logger-daemon.o ring_buffer.o``
   * - ``$<``
     - Prerequisite **đầu tiên**
     - file ``.c`` tương ứng trong ``%.o: %.c``
   * - ``$?``
     - Các prerequisite **mới hơn** target
     - chỉ file vừa đổi (incremental)
   * - ``$*``
     - **Stem** — phần ``%`` khớp được
     - ``ring_buffer`` trong pattern

.. code-block:: text
   :caption: Minh hoạ $@ $^ $< trong rule link

   $(DAEMON_TARGET): $(DAEMON_OBJS)
   └────── $@ ─────┘   └──── $^ ─────────┘
   logger_daemon      logger-daemon.o ring_buffer.o

   %.o: %.c
   └$*┘   └$*┘          $< = file .c,  $@ = file .o
   %.o    %.c

.. tip::
   Nhờ ``$^``, dù project có 2 hay 20 file ``.c``, recipe link vẫn chỉ là một dòng
   ``-o $@ $^`` — ``make`` tự nở danh sách object. Đây là lý do Makefile *co giãn*
   tốt khi thêm file mới.



----

.. rubric:: 4. ``.PHONY`` & target ``clean``

.. code-block:: make
   :caption: Target dọn dẹp và khai báo .PHONY

   # clean up
   clean:
   	rm -f *.o $(DAEMON_TARGET)

   .PHONY: all clean

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: 🧹 ``clean`` — dọn dẹp
      :class-card: sd-shadow-sm

      Xoá mọi file ``.o`` và file thực thi. ``-f`` (force) để ``rm`` *không* báo
      lỗi khi file không tồn tại — nên ``make clean`` chạy lại nhiều lần vẫn an toàn.

   .. grid-item-card:: 🏷️ ``.PHONY`` — "đừng coi là file"
      :class-card: sd-shadow-sm

      ``all`` và ``clean`` là *hành động*, **không** tạo ra file cùng tên. ``.PHONY``
      báo cho make biết điều này để nó **luôn chạy**, kể cả khi lỡ có file tên
      ``clean`` trong thư mục.

.. note::
   Nếu **không** khai báo ``.PHONY: clean`` mà thư mục lại có một file tên ``clean``,
   make sẽ nghĩ "target ``clean`` đã được tạo ra rồi" → **bỏ qua**, không chạy dọn.
   Đây là lỗi kinh điển khiến ``make clean`` "không ăn" một cách bí ẩn.

.. rubric:: 5. Makefile của App (client) — khác gì daemon?

Hai Makefile gần như **giống nhau**, chỉ khác **biến và cờ**:

.. code-block:: make
   :caption: application-code/Makefile — build app từ 2 file nguồn

   CC = gcc
   CFLAGS = -Wall -Wextra -pthread -I.        # ◄── KHÔNG có -std=gnu11

   APP_SRC = app-code.c logger-srv.c
   APP_OBJS = $(APP_SRC:.c=.o)
   APP_TARGET = app

   all: $(APP_TARGET)
   $(APP_TARGET): $(APP_OBJS)
   	$(CC) $(CFLAGS) -o $@ $^
   %.o: %.c
   	$(CC) $(CFLAGS) -c $< -o $@
   clean:
   	rm -f *.o $(APP_TARGET)
   .PHONY: all clean

.. list-table:: So sánh App Makefile vs. Daemon Makefile
   :header-rows: 1
   :widths: 24 38 38

   * - Khía cạnh
     - ``application-code/`` (App)
     - ``daemon-code/`` (Daemon)
   * - Target
     - ``app``
     - ``logger_daemon``
   * - File nguồn
     - ``app-code.c logger-srv.c``
     - ``logger-daemon.c ring_buffer.c``
   * - ``-std=gnu11``
     - ❌ không có (mặc định C99)
     - ✅ có (cần cho ``stdatomic``)
   * - Vai trò
     - Client gửi log (xem :ref:`socket-commu-guide`)
     - Server nhận & ghi log

.. important::
   **Vì sao daemon cần ``-std=gnu11`` mà app thì không?** Vì chỉ daemon biên dịch
   ``ring_buffer.c`` — nơi dùng ``<stdatomic.h>`` (C11). App chỉ gọi API socket nên
   chuẩn mặc định là đủ. Mỗi Makefile được **tinh chỉnh đúng theo nhu cầu** của
   chương trình nó build.

.. rubric:: 6. Luồng build & bản đồ phụ thuộc

.. code-block:: text
   :caption: make suy diễn chuỗi phụ thuộc rồi build từ dưới lên

   $ make
     │
     ▼  đọc "all" → cần logger_daemon
   all ─▶ logger_daemon
              │  cần 2 file .o
              ├──▶ logger-daemon.o ──▶ logger-daemon.c   (dùng %.o: %.c)
              └──▶ ring_buffer.o    ──▶ ring_buffer.c     (dùng %.o: %.c)
     │
     ▼  mỗi .o chỉ build lại nếu .o cũ hơn .c (incremental!)
   gcc -std=gnu11 … -c logger-daemon.c -o logger-daemon.o
   gcc -std=gnu11 … -c ring_buffer.c    -o ring_buffer.o
     │
     ▼  gộp .o thành executable
   gcc … -o logger_daemon logger-daemon.o ring_buffer.o   ✅

.. tip::
   ``make`` **chỉ build lại file đã đổi**. Sửa ``ring_buffer.c`` → chỉ ``ring_buffer.o``
   và bước link chạy lại; ``logger-daemon.o`` giữ nguyên. Với project lớn, điều này
   tiết kiệm *rất nhiều* thời gian so với compile lại tất cả.



----

.. rubric:: 7. Bản đồ CMake (tương đương)

Dự án cũng cung cấp ``CMakeLists.txt`` cho mỗi thư mục — làm **cùng việc** như
Makefile nhưng cú pháp khác. Đối chiếu nhanh:

.. code-block:: cmake
   :caption: daemon-code/CMakeLists.txt — tương đương Makefile phía trên

   cmake_minimum_required(VERSION 3.10)
   project(LoggerDaemon C)

   set(CMAKE_C_STANDARD 11)              # ~ tương đương -std=gnu11
   set(CMAKE_C_STANDARD_REQUIRED ON)
   add_compile_options(-Wall -Wextra)    # ~ CFLAGS

   find_package(Threads REQUIRED)        # ~ -pthread

   set(DAEMON_SRCS                       # ~ DAEMON_SRCS
       logger-daemon.c
       ring_buffer.c
   )
   add_executable(logger_daemon ${DAEMON_SRCS})         # ~ target link
   target_link_libraries(logger_daemon PRIVATE Threads::Threads)

.. list-table:: Makefile ↔ CMake — thuật ngữ tương đương
   :header-rows: 1
   :widths: 34 33 33

   * - Khái niệm
     - Makefile
     - CMake
   * - Trình biên dịch
     - ``CC = gcc``
     - ``project(... C)``
   * - Chuẩn C
     - ``-std=gnu11``
     - ``CMAKE_C_STANDARD 11``
   * - Cờ cảnh báo
     - ``-Wall -Wextra``
     - ``add_compile_options(...)``
   * - Thư viện luồng
     - ``-pthread``
     - ``find_package(Threads)``
   * - File nguồn
     - ``DAEMON_SRCS``
     - ``set(DAEMON_SRCS …)``
   * - Tạo executable
     - ``target: objs`` + link
     - ``add_executable(...)``

.. note::
   **Makefile** đơn giản, minh bạch, phù hợp project nhỏ/board nhúng. **CMake**
   mạnh hơn khi build **đa nền tảng**, tự sinh Makefile/IDE project. Ở đây cả hai
   cùng tồn tại — Makefile để build nhanh tại chỗ, CMake để tích hợp rộng hơn.

.. rubric:: 8. Best practices & cái bẫy

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: ⤶ TAB, không phải space
      :class-card: sd-shadow-sm

      Recipe **bắt buộc** bằng TAB. Space → lỗi *"missing separator"*. Nhiều
      editor tự chuyển TAB→space, hãy tắt "expand tab" khi sửa Makefile.

   .. grid-item-card:: 🏷️ Luôn khai ``.PHONY``
      :class-card: sd-shadow-sm

      Với mọi target *không* tạo file (``all``, ``clean``, ``run``). Tránh trường
      hợp make "tưởng" target đã xong và bỏ qua.

   .. grid-item-card:: 🧮 Một nguồn duy nhất sự thật
      :class-card: sd-shadow-sm

      Khai báo file nguồn **một lần** trong ``*_SRCS`` rồi sinh ``*_OBJS`` bằng
      substitution — đừng liệt kê hai chỗ, dễ lệch.

   .. grid-item-card:: 🧹 Luôn có ``clean``
      :class-card: sd-shadow-sm

      Giúp build lại "từ zero" khi nghi ngờ file ``.o`` cũ gây lỗi lạ:
      ``make clean && make``.

.. rubric:: Tóm tắt

.. list-table::
   :header-rows: 1
   :widths: 22 78

   * - Khía cạnh
     - Điểm cần nhớ
   * - Mục đích
     - Tự động hoá biên dịch + **incremental build** (chỉ build file đã đổi)
   * - Cấu trúc
     - ``target : prerequisites`` + recipe (bắt đầu bằng **TAB**)
   * - Biến
     - ``CC``, ``CFLAGS``, ``*_SRCS`` → ``*_OBJS`` (substitution ``:.c=.o``)
   * - Biến tự động
     - ``$@`` target, ``$^`` mọi deps, ``$<`` dep đầu, ``$*`` stem
   * - Pattern rule
     - ``%.o: %.c`` — một quy tắc cho mọi file ``.c``
   * - Phân biệt
     - Daemon cần ``-std=gnu11`` (stdatomic); App dùng chuẩn mặc định
   * - Dọn dẹp
     - ``clean`` + ``.PHONY: all clean`` để luôn chạy đúng
   * - CMake
     - Bản tương đương đa nền tảng, cùng cấu hình với Makefile

.. seealso::
   * :ref:`architecture` — cây thư mục chứa các Makefile/CMakeLists này.
   * :ref:`atomic-operations` — lý do daemon bắt buộc cờ ``-std=gnu11``.
   * :ref:`ring-buffer-impl` — ``ring_buffer.c`` được build cùng daemon.


