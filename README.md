# T03 — Phát hiện bất thường trong độ trễ mạng theo thời gian

Môn Mạng máy tính (COSH201) — Trường Đại học Ngoại thương.

## 1. Thành viên nhóm và mã sinh viên

| Họ tên | MSSV |
|---|---|---|
| Nguyễn Phương Vy | 2519960060 | 
| Lê Anh Minh | 2519960030 | 
| Nguyễn Vũ Duy Hải | 2519960015 |  
| Trần Ngọc Cẩm | 2519960009 | 

## 2. Mô tả ngắn bài toán

Phát hiện các khoảng thời gian có RTT hoặc các chỉ số mạng liên quan (jitter,
loss rate) khác biệt đáng kể so với trạng thái thường gặp, dựa trên dữ liệu ping
ICMP thực tế thu thập tại mạng WiFi ký túc xá, tới 4 đích: gateway nội bộ
(`10.70.0.1`), 2 DNS công cộng (`8.8.8.8` Google, `1.1.1.1` Cloudflare), và 1
server quốc tế (`www.google.com`). Áp dụng song song 2 phương pháp — baseline
ngưỡng thống kê (mean ± k·std) và mô hình Isolation Forest — huấn luyện riêng
theo từng đích, sau đó đối chiếu kết quả 2 phương pháp bằng chỉ số Jaccard.

## 3. Môi trường chạy và Python version

Môi trường chạy và Python version thiết bị của nhóm đều được cập nhật ở phiên bản mới nhất
- Hệ điều hành: Windows 11.
- Python 3.13

## 4. Thư viện cần cài

```
pip install -r requirements.txt
```

- `src/ping_collector.py` chỉ dùng thư viện chuẩn của Python, không cần cài thêm.
- `src/export_dataset.py` cần `pandas`. 
- Phần feature engineering/ mô hình/ trực quan hoá (notebook) cần thêm `numpy`, `scikit-learn` (dùng `StandardScaler`,
`IsolationForest`) và `matplotlib`.

## 5. Dataset: nguồn, vị trí file, cách tạo

- **Nguồn**: dataset tự thu thập  — ping ICMP thực tế tới 4 đích nêu, chu kỳ danh nghĩa 30 giây/lần, qua mạng WiFi ký
  túc xá dùng chung (KTX_FTU), cùng dải mạng với gateway `10.70.0.1`.
- **Thời gian thu thập**: từ 2026-08-19 15:58:52 UTC (22:58:52 giờ VN) đến
  2026-08-22 18:08:29 UTC (23/08 01:08:29 giờ VN) — khoảng 3 ngày liên tục.
- **Quy mô**: 35.600 dòng dữ liệu thô (8.900 dòng/đích). Sau khi loại 11 dòng
  trùng lặp (xem mục Chất lượng dữ liệu bên dưới): còn lại 35.589 dòng.
- **Vị trí file**:
  - `data/raw/ping_dataset_raw_full.csv` — dataset thô đã gộp toàn bộ các
    ngày (do `src/export_dataset.py` tạo ra), 8 cột, chưa xử lý duplicate/feature.
  - `data/processed/ping_dataset_processed.csv` — dataset sau xử lý duplicate,
    thêm cột `is_loss` và các đặc trưng rolling
- **Cách tạo**: xem mục 7.

### Chất lượng dữ liệu (tóm tắt từ Chương 3 báo cáo)

- **Trùng lặp**: 11 dòng, do sự cố laptop bị gập lại khi script đang chạy (script
  tạm dừng ~264 giây, sau đó chạy bù dồn dập gây trùng mốc thời gian) → đã loại
  bỏ hoàn toàn 11 dòng này, không gộp/trung bình.
- **Thiếu (missing)**: 312 dòng thiếu `rtt_ms`/`ttl` — trùng khớp chính xác với
  312 dòng có `status != ok` (288 timeout, 12 transmit_failed, 12 dns_error),
  do bản chất cấu trúc nên không điền khuyết, giữ nguyên NaN, đánh
  dấu bằng cột `is_loss`.
- **Sự cố tham chiếu (ground truth) dùng để kiểm chứng mô hình**: 1 sự cố mất
  kết nối WiFi thực tế kéo dài khoảng 32 phút (10:41–11:13 UTC, 22/08/2026), xác
  nhận qua việc cả 4 đích (kể cả gateway) đồng thời lỗi.

## 6. Cấu trúc thư mục

```
Group01_TopicT03/
├──Report_Group01.docx
├──Report_Group01.pdf
├── README.md
├── requirements.txt
├── data/
│   └── raw/    
|       └── ping_dataset_raw_full.csv        # CSV thô tổng hợp từ các file theo ngày, giữ nguyên data
|       └── chú thích dataset                # ghi lại thông tin & hoạt động/ lỗi xuất hiện
│   └── processed/
│       └── ping_dataset_processed.csv      # đã xử lý duplicate + feature
|       └── giải thích code                 # giải thích phần feature engineering
├── src_thu_thập_data/
│   └── ping_collector.py            # thu thập dữ liệu ping đa đích theo chu kỳ
│   └── run_forever.bat              # wrapper chạy ping_collector.py, tự khởi động lại khi crash
│   └── export_dataset.py            # gộp CSV theo ngày + báo cáo QC
├── notebook/
│   └── 01_preprocessing_and_features.ipynb        # feature engineering,
│   └── 02_model_and_visualization.ipynb           # baseline, Isolation Forest, trực quan hóa
└── results/
    └── figures/                     # Hình trong báo cáo
    └── tables/                      # Bảng trong báo cáo
        └── summary_stats_by_target.csv
        
```

## 7. Thứ tự chạy các script

Phần thu thập dữ liệu:
1. Nhấn đúp `run_forever.bat` để chạy liên tục `python ping_collector.py` nhiều ngày, dừng bằng `Ctrl+C`.
3. `python export_dataset.py` — gộp dữ liệu, in báo cáo QC, tạo `ping_dataset_raw_full.csv`.

Phần feature engineering / mô hình (thực hiện trong notebook, theo đúng 5 giai
đoạn mô tả tại Bảng 4.1 của báo cáo):
1. Nạp `ping_dataset_raw_full.csv`, xử lý duplicate (loại 11 dòng đã xác định)
   xuất `ping_dataset_processed.csv`.
2. Chuẩn bị đặc trưng: tính rolling mean/std/P95/jitter/loss_rate (cửa sổ 20
   điểm ≈ 10 phút), xử lý cold-start đầu chuỗi, tách riêng nhóm `is_loss = 0`.
3a. Baseline: gắn cờ bất thường theo `|RTT − rolling_mean| > k·rolling_std`
    (k = 3), tính riêng theo từng `target_name`.
3b. Isolation Forest: chuẩn hoá bằng `StandardScaler`, huấn luyện riêng theo
    từng `target_name` (`n_estimators=200`, `contamination=0.02`,
    `random_state=42`).
4. Hợp nhất kết quả (OR giữa rule mất gói và kết quả pattern), so sánh 2
   phương pháp bằng chỉ số Jaccard.
5. Xuất kết quả: timeline đánh dấu anomaly, bảng thống kê, biểu đồ so sánh.

## 8. Output mong đợi

Chương trình tạo ra 4 nhóm output chính:

- Dữ liệu: dữ liệu ping thô theo từng ngày, dữ liệu đã gộp và dữ liệu đã qua xử lý (loại trùng lặp, bổ sung đặc trưng rolling theo từng đích đo).
- Bảng số liệu: bảng thống kê mô tả RTT và tỷ lệ mất gói theo từng đích (mean, median, std, P95, jitter, loss rate), cùng bảng so sánh kết quả giữa phương pháp baseline và Isolation Forest.
- Hình ảnh: biểu đồ chuỗi thời gian RTT có đánh dấu các khoảng bất thường theo từng đích, biểu đồ so sánh tỷ lệ phát hiện bất thường giữa 2 phương pháp.
- Cấu hình mô hình: toàn bộ tham số thực nghiệm (kích thước cửa sổ, hệ số baseline, tham số Isolation Forest) được lưu lại tường minh phục vụ tái lập.
Kết quả tổng hợp: Baseline phát hiện 614 tín hiệu bất thường (302 do RTT biến động, 312 do mất gói); Isolation Forest phát hiện 1.020 tín hiệu (708 theo điểm số mô hình, 312 do mất gói); trong đó 484 bất thường được cả 2 phương pháp cùng phát hiện. Mức độ tương đồng giữa 2 phương pháp, đo bằng chỉ số Jaccard, đạt 0,42 (giảm còn 0,15 nếu không tính nhóm mất gói — cho thấy 2 phương pháp thực sự khác biệt rõ khi xét riêng phần bất thường do biến động RTT).

### Schema `ping_dataset_raw_full.csv` (dữ liệu thô, 8 cột)

| Cột | Kiểu | Ý nghĩa |
|---|---|---|
| `timestamp_utc` | datetime (ISO 8601, UTC) | Thời điểm đo thực tế |
| `timestamp_vn` | datetime | Quy đổi giờ Việt Nam (UTC+7) |
| `target_name` | categorical | `gateway`, `dns_google`, `dns_cloudflare`, `intl_server` |
| `target_host` | categorical | IP/hostname của đích |
| `rtt_ms` | float (có thể NaN) | RTT đo được; chỉ tồn tại khi `status = ok` |
| `ttl` | int (có thể NaN) | TTL của gói phản hồi; đổi đột ngột gợi ý đổi tuyến đường |
| `status` | categorical | `ok`, `timeout`, `transmit_failed`, `dns_error`, `unreachable` |
| `note` | text | Ghi chú/mô tả lỗi, phục vụ debug, không dùng để tính toán |

## 9. Ghi chú về khả năng tái lập kết quả

- Phần thu thập dữ liệu: Chạy lại `ping_collector.py` sẽ tạo dữ liệu mới đo điều kiện mạng thực tế tại thời
  điểm chạy, không tái lập được giá trị RTT giống hệt bản gốc, nhưng tái lập
  đúng quy trình/định dạng dữ liệu.
- Phần mô hình: có yếu tố ngẫu nhiên ở Isolation Forest, đã cố định bằng
  `random_state = 42` — chạy lại trên đúng `ping_dataset_processed.csv` đã nộp
  kèm sẽ cho kết quả giống hệt. Toàn bộ tham số khác gồm kích thước cửa sổ rolling
  = 20 điểm, hệ số baseline k = 3, `n_estimators = 200`,
  `contamination = 0.02`.

## 10. Giới hạn và lưu ý an toàn

- Dữ liệu chỉ phản ánh điều kiện mạng tại 1 phòng ký túc xá cụ thể trong 3 ngày,
  không đại diện cho toàn bộ mạng của trường hay Internet nói chung; baseline
  không đảm bảo còn đúng nếu điều kiện mạng thay đổi lâu dài.
- Isolation Forest huấn luyện và đánh giá trên cùng 1 tập dữ liệu (không giám
  sát, không có tập kiểm tra độc lập); `contamination = 0.02` là giả định chủ
  quan, chưa kiểm định độ nhạy với giá trị khác.
- Việc đối chiếu với sự cố WiFi đã biết chỉ mang tính kiểm chứng định tính trên
  1 sự kiện, không đủ để tính precision/recall định lượng đầy đủ.
- Kết quả phát hiện bất thường KHÔNG được diễn giải thành kết luận về nguyên
  nhân cụ thể (tấn công mạng, lỗi hạ tầng ISP...) nếu không có bằng chứng xác
  thực bổ sung ngoài dữ liệu ping.
- Tuân thủ quy định an toàn: toàn bộ dữ liệu phân tích là traffic ICMP do
  chính nhóm tạo ra gửi tới 4 đích công khai. Không quét port, không
  khai thác lỗ hổng, không có hoạt động ARP poisoning/SYN flood/DoS/DNS
  tunneling. Tần suất 4 gói/30 giây tới mỗi đích, không tạo tải đáng kể lên bất kỳ dịch vụ nào.