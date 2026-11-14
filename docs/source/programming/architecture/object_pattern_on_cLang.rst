[UDEMY] Object Pattern trong Lập Trình C
========================================

.. role:: bolditalic
   :class: bolditalic

.. contents:: Mục lục
   :local:
   :depth: 2

.. rubric:: 1. Definition (Định nghĩa)

**Object Pattern** trong ngôn ngữ C là việc gom nhóm một tập hợp các dữ liệu thành một tổ chức có cấu trúc (hierarchical structure) đi kèm với các hàm (functions) thao tác/sửa đổi (modify) chính các cấu trúc dữ liệu đó.

**Tóm tắt:** Thay vì quản lý dữ liệu rời rạc, ta gom chúng vào ``struct`` và cung cấp các hàm để thao tác an toàn.

Nguyên tắc chính:

  * **Xử lý theo luồng (Data Flow):** Dữ liệu được xử lý rõ ràng theo luồng, tránh tham chiếu (reference) đến các giá trị nằm ngoài phạm vi (scope) của hàm.
  * **Tính nhất quán (Thread Safety/Reentrancy):** Dựa trên nguyên tắc này, nếu 2 thread cùng gọi một hàm với cùng một đầu vào (*input*), dữ liệu đầu ra (*output*) thu được sẽ hoàn toàn như nhau.
  * **Sạch mã nguồn (Clean Code):** Mang lại lợi ích vô cùng to lớn (*tremendous*) cho việc duy trì và quản lý code. Khi dữ liệu được thay đổi, nó sẽ thực hiện trực tiếp thông qua một hàm xác định, vì dữ liệu nằm chính trong *Object* và được truyền dưới dạng đối số (*argument*) vào trong từng hàm.

Quy tắc cốt lõi:

  1. **Context** được truyền xuyên suốt thông qua các đối số (*arguments / parameters*).
  2. **Dữ liệu** tuyệt đối không được phép truy cập toàn cục (*globally*).
  3. **Function** không nên chứa biến tĩnh (*static variables*).
  4. **Data flow** đi theo một luồng duy nhất và thống nhất.

**Ví dụ minh họa:**

.. code-block:: c

   // ❌ Bad: Global variables
   int g_counter;
   
   void increment(void) {
       g_counter++;  // Không rõ ràng, khó debug
   }

   // ✅ Good: Object Pattern
   typedef struct {
       int counter;
   } Counter;
   
   void counter_increment(Counter *self) {
       self->counter++;  // Rõ ràng, dễ test
   }

.. rubric:: 2. Use Cases (Trường hợp sử dụng)

Thiết kế này nên được áp dụng cho **tất cả các components** trong một chương trình viết bằng C. Khi viết một hàm, lập trình viên cần xác định rõ:

  *Hàm này được thiết kế để thực hiện hành động chung (generic) hay dành riêng cho một Object cụ thể nào?*

Object Pattern đóng vai trò là nền tảng hỗ trợ cho hầu hết các Design Pattern khác trong C. Trong mỗi Object thường bao gồm các thành phần sau:

2.1. Grouping Data (Gom nhóm dữ liệu)


**Mục đích:** Gom các biến liên quan vào một ``struct`` duy nhất để dễ quản lý.

**Ví dụ thực tế:**

.. code-block:: c

   // ❌ Before: Biến rời rạc - khó maintain
   char user_name[50];
   int user_age;
   float user_balance;
   int user_id;
   
   void set_user_name(char *name) {
       strcpy(user_name, name);
   }
   
   void set_user_age(int age) {
       user_age = age;
   }

   // ✅ After: Gom nhóm vào Struct
   typedef struct {
       char name[50];
       int age;
       float balance;
       int id;
   } User;
   
   void user_set_name(User *self, char *name) {
       strcpy(self->name, name);
   }
   
   void user_set_age(User *self, int age) {
       self->age = age;
   }
   
   // Sử dụng
   int main(void) {
       User user;
       user_set_name(&user, "John");
       user_set_age(&user, 30);
       return 0;
   }

**Lợi ích:**

  * Dễ dàng truyền dữ liệu giữa các hàm
  * Code dễ đọc và maintain
  * Giảm thiểu lỗi do nhầm tham số

2.2. Singletons


**Mục đích:** Gom các biến tĩnh (``static``) vào một ``struct`` duy nhất, tránh khai báo rải rác.

**Ví dụ thực tế - System Configuration:**

.. code-block:: c

   // ❌ Bad: Nhiều biến static rời rạc
   static int g_system_mode;
   static int g_debug_level;
   static char g_log_path[256];
   static int g_max_connections;
   
   void set_mode(int mode) {
       g_system_mode = mode;
   }

   // ✅ Good: Gom vào Singleton struct
   typedef struct {
       int mode;
       int debug_level;
       char log_path[256];
       int max_connections;
   } SystemConfig;
   
   // Khai báo singleton trong .c file
   static SystemConfig g_config = {
       .mode = 0,
       .debug_level = 1,
       .log_path = "/var/log/app.log",
       .max_connections = 100
   };
   
   // Public API
   void config_set_mode(int mode) {
       g_config.mode = mode;
   }
   
   int config_get_mode(void) {
       return g_config.mode;
   }
   
   // Sử dụng
   int main(void) {
       config_set_mode(2);
       printf("Mode: %d\n", config_get_mode());
       return 0;
   }

**Lợi ích:**

  * Tập trung cấu hình vào một nơi
  * Dễ dàng thay đổi cấu trúc mà không ảnh hưởng API
  * Tránh namespace pollution

2.3. Abstract Interfaces (Trừu tượng hóa giao diện)

**Mục đích:** Ẩn chi tiết implementation, chỉ expose API công khai.

**Ví dụ thực tế - Stack Implementation:**

.. code-block:: c

   // stack.h - Public Interface
   #ifndef STACK_H
   #define STACK_H
   
   typedef struct Stack Stack;
   
   // Constructor/Destructor
   Stack* stack_create(int capacity);
   void stack_destroy(Stack *self);
   
   // Public API
   int stack_push(Stack *self, int value);
   int stack_pop(Stack *self, int *value);
   int stack_is_empty(Stack *self);
   int stack_is_full(Stack *self);
   
   #endif

.. code-block:: c

   // stack.c - Private Implementation
   #include "stack.h"
   #include <stdlib.h>
   #include <stdio.h>
   
   // Struct definition PRIVATE
   struct Stack {
       int *data;
       int top;
       int capacity;
   };
   
   Stack* stack_create(int capacity) {
       Stack *stack = malloc(sizeof(Stack));
       if (!stack) return NULL;
       
       stack->data = malloc(capacity * sizeof(int));
       if (!stack->data) {
           free(stack);
           return NULL;
       }
       
       stack->top = -1;
       stack->capacity = capacity;
       return stack;
   }
   
   void stack_destroy(Stack *self) {
       if (self) {
           free(self->data);
           free(self);
       }
   }
   
   int stack_push(Stack *self, int value) {
       if (stack_is_full(self)) {
           fprintf(stderr, "Stack overflow\n");
           return -1;
       }
       self->data[++self->top] = value;
       return 0;
   }
   
   int stack_pop(Stack *self, int *value) {
       if (stack_is_empty(self)) {
           fprintf(stderr, "Stack underflow\n");
           return -1;
       }
       *value = self->data[self->top--];
       return 0;
   }
   
   int stack_is_empty(Stack *self) {
       return self->top == -1;
   }
   
   int stack_is_full(Stack *self) {
       return self->top == self->capacity - 1;
   }

**Sử dụng:**

.. code-block:: c

   // main.c
   #include "stack.h"
   
   int main(void) {
       // Tạo stack với capacity 10
       Stack *stack = stack_create(10);
       
       // Push giá trị
       stack_push(stack, 10);
       stack_push(stack, 20);
       stack_push(stack, 30);
       
       // Pop giá trị
       int value;
       stack_pop(stack, &value);
       printf("Popped: %d\n", value);  // Output: 30
       
       // Cleanup
       stack_destroy(stack);
       return 0;
   }

**Lợi ích:**

  * Che giấu chi tiết implementation
  * Có thể thay đổi internal structure mà không ảnh hưởng user code
  * Dễ dàng testing và mocking

2.4. Multi-threaded Design (Thiết kế đa luồng)

**Mục đích:** Đảm bảo thread safety với cơ chế "Lock Data, not Code".

**Ví dụ thực tế - Thread-safe Counter:**

.. code-block:: c

   #include <pthread.h>
   
   typedef struct {
       int count;
       pthread_mutex_t mutex;  // Lock thuộc về data
   } ThreadSafeCounter;
   
   // Khởi tạo
   void counter_init(ThreadSafeCounter *self) {
       self->count = 0;
       pthread_mutex_init(&self->mutex, NULL);
   }
   
   // Hủy
   void counter_destroy(ThreadSafeCounter *self) {
       pthread_mutex_destroy(&self->mutex);
   }
   
   // Thread-safe increment
   void counter_increment(ThreadSafeCounter *self) {
       pthread_mutex_lock(&self->mutex);  // Lock data
       self->count++;
       pthread_mutex_unlock(&self->mutex);  // Unlock data
   }
   
   // Thread-safe get
   int counter_get(ThreadSafeCounter *self) {
       pthread_mutex_lock(&self->mutex);
       int value = self->count;
       pthread_mutex_unlock(&self->mutex);
       return value;
   }
   
   // Sử dụng trong multi-thread
   void* worker(void *arg) {
       ThreadSafeCounter *counter = (ThreadSafeCounter *)arg;
       
       for (int i = 0; i < 1000; i++) {
           counter_increment(counter);
       }
       
       return NULL;
   }
   
   int main(void) {
       pthread_t t1, t2;
       ThreadSafeCounter counter;
       
       counter_init(&counter);
       
       // Tạo 2 threads cùng increment
       pthread_create(&t1, NULL, worker, &counter);
       pthread_create(&t2, NULL, worker, &counter);
       
       pthread_join(t1, NULL);
       pthread_join(t2, NULL);
       
       printf("Final count: %d\n", counter_get(&counter));
       // Output: Final count: 2000
       
       counter_destroy(&counter);
       return 0;
   }

**Nguyên tắc "Lock Data, not Code":**

  * Mỗi Object chứa mutex của riêng nó
  * Lock/Unlock tại cấp Object, không phải function
  * Tránh deadlock bằng cách giữ lock thật ngắn

2.5. Opaque Handles (Con trỏ ẩn)

**Mục đích:** Che giấu chi tiết implementation, chỉ expose handle (con trỏ).

**Ví dụ thực tế - Database Connection:**

.. code-block:: c

   // db.h - Public API
   #ifndef DB_H
   #define DB_H
   
   typedef struct DBConnection DBConnection;
   
   // Lifecycle
   DBConnection* db_connect(const char *host, int port);
   void db_disconnect(DBConnection *conn);
   
   // Operations
   int db_execute(DBConnection *conn, const char *query);
   int db_query(DBConnection *conn, const char *query, void **result);
   
   #endif

.. code-block:: c

   // db.c - Private Implementation
   #include "db.h"
   #include <stdlib.h>
   
   // Chi tiết internal - USER KHÔNG THẤY
   struct DBConnection {
       int socket_fd;
       char host[256];
       int port;
       int connected;
       // ... các field private khác
   };
   
   DBConnection* db_connect(const char *host, int port) {
       DBConnection *conn = malloc(sizeof(DBConnection));
       if (!conn) return NULL;
       
       strcpy(conn->host, host);
       conn->port = port;
       conn->connected = 0;
       
       // Logic connect...
       // conn->socket_fd = socket(...);
       // connect(...);
       
       conn->connected = 1;
       return conn;
   }
   
   void db_disconnect(DBConnection *conn) {
       if (conn) {
           // close(conn->socket_fd);
           free(conn);
       }
   }
   
   int db_execute(DBConnection *conn, const char *query) {
       if (!conn || !conn->connected) {
           return -1;
       }
       // Execute query...
       return 0;
   }

**Sử dụng:**

.. code-block:: c

   #include "db.h"
   
   int main(void) {
       // User chỉ thấy DBConnection*, không biết chi tiết bên trong
       DBConnection *db = db_connect("localhost", 5432);
       
       db_execute(db, "SELECT * FROM users");
       
       db_disconnect(db);
       return 0;
   }

**Lợi ích:**

  * Tách biệt hoàn toàn implementation và interface
  * Có thể thay đổi internal structure tự do
  * Binary compatibility - không cần recompile user code
  * Encapsulation đúng nghĩa

.. rubric:: 3. Best Practices (Thực hành tốt)

3.1. Naming Convention


.. code-block:: c

   typedef struct {
       int value;
   } MyObject;
   
   // Functions: object_method(self, ...)
   void myobject_init(MyObject *self);
   void myobject_set_value(MyObject *self, int value);
   int myobject_get_value(MyObject *self);
   void myobject_cleanup(MyObject *self);

3.2. Error Handling


.. code-block:: c

   typedef struct {
       int result;
       char error_msg[256];
   } Result;
   
   Result operation_do_something(Object *self, int param) {
       Result res;
       
       if (!self) {
           res.result = -1;
           strcpy(res.error_msg, "Invalid object");
           return res;
       }
       
       if (param < 0) {
           res.result = -2;
           strcpy(res.error_msg, "Invalid parameter");
           return res;
       }
       
       // Do work...
       res.result = 0;
       strcpy(res.error_msg, "Success");
       return res;
   }

3.3. Memory Management


.. code-block:: c

   // Constructor - returns allocated object
   Object* object_create(void) {
       Object *obj = malloc(sizeof(Object));
       if (!obj) return NULL;
       
       // Initialize
       obj->data = malloc(100);
       if (!obj->data) {
           free(obj);
           return NULL;
       }
       
       return obj;
   }
   
   // Destructor - frees all resources
   void object_destroy(Object *self) {
       if (self) {
           free(self->data);
           free(self);
       }
   }

.. rubric:: 4. Summary (Tóm tắt)

**Object Pattern** là foundation cho clean code trong C:

  * **Grouping Data:** Gom biến liên quan vào struct
  * **Singletons:** Quản lý global state tập trung
  * **Abstract Interfaces:** Che giấu implementation
  * **Multi-threading:** Thread-safe với "Lock Data"
  * **Opaque Handles:** Encapsulation hoàn toàn

**Lợi ích chính:**

  * Code dễ đọc, maintain, và test
  * Thread-safe by design
  * Module hóa rõ ràng
  * Dễ dàng refactor và mở rộng

**Khi nào sử dụng:**

  * Khi codebase lớn, nhiều developers
  * Khi cần thread safety
  * Khi xây dựng library/framework
  * Khi muốn clean architecture

Object Pattern là bước đầu tiên để viết code C chuyên nghiệp, scalable, và maintainable.

.. rubric:: C Object Design Principles Notes

**Question 1:** Why is it so important to avoid static variables inside functions in C? Especially if the function is an object method?

**Static variables inside functions**

- Tránh các hành vi thay đổi giá trị của biến một cách bất thường mà ta không kiểm soát được.

- Bên cạnh đó, việc sử dụng các biến ``static`` làm mất đi tính **isolate**.

- Cả hai object đều có thể tác động và thay đổi giá trị của ``static``.

- Điều này làm mất tính module hóa.

- Trạng thái của object là duy nhất, chứ không chia sẻ.

**Object method considerations**

Với function là một object method thì:

- Thường được đặt tên với prefix là object đó.
- Các thông số thường được truyền thông qua các parameter.

Mục đích:

- Đưa dữ liệu vào luồng xử lý.
- Xử lý các mutex một cách hợp lý.

Điều tối kị:

- Cùng một function và cùng parameter truyền vào mà lại có 2 output khác nhau.
- Điều này dẫn đến unwanted behavior ngầm.

**Question 2:** Why do we avoid functions without parameters? What negative property do these functions possess that make them a very bad design flaw in C source code?

Function được build mà không có parameter:

- Thậm chí không dùng parameter cũng vẫn nên truyền.
- Nếu không cần thì optimize sau.

Lý do:

- Code không được tường minh.

**Explicit context passing**

Quy tắc:

> Toàn bộ ngữ cảnh nên được truyền toàn bộ cho hàm thông qua các đối số để làm rõ, tường minh code.

Những function có argument là ``void``:

- Thường truy xuất các static variable.
- Do đó khá phiền khi ta không biết đang xử lý trên object nào.

**Key questions for object function design:**

Thiết kế function object phải trả lời được các câu hỏi:

**Which object does it work for?**

- Object nào đang sở hữu function này?

**Buffer allocation strategy**

Cần xác định:

- Resource ownership.
- Lifetime của dữ liệu.

**State changes**

Cần biết:

- Function tác động đến state nào.
- Side effect nào xảy ra.

**Required resources**

Việc isolate các function giúp:

- Dễ dàng tạo dummy object để test.
- Dễ hơn so với các function phụ thuộc quá nhiều vào static variable.

**Question 3:** Why do we call our pointer to context 'self'? Why should we avoid using other names to refer to 'self'?

``self`` để nội ám chỉ rằng:

- Cấu trúc dữ liệu là thuộc lớp design này.
- Không được modify ở chỗ nào khác.

Mang ý nghĩa giống như:

- Private method trong C++.

Nhưng khác:

- Trong C++ có chức năng language-level.
- Trong C chỉ là convention.

**Naming convention**

Chữ "should" thể hiện:

- Nên sử dụng.
- Không hẳn là bắt buộc.

Việc sử dụng ``self`` giúp:

- Người khác dễ hình dung workflow của cấu trúc dữ liệu.
- Dễ maintain hơn.

**Question 4:** Why is it sometimes necessary to instantiate objects locally in the C file as singletons?

**Singleton definition**

Singleton được giới thiệu như là một method giúp:

- Tránh overlap các declaration ở nhiều nơi.
- Khiến code khó kiểm soát.

Tuy nhiên trong C:

- Việc khai báo singleton có hơi khác một chút.

Về cơ bản:

- Chống misunderstanding giữa các bên khi sử dụng object.

**Purpose of singleton**

Mục đích sử dụng của singleton:

- Dành cho các thông tin chỉ nên có một lần duy nhất trong suốt quá trình chạy (bật điện).

Ví dụ:

- System status.

**Question 5:** Why is it sometimes necessary to only expose a pointer to the data structure outside of the implementing C file?

**Opaque structure pattern**

Phổ biến khi:

- Ta muốn ẩn các file implement vào trong file ``.c``.

Trong file ``.h``:

.. code-block:: c

   typedef struct Timer Timer;

Trong file ``.c``:

.. code-block:: c

   struct Timer
   {

   };

**Purpose**

Trong trường hợp này:

- Ta chỉ muốn nhắc nhở rằng hãy sử dụng các hàm để set/get giá trị.
- Thay vì can thiệp trực tiếp giá trị đó vào struct.

**Benefits**

**Encapsulation (tính đóng gói)**

- Data được kiểm soát thông qua interface.

**Information hiding**

- User không cần biết internal implementation.

**Giảm coupling**

- Internal implementation có thể thay đổi.

**User không thể modify trực tiếp data trong struct**

Vì:

- Không biết cấu trúc bên trong.

Tuy nhiên:

- Nếu biết thì vẫn có thể edit bình thường.

**Memory allocation**

Opaque object giúp:

- Custom được memory allocation.

Có thể kiểm soát:

- Cách cấp phát memory.
- Resource ownership.

**Question 6:** Why is it a good practice to always name the header and the C file with the same name as the data object they implement?

**Naming convention**

Để xác định:

- Các object liên quan tới function.
- Hỗ trợ trong refactor.
- Hỗ trợ test code.

**Separation benefits**

Giúp:

- Phân chia rõ ràng giữa method và instance.

Ví dụ:

Object:

::

   Timer

File:

::

   Timer.h
   Timer.c

**Question 7:** Why should you never use 'extern' declared variables anywhere in your C code?

**Avoid extern mutable state**

Không nên expose mutable module state thông qua:

.. code-block:: c

   extern

variables.

**Problem**

Khi sử dụng:

.. code-block:: c

   extern variable

thì:

- Bất kỳ module nào cũng có thể thay đổi giá trị.
- Khó kiểm soát ai đang modify state.

**Better approach**

Prefer:

- Functions.
- Object interfaces.

Để:

- Access đến state được kiểm soát.
- Giữ được module boundary.

**Essential principles:**

- Avoid hidden state.
- Keep object state unique.
- Maintain isolation.
- Pass context explicitly.
- Hide implementation details.
- Reduce coupling.
- Control access to mutable data.

.. rubric:: 5. References

- `Embedded C Programming - Design Patterns Udemy <https://samsungu.udemy.com/course/embedded-c-programming-design-patterns/learn/lecture/36401278#overview>`_ - Lesson 1: Object Pattern trong Lập Trình C
