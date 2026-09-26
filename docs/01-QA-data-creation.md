# QA Data Creation Pipeline

## 1. Mô tả
Xây dựng pipeline tự động hóa quy trình tổng hợp bộ dữ liệu Question-Answering (QA) chất lượng cao từ các phân đoạn văn bản (chunks) của đề cương môn học (syllabus). Bộ dữ liệu này đóng vai trò là ground truth (chuẩn đối sánh) để benchmark, fine-tune embedding/LLM hoặc đánh giá độ chính xác (retrieval & generation) của hệ thống RAG Syllabus.

## 2. Mục tiêu
- Cung cấp dữ liệu RAG chất lượng cao, chuẩn đối sánh (ground truth).
- Hỗ trợ đánh giá hiệu năng hệ thống RAG (Retrieval + Generation).
- Tự động hóa quá trình sinh cặp QA có căn cứ (grounded) chặt chẽ từ nội dung syllabus.
- Đảm bảo tính đa dạng về cách diễn đạt của câu hỏi (paraphrasing) nhưng vẫn giữ nguyên ý định truy vấn (intent).
- Giảm thiểu hiện tượng ảo giác (hallucination) qua các bộ quy tắc kiểm định (validation rules) tự động.

## 3. Input / Output
- **Input:** Danh sách các chunks văn bản (kèm metadata: môn học, mã môn, tuần học, nguồn văn bản).
- **Output:** Tập dữ liệu QA hoàn chỉnh (định dạng JSON/CSV), mỗi bản ghi gồm: `chunk_id`, `baseline_question`, `paraphrased_questions`, `ground_truth_answer`, `evidence_quote`.
- **Notes:** Mỗi chunk có thể sinh ra từ 1 đến 3 câu hỏi tùy thuộc vào mật độ thông tin (information density). Chunk quá ngắn hoặc chỉ chứa tiêu đề rỗng cần được lọc bỏ trước khi đưa vào pipeline.

## 4. Pipeline QA Data Creation

### Step 1: Trích xuất thông tin dành cho mỗi chunk
- Lọc bỏ nhiễu và xác định các thực thể thông tin cốt lõi trong chunk (như: chính sách điểm số, deadline, chuẩn đầu ra, tài liệu bắt buộc, lịch trình môn học).
- Gán nhãn ngữ cảnh cơ bản để phục vụ việc tạo câu hỏi độc lập (standalone question).

### Step 2: Tạo câu hỏi chuẩn (Baseline Question)
- **2.1. Prompting baseline question using information needed and evidence**
  - **Template Prompt:**
    > "Dựa vào đoạn trích syllabus dưới đây, hãy tạo 01 câu hỏi rõ ràng, độc lập mà sinh viên thường thắc mắc. Câu hỏi bắt buộc phải được trả lời hoàn toàn dựa vào đoạn trích, không suy đoán. Chỉ xuất ra câu hỏi và câu trích dẫn làm bằng chứng (evidence)."
- **2.2. Generate baseline question using Gemini**
- **2.3. Verify generate baseline question rules**
  - **Rules:**
    - **Rule 1 (Tính độc lập - Context Independence):** Câu hỏi không dùng đại từ tham chiếu mơ hồ (ví dụ: không hỏi "Môn này học gì?", mà phải là "Môn [Tên môn] học gì?").
    - **Rule 2 (Khả năng trả lời - Answerability):** Nội dung hỏi phải xuất hiện trực tiếp trong chunk, không đòi hỏi kiến thức bên ngoài.
    - **Rule 3 (Độ liên quan syllabus - Syllabus Relevance):** Câu hỏi phải phản ánh đúng nghiệp vụ học vụ/quy định môn học, tránh câu hỏi vụn vặt về lỗi định dạng.
    - **Rule 4 (Độ bao quát thông tin - Information Coverage):** Câu hỏi phải khai thác được thông tin giá trị, tránh hỏi những câu trả lời hiển nhiên hoặc không có ý nghĩa.
    - **Rule 5 (Định dạng câu hỏi - Question Format):** Câu hỏi phải ở dạng ngôn ngữ tự nhiên (natural language), không phải câu lệnh (command).

### Step 3: Tạo câu trả lời cho baseline question
- **Quy trình sinh:**
  - Prompt Gemini trả lời ngắn gọn, trực diện dựa trên `baseline_question` và `evidence` đã trích xuất ở Step 2.
  - Câu trả lời nêu rõ điều kiện hoặc ngoại lệ nếu syllabus có đề cập.
- **Verification Rules (Kiểm tra câu trả lời):**
  - **Faithfulness (Độ trung thực):** 100% chi tiết trong câu trả lời phải nằm trong chunk.
  - **Conciseness (Độ súc tích):** Trả lời đúng trọng tâm câu hỏi, không lặp lại toàn bộ chunk nếu không cần thiết.

### Step 4: Tạo các câu hỏi khác cùng nghĩa với baseline
- **4.1. Đa dạng hóa văn phong:** Sinh câu hỏi dạng trang trọng (formal/academic) và dạng đời thường (sinh viên nhắn tin).
- **4.2. Đa dạng hóa cấu trúc:** Biến đổi giữa câu hỏi Yes/No, câu hỏi lấy thông tin (Wh-questions), và câu hỏi điều kiện ("Nếu... thì...").
- **4.3. Sinh biến thể:** Dùng Gemini tạo từ 3 đến 5 câu hỏi đồng nghĩa từ `baseline_question`.
- **Verification Rules (Kiểm tra paraphrase):**
  - **Semantic Equivalence:** Ý định tra cứu không đổi; câu trả lời ở Step 3 vẫn giải quyết trọn vẹn cho câu hỏi mới.
  - **Lexical Diversity:** Tỉ lệ trùng lặp từ vựng với baseline không vượt quá ngưỡng quy định (ví dụ: Jaccard similarity < 0.7).

### Step 5: Tổng hợp QA data
Đóng gói toàn bộ kết quả thành cấu trúc chuẩn:

```json
{
  "id": "qa_001",
  "chunk_id": "chunk_syllabus_cs101_05",
  "baseline_question": "...",
  "paraphrases": [
    "...",
    "...",
    "..."
  ],
  "answer": "...",
  "evidence": "..."
}
```