# Ý tưởng thực hiện

## Luồng kiến túc hệ thống

### 1. Tiền xử lý và tích hợp ngữ cảnh
Dữ liệu văn bản thô thường tồn tại nhiễu và phân mảnh thông tin. Giai đoạn tiền xử lý tiến hành chuẩn hóa và tích hợp không gian ngữ cảnh:
* **Tích hợp Tiêu đề - Nội dung (Title-Text Concatenation):** Tiêu đề tài liệu thường bao hàm các thông tin cô đọng và có giá trị phân loại cao. Việc hợp nhất tiêu đề vào nội dung văn bản giúp gia tăng mật độ đặc trưng và làm phong phú không gian biểu diễn, qua đó nâng cao chất lượng đầu vào cho các mô hình mã hóa ở giai đoạn sau.

### 2. Truy xuất bằng BM25 & Dense
Nhằm khắc phục những giới hạn của các phương pháp truy xuất đơn lẻ, hệ thống vận hành song song hai nhánh trích xuất độc lập nhằm cực đại hóa độ phủ (Recall) trên toàn bộ tập dữ liệu (Corpus):

* BM25 (Kế thừa TF-IDF): Thuật toán này hoạt động dựa trên cơ chế so khớp từ khóa chính xác (Lexical Matching). Nó cực kỳ mạnh khi bạn cần tìm kiếm các mã số đặc thù, tên riêng hiếm gặp, hoặc các thuật ngữ chuyên ngành không thể thay thế. Tuy nhiên, nó sẽ "mù tịt" nếu câu hỏi và tài liệu dùng từ đồng nghĩa (ví dụ: hỏi "xe cộ" nhưng tài liệu ghi "phương tiện giao thông").
* Dense Model (multilingual-e5-base): Việc mã hóa văn bản thành không gian vector (Embeddings) giúp mô hình này hiểu được ngữ cảnh và so khớp ngữ nghĩa (Semantic Matching). Nó giải quyết hoàn hảo điểm yếu từ đồng nghĩa của BM25. Dù vậy, nó không "tốt hơn" một cách tuyệt đối. Các mô hình vector thường kém nhạy bén với những từ khóa chính xác rải rác hoặc các mã ID cụ thể.
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
