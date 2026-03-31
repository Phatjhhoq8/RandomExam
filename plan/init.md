# Kế Hoạch Triển Khai: Hệ Thống Tự Động Tạo và Thi Toán (AI Math Exam Engine & Online Quiz)

## 1. Mô tả Dự án (Project Description)
Hệ thống web-app ứng dụng Trí tuệ Nhân tạo (Generative AI) cho phép người dùng (giáo viên, học sinh) tạo ra các đề kiểm tra toán học tự động theo các tiêu chí cụ thể (lớp, môn, chuyên đề, độ khó, v.v.). Đặc biệt, người dùng có thể làm bài thi trắc nghiệm trực tiếp trên giao diện web. Hệ thống hỗ trợ tính năng tự động chấm điểm (Auto-grading) và kèm theo việc hiển thị lời giải chi tiết thông qua định dạng toán học (LaTeX).

## 2. Các Quyết Định Thiết Kế Lõi

### 2.1 AI-Driven Generation
Sử dụng các LLM mạnh mẽ thông qua API (như `@vercel/ai`) làm Engine tạo câu hỏi. Hệ thống sẽ ép mô hình trả về dữ liệu theo cấu trúc chuẩn JSON (Structured Output) chứa: Nội dung câu hỏi (LaTeX formatting), Các phương án Trắc nghiệm, Đáp án chính xác và Lời giải từng bước.

### 2.2 Chiến Lược Chọn Model
Thiết kế API layer trừu tượng (abstraction) để dễ dàng swap model. Chiến lược sử dụng:

| Ngữ cảnh | Model đề xuất | Lý do |
|-----------|---------------|-------|
| Preview / Draft nhanh | Gemini Flash | Tốc độ nhanh, chi phí thấp, đủ tốt cho bản nháp |
| Đề thi chính thức | Gemini Pro / GPT-4o | Độ chính xác toán học cao hơn, reasoning mạnh |
| Fallback | Model còn lại | Khi model chính bị rate-limit hoặc downtime |

### 2.3 Tính năng Làm bài Trực tuyến (Interactive Quiz)
Giao diện web được thiết kế như một phòng thi thu nhỏ với đồng hồ đếm ngược (Timer). Khi Submit, web ngay lập tức đối chiếu JSON, tính điểm số % và chuyển sang chế độ Review (Xem lời giải).

### 2.4 Phòng chống Hallucination (Toán học ảo giác)
Áp dụng chiến lược phòng thủ nhiều lớp:

1. **Lớp 1 — Prompt Engineering:** Tích hợp kỹ thuật Zero-shot Chain of Thought vào System Prompt, theo quy luật "Luôn giải bài đến bước cuối cùng ở background trước khi kết luận và tạo các đáp án gây nhiễu".
2. **Lớp 2 — Self-Verification:** Sau khi sinh câu hỏi, gọi LLM lần 2 với prompt riêng biệt: "Kiểm tra lời giải sau đây có chính xác không? Nếu sai, chỉ ra bước sai." Nếu 2 kết quả conflict → loại câu hỏi đó, sinh lại.
3. **Lớp 3 — Computational Verification (Nâng cao):** Với các bài toán có đáp án số (phép tính, phương trình), dùng thư viện tính toán phía server (`math.js` hoặc gọi `sympy` qua API) để verify kết quả. Nếu đáp án LLM ≠ kết quả tính toán → reject và sinh lại.

### 2.5 JSON Schema Validation
Sử dụng thư viện `zod` kết hợp `@vercel/ai` structured output để đảm bảo response từ LLM luôn đúng schema mong muốn:

```typescript
// Ví dụ Zod Schema
const QuestionSchema = z.object({
  id: z.number(),
  content: z.string(),           // Nội dung câu hỏi (LaTeX)
  options: z.array(z.object({
    label: z.enum(["A", "B", "C", "D"]),
    value: z.string(),           // Nội dung phương án (LaTeX)
  })).length(4),
  correctAnswer: z.enum(["A", "B", "C", "D"]),
  solution: z.string(),          // Lời giải từng bước (LaTeX)
  difficulty: z.number().min(1).max(5),
  // Metadata cho Weakness Detection (Section 2.6)
  knowledgeArea: z.string(),     // Ví dụ: "Lượng giác", "Đạo hàm"
  chapter: z.string(),           // Ví dụ: "Chương 6: Hàm số lượng giác"
  skillTags: z.array(z.string()),// Ví dụ: ["biến đổi lượng giác", "công thức cộng"]
});
```

### 2.6 Weakness Detection AI (Phân Tích Điểm Yếu)
Sau khi học sinh nộp bài, hệ thống không chỉ chấm điểm mà còn **phân tích chuyên sâu** để phát hiện lỗ hổng kiến thức:

#### Pipeline Phân Tích:
```
Kết quả bài thi
  │
  ├─ Bước 1: Phân loại câu sai theo Knowledge Area & Chapter
  │    → "Sai 3/4 câu Lượng giác, 0/3 câu Đạo hàm"
  │
  ├─ Bước 2: Phân tích Pattern lỗi (gọi LLM)
  │    → Input: Câu hỏi + Đáp án user chọn + Đáp án đúng + Lời giải
  │    → Output: Loại lỗi (nhầm công thức / sai dấu / thiếu điều kiện / ...)
  │
  ├─ Bước 3: Tổng hợp Weakness Report
  │    → Bảng thống kê: Dạng bài × Tỉ lệ sai × Loại lỗi phổ biến
  │
  └─ Bước 4: Sinh Gợi ý Cá nhân hóa (LLM)
       → "Học lại Chương 6: Công thức lượng giác cơ bản"
       → "Làm thêm 5 bài dạng: Biến đổi lượng giác (mức 2-3)"
       → Auto-generate bài tập bổ trợ theo đúng dạng yếu
```

#### Output Weakness Report:
```typescript
const WeaknessReportSchema = z.object({
  totalScore: z.number(),
  totalQuestions: z.number(),
  weakAreas: z.array(z.object({
    knowledgeArea: z.string(),       // "Lượng giác"
    chapter: z.string(),             // "Chương 6"
    totalQuestions: z.number(),      // 4 câu
    wrongCount: z.number(),          // sai 3
    commonMistakeType: z.string(),   // "Nhầm công thức cộng"
    severity: z.enum(["low", "medium", "high", "critical"]),
  })),
  recommendations: z.array(z.object({
    type: z.enum(["review_chapter", "practice_more", "watch_video"]),
    title: z.string(),               // "Ôn lại: Công thức lượng giác"
    description: z.string(),         // Chi tiết hướng dẫn
    suggestedDifficulty: z.number(), // Mức độ nên luyện
  })),
  canAutoGeneratePractice: z.boolean(), // Có thể tự sinh bài luyện?
});
```

#### Tính năng "Luyện lại điểm yếu" (1-click Practice):
- Từ Weakness Report, user bấm **"Luyện lại dạng này"** → Hệ thống tự sinh đề mới tập trung vào đúng dạng bài yếu, cùng mức độ khó hoặc thấp hơn 1 bậc.
- Tracking tiến bộ: So sánh kết quả lần thi trước vs lần luyện → hiển thị "Bạn đã cải thiện 40% ở dạng Lượng giác".

### 2.7 Generate from Image — OCR (Upload Đề Giấy)
Cho phép giáo viên/học sinh **chụp ảnh hoặc scan đề thi giấy** → AI tự động chuyển thành quiz trực tuyến.

#### Pipeline OCR → Quiz:
```
Ảnh đề thi (JPG/PNG/PDF)
  │
  ├─ Bước 1: Tiền xử lý ảnh
  │    → Chỉnh sáng, tăng contrast, deskew (nắn thẳng)
  │    → Thư viện: sharp (Node.js) hoặc xử lý client-side
  │
  ├─ Bước 2: OCR + Math Recognition
  │    → Option A: Gemini Vision API (đọc ảnh trực tiếp, nhận diện cả công thức)
  │    → Option B: Tesseract.js + MathPix API (OCR text + nhận diện LaTeX)
  │    → Output: Raw text chứa câu hỏi + đáp án (nếu có)
  │
  ├─ Bước 3: Structuring (LLM)
  │    → Gửi raw text vào LLM với prompt: "Phân tích đề thi này,
  │       tách thành các câu hỏi riêng biệt, format theo JSON schema"
  │    → Output: JSON theo QuestionSchema (tự động thêm lời giải nếu đề gốc không có)
  │
  └─ Bước 4: Review & Edit
       → Hiển thị kết quả cho user review trước khi tạo quiz
       → User có thể sửa câu hỏi, đáp án bị OCR sai
       → Confirm → Chuyển sang Quiz Engine
```

#### Lựa chọn Công nghệ OCR:
| Tiêu chí | Gemini Vision (Đề xuất MVP) | Tesseract + MathPix |
|----------|----------------------------|---------------------|
| Setup | Đơn giản (1 API call) | Phức tạp (2 service) |
| Nhận diện công thức | ✅ Tốt (native multimodal) | ✅ Rất tốt (MathPix chuyên biệt) |
| Chi phí | Thấp (bundled trong Gemini) | Cao (MathPix tính per-request) |
| Offline | ❌ | ⚠️ Tesseract offline, MathPix online |

> **Đề xuất MVP:** Dùng Gemini Vision API — 1 API call vừa OCR vừa structure, giảm complexity.

## 3. Cấu Trúc System Prompt

### 3.1 Template Prompt
```
[Role Definition]
"Bạn là một giáo viên toán cấp {level} với 20 năm kinh nghiệm ra đề thi.
Bạn chuyên về {topic} thuộc chương trình lớp {grade}."

[Task]
"Tạo {count} câu hỏi trắc nghiệm với độ khó {difficulty}/5."

[Constraints]
- PHẢI giải bài hoàn chỉnh trước khi tạo đáp án
- Đáp án nhiễu PHẢI hợp lý (dựa trên lỗi tính toán phổ biến của học sinh)
- Tất cả công thức toán PHẢI dùng LaTeX formatting
- Output PHẢI tuân theo JSON schema đã định nghĩa

[Output Schema]
{ ... JSON schema mô tả ... }

[Few-shot Examples]
Ví dụ 1: Input → Output mẫu
Ví dụ 2: Input → Output mẫu
```

### 3.2 Nguyên Tắc Tạo Đáp Án Nhiễu (Distractor Design)
- Đáp án nhiễu phải dựa trên **lỗi phổ biến** của học sinh (quên dấu âm, nhầm công thức, sai thứ tự phép tính)
- Không tạo đáp án vô nghĩa hoặc quá xa so với đáp án đúng
- Ít nhất 1 đáp án nhiễu phải là kết quả của một bước giải sai logic phổ biến

## 4. Phân Loại Độ Khó (Difficulty Scale)

| Mức | Tên | Mô tả | Ví dụ (Lớp 10) |
|-----|-----|-------|-----------------|
| 1 | Nhận biết | Áp dụng trực tiếp 1 công thức, 1 bước giải | Tính sin(30°) |
| 2 | Thông hiểu | Áp dụng công thức có biến đổi đơn giản, 1-2 bước | Giải phương trình bậc 2 đơn giản |
| 3 | Vận dụng | Kết hợp 2 kiến thức, 3-4 bước giải | Tìm tập xác định hàm phân thức có căn |
| 4 | Vận dụng cao | Kết hợp 3+ kiến thức, 4-5 bước, cần phân tích | Bài toán min/max hàm bậc 2 có điều kiện |
| 5 | Nâng cao | Bài toán tổng hợp, sáng tạo, >5 bước | Chứng minh BĐT kết hợp hệ phương trình |

> Lưu ý: Scale này cần được tuỳ chỉnh (calibrate) theo chương trình SGK cụ thể của từng lớp trong quá trình phát triển.

## 5. Kiến Trúc & Công Nghệ (Technical Stack)
Để đảm bảo trải nghiệm người dùng cao cấp (Premium UX), dự án sẽ được xây dựng theo kiến trúc Fullstack hiện đại:

- **Khung ứng dụng chung:** Next.js (App Router) cho cả Frontend UI lẫn Backend API (serverless functions).
- **Giao diện (UI/UX):** 
  - Sử dụng Tailwind CSS kết hợp shadcn/ui.
  - Xây dựng phong cách Thiết kế: Dark Mode, hiệu ứng Glassmorphism (Thẻ trong suốt), và Micro-animations để nút bấm có tính tương tác cao.
  - Hỗ trợ Render Toán học chuẩn: Cài đặt thư viện `react-katex` để render đẹp các công thức phức tạp (Tích phân, ma trận, phân số,...).
- **OCR/Vision:** Gemini Vision API cho tính năng Generate from Image (Section 2.7).
- **Khối Quản Trị Data (Tương lai/Phase 2):** Prisma ORM kết nối với PostgreSQL để lưu bộ câu hỏi và kết quả kiểm tra khi dự án đi sâu vào thương mại. 
*Phiên bản MVP đợt này sẽ hoạt động 100% dựa trên trạng thái (States API/Session) để đảm bảo tốc độ proof-of-concept.*

## 6. Error Handling & Resilience

### 6.1 API Error Handling
- **Retry Logic:** Áp dụng exponential backoff (1s → 2s → 4s, tối đa 3 lần) khi API call thất bại hoặc timeout.
- **Fallback Model:** Khi model chính bị rate-limit, tự động chuyển sang model dự phòng.
- **Graceful Degradation:** Nếu tất cả API đều fail, hiển thị thông báo lỗi thân thiện và cho phép user retry thủ công.

### 6.2 Invalid Response Handling
- Nếu LLM trả về JSON không qua được Zod validation → tự động retry (tối đa 2 lần) với prompt bổ sung: "Trả lại kết quả đúng theo schema."
- Log lại các response lỗi để phân tích và cải thiện prompt.

### 6.3 Quiz Session Protection
- **Auto-save:** Dùng `localStorage` lưu tiến trình làm bài (đáp án đã chọn, thời gian còn lại) mỗi khi user chọn/thay đổi đáp án.
- **Reconnect Recovery:** Khi user mở lại trang, kiểm tra `localStorage` — nếu có session dang dở, hỏi user "Tiếp tục bài thi trước?" hay "Bắt đầu mới".
- **Tab Close Warning:** Thêm `beforeunload` event để cảnh báo khi user đóng tab giữa chừng bài thi.

## 7. Lộ Trình Triển Khai (Roadmap & Checklist)

### Bước 1: Khởi tạo Project & Cấu trúc gốc
- **Thời gian ước lượng:** 1 ngày
- **Công việc:** Setup Next.js App Router, cài Tailwind + shadcn/ui, cấu hình font Inter, thiết lập `react-katex`.
- **Definition of Done:** 
  - ✅ `npm run dev` chạy được không lỗi
  - ✅ Trang mẫu render được công thức LaTeX phức tạp (tích phân, ma trận)
  - ✅ Dark mode toggle hoạt động
  - ✅ Cấu trúc thư mục chuẩn Next.js App Router

### Bước 2: Xây dựng Module AI Generator
- **Thời gian ước lượng:** 2-3 ngày
- **Công việc:** Thiết kế System Prompt theo template (Section 3), cài đặt Zod schema, gọi API sinh dữ liệu JSON, implement Validation Layer (self-verification).
- **Definition of Done:**
  - ✅ Gọi API → nhận JSON hợp lệ (pass Zod validation) ≥ 90% requests
  - ✅ Self-verification layer hoạt động (gọi LLM lần 2 kiểm tra)
  - ✅ Retry logic hoạt động khi API fail
  - ✅ Test được qua CLI/Postman trước khi nối Frontend

### Bước 3: Thiết kế UI Configuration (Form nhập liệu)
- **Thời gian ước lượng:** 1-2 ngày
- **Công việc:** Màn hình cho người dùng chọn Tuỳ biến đề: Lớp (1-12), Chủ đề/Chương, Số câu hỏi, Mức độ khó (1-5), Thời gian làm bài.
- **Definition of Done:**
  - ✅ Form validation hoạt động (không submit được nếu thiếu trường)
  - ✅ UI responsive trên mobile và desktop
  - ✅ Truyền params đúng đến API route
  - ✅ Loading state đẹp khi đang chờ AI sinh đề

### Bước 4: UI Tương tác Thi (Quiz Engine)
- **Thời gian ước lượng:** 2-3 ngày
- **Công việc:** Giao diện làm bài bấm giờ, navigation giữa các câu hỏi, lưu đáp án, auto-save vào localStorage.
- **Definition of Done:**
  - ✅ Timer đếm ngược chính xác, hết giờ tự động nộp bài
  - ✅ Câu hỏi render LaTeX đẹp qua react-katex
  - ✅ Lưu được đáp án vào state + localStorage
  - ✅ Navigation: Chuyển câu, đánh dấu câu đã làm/chưa làm
  - ✅ `beforeunload` warning khi đóng tab giữa bài thi
  - ✅ Reconnect recovery từ localStorage

### Bước 5: Auto-grading & Review View
- **Thời gian ước lượng:** 1-2 ngày
- **Công việc:** Hàm đối soát điểm, giao diện thẻ chứa lời giải chi tiết, highlight câu đúng/sai.
- **Definition of Done:**
  - ✅ Tính điểm chính xác 100% (unit test đảm bảo)
  - ✅ Hiển thị lời giải từng bước với LaTeX rendering
  - ✅ Phân biệt rõ câu đúng (xanh) / sai (đỏ) / chưa làm (xám)
  - ✅ Thống kê tổng quan: Điểm, số câu đúng/sai, thời gian hoàn thành

### Bước 6: Weakness Detection AI
- **Thời gian ước lượng:** 2-3 ngày
- **Công việc:** Xây dựng pipeline phân tích điểm yếu sau thi (Section 2.6), UI hiển thị Weakness Report, nút "Luyện lại dạng này".
- **Definition of Done:**
  - ✅ Phân loại câu sai theo knowledge area & chapter chính xác
  - ✅ LLM phân tích được pattern lỗi (loại sai: nhầm công thức, sai dấu, v.v.)
  - ✅ Weakness Report UI: Bảng thống kê + biểu đồ radar/bar chart
  - ✅ Gợi ý cá nhân hóa: "Học lại chương X", "Làm thêm bài dạng Y"
  - ✅ Nút "Luyện lại" auto-generate đề tập trung vào dạng yếu

### Bước 7: Generate from Image (OCR)
- **Thời gian ước lượng:** 2-3 ngày
- **Công việc:** UI upload ảnh, tích hợp Gemini Vision API, pipeline OCR → JSON → Quiz (Section 2.7), màn hình review/edit trước khi tạo quiz.
- **Definition of Done:**
  - ✅ Upload ảnh (JPG/PNG) + preview
  - ✅ Gemini Vision nhận diện được text + công thức toán từ ảnh
  - ✅ LLM structure raw text thành JSON (pass Zod validation)
  - ✅ UI review: User xem và sửa kết quả OCR trước khi confirm
  - ✅ Confirm → chuyển thẳng sang Quiz Engine
  - ✅ Xử lý edge case: ảnh mờ, nghiêng, nhiều trang

**Tổng thời gian MVP ước lượng: 11-17 ngày**

## 8. Chiến Lược Testing

### 8.1 Unit Tests
- **Hàm chấm điểm (grading):** Test với nhiều edge case — toàn đúng, toàn sai, bỏ trống, mix.
- **Zod schema validation:** Test với JSON hợp lệ và không hợp lệ.
- **Timer logic:** Test countdown, auto-submit khi hết giờ.
- **Weakness classification:** Test phân loại câu sai theo đúng knowledge area.

### 8.2 Integration Tests
- **API route test:** Gọi API sinh đề → kiểm tra response format.
- **Full flow test:** Config → Generate → Quiz → Submit → Review → Weakness Report.
- **OCR flow test:** Upload ảnh → OCR → Review → Quiz.

### 8.3 Prompt Regression Tests
- Lưu bộ test cases (input prompt configs → kỳ vọng chất lượng output).
- Khi thay đổi System Prompt, chạy lại bộ test để đảm bảo không giảm chất lượng.
- Metrics theo dõi: Tỉ lệ JSON valid, tỉ lệ đáp án đúng (qua self-verify), thời gian response.
- **OCR accuracy test:** Bộ ảnh mẫu → kiểm tra tỉ lệ nhận diện đúng.

### 8.4 E2E Tests (Manual hoặc Playwright)
- Luồng hoàn chỉnh: Chọn cấu hình → Sinh đề → Làm bài → Nộp bài → Xem điểm + lời giải → Weakness Report.
- Luồng OCR: Upload ảnh → Review → Làm bài → Kết quả.
- Test trên mobile viewport.

## 9. Data Strategy

### 9.1 Cache Đề Đã Generate
Tránh gọi API lặp lại cho các cấu hình tương tự, tiết kiệm chi phí và tăng tốc:

```
Cache Key = hash(grade + topic + chapter + difficulty + questionCount)
```

| Layer | Công nghệ | TTL | Mục đích |
|-------|-----------|-----|----------|
| Client Cache | `localStorage` | 24h | Đề vừa sinh, user quay lại không cần gọi API |
| Server Cache | In-memory (Map) hoặc Redis (Phase 2) | 1h | Nhiều user cùng config → serve từ cache |
| Persistent Store | JSON files hoặc DB (Phase 2) | Vĩnh viễn | Lưu đề chất lượng cao đã qua verification |

- Khi cache hit → trả đề từ cache, **nhưng randomize thứ tự câu hỏi + đáp án** (xem Section 10) để mỗi lần vẫn khác nhau.
- Cache invalidation: Khi thay đổi System Prompt → clear toàn bộ cache.

### 9.2 Batch Generation (Pre-generate)
Sinh sẵn đề trong background để giảm thời gian chờ khi user request:

- **Trigger:** Khi server idle hoặc theo schedule (cron job).
- **Chiến lược:** Pre-generate cho các cấu hình phổ biến nhất:
  - Lớp 10, 11, 12 (thi ĐH)
  - Các chương đang trong mùa thi (mapping theo lịch học)
  - Mức độ khó 2-4 (phổ biến nhất)
- **Số lượng:** Mỗi cấu hình duy trì pool ~5-10 đề sẵn.
- **Quality gate:** Chỉ đề đã pass self-verification mới được đưa vào pool.

### 9.3 Free Tier & Rate Limiting
Kiểm soát chi phí API và ngăn abuse:

| Tier | Giới hạn | Ghi chú |
|------|----------|---------|
| **Free (Anonymous)** | 3 đề/ngày | Tracked bằng fingerprint + localStorage |
| **Free (Đăng nhập)** | 5 đề/ngày | Tracked bằng user ID |
| **Premium (Phase 2)** | Unlimited | Subscription model |

- **Rate limit implementation:** Middleware kiểm tra count trước khi gọi AI API.
- **Khi hết quota:** Hiển thị thông báo thân thiện + suggest đăng ký / quay lại ngày mai.
- **MVP:** Dùng `localStorage` + IP-based tracking (đơn giản). Phase 2 chuyển sang DB-backed.

## 10. Anti-Cheating (Chống Gian Lận)

### 10.1 Randomization
Đảm bảo mỗi lần thi, thứ tự câu hỏi và đáp án đều khác nhau:

- **Randomize thứ tự câu hỏi:** Shuffle mảng questions bằng Fisher-Yates algorithm.
- **Randomize thứ tự đáp án A/B/C/D:** Với mỗi câu, shuffle 4 options → cập nhật lại `correctAnswer` tương ứng.
- **Seed-based randomization (Optional):** Dùng exam ID làm seed → cùng đề nhưng thứ tự khác nhau cho mỗi user, đồng thời reproducible để debug.

```typescript
// Ví dụ shuffle đáp án
function shuffleOptions(question: Question): Question {
  const shuffled = [...question.options].sort(() => Math.random() - 0.5);
  const newCorrectIndex = shuffled.findIndex(
    opt => opt.label === question.correctAnswer
  );
  const labels = ["A", "B", "C", "D"] as const;
  return {
    ...question,
    options: shuffled.map((opt, i) => ({ ...opt, label: labels[i] })),
    correctAnswer: labels[newCorrectIndex],
  };
}
```

### 10.2 Tab Switch Detection (Focus Loss)
Phát hiện khi học sinh chuyển tab (có thể đang tra Google):

- **Cơ chế:** Lắng nghe `document.visibilitychange` event.
- **Hành vi:**
  - Đếm số lần chuyển tab → hiển thị trong kết quả thi: "Chuyển tab: 3 lần".
  - **Cảnh báo nhẹ (mặc định):** Popup thông báo "Bạn vừa rời khỏi trang thi" khi quay lại.
  - **Chế độ nghiêm ngặt (giáo viên bật):** Sau 3 lần chuyển tab → tự động nộp bài.
- **Lưu ý:** Đây là biện pháp **tâm lý** hơn là kỹ thuật — không thể chống hoàn toàn trên web.

### 10.3 Disable Copy (Optional)
Ngăn sao chép nội dung đề thi để paste vào ChatGPT:

- `user-select: none` trên vùng câu hỏi (CSS).
- Chặn `Ctrl+C`, `Ctrl+A`, right-click context menu (JavaScript).
- **Lưu ý:** Đây là opt-in feature (giáo viên chọn bật/tắt), vì có thể gây khó chịu cho user legitimate.
- **Giới hạn:** Không chống được screenshot — chấp nhận trade-off này cho MVP.

### 10.4 Cấu Hình Anti-cheating (Giáo viên)
Giáo viên có thể tuỳ chỉnh mức độ chống gian lận khi tạo đề:

```typescript
const AntiCheatConfigSchema = z.object({
  shuffleQuestions: z.boolean().default(true),
  shuffleOptions: z.boolean().default(true),
  detectTabSwitch: z.boolean().default(true),
  maxTabSwitches: z.number().min(1).max(10).default(5),
  autoSubmitOnExceed: z.boolean().default(false),  // Tự nộp khi vượt limit
  disableCopy: z.boolean().default(false),
  showTabSwitchCount: z.boolean().default(true),   // Hiện số lần chuyển tab trong kết quả
});
```
