# Báo cáo thực hành PointPillars — Day 13

> **Ghi chú:** Bản báo cáo chính thức của nhóm được lưu trữ tại thư mục private `report/K4-DAY13-Nhom02/` theo quy định của Ban tổ chức. Dữ liệu thực nghiệm thực tế được đối chiếu từ output A/B/C và bộ ca kiểm thử QC.

---

## 1. Môi trường Thực thi & Artifacts (Provenance)

- **Mã nhóm / Phòng thực hành:** `LabC301 - K4-DAY13-Nhom02`
- **Thành viên nhóm:** Chi tiết phân công và vai trò xem tại [`TEAMMATES.md`](TEAMMATES.md):
  - **Phạm Đại Phúc** (MSSV: `2A202602256`) — *Runner & Config Analyst*
  - **Nguyễn Văn Thành Long** (MSSV: `2A202602284`) — *Visualizer & QC Auditor*
- **Trạng thái thực thi:** `executed-by-group` (Toàn bộ 3 cấu hình A/B/C và pipeline QC do nhóm tự cấu hình và chạy trực tiếp, không sử dụng `provided-results`).
- **Phần cứng thực thi:** Dell Precision 5530
  - CPU: Intel Core i7-8850H (6 cores, 12 threads @ 2.60GHz up to 4.30GHz)
  - RAM: 32 GB DDR4
  - Kiến trúc: `x86_64` / `amd64`
  - Hệ điều hành host: Windows 11 Pro 64-bit
- **Cấu hình Container runtime:**
  - Docker Desktop on Windows (WSL2 backend)
  - Giới hạn tài nguyên container: `--cpus 4 --memory 4g --network none` (hoàn toàn cô lập mạng trong quá trình chạy inference).
- **Docker Image Tag:** `day13-pointpillars:lab`
- **Docker Image ID:** `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`
- **Phiên bản mã nguồn repo:** `phucdevelopervippro/K4-DAY13-PhamDaiPhuc-2A202602256`
- **Dữ liệu đầu vào (Input PCD):**
  - File: `data/demo.pcd` (Trích xuất từ frame `KITTI 000008`)
  - Tổng số điểm LiDAR: `17,238` điểm
  - Cao độ mặt đất ước lượng ($z_{\text{ground}}$): `0.075 m`
  - Kênh thứ tư / Giả định Intensity: Dữ liệu nguồn đã loại bỏ giá trị reflectance gốc; áp dụng adapter hằng số đồng nhất (synthetic constant reflectance, kênh `RGB=0`). Tuyệt đối không nhầm lẫn RGB với intensity vật lý thật của chùm tia phản xạ LiDAR.
- **Checkpoint & Phạm vi mô hình:**
  - Checkpoint: PointPillars pretrained trên KITLidar benchmark tích hợp sẵn trong Docker image.
  - Vùng quan tâm (ROI): Front-window camera view chuẩn KITTI ($x \in [0, 70.4]\,\text{m}, y \in [-40, 40]\,\text{m}, z \in [-3, 1]\,\text{m}$).
  - Ngưỡng tin cậy (Score Threshold): `0.3` cho toàn bộ các lượt thử nghiệm.

---

## 2. Bảng số liệu ba lượt inference thật (A/B/C) và giải thích chi tiết

### Bảng tổng hợp số liệu thực nghiệm

| Lượt | delta (m) | Pillar XY (m) | Số hộp (`n_boxes`) | `mean_z` (m) | Phân bố lớp (Breakdown Classes) | File minh chứng (JSON / Side / CSV) | Quan sát có bằng chứng kỹ thuật |
| :---: | :---: | :---: | :---: | :---: | :--- | :--- | :--- |
| **A** | `0.0` | `0.16` | **1** | `0.330` | `vehicles`: 1 | `run-A/boxes-demo-delta-0-voxel-0.16.json`<br>`run-A/side-A.png`<br>`run-A/summary.csv` | Chỉ phát hiện 1 hộp duy nhất ở vùng gần; toàn bộ cụm điểm ở xa bị mất do lệch cao độ với anchor model. |
| **B** | `1.73` | `0.16` | **13** | `1.034` | `vehicles`: 10<br>`pedestrian`: 2<br>`two-wheels`: 1 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`<br>`run-B/side-B.png`<br>`run-B/summary.csv` | **Baseline chuẩn**: Bắt trọn 13 đối tượng với độ bám cụm điểm chính xác trên cả trục $x$-$z$. |
| **C** | `1.73` | `0.32` | **6** | `1.091` | `pedestrian`: 6 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`<br>`run-C/side-C.png`<br>`run-C/summary.csv` | Mất toàn bộ 10 hộp xe hơi (`vehicles`), nhận diện sai lệch thành 6 `pedestrian` do mất độ phân giải không gian. |

---

### Phân tích chuyên sâu giữa các lượt chạy

#### 1. So sánh Run A vs Run B (Biến thiên tham số `delta`):
- **Hiện tượng:** Giữ nguyên `voxel_size = 0.16m`, khi thay đổi `delta` từ $0.0\,\text{m}$ (Run A) lên $1.73\,\text{m}$ (Run B), số lượng bounding box tăng đột biến từ **1 hộp** lên **13 hộp**, và giá trị `mean_z` dịch chuyển từ $0.330\,\text{m}$ lên $1.034\,\text{m}$.
- **Nguyên nhân vật lý & thuật toán:**
  - Trong tập dữ liệu KITTI, cảm biến LiDAR Velodyne HDL-64E được gắn trên giá nóc xe thực nghiệm ở độ cao xấp xỉ $1.73\,\text{m}$ so với mặt đường.
  - Khi thiết lập `delta = 0.0m`, bước tiền xử lý chỉ trừ đi $z_{\text{ground}} = 0.075\,\text{m}$. Toàn bộ đám mây điểm đưa vào mạng nơ-ron vẫn bị treo lơ lửng ở cao độ dương cao hơn $1.73\,\text{m}$ so với hệ tọa độ chuẩn của KITTI. Trong khi đó, các prior anchor 3D của mô hình PointPillars được cố định quanh mặt phẳng mặt đường ($z \approx 0$).
  - Sự lệch pha hình học này khiến các cụm điểm không thể kích hoạt các anchor tương ứng, dẫn đến việc mô hình bỏ sót hầu như toàn bộ vật thể (false negatives hàng loạt), chỉ bắt được 1 hộp xe ở khoảng cách gần nơi mật độ điểm đủ dày đặc để vượt qua ngưỡng score 0.3.
  - Khi thiết lập đúng `delta = 1.73m`, đám mây điểm được đưa chính xác về mặt sàn chuẩn của anchor, giúp mạng trích xuất đặc trưng hình học chuẩn xác và phát hiện đầy đủ 13 đối tượng.

#### 2. So sánh Run B vs Run C (Biến thiên tham số `voxel_size` / Pillar XY):
- **Hiện tượng:** Giữ nguyên $\text{delta} = 1.73\,\text{m}$, khi tăng kích thước cạnh pillar từ $0.16\,\text{m}$ lên $0.32\,\text{m}$, số lượng hộp giảm mạnh từ **13 hộp** xuống **6 hộp**. Đáng chú ý nhất: toàn bộ 10 hộp `vehicles` và 1 hộp `two-wheels` biến mất hoàn toàn, thay vào đó mô hình chỉ phát hiện 6 hộp đều bị gán nhãn là `pedestrian`.
- **Nguyên nhân cấu trúc mạng (Spatial Resolution Drop):**
  - Trụ cột (Pillar) là các lăng trụ vô hạn theo trục đứng $z$ có đáy là hình vuông kích thước $\text{voxel\_size} \times \text{voxel\_size}$ trên mặt phẳng $x$-$y$.
  - Khi tăng cạnh từ $0.16\,\text{m}$ lên $0.32\,\text{m}$, diện tích đáy mỗi pillar tăng gấp 4 lần (từ $0.0256\,\text{m}^2 \to 0.1024\,\text{m}^2$). Việc này làm giảm mạnh độ phân giải của feature map sau tầng PointPillars Scatter (Pseudo-image).
  - Các đặc trưng biên dạng đặc trưng của xe ô tô (thân dài, mui, đuôi xe) bị gộp chung vào quá ít cột điểm, làm mất cấu trúc không gian cục bộ. Hơn nữa, mạng PointPillars này sử dụng checkpoint pretrained được huấn luyện tối ưu hóa ở độ phân giải $0.16\,\text{m}$. Khi đưa pseudo-image có kích thước lưới giảm một nửa vào mạng xương sống 2D CNN (Backbone), tỷ lệ co dãn spatial scale bị phá vỡ hoàn toàn, dẫn đến việc phân loại sai nghiêm trọng (misclassification) sang lớp `pedestrian`.
  - **Kết luận:** Số lượng hộp nhiều hơn ở B so với C không đơn thuần là "nhiều hộp hơn", mà là do cấu hình B giữ đúng độ phân giải không gian thiết kế của checkpoint pretrained.

#### 3. Giới hạn của Vùng quan tâm (ROI) và Ảnh chiếu Side view ($x$-$z$):
- **Ảnh hưởng của ROI:** Khung nhìn front-window chỉ quét nửa hình nón phía trước của xe tự hành. Các vật thể ở hai bên hông hoặc phía sau xe nằm ngoài ROI sẽ không xuất hiện trong bounding box. Đây không phải lỗi bỏ sót của mô hình mà do phạm vi giới hạn có chủ đích của bài thực hành.
- **Giới hạn của ảnh Side view:**
  - Ảnh chiếu ngang $x$-$z$ đã triệt tiêu hoàn toàn trục $y$. Hai xe chạy song song ở cùng khoảng cách $x$ nhưng khác làn đường $y$ sẽ bị chiếu đè lên cùng một tọa độ trên ảnh, tạo ảo giác chồng chéo điểm hoặc che khuất giả.
  - Ảnh Side view không thể hiện được góc quay quanh trục đứng (yaw / heading angle).
  - Do đó, ảnh Side view chỉ có giá trị kiểm tra sơ bộ xem bounding box có nằm sát mặt đường hay bị "bay/chìm" theo trục $z$. Không thể dùng ảnh Side view độc lập để nghiệm thu chất lượng cuboid 3D; bắt buộc phải kết hợp Top view (BEV), Front view và camera quang học 2D.

---

## 3. Cơ chế biến đổi tọa độ thuận / ngược

### 1. Công thức toán học biến đổi hệ tọa độ

Quá trình suy luận của pipeline PointPillars trải qua 2 pha biến đổi tọa độ trục cao độ $z$:

$$\text{Pha thuận (Pre-inference):}\quad z_{\text{model}} = z_{\text{source}} - z_{\text{ground}} - \text{delta}$$

$$\text{Pha ngược (Post-inference):}\quad z_{\text{source}} = z_{\text{model}} + z_{\text{ground}} + \text{delta}$$

Trong đó:
- $z_{\text{source}}$: Cao độ thực của điểm mây / tâm bounding box trong hệ tọa độ cảm biến LiDAR (PCD nguồn).
- $z_{\text{ground}}$: Cao độ bề mặt đường ước lượng từ dữ liệu PCD nguồn ($z_{\text{ground}} = 0.075\,\text{m}$).
- $\text{delta}$: Tham số bù trừ cao độ lắp đặt cảm biến LiDAR đối với mặt đường theo cấu hình của checkpoint ($1.73\,\text{m}$ đối với KITTI).
- Tổng lượng dịch chuyển cao độ: $\text{offset} = z_{\text{ground}} + \text{delta} = 0.075 + 1.73 = 1.805\,\text{m}$.

### 2. Bản chất khác biệt giữa Đổi delta trước inference vs Dịch hộp sau inference

| Tiêu chí | Thay đổi $\text{delta}$ trước inference (Pre-inference) | Dịch chuyển hộp sau inference (Post-inference) |
| :--- | :--- | :--- |
| **Bản chất tác động** | Thay đổi trực tiếp phân bố không gian của đám mây điểm đầu vào đưa vào mạng nơ-ron. | Chỉ là phép tịnh tiến hình học đơn thuần trên tập tọa độ bounding box đầu ra ($z_{\text{box}} \pm \Delta z$). |
| **Tác động lên Feature Map** | Thay đổi hoàn toàn biểu diễn đặc trưng trong các cột pillar, thay đổi feature map trích xuất và tương quan với các 3D anchor boxes. | Hoàn toàn không tác động đến đặc trưng hay mạng nơ-ron; feature map đã kết thúc tính toán. |
| **Tính chất kết quả đầu ra** | **Phi tuyến tính (Non-linear):** Làm thay đổi số lượng hộp ($1 \to 13$), thay đổi độ tin cậy (score), thay đổi phân lớp và vị trí $(x, y, z)$. | **Tuyến tính tuyệt đối (Linear translation):** Số lượng hộp, phân lớp, kích thước $(l, w, h)$, tọa độ $(x, y)$ và góc xoay yaw giữ nguyên $100\%$. |

---

## 4. Bảng đánh giá ba ca lỗi QC có kiểm soát (QC Pipeline Cases)

Bộ ba ca kiểm thử được sinh trực tiếp bằng helper script `pipeline-qc-cases.py` từ kết quả prediction chuẩn của **Run B** (`boxes-demo-delta-1.73-voxel-0.16.json`) với độ lệch chủ đích $\text{offset} = z_{\text{ground}} + \text{delta} = 1.805\,\text{m}$. Đây là các trường hợp kiểm toán kiểm soát lỗi pipeline, tuyệt đối không phải kết quả của một model khác và không được import vào CVAT làm ground truth.

| Ca kiểm thử | Số hộp lệch $z$ / Tổng hộp | Lượng lệch $z$ ($\Delta z$) | `mean_z` thực tế | Trường $x, y, \text{class}, \text{yaw}$ có đổi? | Quyết định hành động | Bằng chứng kỹ thuật xác thực |
| :--- | :---: | :---: | :---: | :---: | :--- | :--- |
| **`case-correct`** | **0 / 13** (0%) | $0.000\,\text{m}$ | `1.034 m` | Không đổi | **Chấp thuận giữ nguyên** làm baseline đối chiếu; không import thẳng làm GT. | Giữ nguyên phép chuyển đổi ngược $z_{\text{source}} = z_{\text{model}} + 1.805\,\text{m}$. Tất cả 13 hộp ôm sát cụm điểm mây thật. |
| **`case-batch-z`** | **13 / 13** (100%) | **$-1.805\,\text{m}$** | `-0.771 m` | Không đổi | **DỪNG BATCH NGAY (Stop Batch)**, báo Lab Coordinator kiểm tra calibration. Tuyệt đối không sửa tay! | 100% hộp bị kéo tụt chìm sâu xuống lòng đất đồng loạt đúng một lượng $\Delta z = -1.805\,\text{m}$. Lỗi pipeline hệ thống do thiếu bước bù $z_{\text{ground}} + \text{delta}$. |
| **`case-one-box-z`** | **1 / 13** (7.7%) | **$-1.805\,\text{m}$** (chỉ hộp index 0) | `0.895 m` | Không đổi | **Kiểm tra từng hộp**, chỉnh sửa thủ công vị trí hộp index 0 trên CVAT. | Chỉ duy nhất 1 hộp tại index 0 bị lỗi cao độ, 12 hộp còn lại chuẩn xác hoàn toàn. Đây là lỗi mức đối tượng/cục bộ, không phải lỗi pipeline hệ thống. |

---

## 5. Khẳng định vai trò của Pre-label trong hệ thống xe tự hành (Autonomous Driving)

1. **Pre-label chỉ là Initial Priors (Gợi ý khởi đầu):**
   - Các thuật toán 3D Object Detection (như PointPillars, CenterPoint, VoxelNet) dù tiên tiến nhưng vẫn luôn có xác suất gặp lỗi: nhận diện nhầm (False Positives), bỏ sót vật thể (False Negatives), ước lượng sai kích thước (dimension error) hoặc phán đoán sai góc hướng (heading flip $180^\circ$).
   - Pre-label đóng vai trò trợ lực tăng tốc độ gán nhãn cho chuyên viên (giảm thiểu 60-70% thời gian dựng khung cuboid từ đầu), **tuyệt đối không được coi là Ground Truth**.
2. **Nguy cơ của việc lạm dụng Pre-label:**
   - Nếu đưa trực tiếp prediction chưa qua thẩm định của con người vào tập dữ liệu huấn luyện, mô hình xe tự hành thế hệ tiếp theo sẽ học lại chính những sai số này, gây hiện tượng tích lũy lỗi (error propagation / confirmation bias), dẫn đến các nguy cơ mất an toàn thực địa nghiêm trọng (như phanh gấp ma, đâm vào chướng ngại vật bị bỏ sót).
3. **Quy trình thẩm định đa góc nhìn bắt buộc (Human-in-the-loop):**
   - Mọi hộp cuboid bắt buộc phải được chuyên viên thẩm định và hiệu chỉnh trên nền tảng 3D annotation (như CVAT) thông qua việc kết hợp đồng thời:
     - **Top view (BEV):** Định vị chính xác tâm $(x, y)$, bề rộng, chiều dài và góc xoay yaw theo vạch kẻ đường/hướng di chuyển.
     - **Side view & Front view:** Căn chỉnh chiều cao hộp và áp sát đáy cuboid vào mặt đường cục bộ.
     - **Camera 2D đồng bộ (Sensor Fusion):** Xác nhận chính xác phân lớp đối tượng (ví dụ: phân biệt xe tải vs xe bus, người đi bộ mang balo vs người đi xe đạp) mà đám mây điểm thưa thớt không thể hiện rõ.
