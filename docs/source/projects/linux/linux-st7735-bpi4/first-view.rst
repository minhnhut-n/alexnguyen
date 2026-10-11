1. First View
=============

Các Thuật Ngữ Cốt Lõi Khi Viết Driver ST7735 Cho Board BSP
----------------------------------------------------------

.. contents:: Mục Lục Bài Viết
   :depth: 2
   :local:

Luồng Hoạt Động (System Workflow Diagram)
-----------------------------------------

.. code-block:: text
   :caption: Ba bước cơ bản để vẽ mọi thứ lên màn hình

   +------------------+         +------------------+         +------------------+
   |  1. RESET & INIT |  -----> | 2. ADDRESS WIN   |  -----> | 3. DATA TRANSFER |
   |  RST Pin / SW    |         | CASET / RASET    |         | RAMWR / SPI DMA  |
   +------------------+         +------------------+         +------------------+
           |                             |                            |
           v                             v                            v
       [DC = 0]                      [DC = 0]                     [DC = 1]
     (Gửi Lệnh C)                  (Khoanh Vùng)                 (Đổ Màu RGB)

1. Giao Diện Phần Cứng (Hardware Pinout)
----------------------------------------

.. list-table:: Các chân tín hiệu cần thiết để điều khiển ST7735
   :header-rows: 1
   :widths: 22 26 52

   * - Chân (Pin)
     - Loại Tín Hiệu
     - Vai Trò Chính
   * - ``SCLK`` / ``MOSI``
     - SPI Serial
     - Truyền xung clock và dữ liệu
   * - ``CS``
     - Digital Output
     - Chọn thiết bị slave
   * - ``DC`` (RS)
     - Control Signal
     - Phân biệt Lệnh hay Dữ liệu
   * - ``RST`` / ``BLK``
     - Power/Reset
     - Reset phần cứng & Đèn nền

* **Chi tiết chức năng:**

  * **SCLK / MOSI (SDA):** Đường truyền dữ liệu chính qua chuẩn giao tiếp SPI
    đồng bộ.
  * **CS (Chip Select):** Kéo xuống ``LOW`` để kích hoạt giao tiếp với chip
    ST7735.
  * **DC (Data/Command Select):**

    * ``LOW`` (0): Chip hiểu byte gửi sang là **Lệnh (Command/Opcode)**.
    * ``HIGH`` (1): Chip hiểu byte gửi sang là **Dữ liệu pixel (Pixel Data)**.

  * **RST (Reset):** Kéo ``LOW`` rồi lên ``HIGH`` để đưa chip về trạng thái
    ban đầu.
  * **BLK (Backlight):** Bật/tắt hoặc điều khiển xung PWM chỉnh độ sáng màn
    hình.

2. Lệnh & Giao Thức Hiển Thị (ST7735 Protocol)
----------------------------------------------

.. list-table:: Bộ lệnh cơ bản của ST7735
   :header-rows: 1
   :widths: 25 30 45

   * - Thuật ngữ
     - Mã Lệnh / Chuẩn
     - Ý Nghĩa Tổng Quan
   * - Software Init
     - ``SLPOUT``, ``DISPON``
     - Kích hoạt màn hình
   * - Address Win
     - ``CASET``, ``RASET``
     - Định vị vùng vẽ (X, Y)
   * - Memory Write
     - ``RAMWR``
     - Bắt đầu nạp dữ liệu màu
   * - Color Format
     - ``RGB565``
     - Mã hóa màu 16-bit

* **Chi tiết cấu trúc lệnh:**

  * **Software Init (0x11 / 0x29):** Đưa chip thoát chế độ Sleep (``SLPOUT``)
    và Bật hiển thị (``DISPON``).
  * **CASET (0x2A) & RASET (0x2B):** Thiết lập giới hạn cột
    (:math:`X_{start} \to X_{end}`) và hàng (:math:`Y_{start} \to Y_{end}`).
  * **RAMWR (0x2C):** Lệnh ghi vào VRAM. Ngay sau lệnh này, giữ ``DC = 1`` để
    gửi liên tiếp các byte màu.
  * **RGB565:** Định dạng 16-bit (5-bit Red, 6-bit Green, 5-bit Blue). Mỗi
    điểm ảnh chiếm đúng **2 Bytes** trên đường truyền SPI.

3. Tối Ưu Tầng Phần Mềm (Software Optimization)
-----------------------------------------------

.. list-table:: Các kỹ thuật tối ưu khi viết driver
   :header-rows: 1
   :widths: 25 30 45

   * - Thuật ngữ
     - Cơ Chế
     - Mục Tiêu Tối Ưu
   * - Frame Buffer
     - RAM Cục Bộ
     - Bộ nhớ đệm bức ảnh
   * - SPI DMA
     - Hardware Offload
     - Giảm tải CPU, tăng FPS
   * - Alignment
     - Offset / Endianness
     - Sửa lỗi lệch hình & sai màu

* **Chi tiết giải pháp:**

  * **Frame Buffer:** Mảng dữ liệu trên RAM vi điều khiển đại diện cho toàn bộ
    màn hình.
  * **SPI DMA (Direct Memory Access):** Đẩy dữ liệu từ Frame Buffer ra SPI
    hoàn toàn tự động mà không làm treo CPU.
  * **Screen Offset:** Xử lý lệch tọa độ gốc :math:`(0, 0)` trên một số màn
    hình cắt giảm (như :math:`128 \times 128` hoặc :math:`80 \times 160`).
  * **Byte Swapping:** Đảo byte cao/thấp (High/Low Byte) của chuẩn RGB565 nếu
    hiển thị bị lộn màu (ví dụ: Đỏ bị thành Xanh).