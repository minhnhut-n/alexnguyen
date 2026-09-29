.. _lcd-ili9341-initialization:

===============================================================================
Khởi Tạo Màn Hình LCD ILI9341 (STM32 HAL)
===============================================================================

.. contents:: Mục Lục Bài Viết
   :depth: 2
   :local:

.. rubric:: Giới thiệu

Chuỗi khởi tạo màn hình TFT **ILI9341** thực hiện thiết lập các thông số hoạt
động ban đầu cho chip điều khiển thông qua các lệnh (**Command**) và tham số
(**Parameter**) được truyền nối tiếp trên giao diện SPI.

.. code-block:: text
   :caption: Luồng giao tiếp giữa STM32 và chip ILI9341

   STM32 (MCU)  ──[ Command / Data ]──►  Chip ILI9341 (TFT Driver)
                                         │
                                         ├── Power Control & Sequence
                                         ├── Driver Timing Control
                                         ├── Display & Pixel Format
                                         └── Gamma Correction

-------------------------------------------------------------------------------

.. _lcd-ili9341-power-control:

1. Điều Khiển Khởi Tạo & Nguồn (Power Control & Sequence)
---------------------------------------------------------

* **SOFTWARE RESET** (``0x01``)
  Reset lại toàn bộ bộ nhớ và thanh ghi của ILI9341 về trạng thái mặc định bằng
  phần mềm mà không cần tác động chân phần cứng.

* **POWER CONTROL A & B** (``0xCB`` / ``0xCF``)
  Cấu hình các thông số điện áp nội bộ như VCORE, VCI và bật/tắt các chế độ tiết
  kiệm năng lượng (như ``pceq``).

* **POWER ON SEQUENCE CONTROL** (``0xED``)
  Kiểm soát chu trình bật nguồn (thời gian trễ và thứ tự cấp điện áp) để đảm bảo
  chip không bị sốc điện hay lỗi khởi động.

* **PUMP RATIO CONTROL** (``0xF7``)
  Điều chỉnh tỷ lệ của mạch bơm dòng (Charge Pump) nhằm tạo ra điện áp cao đủ
  điều khiển các tinh thể lỏng.

* **POWER CONTROL 1 & 2** (``0xC0`` / ``0xC1``)

  * ``VRH[5:0]``: Thiết lập điện áp tham chiếu GVDD (ảnh hưởng trực tiếp tới độ
    tương phản).
  * ``SAP[2:0]`` & ``BT[3:0]``: Điều chỉnh mức tiêu thụ dòng của op-amp nội bộ và
    tỷ lệ nâng áp (Boost factor).

* **VCM CONTROL 1 & 2** (``0xC5`` / ``0xC7``)
  Thiết lập điện áp **VCOM** (điện áp chung cho tất cả điểm ảnh). Tinh chỉnh VCOM
  giúp giảm thiểu hiện tượng nhấp nháy (flicker) trên tấm nền.

-------------------------------------------------------------------------------

.. _lcd-ili9341-driver-timing:

2. Định Thời & Xung Nhịp Driver (Driver Timing Control)
-------------------------------------------------------

* **DRIVER TIMING CONTROL A & B** (``0xE8`` / ``0xEA``)
  Điều chỉnh thời gian đóng/mở (timing) của các transistor điều khiển hàng/cột
  (Gate/Source Driver) và cấu hình chống méo tín hiệu.

* **FRAME RATIO CONTROL** (``0xB1``)
  Điều chỉnh tần số quét khung hình (Refresh rate) của màn hình ở chế độ màu RGB
  chuẩn (ví dụ: 70Hz, 80Hz).

* **DISPLAY FUNCTION CONTROL** (``0xB6``)
  Thiết lập chế độ quét của Driver (quét từ trên xuống/dưới lên, cấu hình giao
  tiếp bộ nhớ RGB/MCU).

-------------------------------------------------------------------------------

.. _lcd-ili9341-display-format:

3. Hiển Thị & Định Dạng Dữ Liệu (Display & Format)
--------------------------------------------------

* **MEMORY ACCESS CONTROL (MADCTL)** (``0x36``)
  Quyết định hướng quét bộ nhớ (xoay màn hình, lật gương chiều ngang/dọc) và thứ
  tự phối màu (**RGB** hay **BGR**).

* **PIXEL FORMAT** (``0x3A``)
  Định dạng độ phân giải màu cho mỗi điểm ảnh. Giá trị ``0x55`` đại diện cho chuẩn
  **MCU 16-bit/pixel** (RGB565 - 5 bit Red, 6 bit Green, 5 bit Blue).

* **EXIT SLEEP** (``0x11``)
  Đưa chip ra khỏi chế độ ngủ (Sleep Mode) và kích hoạt mạch tạo dao động nội bộ
  (yêu cầu chờ khoảng ``120ms`` để điện áp ổn định).

* **TURN ON DISPLAY** (``0x29``)
  Bật các đường xuất tín hiệu ra màn hình để hiển thị dữ liệu hình ảnh từ RAM ra
  tấm nền TFT.

.. note::
   Hai lệnh cuối là bước "chốt" của chuỗi khởi tạo: chỉ sau khi đã cấu hình xong
   nguồn, định thời và định dạng điểm ảnh thì chip mới được đánh thức
   (``EXIT SLEEP``) và bật hiển thị (``TURN ON DISPLAY``).

-------------------------------------------------------------------------------

.. _lcd-ili9341-gamma-correction:

4. Hiệu Chỉnh Màu Sắc (Gamma Correction)
-----------------------------------------

* **3GAMMA FUNCTION DISABLE** (``0xF2``)
  Bật hoặc tắt chức năng điều chỉnh 3-Gamma nội bộ.

* **GAMMA CURVE SELECTED** (``0x26``)
  Chọn 1 trong 4 đường cong Gamma mặc định được cài đặt sẵn trong chip.

* **POSITIVE GAMMA CORRECTION** (``0xE0``) & **NEGATIVE GAMMA CORRECTION** (``0xE1``)
  Bảng hiệu chỉnh đường cong Gamma (cho điện áp cực dương và cực âm). Các tham số
  này tinh chỉnh độ sáng, độ tương phản và độ trung thực màu sắc của tấm nền TFT.