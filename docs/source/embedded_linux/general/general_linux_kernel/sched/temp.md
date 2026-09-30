---
orphan: true
---

# Scheduler và bài toán điều phối trong hệ điều hành

Nếu một hệ thống không có scheduler, về cơ bản hệ thống sẽ gặp một số vấn đề rất lớn.

Đầu tiên, hệ thống có xu hướng làm việc kiểu tuần tự. Các block công việc chèn lên nhau, task này phải chờ task kia hoàn thành, từ đó tạo ra hiện tượng chờ giữa các task và làm lãng phí tài nguyên.

Tiếp theo, hệ thống phải dựa rất nhiều vào cơ chế **interrupt** để xử lý các sự kiện và đánh thức những phần công việc cần chạy. Nhưng khi có rất nhiều task hoặc event cần được xử lý tại cùng một thời điểm, vấn đề lại xuất hiện: hệ thống phải quyết định task nào được chạy trước, task nào phải chờ, tài nguyên nào được phép truy cập. Nếu không có một cơ chế điều phối tốt, hệ thống rất dễ rơi vào tình trạng quá tải, I/O congestion hoặc tạo ra những hành vi không mong muốn như race condition.

Ngay cả khi đã giải quyết được hai vấn đề trên, hệ thống vẫn còn một bài toán khác: **sau khi một task bị interrupt hoặc bị preempt, khi nào nó được chạy lại và thứ tự thực thi giữa các task sẽ được duy trì như thế nào?** Nếu việc chia sẻ tài nguyên và thứ tự thực thi không được kiểm soát tốt, chúng ta lại có thể gặp race condition, starvation, priority inversion hoặc các vấn đề về timing.

Để giải quyết bài toán làm việc tuần tự giữa các task, rất nhiều cơ chế đã được sinh ra.

Ví dụ, **multithreading và multiprocessing** cho phép hệ thống có nhiều execution context hoạt động đồng thời. Trên hệ thống multicore, nhiều thread có thể thực sự chạy song song trên nhiều CPU core; còn trên một core, scheduler có thể chuyển đổi giữa các thread để tạo ra concurrency.

Sau đó là các cơ chế synchronization như **mutex, semaphore, spinlock, atomic operation, condition/wait queue...** Chúng giúp kiểm soát việc nhiều task cùng truy cập vào một tài nguyên dùng chung, từ đó hạn chế race condition và giúp hệ thống kiểm soát được việc đồng bộ hóa. Tuy nhiên, nếu sử dụng sai, chính những cơ chế này cũng có thể dẫn đến deadlock, starvation hoặc priority inversion.

Nhưng về cơ bản, những cơ chế trên mới chỉ là phần mà chúng ta nhìn thấy ở bên ngoài. Đằng sau chúng còn có cả một hệ thống rất lớn chịu trách nhiệm điều phối và quản lý tài nguyên. Đó mới là phần cốt lõi khiến một hệ thống có thể được gọi là **operating system**.

Nhìn một cách tổng quát, OS có thể tồn tại dưới nhiều dạng, ví dụ như **RTOS** hoặc **GPOS**. Nhưng đối với một hệ thống có rất nhiều process, thread và workload nặng, một trong những thành phần cốt lõi chính là **scheduler**.

Scheduler chịu trách nhiệm quyết định **task nào được chạy, khi nào được chạy, chạy trong bao lâu và CPU nào sẽ thực thi task đó**. Nó phân bổ CPU time cho các task, xử lý việc preemption, context switching và phối hợp với các cơ chế khác để giữ cho hệ thống có thể đáp ứng được nhiều workload cùng lúc.

Từ đây scheduler có thể giúp hệ thống đạt được nhiều mục tiêu khác nhau: tăng throughput, giảm latency, cải thiện responsiveness, tận dụng nhiều CPU core tốt hơn và trong một số trường hợp còn giúp giảm mức tiêu thụ năng lượng. Chính những cơ chế này là một trong những lý do một hệ điều hành hiện đại có thể duy trì trải nghiệm mượt mà ngay cả khi hệ thống đang chịu heavy load.

Dĩ nhiên, **scheduler chỉ là tên gọi chung cho cả một nhóm cơ chế**, chứ bản thân scheduler không phải là một "cỗ máy duy nhất" giải quyết tất cả mọi vấn đề. Bên dưới nó còn có rất nhiều cơ chế khác như runqueue, scheduling policy, context switching, preemption, timer, CPU affinity, load balancing, synchronization và nhiều thành phần khác phối hợp với nhau.

Điều thú vị là những thành phần này vừa được tách biệt để có thể dễ dàng phát triển và bảo trì, nhưng đồng thời chúng lại liên kết rất chặt chẽ với nhau.

Nó giống kiểu **"tuy hai mà một, tuy một mà hai"** hahhah. Không thể tách chúng hoàn toàn thành những hệ thống độc lập, nhưng cũng không thể gom tất cả lại thành một đống code khổng lồ. Nếu làm vậy thì mỗi lần sửa một thứ lại phải nghĩ xem mình đang ảnh hưởng đến cái gì, gọi cái nào trước, cái nào phụ thuộc cái nào, và cuối cùng code sẽ trở thành một đống **bùi nhùi gối đầu nhau**.

Vì vậy, cái hay của một OS như Linux không chỉ nằm ở việc nó có scheduler, mà nằm ở cách rất nhiều cơ chế được thiết kế thành những subsystem tương đối độc lập, mỗi subsystem giải quyết một bài toán riêng, nhưng vẫn có thể phối hợp với nhau để tạo thành một hệ thống thống nhất.
Scheduler, Concurrency và bài toán điều phối trong Operating
System 
1. Hệ thống nếu không có scheduler 
Nếu một hệ thống không có scheduler, hệ thống sẽ gặp một số vấn đề lớn. 
Làm việc tuần tự và hiện tượng chờ 
Task|
v
Task|
v
Task|
v
Task
A
B
C
D
Các block công việc chèn lên nhau. Task này phải chờ task kia hoàn thành, tạo ra waiting và làm lãng phí tài nguyên. 
CPU
|
+---- Task A --------------------+
|
+---- Task B --------+
|
+---- Task C ----

Nếu Task A phải chờ I/O: 
Task A
|
+---- CPU work
|
+---- WAIT I/O ------------------------------+
|
CPU |
| |
+-------------------- IDLE -------------------+
|
v
I/O done
|
v
Task A

CPU có thể không được sử dụng hiệu quả trong thời gian Task A đang chờ. 
 2. Interrupt không phải scheduler 
Interrupt giúp hệ thống phản ứng với sự kiện: 
Hardware / I/O
|
v
Interrupt
|
vInterrupt Handler
|
v
Handle

Nhưng interrupt không tự giải quyết bài toán scheduling. 
Khi có rất nhiều event: 

I/O A ----+
I/O B ----+
Timer ----+----> Interrupts
I/O C ----+
Network --+
hệ thống vẫn phải quyết định: 
Task nào chạy trước?
Task nào phải chờ?
Task nào được đánh thức?
Task nào được preempt?
CPU nào chạy task?
Tài nguyên nào được phép truy cập?

Do đó: 
Interrupt
|
| "Có sự kiện"
v
Scheduler
|
| "Ai nên chạy?"
v
Task

Interrupt và scheduler giải quyết hai bài toán khác nhau nhưng liên quan chặt chẽ. 
 3. Khi có nhiều task cùng cần chạy 
Giả sử: 
TaskTaskTaskTaskTask
A
B
C
D
E
nhưng chỉ có: 
CPU 0

thì không thể cho tất cả thực sự chạy cùng một thời điểm. 
Kernel cần một execution model: 
Task A ----+
Task B ----+
Task C ----+----> Scheduler ----> CPU
Task D ----+
Task E ----+
Scheduler có thể tạo ra thứ tự thực thi như: 
A → C → B → A → D → ...

Với multicore: 
Scheduler
|
+---------+---------+
| | |
v v v
CPU 0 CPU 1 CPU 2
| | |
v v v
A,C B,E D,F

Nhiều task lúc này có thể thực sự chạy song song. 
 4. Multithreading và multiprocessing 
Các cơ chế như process, thread, multiprocessing và multithreading cho phép tạo nhiều execution context: 
Application
|
+---- Thread A
|
+---- Thread B
|
+---- Thread C
|
+---- Thread D

Trên multicore: 
Thread A --------> CPU 0
Thread B --------> CPU 1
Thread C --------> CPU 2
Thread D --------> CPU 3

Nhưng multithreading không tự giải quyết synchronization. 
Nếu nhiều thread dùng chung resource: 
Thread A ----+
|
v
Shared Resource
^
|
Thread B ----+

cần có synchronization. 
 5. Synchronization 
Các cơ chế synchronization thường gặp: 
mutex
semaphore
spinlockatomic operation
read/write lock
condition variable
wait queue
RCU
completion 
Ví dụ: 
Thread A
|
v
lock
|
v
Shared Resource
|
v
unlock

Thread B có thể phải chờ: 
Thread B
|
v
lock
|
+---- resource đang bị giữ
|
+---- BLOCK / WAIT

Khi Thread A release resource: 
Thread A
|
v
unlock
|
v
wake Thread B
|
v
Scheduler
|
v
Thread B chạy

Điểm quan trọng: 
Synchronization quyết định ai được phép truy cập resource. 
Scheduler quyết định task nào được CPU thực thi và khi nào. 
 6. Race condition và deadlock 
Ví dụ: 
counter++;

Nếu hai thread cùng thực hiện: 
Thread A Thread B
| || read counter|| counter++||
|
| read counter
|
| counter++
|
thứ tự thực thi có thể làm kết quả không như mong muốn. 
Scheduler không tự động làm phép toán này thread-safe. 
Application/kernel code cần synchronization phù hợp: 

Shared Resource
|
+------+------+
| |
Thread A Thread B
| |
+------lock---+
Nếu synchronization được thiết kế sai, có thể xảy ra deadlock: 
Thread A Thread B
 lock A lock B
| |
v v
wait for B wait for A
| |
+-----------+---------------+
|
DEADLOCK

Vì vậy scheduler, synchronization và resource management là các vấn đề liên quan nhưng không đồng nhất. 
 7. Scheduler thực sự giải quyết bài toán gì? 
Có thể mô tả scheduler bằng câu hỏi: 
Task nào được chạy, khi nào được chạy, chạy trên CPU nào và trong bao lâu? 
Một mô hình đơn giản: 
Runnable Tasks
|
v
+-------------+
| Scheduler |
+-------------+
|
+------------------+
| |
v v
CPU 0 CPU 1
| |
v v
Task A Task B

Scheduler phải phối hợp: 
runnablepriority
fairnesstasks
latency
CPU utilization
throughput
preemption
context switch
CPU affinity
load balancing
wakeup
blocking 
 8. Scheduler không trực tiếp sửa race condition 
Cần phân biệt rõ: 
Scheduler
|
| WHEN?
v
Task chạy lúc nào?
 Synchronization
|
| WHO?
v
Ai được truy cập resource?
 Application
|
| WHAT?
v
Task cần làm gì?

Scheduler không thể biến: 
counter++;

thành một operation atomic. 
Nhưng scheduler là một phần quan trọng của execution environment mà synchronization mechanisms hoạt động bên trong. 
 9. Scheduler + Synchronization 
Ví dụ: 
Thread A
|
v
lock(resource)
|
v
use resource
|
v
unlock(resource)
|
v
wake Thread B
|
v
Scheduler
|
v
Thread B
|
v
lock(resource)
|
v
use resource
Ở đây có sự phối hợp: 
Synchronization
|
| resource ownership
v
Scheduler
|
| CPU execution
v
CPU

Application vì thế có thể sử dụng concurrency mà không cần trực tiếp quản lý toàn bộ CPU hardware. 
 10. Application chỉ mô tả concurrency 
Một abstraction quan trọng của OS là: 
Application mô tả concurrency; kernel biến concurrency đó thành execution trên hardware. 
Application có thể nói: 
Tôi
có:
ThreadThreadThreadA
B
C
Nhưng application không trực tiếp quyết định toàn bộ: 
CPU core nào?
Thời điểm nào?
Preempt khi nào?
Task nào được wake?
Task nào phải block?
Task nào chạy tiếp?

Kernel đảm nhiệm execution management: 
Application
|
| threads / processes
v
+--------------------------+
| Kernel |
| |
| Scheduler |
| Synchronization |
| Memory Management |
| I/O Management |
| Interrupt Management |
+--------------------------+
|
v
Hardware
 11. Performance improvement 
Một scheduler tốt có thể ảnh hưởng đến: 
throughput
latency
responsiveness
CPU utilization
multicore utilization
power consumption
fairness 
Ví dụ: 
Poor scheduling
----------------
 Task A -----------+
Task B -----------+
Task C -----------+----> CPU
Task D -----------+
  |
+--> waiting
+--> poor utilization
+--> high latency
+--> contention

Một scheduling system tốt hơn có thể phân phối workload: 
Scheduler
|
+----------+----------+
| | |
v v v
CPU 0 CPU 1 CPU 2
| | |
v v v
A -> C B -> E D -> F

Không nên hiểu rằng scheduler tự động làm performance tăng “hàng trăm lần” trong mọi workload. Mức cải thiện phụ thuộc vào
workload, hardware, I/O behavior, synchronization, memory behavior và scheduling policy. 
 12. Scheduler là một phần của hệ thống lớn hơn 
Scheduler không phải toàn bộ OS. 
Operating System
|
+-------------------+-------------------+
| | |
v v v
Scheduler MM FS
| | |
v v v
CPU Memory Storage

Các subsystem không hoàn toàn độc lập. 
Scheduler
|+---- task state
|
+---- wakeup
|
+---- blocking
|
+---- CPU allocation
|
v
Memory Management
|
v
Filesystem / Network / Drivers

 13. “Tuy hai mà một, tuy một mà hai” 
Các subsystem cần được tách ra để: 
dễ phát triển
dễ debug
dễ bảo trì
giảm coupling
dễ thay đổi implementation 
Nhưng chúng không thể tách hoàn toàn: 

Scheduler
/Memory I/O
\ /
\ /
v v
Tasks
Nếu gom tất cả thành một khối: 
/v v
+--------------------------------+
| |
| HUGE KERNEL BLOCK |
| |
| sched + mm + fs + net + ... |
| |
+--------------------------------+

việc sửa một phần có thể ảnh hưởng rất nhiều phần khác. 
Nhưng nếu tách hoàn toàn: 
Scheduler Memory FS Network
| | | |
X X X X

thì hệ thống cũng không thể hoạt động. 
Vì vậy kernel cần một trạng thái ở giữa: 
Modular
+
Integrated
|
v
Maintainable System
Đây chính là ý: 
Tuy hai mà một, tuy một mà hai. 
 14. Mechanism và Policy 
Một cách quan trọng để hiểu scheduler là phân biệt: 
Mechanism 
Kernel thực hiện việc đó bằng cách nào? 
Ví dụ: 
context switch
preemption
task wakeup
runqueue
timer
CPU migration 
Policy 
Kernel quyết định nên làm gì dựa trên tiêu chí nào? 
Ví dụ: 

priority
fairness
latency
deadline
CPU affinity
load balancing 
Scheduler
|
+---------+---------+
| |
v v
Mechanism Policy
| |
v v
"How?" "What/Which?"
| |
+---------+---------+
|
v
CPU
 15. Scheduler như một bài toán resource allocation / optimization 
Từ góc nhìn hệ thống: 
Scheduler
|
+---------+---------+
| | |
v v v
Latency Throughput Fairness
| | |
+---------+---------+
CPU|
v
utilization
Không có một scheduler luôn tối ưu tất cả mọi thứ cùng lúc. 
Có các trade-off: 
Latency
^
|
|
+-------------------->
Throughput
và: 
Fairness
^
|
|
+--------------------> Priority / responsiveness

Vì vậy scheduling là một bài toán resource allocation với nhiều mục tiêu và constraints. 
 16. Scheduler trong RTOS và GPOS 
RTOS 
Trọng tâm thường là: 
Deterministic timing
|
v
Deadline / priority
|
v
Predictable response

General Purpose OS 
Trọng tâm thường rộng hơn: 
Fairness
Throughput
Latency
Responsiveness
Power
Scalability

Do đó scheduler của RTOS và GPOS có những mục tiêu và trade-off khác nhau. 
 17. Những cơ chế bên dưới scheduler 
“Scheduler” thực tế là tên gọi chung cho một nhóm cơ chế: 
Scheduler
|
+-- runnable task management
|
+-- runqueue|
+-- scheduling policy
|
+-- priority
|
+-- preemption
|
+-- context switch
|
+-- wakeup
|
+-- blocking
|
+-- CPU affinity
|
+-- load balancing
|
+-- timer
|
+-- synchronization

Các cơ chế này được tổ chức thành những subsystem/code path khác nhau nhưng phối hợp rất chặt chẽ. 
 18. Cách nhìn tổng thể 
Application
|
Processes / Threads
|
v
+----------------+
| Synchronization|
+----------------+
|
v
+----------------+
| Scheduler |
+----------------+
|
+-----------+-----------+
| | |
v v v
CPU 0 CPU 1 CPU 2
| | |
+-----------+-----------+
|
v
Hardware

Scheduler lại tương tác với: 
+-------------+
| Scheduler |
+-------------+
/ |MM FS NET
| | |
+-------+-------+
|
v
Drivers
|
v
Hardware/ |vvv

Vì vậy performance của hệ thống không đến từ scheduler riêng lẻ. 
Nó đến từ sự phối hợp của: 
Scheduling
+
Memory Management
+
I/O Management
+
Synchronization
+
Drivers
+
Hardware
+
Application behavior

 19. Kết luận 
Nếu không có scheduler, hệ thống rất khó biến một tập hợp nhiều task concurrent thành một execution model có tổ chức trên CPU. 
Nhưng scheduler không phải cơ chế duy nhất. 
Một OS hiện đại cần nhiều lớp: 
Application
|
v
Concurrency
|
v
Synchronization
|
v
Scheduling
|
v
Memory / I/O / Drivers
|
v
Hardware

Trong đó: 
Synchronization
= ai được phép truy cập resource?
 Scheduler
= task nào được CPU chạy và khi nào?
 Application
= task cần làm gì?
 Hardware
= thực thi instruction như thế nào?

Điểm cốt lõi là: 
Scheduler không trực tiếp giải quyết race condition hay deadlock của application, nhưng scheduler +
synchronization + resource management tạo ra execution environment để application có thể xử lý concurrency vàresource contention một cách có kiểm soát. 
Chính sự phối hợp này cho phép application không phải tự quản lý trực tiếp CPU và hardware, từ đó hệ thống có thể đạt được
throughput cao hơn, latency thấp hơn, responsiveness tốt hơn và tận dụng tài nguyên hiệu quả hơn. 
Vì vậy, khi nghiên cứu scheduler, không nên chỉ hỏi: 
"Scheduler chọn task nào?"

mà nên hỏi: 
Problem
|
v
Resource
|
v
Concurrency
|
v
Synchronization
|
v
Scheduling
|
v
Mechanism + Policy
|
v
Hardware execution
|
v
Performance + Trade-offs

Đó là cách nhìn scheduler như một phần cốt lõi của Operating System, thay vì chỉ xem nó như một function chọn process tiếp theo