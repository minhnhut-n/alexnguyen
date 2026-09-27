Introduction — Scheduler và bài toán điều phối - Vers2
======================================================

.. rst-class:: lead

   Scheduler là **“bộ não điều phối CPU”** của Linux: quyết định *task nào, chạy ở đâu, trong bao lâu* — để vừa **công bằng**, vừa **đúng deadline**, vừa **tiết kiệm pin**.

.. grid:: 1 2 2 3
   :gutter: 3

   .. grid-item-card:: Công bằng
      :text-align: center
      :class-card: sd-shadow-sm

      Chia CPU hợp lý giữa hàng trăm task.
      Đại diện: **CFS, EEVDF**.

   .. grid-item-card:: Đúng giờ
      :text-align: center
      :class-card: sd-shadow-sm

      Đảm bảo task real-time không trễ.
      Đại diện: **RT, Deadline**.

   .. grid-item-card:: Tiết kiệm năng lượng
      :text-align: center
      :class-card: sd-shadow-sm

      Chọn CPU/core “rẻ điện” nhất còn đủ lực.
      Đại diện: **EAS, CAS**.

.. dropdown:: Bạn sẽ học được gì trong chương Scheduler này?
   :color: primary
   :open:

   * **Tư duy gốc:** vì sao OS không thể sống thiếu scheduler (từ ``temp.md``).
   * **Bức tranh toàn cảnh:** các scheduler class đang có trong mainline.
   * **Bộ máy hỗ trợ:** EAS / Capacity-Aware Scheduling hoạt động kề bên scheduler ra sao.
   * **Khung phân tích module kernel** chuẩn để áp dụng cho mọi subsystem.

---

.. rubric:: 1. Nếu hệ thống không có scheduler

Nếu một hệ thống không có scheduler, về cơ bản hệ thống sẽ gặp vấn đề rất lớn về hiệu năng và độ ổn định.

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: Làm việc tuần tự — hiện tượng chờ
      :class-card: sd-shadow-sm

      Các block công việc chèn lên nhau. Task này phải chờ task kia hoàn thành, tạo ra waiting và lãng phí tài nguyên.

      .. code-block:: text
         :caption: Task xếp hàng tuần tự, không điều phối

         Task A  ──▶ Task B ──▶ Task C ──▶ Task D
                   (chờ)        (chờ)        (chờ)

      .. code-block:: text
         :caption: CPU idle trong lúc Task A chờ I/O

         Task A:  [CPU work] ──▶ [WAIT I/O ..............] ──▶ [resume]
         CPU:     [busy......] ──▶ [IDLE ................] ──▶ [busy]

      CPU có thể không được sử dụng hiệu quả trong thời gian Task A đang chờ.

   .. grid-item-card:: Interrupt không phải scheduler
      :class-card: sd-shadow-sm

      Interrupt giúp hệ thống phản ứng với sự kiện, nhưng **không tự giải quyết bài toán scheduling**.

      .. code-block:: text
         :caption: Interrupt chỉ báo “có sự kiện”, scheduler quyết “ai chạy”

         Hardware / I/O
                │
                ▼
            Interrupt ──▶ Interrupt Handler ──▶ Handle event
                │
                ▼  (vẫn phải trả lời: ai chạy? CPU nào? khi nào?)
            Scheduler ──▶ Task

      Khi event dồn dập (I/O A, I/O B, Timer, Network cùng ngắt):

      * Task nào chạy trước? Task nào chờ?
      * Task nào bị preempt? CPU nào chạy task?

.. important::
   **Interrupt** trả lời *“có sự kiện”*, **Scheduler** trả lời *“ai nên chạy?”*.
   Hai bài toán khác nhau nhưng liên quan chặt chẽ.

Sau khi một task bị interrupt hoặc bị preempt, bài toán còn lại là: **khi nào nó được chạy lại và thứ tự thực thi giữa các task duy trì thế nào?**
Nếu chia sẻ tài nguyên và thứ tự thực thi không được kiểm soát tốt, hệ thống sẽ đối mặt với tình trạng CPU starvation (đói tài nguyên), nghẽn I/O, priority inversion và mất khả năng phản hồi theo thời gian thực.

---

.. rubric:: 2. Khi có nhiều task cùng cần chạy — Execution model

Giả sử có 5 task (A, B, C, D, E) nhưng chỉ có 1 CPU — không thể cho tất cả thực sự chạy cùng một thời điểm. Kernel cần một execution model.

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: Single CPU — xếp thứ tự chạy
      :class-card: sd-shadow-sm

      .. code-block:: text
         :caption: Scheduler tạo thứ tự thực thi trên 1 CPU

            Task A ──┐
            Task B ──┤
            Task C ──┼──▶ Scheduler ──▶ CPU ──▶ A → C → B → A → D → ...
            Task D ──┤
            Task E ──┘

   .. grid-item-card:: Multi-core — chạy song song thật
      :class-card: sd-shadow-sm

      .. code-block:: text
         :caption: Scheduler phân task lên nhiều CPU

                     Scheduler
                    ┌────┼────┐
                    ▼    ▼    ▼
                  CPU 0 CPU 1 CPU 2
                    │    │    │
                    ▼    ▼    ▼
                  A, C  B, E  D, F

      Nhiều task lúc này có thể thực sự chạy song song.

---

.. rubric:: 3. Multithreading, multiprocessing và Synchronization

Mỗi execution context (process / thread) là một dòng thực thi độc lập. Multiprocessing / multithreading tạo ra nhiều context như vậy.

.. tab-set::

   .. tab-item:: Execution context

      Application tạo nhiều thread, multicore chạy song song:

      .. grid:: 1 1 2 2
         :gutter: 2

         .. grid-item-card:: Application tạo thread
            :class-card: sd-shadow-sm

            .. code-block:: text

               Application
                    │
                    ├──▶ Thread A
                    ├──▶ Thread B
                    ├──▶ Thread C
                    └──▶ Thread D

         .. grid-item-card:: Multicore chạy song song
            :class-card: sd-shadow-sm

            .. code-block:: text

               Thread A ──▶ CPU 0
               Thread B ──▶ CPU 1
               Thread C ──▶ CPU 2
               Thread D ──▶ CPU 3

      .. warning::
         Multithreading **không tự giải quyết synchronization**.
         Nếu nhiều thread dùng chung resource, vẫn cần cơ chế đồng bộ.

   .. tab-item:: Synchronization — ai được truy cập?

      Các cơ chế thường gặp:

      .. grid:: 1 2 2 4
         :gutter: 2

         .. grid-item-card:: mutex
            :text-align: center
            :class-card: sd-shadow-sm

            khóa độc quyền

         .. grid-item-card:: semaphore
            :text-align: center
            :class-card: sd-shadow-sm

            đếm slot

         .. grid-item-card:: spinlock
            :text-align: center
            :class-card: sd-shadow-sm

            busy-wait

         .. grid-item-card:: atomic
            :text-align: center
            :class-card: sd-shadow-sm

            op nguyên tử

         .. grid-item-card:: rwlock
            :text-align: center
            :class-card: sd-shadow-sm

            đọc / ghi

         .. grid-item-card:: condvar
            :text-align: center
            :class-card: sd-shadow-sm

            chờ / báo

         .. grid-item-card:: wait queue
            :text-align: center
            :class-card: sd-shadow-sm

            hàng chờ sleep

         .. grid-item-card:: RCU / completion
            :text-align: center
            :class-card: sd-shadow-sm

            đọc không khóa

      .. code-block:: text
         :caption: Luồng lock → dùng → unlock → wake → schedule

         Thread A: lock ──▶ use Shared Resource ──▶ unlock ──▶ wake Thread B
                                                                           │
                                                                           ▼
                                  Scheduler ◀── Thread B chạy ◀── lock ── ...
                                  (Thread B BLOCK / WAIT trong lúc resource bị giữ)

      .. note::
         **Synchronization** quyết định *ai được phép truy cập resource*.
         **Scheduler** quyết định *task nào được CPU thực thi và khi nào*.

---

.. rubric:: 4. Race condition, deadlock — Scheduler không sửa thay bạn

Hai thread cùng ``counter++`` có thể xen kẽ đọc-ghi và làm mất update. Scheduler **không tự làm phép toán này thread-safe** — cần lock bảo vệ Shared Resource. Dùng sai mutex / semaphore / spinlock có thể dẫn đến deadlock, starvation hoặc priority inversion.

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: Race condition — counter++
      :class-card: sd-shadow-sm

      .. code-block:: c
         :caption: Hai thread cùng counter++ — xen kẽ gây sai

         counter++;

      .. code-block:: text
         :caption: Xen kẽ đọc-ghi dẫn tới mất update

         Thread A: read ──▶ counter++ ──▶ write
         Thread B:    read ──▶ counter++ ──▶ write (đè mất!)

   .. grid-item-card:: Deadlock — khóa chéo
      :class-card: sd-shadow-sm

      .. code-block:: text
         :caption: A giữ lock A chờ B, B giữ lock B chờ A

         Thread A: lock(A) ──▶ wait for (B) ──┐
                                              ├──▶ DEADLOCK
         Thread B: lock(B) ──▶ wait for (A) ──┘

.. important::
   Scheduler, synchronization và resource management là các vấn đề **liên quan nhưng không đồng nhất**.

---

.. rubric:: 5. Scheduler thực sự giải quyết bài toán gì?

Có thể mô tả scheduler bằng một câu hỏi duy nhất:

.. rst-class:: lead

   Task nào được chạy, khi nào được chạy, chạy trên CPU nào và trong bao lâu?

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: Mô hình điều phối
      :class-card: sd-shadow-sm

      .. code-block:: text
         :caption: Runnable tasks -> Scheduler -> CPU

         Runnable Tasks
                │
                ▼
           Scheduler
                ├──▶ CPU 0 ──▶ Task A
                └──▶ CPU 1 ──▶ Task B

   .. grid-item-card:: Scheduler phải phối hợp
      :class-card: sd-shadow-sm

      * runnable / priority / fairness giữa các task
      * latency / throughput / CPU utilization
      * preemption / context switch
      * CPU affinity / load balancing
      * wakeup / blocking

.. tab-set::

   .. tab-item:: WHEN vs WHO vs WHAT

      .. list-table::
         :header-rows: 1
         :widths: 20 40 40

         * - Thành phần
           - Trả lời câu hỏi
           - Nghĩa là gì?
         * - Scheduler
           - WHEN? — Task chạy lúc nào?
           - Thứ tự + thời điểm + CPU thực thi
         * - Synchronization
           - WHO? — Ai được truy cập resource?
           - Quyền sở hữu tài nguyên dùng chung
         * - Application
           - WHAT? — Task cần làm gì?
           - Logic nghiệp vụ bên trong task

      Scheduler không thể biến ``counter++`` thành operation atomic. Nhược điểm này do tầng lập trình giải quyết, nhưng scheduler tạo lập môi trường cho các cơ chế đồng bộ vận hành.

   .. tab-item:: Luồng phối hợp Lock → Schedule

      .. code-block:: text
         :caption: Synchronization + Scheduler phối hợp

         Thread A: lock ──▶ use ──▶ unlock ──▶ wake Thread B ──▶ Scheduler ──▶ Thread B: lock ──▶ use

      .. code-block:: text
         :caption: Ownership và execution tách bạch

         Synchronization ──▶ resource ownership ──┐
                                                  ├──▶ CPU thực thi
         Scheduler ────────▶ CPU execution ───────┘

      Application dùng được concurrency mà không cần trực tiếp quản lý toàn bộ CPU hardware.

---

.. rubric:: 6. Application mô tả concurrency — Kernel biến thành execution

Application mô tả concurrency (threads / processes); kernel biến concurrency đó thành execution thực sự trên hardware. Application có Thread A / B / C nhưng **không trực tiếp** quyết định các tham số hạ tầng.

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: Application không quyết định hết
      :class-card: sd-shadow-sm

      Application không can thiệp trực tiếp:

      * CPU core nào đảm nhiệm?
      * Thời điểm thực thi cụ thể?
      * Khi nào bị preempt?
      * Task nào được wake / block / chạy tiếp?

   .. grid-item-card:: Kernel đảm nhiệm
      :class-card: sd-shadow-sm

      .. code-block:: text
         :caption: Kernel — tầng biến concurrency thành execution

         Application (threads / processes)
                      │
                      ▼
               ┌──────────────┐
               │    Kernel    │
               │  Scheduler   │
               │  Sync / MM   │
               │  I/O / IRQ   │
               └──────┬───────┘
                      ▼
                   Hardware

.. dropdown:: Scheduler cải thiện performance theo cách nào?
   :color: primary

   Scheduler tốt ảnh hưởng trực tiếp tới throughput, latency, responsiveness, CPU / multicore utilization, power, fairness.

   .. grid:: 1 1 2 2
      :gutter: 2

      .. grid-item-card:: Poor scheduling
         :class-card: sd-shadow-sm

         .. code-block:: text

            Task A,B,C,D ──▶ 1 CPU ──▶ waiting, contention, high latency

      .. grid-item-card:: Good scheduling
         :class-card: sd-shadow-sm

         .. code-block:: text

                Scheduler ──▶ CPU 0: A→C / CPU 1: B→E / CPU 2: D→F

   .. warning::
      Đừng hiểu scheduler tự làm performance tăng “hàng trăm lần”. Mức cải thiện phụ thuộc vào đặc thụ workload, hardware, I/O, sync, MM và policy.

---

.. rubric:: 7. Scheduler là một phần của hệ thống lớn hơn

Scheduler không phải toàn bộ OS:

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: OS gồm nhiều subsystem
      :class-card: sd-shadow-sm

      .. code-block:: text
         :caption: Scheduler chỉ là một nhánh của OS

                 Operating System
               ┌────────┼────────┐
               ▼        ▼        ▼
           Scheduler    MM       FS
               │        │        │
               ▼        ▼        ▼
              CPU     Memory   Storage

   .. grid-item-card:: Scheduler gắn với MM / FS / Net / Driver
      :class-card: sd-shadow-sm

      .. code-block:: text
         :caption: Task state, wakeup, blocking nối Scheduler với MM và I/O

         Scheduler ──▶ task state / wakeup / blocking / CPU alloc ──▶ MM ──▶ FS / Net / Driver

Các subsystem không hoàn toàn độc lập — performance hệ thống không đến từ scheduler riêng lẻ mà từ sự phối hợp: Scheduling + MM + I/O + Sync + Driver + Hardware + Application behavior.

---

.. rubric:: 8. Tuy hai mà một, tuy một mà hai

Các subsystem cần tách ra để dễ phát triển, debug, bảo trì, giảm coupling, dễ thay đổi implementation. Nhưng chúng không thể tách hoàn toàn.

.. tab-set::

   .. tab-item:: Vì sao phải modular?

      .. code-block:: text
         :caption: Gom tất cả thành một khối — sửa một chỗ vỡ nhiều chỗ

         ┌─────────────────────────────────┐
         │        HUGE KERNEL BLOCK        │
         │   sched + mm + fs + net + ...   │
         └─────────────────────────────────┘

      .. code-block:: text
         :caption: Tách rời hoàn toàn — hệ thống không chạy được

         Scheduler   Memory   FS   Network
             │         │      │       │
             X         X      X       X   (không phối hợp)

   .. tab-item:: Trạng thái ở giữa

      .. code-block:: text
         :caption: Modular + Integrated = Maintainable

               Modular + Integrated
                        │
                        ▼
                Maintainable System

      .. grid:: 1 1 3 3
         :gutter: 2

         .. grid-item-card:: Tách để
            :text-align: center
            :class-card: sd-shadow-sm

            dễ phát triển, debug, bảo trì

         .. grid-item-card:: Hợp để
            :text-align: center
            :class-card: sd-shadow-sm

            task + memory + I/O cùng chạy

         .. grid-item-card:: Kết quả
            :text-align: center
            :class-card: sd-shadow-sm

            Tuy hai mà một, tuy một mà hai

.. note::
   Cái hay của Linux không chỉ là có scheduler, mà là cách rất nhiều cơ chế được thiết kế thành subsystem tương đối độc lập, mỗi subsystem giải một bài toán riêng, nhưng vẫn phối hợp thành hệ thống thống nhất.

---

.. rubric:: 9. Mechanism và Policy — How vs What

Một cách quan trọng để hiểu scheduler là phân biệt Mechanism và Policy:

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: Mechanism — How?
      :class-card: sd-shadow-sm

      Kernel thực hiện việc đó **bằng cách nào**?

      * context switch
      * preemption
      * task wakeup
      * runqueue
      * timer
      * CPU migration

   .. grid-item-card:: Policy — What / Which?
      :class-card: sd-shadow-sm

      Kernel quyết định **nên làm gì** dựa trên tiêu chí nào?

      * priority
      * fairness
      * latency
      * deadline
      * CPU affinity
      * load balancing

.. code-block:: text
   :caption: Scheduler = Mechanism (How) + Policy (What)

           Scheduler
          ┌────┴────┐
          ▼         ▼
     Mechanism    Policy
     (How?)       (What/Which?)
          └────┬────┘
               ▼
              CPU

---

.. rubric:: 10. Scheduler như bài toán resource allocation

Scheduler cân ba mục tiêu cùng tranh một CPU: Latency (task chờ bao lâu?), Throughput (xong bao nhiêu việc?), Fairness (chia có công bằng?).

.. grid:: 1 2 2 3
   :gutter: 3

   .. grid-item-card:: Latency
      :text-align: center
      :class-card: sd-shadow-sm

      task chờ bao lâu?

   .. grid-item-card:: Throughput
      :text-align: center
      :class-card: sd-shadow-sm

      xong bao nhiêu việc?

   .. grid-item-card:: Fairness
      :text-align: center
      :class-card: sd-shadow-sm

      chia có công bằng?

.. code-block:: text
   :caption: Ba mục tiêu cùng tranh một CPU

       Scheduler
      ┌────┼────┐
      ▼    ▼    ▼
   Latency Throughput Fairness
      └────┼────┘
           ▼
    CPU utilization

.. dropdown:: Trade-off không thể tránh
   :color: warning

   Không có scheduler nào tối ưu tất cả cùng lúc:

   .. code-block:: text
      :caption: Được latency thì mất throughput và ngược lại

      Latency
         ▲
         │   ★ điểm bạn chọn
         │  ╱
         └────────────▶ Throughput

   Tương tự với Fairness vs Priority / responsiveness:

   .. code-block:: text
      :caption: Công bằng tuyệt đối thì task gấp phải chờ

      Fairness
         ▲
         │   ★ điểm bạn chọn
         │  ╱
         └────────────▶ Priority / responsiveness

   Vì vậy scheduling là bài toán resource allocation với nhiều mục tiêu và constraints.

---

.. rubric:: 11. Scheduler trong RTOS và GPOS

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: RTOS — Deterministic timing
      :class-card: sd-shadow-sm

      .. code-block:: text
         :caption: RTOS ưu tiên deadline và dự đoán được

         Deterministic timing
                  │
                  ▼
         Deadline / priority
                  │
                  ▼
         Predictable response

      Trọng tâm: timing tất định, đáp ứng dự đoán được.

   .. grid-item-card:: GPOS — Cân bằng nhiều mục tiêu
      :class-card: sd-shadow-sm

      .. grid:: 1 2 2 3
         :gutter: 2

         .. grid-item-card:: Fairness
            :text-align: center
            :class-card: sd-shadow-sm

            chia công bằng

         .. grid-item-card:: Throughput
            :text-align: center
            :class-card: sd-shadow-sm

            xong nhiều việc

         .. grid-item-card:: Latency
            :text-align: center
            :class-card: sd-shadow-sm

            chờ ít

         .. grid-item-card:: Responsiveness
            :text-align: center
            :class-card: sd-shadow-sm

            phản hồi nhanh

         .. grid-item-card:: Power
            :text-align: center
            :class-card: sd-shadow-sm

            tiết kiệm pin

         .. grid-item-card:: Scalability
            :text-align: center
            :class-card: sd-shadow-sm

            scale nhiều CPU

Do đó scheduler của RTOS và GPOS có những mục tiêu và trade-off khác nhau.

---

.. rubric:: 12. Những cơ chế bên dưới scheduler

“Scheduler” thực tế là tên gọi chung cho một nhóm cơ chế:

.. grid:: 1 2 2 3
   :gutter: 2

   .. grid-item-card:: runnable task management
      :text-align: center
      :class-card: sd-shadow-sm

      quản lý task sẵn sàng

   .. grid-item-card:: runqueue
      :text-align: center
      :class-card: sd-shadow-sm

      hàng đợi ``rq``

   .. grid-item-card:: scheduling policy
      :text-align: center
      :class-card: sd-shadow-sm

      CFS / RT / Deadline

   .. grid-item-card:: priority
      :text-align: center
      :class-card: sd-shadow-sm

      độ ưu tiên

   .. grid-item-card:: preemption
      :text-align: center
      :class-card: sd-shadow-sm

      giành / chiếm CPU

   .. grid-item-card:: context switch
      :text-align: center
      :class-card: sd-shadow-sm

      đổi ngữ cảnh

   .. grid-item-card:: wakeup / blocking
      :text-align: center
      :class-card: sd-shadow-sm

      đánh thức / chặn

   .. grid-item-card:: CPU affinity
      :text-align: center
      :class-card: sd-shadow-sm

      gán CPU

   .. grid-item-card:: load balancing
      :text-align: center
      :class-card: sd-shadow-sm

      cân tải

   .. grid-item-card:: timer
      :text-align: center
      :class-card: sd-shadow-sm

      tick định kỳ

   .. grid-item-card:: synchronization
      :text-align: center
      :class-card: sd-shadow-sm

      lock, wait queue

.. note::
   Các cơ chế này được tổ chức thành những subsystem / code path khác nhau nhưng phối hợp rất chặt chẽ.

---

.. rubric:: 13. Cách nhìn tổng thể — Scheduler trong stack OS

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: App → Sync → Scheduler → CPU
      :class-card: sd-shadow-sm

      .. code-block:: text
         :caption: Đường đi của một thread xuống hardware

         Application (Processes / Threads)
                      │
                      ▼
               Synchronization (ai được truy cập?)
                      │
                      ▼
                  Scheduler (task nào? CPU nào? khi nào?)
                      ├──▶ CPU 0
                      ├──▶ CPU 1
                      └──▶ CPU 2
                               │
                               ▼
                            Hardware

   .. grid-item-card:: Scheduler ↔ MM / FS / Net / Driver
      :class-card: sd-shadow-sm

      .. code-block:: text
         :caption: Performance đến từ phối hợp, không từ một module

                   Scheduler
                  ┌────┼────┐
                  ▼    ▼    ▼
                  MM   FS   NET
                   └────┼────┘
                        ▼
                      Drivers
                        │
                        ▼
                 Hardware (CPU/Mem/IO)

.. important::
   Performance của hệ thống không đến từ scheduler riêng lẻ. Nó đến từ sự phối hợp của Scheduling + Memory Management + I/O Management + Synchronization + Drivers + Hardware + Application behavior.

---

.. rubric:: 14. Bản đồ các Scheduler trong Linux hiện nay

* **CFS — Completely Fair Scheduler**

  * *Ý tưởng cốt lõi:* mỗi task có ``vruntime``; ai “thiệt thòi” nhất được chạy trước. Công bằng theo weight.
  * *Khi nào quan tâm:* hệ general-purpose, server, desktop. Hiểu nền này trước khi học EEVDF.

* **EEVDF — Earliest Eligible Virtual Deadline First**

  * *Ý tưởng cốt lõi:* thay ``vruntime`` bằng *virtual deadline*; task nào deadline sớm + đủ điều kiện chạy trước. Thay thế dần CFS từ kernel 6.6+.
  * *Khi nào quan tâm:* cần latency tốt hơn, công bằng hơn khi task sleep/wake liên tục, cgroup nặng.

* **Realtime —** ``RT`` **+** ``Deadline``

  * *Ý tưởng cốt lõi:* ``SCHED_FIFO / SCHED_RR`` chạy theo priority; ``SCHED_DEADLINE`` chạy theo EDF + CBS (runtime / period / deadline).
  * *Khi nào quan tâm:* audio, robot, điều khiển công nghiệp, preempt-rt — nơi trễ 1ms cũng là lỗi.

.. note::
   Muốn hiểu nhanh sự khác nhau **CFS vs EEVDF vs RT**, đọc theo thứ tự: công bằng (fairness) → deadline → priority. Đừng nhảy vào code EEVDF khi chưa hình dung được ``vruntime`` của CFS.

.. seealso::

   * `Ubuntu Real-Time — Schedulers explanation <https://ubuntu.com/real-time/docs/explanation/schedulers/>`_
   * `kernel/sched/ source — kernel.org <https://elixir.bootlin.com/linux/latest/source/kernel/sched>`_

---

.. rubric:: 15. Cơ chế hỗ trợ bên cạnh Scheduler — CAS / EAS

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: CAS — Capacity-Aware Scheduling
      :class-card: sd-shadow-sm

      Không coi mọi CPU là như nhau (big.LITTLE, DynamIQ).

      * Dựa vào **capacity** của từng CPU để đặt task nặng lên core khỏe.
      * Liên quan tới ``arch_topology``, ``capacity_orig``, misfit task.

   .. grid-item-card:: EAS — Energy-Aware Scheduling
      :class-card: sd-shadow-sm

      Chọn **nơi đặt task rẻ điện nhất** mà vẫn giữ performance.

      * Dựa vào **Energy Model (EM)** + ``schedutil`` governor.
      * Quyết định tại wake-up và load-balance: có đáng migrate task không?

.. important::
   Scheduler trả lời **“task nào chạy kế tiếp”**, còn **CAS/EAS** trả lời **“task đó nên chạy trên CPU nào”**. Hai câu hỏi này luôn đi song song trong code.

---

.. rubric:: 16. Kết luận — Hỏi đúng câu hỏi khi nghiên cứu scheduler

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: Stack đầy đủ của OS
      :class-card: sd-shadow-sm

      .. code-block:: text
         :caption: Scheduler chỉ là một tầng trong stack

         Application (task cần làm gì?)
                  │
                  ▼
         Concurrency (thread/process?)
                  │
                  ▼
         Synchronization (ai được truy cập resource?)
                  │
                  ▼
         Scheduling (task nào? CPU nào? khi nào?)
                  │
                  ▼
         Memory / I/O / Drivers
                  │
                  ▼
               Hardware (thực thi instruction?)

   .. grid-item-card:: Chuỗi câu hỏi nghiên cứu
      :class-card: sd-shadow-sm

      .. code-block:: text
         :caption: Đừng chỉ hỏi “chọn task nào”

         Problem ──▶ Resource ──▶ Concurrency ──▶ Sync ──▶ Scheduling ──▶ Mechanism + Policy ──▶ HW ──▶ Perf + Trade-offs

.. dropdown:: Điểm cốt lõi cần nhớ
   :color: success
   :open:

   * Synchronization = ai được phép truy cập resource?
   * Scheduler = task nào được CPU chạy và khi nào?
   * Application = task cần làm gì?
   * Hardware = thực thi instruction như thế nào?
   * Scheduler không trực tiếp giải race condition / deadlock của application, nhưng scheduler + synchronization + resource management tạo ra execution environment để application xử lý concurrency có kiểm soát — từ đó đạt throughput cao hơn, latency thấp hơn, responsiveness tốt hơn.

---

.. rubric:: Đọc tiếp ở đâu?

* Quay lại :doc:`index` để xem mục lục chương Scheduler.
* Đọc sâu Memory trước để hiểu PELT / load tracking? Xem :doc:`../../general_paper/kernel-mm`.
* Chuẩn bị lab: bật ``CONFIG_SCHED_DEBUG=y``, ``CONFIG_SCHEDSTATS=y`` rồi chạy ``stress-ng`` + ``trace-cmd``.
* Template phân tích chuẩn cho mọi module: :doc:`../the-rule-of-research-info`.

.. seealso::
   Tài liệu tham khảo chính: `Ubuntu Real-Time Schedulers <https://ubuntu.com/real-time/docs/explanation/schedulers/>`_.