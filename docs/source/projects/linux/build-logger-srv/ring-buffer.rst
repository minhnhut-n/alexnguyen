.. _ring-buffer-impl:


=============================
1. Ring Buffer Implementation
=============================

.. rst-class:: lead

   **Ring Buffer** (bộ đệm vòng) là *"băng chuyền"* nối giữa luồng **Producer**
   (nhận log) và luồng **Consumer** (ghi ra file). Nó cho phép hai bên giao
   tiếp **không cần khóa (lock-free)**, **không cấp phát bộ nhớ động**, với
   chi phí **O(1)** cho mỗi lần ghi / đọc.

.. grid:: 1 2 2 4
   :gutter: 3

   .. grid-item-card:: 🧊 Không ``malloc``
      :text-align: center
      :class-card: sd-shadow-sm

      Mảng cố định ``buffer[1024]`` cấp phát một lần —
      hot-path *không bao giờ* gọi ``malloc``/``free``.

   .. grid-item-card:: ⚡ Lock-free
      :text-align: center
      :class-card: sd-shadow-sm

      Đồng bộ bằng **CAS** (compare-and-swap), không dùng mutex,
      producer/consumer *không bao giờ bị block*.

   .. grid-item-card:: ♻️ Chi phí O(1)
      :text-align: center
      :class-card: sd-shadow-sm

      Ghi và đọc đều *thời gian hằng số*, bất kể buffer có bao nhiêu phần tử.

   .. grid-item-card:: 🎯 Bộ nhớ xác định
      :text-align: center
      :class-card: sd-shadow-sm

      Footprint cố định — quan trọng cho daemon chạy dài hạn trên board nhúng.

----

.. rubric:: 1. Ring Buffer giải quyết bài toán gì?

Logger daemon nhận log từ *nhiều nguồn* nhưng chỉ ghi file *một luồng*. Nếu mỗi
lần có log là ghi ngay xuống đĩa, tiến trình ứng dụng sẽ bị **nghẽn (block)** mỗi
khi đĩa chậm. Ring buffer gỡ vai cho hai bên:

.. code-block:: text
   :caption: Producer và Consumer tách rời — buffer là vùng đệm chung

   Ứng dụng / Socket thread                  File I/O worker
        (Producer)                               (Consumer)
             │                                        ▲
             ▼                                        │
        ┌─────────┐   push()   ┌──────────────────┐   │  pop()
        │ log_msg ├───────────▶│  ring buffer     ├───┘
        └─────────┘   (O(1))   │  head / tail     │   (O(1))
             ▲                 │  [1024 ô]        │
             │                 └──────────────────┘
        không bị block                │ đầy thì drop
        kể cả khi đĩa chậm            ▼
                                bảo toàn oldest

.. note::
   Việc **định dạng** log (thêm timestamp, chọn mức ưu tiên, ghép chuỗi) do tầng
   producer (``log_message()``) thực hiện *trước khi* đẩy vào ring buffer. Ring
   buffer chỉ đảm nhận việc **điều phối dòng dữ liệu** giữa hai luồng.

----

.. rubric:: 2. Cấu trúc dữ liệu — hai ``struct`` cốt lõi

Toàn bộ trạng thái nằm gọn trong ``ring_buffer.h``:

.. code-block:: c
   :caption: ring_buffer.h — mô tả bộ đệm và một ô dữ liệu

   #define BUFFER_SIZE      1024   // lũy thừa của 2
   #define LOG_MESSAGE_SIZE 256

   typedef enum { LOG_INFO, LOG_DEBUG, LOG_ERROR, LOG_FORCE } log_type_t;

   typedef struct {
       log_type_t  level;      // mức log
       char        msg[256];   // nội dung log
       atomic_int  committed;  // cờ "ô đã ghi xong"
   } log_item_t;

   typedef struct {
       _Atomic(uint32_t) head;            // con trỏ GHI  (producer)
       _Atomic(uint32_t) tail;            // con trỏ ĐỌC  (consumer)
       log_item_t buffer[BUFFER_SIZE];    // mảng ô cố định
   } log_ring_buffer_t;

.. list-table:: Vai trò từng thành phần
   :header-rows: 1
   :widths: 26 44 30

   * - Thành phần
     - Ý nghĩa
     - Đặc điểm
   * - ``_Atomic(uint32_t) head``
     - Vị trí producer sẽ ghi tiếp
     - Chỉ tăng, không reset
   * - ``_Atomic(uint32_t) tail``
     - Vị trí consumer sẽ đọc tiếp
     - Chỉ tăng, không reset
   * - ``log_item_t buffer[1024]``
     - Vùng chứa dữ liệu
     - Kích thước = lũy thừa của 2
   * - ``log_type_t level``
     - Mức log (INFO/DEBUG/ERROR/FORCE)
     - Định danh độ ưu tiên
   * - ``char msg[256]``
     - Nội dung thông điệp
     - Cố định ``LOG_MESSAGE_SIZE``
   * - ``atomic_int committed``
     - Cờ đồng bộ dữ liệu của ô
     - *then chốt* — xem mục 7

----

.. rubric:: 3. Bộ đếm đơn điệu & phép mask "lũy thừa 2"

``head`` và ``tail`` là các bộ đếm **chỉ tăng dần**, không bao giờ đếm ngược.
Vị trí ô thật trong mảng được tính bằng **phép AND bit**:

.. code-block:: text
   :caption: ``index = counter & (BUFFER_SIZE - 1)`` thay cho phép chia ``%``

Cái này sử dụng được là do (1024-1) = 0x3FF, do đó nó có thể được dùng làm một mặt
nạ để rotate head và tail. Để áp dụng cách tương tự cho nhiều ứng dụng khác với 
cách dev bitwise này thì nên chứ ý.

.. tip::
   Mask bit (``& 1023``) chỉ tốn **1 chu kỳ CPU**, nhanh hơn nhiều so với phép
   chia lấy dư (``% 1024``). Đây là lý do ``BUFFER_SIZE`` *bắt buộc* là lũy thừa
   của 2.

.. dropdown:: Vì sao ``head - tail`` không sợ tràn số?
   :color: primary

   ``head``/``tail`` là ``uint32_t``. Phép trừ giữa hai số không dấu "quấn vòng"
   theo **modulo 2³²**, nên khoảng cách vẫn đúng ngay cả khi bộ đếm tràn:

   .. code-block:: text

      head = 0x0000_0005
      tail = 0xFFFF_FFF0
      ─────────────────────
      (head - tail) mod 2³² = 0x0000_0015 = 21   → đang chứa 21 phần tử

   Nhờ vậy ta **không cần reset** ``head``/``tail`` về 0 — tránh được toàn bộ
   nhóm lỗi "reset giữa chừng" rất khó debug.

----

.. rubric:: 4. Đầy và rỗng nhận biết thế nào?

Hai điều kiện biên được kiểm tra *trước* khi thực hiện CAS:

.. list-table:: Điều kiện biên của ring buffer
   :header-rows: 1
   :widths: 18 42 40

   * - Trạng thái
     - Điều kiện
     - Hành động
   * - Đầy (full)
     - ``(head - tail) >= BUFFER_SIZE``
     - ``push`` trả ``-1`` → **drop** log mới
   * - Rỗng (empty)
     - ``head == tail``
     - ``pop`` trả ``-1`` → chờ, nhả CPU

.. important::
   Chính nhờ **khoảng cách** ``head - tail`` (không so giá trị tuyệt đối) mà vòng
   đệm **quấn số** vẫn tính đúng mức đầy. Đây là điểm mấu chốt của mọi ring buffer
   lock-free.



----

.. rubric:: 5. Producer — ``ring_buffer_push()``

Đây là đường **nóng (hot-path)** của luồng nhận log. Nhiệm vụ: chiếm một ô trống,
đẩy dữ liệu vào, rồi *mở cờ* cho phép consumer đọc.

.. code-block:: c
   :caption: ring_buffer.c — vòng lặp CAS để giữ chỗ (reserve) trước khi ghi

   int ring_buffer_push(log_ring_buffer_t *rb, int level, const char *msg) {
       uint32_t curr_head, curr_tail;

       // (1) Vòng lặp CAS: chỉ giành 1 ô trống cho producer này
       do {
           curr_head = atomic_load_explicit(&rb->head, memory_order_relaxed);
           curr_tail = atomic_load_explicit(&rb->tail, memory_order_acquire);

           if ((curr_head - curr_tail) >= BUFFER_SIZE) {
               return -1;            // buffer ĐẦY → drop log mới
           }
       } while (!atomic_compare_exchange_weak_explicit(
                      &rb->head, &curr_head, curr_head + 1,
                      memory_order_acquire, memory_order_relaxed));

       // (2) Đã giữ chỗ → ghi dữ liệu vào ô
       int write_idx = curr_head & (BUFFER_SIZE - 1);
       rb->buffer[write_idx].level = level;
       snprintf(rb->buffer[write_idx].msg, LOG_MESSAGE_SIZE, "%s", msg);

       // (3) Mở cờ release: consumer chỉ được đọc khi cờ này = 1
       atomic_store_explicit(&rb->buffer[write_idx].committed, 1,
                             memory_order_release);
       return 0;
   }

.. grid:: 1
   :gutter: 3

   .. grid-item-card:: 🔑 Ý tưởng "giữ chỗ trước — ghi sau"
      :class-card: sd-shadow-sm

      ``head`` được **tăng bằng CAS** để *giành quyền sở hữu* một ô. Chỉ sau khi
      giữ chỗ thành công, producer mới ghi dữ liệu. Việc tách "giữ chỗ" và "ghi"
      ra hai bước giúp **nhiều producer** không đè lên cùng một ô.

      .. code-block:: text
         :caption: Mô hình tuần tự: đọc → so khoảng cách → CAS → ghi → mở cờ

         ┌─ đọc head (relaxed), đọc tail (acquire) ─────────────┐
         │                                                      │
         │   head - tail >= 1024 ?  ──yes──▶  return -1 (drop) │
         │          │ no                                        │
         │          ▼                                           │
         │   CAS(head, curr → curr+1) ──fail──▶ quay lại đầu   |
         │          │ success                                   │
         └──────────▼───────────────────────────────────────────┘
                    ▼
            write_idx = head & 1023
            buffer[write_idx].{level,msg} = dữ liệu
            committed[write_idx] = 1   (release)
                    ▼
                return 0  ✓

.. note::
   **Vì sao lại "drop" khi đầy?** Producer (nhận log) **không bao giờ được block**
   tiến trình ứng dụng. Nếu đĩa chậm mà consumer chưa kịp đọc, ta chấp nhận mất
   log *mới nhất* để bảo vệ log *cũ hơn* đã có trong hệ thống, đồng thời giữ cho
   đường gọi log luôn nhanh và không nghẽn.



----

.. rubric:: 6. Consumer — ``ring_buffer_pop()``

Đây là đường của luồng ghi file (background worker). Nhiệm vụ: chờ có dữ liệu,
đọc ô hợp lệ, rồi *hạ cờ* để ô sẵn sàng cho vòng ghi kế tiếp.

.. code-block:: c
   :caption: ring_buffer.c — CAS trên tail + kiểm tra cờ committed trước khi đọc

   int ring_buffer_pop(log_ring_buffer_t *rb, log_item_t *item) {
       uint32_t curr_head, curr_tail;

       do {
           curr_tail = atomic_load_explicit(&rb->tail, memory_order_relaxed);
           curr_head = atomic_load_explicit(&rb->head, memory_order_acquire);

           if (curr_head == curr_tail) {
               return -1;                 // buffer RỖNG → nhả CPU
           }

           int read_idx = curr_tail & (BUFFER_SIZE - 1);
           if (atomic_load_explicit(&rb->buffer[read_idx].committed,
                                    memory_order_acquire) == 0) {
               return -1;                 // ô đang được ghi dở → đợi
           }
       } while (!atomic_compare_exchange_weak_explicit(
                      &rb->tail, &curr_tail, curr_tail + 1,
                      memory_order_acquire, memory_order_relaxed));

       // Đã giữ chỗ đọc → sao chép dữ liệu ra
       int read_idx = curr_tail & (BUFFER_SIZE - 1);
       *item = rb->buffer[read_idx];

       // Hạ cờ release: ô này sẵn sàng cho lần ghi kế tiếp
       atomic_store_explicit(&rb->buffer[read_idx].committed, 0,
                             memory_order_release);
       return 0;
   }

.. grid:: 1
   :gutter: 3

   .. grid-item-card:: 🚦 Cờ ``committed`` — ranh giới "đọc được"
      :class-card: sd-shadow-sm

      Trước khi đọc, consumer **bắt buộc** kiểm tra ``committed == 1``. Nếu cờ
      chưa bật, nghĩa là producer đã giữ chỗ nhưng **chưa ghi xong dữ liệu** —
      consumer phải rút lui (``return -1``) thay vì đọc dữ liệu rác.

      .. code-block:: text
         :caption: Vòng đời một ô nhớ qua các trạng thái của cờ committed

         ┌──────────────┐   push giữ chỗ   ┌──────────────┐  ghi xong  ┌──────────────┐
         │ committed=0  │ ───────────────▶ │ committed=?  │ ─────────▶ │ committed=1 │
         │  (rảnh)      │  (đang chiếm)    │ (đang ghi...)│  (release) │  (đọc được)  │
         └──────────────┘                  └──────────────┘            └──────┬───────┘
              ▲                                                               │
              │                        pop đọc xong, hạ cờ (release)          │
              └────────────────────────────────────────────────────────────────┘

.. warning::
   Đừng bỏ qua bước ``committed = 0`` sau khi đọc. Nếu quên hạ cờ, producer ở
   **vòng quay kế tiếp** vẫn thấy ô "đang bận" và logic đồng bộ sẽ hỏng. Nhớ:
   *giữ chỗ → ghi → mở cờ* ở producer khớp với *kiểm tra cờ → đọc → hạ cờ* ở
   consumer.



----

.. rubric:: 7. CAS & Thứ tự bộ nhớ (memory ordering)

Đây là *trái tim* giúp ring buffer chạy đúng mà **không cần khóa**. Chi tiết sâu
hơn được trình bày riêng ở mục :ref:`atomic-opts <atomic-operations>`, ở đây ta
chỉ tập trung vào **vai trò của từng memory order** trong ngữ cảnh buffer.

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: ``memory_order_acquire``
      :class-card: sd-shadow-sm

      Dùng khi **đọc** ``tail``/``head`` của đối phương và khi kiểm tra cờ
      ``committed``. Bảo đảm mọi thao tác *sau* lần đọc đều nhìn thấy dữ liệu đã
      được producer công bố (publish) — không đọc phải dữ liệu cũ/rác.

   .. grid-item-card:: ``memory_order_release``
      :class-card: sd-shadow-sm

      Dùng khi **mở/đóng cờ** ``committed`` và khi ``store`` dữ liệu. Bảo đảm mọi
      thao tác *trước* đó (ghi ``level``, ``msg``) hoàn tất và *nhìn thấy được*
      trước khi cờ đổi trạng thái.

   .. grid-item-card:: ``memory_order_relaxed``
      :class-card: sd-shadow-sm

      Dùng khi đọc ``head``/``tail`` *của chính mình* và khi CAS thất bại. Chỉ
      cần đảm bảo tính *nguyên tử (atomic)*, không cần rào chắn thứ tự — tiết kiệm
      chi phí trên kiến trúc yếu (ARM/RISC-V).

   .. grid-item-card:: ``compare_exchange_weak``
      :class-card: sd-shadow-sm

      "So giá trị hiện tại == giá trị tôi đọc? Nếu đúng thì ghi giá trị mới."
      Bản **weak** có thể thất bại giả (spurious) nên luôn đặt trong vòng
      ``do…while`` thử lại.

.. admonition:: Quy tắc vàng: cặp release–acquire

   :class: tip

   Mọi lần *truyền dữ liệu* qua buffer đều là một cặp:

   * **Producer** ghi ``msg`` → ``store committed = 1`` (**release**).
   * **Consumer** ``load committed`` (**acquire**) → thấy ``1`` → đọc ``msg``.

   Cặp release–acquire này tạo *ràng buộc happens-before*, đảm bảo consumer đọc
   được **nguyên văn** dữ liệu producer đã ghi, không bị tái thứ tự (reorder) bởi
   CPU hoặc trình biên dịch.

.. code-block:: text
   :caption: Bức tranh tổng thể — một log đi qua ring buffer như thế nào

   PRODUCER (log_message)                         RING BUFFER              CONSUMER (file worker)
   ─────────────────────                         ───────────              ─────────────────────
        │                                            │                             │
        │ ① CAS giữ chỗ head++                        │                            │
        ├───────────────────────────────────────────▶│  head ──▶                  │
        │ ② ghi level + msg                           │                            │
        ├───────────────────────────────────────────▶│  buffer[idx]               │
        │ ③ committed=1 (release)                     │                            
        ├───────────────────────────────────────────▶│  committed[idx]=1          │
        │                                            │        ④ load committed     
        │                                            │◀───────────────────────────┤  (acquire == 1?)
        │                                            │        ⑤ CAS tail++         │
        │                                            │◀───────────────────────────┤
        │                                            │  *item = buffer[idx]        │
        │                                            │────────────────────────────▶ ⑥ ghi ra file
        │                                            │  committed[idx]=0 (release) │
        │                                            │◀───────────────────────────┤



----

.. rubric:: 8. Đánh đổi & Hạn chế cần biết

Không có giải pháp nào "miễn phí". Ring buffer hiện tại chấp nhận những đánh đổi
sau — và có hướng khắc phục ở mục :ref:`potential-faults <potential-faults>`.

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: 📉 Mất log khi quá tải
      :class-card: sd-shadow-sm

      Buffer đầy → log mới bị **drop**. Với hệ thống debug điều này chấp nhận
      được, nhưng nếu cần *zero-loss* thì phải thêm cơ chế chờ/đẩy ngược (back-pressure).

   .. grid-item-card:: 🧮 Nhiều producer → có thể ghi đè
      :class-card: sd-shadow-sm

      Khi nhiều thread cùng gọi, thứ tự đọc không tuyệt đối atomic giữa các
      thread; một số log có thể bị **ghi đè** thay vì drop sạch. Cần cơ chế
      **đặt slot riêng cho từng thread** để loại bỏ.

   .. grid-item-card:: 📏 Kích thước cố định
      :class-card: sd-shadow-sm

      ``BUFFER_SIZE`` cố định lúc biên dịch. Đổi kích thước = sửa macro + build
      lại — không co giãn động lúc chạy.

   .. grid-item-card:: 🔁 Phụ thuộc lũy thừa của 2
      :class-card: sd-shadow-sm

      Phép mask ``& (SIZE - 1)`` chỉ đúng khi ``BUFFER_SIZE`` là lũy thừa của 2.
      Đặt 1000 hoặc 1500 sẽ làm logic vòng tròn **sai hoàn toàn**.

.. rubric:: Tóm tắt một dòng

.. list-table::
   :header-rows: 1
   :widths: 22 78

   * - Khía cạnh
     - Quyết định thiết kế
   * - Bộ nhớ
     - Mảng tĩnh ``buffer[1024]``, cấp phát một lần, không ``malloc`` runtime
   * - Đồng bộ
     - ``head``/``tail`` atomic + ``CAS`` (``compare_exchange_weak``)
   * - Dữ liệu an toàn
     - Cờ ``committed`` mỗi ô + cặp **release–acquire**
   * - Quấn vòng
     - Bộ đếm đơn điệu + mask ``& (BUFFER_SIZE - 1)``
   * - Khi đầy
     - ``push`` trả ``-1`` → **drop**, không block producer
   * - Khi rỗng
     - ``pop`` trả ``-1`` → consumer chờ, nhả CPU

.. seealso::
   * :ref:`atomic-opts <atomic-operations>` — đi sâu vào CAS và memory ordering.
   * :ref:`potential-faults <potential-faults>` — các lỗi tiềm ẩn và hướng khắc phục.


