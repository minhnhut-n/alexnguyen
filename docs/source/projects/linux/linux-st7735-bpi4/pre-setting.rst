2. Pre Setting
==============

Nhật Ký Cấu Hình SPI Trên Banana Pi BPI-M4
------------------------------------------

:Hệ điều hành: Banana Pi BPI-M4 (Realtek RTD1395, Kernel 4.9 Vendor BSP)
:Mục tiêu: Kích hoạt ``/dev/spidev0.0`` để giao tiếp màn hình ST7735

.. contents:: Mục lục
   :depth: 1

----

Attempt 1
---------

**Mục tiêu:** Kích hoạt node Hardware SPI (``spi@9801BD00``) trong file Device
Tree Binary (``rtd-1395-bananapi-m4-2GB.dtb``) trên phân vùng Boot để hệ thống
tạo ra file thiết bị ``/dev/spidev0.0``.

**Các bước cấu hình:**

1. Mount phân vùng boot chứa file DTB của hệ thống.
2. Decompile file ``.dtb`` gốc sang định dạng ``.dts`` dạng văn bản để chỉnh sửa.
3. Chuyển trạng thái node ``spi@9801BD00`` từ ``disabled`` sang ``okay``.
4. Khai báo thêm node con ``spidev@0`` bên trong node SPI với
   ``compatible = "rohm,dh2228fv"``.
5. Biên dịch ngược từ ``.dts`` thành ``.dtb``, ghi đè lại phân vùng boot và
   khởi động lại thiết bị.

**Câu lệnh sử dụng và ý nghĩa:**

* ``sudo mount /dev/mmcblk0p1 /mnt/boot``: Gắn phân vùng boot (chứa kernel và
  DTB) vào thư mục ``/mnt/boot`` để truy cập.
* ``sudo dtc -I dtb -O dts -o /tmp/bpi-m4.dts /mnt/boot/bananapi/bpi-m4/linux/rtd-1395-bananapi-m4-2GB.dtb``:
  Dùng công cụ Device Tree Compiler (``dtc``) để chuyển file nhị phân DTB thành
  file văn bản DTS để chỉnh sửa.
* ``sudo dtc -I dts -O dtb -o /mnt/boot/bananapi/bpi-m4/linux/rtd-1395-bananapi-m4-2GB.dtb /tmp/bpi-m4.dts``:
  Biên dịch file DTS đã sửa ngược lại thành file nhị phân DTB để U-Boot đọc lúc
  khởi động.
* ``sudo reboot``: Khởi động lại hệ thống để U-Boot nạp cấu hình DTB mới.
* ``ls /dev/spi*``: Kiểm tra xem file thiết bị giao tiếp SPI đã được Kernel
  tạo ra trong ``/dev`` chưa.

.. admonition:: ❌ Kết quả: Thất bại
   :class: warning

   Lệnh ``ls /dev/spi*`` báo *No such file or directory*. Kiểm tra log hệ
   thống không thấy bất kỳ ghi nhận nào về bộ điều khiển SPI.

----

Attempt 2
---------

**Mục tiêu:** Ép Kernel nạp thủ công các module SPI driver và kiểm tra xem
U-Boot đã đọc được cấu hình node SPI mới trong RAM hay chưa.

**Các bước cấu hình:**

1. Nạp thủ công module kernel ``spidev`` và các driver SPI DesignWare.
2. Kiểm tra node SPI trực tiếp trong cây Device Tree Runtime của bộ nhớ RAM.

**Câu lệnh sử dụng và ý nghĩa:**

* ``sudo modprobe spidev``: Nạp module Kernel ``spidev`` (driver giao tiếp SPI
  tầng user-space) vào bộ nhớ.
* ``sudo modprobe spi-dw-mmio`` & ``sudo modprobe spi-designware``: Nạp các
  driver điều khiển phần cứng SPI chuẩn DesignWare của Linux.
* ``cat /proc/device-tree/soc/spi@9801BD00/status``: Đọc trực tiếp trạng thái
  node SPI từ bộ nhớ RAM đang chạy để xác nhận U-Boot đã load đúng file DTB
  mới chưa.

.. admonition:: ❌ Kết quả: Thất bại
   :class: warning

   Lệnh ``cat`` báo *No such file or directory* do sai cấu trúc đường dẫn bus.
   File ``/dev/spidev0.0`` vẫn không xuất hiện.

----

Attempt 3
---------

**Mục tiêu:** Tìm kiếm chính xác vị trí node SPI trong cây Device Tree runtime
(``/proc/device-tree``) nhằm xác định U-Boot có thực sự nạp file DTB đã chỉnh
sửa hay không.

**Các bước cấu hình:**

1. Mount lại phân vùng boot để đảm bảo môi trường kiểm tra.
2. Sử dụng lệnh tìm kiếm hệ thống tệp trong ``/proc/device-tree`` để quét tất
   cả các node có chữ "spi".

**Câu lệnh sử dụng và ý nghĩa:**

* ``find /proc/device-tree/ -name "*spi*" 2>/dev/null``: Quét toàn bộ cây
  Device Tree mà Kernel đang sử dụng trong RAM để tìm các node liên quan đến
  SPI, ẩn các thông báo lỗi truy cập.

.. admonition:: ⚠️ Kết quả: Thành công một phần
   :class: attention

   Tìm thấy 2 node ``/proc/device-tree/spi@9801BD00`` và
   ``/proc/device-tree/spi@9801BD00/spidev@0``. Kết quả này khẳng định
   **U-Boot đã nạp đúng file DTB** vừa sửa vào RAM, nhưng Kernel không tạo
   thiết bị do thiếu driver tương thích.

----

Attempt 4
---------

**Mục tiêu:** Thay đổi chuỗi ``compatible`` của node ``spidev`` về chuẩn mặc
định (``spidev``) để phù hợp với bản Kernel 4.9 Vendor BSP của Realtek.

**Các bước cấu hình:**

1. Decompile file DTB từ phân vùng boot thành DTS.
2. Sửa thuộc tính node ``spidev@0`` từ ``compatible = "rohm,dh2228fv"`` thành
   ``compatible = "spidev"``.
3. Biên dịch lại thành DTB, ghi đè phân vùng boot và khởi động lại.
4. Ép nạp lại module ``spidev`` và kiểm tra log Kernel.

**Câu lệnh sử dụng và ý nghĩa:**

* ``dmesg | grep -iE "spi|dw_apb|9801bd00"``: Lọc toàn bộ nhật ký khởi động
  của Kernel (dmesg) để tìm các thông báo lỗi hoặc cảnh báo liên quan đến bộ
  điều khiển SPI phần cứng (``9801BD00``).
* ``lsmod | grep spidev``: Kiểm tra danh sách các module Kernel đang chạy xem
  ``spidev`` đã được active chưa.

.. admonition:: ❌ Kết quả: Thất bại
   :class: warning

   Thiết bị ``/dev/spidev0.0`` vẫn không xuất hiện. Lệnh ``dmesg`` hoàn toàn
   trống thông tin về SPI.

----

Attempt 5
---------

**Mục tiêu:** Phân tích nguyên nhân gốc rễ và xác định hướng xử lý triệt để
cho bài toán giao tiếp SPI trên Banana Pi BPI-M4.

**Các bước cấu hình:**

1. **Phân tích kết quả lệnh ``dmesg``:** Việc log trống 100% khẳng định bản
   Kernel Realtek 4.9 vendor đã **bị lược bỏ hoàn toàn Driver Hardware SPI
   Controller (``Realtek,rtk-dw-apb-ssi``)** trong lúc build Kernel Image.
2. **Loại trừ khả năng do phần cứng:** Đấu nối dây sai hay tháo hết dây
   **không làm mất ``/dev/spidev0.0``**, vì file này được tạo hoàn toàn bởi
   phần mềm (Kernel + Driver).
3. **Đề xuất giải pháp mới:** Chuyển sang cấu hình **Software SPI (Bitbang
   SPI)** bằng module ``spi-gpio`` tích hợp sẵn trong Kernel chuẩn để tạo
   ``/dev/spidev0.0`` thông qua các chân GPIO thường (GPIO 18, 19, 20, 23).

**Câu lệnh sử dụng và ý nghĩa:**

* ``sudo modprobe spi-gpio``: Nạp module giả lập giao tiếp SPI bằng phần mềm
  thông qua các chân GPIO tổng quát.

.. admonition:: ✅ Kết quả: Xác định được nguyên nhân gốc rễ
   :class: tip

   Nguyên nhân cốt lõi là do **thiếu driver Kernel gốc** (Hardware SPI
   Controller bị lược bỏ khỏi Kernel Image). Sẵn sàng chuyển sang giải pháp
   Software SPI để kích hoạt ``/dev/spidev0.0`` thành công 100%.