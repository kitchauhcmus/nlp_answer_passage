## NLP - Passage Retrieval

### Tổng quan
Cho một **kho 20.000 đoạn văn** lấy từ Wikipedia tiếng Việt và một **câu hỏi**, hệ thống phải tìm ra đoạn văn chứa câu trả lời[cite: 22]. Đây là bước truy xuất (retrieval) của một hệ thống hỏi đáp: thí sinh không cần trích câu trả lời, chỉ cần xếp đúng đoạn văn lên đầu danh sách[cite: 22].

Dữ liệu của bài gồm[cite: 22]:
* `corpus.csv`: 20.000 đoạn văn; mỗi đoạn có tiêu đề bài viết và nội dung, dài trung bình khoảng 100 âm tiết[cite: 22].
* 3.995 cặp (câu hỏi, đoạn đúng) để train và 503 cặp để dev[cite: 22].
* 1.508 câu hỏi test cần tìm đoạn; đoạn đúng của các câu hỏi test được ẩn[cite: 22].
* Câu hỏi của train, dev và test lấy từ **các bài viết khác nhau**[cite: 22].

Với mỗi câu hỏi test, thí sinh nộp **10 đoạn** xếp theo thứ tự từ khả năng đúng cao nhất đến thấp nhất[cite: 22]. Đoạn đúng nằm ở hạng càng cao thì điểm càng cao[cite: 22].

### Nhiệm vụ
Ví dụ một câu hỏi và đoạn đúng của nó[cite: 22]:
> **Câu hỏi:** Thành phố Miaoli nằm ở quốc gia nào?[cite: 22]
>
> **Đoạn đúng** (`title` = "Miêu Lật (thành phố)"): Thành phố Miêu Lật (tiếng Trung: 苗栗市, Bính âm: Miáolì Shì...) là huyện lỵ của Huyện Miêu Lật, Đài Loan. Từ Miêu Lật là kết hợp của hai từ trong tiếng Khách Gia...[cite: 22]

Mỗi câu hỏi có **đúng một đoạn được tính là đúng**: đó là đoạn văn mà người đặt câu hỏi đã đọc khi viết câu hỏi[cite: 22]. Hai đặc điểm của dữ liệu cần chú ý khi đọc đề[cite: 22]:
* Câu hỏi thường **diễn đạt khác** với đoạn văn[cite: 22]. Trong ví dụ trên, câu hỏi viết "Miaoli" và hỏi "quốc gia nào", còn đoạn văn viết "Miêu Lật" và "Đài Loan"[cite: 22].
* Kho có chứa **các đoạn khác của cùng bài viết** với đoạn đúng[cite: 22]. Những đoạn này có cùng chủ đề và cùng tên riêng với đoạn đúng, nhưng không được tính điểm[cite: 22].

### Phương pháp đánh giá

**Điểm của một câu hỏi**
Điểm của một câu hỏi là **nghịch đảo thứ hạng** của đoạn đúng trong 10 đoạn đã nộp:
```text
QueryScore = 1 / hạng của đoạn đúng    (nếu đoạn đúng nằm ở hạng 1 đến 10)
QueryScore = 0                         (nếu đoạn đúng không nằm trong 10 đoạn)
```
### Cấu trúc thư mục dữ liệu
public/
|-- corpus.csv               (20.000 đoạn: pid, title, text)
|-- train.csv                (3.995 cặp câu hỏi và đoạn đúng)
|-- dev.csv                  (503 cặp)
|-- test.csv                 (1.508 câu hỏi)
|-- sample_submission.csv
|-- scorer.py                (trình chấm chạy tại chỗ)
`-- README.md

# Ý tưởng thực hiện

**Tóm tắt ý tưởng:** Tối ưu Recall với kiến trúc truy xuất lai (BM25 & Bi-Encoder), tích hợp kết quả bằng RRF, cực đại hóa Precision bằng Cross-Encoder 


## Luồng kiến trúc hệ thống

### 1. Tiền xử lý và tích hợp ngữ cảnh
Dữ liệu văn bản thô thường tồn tại nhiễu và phân mảnh thông tin. Giai đoạn tiền xử lý tiến hành chuẩn hóa và tích hợp không gian ngữ cảnh:
* **Tích hợp Tiêu đề - Nội dung:** Tiêu đề tài liệu thường bao hàm các thông tin cô đọng và có giá trị phân loại cao. Việc hợp nhất tiêu đề vào nội dung văn bản giúp gia tăng mật độ đặc trưng và làm phong phú không gian biểu diễn, qua đó nâng cao chất lượng đầu vào cho các mô hình mã hóa ở giai đoạn sau.
* **Chuẩn hóa Định dạng Đầu vào (Input Formatting & Tokenization):** Hệ thống thiết lập các luồng xử lý định dạng chuyên biệt nhằm tối ưu hóa cho từng cấu trúc truy xuất:
  * *Đối với luồng từ vựng (BM25):* Áp dụng biểu thức chính quy để loại bỏ nhiễu (dấu câu, ký tự đặc biệt), chuyển đổi toàn bộ về dạng chữ thường và trích xuất danh sách token thuần túy.
  * *Đối với luồng ngữ nghĩa (Dense Model):* Bổ sung các tiền tố chỉ dẫn như `"passage: "` và `"query: "` vào dữ liệu nhằm kích hoạt chuẩn xác không gian nhúng của kiến trúc mạng E5.
### 2. Truy xuất bằng BM25 & Dense
Nhằm khắc phục những giới hạn của các phương pháp truy xuất đơn lẻ, hệ thống vận hành song song hai nhánh trích xuất độc lập nhằm cực đại hóa độ phủ (Recall) trên toàn bộ tập dữ liệu (Corpus):

* **Nhánh Truy xuất Thưa (Sparse/Lexical Retrieval) - BM25:** Kế thừa và tối ưu hóa từ TF-IDF, thuật toán hoạt động dựa trên cơ chế đối khớp từ vựng chính xác (Lexical Matching). Phương pháp này thể hiện hiệu năng vượt trội khi truy xuất các thực thể định danh, mã số đặc thù hoặc thuật ngữ chuyên ngành hiếm gặp. Tuy nhiên, giới hạn của BM25 là sự suy giảm hiệu suất nghiêm trọng khi truy vấn và tài liệu sử dụng từ đồng nghĩa nhưng không trùng khớp về mặt ký tự định dạng.
* **Nhánh Truy xuất Dày (Dense/Semantic Retrieval) - Bi-Encoder (`multilingual-e5-base`):** Bằng cách ánh xạ văn bản vào không gian vector đa chiều (Embeddings), mô hình thực hiện đánh giá độ tương đồng dựa trên ngữ cảnh (Semantic Matching). Đặc tính này giúp hệ thống khắc phục triệt để điểm yếu của BM25 trong việc xử lý hiện tượng từ đồng nghĩa và đa nghĩa. Mặc dù vậy, do cấu trúc biểu diễn vector thường kém nhạy bén với các định danh ID hoặc từ khóa rời rạc, việc kết hợp Dense Model và BM25 tạo ra một cơ chế bù trừ hoàn hảo, đảm bảo không bỏ sót bất kỳ thông tin trọng yếu nào.


### 3. Dung hợp điểm số và lọc kết quả (RRF)
Hệ thống triển khai thuật toán **Reciprocal Rank Fusion (RRF)** trên tập kết quả sơ cấp:

* **Trích xuất và hợp nhất:** Đối với mỗi truy vấn, hệ thống tiến hành truy xuất độc lập Top 50 tài liệu dẫn đầu từ nhánh BM25 và Top 50 tài liệu từ nhánh Dense Model. Thuật toán RRF sau đó được kích hoạt để dung hợp hai danh sách rời rạc này thành một không gian kết quả thống nhất.
* **Định lượng qua nghịch đảo thứ hạng:** Thay vì sử dụng điểm số nguyên bản vốn không cùng hệ quy chiếu, RRF tính toán điểm số mới cho mỗi tài liệu dựa trên nghịch đảo vị trí xếp hạng của nó trong từng danh sách. Cơ chế này giúp triệt tiêu hoàn toàn sự chênh lệch về thang điểm giữa các mô hình.
* **Cộng hưởng thứ hạng:** Những tài liệu xuất hiện ở thứ hạng cao trong cả hai danh sách Top 50 sẽ được cộng dồn trọng số và đẩy lên vị trí dẫn đầu. Kết thúc quá trình dung hợp, thuật toán lọc và giữ lại đúng 50 kết quả tốt nhất. 

### 4. Tái xếp hạng (Re-ranking)
Tập kết quả từ bước dung hợp tiếp tục được đưa vào giai đoạn đánh giá bằng cấu trúc **Cross-Encoder**:
* Khác biệt với cấu trúc Bi-Encoder (chỉ so sánh khoảng cách giữa hai vector độc lập), mô hình Cross-Encoder thực hiện nối ghép trực tiếp truy vấn và từng tài liệu thành một chuỗi duy nhất trước khi đưa qua mạng nơ-ron sâu.
* Dựa trên cơ chế tự chú ý chéo ở cấp độ token, mọi thành phần trong truy vấn đều có khả năng tương tác trực tiếp với các thành phần trong tài liệu qua nhiều tầng ẩn (Hidden Layers). Cấu trúc này cho phép mô hình nắm bắt các quan hệ ngữ cảnh phức tạp và cung cấp điểm số liên quan.
* Việc giới hạn phạm vi suy luận của Cross-Encoder chỉ trên tập Top-K kết quả giúp tối ưu hóa khối lượng tính toán.
