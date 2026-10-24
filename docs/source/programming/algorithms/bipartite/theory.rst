===============================================================================
Bipartite Graph (Đồ thị hai phía)
===============================================================================

.. meta::
   :description: Giải thích chi tiết về Bipartite Graph (Đồ thị hai phía), thuật toán kiểm tra, Maximum Matching, Định lý König và các bài toán liên quan.
   :keywords: Bipartite Graph, Đồ thị hai phía, Coloring, Graph Theory, Maximum Matching, Hopcroft-Karp, König, Ford-Fulkerson

.. contents:: **Mục lục**
   :depth: 2
   :local:

---

1. Định nghĩa
=============

**Bipartite graph** là một đồ thị có thể phân chia tập đỉnh thành hai tập
riêng biệt :math:`U` và :math:`V` sao cho:

* Mỗi cạnh nối một đỉnh từ :math:`U` với một đỉnh từ :math:`V`.
* Không có cạnh nào nối hai đỉnh trong cùng một tập.

**Ký hiệu:** :math:`G = (U, V, E)`

Ví dụ
-----

* **Tập U:** ``{A, B, C}``
* **Tập V:** ``{1, 2, 3}``

.. code-block:: text

    U = {A, B, C}          V = {1, 2, 3}

    A ----- 1
    A ----- 2
    B ----- 2
    C ----- 3

Mỗi cạnh luôn nối **một đỉnh của U với một đỉnh của V** — không có cạnh
nội bộ trong U hay trong V.

Ví dụ **không** bipartite — chứa chu trình lẻ (tam giác):

.. code-block:: text

    A --- B
     \   /
      \ /
       C        # Chu trình A → B → C → A có độ dài 3 (lẻ)

---

2. Cách Kiểm Tra Bipartite Graph
================================

Phương pháp: Tô Màu (2-Coloring) — BFS/DFS
------------------------------------------

Tô 2 màu vào các đỉnh. Nếu tô được mà không có cạnh nối 2 đỉnh cùng màu
:math:`\rightarrow` đó là bipartite.

**Ý tưởng:**

#. Bắt đầu từ một đỉnh chưa tô, tô màu ``0``.
#. BFS/DFS sang các đỉnh kề, tô **màu ngược lại** (``1 - color[u]``).
#. Nếu gặp đỉnh kề **cùng màu** → đồ thị **không** bipartite.
#. Lặp lại cho **mỗi connected component** (vì graph có thể rời rạc).

Mã nguồn Python kiểm tra Bipartite Graph:

.. code-block:: python

   from collections import deque

   def is_bipartite(graph):
       color = {}

       def bfs(start):
           color[start] = 0
           queue = deque([start])          # deque: popleft O(1) thay vì pop(0) O(n)

           while queue:
               node = queue.popleft()
               for neighbor in graph[node]:
                   if neighbor not in color:
                       color[neighbor] = 1 - color[node]
                       queue.append(neighbor)
                   elif color[neighbor] == color[node]:
                       return False        # hai đỉnh kề cùng màu
           return True

       # Kiểm tra từng connected component
       for node in graph:
           if node not in color:
               if not bfs(node):
                   return False
       return True

   # Ví dụ sử dụng
   graph = {
       'A': ['1', '2'],
       'B': ['2'],
       'C': ['3'],
       '1': ['A'],
       '2': ['A', 'B'],
       '3': ['C'],
   }
   print(is_bipartite(graph))   # True

**Độ phức tạp:** :math:`O(V + E)` — mỗi đỉnh và mỗi cạnh chỉ được duyệt
một lần; bộ nhớ :math:`O(V)` cho mảng màu và hàng đợi.

Mối liên hệ với chu trình lẻ
----------------------------

BFS phát hiện "cùng màu" chính là phát hiện **chu trình lẻ (odd cycle)**:
khi hai đỉnh kề nhau có cùng màu, tồn tại chu trình đi qua chúng với độ dài
lẻ. Từ đó có **định lý nền tảng:**

.. important::
   Một đồ thị là **bipartite** khi và chỉ khi nó **không chứa chu trình
   có độ dài lẻ**.

---

3. Tính chất
============

* **Bipartite :math:`\Leftrightarrow` Không có chu trình lẻ:** Một đồ thị
  là bipartite khi và chỉ khi nó không chứa chu trình có độ dài lẻ.
* **Chu trình chẵn :math:`\rightarrow` Bipartite** — một chu trình độ dài
  chẵn luôn tô được so le 2 màu.
* **Cây là bipartite** — vì cây không có chu trình (nên đương nhiên không
  có chu trình lẻ). Có thể tô so le theo **độ sâu** (mức chẵn / mức lẻ).
* **Không có self-loop** — self-loop nối đỉnh với chính nó, vi phạm điều
  kiện hai tập.

---

4. Các Bài Toán Quan Trọng
===========================

1. Maximum Bipartite Matching (Ghép cặp tối đa)
-----------------------------------------------
* Tìm số cạnh tối đa sao cho **không có hai cạnh nào dùng chung một đỉnh**.
* **Ứng dụng:** Ghép việc làm cho người, ghép cặp đôi, v.v.
* **Thuật toán:** Hungarian Algorithm (Kuhn's), **Hopcroft–Karp**.

2. Maximum Weighted Matching
----------------------------
* Tương tự nhưng mỗi cạnh có **trọng số**, tìm tổng trọng số tối đa.
* **Thuật toán:** Hungarian Algorithm (Kuhn–Munkres), :math:`O(n^3)`.

3. Minimum Vertex Cover
-----------------------
* Tìm **tập đỉnh nhỏ nhất** chứa ít nhất một đầu mút của mỗi cạnh.
* **Định lý König:** Trong bipartite graph, kích thước vertex cover tối
  thiểu **bằng** kích thước matching tối đa:

   .. math::

      |V_{cover}| = |M_{max}|

* **Hệ quả quan trọng:** :math:`|V| - |M_{max}|` chính là kích thước
  **Maximum Independent Set** (tập đỉnh lớn nhất không có hai đỉnh nào kề
  nhau) — bài toán NP-hard trên đồ thị tổng quát nhưng **giải được đa
  thức** trên bipartite graph.

Khái niệm then chốt: Đường tăng cường (Augmenting Path)
-------------------------------------------------------

Đây là "trái tim" của mọi thuật toán matching:

#. **Matching hiện tại** :math:`M` — tập các cạnh đã ghép.
#. **Đỉnh tự do** — đỉnh chưa được ghép trong :math:`M`.
#. **Augmenting path** — đường đi bắt đầu và kết thúc tại **hai đỉnh tự
   do**, xen kẽ giữa cạnh *chưa ghép* → *đã ghép* → *chưa ghép* → ...
#. **Bước tăng** — đảo trạng thái các cạnh trên đường (chưa ghép ↔ đã
   ghép), số lượng matching **tăng thêm 1**.

.. note::
   **Định lý Berge:** :math:`M` là matching tối đa :math:`\Leftrightarrow`
   không còn augmenting path đối với :math:`M`. Đây là nền tảng chứng minh
   tính đúng đắn của các thuật toán matching.

---

5. Ứng Dụng Thực Tế
====================

.. list-table:: **Ứng dụng của Bipartite Graph**
   :widths: 25 25 25 25
   :header-rows: 1
   :align: center

   * - Bài toán
     - U
     - V
     - Cạnh
   * - **Ghép việc**
     - Người
     - Công việc
     - Khả năng làm
   * - **Hôn nhân**
     - Trai
     - Gái
     - Tương thích
   * - **Hội họp**
     - Người
     - Phòng
     - Sở thích
   * - **Phân công**
     - Tác vụ
     - Máy tính
     - Khả năng

---

6. Ví dụ Code: Maximum Matching (Thuật toán Ford-Fulkerson / DFS)
===================================================================

Ý tưởng: với mỗi đỉnh :math:`u \in U`, dùng DFS tìm **augmenting path**
sang bên :math:`V`. Độ phức tạp: :math:`O(V \times E)`.

.. code-block:: python

   def max_matching(n_u, n_v, edges):
       """
       n_u   : số đỉnh bên U (0 .. n_u-1)
       n_v   : số đỉnh bên V (0 .. n_v-1)
       edges : danh sách kề — edges[u] = list đỉnh v bên V
       Trả về: số cặp ghép tối đa
       """
       # match_v[v] = u  (đỉnh v bên V đã được ghép với u bên U)
       match_v = [-1] * n_v

       def dfs(u, visited):
           for v in edges[u]:
               if visited[v]:
                   continue
               visited[v] = True

               # v còn tự do, HOẶC chủ cũ của v (match_v[v]) tìm được
               # chỗ khác → v "nhường chỗ" cho u  (augmenting path)
               if match_v[v] == -1 or dfs(match_v[v], visited):
                   match_v[v] = u
                   return True
           return False

       result = 0
       for u in range(n_u):
           visited = [False] * n_v
           if dfs(u, visited):
               result += 1
       return result

   # Ví dụ: 3 người, 3 việc
   #   Người 0 làm được việc [0, 1]
   #   Người 1 làm được việc [1, 2]
   #   Người 2 làm được việc [2]
   print(max_matching(3, 3, [[0, 1], [1, 2], [2]]))   # → 3

Với đồ thị lớn, nên dùng **Hopcroft–Karp** (:math:`O(E \sqrt{V})`) — tìm
nhiều augmenting path ngắn nhất trong **một pha** thay vì từng đường một.

.. list-table:: **So sánh các thuật toán trên bipartite graph**
   :widths: 40 30 30
   :header-rows: 1

   * - Bài toán
     - Thuật toán
     - Độ phức tạp
   * - Kiểm tra bipartite
     - BFS/DFS 2-coloring
     - :math:`O(V + E)`
   * - Maximum Matching
     - Hungarian (Kuhn) / Ford–Fulkerson
     - :math:`O(V \times E)`
   * - Maximum Matching (đồ thị lớn)
     - Hopcroft–Karp
     - :math:`O(E \sqrt{V})`
   * - Maximum Weighted Matching
     - Hungarian / Kuhn–Munkres
     - :math:`O(n^3)`
   * - Minimum Vertex Cover
     - Từ max matching (Định lý König)
     - :math:`O(V + E)`