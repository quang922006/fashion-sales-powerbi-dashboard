Bạn là Lead Power BI Architect & TMDL/PBIR Automation Engineer. Nhiệm vụ của bạn là đọc 13 file dữ liệu CSV trong thư mục `Data/` (file mẫu dự báo 548 ngày sẽ được bổ sung sau), xây dựng Data Model (Star Schema), tạo Measures DAX chuẩn hóa và thiết kế hoàn chỉnh hệ thống Báo cáo Power BI 8 trang theo đúng các đặc tả dưới đây:

---

### 1. MỤC TIÊU DASHBOARD (BUSINESS OBJECTIVE)
* **North Star Question:** Giải mã nguyên nhân số đơn hàng sụt giảm ~58% từ năm 2015 (82,622 đơn) xuống 2021 (34,525 đơn) — đặc biệt là điểm gãy sụt giảm sâu năm 2019 (từ 69,510 xuống 41,601 đơn) — trong khi lượt truy cập website tăng từ 6.8 triệu (2013) lên 11.1 triệu (2022) khiến tỷ lệ chuyển đổi (Conversion Rate) rơi từ 1.13% xuống 0.33%.
* **Mục tiêu hành động:** Xác định các đòn bẩy kinh doanh trọng yếu (giá thực thu, cơ cấu khuyến mãi, kênh phân phối, danh mục sản phẩm, chất lượng vận hành giao vận) để chặn đà suy giảm, tối ưu hóa biên lợi nhuận gộp và quản trị rủi ro tồn kho.
* **Mục tiêu kỹ thuật:** Xây dựng mô hình dữ liệu chuẩn hóa, đối soát dữ liệu (Reconciliation) khớp 100% giữa các bảng nguồn, và theo dõi đường dự báo 548 ngày (2023-01-01 đến 2024-07-01).

---

### 2. ĐỐI TƯỢNG SỬ DỤNG & NHU CẦU RA QUYẾT ĐỊNH (AUDIENCE / USERS)
1. **Ban Giám đốc (C-Suite / Executives):** Theo dõi Doanh thu thuần (Net Revenue), Biên lợi nhuận gộp (Gross Margin %), xu hướng YoY/MoM; ra quyết định phân bổ ngân sách, đầu tư hoặc cắt giảm ngành hàng/kênh.
2. **Kinh doanh & Merchandising (Sales & Category Leads):** Phân tích hiệu quả theo Category (Streetwear, Casual, Outdoor, GenZ), Segment, Size, Color; tối ưu hóa giá trị đơn hàng (AOV), số món mỗi đơn (UPT) và thực thu giá niêm yết (Price Realization).
3. **Marketing & Tăng trưởng (Marketing & Growth Team):** Đo lường hiệu quả phễu chuyển đổi Web Funnel, cơ cấu doanh số theo kênh (Organic, Paid Search, Social Media...), hiệu quả các chương trình khuyến mãi (Promo ROI) và giữ chân khách hàng (Cohort Retention).
4. **Vận hành & Chuỗi cung ứng (Supply Chain & Logistics):** Giám sát thời gian chuẩn bị & vận chuyển (Prep & Transit Time), tỷ lệ giao đúng hạn (On-Time SLA <= 5 ngày), vòng quay hàng tồn (Cover Days, Sell-Through Rate, Stockout Days).
5. **Dịch vụ Khách hàng & Chất lượng (CSKH & CX):** Kiểm soát tỷ lệ trả hàng, phân tích sâu 5 lý do hoàn tiền (Late delivery, Wrong size, Defective...) và xử lý các sản phẩm có tỷ lệ đánh giá tiêu cực (Rating <= 2 sao).
6. **Data & Tài chính (Finance & BI / Planning):** Kiểm toán chất lượng dữ liệu, đối soát lệch số giữa Line Item, Payment và Sales Aggregate, theo dõi doanh thu thực tế vs. dự báo 548 ngày.

---

### 3. HỆ THỐNG KPI CẦN HIỂN THỊ (DAX MEASURES)
Tổ chức toàn bộ Measure trong bảng `_measures` theo các Folder nghiệp vụ:

* **01_Revenue_Profit:**
  - `M01 Gross Sales` = `SUMX('txn_sales_order_line', quantity * unit_price)`
  - `M02 Discount Amount` = `SUM('txn_sales_order_line'[discount_line_amount])`
  - `M03 Net Sales` = `[M01 Gross Sales] - [M02 Discount Amount]`
  - `M24 Refund Amount` = `SUM('txn_return'[refund_amount])`
  - `M04 Net Revenue` (Chỉ số chuẩn) = `CALCULATE([M03 Net Sales], order_status IN {"delivered", "shipped", "returned"}) - [M24 Refund Amount]`
  - `M05 COGS` = `CALCULATE(SUMX(lines, quantity * RELATED(product_unit_cost)), order_status IN {"delivered", "shipped", "returned"})`
  - `M06 Gross Profit` = `[M04 Net Revenue] - [M05 COGS]`
  - `M07 Gross Margin %` = `DIVIDE([M06 Gross Profit], [M04 Net Revenue])` (~6.20%)
  - `M08 Net Revenue YoY %` = So sánh cùng kỳ qua `SAMEPERIODLASTYEAR`.
* **02_Orders:**
  - `M10 Total Orders` = `DISTINCTCOUNT('txn_sales_order_line'[sales_order_id])`
  - `M10b Eligible Orders` = Lọc các đơn có trạng thái `delivered`, `shipped`, `returned`.
  - `M12 AOV` = `DIVIDE([M04 Net Revenue], [M10b Eligible Orders])`
  - `M13 UPT` = `DIVIDE(SUM(quantity), [M10 Total Orders])`
  - `M14 Cancellation Rate %` = Đơn `cancelled` / Tổng số đơn (~9.2%)
  - `M15 Order Return Rate %` = Đơn `returned` / (Đơn `delivered` + `returned`) (~6.54%)
  - `M15b Returned Order Share %` = Đơn `returned` / Tổng số đơn (~5.59%)
* **03_Promotions_Pricing:**
  - `M20 Discount Rate %` = `DIVIDE([M02 Discount Amount], [M01 Gross Sales])` (~4.56%)
  - `M21 Promo Line Share %` = Số dòng áp mã KM / Tổng số dòng hàng (~38.66%)
  - `M22 Price Realization %` = Thực thu gộp / Giá niêm yết danh mục (~92.86%)
* **04_Fulfillment_Returns:**
  - `M26 Avg Prep Time (Days)` = `AVERAGE(ship_date - order_date)` (~1.5 ngày)
  - `M27 Avg Transit Time (Days)` = `AVERAGE(delivery_date - ship_date)` (~4.5 ngày)
  - `M28 On-Time SLA %` = Tỷ lệ đơn có (Prep + Transit) <= 5 ngày (~41.72%, vì tổng thời gian TB ≈ 6 ngày).
* **05_Customers:**
  - `M30 Active Customers` = Khách hàng có phát sinh đơn hợp lệ.
  - `M31 New` vs `M31b Returning Customers` = Phân loại dựa trên `customer_first_order_date` (tuyệt đối không dùng `signup_date`).
  - `M32 Repeat Purchase Rate %` = Khách có >= 2 đơn / Tổng khách phát sinh đơn (75.23%).
  - `M33 Avg Orders per Customer` = 7.17 đơn/khách.
* **06_Reviews & 07_Inventory:**
  - `M40 Average Rating` (3.94/5.0), `M41 Low Rating Share %` (Tỷ lệ rating <= 2 sao: 13.1%).
  - `M51 Total Stockout Days`, `M52 Avg Sell-Through Rate`, `M53 Median Cover Days` (MEDIAN toàn bộ = 240 ngày; không lọc > 365 vì 38.6% dòng vượt ngưỡng này), `M54 Avg Fill Rate`.
* **08_Web_Forecast:**
  - `M60 Total Sessions`, `M61 Web Conversion Rate %` = `[M10 Total Orders] / [M60 Total Sessions]` (Tính từ năm 2013).
  - `M70 Actual vs Forecast Revenue` (Kết nối thực tế 2012–2022 và dự báo 548 ngày — phần dự báo chờ file mẫu).

---

### 4. DỮ LIỆU NGUỒN & KIẾN TRÚC MÔ HÌNH (DATA ARCHITECTURE)
* **Quy chuẩn nguồn:** Tuyệt đối không can thiệp sửa trực tiếp các file CSV trong `Data/`. Dự án PBIP nằm tại `powerBI_model/fahsion_analysis.*` (giữ nguyên tên hiện có). Mọi biến đổi thực hiện qua Power Query.
* **Mô hình Star Schema:** Thiết lập quan hệ 1 chiều (`1 -> Many, Single Direction`), không kích hoạt Bi-directional.
* **Phân lớp bảng:**
  1. `dim_date`: Bảng lịch DAX liên tục từ `2012-01-01` đến `2024-12-31` (phủ hết 548 ngày dự báo), đánh dấu `Mark as Date Table`.
  2. Bảng Dimension: `md_customer`, `md_geography` (giữ `zip` kiểu Text, không pad 0), `md_product` (giữ nguyên cogs/price chính xác đến số thập phân), `md_promotion`, `md_order_status` (6 trạng thái, cờ `revenue_eligible_flag`), `md_channel`, `md_payment_method`, `md_return_reason`.
  3. Bảng Fact: `txn_sales_order` (646,945 dòng), `txn_sales_order_line` (714,669 dòng, thêm surrogate key `sales_order_line_id`), `txn_payment`, `txn_shipment` (loại 564 đơn thiếu shipment khi tính lead-time), `txn_return`, `txn_review`, `txn_inventory_snapshot`, `txn_web_traffic_daily`.
  4. Bảng Staging/QA: `stg_sales_daily`, `stg_sales_submission_template` (chờ file mẫu dự báo).
* **Phạm vi dữ liệu:** Đơn hàng & `sales.csv` từ `2012-07-04` đến `2022-12-31`; web traffic từ `2013-01-01`. YoY năm 2013 phải ẩn vì 2012 chỉ có ~6 tháng.
* **Benchmark QA bắt buộc (Toàn kỳ 2012–2022):**
  - `M01 Gross Sales` = 16,430,476,585.53
  - `M03 Net Sales` = 15,680,869,265.43
  - `M04 Net Revenue` = 13,210,795,006.44
  - `M05 COGS (Eligible)` = 12,392,176,655.41
  - `M05_All COGS` (mọi trạng thái, làm tròn COGS theo ngày đến 2 chữ số) = 14,163,450,519.17 = Σ `sales.csv[COGS]`
  - `M06 Gross Profit` = 818,618,351.03
  - `M07 Gross Margin %` = 6.20%

---

### 5. CẤU TRÚC BÁO CÁO (8 PAGES BLUEPRINT)
* **Trang 1: Executive Overview (Tổng quan điều hành):**
  - KPI Cards: Net Revenue, Gross Margin %, Eligible Orders, AOV, Cancellation Rate %, Return Rate %, Avg Rating (kèm indicator YoY).
  - Line & Clustered Column: Net Revenue & YoY % theo Tháng/Năm.
  - Clustered Bar: Net Revenue & Gross Margin % theo Category.
  - Combo Chart: Phân kỳ giữa Số đơn hàng (giảm 58%) và AOV (tăng đều).
  - Bar Chart: Doanh thu theo Vùng (East, Central, West) và Top 10 Thành phố.
* **Trang 2: Sales & Merchandising (Bán hàng & Sản phẩm):**
  - Treemap Category × Segment theo Net Revenue.
  - Biểu đồ Pareto tích lũy doanh thu theo SKU (80/20).
  - Matrix Heatmap: Size (S, M, L, XL) × Color (10 màu) hiển thị Units Sold & Margin.
  - Clustered Bar: Thực thu giá (Price Realization) theo Category.
  - Bảng Top 10 / Bottom 10 sản phẩm đóng góp lợi nhuận gộp.
* **Trang 3: Promotions & Pricing (Khuyến mãi & Giá):**
  - KPI Cards: Discount Rate (4.6%), Promo Share (38.7%), Price Realization (92.9%).
  - Scatter Chart: Hiệu quả 50 chiến dịch khuyến mãi (Net Sales vs. Gross Margin %).
  - Line Chart: Xu hướng tỷ lệ chiết khấu & tỷ lệ đơn khuyến mãi theo thời gian.
  - Column Chart: So sánh hiệu quả Không KM vs. 1 KM vs. Xếp chồng 2 KM (Stackable).
* **Trang 4: Customer Analytics (Phân tích Khách hàng):**
  - KPI Cards: Active Customers, Repeat Rate (75.2%), Avg Orders/Customer (7.17), CLV.
  - Stacked Column: Khách hàng mới vs. Khách hàng quay lại theo năm (New vs. Returning).
  - Bar Chart: Cơ cấu độ tuổi (Age Group) × Giới tính (Gender).
  - Matrix: Tỷ lệ giữ chân khách hàng qua các năm (Cohort Retention Matrix).
* **Trang 5: Operations, Returns & CX (Vận hành, Trả hàng & Đánh giá):**
  - KPI Cards: Thời gian chuẩn bị TB (1.5d), Thời gian vận chuyển TB (4.5d), SLA On-Time (41.7%), Tỷ lệ đánh giá thấp (13.1%).
  - Funnel / Horizontal Bar: Phân bổ trạng thái đơn (Delivered, Shipped, Returned, Cancelled, Paid, Created).
  - Bar Chart: Phân tích 5 lý do trả hàng theo số lượng và số tiền hoàn (Refund Amount).
  - Matrix: Tỷ lệ trả hàng theo Category × Size (phát hiện lỗi Wrong Size).
  - Bảng cảnh báo: Top SKU có điểm đánh giá trung bình thấp nhất (< 2.5 sao).
* **Trang 6: Marketing & Web Funnel (Hiệu quả Marketing & Lưu lượng Web):**
  - KPI Cards: Total Sessions, Unique Visitors, Page Views, Conversion Rate (1.13% -> 0.33%).
  - Combo Chart: Tương phản giữa Lượt truy cập Web (Sessions tăng gấp đôi) và Đơn hàng (giảm một nửa).
  - 100% Stacked Bar: Cơ cấu đơn hàng theo kênh thu hút (Order Source) qua các năm (chỉ ra kênh sụt giảm năm 2019).
  - Donut Chart: Phân bổ đơn hàng theo loại thiết bị (Desktop, Mobile, Tablet).
* **Trang 7: Inventory Health (Chỉ báo Sức khỏe Tồn kho):**
  - Banner cảnh báo: Ghi rõ dữ liệu tồn kho là snapshot cuối tháng, độ phủ 67.3% SKU (1,624/2,412 SKU).
  - KPI Cards: Ending Stock On Hand, Avg Fill Rate, Avg Sell-Through Rate, Median Cover Days, Total Stockout Days.
  - Line Chart: Xu hướng biến động Fill Rate và Sell-Through Rate theo tháng.
  - Clustered Bar: Median Cover Days (không lọc) theo Category & Segment.
* **Trang 8: Data Governance & Forecast (Đối soát & Dự báo 548 ngày):**
  - Line Chart liên tục: Actual Revenue (2012–2022) nối dài sang Forecast Revenue (548 ngày 2023–2024) — phần Forecast hiển thị khi có file mẫu dự báo.
  - Data Reconciliation Matrix: Đối chiếu giữa Line Items, Orders, Payments và sales.csv với cột Chênh lệch = 0.
  - Bảng kiểm soát ngoại lệ: 564 đơn thiếu shipment, 16 cặp SKU trùng khóa, 382 ngày COGS > Revenue.

---

### 6. KIỂU VISUAL MONG MUỐN & TƯƠNG TÁC (VISUAL TYPES & UX)
* **Visuals chính:**
  - KPI Callout Cards (kèm subtitle so sánh tăng trưởng ▲/▼ YoY).
  - Combo Chart (Line & Clustered Column) cho tương quan Đơn - Doanh thu - Lưu lượng.
  - Matrix Heatmap (có conditional formatting nền theo dải gradient nhạt).
  - Treemap phân tầng (Category -> Segment).
  - Scatter Chart cho phân tích danh mục khuyến mãi.
  - Pareto Chart kết hợp đường tích lũy % 80/20.
* **Thanh lọc Header đồng bộ (Global Slicers Bar tại vị trí y: 0 -> 80 trên cả 8 trang):**
  - Slicer Năm (`dim_date[Year]`), Quý (`dim_date[Quarter]`), Danh mục (`md_product[product_category]`), Vùng (`md_geography[geography_region]`), Kênh (`md_channel[channel_id]`).
  - Slicer Trạng thái đơn (`md_order_status[revenue_eligible_flag]` - Mặc định chọn `TRUE`).
  - Bookmarks tương tác: Tạo nút chuyển đổi nhanh giữa chế độ xem `"Chỉ đơn hợp lệ (Delivered/Shipped/Returned)"` và `"Tất cả trạng thái đơn"`.
* **Mật độ hiển thị:** Giới hạn 6–8 visual chính mỗi trang để tối ưu tốc độ render và tránh quá tải thông tin.

---

### 7. MÀU SẮC & PHONG CÁCH THIẾT KẾ (THEME & STYLING)
* **Kích thước khung vẽ (Canvas):** Chuẩn Widescreen 16:9 (`1920 × 1080 px`).
* **Phong cách tổng thể:** Hiện đại, phẳng, tinh giản theo chuẩn thiết kế `Fluent2` của Microsoft (Clean Corporate Aesthetic).
* **Bảng màu chủ đạo (Color Palette):**
  - **Primary / Hero Color:** Xanh Navy đậm (`#1B365D` hoặc `#0F2C59`) — thể hiện tính tin cậy, dùng cho các biểu đồ doanh số chính và cột tổng hợp.
  - **Secondary Accent:** Xanh Slate / Muted Cyan (`#2E8BC0` hoặc `#4A90E2`) — dùng cho các chỉ số so sánh phụ, đường xu hướng hoặc năm liền trước.
  - **Success / Positive:** Xanh Sage / Muted Emerald (`#2E7D32` hoặc `#27AE60`) — thể hiện tăng trưởng dương YoY, tỷ lệ hoàn tất đơn, rating cao.
  - **Warning / Alert:** Đỏ San hô / Coral Amber (`#D32F2F` hoặc `#E74C3C`) — cảnh báo tỷ lệ hủy/trả, tồn kho stockout, tỷ lệ đánh giá thấp <= 2 sao.
  - **Background & Cards:** Nền canvas xám sáng dịu mắt (`#F4F6F9`), các visual đặt trong container thẻ Card màu trắng tinh (`#FFFFFF`), bo góc nhẹ (`Border Radius: 6-8px`), bóng đổ nhẹ nhàng (`Light Drop Shadow`).
* **Typography:** Font họ Segoe UI hoặc Aptos. Phân cấp rõ ràng: Title trang 18pt Bold, KPI Callout 26pt Bold, Header visual 12pt Semi-bold, Label số liệu 10pt Regular.