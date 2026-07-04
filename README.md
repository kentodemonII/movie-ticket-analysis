# 🎬 Phân tích hành vi mua vé xem phim

## Mục tiêu project

Project được thực hiện để luyện phân tích dữ liệu bằng Python sau khi hoàn thành phần cơ bản của Pandas. Chủ đề vé xem phim được chọn vì dữ liệu có cấu trúc gần giống một hệ thống thương mại điện tử thực tế: thông tin khách hàng, lịch sử giao dịch, chương trình khuyến mãi, thiết bị đặt vé, và trạng thái giao dịch (thành công/thất bại). Đây không chỉ đơn thuần là vẽ biểu đồ mà đòi hỏi join nhiều bảng dữ liệu trước khi tìm ra insight.

Câu hỏi chính đặt ra: khách mua vé là ai, họ mua vào thời điểm nào, và công ty đang giữ chân khách hàng hiệu quả tới đâu.

## Dữ liệu

5 bảng CSV theo mô hình quan hệ:

| Bảng | Nội dung |
|---|---|
| `customer` | thông tin khách hàng: ngày sinh, giới tính |
| `ticket_history` | lịch sử giao dịch: mã vé, thời gian, phim, giá gốc, giảm giá, giá cuối |
| `campaign` | thông tin chương trình khuyến mãi đi kèm giao dịch (nếu có) |
| `device_detail` | thiết bị dùng để đặt vé (model, platform) |
| `status_detail` | trạng thái giao dịch và mô tả lỗi (nếu thất bại) |

Dữ liệu trải dài từ 2019 đến 2022, trùng khoảng thời gian Covid nên phần trend theo tháng xuất hiện một đoạn sụt giảm rõ rệt, được ghi chú lại trong phần phân tích thời gian.

## Quy trình phân tích

### 1. Làm sạch và ghép dữ liệu
5 bảng được join thành 1 bảng tổng (`df_j_all`) qua các khóa `customer_id`, `campaign_id`, `device_number`, `status_id`. Trước khi join, dữ liệu được xử lý:
- Chuyển `dob` và `time` sang định dạng datetime
- Loại bỏ dòng trùng lặp trong `ticket_history`
- Điền `unknown` cho giá trị thiếu ở cột `model` thiết bị
- Sau khi join, các cột không khớp (do left join) được fill `unkown` để thuận tiện cho việc group dữ liệu

### 2. Chân dung khách hàng
Tuổi được tính từ `dob`, phân loại theo thế hệ (Gen Z / Gen Y / Gen X / Baby Boomer). Một điểm đáng chú ý: nhóm khách "chưa xác thực tài khoản" (`Not verify`) chiếm hơn 10% và phần lớn bị hệ thống auto-fill năm sinh 1970 (tương đương 55 tuổi) — nếu không tách riêng, nhóm này sẽ làm méo hoàn toàn phân bố tuổi thật. Nhóm này được loại khỏi các phân tích demographic chính để tránh nhiễu số liệu.

### 3. Xu hướng theo thời gian
Số vé bán ra được phân tích theo tháng, theo thứ trong tuần, và theo giờ trong ngày. Do dữ liệu thiếu một số tháng (đặc biệt giai đoạn Covid 2020–2021), một bảng dim_time đầy đủ mốc tháng được dựng thêm rồi left-join vào, giúp biểu đồ không bị gãy trục thời gian do thiếu dữ liệu.

### 4. Các yếu tố ảnh hưởng đến hành vi mua
Hành vi mua được phân tích theo platform (mobile/website), hệ điều hành thiết bị, phương thức thanh toán, và việc có sử dụng khuyến mãi hay không — cả về tỷ trọng lẫn xu hướng theo thời gian.

### 5. Giá trị khách hàng & phát hiện bất thường
Các chỉ số được tính theo từng khách hàng: số lần mua, tổng chi tiêu, tỷ lệ giao dịch thành công, tỷ lệ dùng khuyến mãi, tỷ lệ được giảm giá. Mục tiêu của bước này là trả lời câu hỏi: có khách hàng nào đang mua vé với hành vi bất thường (kiểu spam/gian lận) không?

Nhóm khách mua từ 30 vé trở lên được kiểm tra riêng — ban đầu đây là nghi vấn về hành vi bất thường, nhưng khi phân tích theo tháng, các giao dịch này phân bố đều theo thời gian chứ không dồn vào một thời điểm cụ thể, cho thấy đây là hành vi mua thực tế chứ không phải spam.

### 6. Phân tích Cohort — giữ chân khách hàng
Đây là phần mang lại insight rõ ràng nhất trong project. Khách hàng được nhóm theo tháng mua vé lần đầu, sau đó theo dõi tỷ lệ quay lại ở các tháng tiếp theo (retention theo cohort, thể hiện qua heatmap).

**Insight đáng chú ý nhất:** trong năm 2022, 97% các giao dịch có khuyến mãi thuộc về khách hàng mới, nhưng tỷ lệ quay lại mua vé lần 2 chỉ khoảng 13% — gần như không khác biệt so với nhóm khách không qua khuyến mãi (12%). Điều này cho thấy ngân sách khuyến mãi đang được sử dụng chủ yếu để thu hút khách mới, trong khi hiệu quả giữ chân gần như không cải thiện — một dấu hiệu cho thấy chiến lược promotion cần được xem xét lại thay vì chỉ tối ưu số lượng khách mới.

### 7. Tỷ lệ thành công & phân tích lỗi
Success rate được theo dõi theo tháng, kèm phân loại lỗi giao dịch thất bại (lỗi từ ngân hàng, lỗi phía khách hàng, lỗi hệ thống nội bộ). Với nhóm khách có success rate gần 0%, phân tích sâu hơn cho thấy phần lớn nguyên nhân đến từ phía ngân hàng (không phản hồi, không đủ số dư) — tức nằm ngoài khả năng xử lý trực tiếp của hệ thống, không phải lỗi kỹ thuật nội bộ.

## Hạn chế và hướng phát triển tiếp theo

- Chưa có mô hình dự đoán churn — phân tích hiện tại dừng ở mức mô tả (descriptive); hướng mở rộng khả thi là logistic regression để dự đoán khách hàng có khả năng không quay lại
- Phần đánh giá loại khuyến mãi mới dừng ở tỷ lệ sử dụng, chưa so sánh được với doanh thu tăng thêm thực tế của từng loại
- Code trong notebook chưa được refactor thành các hàm tái sử dụng, do ưu tiên ban đầu là khám phá dữ liệu nhanh hơn là viết code có cấu trúc

## Công cụ sử dụng
Python (Pandas, NumPy) · Matplotlib, Seaborn để trực quan hóa · Google Colab

---
📓 Notebook đầy đủ: xem file `.ipynb` trong repo (kèm output và biểu đồ đã chạy)
