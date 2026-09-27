Linux Kernel Module / Subsystem Study Template
==============================================

.. rst-class:: lead

   Khung phân tích chuẩn cho **mọi module / subsystem trong Linux Kernel** —
   từ ``kernel/sched/``, ``mm/``, ``net/`` tới device driver.
   Học một lần, tái dùng mọi nơi.

.. grid:: 1 2 2 3
   :gutter: 3

   .. grid-item-card:: Purpose-First
      :text-align: center
      :class-card: sd-shadow-sm

      Luôn bắt đầu từ *bài toán hệ thống*,
      không bắt đầu từ code.

   .. grid-item-card:: Resource-Centric
      :text-align: center
      :class-card: sd-shadow-sm

      Mọi module chỉ tồn tại để
      **quản lý một resource** nào đó.

   .. grid-item-card:: Failure-Aware
      :text-align: center
      :class-card: sd-shadow-sm

      Kernel code = happy path 20% +
      failure path 80%. Phải hỏi
      *“khi nào nó chết?”*.

.. dropdown:: Cách dùng template này
   :color: primary
   :open:

   * **Bước 1:** Fork file này cho mỗi subsystem bạn nghiên cứu
     (ví dụ: ``sched-study.rst``, ``mm-study.rst``, ``binder-study.rst``).
   * **Bước 2:** Điền từng mục theo thứ tự **1 → 2 → 3** — không nhảy vào code
     trước khi trả lời được *Purpose* và *System Problem*.
   * **Bước 3:** Mỗi mục đều có ``TODO`` để bạn điền, ``EXAMPLE`` để tham khảo,
     và ``TIP`` để đọc code nhanh.

.. dropdown:: Quy ước trạng thái điền
   :color: secondary

   * ``TODO`` — phần bạn phải điền khi nghiên cứu module mới.
   * ``EXAMPLE`` — ví dụ minh họa (xóa hoặc giữ khi dùng thật).
   * ``TIP`` — mẹo đọc code / debug kernel liên quan tới mục đó.

---

.. rubric:: 1. Overview — Tổng quan module

.. grid:: 1
   :gutter: 3

   .. grid-item-card:: Module Identity
      :class-card: sd-shadow-sm

      .. list-table::
         :header-rows: 1
         :widths: 30 70

         * - Trường
           - ``TODO`` Nội dung cần điền
         * - Tên module / subsystem
           - *Ví dụ: CFS Scheduler / Binder IPC / SLUB Allocator*
         * - Source directory
           - *Ví dụ:* ``kernel/sched/``, ``mm/slub.c``, ``drivers/net/``
         * - Header / API liên quan
           - *Ví dụ:* ``include/linux/sched.h``, ``include/linux/mm.h``
         * - Kernel version đang nghiên cứu
           - *Ví dụ:* ``v6.6``, ``v6.9-rt``, ``android-14-6.1``

.. grid:: 1
   :gutter: 3

   .. grid-item-card:: Position in Kernel
      :class-card: sd-shadow-sm

      Vị trí module trong kiến trúc kernel — hãy vẽ lại cho module của bạn:

      .. code-block:: text

         +----------------+
         |   User Space   |
         +-------^--------+
                 | syscalls / procfs / sysfs / netlink
         +-------v--------+
         |  YOUR MODULE   |  <-- bạn đang ở đây
         +-------^--------+
                 | calls / callbacks / locks
         +---+---+---+---+
         |       |       |
         v       v       v
       Memory Scheduler Drivers

.. tab-set::

   .. tab-item:: Purpose — Vì sao tồn tại?

      .. list-table::
         :header-rows: 1

         * - Câu hỏi
           - ``TODO`` Trả lời
           - ``EXAMPLE``
         * - Module này giải quyết bài toán gì?
           - *Điền 1–2 câu ngắn gọn.*
           - Scheduler: quyết định *task nào chạy, ở đâu, bao lâu*.
         * - Tại sao Linux cần module này?
           - *Không có nó thì điều gì vỡ?*
           - Không có MM: process giẫm RAM lẫn nhau, mất bảo vệ / chia sẻ.
         * - Module nằm ở đâu trong kiến trúc kernel?
           - *Core / mm / net / drivers / fs / ipc? Vẽ đường gọi.*
           - Binder nằm giữa ``drivers/android/`` + ``mm`` + ``sched``.

      .. tip::
         Quy tắc 30 giây: nếu bạn không giải thích được purpose cho một
         fresher trong 30 giây, bạn chưa hiểu module — đừng mở code vội.

   .. tab-item:: Scope — Ranh giới trách nhiệm

      .. grid:: 1 1 2 2
         :gutter: 2

         .. grid-item-card:: Chịu trách nhiệm
            :class-card: sd-shadow-sm

            * ``TODO`` *Liệt kê 3–5 việc module LÀM.*
            * ``EXAMPLE`` SLUB: cấp phát object nhỏ < 8KB, cache per-CPU, debug.

         .. grid-item-card:: Không chịu trách nhiệm
            :class-card: sd-shadow-sm

            * ``TODO`` *Liệt kê việc module KHÔNG làm (dễ nhầm nhất).*
            * ``EXAMPLE`` Scheduler không cấp phát RAM; MM không quyết định thứ tự chạy.

      .. list-table:: Subsystem phụ thuộc vào module này
         :header-rows: 1

         * - Subsystem phụ thuộc
           - Phụ thuộc theo cách nào?
           - Hậu quả nếu module lỗi?
         * - ``TODO`` *Ví dụ: cpufreq*
           - *Ví dụ: đọc PELT load từ scheduler*
           - *Ví dụ: chọn sai tần số → nóng máy, hao pin*
         * - ``TODO`` *Ví dụ: cgroup*
           - *Ví dụ: ...*
           - *Ví dụ: ...*

      .. important::
         Sai lầm phổ biến nhất khi đọc kernel là **đổ lỗi sai tầng**.
         Ghi rõ “không chịu trách nhiệm” giúp bạn debug nhanh gấp 10 lần.


---

.. rubric:: 2. System Problem — Bài toán hệ thống

.. grid:: 1 2 2 3
   :gutter: 3

   .. grid-item-card:: Input
      :text-align: center
      :class-card: sd-shadow-sm

      ``TODO``

      *Dữ liệu / sự kiện đi vào là gì?*

      ``EXAMPLE`` Page fault: địa chỉ lỗi + quyền truy cập + context process.

   .. grid-item-card:: Transform
      :text-align: center
      :class-card: sd-shadow-sm

      ``TODO``

      *Module biến đổi / quyết định điều gì?*

      ``EXAMPLE`` Scheduler: ``vruntime`` → thứ tự chạy.

   .. grid-item-card:: Output
      :text-align: center
      :class-card: sd-shadow-sm

      ``TODO``

      *Kết quả mong muốn là gì? Đo bằng metric nào?*

      ``EXAMPLE`` Cấp phát thành công + latency < X µs.

.. dropdown:: Problem Definition — Định nghĩa bài toán
   :color: primary
   :open:

   .. list-table::
      :header-rows: 1

      * - Câu hỏi gốc
        - ``TODO`` Phân tích của bạn
      * - Bài toán hệ thống mà module giải quyết là gì?
        - *Viết dưới dạng 1 câu problem statement. Ví dụ: “Chia CPU hữu hạn cho N task vô hạn sao cho công bằng + đúng deadline.”*
      * - Input của bài toán là gì?
        - *Struct / event / syscall / IRQ? Tần suất? Kích thước? Ví dụ:* ``struct task_struct``, ``sk_buff``, ``bio``.
      * - Output mong muốn là gì?
        - *Trạng thái / quyết định / dữ liệu trả về + metric thành công. Ví dụ: latency p99, throughput, fairness index.*

.. rubric:: Constraints — Ràng buộc

.. note::
   Kernel không có “máy vô hạn”. Mọi thiết kế đều là **đánh đổi (trade-off)**
   giữa các ràng buộc dưới đây. Hãy tick vào ràng buộc quan trọng nhất của module bạn.

.. list-table::
   :header-rows: 1
   :widths: 20 40 40

   * - Ràng buộc
     - ``TODO`` Mức độ + con số cụ thể
     - ``EXAMPLE``
   * - CPU
     - *Ví dụ: hot-path O(1), không được sleep*
     - Scheduler pick-next phải O(1), giữ ``rq->lock`` cực ngắn.
   * - Memory
     - *Ví dụ: footprint, GFP flags, không leak*
     - SLUB trên embedded: code size < X KB, dùng ``GFP_ATOMIC`` trong IRQ.
   * - Latency
     - *Ví dụ: p50 / p99, real-time deadline*
     - Audio path: < 1ms; ``SCHED_DEADLINE`` miss = lỗi hệ thống.
   * - Throughput
     - *Ví dụ: IOPS / pps / MB/s*
     - NVMe / net: hàng triệu IOPS, batching + polling.
   * - Concurrency
     - *Ví dụ: SMP / preempt / IRQ context*
     - Per-CPU ``rq``, RCU, lockless read cho hot-path.
   * - Hardware limitations
     - *Ví dụ: cache, NUMA, DMA, alignment*
     - DMA cần contiguous + alignment; big.LITTLE cần capacity-aware.
   * - Scalability
     - *Ví dụ: 1 CPU → 128 CPU, cgroup sâu*
     - CFS → EEVDF để scale tốt khi task sleep/wake liên tục.


.. rubric:: Failure Conditions — Khi nào nó thất bại?

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: Resource Exhaustion
      :class-card: sd-shadow-sm

      * ``TODO`` *Cạn tài nguyên nào? ENOMEM / queue full / fd hết?*
      * ``EXAMPLE`` ``kmalloc`` trả ``NULL`` → OOM killer; ``sk_buff`` drop khi ring full.
      * Kernel xử lý: ``goto out_free``? retry? drop + counter? OOM?

      .. code-block:: c
         :caption: Pattern xử lý ENOMEM chuẩn trong kernel

         ptr = kmalloc(size, GFP_KERNEL);
         if (!ptr)
             return -ENOMEM;

   .. grid-item-card:: Deadlock / Race Condition
      :class-card: sd-shadow-sm

      * ``TODO`` *Lock order? IRQ vs process context? AB-BA?*
      * ``EXAMPLE`` ``rq->lock`` + ``pi_lock`` — sai thứ tự = deadlock SMP.
      * Công cụ: ``lockdep``, ``KASAN``, ``syzkaller``.

      .. code-block:: c
         :caption: Pattern khóa an toàn với IRQ

         spin_lock_irqsave(&lock, flags);
         /* critical section ngắn nhất có thể */
         spin_unlock_irqrestore(&lock, flags);

   .. grid-item-card:: Error Propagation
      :class-card: sd-shadow-sm

      * ``TODO`` *Mã lỗi trả về là gì? Ai chịu trách nhiệm cleanup?*
      * ``EXAMPLE`` ``-EINVAL / -EFAULT / -EAGAIN / -ENOMEM`` → user-space ``errno``.
      * Quy tắc: ai alloc người đó free; dùng ``goto cleanup`` thống nhất.

   .. grid-item-card:: Kernel xử lý failure ra sao?
      :class-card: sd-shadow-sm

      * ``TODO`` *WARN_ON? BUG_ON? panic? recovery?*
      * ``EXAMPLE`` Page fault không hợp lệ → ``SIGSEGV``; lỗi nghiêm trọng → ``oops / panic``.
      * Ghi lại: dmesg signature, tracepoint, ``/sys/kernel/debug/`` nào soi được?

.. warning::
   Trong kernel, **không có exception, không có GC, không có “thử lại vô hạn”**.
   Mọi failure path đều phải trả lời: *ai free? ai unlock? ai thông báo cho user?*

---

.. rubric:: 3. Resource — Tài nguyên module quản lý

.. rst-class:: lead

   Hỏi “module này quản lý resource gì?” trước khi hỏi “code nó viết gì?”.

.. grid:: 1 2 2 4
   :gutter: 2

   .. grid-item-card:: CPU
      :text-align: center
      :class-card: sd-shadow-sm

      ``task_struct`` ``rq`` ``cfs_rq``

   .. grid-item-card:: Memory
      :text-align: center
      :class-card: sd-shadow-sm

      ``struct page`` ``folio`` ``slab``

   .. grid-item-card:: Storage
      :text-align: center
      :class-card: sd-shadow-sm

      ``bio`` ``request`` ``inode``

   .. grid-item-card:: Network
      :text-align: center
      :class-card: sd-shadow-sm

      ``sk_buff`` ``socket`` ``netdev``


   .. grid-item-card:: Device
      :text-align: center
      :class-card: sd-shadow-sm

      ``struct device`` ``platform_device``

   .. grid-item-card:: Process
      :text-align: center
      :class-card: sd-shadow-sm

      ``pid`` ``cred`` ``nsproxy``

   .. grid-item-card:: File
      :text-align: center
      :class-card: sd-shadow-sm

      ``file`` ``dentry`` ``vfsmount``

   .. grid-item-card:: Interrupt
      :text-align: center
      :class-card: sd-shadow-sm

      ``irq_desc`` ``softirq`` ``workqueue``

.. dropdown:: Checklist xác định resource chính
   :color: primary

   * ``TODO`` Resource chính module quản lý là gì? (chọn 1–2 trong 8 ô trên)
   * ``TODO`` Struct đại diện cho resource là gì? (``struct ...``)
   * ``TODO`` Ai là owner? Ai là borrower? Lifetime gắn với ai?
   * ``TIP`` Mẹo: ``grep -rn "struct <tên>" include/linux/`` rồi lần ngược
     ai alloc / ai free là ra ownership.

.. rubric:: Resource Lifecycle — Vòng đời tài nguyên

.. code-block:: text
   :caption: Vòng đời chuẩn — hãy map từng bước sang hàm thật trong module của bạn

   Create -> Initialize -> Use -> Modify -> Release -> Destroy
     |          |           |        |          |           |
     v          v           v        v          v           v
   alloc      init       lookup    update     put/free    kfree
   open       setup      get/find  write      close       destroy
   request    probe      map       migrate    unmap       remove

.. list-table::
   :header-rows: 1
   :widths: 15 25 35 25

   * - Giai đoạn
     - ``TODO`` Hàm / API tương ứng
     - ``EXAMPLE`` (Scheduler / SLUB / Driver)
     - Refcount / Lock?
   * - Create
     - *Ví dụ:* ``kmalloc()``, ``alloc_task_struct()``, ``device_create()``
     - ``fork()`` → ``copy_process()`` tạo ``task_struct``
     - *Ai giữ ref đầu tiên?*
   * - Initialize
     - *Ví dụ:* ``init_task()``, ``driver_probe()``, ``inode_init()``
     - ``sched_fork()`` init ``se.vruntime``, policy, prio
     - *Khởi tạo dưới lock nào?*
   * - Use
     - *Ví dụ:* ``get_task()``, ``kref_get()``, ``try_module_get()``
     - ``pick_next_task()`` chọn task; ``kmem_cache_alloc()`` lấy object
     - *Path này có sleep được không? IRQ-safe?*
   * - Modify
     - *Ví dụ:* ``sched_setattr()``, ``set_priority()``, ``remap()``
     - ``nice()`` đổi weight → recompute ``vruntime``
     - *Cần lock gì? Có cần migrate?*
   * - Release
     - *Ví dụ:* ``put_task()``, ``kfree()``, ``device_destroy()``
     - ``put_task_struct()`` giảm refcount; ``__free_slab()``
     - *Double-free / use-after-free phòng bằng gì? (KASAN, RCU)*
   * - Destroy
     - *Ví dụ:* ``destroy_workqueue()``, ``kmem_cache_destroy()``
     - ``free_task()`` trả về slab; driver ``remove()``
     - *Ai là người cuối cùng gọi? Có RCU grace period?*

.. tab-set::

   .. tab-item:: Gotchas — Bẫy thường gặp

      * **Use-after-free:** free rồi mà con trỏ khác vẫn giữ → dùng ``kref`` + ``RCU``.
      * **Leak trên error path:** ``goto`` thiếu một nhánh free → dùng ``__free()`` cleanup guard.
      * **Sleep trong atomic:** gọi ``kmalloc(GFP_KERNEL)`` trong spinlock/IRQ → phải ``GFP_ATOMIC``.
      * **Refcount imbalance:** ``get`` mà quên ``put`` → leak; ``put`` 2 lần → UAF.

      .. code-block:: c
         :caption: Mẫu refcount + cleanup an toàn

         struct my_obj *obj = kzalloc(sizeof(*obj), GFP_KERNEL);
         if (!obj)
             return -ENOMEM;
         kref_init(&obj->ref);
         kref_put(&obj->ref, my_obj_release);

   .. tab-item:: Debug vòng đời resource

      .. list-table::
         :header-rows: 1

         * - Công cụ
           - Dùng khi nào?
           - Lệnh mẫu
         * - ``slabtop`` / ``/proc/slabinfo``
           - Nghi leak slab
           - ``watch -n1 'cat /proc/slabinfo | head'``
         * - ``KASAN`` / ``KMSAN``
           - Nghi UAF / OOB
           - ``CONFIG_KASAN=y`` + ``dmesg | grep KASAN``
         * - ``refcount`` + ``lockdep``
           - Nghi race / deadlock
           - ``dmesg | grep lockdep``
         * - ``ftrace`` / ``trace-cmd``
           - Lần theo alloc → free
           - ``trace-cmd record -e kmem:kmalloc -e kmem:kfree``
         * - ``crash`` / ``drgn``
           - Soi struct sau oops
           - ``drgn prog <script>`` / ``crash vmlinux vmcore``

.. seealso::

   * `Linux Kernel Documentation — Memory Management <https://www.kernel.org/doc/html/latest/mm/index.html>`_
   * `kernel/sched/ source — Elixir Bootlin <https://elixir.bootlin.com/linux/latest/source/kernel/sched>`_
   * :doc:`sched/index` — ví dụ áp dụng khung này vào Scheduler.
   * :doc:`../general_paper/kernel-mm` — ví dụ áp dụng vào Memory Management.

           - *Ví dụ:* ``kernel/sched/``, ``mm/slub.c``, ``drivers/net/``
