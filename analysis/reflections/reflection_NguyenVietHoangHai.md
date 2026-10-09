# Individual Reflection — Lab 18: Production RAG

**Họ và tên:** Nguyễn Việt Hoàng Hải  
**MSSV:** 2A202602967  
**Khóa:** K4 - Track 3A  
**Ngày hoàn thành:** 09/10/2026  

---

## Phần 1: Mapping bài giảng (Lecture Mapping)

Map từng concept cốt lõi trong lecture vào code thực tế trong 5 modules của bài lab:

| Lecture Concept | Module | Hàm cụ thể | Observation & Phân tích chuyên sâu |
|----------------|--------|-------------|------------------------------------|
| **Semantic & Hierarchical Chunking** | M1 Chunking | `chunk_semantic()`, `chunk_hierarchical()`, `chunk_structure_aware()` | Khi dùng basic chunking (cắt cứng 500 ký tự theo đoạn văn), nhiều điều khoản chính sách bị ngắt giữa chừng làm mất chủ ngữ. `chunk_semantic()` với ngưỡng cosine 0.85 gom các câu cùng mạch ý vào cùng một chunk. `chunk_hierarchical()` giải quyết mâu thuẫn giữa kích thước chunk khi tìm kiếm và khi sinh text: retrieve child nhỏ (256 chars) cho precision cao, nhưng trả về parent (2048 chars) để LLM có đầy đủ ngữ cảnh trả lời. `chunk_structure_aware()` giữ nguyên tiêu đề Markdown và cấu trúc bảng hạn mức tài chính. |
| **Hybrid Search (BM25 + Dense Fusion)** | M2 Search | `BM25Search.search()`, `DenseSearch.search()`, `reciprocal_rank_fusion()` | Dense embedding (BGE-M3) rất mạnh về hiểu ngữ nghĩa nhưng thường bỏ sót các mã hiệu, số hiệu ngày tháng chính xác (như `12 ngày`, `120 ngày`, `WireGuard`). BM25 Okapi xử lý tiếng Việt qua `underthesea` (tách từ và thay `_` bằng khoảng trắng) bù đắp lỗ hổng từ khóa chính xác. Thuật toán RRF ($k=60$) kết hợp thứ hạng mượt mà không phụ thuộc vào scale điểm số khác nhau của 2 phương pháp. |
| **Cross-Encoder Reranking** | M3 Rerank | `CrossEncoderReranker.rerank()` | Bi-encoder chỉ xem xét riêng biệt query và document embedding nên dễ bỏ sót tương tác ngữ nghĩa tinh tế giữa câu hỏi và tài liệu. Mô hình Cross-Encoder `BAAI/bge-reranker-v2-m3` đánh giá tương tác full self-attention giữa cặp `(query, document)`, xếp lại top-20 ứng viên từ Hybrid Search và chỉ chọn ra top-3 chuẩn xác nhất. Kết quả: Context Precision nhảy vọt từ 0.9091 lên 0.9792, loại bỏ hầu hết các chunk nhiễu. |
| **RAGAS 4 Metrics & Failure Analysis** | M4 Eval | `evaluate_ragas()`, `failure_analysis()` | Đánh giá RAG định lượng thay vì trực giác: 4 chỉ số chia thành 2 trục độc lập: Trục Retrieval (Context Precision 0.9792, Context Recall 0.9450) và Trục Generation (Faithfulness 0.8850, Answer Relevancy 0.8620). Hàm `failure_analysis()` tự động ánh xạ qua Diagnostic Tree giúp định vị lỗi phát sinh ở khâu nào (LLM ảo giác, cắt chunk lỗi hay câu trả lời chưa đúng trọng tâm). |
| **Document & Chunk Enrichment** | M5 Enrichment | `_enrich_single_call()`, `enrich_chunks()` | Triển khai chế độ Combined Single-Call (1 LLM call/chunk) tạo đồng thời: Summary, Hypothesis Questions (HyQA), Contextual Prepend (Anthropic-style) và Auto Metadata. Việc prepend tiêu đề tài liệu và tóm tắt vị trí chunk vào đầu văn bản giúp chunk mang đầy đủ ngữ cảnh độc lập ngay cả khi đứng tách rời trong vector database. |

---

## Phần 2: Khó khăn & Cách giải quyết (Challenges & Debugging)

### 1. Lỗi kỹ thuật gặp phải (Exact error messages)

- **Lỗi 1 (Cross-Encoder / Transformers compatibility & Reload Latency):**
  - *Lỗi ban đầu:* Quá trình chạy test `tests/test_m3.py` bị chậm do mỗi lần gọi `CrossEncoderReranker()` trong từng test case lại load lại toàn bộ mô hình `bge-reranker-v2-m3` nặng 2.2GB vào RAM, dẫn đến test suite bị timeout 120s.
  - *Lỗi API rate limit:* `APIStatusError: Error code: 402 - {'error': {'message': 'This request would exceed your available credits given your current in-flight requests...'}}` khi chạy đánh giá RAGAS đồng thời nhiều luồng trên OpenRouter.

### 2. Nguyên nhân gốc rễ & Cách debug

- **Xử lý tốc độ load mô hình:**
  - Khởi tạo cơ chế module-level cache `_CROSS_ENCODER_CACHE` và `_DENSE_ENCODER`. Chỉ nạp trọng số mô hình một lần duy nhất vào bộ nhớ khi lần đầu được gọi. Các lần khởi tạo `CrossEncoderReranker()` hoặc `DenseSearch()` tiếp theo tái sử dụng instance đã nạp sẵn, giảm thời gian thực thi của test từ hơn 2 phút xuống dưới 20 giây.
- **Xử lý Rate Limit & Concurrent Enrichment:**
  - Trong `src/m5_enrichment.py`, kiểm soát `ThreadPoolExecutor(max_workers=8)` để gửi request song song vừa đủ, tránh bị nghẽn và không vi phạm trần request in-flight.
  - Đối với RAGAS eval, thêm xử lý an toàn chuyển đổi `NaN` thành giá trị hợp lệ `0.0` và bọc try-except fallback đảm bảo script luôn hoàn thành mà không crash pipeline giữa chừng.

### 3. Kiến thức còn thiếu & Cách khắc phục

- **Kiến thức:** Cách hoạt động của BM25 đối với tiếng Việt có dấu và từ ghép.
- **Bổ sung:** Đã nghiên cứu cơ chế tokenization của `underthesea` và `rank-bm25`. Nhận ra việc `underthesea` gán dấu gạch dưới `_` cho từ ghép (ví dụ: `nghỉ_phép`) nếu không xử lý đồng nhất giữa corpus và query sẽ khiến BM25 tra cứu trượt hoàn toàn. Do đó, chuẩn hóa `replace("_", " ")` trước khi split token là bước tiền xử lý bắt buộc.

---

## Phần 3: Action Plan cho Project cá nhân (Application Plan)

### Project: Hệ thống AI Trợ lý Tra cứu Văn bản Pháp luật & Hợp đồng Thương mại (Legal-RAG Assistant)

#### 1. Hiện trạng
- **Pipeline hiện tại:** Sử dụng Naive RAG cơ bản với RecursiveCharacterTextSplitter (chunk size 1000, overlap 200), lưu trữ ChromaDB và tìm kiếm Dense Embedding đơn thuần bằng OpenAI `text-embedding-3-small`.
- **Vấn đề / Bottlenecks đang gặp:**
  1. *Đứt gãy điều khoản:* Các điều khoản hợp đồng hoặc văn bản luật có cấu trúc Điều/Khoản/Điểm thường bị cắt ngang giữa chừng, mất ngữ cảnh của Điều cha.
  2. *Tra cứu số hiệu văn bản kém:* Người dùng hỏi số hiệu nghị định/thông tư (ví dụ: "Nghị định 13/2023/NĐ-CP") thì Dense Search trả về kết quả mờ nhạt do biểu diễn vector không phân biệt rõ các ký tự số hiệu.
  3. *Ảo giác do xung đột phiên bản:* Văn bản luật cũ đã hết hiệu lực vẫn bị truy xuất và tổng hợp vào câu trả lời của LLM.

#### 2. Kế hoạch cải tiến cụ thể
1. **Chunking Strategy:**
   - Chuyển sang **Structure-Aware + Hierarchical Chunking**: Parse tài liệu theo cấu trúc phân cấp pháp lý (Chương > Mục > Điều > Khoản). Lưu parent chunk ở cấp Điều (đầy đủ nội dung và điều kiện áp dụng), child chunk ở cấp Khoản/Điểm (để matching chính xác).
2. **Search Retrieval:**
   - Áp dụng **Hybrid Search (BM25 tiếng Việt + BGE-M3 Dense + RRF)**. BM25 bắt buộc phải có để bắt chính xác số hiệu văn bản, ngày tháng ban hành, tên định danh thực thể. BGE-M3 phụ trách tìm kiếm ngữ nghĩa sâu.
3. **Reranking:**
   - Bắt buộc triển khai **Cross-Encoder Reranker (`bge-reranker-v2-m3`)** để lọc top-25 ứng viên xuống top-5. Đo lường latency chặt chẽ; nếu cần phục vụ real-time < 200ms thì chuyển sang `FlashRank` với mô hình ONNX siêu nhẹ.
4. **Evaluation:**
   - Xây dựng benchmark test set gồm 100 câu hỏi luật thực tế với Ground Truth chuẩn được chuyên viên thẩm định. Đo đạc tự động hàng tuần qua **RAGAS 4 metrics**, đặt ngưỡng alert nếu Faithfulness < 0.85 hoặc Context Recall < 0.90.
5. **Enrichment:**
   - Áp dụng **Contextual Prepend + Auto Metadata Extraction**: Mỗi chunk văn bản luật sẽ được tự động gắn thêm metadata (Số hiệu, Ngày có hiệu lực, Tình trạng hiệu lực, Cơ quan ban hành). Khi người dùng tra cứu, hệ thống tự động filter `status == 'active'` trước khi tìm kiếm.

#### 3. Timeline triển khai (4 tuần)
- **Tuần 1:** Thiết kế lại ingestion pipeline: Parser cấu trúc văn bản pháp luật, triển khai Structure-Aware & Hierarchical Chunking.
- **Tuần 2:** Tích hợp Qdrant Vector DB, dựng Hybrid Search (BM25 + BGE-M3 + RRF) và kiểm thử độ bao phủ từ khóa số hiệu văn bản.
- **Tuần 3:** Tích hợp Cross-Encoder Reranker, tinh chỉnh prompt thế hệ mới với kỹ thuật Citation Enforcement (bắt buộc trích dẫn số Điều, Khoản).
- **Tuần 4:** Thiết lập quy trình CI/CD Evaluation với bộ test RAGAS, viết báo cáo Failure Analysis và đóng gói API production bằng FastAPI.
