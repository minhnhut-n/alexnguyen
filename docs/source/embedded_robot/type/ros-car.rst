1. ROS Car — Giải thích chi tiết cấu trúc project autocar_ros
=============================================================

.. meta::
   :description: Giai thich chi tiet cau truc project autocar_ros - ROS 2 Jazzy, Gazebo, RViz
   :keywords: ROS 2, Gazebo, RViz, autocar_ros, Mobile Robot, DiffDrive, Jazzy

.. contents:: **Mục lục chi tiết**
   :depth: 2
   :local:
   :backlinks: entry

---

.. rubric:: Giới thiệu

Tài liệu này giải thích chi tiết:

* Vai trò và chức năng của từng file trong hệ thống.
* Lý do project cần file đó.
* **Hậu quả / Lỗi hệ thống** nếu file bị thiếu hoặc sai cấu hình.

---

2. Sơ đồ cây hệ thống (tree tổng)
=================================

Cấu trúc thư mục hiện tại của workspace:

.. code-block:: text

   ros_project/                        # Workspace build root (chạy colcon tại đây)
   ├── .gitignore                      # [Phụ] Chặn tracking build/install/log
   ├── LICENSE                         # [Phụ] Giấy phép MIT
   ├── ROS_TONG_HOP.md                 # [Phụ] Tổng hợp hiện trạng hệ thống
   ├── ROS_DEVELOPMENT_PLAN.md         # [Phụ] Kế hoạch phát triển
   ├── build/ install/ log/            # [Tự sinh] Artefact colcon — không commit
   └── autocar_ros/                    # ROS Package duy nhất
       ├── package.xml                 # ★ QUAN TRỌNG — Khai sinh + Phụ thuộc
       ├── CMakeLists.txt              # ★ QUAN TRỌNG — Quy tắc biên dịch & Cài đặt
       ├── urdf/
       │   └── autocar.urdf.xacro      # ★ QUAN TRỌNG NHẤT — Mô hình & Khớp xe
       ├── worlds/
       │   ├── autocar_world.sdf       # ★ QUAN TRỌNG — Sân mô phỏng Gazebo
       │   └── gz_args.yaml            # [Phụ] Ghi chú tham số
       ├── launch/
       │   ├── sim.launch.py           # ★ QUAN TRỌNG — Nút khởi động toàn hệ thống
       │   └── display.launch.py       # [Hỗ trợ] Xem tĩnh model trên RViz
       ├── config/
       │   └── gz_bridge.yaml          # ★ QUAN TRỌNG — Cầu nối ROS ↔ Gazebo
       ├── src/
       │   ├── console_drive.cpp       # ★ QUAN TRỌNG — Điều khiển qua Console
       │   └── hello_node.cpp          # [Hỗ trợ] Node demo kiểm tra môi trường
       ├── rviz/
       │   └── autocar.rviz            # [Hỗ trợ] Cấu hình giao diện RViz
       ├── include/autocar_ros/        # [Phụ] Lưu trữ C++ Header files
       ├── test/                       # [Phụ] Khung chờ test suite
       ├── HD_CHAY_XE.md               # [Phụ] Hướng dẫn vận hành nhanh
       └── CAC_BUOC_TU_PROJECT_TRONG.md# [Phụ] Nhật ký tái hiện từ 0

.. rubric:: Luồng vận hành khi chạy ``ros2 launch autocar_ros sim.launch.py``

.. code-block:: text

   sim.launch.py
    ├─> autocar.urdf.xacro --xacro--> robot_description ─> robot_state_publisher ─> /tf_static
    ├─> autocar_world.sdf ─────────────────────────────────> Gazebo (Sân mô phỏng + Vật lý)
    ├─> create (-topic robot_description) ─────────────────> Spawn xe vào sân
    ├─> gz_bridge.yaml ────────────────────────────────────> Cầu nối cmd_vel/odom/tf (ROS ↔ GZ)
    ├─> autocar.rviz ──────────────────────────────────────> RViz2 vẽ mô hình xe & Odometry
    └─> console_drive.cpp --/cmd_vel--> DiffDrive plugin ──> Bánh quay, xe di chuyển ─> /odom

---

3. Nhóm A: Các file quan trọng (cốt lõi)
========================================

A1. `urdf/autocar.urdf.xacro` — HÌNH HÀI & CẤU TRÚC KHỚP XE
-----------------------------------------------------------

.. important::

   **Đây là file quan trọng nhất project.**
   Mọi thành phần (RViz, Gazebo, Tính Odometry, TF Tree) đều phụ thuộc vào cây liên kết (joint/link) định nghĩa tại đây.
   **Chỉ cần sai 1 thông số origin:** Bánh xe sẽ bị trật, lún đất hoặc mô phỏng bị văng ra ngoài.

* **Thuộc tính cơ bản (`xacro:property`):**
  
  Quản lý kích thước hình học (``body_l/w/h``, ``wheel_r``, ``track``, ``wheelbase``). Tất cả thông số được gom về một chỗ giúp dễ dàng tùy chỉnh scale xe.

* **Vật liệu & Màu sắc (`material`):**
  
  Bảng màu trực quan bao gồm ``blue``, ``cyan``, ``yellow``, ``orange``, ``grey``, ``glass``, ``tire``, ``hub``, ``lidar_red``. Chỉ phục vụ hiển thị.

* **Công thức quán tính (`inertial_box` / `inertial_cylinder`):**
  
  Cung cấp khối lượng và ma trận quán tính cho mô phỏng vật lý Gazebo. Thiếu thông số này xe sẽ bị rỗng ruột, bay tự do hoặc gây lỗi warning vật lý.

* **Macro bánh xe (`wheel`):**
  
  Khuôn đúc chi tiết bánh xe (mâm trắng + lốp đen) với Joint tại gốc tọa độ :math:`(x, y, wheel\_r)` quay quanh trục Y.
  
  * **Chủ động (2 bánh sau):** Khớp ``continuous``.
  * **Bị động (2 bánh trước):** Khớp ``revolute``.

* **Liên kết Base & Chassis:**
  
  Chuyển giao từ ``base_link`` → ``base_joint`` (nâng độ cao :math:`z = 0.20\text{m}`) → ``chassis``. Độ cao được dồn hoàn toàn vào một joint duy nhất giúp khắc phục triệt để lỗi trật bánh do cộng dồn tọa độ Z ở từng visual.

* **Cảm biến (`lidar` + `camera`):**
  
  Các mô hình cảm biến tĩnh gắn fixed trên khung xe. Đây là vị trí chuẩn bị sẵn để gắn plugin thu thập dữ liệu thật về sau.

* **Plugin động cơ (`gz-sim-diff-drive-system`):**
  
  Plugin điều khiển vi sai (Differential Drive). Nhận lệnh từ topic ``cmd_vel``, tính toán quy đổi ra tốc độ góc từng bánh dựa trên ``track`` + ``wheel_radius``, sau đó xuất tọa độ phản hồi ra topic ``odom``.

A2. `launch/sim.launch.py` — NÚT KHỞI ĐỘNG TOÀN HỆ THỐNG
--------------------------------------------------------

Giúp tự động hóa khởi tạo toàn bộ tiến trình chỉ bằng **một câu lệnh duy nhất**.

.. list-table::
   :widths: 35 65
   :header-rows: 1

   * - Thành phần
     - Chức năng
   * - ``get_package_share_directory``
     - Tìm kiếm đường dẫn tuyệt đối đến thư mục ``share`` của package sau khi build.
   * - ``Command(['xacro', ...])``
     - Biên dịch động các Macro Xacro thành dữ liệu URDF thuần. Ép kiểu ``ParameterValue(str)`` để tránh lỗi launch parser nhầm định dạng YAML.
   * - ``robot_state_publisher``
     - Lắng nghe URDF, tính toán hệ tọa độ TF tĩnh và phát các topic ``/robot_description`` & ``/tf_static``.
   * - Include ``gz_sim.launch.py``
     - Kích hoạt không gian mô phỏng Gazebo (mở World).
   * - Node ``create``
     - Spawns mô hình xe vào không gian Gazebo từ topic ``robot_description`` tại độ cao :math:`z = 0.15\text{m}`.
   * - Node ``parameter_bridge``
     - Nối dây giao tiếp giữa 2 môi trường ROS 2 và Gazebo qua file config bridge.
   * - Node ``rviz2``
     - Khai phóng màn hình quan sát đồ họa với cấu hình file ``autocar.rviz``.

A3. `worlds/autocar_world.sdf` — MÔ TRƯỜNG SÂN MÔ PHỎNG (GAZEBO WORLD)
----------------------------------------------------------------------

.. warning::

   Thiếu World đúng chuẩn hoặc sai phiên bản SDF (chuẩn là SDF version 1.11 cho Gazebo Harmonic / ROS 2 Jazzy) sẽ khiến Gazebo tự động crash hoặc treo vô hạn ở node ``create``.

* **Ánh sáng & Bầu trời:** Cấu hình nguồn sáng Mặt Trời đổ bóng thực và nền trời quan sát.
* **Cấu hình Vật lý (Physics):** Động cơ vật lý **DART** với bước thời gian (step size) :math:`1\text{ms}` cho độ chính xác cao.
* **Hệ thống Plugins cốt lõi:**
  
  #. ``Physics``: Tính toán trọng lực, va chạm.
  #. ``UserCommands``: Cho phép thao tác spawn/di chuyển vật thể qua giao diện.
  #. ``SceneBroadcaster``: Render hình ảnh 3D ra màn hình.
  #. ``Contact``: Xử lý lực va chạm bề mặt.

* **Bản đồ:** Sàn xám diện tích :math:`60\times60\text{m}`, đường nhựa kích thước :math:`14\times8\text{m}` có vạch kẻ vàng, kèm 4 cọc tiêu màu cam làm mốc định vị.

A4. `config/gz_bridge.yaml` — CẦU NỐI DỮ LIỆU ROS ↔ GAZEBO
----------------------------------------------------------

File đóng vai trò là "Thông dịch viên" giữa ROS 2 và Gazebo.

Cấu hình bao gồm 3 luồng giao tiếp chính:

1. **``cmd_vel`` (ROS 2 → Gazebo):** Truyền lệnh vận tốc từ tay lái vào plugin ``DiffDrive``.
2. **``odom`` (Gazebo → ROS 2):** Truyền dữ liệu đồng hồ hành trình (Odometry) về cho RViz / Nav2.
3. **``tf`` (Gazebo → ROS 2):** Truyền tư thế chuyển động thực của xe.

.. note::

   Đã áp dụng chuẩn ROS 2 Jazzy: Khai báo cặp khóa ``ros_topic_name`` và ``gz_topic_name`` minh bạch để tránh xung đột dữ liệu.

A5. `src/console_drive.cpp` — BỘ ĐIỀU KHIỂN TAY LÁI CONSOLE
-----------------------------------------------------------

Chương trình C++ đọc bàn phím ở chế độ **termios raw** (nhận phím tức thì không cần nhấn Enter).

* **Thao tác phím:**
  
  * **``W`` / ``S``:** Tăng / Giảm ga (Bước tăng :math:`0.1\text{m/s}`, Giới hạn tối đa :math:`1.5\text{m/s}`).
  * **``A`` / ``D``:** Rẽ Trái / Rẽ Phải (Giới hạn tốc độ góc :math:`2.5\text{rad/s}`).
  * **``Q`` / ``E``:** Xoay tròn tại chỗ.
  * **``SPACE`` (Phím cách):** Phanh gấp (Dừng xe).
  * **``X``:** Thoát chương trình.

* **Kiến trúc:** Chạy Timer chu kỳ :math:`100\text{ms}` để duy trì phát tin nhắn ``geometry_msgs/msg/Twist`` liên tục lên topic ``/cmd_vel``. Sử dụng luồng riêng (Separate Thread) để vòng lặp timer không bị gián đoạn khi chờ phím.

A6. `package.xml` & `CMakeLists.txt` — BỘ HỒ SƠ & QUY TRÌNH BIÊN DỊCH
---------------------------------------------------------------------

`package.xml`
   Khai báo thông tin package, người phát triển và các gói phụ thuộc:
   
   * **Build dependencies:** ``rclcpp``, ``geometry_msgs``, ``sensor_msgs``, ``nav_msgs``.
   * **Execution dependencies:** ``robot_state_publisher``, ``xacro``, ``ros_gz_sim``, ``ros_gz_bridge``, ``rviz2``.

`CMakeLists.txt`
   Chứa chỉ thị biên dịch C++ (tạo file thực thi ``console_drive``, ``hello_node``) và **quản lý cài đặt (install)**.
   
   .. danger::

      Nếu quên khai báo `install()` cho các thư mục ``launch/``, ``config/``, ``urdf/``, ``worlds/``, ``rviz/`` vào không gian ``share/``, hệ thống Launch sẽ không thể tìm thấy tài nguyên khi chạy!

---

4. Nhóm B: Các file hỗ trợ & phụ tùng
=====================================

.. list-table::
   :widths: 30 70
   :header-rows: 1

   * - Tên File / Thư mục
     - Mô tả & Vai trò
   * - ``launch/display.launch.py``
     - Khởi chạy chế độ xem tĩnh model trên RViz2 (sử dụng ``joint_state_publisher_gui``). Dùng để soi lỗi khớp/lệch gốc tọa độ mà không cần bật Gazebo.
   * - ``src/hello_node.cpp``
     - File C++ mẫu đơn giản, dùng để kiểm tra bộ biên dịch (Toolchain) của ROS 2 hoạt động đúng hay không.
   * - ``rviz/autocar.rviz``
     - File lưu cấu hình giao diện RViz (Bao gồm Grid, RobotModel, Odometry, TF Tree, cấu hình Fixed Frame là ``odom`` và Camera góc nhìn bám theo ``base_link``).
   * - ``worlds/gz_args.yaml``
     - File lưu trữ phụ các tham số ghi chú cho môi trường mô phỏng.
   * - ``include/autocar_ros/``
     - Thư mục giữ chỗ để chứa các file Header (``.hpp``) khi phát triển các Node C++ phức tạp hơn.
   * - ``test/``
     - Thư mục chuẩn bị cho các kịch bản Unit Test và Integration Test.
   * - ``HD_CHAY_XE.md``
     - Tài liệu hướng dẫn thao tác nhanh dành cho người vận hành.
   * - ``CAC_BUOC_TU_PROJECT_TRONG.md``
     - Nhật ký ghi lại 8 bước xây dựng lại project từ con số 0.
   * - ``ROS_TONG_HOP.md`` & ``ROS_DEVELOPMENT_PLAN.md``
     - Báo cáo tổng hợp kiến trúc hệ thống và lộ trình nâng cấp phần mềm.
   * - ``.gitignore`` / ``LICENSE``
     - Chặn các file tạm/artefacts của colcon và khai báo bản quyền mã nguồn mở MIT.

---

.. rubric:: Tài liệu tham khảo

- `ROS 2 Jazzy documentation <https://docs.ros.org/en/jazzy/>`_
- `Gazebo Harmonic documentation <https://gazebosim.org/docs>`_
- `ros_gz_bridge <https://github.com/gazebosim/ros_gz>`_
