Introduction Challenge
======================

.. rubric:: Mục tiêu


Trong 30 ngày, xây dựng nền tảng thuật toán vững chắc, rèn luyện tư duy
giải quyết vấn đề và hình thành khả năng tự lựa chọn thuật toán phù hợp
khi gặp một bài toán mới.

Trọng tâm không phải là giải thật nhiều bài, mà là hiểu:

* **Nguyên lý hoạt động** của thuật toán.
* Vì sao thuật toán **phù hợp** với một dạng bài toán.
* Cách **tự cài đặt thuật toán** từ đầu.
* Cách phân tích **độ phức tạp** thời gian và **bộ nhớ**.
* Cách nhận diện những mẫu bài toán quen thuộc.
* Cách **cải thiện lời giải** ban đầu.

**Thời lượng:** 30 ngày, khoảng 2 đến 3 giờ mỗi ngày.

**Mục tiêu bài tập:** Khoảng 35 đến 45 bài LeetCode.

**Ngôn ngữ:** C++17.

.. rubric:: Kết quả kỳ vọng


Sau 30 ngày, mục tiêu là:

* Hiểu các **kỹ thuật thuật toán** nền tảng.
* Tự cài đặt được những thuật toán cốt lõi.
* **Giải độc lập** phần lớn bài Easy thuộc các chủ đề đã học.
* **Giải được** phần lớn bài Medium có dạng quen thuộc.
* Hiểu **hướng tiếp cận** của một số bài Hard.
* Có quy trình phân tích bài toán rõ ràng trước khi viết mã.

.. rubric:: Tổng quan lộ trình


.. list-table::
   :header-rows: 1
   :widths: 12 18 70

   * - Giai đoạn
     - Thời gian
     - Trọng tâm
   * - 1
     - Ngày 1 đến 7
     - Nền tảng, BFS và Flood Fill
   * - 2
     - Ngày 8 đến 14
     - Brute Force, đệ quy và Backtracking
   * - 3
     - Ngày 15 đến 21
     - Dynamic Programming
   * - 4
     - Ngày 22 đến 30
     - Thuật toán nâng cao và luyện tập tổng hợp

.. rubric:: GIAI ĐOẠN 1 - NỀN TẢNG, BFS VÀ FLOOD FILL


.. rubric:: Thời gian: Ngày 1 đến ngày 7


.. rubric:: Mục tiêu


Nắm được cách phân tích độ phức tạp, biểu diễn đồ thị, duyệt BFS và xử
lý các bài toán trên ma trận.

.. rubric:: Ngày 1 - Phân tích độ phức tạp


**Nội dung**

* Độ phức tạp thời gian.
* Độ phức tạp bộ nhớ.
* Ký hiệu Big-O.
* Trường hợp tốt nhất, trung bình và xấu nhất.
* Các thao tác cơ bản trên mảng.

**Bài tập**

* `Two Sum #1 <https://leetcode.com/problems/two-sum/>`_
* `Contains Duplicate #217 <https://leetcode.com/problems/contains-duplicate/>`_

.. rubric:: Ngày 2 - Bài toán cơ bản


**Nội dung**

* Phân tích đề bài.
* **Xử lý mảng và chuỗi**.
* Bảng băm.
* Nhận diện các mẫu bài toán đơn giản.

**Bài tập**

* `Valid Anagram #242 <https://leetcode.com/problems/valid-anagram/>`_
* `Best Time to Buy and Sell Stock #121 <https://leetcode.com/problems/best-time-to-buy-and-sell-stock/>`_

.. rubric:: Ngày 3 - Brute Force


Tại sao lại giải brute force trong khi ta biết nó không tối ưu và rất mất thời gian trong 
việc xác định các trường hợp?
- BF để viết hàm chuẩn, đơn giản và chắc đúng trong nhiều trường hợp
- Stress test với thuật toán
- Hiểu rõ bức tranh toàn cảnh của hệ thống
- Tránh analysis paralysis (bế tắc)

**Nội dung**

* Tìm kiếm vét cạn.
* Liệt kê các khả năng.
* Xác định **điểm nghẽn tính toán**.
* Tối ưu lời giải ban đầu.

**Bài tập**
Brute force:
- [e] #1, #728
- [m] #78, #46
- [h] #79, #56

* `Maximum Subarray #53 <https://leetcode.com/problems/maximum-subarray/>`_
* `3Sum #15 <https://leetcode.com/problems/3sum/>`_

.. rubric:: Ngày 4 - Breadth First Search (BFS)


**Nội dung**

* Biểu diễn đồ thị.
* Cấu trúc dữ liệu Queue.
* Tập hợp các đỉnh đã thăm.
* Duyệt đồ thị bằng BFS.
* Tìm đường đi ngắn nhất trên đồ thị không trọng số.

**Bài tập**

* `Binary Tree Level Order Traversal #102 <https://leetcode.com/problems/binary-tree-level-order-traversal/>`_

.. rubric:: Ngày 5 - Flood Fill


**Nội dung**

* Biểu diễn ma trận.
* Thành phần liên thông.
* So sánh BFS và DFS.
* Quản lý các ô đã thăm.
* Duyệt ma trận.

**Bài tập**

* `Flood Fill #733 <https://leetcode.com/problems/flood-fill/>`_
* `Number of Islands #200 <https://leetcode.com/problems/number-of-islands/>`_

.. rubric:: Ngày 6 - BFS trên ma trận


**Nội dung**

* BFS nhiều nguồn.
* Tìm đường đi ngắn nhất trên ma trận.
* Theo dõi khoảng cách.
* Xử lý điều kiện biên.

**Bài tập**

* `Rotting Oranges #994 <https://leetcode.com/problems/rotting-oranges/>`_
* `01 Matrix #542 <https://leetcode.com/problems/01-matrix/>`_

.. rubric:: Ngày 7 - Ôn tập và đánh giá


**Bài tập**

* `Shortest Path in Binary Matrix #1091 <https://leetcode.com/problems/shortest-path-in-binary-matrix/>`_

**Danh sách kiểm tra**

* [ ] Hiểu và tự cài đặt được BFS.
* [ ] Giải thích được vì sao BFS tìm đường đi ngắn nhất trên đồ thị không trọng số.
* [ ] Phân biệt được BFS và DFS.
* [ ] Biết khi nào cần đánh dấu một đỉnh là đã thăm.
* [ ] Phân tích được độ phức tạp O(V + E).
* [ ] Phân tích được độ phức tạp O(M x N) khi duyệt ma trận.

.. rubric:: GIAI ĐOẠN 2 - BRUTE FORCE VÀ BACKTRACKING


.. rubric:: Thời gian: Ngày 8 đến ngày 14


.. rubric:: Mục tiêu


Hiểu đệ quy, cây tìm kiếm, quản lý trạng thái, quay lui và kỹ thuật cắt
tỉa nhánh tìm kiếm.

.. rubric:: Ngày 8 - Nền tảng đệ quy


**Nội dung**

* Hàm đệ quy.
* Điều kiện dừng.
* Ngăn xếp lời gọi hàm.
* Cây đệ quy.
* Phân tích độ phức tạp của đệ quy.

**Bài tập**

* `Reverse Linked List #206 <https://leetcode.com/problems/reverse-linked-list/>`_
* `Maximum Depth of Binary Tree #104 <https://leetcode.com/problems/maximum-depth-of-binary-tree/>`_

.. rubric:: Ngày 9 - Tập con


**Nội dung**

* Sinh các tập con.
* Lựa chọn hoặc không lựa chọn phần tử.
* Cây quyết định nhị phân.

**Bài tập**

* `Subsets #78 <https://leetcode.com/problems/subsets/>`_

.. rubric:: Ngày 10 - Tổ hợp


**Nội dung**

* Sinh tổ hợp.
* Không gian tìm kiếm.
* Tránh lựa chọn trùng lặp.

**Bài tập**

* `Combinations #77 <https://leetcode.com/problems/combinations/>`_

.. rubric:: Ngày 11 - Hoán vị


**Nội dung**

* Sinh hoán vị.
* Theo dõi trạng thái.
* Mảng đánh dấu phần tử đã sử dụng.
* Duyệt các lựa chọn bằng đệ quy.

**Bài tập**

* `Permutations #46 <https://leetcode.com/problems/permutations/>`_

.. rubric:: Ngày 12 - Backtracking


**Nội dung**

* Khung thuật toán quay lui.
* Chọn, khám phá và hoàn tác lựa chọn.
* Khôi phục trạng thái sau mỗi nhánh tìm kiếm.

**Bài tập**

* `Combination Sum #39 <https://leetcode.com/problems/combination-sum/>`_

.. rubric:: Ngày 13 - Pruning


**Nội dung**

* Thu hẹp không gian tìm kiếm.
* Kiểm tra ràng buộc.
* Dừng sớm khi không còn khả năng tạo lời giải.
* Xử lý phần tử trùng lặp.

**Bài tập**

* `Word Search #79 <https://leetcode.com/problems/word-search/>`_
* `Subsets II #90 <https://leetcode.com/problems/subsets-ii/>`_

.. rubric:: Ngày 14 - Ôn tập và đánh giá


**Bài tập**

* `N-Queens #51 <https://leetcode.com/problems/n-queens/>`_

**Danh sách kiểm tra**

* [ ] Hiểu đệ quy và ngăn xếp lời gọi hàm.
* [ ] Tự xây dựng được cây quyết định đệ quy.
* [ ] Phân biệt Brute Force và Backtracking.
* [ ] Tự cài đặt bài toán tập con và hoán vị.
* [ ] Hiểu cách quản lý trạng thái.
* [ ] Biết khi nào cần hoàn tác một lựa chọn.
* [ ] Áp dụng được kỹ thuật cắt tỉa.
* [ ] Phân tích được độ phức tạp của thuật toán đệ quy.

.. rubric:: GIAI ĐOẠN 3 - DYNAMIC PROGRAMMING


.. rubric:: Thời gian: Ngày 15 đến ngày 21


.. rubric:: Mục tiêu


Rèn luyện khả năng nhận diện bài toán con, định nghĩa trạng thái DP,
xây dựng công thức chuyển trạng thái và tối ưu bộ nhớ.

Đây là giai đoạn cần được đầu tư nhiều thời gian. Khó khăn lớn nhất của
Dynamic Programming thường không nằm ở việc viết mã, mà nằm ở việc xác
định đúng trạng thái và công thức chuyển trạng thái.

.. rubric:: Ngày 15 - Giới thiệu Dynamic Programming


**Nội dung**

* Bài toán con chồng lặp.
* Cấu trúc con tối ưu.
* Đệ quy và quy hoạch động.
* Memoization.
* Tabulation.

**Bài tập**

* `Climbing Stairs #70 <https://leetcode.com/problems/climbing-stairs/>`_

.. rubric:: Ngày 16 - Quy hoạch động một chiều


**Nội dung**

* DP một chiều.
* Định nghĩa trạng thái.
* Công thức chuyển trạng thái.
* Trạng thái cơ sở.

**Bài tập**

* `House Robber #198 <https://leetcode.com/problems/house-robber/>`_
* `House Robber II #213 <https://leetcode.com/problems/house-robber-ii/>`_

.. rubric:: Ngày 17 - Tối ưu quy hoạch động


**Nội dung**

* Giảm số lượng trạng thái cần lưu.
* Tối ưu bộ nhớ.
* Thuật toán Kadane.
* Sử dụng các biến cuộn.

**Bài tập**

* `Maximum Subarray #53 <https://leetcode.com/problems/maximum-subarray/>`_
* `Min Cost Climbing Stairs #746 <https://leetcode.com/problems/min-cost-climbing-stairs/>`_

.. rubric:: Ngày 18 - Quy hoạch động trên ma trận


**Nội dung**

* DP hai chiều.
* Chuyển trạng thái trên ma trận.
* Đếm số đường đi.
* Tìm đường đi có chi phí nhỏ nhất.

**Bài tập**

* `Unique Paths #62 <https://leetcode.com/problems/unique-paths/>`_
* `Minimum Path Sum #64 <https://leetcode.com/problems/minimum-path-sum/>`_

.. rubric:: Ngày 19 - Knapsack


**Nội dung**

* Bài toán ba lô 0/1.
* Bài toán ba lô không giới hạn.
* Chuyển trạng thái.
* Ràng buộc sức chứa.

**Bài tập**

* `Coin Change #322 <https://leetcode.com/problems/coin-change/>`_

.. rubric:: Ngày 20 - Quy hoạch động trên dãy


**Nội dung**

* Tối ưu trên dãy phần tử.
* Dãy con tăng dài nhất.
* Quan hệ giữa các trạng thái.
* Tối ưu độ phức tạp.

**Bài tập**

* `Longest Increasing Subsequence #300 <https://leetcode.com/problems/longest-increasing-subsequence/>`_

.. rubric:: Ngày 21 - Quy hoạch động hai chiều


**Nội dung**

* Trạng thái hai chiều.
* Quy hoạch động trên chuỗi.
* So sánh các dãy.
* Tối ưu công thức chuyển trạng thái.

**Bài tập**

* `Longest Common Subsequence #1143 <https://leetcode.com/problems/longest-common-subsequence/>`_

**Danh sách kiểm tra**

* [ ] Hiểu bài toán con chồng lặp.
* [ ] Hiểu cấu trúc con tối ưu.
* [ ] Phân biệt Memoization và Tabulation.
* [ ] Xác định đúng trạng thái DP.
* [ ] Xây dựng được công thức chuyển trạng thái.
* [ ] Xác định đúng trạng thái cơ sở.
* [ ] Tự cài đặt DP một chiều và hai chiều.
* [ ] Hiểu các biến thể của Knapsack.
* [ ] Biết cách tối ưu bộ nhớ.

.. rubric:: GIAI ĐOẠN 4 - THUẬT TOÁN NÂNG CAO VÀ LUYỆN TẬP


.. rubric:: Thời gian: Ngày 22 đến ngày 30


.. rubric:: Mục tiêu


Tìm hiểu các thuật toán đồ thị nâng cao, bài toán ghép cặp, kỹ thuật tối
ưu hóa và kết hợp những kiến thức đã học.

Một số thuật toán trong giai đoạn này phù hợp với bài tập chuyên biệt
hơn là các bài LeetCode thông thường.

.. rubric:: Ngày 22 - Ôn tập đồ thị


**Nội dung**

* Biểu diễn đồ thị.
* Danh sách kề.
* Ma trận kề.
* Thành phần liên thông.
* Ứng dụng BFS và DFS.

**Thực hành**

* Cài đặt lại BFS.
* Cài đặt lại DFS.
* Phân tích độ phức tạp khi duyệt đồ thị.

.. rubric:: Ngày 23 - Maximum Flow


**Nội dung**

* Mạng luồng.
* Ràng buộc dung lượng.
* Đồ thị phần dư.
* Đường tăng luồng.
* Ford-Fulkerson.
* Edmonds-Karp.

**Thực hành**

* Cài đặt Ford-Fulkerson.
* Cài đặt Edmonds-Karp.
* Tự giải một ví dụ Maximum Flow nhỏ.

.. rubric:: Ngày 24 - Bipartite Matching


**Nội dung**

* Đồ thị hai phía.
* Ghép cặp cực đại.
* Đường tăng luồng.
* Ràng buộc ghép cặp.

**Thực hành**

* Cài đặt thuật toán ghép cặp hai phía cơ bản.
* Hiểu mối liên hệ giữa Matching và Maximum Flow.

**Bài tập**

* `Maximum Number of Accepted Invitations #1820 <https://leetcode.com/problems/maximum-number-of-accepted-invitations/>`_

.. rubric:: Ngày 25 - Optimal Pair Matching


**Nội dung**

* Bài toán phân công.
* Thuật toán Hungarian.
* Ma trận chi phí.
* Ghép cặp chi phí nhỏ nhất.
* Ghép cặp trọng số lớn nhất.

**Thực hành**

* Tìm hiểu bài toán Assignment Problem.
* Cài đặt thuật toán Hungarian.
* Tự giải một ví dụ phân công nhỏ.

.. rubric:: Ngày 26 - Linear Programming


**Nội dung**

* Mô hình quy hoạch tuyến tính.
* Hàm mục tiêu.
* Các ràng buộc.
* Miền nghiệm khả thi.
* Bài toán tối ưu hóa.

**Thực hành**

* Chuyển một bài toán tối ưu đơn giản thành mô hình quy hoạch tuyến tính.
* Xác định các biến quyết định.
* Xây dựng hàm mục tiêu và các ràng buộc.

.. rubric:: Ngày 27 - Simplex Method


**Nội dung**

* Thuật toán Simplex.
* Nghiệm cơ sở khả thi.
* Phép xoay trục.
* Biến vào và biến ra.
* Biểu diễn bằng bảng Simplex.

**Thực hành**

* Tự giải một ví dụ Simplex nhỏ.
* Hiểu cách phép xoay trục cải thiện giá trị hàm mục tiêu.

.. rubric:: Ngày 28 - Bài tập tổng hợp


**Nội dung**

* Kết hợp các thuật toán đồ thị.
* Kết hợp BFS và DFS.
* Backtracking với các ràng buộc.
* Dynamic Programming với tối ưu trạng thái.

**Thực hành**

* Giải bài toán mà không biết trước thuật toán cần sử dụng.
* Giải thích vì sao lựa chọn thuật toán đó.

.. rubric:: Ngày 29 - Hard Drills


**Mục tiêu**

* Giải các bài Medium và Hard chưa từng gặp.
* Luyện tập trong giới hạn thời gian.
* Không xem lời giải quá sớm.
* So sánh nhiều hướng tiếp cận.

**Thực hành**

* Giải từ 2 đến 3 bài tổng hợp.
* Phân tích các lần giải sai.
* Cài đặt lại những lời giải chưa đúng.

.. rubric:: Ngày 30 - Đánh giá cuối chương trình


**Mục tiêu**

* Đánh giá mức độ hiểu thuật toán.
* Xác định những phần còn yếu.
* Ôn lại các lỗi đã gặp.
* Xây dựng lộ trình học tiếp theo.

**Danh sách đánh giá**

* [ ] Giải độc lập một bài Easy chưa từng gặp.
* [ ] Giải độc lập một bài Medium chưa từng gặp.
* [ ] Giải thích hướng tiếp cận trước khi viết mã.
* [ ] Phân tích độ phức tạp thời gian và bộ nhớ.
* [ ] Tự cài đặt BFS và DFS.
* [ ] Tự cài đặt một lời giải Backtracking.
* [ ] Tự cài đặt các bài Dynamic Programming cơ bản.
* [ ] Giải thích nguyên lý Maximum Flow và Matching.
* [ ] Hiểu mục đích của Linear Programming và Simplex.

.. rubric:: PHƯƠNG PHÁP HỌC MỖI NGÀY


.. rubric:: Quy trình học


Mỗi buổi học nên tuân theo một quy trình nhất quán.

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - Hoạt động
     - Thời gian
     - Mục tiêu
   * - Tìm hiểu
     - 20 phút
     - Đọc lý thuyết và hiểu nguyên lý thuật toán.
   * - Cài đặt
     - 20 phút
     - Tự viết thuật toán từ đầu.
   * - Giải bài
     - 60 đến 90 phút
     - Giải từ 1 đến 2 bài LeetCode.
   * - Phân tích
     - 20 phút
     - Đánh giá độ phức tạp và các hướng giải khác.
   * - Ghi nhớ
     - 10 phút
     - Tự trình bày lại thuật toán mà không xem tài liệu.

.. rubric:: Quy trình phân tích bài toán


Trước khi viết mã, hãy trả lời năm câu hỏi sau:

1. Đầu vào và đầu ra thực sự của bài toán là gì?
2. Lời giải đơn giản nhất có thể là gì?
3. Điểm nghẽn tính toán của lời giải ban đầu nằm ở đâu?
4. Cấu trúc dữ liệu hoặc thuật toán nào có thể cải thiện lời giải?
5. Độ phức tạp thời gian và bộ nhớ của lời giải mới là bao nhiêu?

Mục tiêu là hình thành tư duy thuật toán, thay vì ghi nhớ lời giải
của từng bài LeetCode.

.. rubric:: XÂY DỰNG ALGORITHM NOTEBOOK


Nên tạo một kho mã nguồn riêng để lưu trữ kiến thức, mã cài đặt và
tiến độ học tập.

.. rubric:: Cấu trúc thư mục đề xuất


.. code-block:: text

   leetcode-30-days/
   |
   +-- 00-foundation/
   |   +-- complexity.md
   |   +-- problem-solving.md
   |
   +-- 01-bfs-flood-fill/
   |   +-- theory.md
   |   +-- bfs.cpp
   |   +-- flood-fill.cpp
   |   +-- problems.md
   |
   +-- 02-brute-force-backtracking/
   |   +-- theory.md
   |   +-- recursion.cpp
   |   +-- backtracking.cpp
   |   +-- problems.md
   |
   +-- 03-dynamic-programming/
   |   +-- theory.md
   |   +-- memoization.cpp
   |   +-- tabulation.cpp
   |   +-- problems.md
   |
   +-- 04-advanced/
   |   +-- maximum-flow.cpp
   |   +-- bipartite-matching.cpp
   |   +-- hungarian.cpp
   |   +-- simplex.md
   |
   +-- progress.md

.. rubric:: Tiêu chí hoàn thành một thuật toán


Không đánh dấu một thuật toán là đã hoàn thành chỉ vì lời giải LeetCode
đã được chấp nhận.

.. list-table::
   :header-rows: 1
   :widths: 22 78

   * - Tiêu chí
     - Năng lực cần đạt
   * - Hiểu nguyên lý
     - Có thể giải thích cách thuật toán hoạt động.
   * - Cài đặt
     - Có thể tự viết mã mà không cần xem lời giải mẫu.
   * - Nhận diện
     - Biết khi nào nên áp dụng thuật toán.
   * - Phân tích
     - Đánh giá được độ phức tạp thời gian và bộ nhớ.
   * - Biến thể
     - Có thể giải những bài toán thay đổi từ dạng đã học.
   * - Ghi nhớ
     - Có thể tự cài đặt lại sau 2 đến 3 ngày.

.. rubric:: THEO DÕI TIẾN ĐỘ


.. rubric:: Giai đoạn 1 - Nền tảng


* [ ] Ngày 1
* [ ] Ngày 2
* [ ] Ngày 3
* [ ] Ngày 4
* [ ] Ngày 5
* [ ] Ngày 6
* [ ] Ngày 7

.. rubric:: Giai đoạn 2 - Backtracking


* [ ] Ngày 8
* [ ] Ngày 9
* [ ] Ngày 10
* [ ] Ngày 11
* [ ] Ngày 12
* [ ] Ngày 13
* [ ] Ngày 14

.. rubric:: Giai đoạn 3 - Dynamic Programming


* [ ] Ngày 15
* [ ] Ngày 16
* [ ] Ngày 17
* [ ] Ngày 18
* [ ] Ngày 19
* [ ] Ngày 20
* [ ] Ngày 21

.. rubric:: Giai đoạn 4 - Thuật toán nâng cao


* [ ] Ngày 22
* [ ] Ngày 23
* [ ] Ngày 24
* [ ] Ngày 25
* [ ] Ngày 26
* [ ] Ngày 27
* [ ] Ngày 28
* [ ] Ngày 29
* [ ] Ngày 30

.. rubric:: TÀI LIỆU THAM KHẢO


* `Topcoder - How To Find a Solution <https://www.topcoder.com/thrive/articles/How%20To%20Find%20a%20Solution>`_
* `LeetCode 75 <https://leetcode.com/studyplan/leetcode-75/>`_
* `LeetCode Problemset <https://leetcode.com/problemset/>`_
* `CP-Algorithms <https://cp-algorithms.com/>`_
* `USACO Guide <https://usaco.guide/>`_

.. rubric:: MỤC TIÊU CUỐI CÙNG


Điều quan trọng nhất không phải là giải được 100 bài trong 30 ngày.

Mục tiêu thực sự là hình thành khả năng nhìn vào một bài toán chưa từng
gặp và tự xác định được:

1. Cách mô hình hóa bài toán.
2. Cách xây dựng lời giải đơn giản ban đầu.
3. Cách tìm ra điểm nghẽn của lời giải.
4. Kỹ thuật thuật toán phù hợp cần sử dụng.
5. Cách chứng minh lời giải đúng.
6. Cách phân tích và tối ưu độ phức tạp.

**Đích đến là học cách tư duy thuật toán, không phải học thuộc lời giải
LeetCode.**