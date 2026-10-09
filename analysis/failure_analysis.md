# Failure Analysis — Lab 18: Production RAG

**Họ và tên học viên:** Nguyễn Việt Hoàng Hải  
**MSSV:** 2A202602967  
**Khóa:** K4 - Track 3A  

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.8214 | 0.8850 | +0.0636 |
| Answer Relevancy | 0.7941 | 0.8620 | +0.0679 |
| Context Precision | 0.9091 | 0.9792 | +0.0701 |
| Context Recall | 0.9242 | 0.9450 | +0.0208 |

---

## Bottom-5 Failures

### #1
- **Question:** Bao lâu phải đổi mật khẩu một lần?
- **Expected:** Theo chính sách hiện hành (v2.0), mật khẩu phải được thay đổi mỗi 120 ngày. Chính sách cũ yêu cầu 90 ngày nhưng đã bị thay thế.
- **Got:** Mật khẩu phải thay đổi mỗi 90 ngày theo quy định an toàn thông tin nội bộ.
- **Worst metric:** Faithfulness (0.7200)
- **Error Tree:** Output sai → Context có chứa cả 2 phiên bản tài liệu (v1.0 và v2.0) → Query retrieval lấy về cả 2 chunks → LLM ưu tiên nhầm văn bản v1.0 cũ.
- **Root cause:** Xung đột phiên bản tài liệu (Temporal / Versioning conflict). Cả `mat_khau_v1.md` và `mat_khau_v2.md` đều có độ tương đồng embedding cao với từ khóa truy vấn, nhưng pipeline chưa có cơ chế lọc metadata theo `status: active` hoặc `effective_date`.
- **Suggested fix:** Thêm metadata filtering vào M2 Search (ưu tiên các văn bản có hiệu lực mới nhất) hoặc áp dụng Recency-weighted Reranker trong M3.

### #2
- **Question:** Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt?
- **Expected:** Các khoản chi tiêu mua sắm tài sản cố định từ 50 triệu đến dưới 100 triệu VNĐ cần được Giám đốc Khối hoặc Phó Tổng Giám đốc phụ trách phê duyệt bằng văn bản.
- **Got:** Cần Trưởng phòng và Kế toán trưởng phê duyệt theo hạn mức dưới 50 triệu đồng.
- **Worst metric:** Context Recall (0.7400)
- **Error Tree:** Output sai → Context thiếu đoạn quy định hạn mức 50 - 100 triệu → Semantic chunking cắt đôi bảng biểu hạn mức tài chính → Mất liên kết giữa con số 55 triệu và thẩm quyền phê duyệt tương ứng.
- **Root cause:** Bảng biểu markdown bị chia cắt khi áp dụng sentence splitting hoặc chunk size nhỏ, làm mất cấu trúc hàng/cột của ma trận phân quyền mua sắm.
- **Suggested fix:** Áp dụng Structure-aware chunking (Module 1) để nhận diện và bảo toàn toàn vẹn bảng markdown (Markdown Table Preserving Chunking) kèm tiêu đề mục.

### #3
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** Theo chính sách v2024: 15 ngày cơ bản + 3 ngày thâm niên (9÷3=3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got:** Nhân viên Senior có 9 năm thâm niên được hưởng 18 ngày phép năm. Thông tin về mức lương không được đề cập trong tài liệu chính sách nghỉ phép.
- **Worst metric:** Context Recall (0.7500)
- **Error Tree:** Output thiếu 1 vế → Context chỉ lấy được tài liệu nghỉ phép, thiếu tài liệu bảng lương → Query đơn luồng không cover hết 2 thực thể độc lập (`chính sách phép` và `bảng lương`).
- **Root cause:** Multi-hop query: Câu hỏi chứa 2 ý định truy vấn riêng biệt nằm ở 2 tệp tài liệu hoàn toàn khác nhau (`nghi_phep_nam_v2024.md` và `bang_luong_2024.md`). Dense/BM25 retrieval thông thường chỉ lấy top chunks tập trung vào vế có trọng số từ khóa cao hơn.
- **Suggested fix:** Triển khai Query Decomposition (tách câu hỏi thành 2 sub-queries: "ngày phép thâm niên 9 năm" và "mức lương cấp bậc Senior"), sau đó merge contexts từ cả hai nhánh tìm kiếm.

### #4
- **Question:** Có cần kích hoạt xác thực đa yếu tố (MFA) không?
- **Expected:** Có, theo chính sách mật khẩu v2.0 hiện hành, tất cả nhân viên bắt buộc kích hoạt MFA cho email, VPN và hệ thống nội bộ. Chính sách cũ v1.0 không yêu cầu MFA.
- **Got:** Cần tuân thủ quy chế bảo mật công nghệ thông tin gồm VPN WireGuard AES-256, đổi mật khẩu định kỳ và sử dụng phần mềm quét mã độc tập trung do IT cài đặt.
- **Worst metric:** Answer Relevancy (0.7600)
- **Error Tree:** Output lan man → Context chứa nhiều đoạn sổ tay an toàn thông tin chung chung → LLM tóm tắt quá nhiều chi tiết ngoài lề thay vì trả lời trực diện câu hỏi yes/no về MFA.
- **Root cause:** Prompt generation chưa có định hướng cụ thể (Direct Answer Instruction), khiến LLM sinh câu trả lời tổng quan thay vì trọng tâm câu hỏi.
- **Suggested fix:** Bổ sung instruction trong generation prompt: "Ưu tiên trả lời trực tiếp khẳng định/phủ định (Có/Không) ở đầu câu, sau đó trích dẫn chính sách cụ thể".

### #5
- **Question:** Nhân viên thử việc có được nghỉ phép năm không?
- **Expected:** KHÔNG. Nhân viên thử việc KHÔNG được nghỉ phép năm. Nếu cần nghỉ, phải xin nghỉ không lương và được trưởng phòng phê duyệt.
- **Got:** Nhân viên thử việc không được hưởng phép năm nhưng có thể được giải quyết nghỉ phép nếu có lý do chính đáng và nộp đơn trước 3 ngày.
- **Worst metric:** Faithfulness (0.7800)
- **Error Tree:** Output thừa chi tiết suy đoán → Context gốc ghi: "Nhân viên thử việc KHÔNG được hưởng phép năm có lương" → LLM tự suy luận thêm quy định nghỉ ốm/nghỉ đặc biệt vào điều kiện thử việc.
- **Root cause:** Hallucination mức độ nhẹ do LLM khái quát hóa quá đà (Overgeneralization) khi nhiệt độ tạo sinh hoặc prompt chưa đủ chặt chẽ.
- **Suggested fix:** Thiết lập `temperature = 0.0` và đưa vào negative constraint rõ ràng: "Không suy diễn, không tự thêm các điều kiện ngoại lệ nếu không có văn bản chứng minh rõ ràng trong context".

---

## Case Study (cho presentation)

**Question chọn phân tích:**  
*"Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?"*

**Error Tree walkthrough:**
1. **Output đúng?** → Không hoàn toàn (Đúng vế ngày phép năm: 18 ngày; Thiếu vế khung lương Senior: 20-35 triệu).
2. **Context đúng?** → Context chỉ có chunk từ `nghi_phep_nam_v2024.md`, hoàn toàn không có chunk từ `bang_luong_2024.md`.
3. **Query rewrite OK?** → Hiện tại chưa có bước Query Rewriting / Sub-query Decomposition, query gốc bị "bias" mạnh về vế ngày phép do nhiều từ khóa đồng quy tụ (`thâm niên`, `nghỉ`, `ngày phép`).
4. **Fix ở bước:**  
   - Bổ sung **Query Rewriter / Decomposer** trước M2 Search:
     - Sub-query 1: `nhân viên thâm niên 9 năm ngày phép năm v2024`
     - Sub-query 2: `khung lương vị trí Senior 2024`
   - Chạy Hybrid Search độc lập cho từng sub-query rồi dùng RRF tổng hợp trước khi đưa qua M3 Cross-Encoder Reranker.

**Nếu có thêm 1 giờ, sẽ optimize:**
- **Triển khai Metadata Routing / Date Filtering:** Loại bỏ hoàn toàn các văn bản đã hết hiệu lực (`v2023`, `v1.0`) để triệt tiêu lỗi xung đột phiên bản văn bản.
- **Tích hợp Query Decomposition & HyDE:** Xử lý triệt để các câu hỏi multi-hop và bắc cầu khoảng cách từ vựng (vocabulary gap) giữa câu hỏi đời thường và văn bản chính sách pháp quy.
