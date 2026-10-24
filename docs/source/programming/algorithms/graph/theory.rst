Graph Algorithms
================

Thuật toán tìm đường đi ngắn nhất, duyệt đồ thị, và các ứng dụng liên quan.

Dijkstra — Thuật toán đường đi ngắn nhất
----------------------------------------

**What is this? (Đây là gì?)**
Dijkstra là thuật toán tham lam (greedy) dùng để tìm đường đi ngắn nhất từ một node xuất phát đến tất cả các node còn lại trong đồ thị có trọng số không âm.

**Where can it be used? (Có thể dùng ở đâu?)**
- GPS navigation — tìm đường đi ngắn nhất giữa hai địa điểm
- Mạng máy tính — định tuyến gói tin (OSPF, IS-IS)
- AI trong game — path finding cho nhân vật
- Mô hình hóa giao thông, logistics — tối ưu lộ trình vận chuyển
- Mạng xã hội — tìm đường kết nối ngắn nhất giữa hai người

**Which circumstance to use? (Dùng trong hoàn cảnh nào?)**
- Đồ thị có trọng số **không âm** (nếu có cạnh âm → dùng Bellman-Ford)
- Cần tìm đường đi ngắn nhất từ một điểm đến nhiều điểm khác
- Dữ liệu có thể biểu diễn dưới dạng node (đỉnh) và edge (cạnh) với chi phí di chuyển

**How to use? (Cách dùng?)**
1. Khởi tạo khoảng cách từ start đến tất cả node là ∞, riêng start = 0
2. Dùng Priority Queue (min-heap) để luôn lấy node có khoảng cách nhỏ nhất
3. Duyệt các neighbor của node đó, cập nhật khoảng cách nếu tìm được đường ngắn hơn
4. Lặp lại cho đến khi queue rỗng
5. Kết quả là khoảng cách ngắn nhất từ start đến mọi node

**When to use? (Khi nào dùng?)**
- Khi bài toán yêu cầu "đường đi ngắn nhất" hoặc "chi phí thấp nhất"
- Khi đồ thị có trọng số dương
- Khi cần kết quả chính xác, không chấp nhận xấp xỉ
- Khi số lượng node lớn, cần độ phức tạp O((V + E) log V)

.. code-block:: python

   import heapq

   def dijkstra(graph, start):
       distances = {node: float('inf') for node in graph}
       distances[start] = 0
       pq = [(0, start)]  # priority queue

       while pq:
           current_dist, node = heapq.heappop(pq)
           if current_dist > distances[node]:
               continue
           for neighbor, weight in graph[node]:
               distance = current_dist + weight
               if distance < distances[neighbor]:
                   distances[neighbor] = distance
                   heapq.heappush(pq, (distance, neighbor))
       return distances