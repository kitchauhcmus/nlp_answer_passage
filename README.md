# Ý tưởng thực hiện

## Luồng kiến trúc hệ thống

### 1. Tiền xử lý và tích hợp ngữ cảnh
Dữ liệu văn bản thô thường tồn tại nhiễu và phân mảnh thông tin. Giai đoạn tiền xử lý tiến hành chuẩn hóa và tích hợp không gian ngữ cảnh:
* **Tích hợp Tiêu đề - Nội dung (Title-Text Concatenation):** Tiêu đề tài liệu thường bao hàm các thông tin cô đọng và có giá trị phân loại cao. Việc hợp nhất tiêu đề vào nội dung văn bản giúp gia tăng mật độ đặc trưng và làm phong phú không gian biểu diễn, qua đó nâng cao chất lượng đầu vào cho các mô hình mã hóa ở giai đoạn sau.
* **Chuẩn hóa Định dạng Đầu vào (Input Formatting & Tokenization):** Hệ thống thiết lập các luồng xử lý định dạng chuyên biệt nhằm tối ưu hóa cho từng cấu trúc truy xuất:
  * *Đối với luồng từ vựng (BM25):* Áp dụng biểu thức chính quy (Regular Expression) để loại bỏ nhiễu (dấu câu, ký tự đặc biệt), chuyển đổi toàn bộ về dạng chữ thường (Lower-casing) và trích xuất danh sách token (Tokens) thuần túy.
  * *Đối với luồng ngữ nghĩa (Dense Model):* Bổ sung các tiền tố chỉ dẫn (Instructional Prefixes) như `"passage: "` và `"query: "` vào dữ liệu nhằm kích hoạt chuẩn xác không gian nhúng của kiến trúc mạng E5.
### 2. Truy xuất Giai đoạn một (First-Stage Retrieval: BM25 & Dense)
Nhằm khắc phục những giới hạn của các phương pháp truy xuất đơn lẻ, hệ thống vận hành song song hai nhánh trích xuất độc lập nhằm cực đại hóa độ phủ (Recall) trên toàn bộ tập dữ liệu (Corpus):

* **Nhánh Truy xuất Thưa (Sparse/Lexical Retrieval) - BM25:** Kế thừa và tối ưu hóa từ TF-IDF, thuật toán hoạt động dựa trên cơ chế đối khớp từ vựng chính xác (Lexical Matching). Phương pháp này thể hiện hiệu năng vượt trội khi truy xuất các thực thể định danh, mã số đặc thù hoặc thuật ngữ chuyên ngành hiếm gặp (Out-of-Vocabulary terms). Tuy nhiên, giới hạn của BM25 là sự suy giảm hiệu suất nghiêm trọng khi truy vấn và tài liệu sử dụng từ đồng nghĩa nhưng không trùng khớp về mặt ký tự định dạng.
* **Nhánh Truy xuất Dày (Dense/Semantic Retrieval) - Bi-Encoder (`multilingual-e5-base`):** Bằng cách ánh xạ văn bản vào không gian vector đa chiều (Embeddings), mô hình thực hiện đánh giá độ tương đồng dựa trên ngữ cảnh (Semantic Matching). Đặc tính này giúp hệ thống khắc phục triệt để điểm yếu của BM25 trong việc xử lý hiện tượng từ đồng nghĩa (Synonymy) và đa nghĩa (Polysemy). Mặc dù vậy, do cấu trúc biểu diễn vector thường kém nhạy bén với các định danh ID hoặc từ khóa rời rạc, việc kết hợp Dense Model và BM25 tạo ra một cơ chế bù trừ hoàn hảo, đảm bảo không bỏ sót bất kỳ thông tin trọng yếu nào.


### 3. Dung hợp điểm số (RRF)
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
