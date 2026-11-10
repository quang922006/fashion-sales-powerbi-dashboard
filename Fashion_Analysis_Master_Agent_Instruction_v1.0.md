# MASTER AGENT INSTRUCTION: FASHION_ANALYSIS POWER BI PROJECT (PBIP / TMDL / PBIR)
**Version:** 1.1 (Production-Ready Blueprint — đã đối soát với dữ liệu thực tế ngày 2026-11-05)
**Base Plan:** `Fashion_Analysis_Dashboard_Plan_v0.1_2026-10-25_DRAFT.md` (OP-1…OP-18 APPROVED) & `Data Dictionary v0.2`.

### Nhật ký thay đổi v1.0 → v1.1 (dựa trên profiling 13 file CSV trong `Data/`)
| # | Hạng mục | Thay đổi | Lý do (bằng chứng từ dữ liệu) |
|---|---|---|---|
| C1 | Đường dẫn | Dùng đúng tên thực tế: `Data/`, `powerBI_model/fahsion_analysis.SemanticModel`, `powerBI_model/fahsion_analysis.Report` | Tên trong workspace khác bản v1.0; giữ nguyên để không phá file `.pbip` |
| C2 | Số file nguồn | 13 file CSV có sẵn + 1 file mẫu dự báo **chưa có** | `stg_sales_submission_template` chưa được cung cấp → M70 Forecast ở trạng thái PENDING |
| C3 | `M05_All COGS` | Làm tròn COGS **theo ngày** đến 2 chữ số rồi cộng | Tính từ line-level = 14,163,450,519.13 (chính xác 14,163,450,519.12786…); `sales.csv` làm tròn từng ngày → 14,163,450,519.17. Cách làm tròn theo ngày khớp tuyệt đối |
| C4 | `M15 Order Return Rate %` | Giữ công thức returned / (delivered + returned), kỳ vọng **6.54%**; thêm `M15b Returned Order Share %` = **5.59%** | Con số "~5.6%" bản cũ là tỷ trọng đơn returned trên tổng đơn, không phải kết quả của công thức M15 |
| C5 | `M53 Median Cover Days` | Bắt buộc dùng `MEDIAN` toàn bộ (= **240** ngày), bỏ phương án lọc <= 365 | 23,251 / 60,247 dòng (38.6%) có `days_of_supply` > 365 → lọc <= 365 làm mất ~40% dữ liệu, không phải loại outlier |
| C6 | `M28 On-Time SLA %` | Bổ sung giá trị kỳ vọng **41.72%** | Prep TB 1.498 + Transit TB 4.499 ≈ 6 ngày > ngưỡng 5 ngày → cần báo trước cho stakeholder |
| C7 | QA Benchmark | Bổ sung M14, M15, M15b, M22, M26, M27, M28, M53, M61, số đơn/năm | Đều đã được kiểm chứng trên dữ liệu |
| C8 | `M08 YoY` | Ghi rõ dữ liệu đơn bắt đầu **2012-07-04** | Năm 2012 chỉ có ~6 tháng → YoY 2013 bị phóng đại, phải ẩn |
| C9 | Slicer trạng thái đơn & `M14/M15/M15b` | Slicer `revenue_eligible_flag` để mặc định **"All"** (không ép TRUE), không tạo Bookmark; `M14/M15/M15b` dùng `REMOVEFILTERS('md_order_status')` | Các chỉ số chuẩn (M04, M05, M10b, M12, M30) đã tự lọc đơn hợp lệ; nếu slicer mặc định TRUE thì M14 = 0 và các benchmark mọi trạng thái (M01, M03, M10, M61) sai lệch |
| C10 | `M24 Refund Amount` | Khi lọc theo khuyến mãi / cột của bảng line, hoàn tiền được lọc qua cặp `(sales_order_id, product_id)` bằng `TREATAS` | `txn_return` không nối với `md_promotion`; nếu không xử lý, M04/M07 theo từng mã KM sẽ trừ toàn bộ 510.6M hoàn tiền |
| C11 | Kiểu khóa | `order_id`, `customer_id`, `product_id` dùng **Int64**; `zip` vẫn là Text (Q8) | Khóa số nén tốt hơn, quan hệ khớp 100% |
| C12 | Trang 2 & 6 | Pareto theo nhóm 5% SKU (xếp hạng doanh thu toàn kỳ); Top/Bottom 10 SKU xếp theo lợi nhuận gộp toàn kỳ (cột tĩnh); `M61` chỉ đếm đơn từ 2013-01-01; chưa làm Drill-through | Tránh RANKX động trên 2,412 SKU mỗi ô (chậm); web traffic bắt đầu 2013 (Q20) |
| C13 | `M30/M31/M32/M33/M35` & `M26/M27/M28` | Đếm khách qua bảng line (`COUNTROWS(SUMMARIZE('txn_sales_order_line', 'txn_sales_order'[customer_id]))`); measure giao vận dùng `CROSSFILTER(line, order, Both)`. Toàn kỳ không đổi (87,280 / 75.23% / 7.17 / 41.72%) | Bản cũ đếm trên `txn_sales_order` nên slicer Danh mục không tác động; `M33` theo danh mục ra 0.26–4.3 đơn/khách (sai) và `M34 CLV` chia cho toàn bộ khách |
| C14 | `M31b Returning Customers` | `= M30 − M31` (khách hoạt động không phải khách mới trong kỳ) | Bản cũ đếm khách có đơn `Returning`, một khách mua lần đầu và mua lại cùng năm bị đếm 2 lần → cột chồng New + Returning > Active (2013: 43,828 vs 36,625) |
| C15 | Pareto (`product_pareto_bucket`) | Xếp hạng duy nhất (tie-break `product_id`) chỉ trên **1,578 SKU có bán hợp lệ**; 834 SKU chưa bán để trống | Bản cũ dùng `RANKX ... Skip` trên 2,412 SKU: 845 SKU đồng hạng dồn vào nhóm 70%, trục dừng ở 70% thay vì 100% |
| C16 | Trang 5 & 6 & 8 | Matrix Danh mục × Size dùng `M25b Unit Return Rate %` (SL trả / SL đã giao, ~3.99%); bảng rating thấp chỉ hiện SKU ≥ 30 review (`M43b`, `M40b`); combo Sessions vs Đơn dùng `M10d` (từ 2013); `M86–M88` chỉ theo slicer thời gian, làm tròn 2 số | `M15` cấp đơn gán nhầm đơn `returned` cho mọi size trong đơn; bảng cũ liệt kê cả 840 SKU < 30 review; biểu đồ "2013–2022" vẫn vẽ đơn 2012; thẻ đối soát lệch hàng tỷ khi chọn Danh mục/Vùng |

---

## PHẦN 0: VAI TRÒ & QUY TẮC THỰC THI CỦA AI AGENT (AGENT OPERATING RULES)

Bạn là **Lead Power BI Architect & TMDL/PBIR Engineer**. Nhiệm vụ của bạn là xây dựng hoàn chỉnh dự án Power BI trong thư mục `powerBI_model/` (gồm `fahsion_analysis.SemanticModel` định dạng TMDL và `fahsion_analysis.Report` định dạng PBIR — giữ nguyên chính tả `fahsion` của dự án hiện có) từ 13 file CSV trong thư mục `Data/` (cộng 1 file mẫu dự báo 548 ngày khi được bổ sung — xem C2).

### Quy tắc bất khả xâm phạm (Strict Guardrails):
1. **Không sửa đổi dữ liệu gốc:** Tuyệt đối không ghi đè hay chỉnh sửa các file CSV trong `Data/`. Mọi bước làm sạch (Q1–Q23) phải thực hiện qua Power Query (`qry_*`) đọc từ tham số `prm_data_directory`.
2. **Tuân thủ tuyệt đối Quyết định Nghiệp vụ (OP-1…OP-18):** Không tự ý thay đổi công thức `M04 (Net Revenue)`, không dùng `signup_date` cho Cohort, không đưa `reorder_flag` và `bounce_rate` lên biểu đồ chính, không cộng `shipping_fee` vào doanh thu.
3. **Kiểm tra chéo (Self-Verification):** Sau khi viết M-Code và DAX, mọi con số tổng hợp phải khớp 100% đến từng đơn vị với **Bảng QA Benchmark tại Phần 6**.
4. **Cấu trúc Star Schema 1 chiều:** Mọi quan hệ đều là `1 -> Many (Single Direction)`. Không bật Bi-directional cross-filtering trong Model.

---

## PHẦN 1: MỤC TIÊU KINH DOANH & BỐI CẢNH DỮ LIỆU (BUSINESS CONTEXT)

### 1.1. Câu hỏi Kinh doanh Bắc Đẩu (North Star Question)
> **"Vì sao số đơn hàng sụt giảm ~58% từ năm 2015 (82,622 đơn) xuống 2021 (34,525 đơn) — với năm 2019 là điểm gãy lớn (rơi từ 69,510 xuống 41,601 đơn) — trong khi phiên truy cập web lại tăng từ 6.8 triệu (2013) lên 11.1 triệu (2022) khiến tỷ lệ chuyển đổi rơi từ 1.13% xuống 0.33%? Đòn bẩy nào (giá thực thu, khuyến mãi, kênh, danh mục, vận hành) có thể đảo ngược xu hướng này?"**

### 1.2. Đối tượng sử dụng & Phân bổ 8 Trang Báo cáo
| Đối tượng | Quyết định cần đưa ra | Mối quan tâm chính | Trang chính |
|---|---|---|---|
| **Ban giám đốc** | Giữ, đầu tư hay cắt danh mục/kênh; đặt mục tiêu năm | Doanh thu thuần (`M04`), biên lợi nhuận (`M07`), xu hướng, YoY | **Trang 1: Tổng quan** |
| **Kinh doanh / Ecommerce** | Điều chỉnh giá, danh mục sản phẩm, khuyến mãi | Doanh thu theo danh mục/segment/size/màu, AOV, tỷ lệ áp KM, thực thu giá (`M22`) | **Trang 2: Bán hàng**, **Trang 3: Khuyến mãi** |
| **Marketing** | Phân bổ ngân sách theo kênh, chọn khuyến mãi | Tỷ lệ chuyển đổi (`M61`), đơn theo kênh, khách mới vs quay lại, hiệu quả KM | **Trang 3, 4, 6** |
| **Vận hành / Supply Chain** | Mức tồn, đặt hàng bổ sung, SLA giao hàng, giảm hủy/trả | Hết hàng, cover days, thời gian giao (`M26, M27`), lý do trả | **Trang 5: Vận hành**, **Trang 7: Tồn kho** |
| **CSKH / Chất lượng** | Xử lý nguyên nhân trả hàng và đánh giá thấp | Lý do trả, rating (`M40`), sản phẩm bị đánh giá thấp (`M41`) | **Trang 5: Vận hành & Trả hàng** |
| **Data / BI / Kế hoạch** | Kiểm soát chất lượng dữ liệu & theo dõi dự báo 548 ngày | Đối chiếu `sales.csv`, theo dõi lỗi ngữ nghĩa Q1–Q23, thực tế vs dự báo (`M70`) | **Trang 8: Dự báo & Chất lượng DL** |

---

## PHẦN 2: KIẾN TRÚC MÔ HÌNH DỮ LIỆU & ÁNH XẠ SCHEMA (DATA MODEL & SCHEMA MAPPING)

### 2.1. Tham số & Quy ước Đặt tên (Naming Conventions)
- **Parameter:** `prm_data_directory` (Text) trỏ tới thư mục `Data/` (ví dụ `E:\Projects\PowerBimain_automation\Data\`).
- **Queries trung gian (nếu có):** Tiền tố `qry_*`.
- **Bảng Dimension (9 bảng):**
  1. `dim_date` (Sinh bằng DAX từ `2012-01-01` đến `2024-12-31` để bao phủ cả 548 ngày dự báo `2023-01-01` → `2024-07-01`). Đánh dấu `Mark as Date Table`.
  2. `md_customer` (Nguồn: `customers.csv` | 121,930 dòng | PK: `customer_id`).
  3. `md_geography` (Nguồn: `geography.csv` | 39,948 dòng | PK: `geography_zip_id`).
  4. `md_product` (Nguồn: `products.csv` | 2,412 dòng | PK: `product_id`).
  5. `md_promotion` (Nguồn: `promotions.csv` | 50 dòng | PK: `promotion_id`).
  6. `md_order_status` (Bảng tạo mới | 6 dòng | PK: `order_status`).
  7. `md_channel` (Bảng tạo mới | 6 dòng | PK: `channel_id` — dùng chung cho `order_source`, `acquisition_channel`, `traffic_source`).
  8. `md_payment_method` (Bảng tạo mới | 5 dòng | PK: `payment_method_id`).
  9. `md_return_reason` (Bảng tạo mới | 5 dòng | PK: `return_reason_id`).
- **Bảng Fact & Staging (10 bảng):**
  1. `txn_sales_order` (Nguồn: `orders.csv` | 646,945 dòng | PK: `sales_order_id`).
  2. `txn_sales_order_line` (Nguồn: `order_items.csv` | 714,669 dòng | PK: `sales_order_line_id`).
  3. `txn_payment` (Nguồn: `payments.csv` | 646,945 dòng | FK: `sales_order_id`).
  4. `txn_shipment` (Nguồn: `shipments.csv` | 566,067 dòng | FK: `sales_order_id`).
  5. `txn_return` (Nguồn: `returns.csv` | 39,939 dòng | PK: `return_id`).
  6. `txn_review` (Nguồn: `reviews.csv` | 113,551 dòng | PK: `review_id`).
  7. `txn_inventory_snapshot` (Nguồn: `inventory.csv` | 60,247 dòng | Grain: `product_id` × `snapshot_date`).
  8. `txn_web_traffic_daily` (Nguồn: `web_traffic.csv` | 3,652 dòng | Grain: `web_traffic_date`).
  9. `stg_sales_daily` (Nguồn: `sales.csv` | 3,833 dòng | Ẩn khỏi Report View, dùng cho M70 & QA).
  10. `stg_sales_submission_template` (Nguồn: file mẫu dự báo 548 ngày `2023-01-01` → `2024-07-01` | Ẩn khỏi Report View, dùng cho M70). **⏳ PENDING:** file này hiện chưa có trong `Data/`. Chỉ tạo bảng và phần Forecast của M70 khi file được bổ sung; trước đó Trang 8 chỉ hiển thị phần Actual.

> **Phạm vi thời gian thực tế:** `orders` / `sales.csv`: `2012-07-04` → `2022-12-31` (3,833 ngày) · `web_traffic`: `2013-01-01` → `2022-12-31` · `inventory`: 126 snapshot cuối tháng `2012-07-31` → `2022-12-31`.

### 2.2. Bảng Ánh xạ Cột Chuẩn (Column Mapping Table)
*(Lưu ý: Nếu trong workspace có file `Fashion_Analysis_Data_Dictionary_v0.2_2026-10-25_APPROVED.md`, hãy ưu tiên đối chiếu 100% tên cột trong file đó. Dưới đây là cấu trúc ánh xạ cốt lõi)*:

| Bảng Đích | Cột Gốc (CSV) | Cột Chuẩn (Model) | Kiểu Dữ Liệu | Quy tắc biến đổi (Power Query / DAX) |
|---|---|---|---|---|
| **`md_geography`** | `zip`, `city`, `region`, `district` | `geography_zip_id`, `geography_city`, `geography_region`, `geography_district` | Text | Ép `zip` thành **Text** (giữ nguyên, không pad số 0 đầu để khớp 100% giữa các file - Q8). |
| **`md_customer`** | `customer_id`, `zip`, `signup_date`, `gender`, `age_group`, `acquisition_channel` | `customer_id`, `geography_zip_id`, `customer_signup_date`, `customer_gender`, `customer_age_group`, `acquisition_channel_id` | Text, Date | Bỏ cột `city` dư thừa (Q9). Thêm cột `customer_first_order_date` = `MIN(order_date)` theo khách (Q6). |
| **`md_product`** | `product_id`, `product_name`, `category`, `segment`, `size`, `color`, `price`, `cogs` | `product_id`, `product_name`, `product_category`, `product_segment`, `product_size`, `product_color`, `product_unit_price`, `product_unit_cost` | Text, Decimal | Giữ nguyên toàn bộ phần thập phân của `price` và `cogs` khi tính toán để khớp `sales.csv` (Q11), chỉ format hiển thị 2 chữ số. |
| **`md_promotion`** | `promo_id`, `promo_name`, `promo_type`, `discount_value`, `start_date`, `end_date`, `applicable_category`, `promo_channel`, `stackable_flag`, `min_order_value` | `promotion_id`, `promotion_name`, `promotion_type`, `promotion_discount_value`, `promotion_start_date`, `promotion_end_date`, `promotion_applicable_category`, `promotion_channel`, `promotion_stackable_flag`, `promotion_min_order_value` | Text, Decimal, Date, Boolean | Chuyển `stackable_flag` (1/0) sang Boolean `TRUE/FALSE` (Q23). Giữ `null` ở `applicable_category` (Q3). |
| **`txn_sales_order`** | `order_id`, `order_date`, `customer_id`, `zip`, `order_status`, `payment_method`, `device_type`, `order_source` | `sales_order_id`, `sales_order_date`, `customer_id`, `geography_zip_id`, `order_status`, `payment_method_id`, `device_type`, `sales_order_source_id` | Text, Date | Thêm `revenue_eligible_flag` (Boolean: `TRUE` nếu `order_status` thuộc `['delivered', 'shipped', 'returned']`), `shipment_missing_flag` (Boolean: `TRUE` cho 564 đơn đã giao/gửi/trả nhưng thiếu shipment - Q5), và `customer_type` (`New` nếu `sales_order_date = customer_first_order_date`, ngược lại `Returning` - Q6). |
| **`txn_sales_order_line`** | `order_id`, `product_id`, `quantity`, `unit_price`, `discount_amount`, `promo_id`, `promo_id_2` | `sales_order_line_id` (Mới), `sales_order_id`, `product_id`, `sales_order_quantity`, `unit_price`, `discount_line_amount`, `promotion_id`, `secondary_promotion_id` | Int, Text, Decimal | Thêm Index `sales_order_line_id` vì có 16 cặp `(order_id, product_id)` trùng khóa (Q1). Thêm `promotion_applied_flag` (`TRUE` nếu `promotion_id <> null` - Q3). |
| **`txn_shipment`** | `order_id`, `ship_date`, `delivery_date`, `shipping_fee` | `sales_order_id`, `ship_date`, `delivery_date`, `shipping_fee` | Text, Date, Decimal | Merge `sales_order_date` từ `orders` để tính `shipment_preparation_days = ship_date - sales_order_date` và `shipment_transit_days = delivery_date - ship_date`. |
| **`txn_return`** | `return_id`, `order_id`, `product_id`, `return_date`, `return_reason`, `return_quantity`, `refund_amount` | `return_id`, `sales_order_id`, `product_id`, `return_date`, `return_reason_id`, `return_quantity`, `return_refund_amount` | Text, Date, Int, Decimal | Giữ nguyên 39,939 dòng (kể cả 2 cặp trùng `order_id + product_id` vì khác `return_id` - Q2). |
| **`txn_review`** | `review_id`, `order_id`, `product_id`, `customer_id`, `review_date`, `rating`, `review_title` | `review_id`, `sales_order_id`, `product_id`, `customer_id`, `review_date`, `review_rating`, `review_title` | Text, Date, Int | Giữ nguyên 113,551 dòng. |
| **`txn_inventory_snapshot`**| `snapshot_date`, `product_id`, `stock_on_hand`, `units_received`, `units_sold`, `stockout_days`, `days_of_supply`, `fill_rate`, `stockout_flag`, `overstock_flag`, `reorder_flag`, `sell_through_rate` | `snapshot_date`, `product_id`, `inventory_stock_on_hand`, `inventory_units_received`, `inventory_units_sold`, `inventory_stockout_days`, `inventory_cover_days`, `inventory_fill_rate`, `inventory_stockout_flag`, `inventory_overstock_flag`, `inventory_reorder_flag`, `inventory_sell_through_rate` | Date, Text, Int, Decimal, Boolean | Xóa các cột dư thừa (`product_name`, `category`, `segment`, `year`, `month`). Ép cờ 1/0 sang Boolean. |

### 2.3. Định nghĩa Bảng Ánh xạ Trạng thái Đơn (`md_order_status` - OP-11 & Q21)
Khởi tạo bảng `md_order_status` gồm 6 dòng cố định sau:
| `order_status` | `status_group` (Nhóm trạng thái) | `revenue_eligible_flag` (Tính vào Doanh thu chuẩn M04) | `sort_order` |
|---|---|---|---|
| `delivered` | Hoàn tất / Đã xuất (Fulfilled) | `TRUE` | 1 |
| `shipped` | Hoàn tất / Đã xuất (Fulfilled) | `TRUE` | 2 |
| `returned` | Hoàn tất / Đã xuất (Fulfilled) | `TRUE` | 3 |
| `paid` | Đang xử lý (In Progress) | `FALSE` | 4 |
| `created` | Đang xử lý (In Progress) | `FALSE` | 5 |
| `cancelled` | Đã hủy (Cancelled) | `FALSE` | 6 |

### 2.4. Cấu hình Sơ đồ Quan hệ (Star Schema Relationships Matrix)
Tất cả các quan hệ đều là **Many-to-One (`*:1`)** và **Single Direction (`OneWay`)**:

| From Table (Many) | From Column (FK) | To Table (One) | To Column (PK) | Trạng thái | Ghi chú Kỹ thuật |
|---|---|---|---|---|---|
| `txn_sales_order` | `sales_order_date` | `dim_date` | `date` | **Active** | Ngày đặt hàng chính |
| `txn_sales_order` | `customer_id` | `md_customer` | `customer_id` | **Active** | |
| `txn_sales_order` | `geography_zip_id` | `md_geography` | `geography_zip_id`| **Active** | Khớp 100% với zip của khách (Q9) |
| `txn_sales_order` | `order_status` | `md_order_status` | `order_status` | **Active** | Lọc đơn hợp lệ (`revenue_eligible_flag = TRUE`) |
| `txn_sales_order` | `sales_order_source_id`| `md_channel` | `channel_id` | **Active** | Kênh phát sinh đơn |
| `txn_sales_order` | `payment_method_id` | `md_payment_method`| `payment_method_id`| **Active** | |
| `txn_sales_order_line`| `sales_order_id` | `txn_sales_order` | `sales_order_id` | **Active** | Header → Detail (1 chiều từ Order xuống Line) |
| `txn_sales_order_line`| `product_id` | `md_product` | `product_id` | **Active** | |
| `txn_sales_order_line`| `promotion_id` | `md_promotion` | `promotion_id` | **Active** | Khuyến mãi chính |
| `txn_sales_order_line`| `secondary_promotion_id`| `md_promotion` | `promotion_id` | *Inactive* | Khuyến mãi xếp chồng thứ 2 (Q4 / OP-16) |
| `txn_payment` | `sales_order_id` | `txn_sales_order` | `sales_order_id` | **Active** | |
| `txn_shipment` | `sales_order_id` | `txn_sales_order` | `sales_order_id` | **Active** | |
| `txn_shipment` | `ship_date` | `dim_date` | `date` | *Inactive* | Kích hoạt bằng `USERELATIONSHIP` khi cần |
| `txn_shipment` | `delivery_date` | `dim_date` | `date` | *Inactive* | Kích hoạt bằng `USERELATIONSHIP` khi cần |
| `txn_return` | `sales_order_id` | `txn_sales_order` | `sales_order_id` | **Active** | Lọc hoàn tiền theo Cohort ngày đặt đơn |
| `txn_return` | `product_id` | `md_product` | `product_id` | **Active** | Lọc hoàn tiền theo Sản phẩm/Danh mục |
| `txn_return` | `return_reason_id` | `md_return_reason` | `return_reason_id`| **Active** | |
| `txn_return` | `return_date` | `dim_date` | `date` | *Inactive* | Kích hoạt bằng `USERELATIONSHIP` khi cần |
| `txn_review` | `sales_order_id` | `txn_sales_order` | `sales_order_id` | **Active** | |
| `txn_review` | `product_id` | `md_product` | `product_id` | **Active** | Lọc đánh giá theo Sản phẩm/Danh mục |
| `txn_review` | `review_date` | `dim_date` | `date` | *Inactive* | |
| `txn_inventory_snapshot`| `product_id` | `md_product` | `product_id` | **Active** | |
| `txn_inventory_snapshot`| `snapshot_date` | `dim_date` | `date` | **Active** | |
| `txn_web_traffic_daily`| `web_traffic_date` | `dim_date` | `date` | **Active** | Không nối với `md_channel` (Q19) |
| `stg_sales_daily` | `sales_date` | `dim_date` | `date` | **Active** | Dùng đối chiếu & dự báo |
| `stg_sales_submission_template`| `sales_date` | `dim_date` | `date` | **Active** | 548 ngày dự báo |

> **⚠️ LƯU Ý KỸ THUẬT QUAN TRỌNG VỀ LỌC 1 CHIỀU TRONG TABULAR MODEL:**
> Để đảm bảo khi người dùng lọc `md_product` (như Category, Segment, Size, Color) hoặc `md_promotion` thì các chỉ số cấp đơn hàng (`M10 Số đơn`, `M12 AOV`, `M14 Tỷ lệ hủy`, `M15 Tỷ lệ trả`) đều phản ánh đúng danh mục sản phẩm được chọn:
> - Với các Measure đếm đơn hàng (`M10`, `M14`, `M15`): Hãy đếm `DISTINCTCOUNT('txn_sales_order_line'[sales_order_id])` hoặc dùng `CALCULATE(..., CROSSFILTER('txn_sales_order_line'[sales_order_id], 'txn_sales_order'[sales_order_id], Both))` bên trong Measure đó!

---

## PHẦN 3: BỘ CÔNG THỨC DAX CHUẨN HÓA (DAX MEASURES SPECIFICATION: M01 – M70)

Tất cả các Measure được lưu trong bảng `_measures` và phân nhóm theo `displayFolder`. Dưới đây là định nghĩa logic bắt buộc:

### Nhóm 1: Doanh thu và Lợi nhuận (`01_Revenue_Profit`)
- **`M01 Gross Sales`**:
  ```dax
  SUMX('txn_sales_order_line', 'txn_sales_order_line'[sales_order_quantity] * 'txn_sales_order_line'[unit_price])
  ```
- **`M02 Discount Amount`**:
  ```dax
  SUM('txn_sales_order_line'[discount_line_amount])
  ```
- **`M03 Net Sales`**:
  ```dax
  [M01 Gross Sales] - [M02 Discount Amount]
  ```
- **`M24 Refund Amount`**:
  ```dax
  SUM('txn_return'[return_refund_amount])
  ```
- **`M04 Net Revenue` (Chỉ số chuẩn chính thức - OP-11)**:
  ```dax
  VAR _EligibleNetSales = 
      CALCULATE(
          [M03 Net Sales],
          'md_order_status'[revenue_eligible_flag] = TRUE()
      )
  VAR _EligibleRefunds = 
      CALCULATE(
          [M24 Refund Amount],
          'md_order_status'[revenue_eligible_flag] = TRUE()
      )
  RETURN
      _EligibleNetSales - _EligibleRefunds
  ```
- **`M05 COGS` (Tính theo cùng phạm vi đơn hợp lệ với M04 - Mục 2.1)**:
  ```dax
  CALCULATE(
      SUMX(
          'txn_sales_order_line',
          'txn_sales_order_line'[sales_order_quantity] * RELATED('md_product'[product_unit_cost])
      ),
      'md_order_status'[revenue_eligible_flag] = TRUE()
  )
  ```
- **`M05_All COGS (All Statuses)`** (không lọc trạng thái, dùng cho Trang 8 đối chiếu với `Σ sales.csv[COGS]` = 14,163,450,519.17). **Bắt buộc làm tròn theo ngày** để khớp cách `sales.csv` lưu số liệu (C3):
  ```dax
  SUMX(
      VALUES('dim_date'[date]),
      ROUND(
          CALCULATE(
              SUMX('txn_sales_order_line', 'txn_sales_order_line'[sales_order_quantity] * RELATED('md_product'[product_unit_cost])),
              ALL('md_order_status')
          ),
          2
      )
  )
  -- Không làm tròn: 14,163,450,519.13 (lệch 0.04). Làm tròn theo ngày: 14,163,450,519.17 (khớp)
  ```
- **`M06 Gross Profit`**: `[M04 Net Revenue] - [M05 COGS]`
- **`M07 Gross Margin %`**: `DIVIDE([M06 Gross Profit], [M04 Net Revenue])`
- **`M08 Net Revenue YoY %`** & **`M08b Net Revenue MoM %`**: Dùng `SAMEPERIODLASTYEAR('dim_date'[date])` và `DATEADD('dim_date'[date], -1, MONTH)`. Kiểm tra biên năm 2012/2013 để ẩn YoY khi kỳ trước không đủ ngày (dữ liệu đơn bắt đầu `2012-07-04`, nên mọi kỳ trước `2012-07-04` coi là không có dữ liệu — C8).

### Nhóm 2: Đơn hàng (`02_Orders`)
- **`M10 Total Orders (All Statuses)`**:
  ```dax
  DISTINCTCOUNT('txn_sales_order_line'[sales_order_id]) 
  -- Dùng bảng line để tự động nhận bộ lọc từ md_product và md_promotion (100% đơn đều có mặt trong line)
  ```
- **`M10b Eligible Orders` (Số đơn hợp lệ cho M04/M12)**:
  ```dax
  CALCULATE([M10 Total Orders (All Statuses)], 'md_order_status'[revenue_eligible_flag] = TRUE())
  ```
- **`M11 Total Units Sold`**: `SUM('txn_sales_order_line'[sales_order_quantity])` (và bản `M11b Eligible Units Sold` lọc `revenue_eligible_flag = TRUE()`).
- **`M12 AOV` (Giá trị đơn trung bình chuẩn)**: `DIVIDE([M04 Net Revenue], [M10b Eligible Orders])` (và `M12b Gross AOV = DIVIDE([M01 Gross Sales], [M10 Total Orders (All Statuses)])` để đối chiếu bảng xu hướng mục 0).
- **`M13 UPT` (Sản phẩm mỗi đơn)**: `DIVIDE([M11 Total Units Sold], [M10 Total Orders (All Statuses)])`
- **`M14 Cancellation Rate %`**:
  ```dax
  VAR _Cancelled = CALCULATE([M10 Total Orders (All Statuses)], 'md_order_status'[order_status] = "cancelled")
  VAR _Total = CALCULATE([M10 Total Orders (All Statuses)], ALL('md_order_status'))
  RETURN DIVIDE(_Cancelled, _Total)
  ```
- **`M15 Order Return Rate %` (Đơn `returned` / Đơn đã giao thực tế gồm `delivered` + `returned`)**:
  ```dax
  VAR _Returned = CALCULATE([M10 Total Orders (All Statuses)], 'md_order_status'[order_status] = "returned")
  VAR _DeliveredAndReturned = CALCULATE([M10 Total Orders (All Statuses)], 'md_order_status'[order_status] IN {"delivered", "returned"})
  RETURN DIVIDE(_Returned, _DeliveredAndReturned)
  -- Toàn kỳ = 36,142 / (516,716 + 36,142) = 6.54%
  ```
- **`M15b Returned Order Share %`** (tỷ trọng đơn `returned` trên tổng mọi đơn — dùng cho phân bổ trạng thái Trang 5; C4):
  ```dax
  VAR _Returned = CALCULATE([M10 Total Orders (All Statuses)], 'md_order_status'[order_status] = "returned")
  VAR _Total = CALCULATE([M10 Total Orders (All Statuses)], ALL('md_order_status'))
  RETURN DIVIDE(_Returned, _Total)
  -- Toàn kỳ = 36,142 / 646,945 = 5.59%
  ```

### Nhóm 3: Khuyến mãi & Định giá (`03_Promotions_Pricing`)
- **`M20 Discount Rate %`**: `DIVIDE([M02 Discount Amount], [M01 Gross Sales])`
- **`M21 Promo Line Share %`**:
  ```dax
  VAR _PromoLines = CALCULATE(COUNTROWS('txn_sales_order_line'), 'txn_sales_order_line'[promotion_applied_flag] = TRUE())
  RETURN DIVIDE(_PromoLines, COUNTROWS('txn_sales_order_line'))
  ```
- **`M22 Price Realization %` (Thực thu giá: Giá bán thực / Giá niêm yết - TB ~93%)**:
  ```dax
  VAR _ActualGross = [M01 Gross Sales]
  VAR _ListGross = SUMX('txn_sales_order_line', 'txn_sales_order_line'[sales_order_quantity] * RELATED('md_product'[product_unit_price]))
  RETURN DIVIDE(_ActualGross, _ListGross)
  ```
- **`M23 Promo Net Sales`** vs **`M23b Non-Promo Net Sales`**: Lọc `[M03 Net Sales]` theo `'txn_sales_order_line'[promotion_applied_flag] = TRUE()` và `FALSE()`.

### Nhóm 4: Trả hàng & Giao vận (`04_Fulfillment_Returns`)
- **`M25 Refund Rate %`**:
  ```dax
  VAR _DeliveredNetSales = CALCULATE([M03 Net Sales], 'md_order_status'[order_status] IN {"delivered", "returned"})
  RETURN DIVIDE([M24 Refund Amount], _DeliveredNetSales)
  ```
- **`M26 Avg Preparation Time (Days)`** (Loại 564 đơn thiếu shipment - Q5):
  ```dax
  AVERAGE('txn_shipment'[shipment_preparation_days])
  ```
- **`M27 Avg Transit Time (Days)`**: `AVERAGE('txn_shipment'[shipment_transit_days])`
- **`M28 On-Time SLA %` (OP-13: Giao <= 5 ngày kể từ ngày đặt)**:
  ```dax
  VAR _ValidShipments = COUNTROWS('txn_shipment')
  VAR _OnTime = CALCULATE(COUNTROWS('txn_shipment'), 'txn_shipment'[shipment_preparation_days] + 'txn_shipment'[shipment_transit_days] <= 5)
  RETURN DIVIDE(_OnTime, _ValidShipments)
  -- Toàn kỳ = 41.72% (prep TB 1.498 + transit TB 4.499 ≈ 6 ngày > ngưỡng 5) — C6
  ```

### Nhóm 5: Khách hàng (`05_Customers`)
- **`M30 Active Customers`**:
  ```dax
  CALCULATE(DISTINCTCOUNT('txn_sales_order'[customer_id]), 'md_order_status'[revenue_eligible_flag] = TRUE())
  ```
- **`M31 New Customers`** & **`M31b Returning Customers`** (Dựa trên `customer_first_order_date`, tuyệt đối KHÔNG dùng `signup_date` - Q6):
  ```dax
  CALCULATE(
      DISTINCTCOUNT('txn_sales_order'[customer_id]),
      'txn_sales_order'[customer_type] = "New",
      'md_order_status'[revenue_eligible_flag] = TRUE()
  )
  ```
- **`M32 Repeat Purchase Rate %`** (Tỷ lệ khách có >= 2 đơn / tổng khách có đơn = 67,888 / 90,246 = 75.2%):
  ```dax
  VAR _CustomersWithOrders = SUMMARIZE('txn_sales_order', 'txn_sales_order'[customer_id], "@OrderCount", DISTINCTCOUNT('txn_sales_order'[sales_order_id]))
  VAR _RepeatCustomers = COUNTROWS(FILTER(_CustomersWithOrders, [@OrderCount] >= 2))
  RETURN DIVIDE(_RepeatCustomers, COUNTROWS(_CustomersWithOrders))
  ```
- **`M33 Avg Orders per Customer`**: `DIVIDE([M10 Total Orders (All Statuses)], DISTINCTCOUNT('txn_sales_order'[customer_id]))` (Toàn kỳ = 7.17).
- **`M34 Simple CLV`**: `DIVIDE([M06 Gross Profit], [M30 Active Customers])`

### Nhóm 6: Đánh giá (`06_Reviews`)
- **`M40 Average Rating`**: `AVERAGE('txn_review'[review_rating])` (Toàn kỳ = 3.94).
- **`M41 Low Rating Share %` (Rating <= 2)**:
  ```dax
  DIVIDE(CALCULATE(COUNTROWS('txn_review'), 'txn_review'[review_rating] <= 2), COUNTROWS('txn_review'))
  -- Toàn kỳ = 13.1%
  ```
- **`M42 Review Coverage Rate %`**: `DIVIDE(DISTINCTCOUNT('txn_review'[sales_order_id]), [M10b Eligible Orders])`

### Nhóm 7: Tồn kho (`07_Inventory` — Chỉ báo xu hướng, độ phủ 67% SKU - Q15, Q16, Q17)
- **`M50 Stockout Rate %`** *(Tạm ẩn khỏi visual chính theo OP-14)*: `DIVIDE(CALCULATE(COUNTROWS('txn_inventory_snapshot'), 'txn_inventory_snapshot'[inventory_stockout_flag] = TRUE()), COUNTROWS('txn_inventory_snapshot'))`
- **`M51 Total Stockout Days`**: `SUM('txn_inventory_snapshot'[inventory_stockout_days])`
- **`M52 Avg Sell-Through Rate`**: `AVERAGE('txn_inventory_snapshot'[inventory_sell_through_rate])`
- **`M53 Median Cover Days` (Chống méo bởi outlier 68,100 ngày - Q17)**: `MEDIAN('txn_inventory_snapshot'[inventory_cover_days])` — Toàn kỳ = **240** ngày. **Không** dùng phương án `AVERAGE` lọc `<= 365`, vì 23,251 / 60,247 dòng (38.6%) vượt 365 ngày; lọc như vậy làm mất ~40% dữ liệu chứ không phải loại outlier (C5).
- **`M54 Avg Fill Rate`**: `AVERAGE('txn_inventory_snapshot'[inventory_fill_rate])`
- **`M55 Ending Stock On Hand` (Semi-Additive)**:
  ```dax
  CALCULATE(SUM('txn_inventory_snapshot'[inventory_stock_on_hand]), LASTDATE('dim_date'[date]))
  ```

### Nhóm 8: Web Traffic & Dự báo (`08_Web_Forecast`)
- **`M60 Total Sessions`**, **`M60b Unique Visitors`**, **`M60c Page Views`**: `SUM` từ `txn_web_traffic_daily` *(Không đưa `bounce_rate` lên báo cáo - OP-15)*.
- **`M61 Web Conversion Rate %` (Chỉ tính từ 2013-01-01 - Q20)**:
  ```dax
  VAR _Sessions = [M60 Total Sessions]
  VAR _Orders = [M10 Total Orders (All Statuses)]
  RETURN IF(ISBLANK(_Sessions) || _Sessions = 0, BLANK(), DIVIDE(_Orders, _Sessions))
  ```
- **`M62 Orders by Channel`**: `[M10 Total Orders (All Statuses)]` đặt vào biểu đồ cùng `md_channel[channel_id]`.
- **`M70 Actual vs Forecast Revenue`** & **`M70b Actual vs Forecast COGS`**: Kết hợp `SUM('stg_sales_daily'[Revenue])` (2012–2022) và `SUM('stg_sales_submission_template'[Revenue])` (548 ngày: `2023-01-01` → `2024-07-01`). **⏳ PENDING phần Forecast** cho tới khi có file mẫu dự báo (C2); trong lúc chờ, M70 chỉ trả về phần Actual từ `stg_sales_daily`.

---

## PHẦN 4: THIẾT KẾ CHI TIẾT 8 TRANG BÁO CÁO (REPORT LAYOUT & VISUALS BLUEPRINT)

- **Kích thước Canvas:** `1920 × 1080 px`, Theme `Fluent2`.
- **Global Header & Slicers (Đồng bộ toàn bộ 8 trang, tọa độ `y: 0 -> 80`):**
  * Tiêu đề trang + Navigation Menu bên trái (`x: 0 -> 200`).
  * Bộ Slicer cố định: `Năm (dim_date[Year])`, `Quý (dim_date[Quarter])`, `Danh mục (md_product[product_category])`, `Vùng (md_geography[geography_region])`, `Kênh (md_channel[channel_id])`, `Nhóm trạng thái đơn (md_order_status[revenue_eligible_flag] - Mặc định = TRUE, kèm Bookmark chuyển nhanh sang "Tất cả trạng thái")`.
- **Quy tắc giới hạn:** Tối đa 6–8 Visuals chính mỗi trang (không tính Slicer & KPI Card đơn).

### Chi tiết từng trang:
1. **Trang 1 — Tổng quan (`Page 1: Executive Overview`)**
   - *Câu hỏi:* Kinh doanh đang đi thế nào và điểm gãy 2019 tác động ra sao?
   - *Hàng KPI Cards (`y: 90 -> 210`):* 7 thẻ gồm `M04 Net Revenue` (▲YoY), `M07 Gross Margin %` (▲YoY), `M10b Eligible Orders` (▲YoY), `M12 AOV` (▲YoY), `M14 Cancellation Rate %` (9.19%), `M15 Order Return Rate %` (6.54%), `M40 Average Rating`.
   - *Visual 1 (Line & Clustered Column - Trái trên):* `M04 Net Revenue` (Cột) + `M08 Net Revenue YoY %` (Đường) theo Tháng/Năm.
   - *Visual 2 (Clustered Bar - Phải trên):* `M04 Net Revenue` và `M07 Gross Margin %` theo `product_category` (`Streetwear`, `Casual`, `Outdoor`, `GenZ`).
   - *Visual 3 (Line & Clustered Column - Trái dưới):* Sự phân kỳ giữa `M10 Total Orders` (Cột - giảm 58% từ 2015–2021) và `M12 AOV` (Đường - tăng đều) theo Năm.
   - *Visual 4 (Clustered Bar - Phải dưới):* `M04 Net Revenue` theo `geography_region` (`East`, `Central`, `West`) và Top 10 `geography_city` (Không dùng Map vì không có tọa độ và zip bị mất số 0 đầu - Mục 4.2).

2. **Trang 2 — Bán hàng và Sản phẩm (`Page 2: Sales & Merchandising`)**
   - *Câu hỏi:* Sản phẩm/phân khúc nào bán tốt hoặc đang kéo tụt doanh số?
   - *Visuals:* (1) Treemap `product_category` × `product_segment` theo `M04 Net Revenue`; (2) Biểu đồ Pareto (Đường cộng dồn % doanh thu theo SKU); (3) Matrix Heatmap `product_size` (S, M, L, XL) × `product_color` (10 màu) hiển thị `M11 Units Sold` & `M07 Gross Margin %`; (4) Clustered Bar `M22 Price Realization %` (Thực thu giá TB 93%) theo Category & Segment; (5) Bảng Top 10 / Bottom 10 SKU (kèm Drill-through sang chi tiết dòng đơn).

3. **Trang 3 — Khuyến mãi (`Page 3: Promotions & Pricing`)**
   - *Câu hỏi:* Khuyến mãi (38.7% dòng đơn) có thực sự mang lại hiệu quả biên lợi nhuận?
   - *Visuals:* (1) KPI Cards: `M20 Discount Rate %` (4.6%), `M21 Promo Line Share %` (38.7%), `M22 Price Realization %` (93.0%); (2) Scatter / Bar Chart so sánh 50 mã `promotion_name` theo `M03 Net Sales` và `M07 Gross Margin %`; (3) Line Chart `M20 Discount Rate %` và `M21 Promo Line Share %` theo tháng/năm; (4) Clustered Column so sánh đơn Không KM vs. 1 KM vs. Xếp chồng 2 KM (`secondary_promotion_id` - 206 dòng); (5) Bảng chi tiết hiệu suất theo `promotion_type` (`percentage` vs `fixed`) và `promotion_channel`.

4. **Trang 4 — Khách hàng (`Page 4: Customer Analytics`)**
   - *Câu hỏi:* Ai đang mua hàng và khách hàng có quay lại sau năm 2019 không?
   - *Visuals:* (1) KPI Cards: `M30 Active Customers`, `M32 Repeat Purchase Rate %` (75.2%), `M33 Avg Orders/Customer` (7.17), `M34 Simple CLV`; (2) Stacked Column `M31 New Customers` vs `M31b Returning Customers` theo Năm/Tháng (tính từ `customer_first_order_date`); (3) Clustered Bar theo `customer_age_group` × `customer_gender`; (4) Hiệu quả kênh thu hút ban đầu `acquisition_channel_id` theo CLV & Số khách; (5) Matrix Cohort Retention (Tỷ lệ mua lại theo năm gia nhập đầu tiên); (6) Histogram phân bố số đơn/khách.

5. **Trang 5 — Vận hành, Giao hàng & Trả hàng (`Page 5: Operations, Returns & CX`)**
   - *Câu hỏi:* Đơn hàng đi đâu và vì sao bị trả hàng / đánh giá thấp?
   - *Visuals:* (1) KPI Cards: `M26 Avg Prep Time` (1.5 ngày), `M27 Avg Transit Time` (4.5 ngày), `M28 On-Time SLA <=5d %` (41.72%), `M24 Refund Amount`, `M41 Low Rating Share %` (13.1%); (2) Phễu/Bar trạng thái đơn hàng theo tỷ trọng trên tổng đơn (`delivered` 79.9%, `cancelled` 9.2%, `returned` 5.6% = `M15b`, `shipped` 2.1%, `paid` 2.1%, `created` 1.1%); (3) Clustered Bar `M24 Refund Amount` & Số lượt trả theo 5 lý do `return_reason_id` (`late_delivery`, `wrong_size`, `defective`, `changed_mind`, `not_as_described`); (4) Matrix Tỷ lệ trả hàng theo `product_category` × `product_size` (làm rõ lỗi `wrong_size`); (5) Tương quan `M40 Average Rating` theo Category & Top sản phẩm có `M41 Low Rating Share %` cao nhất.

6. **Trang 6 — Marketing và Web Traffic (`Page 6: Marketing & Web Funnel`)**
   - *Câu hỏi:* Tại sao lượng truy cập tăng gấp đôi nhưng đơn hàng lại giảm một nửa từ 2019?
   - *Visuals:* (1) KPI Cards: `M60 Sessions`, `M60b Unique Visitors`, `M60c Page Views`, `M61 Web Conversion Rate %` (rơi từ 1.13% xuống 0.33%); (2) Combo Chart `M60 Sessions` (Cột) vs `M10 Total Orders` (Đường) theo Tháng/Năm (2013–2022); (3) Line Chart `M61 Web Conversion Rate %` theo năm; (4) Stacked Bar `M62 Orders by Channel` (`sales_order_source_id`) qua các năm để xem kênh nào suy giảm mạnh nhất năm 2019; (5) Donut/Bar phân bổ đơn theo `device_type` (`desktop`, `mobile`, `tablet`). *(Tuyệt đối không vẽ Sessions theo `traffic_source` - Q19, và ẩn `bounce_rate` - OP-15)*.

7. **Trang 7 — Tồn kho (`Page 7: Inventory Indicators`)**
   - *Câu hỏi:* Tình trạng cung ứng, ngày hết hàng và tốc độ bán hết (Sell-through) biến động thế nào?
   - *Banner cảnh báo bắt buộc ở đầu trang:* `"Lưu ý: Dữ liệu tồn kho là chỉ báo xu hướng cuối tháng, hiện phủ 1,624 / 2,412 SKU (67.3% danh mục). Không đối chiếu trực tiếp tổng lượng bán tồn kho với doanh số đơn hàng (Q15)."`
   - *Visuals:* (1) KPI Cards: `M55 Ending Stock On Hand`, `M54 Avg Fill Rate`, `M52 Avg Sell-Through Rate`, `M53 Median Cover Days`, `M51 Total Stockout Days`; (2) Line Chart xu hướng `M54 Avg Fill Rate` & `M52 Avg Sell-Through Rate` theo tháng; (3) Clustered Bar `M53 Median Cover Days` (dùng Median toàn bộ, không lọc <= 365 — C5) theo `product_category` & `product_segment`; (4) Bảng Top SKU có `M51 Total Stockout Days` cao nhất. *(Không đưa `reorder_flag`, `stockout_flag`, `overstock_flag` lên visual chính - Q16 & OP-14)*.

8. **Trang 8 — Dự báo và Chất lượng Dữ liệu (`Page 8: Forecast & Data Governance QA`)**
   - *Câu hỏi:* Số liệu trên Dashboard có khớp 100% với nguồn đối soát không, và dự báo 548 ngày tới như thế nào?
   - *Visuals:* (1) Line Chart `M70 Actual vs Forecast Revenue` & `COGS` (phần Forecast ⏳ PENDING — C2) nối liền mạch từ lịch sử (`stg_sales_daily`: 2012–2022) sang tương lai (`stg_sales_submission_template`: 548 ngày từ `2023-01-01` → `2024-07-01`); (2) Bảng Đối soát Tự động (Data Reconciliation Table) so sánh trực tiếp Measure từ `txn_sales_order_line` với `stg_sales_daily` và `txn_payment` (Hiển thị cột `Chênh lệch = 0.00`); (3) Thẻ theo dõi các ngoại lệ dữ liệu: Số đơn thiếu vận đơn (`564 đơn` - Q5), Số dòng đơn trùng cặp SKU (`16 cặp` - Q1), Số ngày `COGS > Revenue` (`382 ngày`); (4) Bảng tóm tắt Quy tắc Nghiệp vụ & Từ điển Chỉ số (M01–M70).

---

## PHẦN 5: QUY TRÌNH TRIỂN KHAI KỸ THUẬT TRONG `powerBI_model/` (IMPLEMENTATION WORKFLOW)

Khi được yêu cầu xây dựng mô hình hoặc viết code, AI Agent phải thực hiện theo đúng thứ tự 4 bước sau:
1. **Bước 1 (TMDL Semantic Model - Expressions & Tables):**
   - Tạo `expressions.tmdl` chứa `prm_data_directory` và các query chuẩn hóa.
   - Tạo các file `tables/*.tmdl` cho 9 bảng Dimension, 10 bảng Fact/Staging và bảng `_measures.tmdl`.
   - Đảm bảo tắt `Auto Date/Time` trong `model.tmdl` (hiện đang bật: `annotation __PBI_TimeIntelligenceEnabled = 1` → đổi thành `0`).
2. **Bước 2 (TMDL Relationships):**
   - Tạo `relationships.tmdl` khai báo chính xác 25 quan hệ ở Mục 2.4 (`crossFilteringBehavior: oneDirection`, đánh dấu `isActive: false` cho các quan hệ phụ).
3. **Bước 3 (DAX Measures & Formatting):**
   - Viết toàn bộ Measure M01–M70 vào `_measures.tmdl` với `formatString` chuẩn (`#,0` hoặc `$#,0.00` cho tiền tệ, `0.0%` cho tỷ lệ, `#,0` cho số lượng) và `displayFolder` rõ ràng.
4. **Bước 4 (PBIR Report Pages):**
   - Tạo cấu trúc 8 trang trong `fahsion_analysis.Report/definition/pages/` (thay trang trống mặc định 1280×720 hiện có bằng các trang 1920×1080), áp dụng theme `Fluent2`, đồng bộ Slicer và Bookmark `"Chỉ đơn hợp lệ"` vs `"Tất cả trạng thái"`.

---

## PHẦN 6: BẢNG ĐỐI SOÁT SỐ LIỆU BẮT BUỘC (MANDATORY QA BENCHMARKS)

Mọi công thức Power Query và DAX sau khi dựng xong **BẮT BUỘC** phải trả về kết quả khớp tuyệt đối với bảng kiểm thử độc lập dưới đây (khi để Slicer thời gian = Toàn kỳ `2012–2022`):

| Mã / Phép thử | Phạm vi lọc | Kết quả Kỳ vọng Chính xác | Nguồn Đối chiếu |
|---|---|---|---|
| **Số dòng bảng Fact chính** | Toàn bộ | Orders: `646,945` \| Lines: `714,669` \| Payments: `646,945` \| Shipments: `566,067` \| Returns: `39,939` \| Reviews: `113,551` \| Inventory: `60,247` \| Web: `3,652` | Đếm dòng CSV gốc |
| **`M01 Gross Sales`** | Mọi trạng thái | **`16,430,476,585.53`** | Khớp 100% `Σ sales.csv[Revenue]` (3,833 ngày) |
| **`M02 Discount Amount`** | Mọi trạng thái | **`749,607,320.10`** | `4.56%` của Gross Sales |
| **`M03 Net Sales`** | Mọi trạng thái | **`15,680,869,265.43`** | Khớp 100% `Σ payments.csv[payment_value]` |
| **`M05_All COGS`** | Mọi trạng thái, làm tròn theo ngày (C3) | **`14,163,450,519.17`** | Khớp 100% `Σ sales.csv[COGS]` (không làm tròn = 14,163,450,519.13) |
| **`M03 Net Sales` theo từng `order_status`** | Từng trạng thái | `delivered`: **12,518,175,957.20** \| `cancelled`: **1,447,013,637.43** \| `returned`: **865,938,715.59** \| `shipped`: **337,278,840.20** \| `paid`: **330,627,096.24** \| `created`: **181,835,018.77** | Kiểm tra phân rã trạng thái |
| **`M03_Eligible Net Sales`** | `revenue_eligible_flag = TRUE` (`delivered`, `shipped`, `returned`) | **`13,721,393,512.99`** | Tổng Net Sales của 3 trạng thái hợp lệ |
| **`M24 Refund Amount`** | Toàn bộ / Hợp lệ | **`510,598,506.55`** | 100% phiếu trả đều thuộc đơn `returned` |
| **`M04 Net Revenue` (CHUẨN)** | `revenue_eligible_flag = TRUE` | **`13,210,795,006.44`** | `= 13,721,393,512.99 - 510,598,506.55` |
| **`M05 COGS` (CHUẨN)** | `revenue_eligible_flag = TRUE` | **`12,392,176,655.41`** | COGS của `delivered` + `shipped` + `returned` |
| **`M06 Gross Profit` (CHUẨN)**| `revenue_eligible_flag = TRUE` | **`818,618,351.03`** | `= M04 (13,210.80M) - M05 (12,392.18M)` |
| **`M07 Gross Margin %` (CHUẨN)**| `revenue_eligible_flag = TRUE` | **`6.20%`** (`6.1966%`) | `= M06 / M04` |
| **Chỉ số Hành vi & Chất lượng**| Toàn kỳ | Repeat Rate (`M32`): **`75.23%`** (`67,888 / 90,246` khách) \| Đơn/khách (`M33`): **`7.17`** \| Rating TB (`M40`): **`3.94`** \| Rating <= 2 (`M41`): **`13.1%`** \| Promo Line Share (`M21`): **`38.66%`** | Kiểm tra logic DAX nâng cao |
| **Đơn hàng & Trả hàng** | Toàn kỳ | `M14`: **`9.19%`** (59,462 / 646,945) \| `M15`: **`6.54%`** (36,142 / 552,858) \| `M15b`: **`5.59%`** (36,142 / 646,945) | Đã kiểm chứng trên `orders.csv` |
| **Giá & Khuyến mãi** | Toàn kỳ | `M20`: **`4.56%`** \| `M22`: **`92.86%`** \| Dòng có `secondary_promotion_id`: **`206`** | Đã kiểm chứng |
| **Giao vận** | 566,067 shipment | `M26`: **`1.50`** ngày \| `M27`: **`4.50`** ngày \| `M28`: **`41.72%`** \| Đơn hợp lệ thiếu shipment: **`564`** | Đã kiểm chứng |
| **Tồn kho** | 60,247 snapshot | SKU phủ: **`1,624`** (67.3%) \| `M53` Median: **`240`** ngày \| Max cover days: **`68,100`** | Đã kiểm chứng |
| **Số đơn theo năm (`M10`)** | Từng năm | 2012: 32,051 \| 2013: 76,849 \| 2014: 80,645 \| 2015: **82,622** \| 2016: 82,247 \| 2017: 76,010 \| 2018: **69,510** \| 2019: **41,601** \| 2020: 34,881 \| 2021: **34,525** \| 2022: 36,004 | North Star |
| **Web (`M60`, `M61`)** | Từng năm | Sessions 2013: **6,801,940** → 2022: **11,063,658** \| `M61` 2013: **1.13%** → 2019: 0.42% → 2021: 0.31% → 2022: **0.33%** | North Star |
| **Ngoại lệ dữ liệu (Trang 8)** | Toàn bộ | Cặp `(order, product)` trùng: **16** (Q1) \| Cặp trả hàng trùng: **2** (Q2) \| Ngày `COGS > Revenue`: **382** | Đã kiểm chứng |
