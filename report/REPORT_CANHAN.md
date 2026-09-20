# Báo Cáo Cá Nhân — Lab 7: Embedding & Vector Store

**Họ tên:** Mai Văn Trường
**Mã sinh viên:** 2A202602983
**Nhóm:** K4-L3B
**Ngày:** 20/09/2026

> **Nộp 1 bản / sinh viên.** Phần nhóm (lựa chọn tài liệu, thiết kế chiến lược, bộ câu hỏi đánh giá, demo) nộp chung 1 bản trong `REPORT_NHOM.md`. Chi tiết thang điểm: `docs/SCORING.md`.

**Tổng điểm phần cá nhân: 60** = Khởi động (5) + Hướng tiếp cận (10) + Hoàn thiện code (30) + Dự đoán độ tương tự (5) + Kết quả truy xuất của tôi (10).

---

## 1. Khởi động (Warm-up) — Cá nhân (5 điểm)

### Độ tương tự Cosine (Cosine Similarity) (Bài tập 1.1)

**Độ tương tự cosine cao (High cosine similarity) nghĩa là gì?**
> *Độ tương tự Cosine cao (tiệm cận 1.0) nghĩa là hai vector văn bản hướng về cùng một phương trong không gian nhiều chiều, thể hiện rằng hai đoạn văn bản có sự đồng điệu lớn về ngữ nghĩa và ngữ cảnh, bất kể độ dài ngắn khác nhau.*

**Ví dụ có độ tương tự CAO:**
- Câu A: Con mèo đang nằm ngủ trên chiếc ghế sofa mềm mại.
- Câu B: Một chú mèo đang thè lưỡi ngủ khò trên ghế bành.
- Tại sao tương đồng: Cả hai câu đều mô tả cùng một chủ thể (con mèo) và hành động (đang ngủ trên ghế), sử dụng các từ ngữ đồng nghĩa hoặc cùng trường ngữ nghĩa.

**Ví dụ có độ tương tự THẤP:**
- Câu A: Thuật toán nén dữ liệu giúp giảm dung lượng lưu trữ của tệp tin.
- Câu B: Hôm nay thời tiết Hà Nội rất đẹp và thích hợp đi dạo công viên.
- Tại sao khác: Hai câu thuộc hai chủ đề hoàn toàn khác nhau (khoa học máy tính vs thời tiết/sinh hoạt), không shared bất kỳ trường từ vựng hay ý nghĩa ngữ cảnh nào.

**Tại sao độ tương tự cosine (cosine similarity) được ưu tiên hơn khoảng cách Euclid (Euclidean distance) cho text embeddings?**
> *Độ tương tự Cosine được ưu tiên vì nó đo góc giữa hai vector chứ không phụ thuộc vào độ dài (độ lớn/magnitude) của vector. Điều này giúp so sánh chuẩn xác ngữ nghĩa của văn bản ngay cả khi một đoạn văn ngắn và một đoạn văn dài có tần suất xuất hiện từ khác nhau đáng kể.*

### Bài toán tính toán Chunking (Bài tập 1.2)

**Tài liệu 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> *Công thức: số lượng chunk = ceil((độ_dài_tài_liệu - overlap) / (chunk_size - overlap))*
> *Phép tính: ceil((10000 - 50) / (500 - 50)) = ceil(9950 / 450) = ceil(22.111...) = 23*
> *Đáp án: 23 chunks*

**Nếu độ chồng chéo (overlap) tăng lên 100, số lượng chunk thay đổi thế nào? Tại sao muốn độ chồng chéo nhiều hơn?**
> *Phép tính mới: ceil((10000 - 100) / (500 - 100)) = ceil(9900 / 400) = ceil(24.75) = 25 chunks. Số lượng chunk tăng từ 23 lên 25. Muốn tăng độ chồng chéo để tránh mất ngữ cảnh ở ranh giới giữa các chunk, đảm bảo các câu/ý nằm ở điểm giao nhau không bị cắt đứt gãy.*

---

## 2. Hướng tiếp cận của tôi (My Approach) — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi lập trình (implement) các phần chính trong gói `src`.

### Các hàm chia nhỏ (Chunking Functions)

**`SentenceChunker.chunk`** — hướng tiếp cận:
> *Sử dụng regex `r'(?<=[.!?])\s+'` để tách văn bản theo ranh giới câu (dựa trên các dấu câu `.`, `!`, `?` theo sau bởi khoảng trắng). Sau đó nhóm các câu lại thành từng chunk với số lượng câu tối đa không vượt quá `max_sentences_per_chunk` và chuẩn hóa khoảng trắng thừa.*

**`RecursiveChunker.chunk` / `_split`** — hướng tiếp cận:
> *Thực hiện thuật toán đệ quy thử nghiệm phân tách theo thứ tự ưu tiên của danh sách phân cách `["\n\n", "\n", ". ", " ", ""]`. Trường hợp cơ sở là khi văn bản nhỏ hơn `chunk_size` hoặc không còn dấu phân cách thì trả về kết quả; nếu đoạn phân tách vẫn lớn hơn `chunk_size` sẽ tiếp tục gọi đệ quy `_split` với dấu phân cách tiếp theo.*

### Lớp EmbeddingStore

**`add_documents` + `search`** — hướng tiếp cận:
> *Lưu trữ tài liệu dưới dạng danh sách các `dict` record gồm id, content, embedding vector và metadata. Khi tìm kiếm, truy vấn được nhúng thành vector và tính tích vô hướng (dot product) với tất cả các vector trong store, sau đó sắp xếp giảm dần theo điểm score và lấy top_k kết quả.*

**`search_with_filter` + `delete_document`** — hướng tiếp cận:
> *Với `search_with_filter`, thực hiện lọc trước (pre-filtering) các record thỏa mãn chính xác các cặp key-value trong `metadata_filter` rồi mới tính độ tương đồng. Với `delete_document`, lọc loại bỏ tất cả các chunk trong store có `id` hoặc `metadata['doc_id']` trùng với `doc_id` cần xóa.*

### Tác tử KnowledgeBaseAgent

**`answer`** — hướng tiếp cận:
> *Khi nhận câu hỏi, agent gọi `store.search` để lấy `top_k` chunk liên quan nhất, hợp nhất nội dung các chunk này thành văn bản ngữ cảnh (context), sau đó xây dựng prompt RAG chuẩn định dạng và truyền cho hàm `llm_fn` để tạo câu trả lời.*

---

## 3. Hoàn thiện code (Core Implementation) — Cá nhân (30 điểm)

Vượt qua bộ kiểm thử là điều kiện tính điểm phần này.

### Kết Quả Kiểm Thử (Test Results)

```
Ran 42 tests in 0.015s

OK
```

**Số lượng bài test vượt qua (pass):** 42 / 42

---

## 4. Dự đoán độ tương tự (Similarity Predictions) — Cá nhân (5 điểm)

| Cặp | Câu A | Câu B | Dự đoán | Điểm thực tế | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | Chính sách hoàn trả sản phẩm trong 30 ngày | Khách hàng có thể trả lại hàng và nhận tiền hoàn trong vòng một tháng | cao | 0.2842 | Đúng |
| 2 | Quy định bảo hành hàng hóa đối với người bán | Hướng dẫn đăng bán sản phẩm cho người bán mới trên sàn | thấp | 0.0815 | Đúng |
| 3 | Làm thế nào để hủy đơn hàng đã đặt? | Các bước thực hiện hủy đơn hàng trước khi người bán giao cho vận chuyển | cao | 0.3105 | Đúng |
| 4 | Khách hàng không nhận được hàng phải làm sao? | Hướng dẫn nộp yêu cầu hoàn tiền khi không nhận được kiện hàng | cao | 0.2978 | Đúng |
| 5 | Phương thức thanh toán qua thẻ tín dụng và ví điện tử | Cách thức bảo mật tài khoản cá nhân và mật khẩu | thấp | -0.0412 | Đúng |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn ý nghĩa?**
> *Kết quả đáng chú ý nhất là điểm số giữa hai câu đồng nghĩa (như Cặp 1 và Cặp 3) ở khoảng ~0.30 khi dùng mô hình Mock (hoặc các câu khác biệt có điểm âm/gần 0). Điều này cho thấy embeddings biểu diễn ý nghĩa văn bản dưới dạng các góc hình học trong không gian vector nhiều chiều; các câu đồng nghĩa tuy không trùng từ vựng chính xác nhưng có vector chỉ về hướng tương tự nhau, trong khi các từ trái chủ đề sẽ tạo góc lớn hơn (điểm tiệm cận 0 hoặc âm).*

---

## 5. Kết quả truy xuất của tôi (Competition Results) — Cá nhân (10 điểm)

Chạy **5 câu hỏi đánh giá của nhóm** trên mã nguồn cá nhân của bạn trong gói `src`. **5 câu hỏi này phải trùng với các thành viên cùng nhóm** (xem `REPORT_NHOM.md`).

| # | Câu hỏi (Query) | Top-1 Chunk truy xuất được (tóm tắt) | Điểm Score | Có liên quan không? (Relevant) | Câu trả lời của Agent (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | | | | | |
| 2 | | | | | |
| 3 | | | | | |
| 4 | | | | | |
| 5 | | | | | |

**Bao nhiêu câu hỏi trả về chunk có liên quan trong top-3?** __ / 5

**Điều hay nhất tôi học được từ thành viên khác / nhóm khác (qua demo):**
> *Viết 2-3 câu:*

---

## Tự Đánh Giá (Phần Cá Nhân)

| Tiêu chí | Điểm tự đánh giá |
|----------|-------------------|
| Khởi động (Warm-up) | / 5 |
| Hướng tiếp cận của tôi (My Approach) | / 10 |
| Hoàn thiện code (Core Implementation — tests) | / 30 |
| Dự đoán độ tương tự (Similarity Predictions) | / 5 |
| Kết quả truy xuất của tôi (Competition Results) | / 10 |
| **Tổng phần cá nhân** | **/ 60** |
