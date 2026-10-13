# NimbusAI GPU FinOps — Bài viết ngắn

## 1. Baseline và optimized

Theo phép tổng hợp của M5, chi phí baseline ước tính là **$27,133/tháng**; sau
khi áp dụng các tối ưu, chi phí còn khoảng **$14,626/tháng**. NimbusAI tiết kiệm
khoảng **$12,507/tháng, tương đương 46.1%**. Đây là mô hình tổng hợp từ mức dùng
token một ngày nhân 30 ngày và chi phí GPU theo workload trong một tháng, không
phải hóa đơn thực tế.

Riêng workload inference trong M2, baseline là **$48.87/ngày** hay
**$6.488/1M-token**. Sau khi chuyển một phần request sang model nhỏ hơn, dùng
cache và batch, chi phí còn **$8.48/ngày** hay **$1.126/1M-token**: giảm **82.6%**
trên tập token được phân tích. Con số `$ / 1M-token` này chỉ áp dụng cho inference;
không chia tổng hóa đơn GPU của M5 cho tập token vì hai mô hình có phạm vi dữ
liệu và workload khác nhau.

## 2. Đòn bẩy tiết kiệm

Đóng góp lớn nhất trong M5 đến từ **mua GPU theo tier phù hợp**: spot cho workload
có thể gián đoạn và reserved cho workload ổn định. Đòn bẩy này tiết kiệm khoảng
**$10,040/tháng**, tương đương **80.3% tổng mức tiết kiệm**. Lý do là GPU-hours
chiếm phần lớn baseline và giá spot/reserved thấp hơn on-demand đáng kể.

Các đóng góp còn lại là inference **$1,212/tháng** (9.7%), right-sizing các GPU
bị util-lie **$655/tháng** (5.2%) và tắt GPU idle **$600/tháng** (4.8%). Đây là
các con số do M5 tổng hợp, không cộng thêm các ước tính của M6 vào tiền tiết
kiệm: M6 đo phát thải, không định giá chi phí vùng hay di chuyển workload.

## 3. GPU-Util Lie và tác động tài chính

M1 phát hiện hai GPU có utilization trung bình trên 90% nhưng MFU dưới 30%:

| GPU | GPU-Util trung bình | MFU | MBU |
|---|---:|---:|---:|
| `gpu-h100-4` | 98.2% | 19.4% | 20.7% |
| `gpu-a10g-1` | 96.9% | 26.8% | 30.2% |

GPU-Util chỉ cho biết thiết bị đang bận, không chứng minh rằng nó đang tạo ra
nhiều FLOPs hữu ích. Với M5, chuyển một H100 sang A100 và một A10G sang L4 theo
bản đồ right-size giúp tiết kiệm mô hình khoảng **$655/tháng**. Đây là ước tính
giả định dùng liên tục 24 giờ/ngày trong 30 ngày; trước khi thay đổi production,
cần kiểm tra bộ nhớ, thông lượng, độ trễ và chất lượng đầu ra để xác nhận GPU nhỏ
hơn vẫn đáp ứng SLA.

## 4. Các phần mở rộng đã thực hiện

**Extension 3 — quyết định có nên cache.** `cache_is_worth_it()` so sánh tổng
tiết kiệm từ các lần đọc lại với chi phí ghi cache; hàm cũng cho phép truyền giá
đọc uncached riêng. Với giá đọc mặc định bằng giá ghi và mức giảm giá đọc 90%,
điểm hòa vốn là **hơn 1.11 lần đọc lại**: một lần đọc chưa đủ, hai lần thì cache
có lợi. Hàm kiểm tra cả giá trị âm và không hữu hạn. Đây là phép thử hòa vốn,
không phải khoản tiết kiệm thực tế được cộng vào M2.

**Extension 5 — carbon-aware scheduling.** Với năm workload interruptible trong
`workloads.csv`, M6 ước tính điện năng từ watt GPU trong catalog, giả định GPU
tiêu thụ ở mức đó liên tục trong thời gian chạy. Chuyển giả định từ
`us-east-1` (380 gCO₂e/kWh) sang vùng sạch nhất `europe-north1`
(30 gCO₂e/kWh) giảm ước tính từ **679,820 xuống 53,670 gCO₂e**: tiết kiệm
**626,150 gCO₂e, tức 92.1%**. Insight chính là cùng lượng điện sử dụng vẫn tạo
ra dấu chân carbon rất khác nhau tùy cường độ carbon của lưới điện. Đây là mô
phỏng theo vùng, chưa bao gồm giới hạn vị trí dữ liệu, độ trễ, giá điện hoặc khả
năng cung cấp GPU.

## 5. Ba hành động đầu tiên cho NimbusAI

1. **Áp dụng purchasing policy có kiểm soát:** chuyển job có checkpoint và khả
   năng retry sang spot; chỉ cam kết reserved cho tải nền ổn định sau khi so
   sánh thời hạn hợp đồng, utilization và giá 1yr/3yr. Theo dõi interruption,
   thời gian khôi phục và chi phí thực tế theo GPU type.
2. **Điều tra và right-size các util-lie:** ưu tiên H100 `gpu-h100-4` và A10G
   `gpu-a10g-1`; chạy benchmark với model và batch đại diện, rồi triển khai canary
   có ngưỡng rollback dựa trên latency, throughput, memory và chất lượng.
   Đồng thời tự động tắt GPU idle thay vì để tài nguyên chạy qua đêm.
3. **Đưa tối ưu inference và carbon vào vận hành:** đo chi phí `$ / 1M-token`
   theo route, cache-hit và batch; bật cache khi số lần tái sử dụng vượt điểm
   hòa vốn. Lập thử nghiệm chuyển workload interruptible sang vùng carbon thấp,
   nhưng chỉ sau khi xác nhận tuân thủ dữ liệu, SLA, chi phí và năng lực vùng.

*Các kết quả là ước tính trên dữ liệu tổng hợp và giá snapshot trong lab. Cần
đối chiếu billing, telemetry và SLA production trước khi chốt mục tiêu tiết
kiệm.*
