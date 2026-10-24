===============================================================================
Thuật toán tìm kiếm theo chiều sâu (Depth-First Search - DFS)
===============================================================================

.. meta::
   :description: Tài liệu chi tiết về thuật toán Depth-First Search (DFS), ý tưởng cốt lõi, so sánh với BFS, mã giả và sơ đồ minh họa.
   :keywords: DFS, Depth First Search, Graph Theory, Algorithm, Stack, Recursion, Backtracking

1. Tổng quan về Thuật toán DFS
------------------------------

**Depth-First Search (DFS)** là thuật toán tìm kiếm/duyệt cây hoặc đồ thị theo **chiều sâu**.

Khái niệm Kỹ thuật
~~~~~~~~~~~~~~~~~~
* Thuật toán bắt đầu từ một **node nguồn (Start Node)** và tiến hành đi sâu nhất có thể theo một nhánh duy nhất (*Dive to deep*).
* Khi chạm tới đỉnh tận cùng (không còn node lân cận nào chưa duyệt), thuật toán sẽ **quay lùi (Backtrack)** về nút giao gần nhất để thử sang một nhánh mới.

.. note::
   Khác với BFS duyệt theo cơ chế "vết dầu loang", DFS di chuyển như việc đi qua một mê cung: đi sâu vào một con đường cho tới khi gặp đường cụt rồi mới quay lại chọn đường khác.

---

2. So sánh BFS vs DFS
----------------------

Bảng dưới đây tổng hợp các điểm khác biệt cốt lõi giữa hai thuật toán duyệt đồ thị kinh điển:

.. list-table:: **Bảng so sánh chi tiết BFS và DFS**
   :widths: 20 40 40
   :header-rows: 1
   :align: center

   * - Tiêu chí
     - BFS (Breadth-First Search)
     - DFS (Depth-First Search)
   * - **Cấu trúc dữ liệu**
     - Hàng chờ **Queue** (FIFO - First In, First Out)
     - Ngăn xếp **Stack** (LIFO) hoặc **Đệ quy**
   * - **Chiến lược duyệt**
     - Duyệt từng cấp lân cận (*Vết dầu loang*)
     - Đi sâu hết một nhánh (*Dive to deep*)
   * - **Đặc điểm chính**
     - Khám phá các đỉnh theo chiều ngang
     - Khám phá kiệt cùng một nhánh rồi quay lùi (Backtrack)
   * - **Ưu thế bài toán**
     - Tìm đường đi ngắn nhất (Shortest Path)
     - Giải mê cung, phát hiện chu trình, sắp xếp Topo

---

3. Mã giả Thuật toán DFS (Pseudo Code)
--------------------------------------

Mã giả DFS triển khai bằng Đệ quy (Recursion)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

   def dfs_recursive(node, adjacency_matrix, visited):
       # Đánh dấu đỉnh hiện tại là đã duyệt
       visited[node] = True
       print(f"Thăm node: {node}")
       
       # Duyệt qua tất cả các đỉnh kề trong ma trận kề
       for neighbor in range(len(adjacency_matrix[node])):
           # Nếu có cạnh nối và đỉnh kề chưa được thăm
           if adjacency_matrix[node][neighbor] == 1 and not visited[neighbor]:
               # Đệ quy đi sâu vào đỉnh kề đó
               dfs_recursive(neighbor, adjacency_matrix, visited)

Mã giả DFS triển khai bằng Stack (Iterative)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

   def dfs_stack(start_node, adjacency_matrix):
       visited = [False] * len(adjacency_matrix)
       stack = [start_node]
       
       while len(stack) > 0:
           # Lấy đỉnh ở top of stack ra (LIFO)
           node = stack.pop()
           
           if not visited[node]:
               visited[node] = True
               print(f"Thăm node: {node}")
               
               # Đưa các node kề chưa thăm vào stack
               for neighbor in range(len(adjacency_matrix[node]) - 1, -1, -1):
                   if adjacency_matrix[node][neighbor] == 1 and not visited[neighbor]:
                       stack.append(neighbor)

---

4. Mô phỏng Sơ đồ Tiến trình Duyệt DFS
--------------------------------------

Sơ đồ Đồ thị Mẫu (5 Đỉnh)
~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: text

        (0)-------------* (1)
         |                 |
         |                 |
        *|                 |
        (3)------------- (2)
          \               /
           \             /
            \           /
             --- (4) ---

Sơ đồ Luồng di chuyển "Dive to Deep & Backtrack"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Dưới đây là mô phỏng luồng đi sâu hết một nhánh và quay đầu (giả sử xuất phát từ nút ``0``):

.. code-block:: text

   [Bước 1] Dive in:   (0) ----> (1) ----> (2) ----> (4) ----> (3)
                                                                 |
   [Bước 2] Trạng thái: (3) Không còn node kề chưa thăm (Đường cụt!)
                                                                 |
   [Bước 3] Backtrack: (3) --quay lùi--> (4) --quay lùi--> (2) --quay lùi--> (0)
                                                                 |
   [Bước 4] Kết thúc: Tất cả đỉnh đã được đánh dấu trong visited[].

Mô phỏng Trạng thái Stack qua từng bước
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: text

     +---+         +---+         +---+         +---+         +---+
     | 3 |         | 4 |         | 2 |         | 1 |         | 0 |
     +---+         +---+         +---+         +---+         +---+
     Stack (1)     Stack (2)     Stack (3)     Stack (4)     Stack (5)
     [Start 0]     [Push 1]      [Push 2]      [Push 4]      [Push 3]
         |             |             |             |             |
         v             v             v             v             v
     Thăm (0)      Thăm (1)      Thăm (2)      Thăm (4)      Thăm (3) -> POP ALL & EXIT