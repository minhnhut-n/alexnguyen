.. _atomic-operations:

====================
2. Atomic Operations
====================

.. rst-class:: lead

   **Atomic operation** (phép toán nguyên tử) là thao tác **không thể chia nhỏ**:
   nó *hoặc thực hiện trọn vẹn, hoặc không thực hiện chút nào* — không có trạng
   thái "làm nửa vời". Trong C, cả họ atomic này được chuẩn hoá bởi **C11** qua
   header ``<stdatomic.h>``, cho phép viết code **đa luồng không cần khóa
   (lock-free)** một cách *chuẩn C*, không phụ thuộc compiler.

.. grid:: 1 2 2 4
   :gutter: 3

   .. grid-item-card:: 🧩 Nguyên tử
      :text-align: center
      :class-card: sd-shadow-sm

      Không bị *torn read/write*: giá trị luôn nhất quán, không "xen ở giữa".

   .. grid-item-card:: 🧠 Thứ tự bộ nhớ
      :text-align: center
      :class-card: sd-shadow-sm

      6 cấp ``memory_order`` để điều tiến *nhìn thấy* dữ liệu giữa các luồng.

   .. grid-item-card:: 🔧 Đa dạng phép toán
      :text-align: center
      :class-card: sd-shadow-sm

      load/store, exchange, CAS, fetch-add/or/and/xor, flag, fence.

   .. grid-item-card:: ⚡ Không cần mutex
      :text-align: center
      :class-card: sd-shadow-sm

      Nhiều hot-path chạy **lock-free**, tránh block và deadlock.

----

.. rubric:: 1. Vấn đề: vì sao ``x++`` không an toàn?

Với biến thường, ``counter++`` trông như *một* lệnh nhưng thực ra là **ba bước**
tại tầng CPU: **đọc → cộng → ghi**. Nếu hai luồng cùng thực hiện, chúng có thể
đè lên nhau và làm **mất** một lần tăng.

.. code-block:: text
   :caption: Race condition — hai luồng tranh giành counter thường

   Luồng A đọc counter=5          Luồng B đọc counter=5
        │                              │
        ▼                              ▼
   A tính 5+1 = 6                 B tính 5+1 = 6
        │                              │
        └──────────────┬───────────────┘
                       ▼
              GHI ĐÈ LẪN NHAU → counter = 6   (lẽ ra phải là 7!)
                       ▲
                  ❌ mất 1 lần tăng (lost update)

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: 🔒 Cách cũ — mutex
      :class-card: sd-shadow-sm

      Khoá lại đoạn găng (critical section). **An toàn** nhưng có chi phí:
      khóa/mở, có thể *block*, nghẽn, thậm chí *deadlock* nếu dùng sai thứ tự.

   .. grid-item-card:: ⚛️ Cách hiện đại — atomic
      :class-card: sd-shadow-sm

      Nhờ lệnh phần cứng đặc biệt (LL/SC, CMPXCHG…), CPU thực hiện *nguyên tử*
      trong **một** bước. Không cần mutex, không block hot-path.

.. note::
   Trong dự án **Build Logger**, ring buffer (xem :ref:`ring-buffer-impl`) dùng
   atomic cho ``head``/``tail`` và cờ ``committed`` để producer/consumer giao
   tiếp **không cần khóa** — bài viết này chính là "từ điển" cho những atomic đó.

.. seealso::
   * ``<stdatomic.h>`` — header chuẩn C11 chứa toàn bộ họ atomic.
   * :ref:`socket-commu-guide` và :ref:`ring-buffer-impl` — nơi atomic được *vận dụng* thực tế.


----

.. rubric:: 2. Kiểu dữ liệu atomic & khởi tạo

C cung cấp **hai cách** khai báo biến atomic:

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: 🔤 Qualifier ``_Atomic``
      :class-card: sd-shadow-sm

      Gắn ``_Atomic`` vào kiểu bất kỳ (C11):

      .. code-block:: c

         _Atomic int counter;          // int atomic
         _Atomic(uint32_t) head;       // kiểu trong ngoặc (như dự án)
         atomic_bool done;             // dùng typedef sẵn có

   .. grid-item-card:: 📦 Các typedef sẵn có
      :class-card: sd-shadow-sm

      Thư viện định nghĩa sẵn cho các kiểu phổ biến:

      .. code-block:: c

         atomic_bool      atomic_char    atomic_schar
         atomic_uchar     atomic_short   atomic_ushort
         atomic_int       atomic_uint    atomic_long
         atomic_ulong     atomic_llong   atomic_ullong
         // …và kiểu cỡ cố định: atomic_int32_t, atomic_uintptr_t…

.. list-table:: Cách khởi tạo biến atomic
   :header-rows: 1
   :widths: 30 40 30

   * - Cách
     - Cú pháp
     - Ghi chú
   * - Khởi tạo tĩnh (compile-time)
     - ``atomic_int x = 0;`` hoặc ``= ATOMIC_VAR_INIT(5);``
     - ``ATOMIC_VAR_INIT`` **lỗi thời** ở C23, ưu tiên gán trực tiếp
   * - Khởi tạo lúc chạy
     - ``atomic_init(&x, 5);``
     - Dùng khi chưa biết giá trị ban đầu
   * - Zero-init cho struct
     - ``= {.head=0, .tail=0}``
     - Như ``g_ringbuf`` trong daemon

.. code-block:: c
   :caption: Ví dụ khớp với mã dự án — biến atomic trong ring buffer

   // ring_buffer.h
   typedef struct {
       _Atomic(uint32_t) head;            // con trỏ GHI  (producer)
       _Atomic(uint32_t) tail;            // con trỏ ĐỌC  (consumer)
       log_item_t buffer[BUFFER_SIZE];
   } log_ring_buffer_t;

   // logger-daemon.c — khởi tạo tĩnh, head/tail = 0
   static log_ring_buffer_t g_ringbuf = {.head = 0, .tail = 0};

.. tip::
   ``ATOMIC_VAR_INIT(value)`` từng là cách khởi tạo compile-time, nhưng kể từ
   **C23** nó bị *loại bỏ*. Giờ đây ta **gán trực tiếp** (``atomic_int x = 5;``)
   cho compile-time, và dùng ``atomic_init()`` cho runtime.



----

.. rubric:: 3. Thứ tự bộ nhớ (memory ordering)

Mỗi phép toán atomic đều nhận một tham số **memory order** quyết định: (1) thao
tác này *ràng buộc thứ tự* với các thao tác khác thế nào, và (2) khi nào dữ liệu
*trở nên nhìn thấy* được cho luồng khác. C11 định nghĩa **6 mức**:

.. list-table:: Sáu cấp độ ``memory_order``
   :header-rows: 1
   :widths: 22 44 34

   * - Mức
     - Ý nghĩa
     - Dùng khi
   * - ``memory_order_relaxed``
     - Chỉ bảo đảm *nguyên tử*, **không** lập thứ tự với thao tác khác
     - Bộ đếm, thống kê — không truyền dữ liệu
   * - ``memory_order_consume``
     - Như acquire nhưng chỉ cho dữ liệu *phụ thuộc* (hiện ít dùng)
     - Rarely used (C++17 khuyên tránh)
   * - ``memory_order_acquire``
     - **Load**: mọi thao tác *sau* không bị đẩy lên *trước* load này
     - Đọc cờ/dữ liệu do luồng khác publish
   * - ``memory_order_release``
     - **Store**: mọi thao tác *trước* không bị đẩy xuống *sau* store này
     - Publish dữ liệu, mở cờ cho luồng khác
   * - ``memory_order_acq_rel``
     - Kết hợp acquire + release (cho RMW/CAS)
     - CAS vừa đọc vừa ghi
   * - ``memory_order_seq_cst``
     - Mạnh nhất: **mọi** atomic đều có *một thứ tự toàn cục* duy nhất
     - Mặc định khi không chỉ định; khi cần chắc chắn tuyệt đối

.. grid:: 1 1 2 3
   :gutter: 2

   .. grid-item-card:: 🌀 relaxed — "chỉ đếm"
      :class-card: sd-shadow-sm

      Không tạo *ràng buộc thứ tự*. Dùng cho bộ đếm thuần túy, nơi bạn chỉ cần
      phép cộng *không mất mát*, không cần đồng bộ dữ liệu khác.

   .. grid-item-card:: 🔓 acquire — "cửa vào"
      :class-card: sd-shadow-sm

      Đặt ở phía **đọc**. Bảo đảm sau khi đọc xong, ta *nhìn thấy* mọi ghi trước
      lệnh release tương ứng của luồng kia.

   .. grid-item-card:: 🔐 release — "cửa ra"
      :class-card: sd-shadow-sm

      Đặt ở phía **ghi/publish**. Bảo đảm mọi ghi *trước* đó đã hoàn tất & hiện
      hữu trước khi store này trở nên nhìn thấy.

   .. grid-item-card:: 🔁 acq_rel — "hai chiều"
      :class-card: sd-shadow-sm

      Vừa acquire vừa release — dùng cho **CAS/fetch** khi một thao tác vừa
      đọc vừa ghi và cần cả hai ràng buộc.

   .. grid-item-card:: 🌐 seq_cst — "mạnh nhất"
      :class-card: sd-shadow-sm

      Thứ tự tuần tự toàn cục. Dễ đoán nhất, đắt nhất trên ARM/Power. Đây là
      **mặc định** của ``atomic_load``/``atomic_store`` (bản không ``_explicit``).

.. important::
   **Quy tắc vàng:** *"Ghi dữ liệu → store release"* ở producer khớp với *"load
   acquire → đọc dữ liệu"* ở consumer. Cặp **release–acquire** này tạo quan hệ
   *happens-before*, là nền tảng để truyền dữ liệu an toàn giữa các luồng.

.. code-block:: text
   :caption: Cặp release–acquire truyền dữ liệu — consumer luôn thấy dữ liệu đủ

   LUỒNG A (Producer)                              LUỒNG B (Consumer)
   ──────────────────                              ──────────────────
   data = 42;            (ghi thường)             if (atomic_load(&flag, acquire) == 1)
   atomic_store(&flag,   ─────────synchronize────────▶  assert(data == 42);
      1, release);          with                       // đọc data hợp lệ!
                          (release)
   ── mọi ghi TRƯỚC store đã xong ──▶ ── load acquire nhìn thấy TẤT CẢ ──▶

.. warning::
   Chỉ *một phía* release/acquire là **không đủ**. Thiếu release ở producer, hoặc
   thiếu acquire ở consumer, thì thứ tự không được bảo đảm → consumer có thể đọc
   phải ``data`` cũ dù ``flag`` đã bật. Cả hai phía phải *khớp cặp*.



----

.. rubric:: 4. Các phép toán atomic cơ bản

Mỗi thao tác đều có **hai biến thể**: bản ngắn (mặc định ``seq_cst``) và bản
``_explicit`` (tự chọn memory order).

.. grid:: 1
   :gutter: 3

   .. grid-item-card:: 📥 ``atomic_load`` / ``atomic_store``
      :class-card: sd-shadow-sm

      Đọc / ghi một giá trị atomic.

      .. code-block:: c

         // Đọc — mặc định seq_cst
         uint32_t h = atomic_load(&rb->head);
         // Đọc với thứ tự tuỳ chọn
         uint32_t h2 = atomic_load_explicit(&rb->head, memory_order_relaxed);

         // Ghi — mặc định seq_cst
         atomic_store(&flag, 1);
         // Ghi với release (publish dữ liệu trước đó)
         atomic_store_explicit(&rb->buffer[i].committed, 1, memory_order_release);

   .. grid-item-card:: 🔄 ``atomic_exchange``
      :class-card: sd-shadow-sm

      Ghi giá trị mới và **trả về giá trị cũ** — tất cả trong một bước nguyên tử.

      .. code-block:: c

         atomic_int x = 10;
         int old = atomic_exchange(&x, 20);   // old = 10, x = 20
         // với thứ tự tuỳ chọn:
         atomic_exchange_explicit(&x, 30, memory_order_acq_rel);

.. note::
   **Load trả về *giá trị*, không phải con trỏ.** Bạn không thể lấy địa chỉ của
   kết quả ``atomic_load()`` hay gán ``&atomic_load(&x)`` — nó là một *bản sao*
   theo giá trị.

----

.. rubric:: 5. Read–Modify–Write (RMW)

Đây là nhóm thao tác **đọc–sửa–ghi trong một bước nguyên tử**. Tất cả đều *trả
về giá trị trước khi sửa*.

.. code-block:: c
   :caption: Họ atomic_fetch_* — cộng, bitwise, tất cả đều nguyên tử

   atomic_fetch_add(&counter, 5);   // counter += 5, trả về giá trị CŨ
   atomic_fetch_sub(&counter, 3);   // counter -= 3, trả về giá trị CŨ
   atomic_fetch_or (&flags,  0x01); // flags  |= 0x01
   atomic_fetch_and(&flags,  0xFE); // flags  &= 0xFE
   atomic_fetch_xor(&flags,  0xFF); // flags  ^= 0xFF

   // Mỗi hàm đều có bản _explicit:
   atomic_fetch_add_explicit(&counter, 5, memory_order_relaxed);

.. important::
   **Nhớ: ``fetch_*`` trả về giá trị TRƯỚC, không phải sau.** Đây là cái bẫy
   kinh điển. Nếu ``counter == 10`` và gọi ``atomic_fetch_add(&counter, 5)`` thì
   giá trị trả về là ``10`` (không phải ``15``), còn ``counter`` trong bộ nhớ
   đã thành ``15``.

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: ➕ ``fetch_add`` / ``fetch_sub``
      :class-card: sd-shadow-sm

      Chỉ áp dụng cho kiểu **số** (con trỏ cũng được). Đây là cách *an toàn*
      thay cho ``counter++`` trong đa luồng.

   .. grid-item-card:: 🔣 ``fetch_or``/``and``/``xor``
      :class-card: sd-shadow-sm

      Chỉ áp dụng cho kiểu **nguyên**. Bật/tắt/đảo bit cờ (flag) một cách
      nguyên tử — ví dụ quản lý ``bitmask`` trạng thái.



----

.. rubric:: 6. Compare-And-Swap (CAS) — trái tim của lock-free

CAS là phép toán **quyết định** của mọi cấu trúc lock-free. Nó hỏi: *"giá trị
hiện tại có đúng bằng giá trị tôi đọc lúc trước không? Nếu đúng → ghi giá trị
mới; nếu không → báo cho tôi biết để thử lại."*

.. code-block:: c
   :caption: Cặp lệnh CAS trong C11

   // STRONG: không bao giờ thất bại giả (spurious)
   _Bool atomic_compare_exchange_strong(volatile A *obj,
                                        C *expected, C desired);
   // WEAK: CÓ THỂ thất bại giả → thường rẻ hơn, dùng trong vòng lặp
   _Bool atomic_compare_exchange_weak (volatile A *obj,
                                       C *expected, C desired);
   // Bản _explicit: tự chọn memory order cho success / failure
   atomic_compare_exchange_weak_explicit(&rb->head, &curr_head, curr_head + 1,
                                         memory_order_acquire,   // nếu thành công
                                         memory_order_relaxed);  // nếu thất bại

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: 🦾 strong — "chắc ăn"
      :class-card: sd-shadow-sm

      Chỉ trả ``false`` khi giá trị **thực sự** khác. Đắt hơn một chút trên vài
      kiến trúc. Dùng khi bạn *không* lặp lại ngay (ví dụ CAS đơn lẻ).

   .. grid-item-card:: 🪶 weak — "nhẹ, thử lại"
      :class-card: sd-shadow-sm

      Có thể trả ``false`` *dù giá trị vẫn khớp* (thất bại giả). Rẻ hơn, và vì
      luôn đặt trong ``do…while`` nên không sao. **Ring buffer dùng bản này.**

.. important::
   **Điểm dễ nhầm nhất của CAS:** tham số ``expected`` được **truyền bằng con
   trỏ** và **bị ghi đè** bằng giá trị hiện tại của ``obj`` khi CAS thất bại.
   Nhờ vậy bạn *không cần* đọc lại — dùng trực tiếp ``expected`` cho vòng lặp kế tiếp.

.. code-block:: text
   :caption: Cơ chế CAS trong ring_buffer_push — giữ chỗ trước khi ghi

   do {
       curr_head = atomic_load (&rb->head, relaxed);   // đọc giá trị hiện tại
       curr_tail = atomic_load (&rb->tail, acquire);
       if ((curr_head - curr_tail) >= BUFFER_SIZE)      // buffer đầy?
           return -1;                                   //   → drop
   } while (!CAS_weak(&rb->head,                        // ┌ thử giữ chỗ:
                      &curr_head,                        // │ "head còn == curr_head?"
                      curr_head + 1,                     // │ nếu đúng → head++
                      acquire, relaxed));                // └ kết quả: true/false
   // ── thoát vòng lặp nghĩa là: ta ĐÃ sở hữu ô curr_head ──
   write_idx = curr_head & (BUFFER_SIZE - 1);
   ...ghi dữ liệu...; committed[write_idx] = 1 (release);

.. tip::
   Vì sao dùng **CAS trên head** thay vì chỉ ``atomic_fetch_add``? Vì ``push``
   cần **kiểm tra đầy** *trước khi* chiếm ô. CAS cho phép "kiểm tra + chiếm"
   trong một bước, tránh tình trạng hai producer cùng chiếm một ô khi buffer gần đầy.



----

.. rubric:: 7. ``atomic_flag`` — kiểu atomic đặc biệt

``atomic_flag`` là kiểu atomic **đơn giản nhất** và là kiểu **duy nhất được
bảo đảm lock-free** trên mọi nền tảng. Nó chỉ có hai trạng thái: bật/tắt — lý
tưởng để chế spinlock tối thiản.

.. code-block:: c
   :caption: atomic_flag — hai thao tác duy nhất

   atomic_flag lock = ATOMIC_FLAG_INIT;   // khởi tạo: trạng thái "tắt"

   // Test-and-set: đặt cờ = 1 và trả về giá trị CŨ (nguyên tử)
   while (atomic_flag_test_and_set(&lock)) { /* bận → chờ */ }
   // ── vào được đây nghĩa là đã "giữ" lock ──
   atomic_flag_clear(&lock);              // thả lock
   // Bản _explicit:
   atomic_flag_test_and_set_explicit(&lock, memory_order_acquire);
   atomic_flag_clear_explicit(&lock, memory_order_release);

.. note::
   ``atomic_flag`` **không** có ``atomic_load`` — bạn chỉ được ``test_and_set``.
   Đây là lý do các kiểu atomic khác (``atomic_bool``) *không* bảo đảm lock-free,
   còn ``atomic_flag`` thì có.

----

.. rubric:: 8. Fence & truy vấn

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: 🚧 ``atomic_thread_fence``
      :class-card: sd-shadow-sm

      Rào chắn **thứ tự** độc lập, không gắn với một biến cụ thể. Dùng khi có
      nhiều biến atomic cần cùng một ràng buộc.

      .. code-block:: c

         data = 42;                        // ghi thường
         atomic_thread_fence(memory_order_release);
         atomic_store_explicit(&flag, 1, memory_order_relaxed);

   .. grid-item-card:: 📡 ``atomic_signal_fence``
      :class-card: sd-shadow-sm

      Rào chắn *trong cùng một luồng* (thread vs. signal handler của nó). Không
      sinh lệnh CPU — chỉ chặn compiler tái sắp xếp.

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: 🔍 ``atomic_is_lock_free``
      :class-card: sd-shadow-sm

      Hỏi lúc **runtime**: biến atomic này có thực sự không dùng khoá nội bộ
      không? Trả về ``true`` nếu dùng lệnh phần cứng.

      .. code-block:: c

         if (atomic_is_lock_free(&rb->head))
             printf("head: lock-free\n");
         else
             printf("head: dùng mutex nội bộ!\n");

   .. grid-item-card:: ⚠️ Khi nào cần quan tâm?
      :class-card: sd-shadow-sm

      Trên **x86-64**, hầu hết kiểu nguyên đều lock-free. Trên **ARM 32-bit**,
      ``atomic_llong`` (64-bit) có thể rơi vào *mutex nội bộ* → chậm. Kiểm tra
      nếu bạn nhắm tới hiệu năng thời gian thực.

.. rubric:: 9. Bảng tra toàn bộ atomic trong C

.. list-table:: Tổng hợp các thao tác (mỗi cái có bản ngắn + ``_explicit``)
   :header-rows: 1
   :widths: 34 40 26

   * - Hàm
     - Tác dụng
     - Trả về
   * - ``atomic_init``
     - Khởi tạo giá trị (runtime)
     - ``void``
   * - ``atomic_store``
     - Ghi giá trị atomic
     - ``void``
   * - ``atomic_load``
     - Đọc giá trị atomic
     - Giá trị
   * - ``atomic_exchange``
     - Ghi mới, đổi lấy cũ
     - Giá trị **cũ**
   * - ``atomic_compare_exchange_strong``
     - CAS chắc ăn (không lỗi giả)
     - ``bool`` (thành công?)
   * - ``atomic_compare_exchange_weak``
     - CAS nhẹ (có thể lỗi giả)
     - ``bool`` (thành công?)
   * - ``atomic_fetch_add`` / ``_sub``
     - Cộng / trừ nguyên tử (kiểu số)
     - Giá trị **cũ**
   * - ``atomic_fetch_or`` / ``_and`` / ``_xor``
     - Phép bit nguyên tử (kiểu nguyên)
     - Giá trị **cũ**
   * - ``atomic_flag_test_and_set``
     - Đặt cờ, đọc cũ (``atomic_flag``)
     - Giá trị **cũ**
   * - ``atomic_flag_clear``
     - Xoá cờ (``atomic_flag``)
     - ``void``
   * - ``atomic_thread_fence``
     - Rào chắn thứ tự giữa các luồng
     - ``void``
   * - ``atomic_signal_fence``
     - Rào chắn thread ↔ signal handler
     - ``void``
   * - ``atomic_is_lock_free``
     - Kiểm tra có lock-free không
     - ``bool``

.. rubric:: 10. Best practices & cái bẫy

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: ✅ Mặc định là ``seq_cst``
      :class-card: sd-shadow-sm

      Khi **không** dùng bản ``_explicit``, mọi thao tác là ``seq_cst`` — an toàn
      nhất, dễ đoán. Chỉ *tinh chỉnh* xuống ``relaxed/acquire/release`` khi đã
      đo điểm nóng và thực sự cần.

   .. grid-item-card:: 🪤 Bẫy ``fetch_*`` trả giá trị cũ
      :class-card: sd-shadow-sm

      ``int old = atomic_fetch_add(&x, 1);`` cho ``old`` = giá trị **trước** khi
      cộng. Nhiều người tưởng là giá trị sau → bug ngớ ngẩn.

   .. grid-item-card:: 🧵 ``volatile`` ≠ atomic
      :class-card: sd-shadow-sm

      ``volatile`` chỉ ngăn compiler *tối ưu hoá/cache*, **không** ngăn CPU tái
      thứ tự và **không** nguyên tử. Đa luồng phải dùng ``_Atomic``.

   .. grid-item-card:: 🔗 Cặp release–acquire phải khớp
      :class-card: sd-shadow-sm

      Muốn truyền dữ liệu: producer ``store(release)``, consumer
      ``load(acquire)``. Một phía thiếu là mất đồng bộ. Chi tiết vận dụng ở
      :ref:`ring-buffer-impl`.

.. rubric:: Tóm tắt

.. list-table::
   :header-rows: 1
   :widths: 22 78

   * - Chủ đề
     - Điểm cần nhớ
   * - Vấn đề
     - ``x++`` = đọc+cộng+ghi → **không** nguyên tử, có race
   * - Giải pháp
     - Dùng ``<stdatomic.h>`` (C11) — chuẩn, không phụ thuộc compiler
   * - Kiểu
     - ``_Atomic(T)`` hoặc typedef ``atomic_int``, ``atomic_bool``, ``atomic_flag``…
   * - Thứ tự
     - 6 mức: relaxed → consume → acquire/release → acq_rel → seq_cst
   * - Cặp then chốt
     - **release** (publish) ↔ **acquire** (nhận) tạo *happens-before*
   * - RMW
     - exchange, CAS strong/weak, fetch_add/sub/or/and/xor — đều trả giá trị **cũ**
   * - Vận dụng
     - Ring buffer dùng CAS(weak) trên head/tail + release/acquire cho cờ committed

.. seealso::
   * :ref:`ring-buffer-impl` — CAS và release/acquire được *dùng thật* trong ring buffer.
   * :ref:`potential-faults` — đánh đổi khi nhiều producer dùng chung atomic.


