.. _potential-faults:

================================
4. Potential Faults and Solution
================================

.. rst-class:: lead

   Hệ thống **Build Logger** đã chạy được, nhưng "chạy được" khác "chạy đúng".
   Bài viết này liệt kê các **lỗi tiềm ẩn** đã gặp trong quá trình phát triển —
   từ *mất log khi nhiều luồng cùng ghi* đến *không tạo được file* — cùng **nguyên
   nhân gốc** và **hướng khắc phục** cho từng trường hợp.

.. grid:: 1 2 2 3
   :gutter: 3

   .. grid-item-card:: 🔀 #1 Ghi đè thay vì drop
      :text-align: center
      :class-card: sd-shadow-sm

      Nhiều producer cùng ghi → log bị **ghi đè** lẫn nhau, mất dữ liệu thầm lặng.

   .. grid-item-card:: 🔀 #2 Thiếu slot riêng
      :text-align: center
      :class-card: sd-shadow-sm

      Producer tranh giành ô → cần cơ chế **đặt slot** cho từng luồng.

   .. grid-item-card:: 📁 #3 Không tạo được file
      :text-align: center
      :class-card: sd-shadow-sm

      Thiếu **permission** tại đường dẫn đích, dù daemon chạy trong ``root``.

   .. grid-item-card:: 🔑 #4 Quyền thực thi
      :text-align: center
      :class-card: sd-shadow-sm

      Khắc phục chắc ăn: ``chmod 775`` cho các **executable file**.

.. note::
   Mỗi lỗi dưới đây được trình bày theo cấu trúc **Triệu chứng → Nguyên nhân →
   Giải pháp**, để bạn vừa *nhận diện* nhanh khi gặp, vừa *hiểu* vì sao lại sửa
   như vậy.


----

.. rubric:: Lỗi #1 — Nhiều luồng ghi: ghi đè thay vì drop

Đây là lỗi **quan trọng nhất** và khó phát hiện nhất, vì hệ thống *vẫn chạy*,
*không crash*, chỉ đơn giản là **mất log mà không báo**.

.. rubric:: Triệu chứng

* Gửi 10 bản tin log từ **nhiều thread** khác nhau, nhưng file log cuối cùng chỉ
  ghi nhận **khoảng 5–6 dòng** (con số ví dụ).
* Không có thông báo lỗi nào — số log "bốc hơi" **một cách thầm lặng**.
* Càng nhiều thread gọi ``logger()`` đồng thời, tỉ lệ mất càng cao.

.. rubric:: Nguyên nhân gốc

Bản chất ``ring_buffer_push`` đúng ở chỗ nó **kiểm tra đầy rồi mới CAS** — nhưng
hai bước đó **không gộp làm một**. Giữa lúc *đọc* ``head``/``tail`` và lúc *CAS*
thành công, một luồng khác có thể chen ngang:

.. code-block:: c
   :caption: Khoảng trống TOCTOU — kiểm tra và chiếm ô KHÔNG nguyên tử

   do {
       curr_head = atomic_load(&rb->head, relaxed);   // (A) đọc head
       curr_tail = atomic_load(&rb->tail, acquire);
       if ((curr_head - curr_tail) >= BUFFER_SIZE)     // (B) kiểm tra đầy
           return -1;
   } while (!CAS_weak(&rb->head, &curr_head,           // (C) chiếm ô — KHÁC BƯỚC (A)!
                      curr_head + 1, acquire, relaxed));

.. code-block:: text
   :caption: Hai producer cùng "đỗ" vào một ô — write_idx trùng nhau

   Buffer gần đầy, head = 1000, còn đúng 1 ô trống:

   Thread-1: đọc head=1000 ─┐
   Thread-2: đọc head=1000 ─┤  (cả hai đọc CÙNG giá trị)
                             │
   Thread-1: CAS head 1000→1001  ✅ thành công → write_idx = 1000
   Thread-2: CAS head 1000→1001  ❌ thất bại (head đã đổi)
             ── đỌC LẠI: head=1001, tail phải theo kịp ──
             nếu consumer chưa kịp đọc → (1001 - tail) có thể vẫn < 1024
             → CAS head 1001→1002 ✅ → write_idx = 1001  (an toàn?)
                                     ▲
        Nhưng nếu kiểm tra đầy ở (B) dùng tail CŨ, thread-2 có thể
        "lọt" qua khi buffer thực ra ĐÃ đầy → ghi đè ô chưa được đọc

.. important::
   Vấn đề cốt lõi: phép **kiểm tra đầy (B)** và phép **chiếm ô (C)** không nguyên
   tử với nhau. Khi nhiều producer dồn về cùng lúc, ``head`` nhảy vọt qua ``tail``
   vượt ``BUFFER_SIZE`` trước khi consumer kịp đọc → ô bị **ghi đè** thay vì bị
   *drop* sạch như ý muốn ban đầu.

.. rubric:: Giải pháp — đặt slot riêng cho từng luồng

Thay vì để mọi luồng cùng tranh giành một biến ``head`` chung, ta **phân phát
một slot (chỉ số) duy nhất** cho mỗi producer trước khi cho phép ghi:

.. code-block:: c
   :caption: Hướng khắc phục — mỗi producer "nhảy vào đòi" một slot riêng

   // Mỗi producer đăng ký một slot TRƯỚC, rồi mới ghi vào slot đó.
   // Hai luồng không bao giờ nhận cùng một slot.
   uint32_t my_slot = atomic_fetch_add(&rb->head, 1);   // chiếm slot nguyên tử

   // Kiểm tra: slot của ta có bị "quá tay" so với vùng hợp lệ không?
   uint32_t tail = atomic_load(&rb->tail, acquire);
   if (my_slot - tail >= BUFFER_SIZE) {
       return -1;   // buffer thực sự đầy → drop đúng cách, KHÔNG ghi đè
   }

   int write_idx = my_slot & (BUFFER_SIZE - 1);
   ...ghi dữ liệu vào buffer[write_idx]...;
   committed[write_idx] = 1;   // release

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: ✅ Ưu điểm
      :class-card: sd-shadow-sm

      ``fetch_add`` **nguyên tử** — mỗi luồng nhận **một** ``my_slot`` khác biệt.
      Không còn hai producer cùng ghi một ô.

   .. grid-item-card:: ⚠️ Đánh đổi
      :class-card: sd-shadow-sm

      ``fetch_add`` làm ``head`` nhảy liên tục; cần **kiểm tra lại** sau khi chiếm
      để đảm bảo slot chưa bị consumer "vượt mặt". Chi phí nhỏ, đổi lấy tính đúng.

.. tip::
   Với hệ thống *log cấp debug* như dự án này, mất vài dòng khi quá tải thường
   **chấp nhận được**. Nhưng nếu cần *zero-loss*, hướng "slot riêng" trên là bước
   đệm; bền vững hơn nữa là cho mỗi producer một **hàng đợi (queue) riêng** rồi
   mới gộp vào buffer trung tâm.



----

.. rubric:: Lỗi #3 — Không tạo được file log (thiếu permission)

.. rubric:: Triệu chứng

* Daemon chạy, nhận log qua socket bình thường, nhưng **không thấy file log** xuất
  hiện tại đường dẫn mong muốn.
* Chương trình in ra thông báo *"can not create file log"* (trong ``worker_thread``).

.. code-block:: c
   :caption: logger-daemon.c — điểm thất bại khi fopen không được cấp quyền

   FILE *file = fopen(FILE_PATH, "a");      // FILE_PATH = /home/aelius/envDev/logger.log
   if (file != NULL) {
       fprintf(file, "%d : %s\n", item.level, item.msg);
       fclose(file);
   } else {
       printf("can not create file log\n");  // ◄── rơi vào nhánh này
   }

.. rubric:: Nguyên nhân gốc

Không có quyền ghi tại **thư mục đích**. Dù đường dẫn trông hợp lệ, kernel vẫn
từ chối ``fopen(..., "a")`` khi:

* Thư mục cha thuộc về user khác và không có quyền ghi (``w``) cho user hiện tại.
* Thư mục đang được mount **read-only**, hoặc dùng hệ thống tệp chỉ đọc.
* Trên board nhúng, phân vùng đích bị **khóa** hoặc dùng cho mục đích khác.

.. warning::
   **Hiểu lầm phổ biến:** *"daemon chạy trong ``root`` thì muốn ghi đâu cũng được"*
   là **sai**. ``root`` bỏ qua hầu hết kiểm tra quyền, nhưng **không phải tất cả**:
   thư mục *read-only*, mount đặc biệt, hoặc chính sách **SELinux/AppArmor** vẫn
   có thể chặn ``root``. Đừng cho rằng "có root là xong" — vẫn phải **xin phép**
   đúng chỗ.

.. rubric:: Giải pháp

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: 📂 Đổi sang vị trí có quyền
      :class-card: sd-shadow-sm

      Chuyển ``FILE_PATH`` tới thư mục user hiện tại **thực sự** được ghi:

      .. code-block:: bash

         # Chọn vị trí an toàn, ví dụ thư mục home của user
         FILE_PATH="/home/$USER/logger.log"
         # hoặc vùng tạm luôn ghi được
         FILE_PATH="/tmp/logger.log"

   .. grid-item-card:: 🛠️ Kiểm tra nhanh quyền
      :class-card: sd-shadow-sm

      Trước khi sửa code, xác minh thư mục có ghi được không:

      .. code-block:: bash

         touch /home/aelius/envDev/test_write && echo "OK" || echo "DENIED"
         mount | grep <partition>     # xem có ro (read-only) không

.. rubric:: Lỗi #4 — Quyền thực thi file (executable permission)

Một cách **chắc ăn** khác để tránh cả lỗi tạo file lẫn lỗi "không chạy được
daemon": đảm bảo các **executable file** có quyền phù hợp ngay từ khâu build/deploy.

.. code-block:: bash
   :caption: Chmod 775 cho executable — chủ sở hữu & nhóm ghi được, người khác chỉ chạy

   chmod 775 logger-daemon        # rwxrwxr-x
   # 7 (rwx) = chủ sở hữu: đọc + ghi + thực thi
   # 7 (rwx) = nhóm:        đọc + ghi + thực thi
   # 5 (r-x) = người khác:  đọc + thực thi (không ghi)

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: 🔢 Giải mã ``775``
      :class-card: sd-shadow-sm

      Ba chữ số cho ba nhóm: **owner / group / others**.

      .. code-block:: text

         7 = rwx (4+2+1)   đọc + ghi + thực thi
         5 = r-x (4+0+1)   đọc + thực thi

   .. grid-item-card:: 🎯 Khi nào dùng
      :class-card: sd-shadow-sm

      Khi daemon do **cùng một user/nhóm** vận hành, ``775`` vừa đảm bảo chạy được,
      vừa cho phép nhóm cập nhật — không mở toang quyền ghi cho *người ngoài*.

.. tip::
   Nếu deploy qua **systemd**, cân nhắc để ``User=`` trong ``logger.service`` trỏ
   đúng user sở hữu thư mục đích. Thiết lập quyền *khớp* giữa user chạy daemon và
   user sở hữu đường dẫn log sẽ tránh phần lớn lỗi permission ngay từ đầu.



----

.. rubric:: Bảng tổng hợp — Lỗi & Khắc phục

.. list-table:: Tra nhanh toàn bộ lỗi tiềm ẩn
   :header-rows: 1
   :widths: 8 30 32 30

   * - #
     - Triệu chứng
     - Nguyên nhân gốc
     - Giải pháp
   * - 1
     - Mất log khi nhiều thread ghi (10 log → ~5–6)
     - Kiểm tra đầy & chiếm ô **không** nguyên tử (TOCTOU)
     - **Đặt slot riêng** cho từng luồng (``fetch_add``)
   * - 2
     - Hai producer ghi đè cùng một ô
     - Cùng tranh giành biến ``head`` dùng chung
     - Phân phát slot duy nhất + kiểm tra lại sau chiếm
   * - 3
     - *"can not create file log"*
     - Thiếu quyền ghi tại thư mục đích / mount ro
     - Đổi sang vị trí có quyền (home/tmp), kiểm tra ``mount``
   * - 4
     - Daemon không chạy / không deploy được
     - Executable thiếu quyền thực thi
     - ``chmod 775`` cho executable; khớp ``User=`` systemd

.. rubric:: Bài học rút ra

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: 🧠 Lock-free ≠ "không cần suy nghĩ"
      :class-card: sd-shadow-sm

      Atomic giúp *từng thao tác* an toàn, nhưng **chuỗi** thao tác (kiểm tra →
      chiếm) vẫn có thể có lỗ hổng. Phải phân tích cả *quy trình*, không chỉ từng
      dòng. Xem lại :ref:`atomic-operations`.

   .. grid-item-card:: 📁 Quyền là chuyện của môi trường
      :class-card: sd-shadow-sm

      Lỗi permission không phải lỗi code — mà là lỗi **triển khai**. Kiểm tra sớm
      quyền thư mục đích và user chạy daemon sẽ tiết kiệm hàng giờ debug.

.. seealso::
   * :ref:`ring-buffer-impl` — cấu trúc dữ liệu mà Lỗi #1/#2 xảy ra bên trong.
   * :ref:`atomic-operations` — nền tảng CAS/atomic để hiểu nguyên nhân gốc.
   * :ref:`socket-commu-guide` — đường đi của dữ liệu trước khi vào ring buffer.


