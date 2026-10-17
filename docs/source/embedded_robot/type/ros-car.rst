ROS Car — Giải thích chi tiết cấu trúc project autocar_ros
===========================================================

.. _ros-car-label:

.. meta::
   :description: Giải thích chi tiết cấu trúc project autocar_ros - ROS 2 Jazzy, Gazebo, RViz
   :keywords: ROS 2, Gazebo, RViz, autocar_ros, Mobile Robot, DiffDrive, Jazzy

.. contents:: **Mục lục chi tiết**
   :depth: 2
   :local:
   :backlinks: entry

Giới thiệu
----------

Tài liệu này giải thích chi tiết:

* Vai trò và chức năng của từng file trong hệ thống.
* Lý do project cần file đó.
* **Hậu quả / Lỗi hệ thống** nếu file bị thiếu hoặc sai cấu hình.

Mục tiêu của tài liệu:

* Mô tả tổng thể về hệ thống xe mô phỏng ROS Car.
* Phân giải các thành phần chính và trách nhiệm của từng thành phần.
* Trình tự thực thi, tương thích, phụ thuộc và luồng dữ liệu/sự kiện.
* Các tham số, lệnh, API và cơ chế quan trọng liên quan.
* Cách kiểm tra, chẩn đoán và cách ly lỗi.
* Điều kiện tiên quyết, hạn chế và sai lầm thường gặp.

Content map
-----------

.. list-table:: Các phần chính của tài liệu
   :widths: 25 75
   :header-rows: 1

   * - Phần
     - Nội dung
   * - :ref:`big-picture-ros-car`
     - Tổng quan hệ thống, vai trò của các thành phần chính và luồng hoạt động.
   * - :ref:`directory-tree-ros-car`
     - Cấu trúc thư mục workspace và package autocar_ros.
   * - :ref:`core-components-ros-car`
     - Các file cốt lõi: URDF, launch, world, bridge, điều khiển console, build.
   * - :ref:`support-files-ros-car`
     - Các file hỗ trợ: display.launch, hello_node, RViz, header, test, tài liệu kèm.
   * - :ref:`troubleshooting-ros-car`
     - Bảng chẩn đoán lỗi: biểu hiện, nguyên nhân, cách xác minh, xử lý khắc phục.
   * - :ref:`checklist-ros-car`
     - Danh sách kiểm tra điều kiện tiên quyết và thứ tự thực thi.

.. _big-picture-ros-car:

Big picture
-----------

Hệ thống **ROS Car** (project ``autocar_ros``) là một mô phỏng xe di động hai bánh chủ động (Differential Drive) chạy trên ROS 2 Jazzy và Gazebo Harmonic. Các thành phần chính chịu trách nhiệm như sau:

* **``urdf/autocar.urdf.xacro``**: Định hình hình học, khớp, chất lượng quán tính, cảm biến và plugin động cơ của xe.
* **``launch/sim.launch.py``**: Khởi động toàn bộ hệ thống bằng một lệnh duy nhất (URDF → Gazebo → spawn → bridge → RViz).
* **``worlds/autocar_world.sdf``**: Mô phỏng môi trường sân, ánh sáng, vật lý và plugin Gazebo.
* **``config/gz_bridge.yaml``**: Cầu nối topic giữa ROS 2 và Gazebo (cmd_vel, odom, tf).
* **``src/console_drive.cpp``**: Đọc phím bàn phím và phát lệnh ``/cmd_vel`` để điều khiển xe.
* **``package.xml`` & ``CMakeLists.txt``**: Khai báo phụ thuộc và quy tắc biên dịch/cài đặt package.

Luồng hoạt động tổng thể khi chạy ``ros2 launch autocar_ros sim.launch.py``::

   sim.launch.py
   ├─> autocar.urdf.xacro --xacro--> robot_description ─> robot_state_publisher ─> /tf_static
   ├─> autocar_world.sdf ─────────────────────────────────> Gazebo (Sân mô phỏng + Vật lý)
   ├─> create (-topic robot_description) ─────────────────> Spawn xe vào sân
   ├─> gz_bridge.yaml ────────────────────────────────────> Cầu nối cmd_vel/odom/tf (ROS ↔ GZ)
   ├─> autocar.rviz ──────────────────────────────────────> RViz2 vẽ mô hình xe & Odometry
   └─> console_drive.cpp --/cmd_vel--> DiffDrive plugin ──> Bánh quay, xe di chuyển ─> /odom

.. _directory-tree-ros-car:

Cấu trúc thư mục
----------------

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

.. code-block:: text

   sim.launch.py
   ├─> autocar.urdf.xacro --xacro--> robot_description ─> robot_state_publisher ─> /tf_static
   ├─> autocar_world.sdf ─────────────────────────────────> Gazebo (Sân mô phỏng + Vật lý)
   ├─> create (-topic robot_description) ─────────────────> Spawn xe vào sân
   ├─> gz_bridge.yaml ────────────────────────────────────> Cầu nối cmd_vel/odom/tf (ROS ↔ GZ)
   ├─> autocar.rviz ──────────────────────────────────────> RViz2 vẽ mô hình xe & Odometry
   └─> console_drive.cpp --/cmd_vel--> DiffDrive plugin ──> Bánh quay, xe di chuyển ─> /odom

.. _core-components-ros-car:

Các thành phần cốt lõi (Nhóm A)
-------------------------------

A1. `urdf/autocar.urdf.xacro` — HÌNH HÀI & CẤU TRÚC KHỚP XE
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

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
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. _sim-launch-label:

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
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. _autocar-world-label:

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
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. _gz-bridge-label:

File đóng vai trò là "Thông dịch viên" giữa ROS 2 và Gazebo.

Cấu hình bao gồm 3 luồng giao tiếp chính:

1. **``cmd_vel`` (ROS 2 → Gazebo):** Truyền lệnh vận tốc từ tay lái vào plugin ``DiffDrive``.
2. **``odom`` (Gazebo → ROS 2):** Truyền dữ liệu đồng hồ hành trình (Odometry) về cho RViz / Nav2.
3. **``tf`` (Gazebo → ROS 2):** Truyền tư thế chuyển động thực của xe.

.. note::

   Đã áp dụng chuẩn ROS 2 Jazzy: Khai báo cặp khóa ``ros_topic_name`` và ``gz_topic_name`` minh bạch để tránh xung đột dữ liệu.

A5. `src/console_drive.cpp` — BỘ ĐIỀU KHIỂN TAY LÁI CONSOLE
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. _console-drive-label:

Chương trình C++ đọc bàn phím ở chế độ **termios raw** (nhận phím tức thì không cần nhấn Enter).

* **Thao tác phím:**
  
  * **``W`` / ``S``:** Tăng / Giảm ga (Bước tăng :math:`0.1\text{m/s}`, Giới hạn tối đa :math:`1.5\text{m/s}`).
  * **``A`` / ``D``:** Rẽ Trái / Rẽ Phải (Giới hạn tốc độ góc :math:`2.5\text{rad/s}`).
  * **``Q`` / ``E``:** Xoay tròn tại chỗ.
  * **``SPACE`` (Phím cách):** Phanh gấp (Dừng xe).
  * **``X``:** Thoát chương trình.

* **Kiến trúc:** Chạy Timer chu kỳ :math:`100\text{ms}` để duy trì phát tin nhắn ``geometry_msgs/msg/Twist`` liên tục lên topic ``/cmd_vel``. Sử dụng luồng riêng (Separate Thread) để vòng lặp timer không bị gián đoạn khi chờ phím.

A6. `package.xml` & `CMakeLists.txt` — BỘ HỒ SƠ & QUY TRÌNH BIÊN DỊCH
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. _build-files-label:

`package.xml`
   Khai báo thông tin package, người phát triển và các gói phụ thuộc:
   
   * **Build dependencies:** ``rclcpp``, ``geometry_msgs``, ``sensor_msgs``, ``nav_msgs``.
   * **Execution dependencies:** ``robot_state_publisher``, ``xacro``, ``ros_gz_sim``, ``ros_gz_bridge``, ``rviz2``.

`CMakeLists.txt`
   Chứa chỉ thị biên dịch C++ (tạo file thực thi ``console_drive``, ``hello_node``) và **quản lý cài đặt (install)**.
   
   .. danger::

      Nếu quên khai báo `install()` cho các thư mục ``launch/``, ``config/``, ``urdf/``, ``worlds/``, ``rviz/`` vào không gian ``share/``, hệ thống Launch sẽ không thể tìm thấy tài nguyên khi chạy!


.. _support-files-ros-car:

Các file hỗ trợ & phụ tùng (Nhóm B)
-----------------------------------

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


.. _troubleshooting-ros-car:

Chẩn đoán và xử lý lỗi
----------------------

Bảng dưới đây liệt kê các lỗi thường gặp, nguyên nhân có thể, cách xác minh và hành động khắc phục.

.. list-table:: Bảng chẩn đoán lỗi
   :widths: 20 25 25 30
   :header-rows: 1

   * - Biểu hiện
     - Nguyên nhân có thể
     - Cách xác minh
     - Hành động khắc phục
   * - Gazebo crash hoặc treo ở node ``create``
     - Sai phiên bản SDF hoặc thiếu file world
     - Kiểm tra phiên bản SDF (phải là 1.11 cho Gazebo Harmonic / ROS 2 Jazzy) và file ``autocar_world.sdf`` tồn tại
     - Cập nhật file world theo chuẩn SDF 1.11
   * - Bánh xe trật, lún đất hoặc mô phỏng văng ra ngoài
     - Tham số ``origin`` sai trong URDF/xacro
     - Xem lại các thông số ``origin`` trong ``autocar.urdf.xacro``
     - Sửa lại tọa độ origin cho chính xác
   * - Xe không di chuyển khi nhấn phím điều khiển
     - Cầu bridge không kết nối topic ``cmd_vel``
     - Kiểm tra ``gz_bridge.yaml`` và node ``parameter_bridge`` đang chạy
     - Khai báo lại cặp ``ros_topic_name`` / ``gz_topic_name`` trong bridge
   * - RViz không hiển thị mô hình xe
     - Thiếu cài đặt resource ``share/`` hoặc sai file RViz
     - Kiểm tra ``CMakeLists.txt`` đã ``install()`` thư mục ``rviz/``
     - Thêm lệnh ``install()`` cho ``rviz/`` và build lại
   * - Launch không tìm thấy tài nguyên (urdf, world, config...)
     - Quên ``install()`` vào không gian ``share/``
     - Kiểm tra ``CMakeLists.txt`` có khai báo ``install(DIRECTORY ... DESTINATION share)``
     - Bổ sung ``install()`` cho các thư mục cần thiết

.. _checklist-ros-car:

Danh sách kiểm tra
------------------

Danh sách kiểm tra theo thứ tự để xác minh điều kiện tiên quyết trước khi kết luận hệ thống lỗi hoặc chuyển sang gỡ lỗi phức tạp:

1. [ ] Cài đặt ROS 2 Jazzy và Gazebo Harmonic đúng phiên bản.
2. [ ] Package ``autocar_ros`` đã build bằng ``colcon build`` thành công.
3. [ ] Các thư mục ``launch/``, ``config/``, ``urdf/``, ``worlds/``, ``rviz/`` đã được ``install()`` vào ``share/``.
4. [ ] File ``autocar.urdf.xacro`` có thông số ``origin`` chính xác, không bị sai lệch tọa độ Z.
5. [ ] File ``gz_bridge.yaml`` khai báo đầy đủ cặp ``ros_topic_name`` và ``gz_topic_name``.
6. [ ] Chạy lệnh ``ros2 launch autocar_ros sim.launch.py`` thành công, Gazebo hiển thị sân và xe.
7. [ ] RViz2 mở được và hiển thị mô hình xe cùng Odometry với Fixed Frame là ``odom``.
8. [ ] Nhấn phím ``W``/``S``/``A``/``D`` trên console_drive, xe di chuyển và topic ``/cmd_vel`` có dữ liệu.
9. [ ] Kiểm tra topic ``/odom`` và ``/tf`` có dữ liệu cập nhật khi xe di chuyển.

.. _visual-summary-ros-car:

Tóm tắt trực quan
-----------------

Luồng dữ liệu chính của hệ thống ROS Car::

   Bàn phím (console_drive)
         │
         ▼
   Topic /cmd_vel (geometry_msgs/Twist)
         │
         ▼
   Cầu nối gz_bridge (ROS 2 ↔ Gazebo)
         │
         ▼
   Plugin DiffDrive trong Gazebo
         │
         ▼
   Bánh xe quay → Xe di chuyển → Odometry (odom, tf)
         │
         ▼
   RViz2 hiển thị mô hình xe & Odometry

**Các điểm cần nhớ:**

* File ``urdf/autocar.urdf.xacro`` là trọng tâm — sai thông số ``origin`` làm hỏng toàn bộ mô phỏng.
* Cầu nối ``gz_bridge.yaml`` phải đồng bộ topic giữa ROS 2 và Gazebo.
* ``CMakeLists.txt`` phải ``install()`` tất cả thư mục tài nguyên vào ``share/``.
* Khi lỗi xảy ra, hãy kiểm tra từ kiểm tra đơn giản (phiên bản, file tồn tại) sang kiểm tra phức tạp (node, topic, plugin).

Tài liệu tham khảo
------------------

- `ROS 2 Jazzy documentation <https://docs.ros.org/en/jazzy/>`_
- `Gazebo Harmonic documentation <https://gazebosim.org/docs>`_
- `ros_gz_bridge <https://github.com/gazebosim/ros_gz>`_
