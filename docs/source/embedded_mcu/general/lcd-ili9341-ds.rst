.. _lcd-ili9341-initialization:

================================================================================
Khởi Tạo Màn Hình LCD ILI9341 (STM32 HAL)
================================================================================

.. contents:: Mục Lục Bài Viết
   :depth: 2
   :local:

.. rst-class:: lead

   Khởi tạo **ILI9341** thực chất chỉ gồm một việc: đẩy **Command** và
   **Parameter** lên bus **SPI** theo đúng **thứ tự** và đúng **thời gian**.
   Vì vậy bài viết này được trình bày bằng **sơ đồ** để *nhìn* là hiểu.

.. grid:: 2 2 4 4
   :gutter: 3

   .. grid-item-card:: ⚡ 1. Power Control & Sequence
      :link: lcd-ili9341-power-control
      :link-type: ref
      :class-card: sd-shadow-sm

      Sinh và ổn định điện áp nội bộ: **VCORE, VCI, VCOM, GVDD, VGH/VGL**.

   .. grid-item-card:: ⏱️ 2. Driver Timing Control
      :link: lcd-ili9341-driver-timing
      :link-type: ref
      :class-card: sd-shadow-sm

      Thời gian đóng/mở của **Gate / Source driver** và tần số quét khung.

   .. grid-item-card:: 🖼️ 3. Display & Pixel Format
      :link: lcd-ili9341-display-format
      :link-type: ref
      :class-card: sd-shadow-sm

      Hướng quét **MADCTL**, độ sâu màu **RGB565**, thức tỉnh & bật hiển thị.

   .. grid-item-card:: 🎨 4. Gamma Correction
      :link: lcd-ili9341-gamma-correction
      :link-type: ref
      :class-card: sd-shadow-sm

      Uốn đường cong **Gamma** cho độ sáng, tương phản và trung thực màu.

.. rubric:: Bức tranh toàn cảnh — 6 tầng của một chuỗi khởi tạo

.. code-block:: text
   :caption: Thứ tự cố định: nguồn → định thời → định dạng → màu → thức tỉnh → bật hiển thị

   ┌──────────────────────────────────────────────────────────────────────►
   │ ① POWER CONTROL & SEQUENCE   nguồn nội bộ: VCORE, VCOM, GVDD, VGH/VGL
   │ ② DRIVER TIMING CONTROL      gate / source driver + thời gian quét hàng
   │ ③ DISPLAY & PIXEL FORMAT     MADCTL (hướng quét) + RGB565 (độ sâu màu)
   │ ④ GAMMA CORRECTION           đường cong sáng/tối theo từng mức xám
   │ ⑤ EXIT SLEEP                 đánh thức dao động nội, bắt buộc chờ ≥ 120 ms
   │ ⑥ TURN ON DISPLAY            xuất RAM ra tấm nền → hình hiện ra
   └──────────────────────────────────────────────────────────────────────►

.. grid:: 1 2 2 2
   :gutter: 3

   .. grid-item-card:: 🧱 Standard Command — application level
      :class-card: sd-shadow-sm

      Thao tác "hằng ngày": đặt tọa độ, chọn màu, xoay/lật màn hình, xóa màn
      hình, sleep/wake — xuất hiện trong mọi driver ILI9341.

   .. grid-item-card:: ⚙️ Extended Command — cấu hình phần cứng
      :class-card: sd-shadow-sm

      Can thiệp sâu vào chip: tăng/giảm tần số quét, hiệu chỉnh gamma, tốc độ
      bus giao tiếp và các chế độ tiết kiệm điện.

.. rubric:: Cheat sheet — cả chuỗi khởi tạo chỉ là một dãy lệnh

.. code-block:: text
   :caption: Đọc như một câu: reset → cấu hình → thức tỉnh → bật hiển thị

   [ 0x01 ]─►[ 0xCB ]─►[ 0xCF ]─►[ 0xED ]─►[ 0xF7 ]─►[ 0xC0 ]
       ┌────────────────────────────────────────────────────┘
       ▼
   [ 0xC1 ]─►[ 0xC5 ]─►[ 0xC7 ]─►[ 0xE8 ]─►[ 0xEA ]─►[ 0xB1 ]
       ┌────────────────────────────────────────────────────┘
       ▼
   [ 0xB6 ]─►[ 0x36 ]─►[ 0x3A ]─►[ 0xF2 ]─►[ 0x26 ]─►[ 0xE0 ]
       ┌────────────────────────────────────────────────────┘
       ▼
   [ 0xE1 ]─►[ 0x11 ]──(≥ 120 ms)──►[ 0x29 ]──► hình hiện ra

.. rubric:: Bus nói chuyện với panel như thế nào?

.. code-block:: text
   :caption: Chỉ có hai loại byte được gửi đi: Command và Parameter

   STM32 (SPI Master)                      ILI9341 (SPI Slave)
        │ SCK   ───────────────────────►   chốt từng bit theo nhịp xung
        │ MOSI  ───────────────────────►   Command  →  thanh ghi nội bộ
        │ CS    ───────────────────────►   chọn chip cho mỗi khung truyền
        │ D/CX  ───────────────────────►   0 = Command, 1 = Parameter
        └ MISO  ◄───────────────────────   đọc ID / đọc lại RAM

.. code-block:: text
   :caption: Một khung truyền: D/CX quyết định byte đó là Command hay Parameter

   D/CX = 0              D/CX = 1
   ┌──────────┐          ┌──────────┬──────────┬──────────┐
   │   0x3A   │          │   0x55   │   ...    │   ...    │
   └──────────┘          └──────────┴──────────┴──────────┘
   1 byte COMMAND        mỗi byte là 1 PARAMETER của lệnh trước đó

   Ví dụ: gửi lệnh PIXEL FORMAT (0x3A) = 0x55 (RGB565)

   CS       ▼────────────────────▲
   D/CX          0         1
            ┌─────────┬──────────┐
   MOSI     │  0x3A   │   0x55   │
            └─────────┴──────────┘
            └─ COMMAND ┴─ PARAMETER

   Đây là "hợp đồng" giữa MCU và panel: sai một byte → sai màu, sai hướng
   quét, hoặc màn hình không hiện gì.

-------------------------------------------------------------------------------

.. _lcd-ili9341-power-control:

1. Điều Khiển Khởi Tạo & Nguồn (Power Control & Sequence)
---------------------------------------------------------

.. rubric:: Cây nguồn nội bộ — lệnh nào sinh ra đường điện áp nào

.. code-block:: text
   :caption: Điện áp VCI đi vào và các đường điện áp được sinh ra bên trong chip

   VCI (2.5 V – 3.3 V)
     │
     ├─ [ 0xCB / 0xCF ]  ─► VCORE           ──► khối logic nội bộ
     ├─ [ 0xF7 ]         ─► charge pump     ──► DDVDH = 2 × VCI
     ├─ [ 0xC0 VRH ]     ─► mức GVDD        ──► độ tương phản
     ├─ [ 0xC1 BT ]      ─► boost factor    ──► VGH / VGL ──► gate driver
     ├─ [ 0xC1 SAP ]     ─► dòng op-amp     ──► mức xám ổn định
     └─ [ 0xC5 / 0xC7 ]  ─► VCOM            ──► điện áp chung tấm nền

.. rubric:: Trình tự gửi lệnh nhóm nguồn

.. code-block:: text
   :caption: Reset trước, cấu hình nguồn sau — không được đảo thứ tự

   [ 0x01 ]  SOFTWARE RESET   ── reset thanh ghi & RAM về mặc định
       │
       ▼  (chờ ~5 ms rồi mới gửi lệnh tiếp)
   [ 0xCB ]  POWER CONTROL A  ── VCORE / DDVDH / bit tiết kiệm điện pceq
   [ 0xCF ]  POWER CONTROL B  ── chế độ ổn áp & dòng nạp
       │
       ▼
   [ 0xED ]  POWER ON SEQ     ── thứ tự & độ trễ cấp nguồn (soft-start)
   [ 0xF7 ]  PUMP RATIO       ── tỷ lệ mạch bơm (charge pump)
       │
       ▼
   [ 0xC0 ]  POWER CONTROL 1  ── VRH[5:0]  → mức GVDD
   [ 0xC1 ]  POWER CONTROL 2  ── SAP[2:0] / BT[3:0]
       │
       ▼
   [ 0xC5 ]  VCM CONTROL 1    ── VCOM dương
   [ 0xC7 ]  VCM CONTROL 2    ── VCOM âm (tinh chỉnh)
       │
       ▼
   ✅ nguồn nội bộ ổn định  →  sang tầng ② Driver Timing

.. list-table:: Tra cứu nhanh — nhóm lệnh cấp nguồn
   :header-rows: 1
   :widths: 12 22 66

   * - Mã
     - Tên lệnh
     - Nhiệm vụ (một dòng)
   * - ``0x01``
     - SOFTWARE RESET
     - Đưa thanh ghi và RAM về mặc định bằng phần mềm, không cần chân RESET.
   * - ``0xCB``
     - POWER CONTROL A
     - Cấu hình điện áp nội bộ VCORE/DDVDH và bit tiết kiệm điện ``pceq``.
   * - ``0xCF``
     - POWER CONTROL B
     - Chọn chế độ ổn áp, dòng nạp và thời gian ổn định cho các đường nguồn.
   * - ``0xED``
     - POWER ON SEQUENCE
     - Đặt độ trễ và thứ tự bật các đường nguồn để chip không bị sốc điện.
   * - ``0xF7``
     - PUMP RATIO CONTROL
     - Điều chỉnh tỷ lệ mạch bơm dòng tạo ra điện áp cao cho tinh thể lỏng.
   * - ``0xC0``
     - POWER CONTROL 1
     - ``VRH[5:0]`` = mức GVDD, ảnh hưởng trực tiếp tới độ tương phản.
   * - ``0xC1``
     - POWER CONTROL 2
     - ``SAP[2:0]`` dòng op-amp nội bộ, ``BT[3:0]`` tỷ lệ nâng áp (boost).
   * - ``0xC5`` / ``0xC7``
     - VCOM CONTROL 1 / 2
     - Chỉnh điện áp **VCOM** chung cho mọi điểm ảnh → giảm nhấp nháy.

.. tip::
   Nếu hình hiển thị đúng nhưng **nhấp nháy (flicker)** hoặc ám màu nhẹ thì
   gần như luôn nằm ở ``0xC5`` / ``0xC7`` — không phải ở phần màu hay gamma.

-------------------------------------------------------------------------------

.. _lcd-ili9341-driver-timing:

2. Định Thời & Xung Nhịp Driver (Driver Timing Control)
-------------------------------------------------------

.. rubric:: Hai driver phải "đóng/mở" đúng nhịp

.. code-block:: text
   :caption: Gate driver quét theo hàng, Source driver nạp dữ liệu theo cột

   Gate driver (quét theo HÀNG)          Source driver (nạp theo CỘT)
     │ STV ──► dịch hàng 1 → 320         │ 1 hàng ◄── dữ liệu RGB565
     │ CLK ──► │││││││││││││││││││       │ nạp vào tụ của từng điểm ảnh
     └ 0xE8 / 0xEA: timing & chống méo    └ 0xC1 SAP: dòng op-amp nguồn

.. code-block:: text
   :caption: Một frame và phần mà 0xB1 điều khiển

   1 frame  ≈  60 – 70 Hz
   ╞═════════════════════════════════════════════════════════╡
   │  quét hàng 1 → hàng 2 → ... → hàng 320  │  blanking
   ╞═════════════════════════════════════════════════════════╡
   ▲                                                         ▲
   └────── [ 0xB1 ] DIVA + RTNA + RTNB ──────────────────────┘
              giá trị lớn hơn → tần số quét thấp hơn

.. list-table:: Tra cứu nhanh — nhóm lệnh định thời
   :header-rows: 1
   :widths: 12 24 64

   * - Mã
     - Tên lệnh
     - Nhiệm vụ (một dòng)
   * - ``0xE8``
     - DRIVER TIMING CONTROL A
     - Thời gian bật/tắt transistor của Gate/Source driver (timing, EQ).
   * - ``0xEA``
     - DRIVER TIMING CONTROL B
     - Tinh chỉnh pre-charge và timing của gate driver.
   * - ``0xB1``
     - FRAME RATIO CONTROL
     - Chọn tần số quét khung (ví dụ 70 Hz, 80 Hz) qua ``DIVA``, ``RTNA``, ``RTNB``.
   * - ``0xB6``
     - DISPLAY FUNCTION CONTROL
     - Chọn chế độ quét của driver (``PT``, ``REV``, ``GS``, ``SS``) và cách nối RAM RGB.

.. note::
   ``0xB1`` là lệnh **duy nhất** trong nhóm này có thể chỉnh "mượt/giật" của
   hình. Công thức quy đổi từ giá trị ghi vào tần số quét thực tế nằm ở phần
   FRAME RATIO CONTROL của datasheet — nên chép theo bảng tham chiếu của panel.

-------------------------------------------------------------------------------

.. _lcd-ili9341-display-format:

3. Hiển Thị & Định Dạng Dữ Liệu (Display & Format)
---------------------------------------------------

.. rubric:: MADCTL (0x36) — một byte quyết định hướng quét và thứ tự màu

.. code-block:: text
   :caption: Bản đồ bit của MEMORY ACCESS CONTROL

          bit7  bit6  bit5  bit4  bit3  bit2  bit1  bit0
         ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
         │ MY  │ MX  │ MV  │ ML  │ BGR │ MH  │ --  │ --  │
         └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
            │     │     │     │     │     │
            │     │     │     │     │     └────── MH: thứ tự refresh ngang
            │     │     │     │     └──────────── BGR: 0 = RGB, 1 = BGR
            │     │     │     └────────────────── ML: thứ tự refresh dọc
            │     │     └──────────────────────── MV: đổi chỗ hàng ↔ cột
            │     └────────────────────────────── MX: mirror trục X
            └──────────────────────────────────── MY: mirror trục Y

.. list-table:: Xoay màn hình — giá trị MADCTL thường dùng trong code driver
   :header-rows: 1
   :widths: 22 16 22 40

   * - Hướng
     - Giá trị
     - Bit bật
     - Hệ quả
   * - Portrait 0°
     - ``0x48``
     - ``MX`` + ``BGR``
     - Mặc định: 240 × 320
   * - Landscape 90°
     - ``0x28``
     - ``MV`` + ``BGR``
     - Đổi chỗ hàng/cột: 320 × 240
   * - Portrait 180°
     - ``0x88``
     - ``MY`` + ``BGR``
     - Lật dọc so với 0°
   * - Landscape 270°
     - ``0xE8``
     - ``MV`` + ``MX`` + ``MY`` + ``BGR``
     - Xoay ngược chiều so với 90°

.. warning::
   Giá trị trên đúng với phần lớn module ILI9341 2.4" – 2.8", nhưng **tùy tấm
   nền**. Nếu hình bị lật hoặc đỏ ↔ xanh hoán đổi, chỉ cần sửa ``MY/MX/MV``
   hoặc bỏ/giữ bit ``BGR`` trong ``0x36`` — không cần đổi đường dây phần cứng.

.. rubric:: PIXEL FORMAT (0x3A) — quyết định độ sâu màu

.. code-block:: text
   :caption: 0x3A = 0x55 → MCU 16 bit/pixel = RGB565 (1 pixel = 2 byte)

   ┌───────────┬─────────────┬───────────┐
   │ R[15:11]  │  G[10:5]    │ B[4:0]    │
   └───────────┴─────────────┴───────────┘
   5 bit Đỏ        6 bit Xanh lá     5 bit Xanh dương
   → Mắt người nhạy với kênh Xanh lá nhất, nên nó được cấp nhiều bit nhất.

.. list-table:: Vài giá trị RGB565 để nhớ nhanh
   :header-rows: 1
   :widths: 22 18 60

   * - Màu
     - Hex
     - Bit (R5 | G6 | B5)
   * - Trắng
     - ``0xFFFF``
     - ``11111 | 111111 | 11111``
   * - Đen
     - ``0x0000``
     - ``00000 | 000000 | 00000``
   * - Đỏ
     - ``0xF800``
     - ``11111 | 000000 | 00000``
   * - Xanh lá
     - ``0x07E0``
     - ``00000 | 111111 | 00000``
   * - Xanh dương
     - ``0x001F``
     - ``00000 | 000000 | 11111``

.. rubric:: Hai lệnh "chốt" cuối chuỗi — thiếu một trong hai là màn hình trắng

.. code-block:: text
   :caption: EXIT SLEEP → chờ ≥ 120 ms → TURN ON DISPLAY

   [ 0x11 ]  EXIT SLEEP ──► khởi động mạch dao động nội bộ
        │
        └──► bắt buộc chờ ≥ 120 ms   ◄── thiếu delay → trắng / nhiễu hình
                 │
                 ▼
   [ 0x29 ]  TURN ON DISPLAY ──► xuất dữ liệu RAM ra tấm nền TFT

.. list-table:: Tra cứu nhanh — nhóm lệnh hiển thị & định dạng
   :header-rows: 1
   :widths: 12 26 62

   * - Mã
     - Tên lệnh
     - Nhiệm vụ (một dòng)
   * - ``0x36``
     - MEMORY ACCESS CONTROL
     - Chọn hướng quét bộ nhớ (xoay/lật) và thứ tự phối màu **RGB** hay **BGR**.
   * - ``0x3A``
     - PIXEL FORMAT
     - Chọn độ sâu màu mỗi điểm ảnh; ``0x55`` = MCU 16 bit/pixel (RGB565).
   * - ``0x11``
     - EXIT SLEEP
     - Đưa chip ra khỏi chế độ ngủ, kích hoạt dao động nội (phải chờ ≥ 120 ms).
   * - ``0x29``
     - TURN ON DISPLAY
     - Bật đường xuất tín hiệu để dữ liệu RAM được đưa ra tấm nền TFT.

.. admonition:: Thứ tự là ràng buộc, không phải gợi ý
   :class: warning

   Chỉ khi **nguồn → định thời → định dạng điểm ảnh** đã xong thì chip mới được
   đánh thức (``0x11``) và bật hiển thị (``0x29``). Gửi ``0x29`` trước khi cấu
   hình ``0x3A`` → độ sâu màu sai; bỏ ``0x11`` → chip vẫn ngủ, panel trắng.

-------------------------------------------------------------------------------

.. _lcd-ili9341-gamma-correction:

4. Hiệu Chỉnh Màu Sắc (Gamma Correction)
----------------------------------------

.. rubric:: Vì sao phải có hai bảng gamma?

.. code-block:: text
   :caption: Tấm nền được lái bằng điện áp xoay dấu (AC) để không "cháy" tinh thể lỏng

   frame chẵn  ──►  cực dương  ──►  dùng bảng 0xE0 (V0  → V14)
   frame lẻ    ──►  cực âm    ──►  dùng bảng 0xE1 (V14 → V0)
                         │
                         └──► 2 bảng phải ĐỐI XỨNG
                              lệch nhau → nhấp nháy (flicker) dù màu vẫn đúng

.. code-block:: text
   :caption: Gamma = đường cong ánh xạ "mã xám" sang "độ sáng thực tế"

   độ sáng
     ▲                                            ●
     │                                         ●
     │                                     ●            ← đường cong lý tưởng
     │                                ●                 (≈ 2.2, mắt người)
     │                          ●
     │                   ●
     │           ●
     │  ●
     └──────────────────────────────────────────────►  mã xám
        0x00                                   0xFF

    0xE0 (POSITIVE) uốn nửa dương ─┐
                                   ├──► cùng quyết định: độ sáng, tương phản, màu
    0xE1 (NEGATIVE) uốn nửa âm  ───┘

.. list-table:: Tra cứu nhanh — nhóm lệnh gamma
   :header-rows: 1
   :widths: 12 28 60

   * - Mã
     - Tên lệnh
     - Nhiệm vụ (một dòng)
   * - ``0xF2``
     - 3GAMMA FUNCTION DISABLE
     - Bật/tắt chức năng 3-Gamma nội bộ (``0x00`` = tắt — mặc định).
   * - ``0x26``
     - GAMMA CURVE SELECTED
     - Chọn 1 trong 4 đường cong dựng sẵn bên trong chip (xem bảng dưới).
   * - ``0xE0``
     - POSITIVE GAMMA CORRECTION
     - 15 byte ``V0 … V14`` uốn đường cong cho cực dương.
   * - ``0xE1``
     - NEGATIVE GAMMA CORRECTION
     - 15 byte ``V0 … V14`` uốn đường cong cho cực âm.

.. list-table:: Bốn đường cong dựng sẵn của lệnh ``0x26`` (theo datasheet)
   :header-rows: 1
   :widths: 18 16 66

   * - Giá trị ghi
     - Gamma
     - Đặc điểm khi nhìn bằng mắt
   * - ``0x01``
     - 2.2
     - Mặc định — màu trung thực nhất, dùng cho gần như mọi ứng dụng.
   * - ``0x02``
     - 1.8
     - Sáng hơn — hợp giao diện nhiều chữ, nền tối.
   * - ``0x04``
     - 2.5
     - Tương phản cao hơn — ảnh/nền sáng trông "sâu" hơn.
   * - ``0x08``
     - 1.0
     - Tuyến tính — dùng để đối chiếu, hiệu chuẩn.

-------------------------------------------------------------------------------

.. rubric:: Đối chiếu code — các sơ đồ trên khi viết thành hàm

.. code-block:: c
   :caption: Cùng một thứ tự, chỉ khác ngôn ngữ

   ILI9341_WriteCmd(0x01);                    /* SOFTWARE RESET          */
   HAL_Delay(5);

   ILI9341_WriteCmd(0xCB); ILI9341_WriteData(0x39);  /* POWER CONTROL A   */
   ILI9341_WriteCmd(0xCF); ILI9341_WriteData(0x00);  /* POWER CONTROL B   */
   ILI9341_WriteCmd(0xED); ILI9341_WriteData(0x64);  /* POWER ON SEQUENCE */
   ILI9341_WriteCmd(0xF7); ILI9341_WriteData(0x20);  /* PUMP RATIO        */
   ILI9341_WriteCmd(0xC0); ILI9341_WriteData(0x23);  /* POWER CONTROL 1   */
   ILI9341_WriteCmd(0xC1); ILI9341_WriteData(0x10);  /* POWER CONTROL 2   */
   ILI9341_WriteCmd(0xC5); ILI9341_WriteData(0x3E, 0x28);   /* VCOM 1     */

   ILI9341_WriteCmd(0x36); ILI9341_WriteData(0x48);  /* 0°  : MX | BGR    */
   ILI9341_WriteCmd(0x3A); ILI9341_WriteData(0x55);  /* RGB565, 16 bit    */

   ILI9341_WriteCmd(0x11);                    /* EXIT SLEEP             */
   HAL_Delay(120);                            /* bắt buộc ≥ 120 ms      */
   ILI9341_WriteCmd(0x29);                    /* TURN ON DISPLAY        */

.. warning::
   Bộ tham số trên chỉ minh hoạ **thứ tự và độ dài** của lệnh. Mỗi lệnh có số
   parameter khác nhau (``0xC5`` cần 2 byte, ``0xE0`` cần tới 15 byte...), nên
   lấy bộ giá trị chuẩn từ driver mẫu của nhà sản xuất module để không sai bit.

.. rubric:: Dashboard chẩn đoán — nhìn triệu chứng, tra ra lệnh

.. list-table::
   :header-rows: 1
   :widths: 26 30 44

   * - Triệu chứng
     - Nghi ngờ chính
     - Kiểm tra ngay
   * - Trắng bóc, không có hình
     - Thiếu ``0x11`` / ``0x29`` hoặc thiếu delay
     - Đã chờ ≥ 120 ms sau ``0x11`` rồi mới gửi ``0x29``?
   * - Đỏ ↔ Xanh dương hoán đổi
     - Bit ``BGR`` trong ``0x36``
     - Thử ``0x36 = 0x48`` và ``0x36 = 0x40`` xem cái nào đúng màu
   * - Hình bị lật / xoay không đúng
     - ``MY``, ``MX``, ``MV`` trong ``0x36``
     - Đối chiếu bảng xoay màn hình ở mục 3
   * - Màu vỡ, răng cưa, chữ bị xé
     - Tốc độ SPI hoặc ``0x3A``
     - ``0x3A`` có đúng ``0x55``? Thử giảm prescaler SPI
   * - Nhấp nháy nhẹ, ám màu
     - ``0xC5`` / ``0xC7`` (VCOM)
     - Tinh chỉnh theo bảng VCOM tham chiếu của module
   * - Vệt ngang chạy qua màn hình
     - ``0xB1`` / ``0xB6``
     - Tần số quét và hướng quét gate có khớp panel?
   * - Tối thui, không sáng
     - Phần cứng: backlight, RESET, CS
     - Đo chân BL, kiểm tra xung RESET và mức CS

.. rubric:: Checklist 3 bước trước khi kết luận "lỗi phần cứng"

.. grid:: 1 1 3 3
   :gutter: 3

   .. grid-item-card:: ① Nguồn
      :text-align: center
      :class-card: sd-shadow-sm

      ``0xCB 0xCF 0xED 0xF7``

      ``0xC0 0xC1 0xC5 0xC7``

   .. grid-item-card:: ② Định dạng
      :text-align: center
      :class-card: sd-shadow-sm

      ``0x36`` hướng quét

      ``0x3A = 0x55`` (RGB565)

   .. grid-item-card:: ③ Chốt
      :text-align: center
      :class-card: sd-shadow-sm

      ``0x11`` → ≥ 120 ms

      → ``0x29``

.. rubric:: Tóm lại trong một hình

.. code-block:: text

   nguồn ổn định  ──►  timing đúng  ──►  MADCTL + RGB565 khớp panel
        │
        └──►  gamma đối xứng  ──►  0x11 ─(≥ 120 ms)─►  0x29  ──►  hình hiện ra
