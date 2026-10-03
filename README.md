# Kiến Trúc Hệ Thống Truy Xuất Thông Tin Lai (Robust Hybrid Retrieval Framework)

Dự án này triển khai một chu trình truy xuất thông tin hai pha (Two-Stage Retrieval Pipeline), kết hợp phương pháp đối khớp từ vựng (Lexical Matching) và đối khớp ngữ nghĩa (Semantic Matching). Kiến trúc được thiết kế nhằm tối ưu hóa hiệu suất cho các tác vụ Xử lý Ngôn ngữ Tự nhiên (NLP) như Hỏi đáp tự động (Question Answering) và Tạo văn bản tăng cường truy xuất (Retrieval-Augmented Generation - RAG), với mục tiêu cực đại hóa cả độ phủ (Recall) và độ chuẩn xác (Precision).

## 🧠 Luồng Kiến Trúc Hệ Thống (System Architecture Flow)

Chu trình dữ liệu được thiết kế tuyến tính, xử lý thông tin thông qua 4 giai đoạn cốt lõi:

### 1. Tiền xử lý và Tích hợp Ngữ cảnh (Context Enrichment)
Dữ liệu văn bản thô thường tồn tại nhiễu và phân mảnh thông tin. Giai đoạn tiền xử lý tiến hành chuẩn hóa và tích hợp không gian ngữ cảnh:
* **Tích hợp Tiêu đề - Nội dung (Title-Text Concatenation):** Tiêu đề tài liệu thường bao hàm các thông tin cô đọng và có giá trị phân loại cao. Việc hợp nhất tiêu đề vào nội dung văn bản giúp gia tăng mật độ đặc trưng (Feature Density) và làm phong phú không gian biểu diễn, qua đó nâng cao chất lượng đầu vào cho các mô hình mã hóa ở giai đoạn sau.

### 2. Truy xuất Giai đoạn một (First-Stage Retrieval)
Nhằm khắc phục những giới hạn của các phương pháp truy xuất đơn lẻ, hệ thống vận hành song song hai nhánh trích xuất độc lập nhằm cực đại hóa độ phủ (Recall) trên toàn bộ tập dữ liệu (Corpus):

* **Nhánh Truy xuất Thưa (Sparse/Lexical Retrieval) - Thuật toán BM25:**
  * Triển khai hàm tính điểm Okapi BM25 nhằm giải quyết bài toán đối khớp từ vựng (Exact Match). 
  * Thông qua việc tích hợp hệ số chuẩn hóa độ dài tài liệu (Document Length Normalization) và hàm tiệm cận bão hòa tần suất (Term Frequency Saturation), thuật toán kiểm soát hiệu quả hiện tượng nhiễu do lặp từ khóa. Nhánh này đặc biệt tối ưu trong việc trích xuất các danh từ riêng, định danh số học và thuật ngữ ngoại lai (Out-of-Vocabulary Terms).
* **Nhánh Truy xuất Dày (Dense/Semantic Retrieval) - Mô hình Bi-Encoder:**
  * Ứng dụng kiến trúc mã hóa kép (như `multilingual-e5`) để ánh xạ các truy vấn (Queries) và tài liệu (Passages) vào cùng một không gian nhúng liên tục (Continuous Embedding Space). 
  * Độ tương đồng được đo lường thông qua khoảng cách Cosine, cho phép hệ thống đánh giá tính liên kết về mặt ngữ nghĩa tiềm ẩn (Latent Semantic Relatedness), từ đó xử lý triệt để các hiện tượng đồng nghĩa (Synonymy) và đa nghĩa (Polysemy).

### 3. Dung hợp Điểm số (Rank Aggregation - RRF)
Việc kết hợp kết quả từ hai không gian biểu diễn (Sparse và Dense) đặt ra thách thức về sự bất đồng nhất trong phân phối hàm điểm (Score Distribution). Hệ thống áp dụng thuật toán **Reciprocal Rank Fusion (RRF)** để giải quyết vấn đề này:
* Thuật toán RRF loại bỏ sự phụ thuộc vào điểm số nguyên bản của từng mô hình, thay vào đó tính toán trọng số dung hợp dựa trên nghịch đảo thứ hạng (Reciprocal Rank).
* Các tài liệu đạt thứ hạng cao ở cả hai nhánh truy xuất sẽ được gia tăng trọng số tích lũy. Cơ chế này hoạt động như một bộ lọc nhiễu (Noise Filter), đảm bảo tính ổn định và tính đa dạng cho tập ứng viên Top-$K$ (Top-K Candidates) được trích xuất.

### 4. Tái xếp hạng Giai đoạn hai (Second-Stage Re-ranking)
Tập ứng viên từ bước dung hợp tiếp tục được đưa vào giai đoạn đánh giá độ liên quan chuyên sâu bằng cấu trúc **Cross-Encoder**:
* Khác biệt với cấu trúc Bi-Encoder (chỉ so sánh khoảng cách giữa hai vector độc lập), mô hình Cross-Encoder thực hiện nối ghép trực tiếp truy vấn và từng tài liệu ứng viên thành một chuỗi duy nhất trước khi đưa qua mạng nơ-ron sâu (Transformer).
* Dựa trên cơ chế tự chú ý chéo (Cross-Attention) ở cấp độ token, mọi thành phần trong truy vấn đều có khả năng tương tác trực tiếp với các thành phần trong tài liệu qua nhiều tầng ẩn (Hidden Layers). Cấu trúc này cho phép mô hình nắm bắt các quan hệ ngữ cảnh phức tạp và cung cấp điểm số liên quan (Relevance Score) với độ chuẩn xác tối đa.
* Việc giới hạn phạm vi suy luận (Inference) của Cross-Encoder chỉ trên tập Top-$K$ ứng viên giúp tối ưu hóa khối lượng tính toán, tạo ra sự cân bằng thiết yếu giữa chi phí phần cứng (Computational Complexity) và độ chuẩn xác (Precision).

---
**Tóm tắt Luồng Xử lý:** Trải qua quy trình đánh giá đa tầng — từ việc tối ưu Recall với kiến trúc truy xuất lai (BM25 & Bi-Encoder), đa dạng hóa ứng viên bằng RRF, đến việc cực đại hóa Precision bằng Cross-Encoder — hệ thống cung cấp danh sách các tài liệu có độ liên quan cao nhất, được chuẩn hóa cấu trúc để tích hợp trực tiếp vào các mô hình sinh văn bản (Generation Models) ở giai đoạn hạ nguồn (Downstream Tasks).
