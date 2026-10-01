# Thành viên nhóm — Day 13

Điền trong thư mục nhóm private, không commit bản có thông tin cá nhân lên repo public.

Mã nhóm/phòng: **LabC301 - K4-DAY13-Nhom02**

| Họ và tên | MSSV | Vai trò lượt A | Vai trò lượt B | Vai trò lượt C |
| --- | --- | --- | --- | --- |
| **Phạm Đại Phúc** | 2A202602256 | Runner & Config Analyst | Runner & Environment Lead | Runner & Pipeline Analyst |
| **Nguyễn Văn Thành Long** | 2A202602284 | Visualizer & QC Auditor | Visualizer & Logger | QC Auditor & Report Lead |

---

## Nhận xét cá nhân của các thành viên

### 1. Thành viên 1: Phạm Đại Phúc (MSSV: 2A202602256)
- **Vai trò đảm nhiệm:** Runner & Config Analyst trực tiếp cấu hình môi trường, nạp Docker image `day13-pointpillars:lab` native trên phần cứng Dell Precision 5530 (Intel Core i7-8850H, 32GB RAM, kiến trúc amd64/x86_64, Windows 11). Thực thi thành công pipeline 3 lượt A/B/C và sinh bộ ca kiểm thử QC (trạng thái: `executed-by-group`).
- **Phân tích cơ chế biến đổi tọa độ:**
  - Áp dụng công thức chuyển đổi trước và sau inference:
    $$z_{\text{model}} = z_{\text{source}} - z_{\text{ground}} - \text{delta}$$
    $$z_{\text{source}} = z_{\text{model}} + z_{\text{ground}} + \text{delta}$$
  - Với KITTI point cloud, cảm biến LiDAR được đặt trên nóc xe (cách mặt đất xấp xỉ 1.73m). Việc cấu hình $\text{delta} = 1.73\,\text{m}$ (kèm $z_{\text{ground}} = 0.075\,\text{m}$) là bắt buộc để đưa điểm mây về hệ quy chiếu chuẩn mà mạng PointPillars đã được pretrained (nơi mặt đường nằm quanh $z \approx 0$).
  - Ở lượt A ($\text{delta} = 0.0\,\text{m}$), toàn bộ cụm điểm bị treo lơ lửng cao hơn 1.73m so với anchor boxes mà mạng nơ-ron tìm kiếm. Kết quả là mạng chỉ phát hiện được duy nhất 1 hộp thay vì 13 hộp như lượt B.
- **Phân tích suy giảm độ phân giải không gian (Run C vs Run B):**
  - Khi tăng `voxel_size` từ $0.16\,\text{m}$ lên $0.32\,\text{m}$, diện tích đáy mỗi cột pillar tăng gấp 4 lần ($0.0256\,\text{m}^2 \to 0.1024\,\text{m}^2$).
  - Sự suy giảm độ phân giải không gian nghiêm trọng (Spatial Resolution Drop) khiến các đặc trưng biên dạng của xe con (vehicles) bị làm nhòe và gộp chung vào các cột lớn. Mạng PointPillars mất hoàn toàn 10 hộp `vehicles` và nhận diện nhầm thành 6 hộp `pedestrian` do kích thước cụm điểm bị biến dạng.
- **Quyết định xử lý sự cố lỗi QC:**
  - Khi gặp ca lỗi `case-batch-z` (toàn bộ 13/13 hộp bị tụt đồng loạt $-1.805\,\text{m}$): **DỪNG BATCH NGAY LẬP TỨC (Stop Batch)**, tuyệt đối không chỉnh sửa thủ công trên CVAT. Lỗi này bắt nguồn từ sai lệch pipeline/calibration trong khâu bù tọa độ ngược ($z_{\text{ground}} + \text{delta}$). Cần báo ngay cho Lab Coordinator (LC) để kiểm tra và nạp lại prediction chuẩn.
- **Xử lý độ không chắc chắn (Uncertainty Handling):**
  - Đối với các phương tiện ở xa (>30m), mật độ tia LiDAR thưa thớt (chỉ còn vài điểm ở phần đuôi hoặc nóc), tuân thủ nguyên tắc duy trì kích thước bounding box chuẩn theo kích thước vật lý trung bình của dòng xe (~$4.5\,\text{m} \times 1.8\,\text{m} \times 1.5\,\text{m}$), tuyệt đối không co cụm hộp lại chỉ vừa khít vài điểm LiDAR rời rạc.

---

### 2. Thành viên 2: Nguyễn Văn Thành Long (MSSV: 2A202602284)
- **Vai trò đảm nhiệm:** Visualizer & QC Auditor, chịu trách nhiệm trực quan hóa và đối chiếu hình ảnh Side view ($x$-$z$), rà soát file JSON/CSV giữa các lượt chạy và thực hiện kiểm toán QC 3 ca lỗi có kiểm soát.
- **Phân tích giới hạn của ảnh chiếu Side view ($x$-$z$):**
  - Ảnh chiếu Side view chiếu toàn bộ không gian 3D lên mặt phẳng đứng $x$-$z$, triệt tiêu hoàn toàn trục hoành $y$ (chiều ngang xe/làn đường).
  - Do trục $y$ bị nén phẳng, các vật thể nằm song song hoặc lệch làn ở cùng khoảng cách $x$ sẽ bị chồng lấn điểm lên nhau, gây hiện tượng che khuất giả hoặc ngộ nhận cụm điểm. Ngoài ra, ảnh Side view hoàn toàn mất thông tin về góc xoay (heading/yaw) trên mặt phẳng $x$-$y$.
  - Do đó, ảnh Side view chỉ có giá trị sàng lọc thô lỗi độ cao $z$ (phát hiện hộp bay/hộp chìm dưới đất). Việc kiểm định chất lượng cuboid bắt buộc phải kết hợp Top view (BEV), Front view và camera RGB đồng bộ.
- **Nguyên tắc cốt lõi về Pre-label:**
  - Khẳng định mô hình AI chỉ sinh ra **initial priors** (gợi ý thô) để tăng tốc độ gán nhãn, **tuyệt đối không phải Ground Truth**.
  - Đối với ca `case-one-box-z`, kiểm tra từng hộp trên CVAT, đối chiếu đa góc nhìn để chỉnh sửa tọa độ tâm $z$ và đáy hộp sát mặt đường cục bộ. Mọi quyết định gán nhãn cuối cùng phải dựa trên bằng chứng hình học 3D xác thực từ con người.
