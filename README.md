# 🚀 Kiến trúc Truy xuất Thông tin Lai (Robust Hybrid Retrieval Pipeline)

Dự án này đề xuất và triển khai một đường ống truy xuất thông tin hai giai đoạn (Two-Stage Retrieval Pipeline) kết hợp phương pháp đối sánh từ vựng (Lexical Matching) và đối sánh ngữ nghĩa (Semantic Matching). Kiến trúc được thiết kế tối ưu cho các bài toán NLP như Hỏi đáp tự động (QA) và Tạo văn bản tăng cường truy xuất (RAG), nhằm tối đa hóa cả độ bao phủ (Recall) lẫn độ chính xác (Precision).

## 🧠 Luồng Kiến trúc Hệ thống (System Architecture Flow)

Đường ống dữ liệu được thiết kế tuyến tính, xử lý thông tin thông qua 4 giai đoạn cốt lõi:

### 1. Tiền xử lý và Làm giàu ngữ cảnh (Context Enrichment)
Dữ liệu thô hiếm khi đạt trạng thái tối ưu cho các bộ mã hóa (Encoders). Giai đoạn tiền xử lý thực hiện việc chuẩn hóa và hợp nhất không gian thông tin:
* **Hợp nhất Tiêu đề - Nội dung (Title-Text Concatenation):** Tiêu đề tài liệu mang mật độ ngữ nghĩa rất cao. Việc nối trực tiếp tiêu đề vào trước nội dung văn bản giúp gia tăng trọng số của các thực thể bổ nghĩa, tối ưu hóa không gian biểu diễn đặc trưng (feature representation) trước khi đưa vào các mô hình tìm kiếm.

### 2. Truy xuất Giai đoạn một (First-Stage Retrieval: Dual-Stream)
Để vượt qua giới hạn của từng phương pháp truy xuất đơn lẻ, hệ thống vận hành song song hai luồng trinh sát độc lập nhằm tối đa hóa độ bao phủ (Recall) trên toàn bộ kho tài liệu (Corpus):

* **Luồng Truy xuất Từ vựng (Sparse/Lexical Retrieval) - Cốt lõi BM25:**
  * Kế thừa và khắc phục những hạn chế của TF-IDF, thuật toán Okapi BM25 được triển khai để giải quyết bài toán đối sánh từ khóa chính xác (Exact Match). 
  * Bằng cách áp dụng hàm chuẩn hóa độ dài tài liệu (Document Length Normalization) và đường cong bão hòa tần suất (Term Frequency Saturation), BM25 loại bỏ nhiễu từ các tài liệu lặp từ khóa quá mức, đóng vai trò sống còn trong việc truy xuất các danh từ riêng, mã định danh và thuật ngữ chuyên ngành (Out-of-Vocabulary terms).
* **Luồng Truy xuất Ngữ nghĩa (Dense Retrieval) - Bi-Encoder (Multilingual-E5):**
  * Sử dụng kiến trúc Bi-Encoder, mô hình chiếu (map) các truy vấn (query) và tài liệu (passage) vào một không gian vector đa chiều chung (Embedding Space). 
  * Phương pháp này đo lường khoảng cách tương đồng (Cosine Similarity) dựa trên các đặc trưng tiềm ẩn (Latent Semantic Relationships), giúp hệ thống vượt qua rào cản về ranh giới từ vựng để giải quyết xuất sắc các hiện tượng từ đồng nghĩa (Synonymy) và đa nghĩa (Polysemy).

### 3. Hợp nhất Đối sánh (Rank Aggregation - RRF)
Quá trình hợp nhất kết quả từ hai luồng (Sparse và Dense) đối mặt với thách thức lớn về sự bất đồng nhất trong phân phối điểm số (Score Distribution). Để giải quyết vấn đề này, hệ thống áp dụng thuật toán **Reciprocal Rank Fusion (RRF)**:
* RRF loại bỏ hoàn toàn giá trị điểm số thô của các mô hình, chỉ sử dụng nghịch đảo thứ hạng (Rank) để tính toán trọng số hợp nhất.
* Các tài liệu đạt thứ hạng cao ở cả hai luồng sẽ được khuếch đại thứ hạng chung. Thuật toán này đóng vai trò như một bộ lọc (Filter) ổn định, dung hòa đặc tính cốt lõi của cả hai phương pháp để tạo ra một tập hợp $K$ ứng viên (Top-K Candidates) toàn diện nhất.

### 4. Xếp hạng lại Giai đoạn hai (Second-Stage Re-ranking - Cross-Encoder)
Tập ứng viên tinh gọn từ bước RRF tiếp tục được đưa vào giai đoạn đánh giá chuyên sâu (Re-ranking) bằng kiến trúc **Cross-Encoder**:
* Khác với Bi-Encoder (chỉ so sánh hai vector tĩnh), Cross-Encoder thực hiện việc nối chuỗi trực tiếp truy vấn và từng tài liệu ứng viên. 
* Toàn bộ chuỗi nối này được xử lý qua mạng nơ-ron sâu (Transformer). Nhờ cơ chế tự chú ý (Self-Attention) toàn cục, mọi token trong truy vấn đều có khả năng tương tác trực tiếp với mọi token trong tài liệu (Token-level cross-attention) ở tất cả các tầng của mô hình. 
* Cơ chế này giúp mô hình trích xuất được những tương quan ngữ cảnh phức tạp nhất, từ đó đưa ra một điểm số liên quan (Relevance Score) mang độ chính xác tuyệt đối. Việc giới hạn phân tích chỉ trên Top-K ứng viên (thay vì toàn bộ Corpus) tạo ra sự cân bằng hoàn hảo giữa độ chính xác (Precision) và chi phí tính toán (Computational Cost).

---
**Tóm tắt đầu ra:** Trải qua luồng xử lý nghiêm ngặt từ BM25/E5 (tối ưu Recall), trộn hạng RRF (tối ưu tính đa dạng) và Cross-Encoder (tối ưu Precision), hệ thống trích xuất và trả về danh sách các tài liệu liên quan nhất theo thứ tự điểm số giảm dần, sẵn sàng tích hợp vào các 파ipeline RAG hạ nguồn.
