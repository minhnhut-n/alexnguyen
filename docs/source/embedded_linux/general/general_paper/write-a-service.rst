==============================
Write a sample systemd service
==============================

Tài liệu này cung cấp cái nhìn tổng quan về kiến trúc 3 tầng của một hệ thống Linux, đồng thời hướng dẫn chi tiết cách cấu hình và triển khai một Linux Service đơn giản bằng ``systemd``.
Github resource: https://github.com/minhnhut-n/helper-tool/tree/main/demo/linux/sample-srv

Cấu Trúc Hệ Thống (3 Tầng)
--------------------------

Để hiểu cách một ứng dụng/service vận hành, hệ thống Linux được chia làm 3 tầng chính:

1. **Tầng 1: systemd (Service Manager)**

   - Đóng vai trò quản lý các tiến trình và dịch vụ trên hệ thống.
   - Nhiệm vụ chính: Khởi động (boot), khởi chạy (start), dừng (stop), tái khởi động (restart) và giám sát (supervision) trạng thái của các service.

2. **Tầng 2: Linux Service Process (User Space)**

   - Nơi chứa mã nguồn logic do người dùng viết.
   - Vòng đời của một service tiêu chuẩn ở tầng này bao gồm:

     * **Initialization**: Khởi tạo biến, môi trường, kết nối.
     * **Main loop**: Vòng lặp chính xử lý tác vụ.
     * **Business logic**: Nghiệp vụ xử lý chính của ứng dụng.
     * **Logging**: Ghi lại nhật ký hoạt động/lỗi.
     * **Cleanup**: Giải phóng tài nguyên trước khi thoát.

3. **Tầng 3: Linux Kernel (Kernel Space)**

   - Lõi điều hành thực hiện nhiệm vụ vận hành toàn bộ hệ thống (service chỉ là một tiến trình nhỏ được quản lý tại đây).
   - Quản lý các tài nguyên phần cứng và giao tiếp hệ thống:

     * **I/O Management**: Quản lý vào/ra dữ liệu.
     * **Scheduler**: Lập lịch phân chia CPU cho các tiến trình.
     * **File System**: Quản lý hệ thống tập tin.
     * **Signals & IPC**: Xử lý tín hiệu giao tiếp và cơ chế truyền thông giữa các tiến trình.


Cấu Trúc File Cấu Hình Unit (`.service`)
----------------------------------------

Để ``systemd`` có thể nhận diện và quản lý một dịch vụ, ta cần tạo một file cấu hình `.service` đặt tại `/etc/systemd/system/`.

Cấu trúc cơ bản như sau:

.. code-block:: ini

   [Unit]
   Description=<Mô tả ngắn gọn về service>
   After=<tên_service_khac.service>
   StartLimitIntervalSec=0

   [Service]
   Type=simple
   ExecStart=<đường_dẫn_tới_file_thực_thi> [tham_số_nếu_có]
   User=<tên_user_chạy_service>
   Environment=ENV=production
   Restart=always
   RestartSec=1

   [Install]
   WantedBy=multi-user.target

**Giải thích các thông số quan trọng:**

- ``[Unit]``:

  - **Description**: Mô tả thông tin dịch vụ.
  - **After**: Đảm bảo dịch vụ này chỉ chạy sau khi dịch vụ được chỉ định đã khởi động xong.
  - **StartLimitIntervalSec=0**: Vô hiệu hóa giới hạn số lần khởi động lại trong một khoảng thời gian (giúp service luôn luôn thử khởi động lại mà không bị đình chỉ).

- ``[Service]``:

  - **Type=simple**: Tiến trình được chỉ định ở ``ExecStart`` là tiến trình chính của dịch vụ.
  - **ExecStart**: Đường dẫn tuyệt đối tới file thực thi.
  - **User**: Định danh tài khoản người dùng thực thi tiến trình.
  - **Environment**: Khai báo các biến môi trường.
  - **Restart=always**: Tự động khôi phục và chạy lại service mỗi khi tiến trình bị dừng hoặc hỏng (crash).
  - **RestartSec=1**: Khoảng thời gian chờ (1 giây) trước khi tiến hành khởi động lại service.

- ``[Install]``:

  - **WantedBy=multi-user.target**: Xác định target để khởi động service cùng hệ thống khi boot ở chế độ đa người dùng (chế độ tiêu chuẩn).


Thực Hành Triển Khai Ví Dụ
--------------------------

Kịch bản demo: Sử dụng một chương trình đơn giản ``/usr/bin/sample_test`` (chỉ in ra "hello world!" rồi kết thúc). Khi tiến trình thoát, ``systemd`` sẽ tự động phát hiện và kích hoạt cơ chế `auto-restart` để chạy lại ứng dụng.

1. Chuẩn bị File Service
~~~~~~~~~~~~~~~~~~~~~~~~

Tạo file `/etc/systemd/system/sample.service` với nội dung:

.. code-block:: ini

   [Unit]
   Description=sample service
   After=sample.service
   StartLimitIntervalSec=0

   [Service]
   Type=simple
   ExecStart=/usr/bin/sample_test
   User=aelius
   Restart=always
   RestartSec=1

   [Install]
   WantedBy=multi-user.target

2. Các Lệnh Thao Tác Hệ Thống
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Kiểm tra sự tồn tại của file thực thi và file service:

.. code-block:: bash

   $ ls /usr/bin/ | grep sample
   sample_test

   $ ls /etc/systemd/system | grep sample
   sample.service

Nạp lại cấu hình manager của systemd để nhận diện service mới:

.. code-block:: bash

   $ sudo systemctl daemon-reload

Khởi chạy và bật tự động khởi động cùng hệ thống:

.. code-block:: bash

   $ sudo systemctl start sample.service
   $ sudo systemctl enable sample.service
   Created symlink /etc/systemd/system/multi-user.target.wants/sample.service → /etc/systemd/system/sample.service.

Kiểm tra trạng thái hoạt động:

.. code-block:: bash

   $ sudo systemctl status sample.service
   ● sample.service - sample service
      Loaded: loaded (/etc/systemd/system/sample.service; enabled; vendor preset: enabled)
      Active: activating (auto-restart) since Fri 2026-10-09 22:15:23 CST; 587ms ago
      Process: 7065 ExecStart=/usr/bin/sample_test (code=exited, status=0/SUCCESS)
    Main PID: 7065 (code=exited, status=0/SUCCESS)

3. Kết Quả Nhật Ký Hoạt Động (Logs)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Dưới đây là nhật ký hoạt động thực tế ghi nhận từ hệ thống (`journalctl` hoặc log đầu ra):

.. code-block:: text

   Oct 09 22:13:57 minion-pi sample_test[6464]: hello world!
   Oct 09 22:13:58 minion-pi systemd[1]: sample.service: Service hold-off time over, scheduling restart.
   Oct 09 22:13:58 minion-pi systemd[1]: sample.service: Scheduled restart job, restart counter is at 1.
   Oct 09 22:13:58 minion-pi systemd[1]: Stopped sample service.
   Oct 09 22:13:58 minion-pi systemd[1]: Started sample service.
   Oct 09 22:13:58 minion-pi sample_test[6471]: hello world!
   Oct 09 22:13:59 minion-pi systemd[1]: sample.service: Service hold-off time over, scheduling restart.
   Oct 09 22:13:59 minion-pi systemd[1]: sample.service: Scheduled restart job, restart counter is at 2.
   Oct 09 22:13:59 minion-pi systemd[1]: Stopped sample service.
   Oct 09 22:13:59 minion-pi systemd[1]: Started sample service.
   Oct 09 22:13:59 minion-pi sample_test[6485]: hello world!

*Phân tích kết quả:* 
Chương trình ``sample_test`` in dòng chữ ``hello world!`` rồi thoát (status 0). Do cấu hình ``Restart=always`` và ``RestartSec=1``, ``systemd`` sẽ đợi 1 giây, sau đó lập lịch kích hoạt lại service (tăng đếm `restart counter`) và tiếp tục chu kỳ.