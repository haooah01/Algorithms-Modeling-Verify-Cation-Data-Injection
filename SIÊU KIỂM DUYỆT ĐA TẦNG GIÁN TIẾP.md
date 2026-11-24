# KIẾN TRÚC HỆ THỐNG SIÊU KIỂM DUYỆT ĐA TẦNG GIÁN TIẾP VÀ CHIẾN LƯỢC TỐI ƯU HÓA CHI PHÍ QUY MÔ LỚN

## PHẦN I: TỔNG QUAN VỀ KIẾN TRÚC ĐỐI CHIẾU GIÁN TIẾP

Để xây dựng một hệ thống kiểm duyệt hoạt động theo cơ chế đối chiếu gián tiếp (Zero-Knowledge / Privacy-Preserving Moderation), hệ thống loại bỏ hoàn toàn việc lưu trữ và so sánh dữ liệu thô (raw data). Thay vào đó, toàn bộ luồng xử lý vận hành dựa trên toán học không gian vector, mật mã học nhận thức và đồ thị hành vi.

Hệ thống được cấu trúc thành **4 lớp xử lý khép kín** và **1 vòng lặp tự tiến hóa**, chuyển hóa mọi định dạng dữ liệu (văn bản, hình ảnh, video, âm thanh) thành các biểu diễn toán học bất khả nghịch trước khi đưa vào phễu lọc.

---

## PHẦN II: CHI TIẾT CÁC KHÂU TRONG LUỒNG XỬ LÝ (PIPELINE)

### Khâu 1: Tiền xử lý & Trích xuất đặc trưng gián tiếp (Feature Extraction Engine)
Mục tiêu của khâu này là biến dữ liệu thô thành các "mã định danh toán học" không thể dịch ngược về dữ liệu gốc (One-way transformation), đảm bảo tính gián tiếp và bảo mật quyền riêng tư.

1. **Đối với Hình ảnh & Video — Thuật toán Băm nhận thức (Perceptual Hashing - PDQ / TMK+PDQF):**
   * *Cơ chế bên trong:* Khác với băm mật mã truyền thống (như SHA-256) sẽ thay đổi hoàn toàn nếu chỉ sửa 1 pixel, thuật toán PDQ chuyển ảnh sang không gian màu Luminance, áp dụng làm mờ Gaussian (Gaussian blur) để loại bỏ nhiễu, sau đó thực hiện Biến đổi Cosin Rời rạc (DCT - Discrete Cosine Transform) để trích xuất các tần số thấp (cấu trúc cốt lõi của ảnh).
   * *Cách thức kiểm duyệt:* Tạo ra một chuỗi nhị phân 256-bit. Nếu người dùng lật ảnh, xoay góc, chỉnh bộ lọc màu, hoặc chèn sticker, cấu trúc tần số thấp không thay đổi nên chuỗi 256-bit vẫn giữ nguyên phần lớn giá trị. Đối với video, thuật toán TMK gom các khung hình (frames) theo trục thời gian để tạo ra một vector đại diện cho toàn bộ đoạn video.

2. **Đối với Văn bản & Âm thanh — Chuyển đổi Không gian Ngữ nghĩa (Dense Vector Embeddings):**
   * *Cơ chế bên trong:* Văn bản (hoặc âm thanh đã qua lớp chuyển đổi Speech-to-Text tốc độ cao) được đưa qua mạng nơ-ron kiến trúc Transformer (ví dụ: BGE-M3 hoặc E5-Mistral). Mô hình ánh xạ ý nghĩa của câu thành một mảng số thực (Dense Vector) gồm $1024$ chiều.
   * *Cách thức kiểm duyệt:* Trong không gian $1024$ chiều, các từ lóng, từ viết lái, ký tự đặc biệt thay thế, hoặc các câu mang tính kích động diễn đạt vòng vo đều bị kéo về cùng một vùng tọa độ. Hệ thống chỉ lưu tọa độ toán học này và xóa bỏ hoàn toàn văn bản gốc.

3. **Đối với Hành vi & Ngữ cảnh — Trích xuất Đồ thị Tương tác (Graph Metadata):**
   * *Cơ chế bên trong:* Trích xuất tần suất đăng bài, tốc độ lan truyền, mối liên kết giữa các tài khoản và chuyển thành các vector đặc trưng thông qua thuật toán GraphSAGE.
   * *Cách thức kiểm duyệt:* Phát hiện các chiến dịch tấn công có chủ đích (coordinated attacks) hoặc mạng lưới spam tự động ngay cả khi nội dung từng bài viết riêng lẻ trông có vẻ vô hại.

---

### Khâu 2: Tầng Lọc Tốc Độ Cao (Tầng 1 — Deterministic & Fuzzy Hashing)
Tầng này xử lý khoảng **80% lưu lượng** trong thời gian dưới $5\text{ ms}$, chặn đứng các nội dung vi phạm đã từng xuất hiện hoặc bị biến tấu nhẹ.

* **Cấu trúc lưu trữ:** Sử dụng cấu trúc dữ liệu Multi-Index Hashing (MIH) kết hợp Bloom Filter trên bộ nhớ RAM (Redis/RocksDB) để tra cứu hàng tỷ mã hash với độ trễ cực thấp.
* **Toán học đối chiếu:** Sử dụng **Khoảng cách Hamming (Hamming Distance)** để đếm số lượng bit khác nhau giữa chuỗi 256-bit của dữ liệu mới và kho dữ liệu vi phạm:
  $$d_H(x, y) = \sum_{i=1}^{256} |x_i - y_i|$$
* **Ngưỡng quyết định:**
  * $d_H \le 15\text{ bit}$ (tương đồng $> 94\%$): **Chặn tự động (Hard Block)**.
  * $16\text{ bit} \le d_H \le 35\text{ bit}$: Nghi ngờ biến thể phức tạp, đẩy sang Tầng 2.

---

### Khâu 3: Tầng Đối Chiếu Ngữ Nghĩa Không Gian (Tầng 2 — Vector Similarity & Clustering)
Tầng này xử lý **15% lưu lượng** tiếp theo (dữ liệu mới chưa từng có trong kho hash), thời gian xử lý khoảng $20 - 50\text{ ms}$.

* **Cấu trúc lưu trữ:** Cơ sở dữ liệu Vector chuyên dụng (Qdrant, Milvus) sử dụng thuật toán lập chỉ mục đồ thị **HNSW (Hierarchical Navigable Small World)** để tìm kiếm hàng xóm gần nhất (ANN).
* **Toán học đối chiếu:** Kho dữ liệu chia sẵn các vi phạm thành hàng ngàn "Cụm độc hại" (Toxic Clusters). Hệ thống tính **Độ tương đồng Cosin (Cosine Similarity)** giữa vector đầu vào $\mathbf{A}$ và tâm của các cụm vi phạm $\mathbf{B}$:
  $$\text{Similarity}(\mathbf{A}, \mathbf{B}) = \frac{\mathbf{A} \cdot \mathbf{B}}{\|\mathbf{A}\| \|\mathbf{B}\|}$$
* **Ngưỡng quyết định:**
  * $\text{Similarity} > 0.88$: Nội dung mang ý nghĩa cực kỳ sát với mẫu vi phạm $\rightarrow$ Chặn tự động hoặc ẩn tạm thời (Shadowban).
  * $0.72 \le \text{Similarity} \le 0.88$: Rơi vào **"Vùng xám" (Ambiguous Zone)** $\rightarrow$ Đẩy sang Tầng 3.

---

### Khâu 4: Tầng Trọng Tài Ngữ Cảnh & Suy Luận (Tầng 3 — Cross-Modal & SLM Arbitration)
Tầng này chỉ xử lý **5% lưu lượng** khó nhất (vùng xám) và kết hợp đa phương thức, thời gian xử lý khoảng $200 - 500\text{ ms}$.

* **Cơ chế đối chiếu chéo (Cross-Modal Alignment):** Sử dụng kiến trúc CLIP / SigLIP. Hệ thống nhân ma trận vector của Hình ảnh với vector của Văn bản đi kèm. Nếu tích vô hướng rơi vào vùng nguy hiểm (ví dụ: ảnh vật dụng bình thường kết hợp văn bản ẩn ý nguy hiểm), hệ thống sẽ phát hiện vi phạm ngay lập tức.
* **Mô hình Trọng tài (Small Language Model - SLM):** Sử dụng mô hình ngôn ngữ chuyên biệt được tinh chỉnh cho an toàn nội dung (như Llama-Guard-3-8B). SLM nhận báo cáo đặc trưng gián tiếp từ Tầng 1, Tầng 2, kết hợp điểm uy tín tài khoản (Trust Score) để xuất ra xác suất vi phạm theo từng danh mục chính sách.

---

### Khâu 5: Vòng Lặp Tự Tiến Hóa (Active Learning & Zero-Day Immunization)
Cơ chế giúp hệ thống tự miễn dịch với các chiêu trò lách luật mới (Zero-day violations):

1. **Bẫy mật (Honeypot Vectors):** Khi Tầng 3 hoặc kiểm duyệt viên xác nhận một biến thể lách luật mới, hệ thống trích xuất ngay mã PDQ và Vector $1024$ chiều của nội dung đó.
2. **Tiêm vắc-xin toán học:** Sử dụng thuật toán Data Augmentation để tự động tạo ra $100$ biến thể vector xung quanh tọa độ gốc bằng cách thêm nhiễu phân phối chuẩn (Gaussian noise).
3. **Cập nhật tức thì:** Toàn bộ $100$ vector này được nạp thẳng vào chỉ mục HNSW của Tầng 2, giúp chặn đứng mọi biến thể tương tự trên toàn hệ thống chỉ sau vài giây.

---

## PHẦN III: CHIẾN LƯỢC TỐI ƯU HÓA PHẦN CỨNG & CHI PHÍ QUY MÔ LỚN

Khi vận hành ở quy mô hàng chục triệu đến hàng tỷ lượt mỗi ngày, các kỹ thuật tối ưu hóa cấp độ phần cứng và toán học sau giúp giảm từ **80% đến 90% chi phí hạ tầng**:

### 1. Tối ưu hóa Khâu Trích xuất & Tầng 3 (Giảm tải GPU)
* **Lượng tử hóa mô hình (Model Quantization - INT8 / FP8 / AWQ):** Ép các mô hình Embedding và SLM từ chuẩn $32\text{-bit}/16\text{-bit}$ xuống $8\text{-bit}$ hoặc $4\text{-bit}$. Giảm $75\%$ dung lượng VRAM cần thiết và tăng tốc độ suy luận gấp $2 - 3$ lần với độ hao hụt chính xác dưới $1\%$.
* **Chuyển dịch tác vụ nhẹ về CPU (ONNX Runtime / OpenVINO):** Biên dịch các thuật toán băm ảnh (PDQ) và mô hình ngôn ngữ siêu nhỏ sang định dạng ONNX, chạy bằng tập lệnh tăng tốc phần cứng AVX-512 trên CPU đời mới (AMD EPYC / Intel Xeon).
* **Gom lô động (Dynamic Batching - NVIDIA Triton):** Gom các yêu cầu đến trong vòng $5 - 10\text{ ms}$ thành một mẻ lớn (Batch) trước khi đẩy vào GPU, giúp nâng hiệu suất sử dụng GPU từ $20\%$ lên $90\%$.

### 2. Tối ưu hóa Tầng 2: Nén Vector giảm 95% chi phí RAM
Lưu $1\text{ tỷ}$ vector $1024$ chiều ở định dạng gốc ($\text{Float32}$) tiêu tốn khoảng $4\text{ TB}$ RAM. Giải pháp thay thế:
* **Lượng tử hóa nhị phân (Binary Quantization - BQ):** Chuyển đổi mỗi số thực $32\text{-bit}$ thành $1\text{ bit}$ duy nhất (dương thành $1$, âm thành $0$). Dung lượng RAM giảm $32$ lần (từ $4\text{ TB}$ xuống còn $128\text{ GB}$). Phép tính Cosin biến thành phép toán bit `XOR` và `POPCNT` chạy trực tiếp trên thanh ghi CPU nhanh gấp $20 - 40$ lần.
* **Cơ chế Rescoring (Tìm kiếm 2 bước):** Dùng Vector nén nhị phân trên RAM để lọc nhanh Top $50$ kết quả gần nhất trong $2\text{ ms}$, sau đó mới đọc Vector gốc từ ổ cứng NVMe SSD lên để tính điểm lại chính xác cho $50$ kết quả này.

### 3. Tối ưu hóa Kiến trúc Lưu trữ & Luồng Dữ liệu
* **Bộ nhớ đệm ngữ nghĩa (Semantic Cache):** Đặt lớp Redis Cache trước toàn bộ hệ thống. Nếu mã hash đầu vào đã được kiểm duyệt trong $24\text{ giờ}$ qua, trả về kết quả ngay lập tức mà không chạy lại AI.
* **Định tuyến Đồng bộ & Bất đồng bộ (Sync vs. Async Routing):**
  * *Luồng đồng bộ (Sync):* Áp dụng cho Tầng 1 và Tầng 2 (tổng thời gian $< 15\text{ ms}$) để chặn trước khi hiển thị mà không gây trễ trải nghiệm người dùng.
  * *Luồng bất đồng bộ (Async):* Đẩy các nội dung "vùng xám" (Tầng 3) và video nặng vào hàng đợi thông điệp (Apache Kafka) để xử lý ngầm, tránh phải mua dư thừa GPU cho các khung giờ cao điểm.

### 4. Bảng So Sánh Hiệu Quả Chi Phí (Ước tính cho 100 triệu lượt/ngày)

| Thành phần hạ tầng | Kiến trúc thông thường | Kiến trúc tối ưu sâu | Mức tiết kiệm |
| :--- | :--- | :--- | :--- |
| **Lưu trữ 1 tỷ Vector** | $4\text{ TB}$ RAM chuyên dụng | $128\text{ GB}$ RAM + NVMe SSD (Binary Quantization) | **~90% chi phí RAM** |
| **Tính toán Embedding** | $8\times$ GPU NVIDIA A100 | CPU AMD EPYC (ONNX + AVX-512) | **~85% chi phí tính toán** |
| **Vận hành SLM Tầng 3** | $4\times$ GPU A100 (FP16) | $1\times$ GPU L40S (INT8 + Dynamic Batching) | **~75% chi phí GPU** |