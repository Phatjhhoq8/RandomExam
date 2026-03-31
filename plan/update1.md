# Phase 2: Kế Hoạch Mở Rộng MathQuiz AI

> **Phiên bản**: update1 — 2026-04-01
> **Trạng thái**: Đã cập nhật

## 1. Tổng quan

Mở rộng hệ thống từ **ứng dụng thi trực tuyến đơn giản** thành **nền tảng quản lý đề thi toàn diện** phục vụ cả học sinh lẫn giáo viên.

Đồng thời, bản kế hoạch này được cập nhật để **đóng toàn bộ các hạng mục MVP còn dang dở từ `init.md`** trước khi mở rộng sâu sang Phase 2. Nguyên tắc mới là:

- Không coi mock AI/OCR là tính năng đã hoàn thành.
- Không mở rộng kiến trúc lớn khi các luồng MVP cốt lõi chưa ổn định.
- Mọi phase mới phải đi qua **MVP Verification Gate** trước khi được xem là sẵn sàng merge vào luồng chính.

### Các tính năng mới:

| # | Tính năng | Mô tả |
|---|-----------|-------|
| 1 | Gemini API | Kết nối AI thật, sinh đề hoàn toàn mới mỗi lần |
| 2 | Đa chương | Chọn nhiều chương, cấu hình số câu/chương |
| 3 | Phân bổ độ khó | Dễ / Trung bình / Khó theo tỷ lệ tùy chỉnh |
| 4 | In đề giấy | Layout chuẩn THPT QG, in từ browser |
| 5 | Nhiều mã đề | Tạo N mã đề xáo trộn khác nhau + bảng đáp án |
| 6 | Lưu đề thi | Lưu đề + đáp án để chấm bài sau |
| 7 | Question Bank | Kho câu hỏi phân loại theo chương/đề/độ khó |
| 8 | Template thay số | Tạo câu mới bằng cách thay số từ câu cũ (fallback khi API lỗi) |
| 9 | Quét phiếu chấm | Upload ảnh phiếu trả lời → OCR → chấm điểm tự động |
| 10 | **Adaptive Difficulty** | Tự điều chỉnh độ khó: sai nhiều → giảm khó, đúng nhiều → tăng khó (có nút bật/tắt) |

---

## 2. Kiến trúc dữ liệu mới

### 2.1 Question Bank (Kho câu hỏi)

Mỗi câu hỏi được sinh ra (bất kể từ AI, OCR, hay thủ công) đều được lưu vào kho để tái sử dụng.

```typescript
interface BankQuestion {
  id: string;                          // UUID duy nhất
  content: string;                     // Nội dung câu hỏi (LaTeX)
  options: Option[];                   // 4 đáp án A/B/C/D
  correctAnswer: "A"|"B"|"C"|"D";
  solution: string;                    // Lời giải chi tiết

  // --- Phân loại ---
  grade: number;                       // Lớp (6-12)
  topic: string;                       // Chuyên đề (Đại số, Hình học...)
  chapter: string;                     // Chương cụ thể
  difficulty: 1|2|3|4|5;               // Nhận biết → Nâng cao
  skillTags: string[];                 // Tags kỹ năng

  // --- Metadata ---
  source: "ai" | "ocr" | "manual" | "template";
  templateId?: string;                 // ID template gốc (nếu sinh từ template)
  createdAt: string;
  usageCount: number;                  // Số lần đã dùng trong đề thi
}
```

**Lưu trữ**: File `data/question-bank.json` trên server (Next.js API Routes đọc/ghi).

### 2.2 Exam Config V2 (Cấu hình đề thi mới)

```typescript
interface ExamConfigV2 {
  grade: number;
  title: string;                       // "Kiểm tra HK1 Toán 12"

  // --- Đa chương ---
  chapters: ChapterConfig[];           // Mảng các chương + số câu

  // --- Phân bổ độ khó ---
  difficultyDistribution: {
    easy: number;                      // Số câu dễ (mức 1-2)
    medium: number;                    // Số câu trung bình (mức 3)
    hard: number;                      // Số câu khó (mức 4-5)
  };

  timeLimit: number;                   // Phút

  // --- Chế độ sử dụng ---
  mode: "online" | "print";           // Thi online hay in giấy
  examCodeCount?: number;              // Số mã đề (1-6, chỉ khi print)

  // --- Nguồn câu hỏi ---
  questionSource: "ai" | "bank" | "mixed";
}

interface ChapterConfig {
  topic: string;                       // Chuyên đề
  chapter: string;                     // Tên chương
  questionCount: number;               // Số câu cho chương này
}
```

**Ví dụ cấu hình**:
```json
{
  "grade": 12,
  "title": "Kiểm tra HK1 Toán 12",
  "chapters": [
    { "topic": "Giải tích", "chapter": "Ứng dụng đạo hàm", "questionCount": 10 },
    { "topic": "Giải tích", "chapter": "Nguyên hàm, tích phân", "questionCount": 10 },
    { "topic": "Hình học", "chapter": "Khối đa diện", "questionCount": 10 }
  ],
  "difficultyDistribution": { "easy": 12, "medium": 12, "hard": 6 },
  "timeLimit": 90,
  "mode": "print",
  "examCodeCount": 4,
  "questionSource": "mixed"
}
```

### 2.3 Saved Exam (Đề đã lưu)

```typescript
interface SavedExam {
  id: string;
  config: ExamConfigV2;
  codes: ExamCode[];                   // Nhiều mã đề
  createdAt: string;
  status: "draft" | "active" | "graded";
}

interface ExamCode {
  code: string;                        // VD: "132", "246", "358", "470"
  questions: Question[];               // Thứ tự câu hỏi cho mã này
  answerKey: Record<number, "A"|"B"|"C"|"D">;
}
```

### 2.4 Question Template (Mẫu câu hỏi thay số)

```typescript
interface QuestionTemplate {
  id: string;
  contentTemplate: string;             // "Giải PT: ${{a}}x^2 + {{b}}x + {{c}} = 0$"
  optionTemplates: string[];           // Đáp án cũng chứa biến
  solutionTemplate: string;

  variables: TemplateVariable[];
  computeAnswer: string;               // JS expression tính đáp án từ biến

  // Metadata
  grade: number;
  topic: string;
  chapter: string;
  difficulty: number;
}

interface TemplateVariable {
  name: string;                        // "a", "b", "c"  
  type: "integer" | "fraction";
  min: number;
  max: number;
  excludeZero?: boolean;
}
```

**Ví dụ workflow thay số**:
```
Câu gốc (từ AI hoặc Bank):
  "Giải PT: $2x^2 - 5x + 3 = 0$" → Đáp án: x=1, x=3/2

      ↓ Trích xuất số → tạo template

Template:
  "Giải PT: ${{a}}x^2 + {{b}}x + {{c}} = 0$"
  Variables: a ∈ [1,5], b ∈ [-10,10], c ∈ [-10,10]

      ↓ Random số mới (đảm bảo delta >= 0)

Câu mới:
  "Giải PT: $3x^2 + 7x - 2 = 0$" → Tính lại đáp án tự động
```

---

## 3. Cấu trúc thư mục mới

```
RandomEx/app/src/
├── app/
│   ├── api/
│   │   ├── generate/route.ts              ← [SỬA] Gemini API + fallback
│   │   ├── bank/route.ts                  ← [MỚI] GET/POST Question Bank
│   │   ├── bank/[id]/route.ts             ← [MỚI] GET/DELETE single question
│   │   ├── exams/route.ts                 ← [MỚI] GET/POST Saved Exams
│   │   └── exams/[id]/route.ts            ← [MỚI] GET Exam detail + codes
│   └── page.tsx                           ← [SỬA] Thêm views mới
│
├── components/
│   ├── ConfigForm.tsx                     ← [GIỮ] (backward compatible)
│   ├── ConfigFormV2.tsx                   ← [MỚI] Multi-chapter form
│   ├── DifficultyDistribution.tsx         ← [MỚI] Slider dễ/TB/khó
│   ├── PrintExam.tsx                      ← [MỚI] Layout in đề chuẩn THPT QG
│   ├── AnswerSheet.tsx                    ← [MỚI] Phiếu trả lời bubble
│   ├── AnswerKeySheet.tsx                 ← [MỚI] Bảng đáp án cho GV
│   ├── ExamManager.tsx                    ← [MỚI] Quản lý đề đã tạo
│   ├── QuestionBankBrowser.tsx            ← [MỚI] Duyệt/tìm kho câu hỏi
│   ├── QuizEngine.tsx                     ← [GIỮ]
│   ├── ResultView.tsx                     ← [GIỮ]
│   ├── WeaknessView.tsx                   ← [GIỮ]
│   ├── MathRenderer.tsx                   ← [GIỮ]
│   ├── OCRUpload.tsx                      ← [SỬA] Thêm mode quét phiếu
│   └── LoadingScreen.tsx                  ← [GIỮ]
│
├── lib/
│   ├── schemas.ts                         ← [SỬA] Thêm schemas mới
│   ├── ai-generator.ts                   ← [SỬA] Gemini API thật + fallback
│   ├── question-bank.ts                  ← [MỚI] CRUD kho câu hỏi (JSON)
│   ├── template-engine.ts               ← [MỚI] Thay số tạo câu mới
│   ├── exam-builder.ts                   ← [MỚI] Build đề từ nhiều nguồn
│   ├── print-utils.ts                    ← [MỚI] Helper cho in ấn
│   └── utils.ts                           ← [GIỮ]
│
└── data/                                  ← [MỚI] Persistent storage
    ├── question-bank.json                 ← Kho câu hỏi (tự tích lũy)
    ├── templates.json                     ← Mẫu câu hỏi thay số
    └── exams/                             ← Thư mục đề đã lưu
        ├── exam_xxxx.json
        └── ...
```

---

## 4. Chi tiết các Phase triển khai

### Phase 0: Init Completion / MVP Hardening
**Ưu tiên: 🔴 RẤT CAO | Thời gian: ~2-4 giờ**

**Mục tiêu**: Hoàn tất các Definition of Done còn thiếu trong `init.md`, để bản MVP hiện tại đạt trạng thái ổn định trước khi mở rộng kiến trúc.

#### Công việc:

1. **Chuẩn hóa nền tảng UI/UX từ init**
   - Xác nhận lại quyết định về `dark mode toggle`:
     - Nếu vẫn giữ theo `init.md` -> triển khai thật
     - Nếu bỏ -> cập nhật lại spec để không còn lệch giữa kế hoạch và code
   - Rà soát việc render toán học:
     - Chuẩn hóa 1 hướng duy nhất giữa `react-katex` và renderer KaTeX custom
     - Cập nhật tài liệu để không còn mâu thuẫn giữa stack mô tả và code thực tế
   - Dọn `README.md` khỏi boilerplate, thay bằng mô tả đúng về MathQuiz AI MVP

2. **Hoàn tất Quiz Session Recovery**
   - Kích hoạt luồng `loadQuizSession()` khi vào quiz hoặc quay lại trang
   - Hiển thị lựa chọn: `Tiếp tục bài thi trước` / `Bắt đầu bài mới`
   - Khôi phục đầy đủ:
     - đáp án đã chọn
     - thời gian còn lại
     - số lần chuyển tab
     - thời điểm bắt đầu
   - Bảo đảm clear session đúng lúc sau khi submit hoặc hủy

3. **Siết chặt luồng anti-cheat và submit**
   - Rà soát `beforeunload`, tab-switch warning, auto-submit khi vượt ngưỡng
   - Xác minh flow shuffle câu hỏi/đáp án không làm lệch kết quả chấm
   - Kiểm tra lại retry flow khi user làm lại đề (`retry`) để không tạo state sai

4. **Chuẩn hóa trạng thái tính năng đang là heuristic / demo**
   - Gắn rõ trong kế hoạch và UI rằng `Weakness Analysis` hiện tại là heuristic nếu chưa có AI backend
   - Gắn rõ OCR hiện tại là demo/mock nếu chưa gọi Gemini Vision thật
   - Không cho phép mô tả các phần này là `completed` trong tài liệu release nếu chưa có implementation thật

5. **Thiết lập kiểm thử tối thiểu cho MVP**
   - Thêm unit test cho grading logic
   - Thêm test hoặc checklist xác minh schema validation của `/api/generate`
   - Thêm checklist manual cho full flow:
     - config -> generate -> quiz -> submit -> result -> weakness
     - upload ảnh -> review -> confirm -> quiz
     - refresh tab / mở lại tab -> session recovery

#### Files thay đổi:
- `[SỬA]` `src/components/QuizEngine.tsx` — reconnect recovery + anti-cheat hardening
- `[SỬA]` `src/lib/utils.ts` — session restore helpers + grading verification helpers
- `[SỬA]` `src/components/WeaknessView.tsx` — tách rõ heuristic mode
- `[SỬA]` `src/components/OCRUpload.tsx` — ghi rõ demo mode nếu OCR thật chưa sẵn sàng
- `[SỬA]` `src/app/page.tsx` — thêm prompt recovery / state recovery flow
- `[SỬA]` `README.md` — thay boilerplate bằng tài liệu MVP thực tế
- `[MỚI]` `tests/*` hoặc checklist manual tương đương — verification cho MVP

---

### Phase 1: Gemini API + Question Bank Storage + Verification
**Ưu tiên: 🔴 CAO | Thời gian: ~3-4 giờ**

**Mục tiêu**: Kết nối AI thật + mọi câu hỏi sinh ra đều tự động **kiểm tra đúng sai** rồi lưu vào kho.

#### Công việc:

1. **Kết nối Gemini API** (`ai-generator.ts`)
   - Sử dụng `@ai-sdk/google` (đã cài sẵn trong `package.json`)
   - Gọi `generateObject()` với Zod schema để nhận structured JSON
   - Retry logic: 3 lần, exponential backoff (1s → 2s → 4s)
   - **Fallback chain**: Gemini API → Template → Mock data

2. **Hệ thống Verification 5 lớp** (chi tiết xem Phase 8)
   - Tích hợp trực tiếp vào pipeline sinh đề
   - Mỗi câu AI sinh ra → qua 5 lớp kiểm tra → gán confidence score
   - Câu LOW confidence → tự gắn tag "⚠️ Cần GV kiểm tra"

3. **Question Bank module** (`question-bank.ts`)
   - `saveQuestions(questions: BankQuestion[])` → Append vào JSON file
   - `getQuestions(filters: {grade?, topic?, chapter?, difficulty?})` → Query
   - `getRandomQuestions(filters, count)` → Random câu từ kho
   - Tự deduplicate (kiểm tra content hash, không lưu trùng)

4. **API Routes** (`/api/bank`)
   - `GET /api/bank?grade=10&topic=Đại số` → Trả về câu hỏi matching
   - `POST /api/bank` → Thêm câu thủ công
   - `DELETE /api/bank/[id]` → Xóa câu

5. **Auto-save pipeline**:
   - Sau khi AI sinh đề → verify → tự động gọi `saveQuestions()` lưu vào bank
   - Mỗi câu lưu kèm `confidenceScore` + `verified: boolean`
   - Log: "Đã lưu X câu mới vào kho (Y câu verified, Z câu cần review)"

#### Files thay đổi:
- `[SỬA]` `src/lib/ai-generator.ts` — Gemini API call + fallback
- `[MỚI]` `src/lib/question-bank.ts` — CRUD kho câu hỏi
- `[MỚI]` `src/lib/verification.ts` — 5-layer verification engine
- `[MỚI]` `src/app/api/bank/route.ts` — API endpoint
- `[MỚI]` `src/data/question-bank.json` — File storage (khởi tạo `[]`)
- `[SỬA]` `.env` — Đã có `GOOGLE_GENERATIVE_AI_API_KEY`

---

### Phase 2: Multi-chapter Config + Phân bổ độ khó
**Ưu tiên: 🔴 CAO | Thời gian: ~2-3 giờ**

**Mục tiêu**: Form mới cho phép chọn nhiều chương + kiểm soát tỷ lệ dễ/khó.

#### Công việc:

1. **ConfigFormV2.tsx** — Form cấu hình đề thi mới
   - Bước 1: Chọn Lớp (giữ nguyên UI pills)
   - Bước 2: Chọn Chuyên đề (giữ nguyên)
   - Bước 3: **Multi-chapter picker**
     - Hiển thị tất cả chương dạng checkbox/pills
     - Nút "Chọn tất cả" / "Bỏ chọn hết" cho mỗi chuyên đề
     - Mỗi chương đã chọn hiển thị input số câu
   - Bước 4: **Phân bổ độ khó** (component riêng)
   - Bước 5: Thời gian + Chế độ (online/print)

2. **DifficultyDistribution.tsx**
   - 3 row: Dễ (mức 1-2) / TB (mức 3) / Khó (mức 4-5)
   - Mỗi row: label + số input + thanh progress
   - Auto-suggest khi nhập tổng câu:
     - 30 câu → gợi ý 12 dễ / 12 TB / 6 khó (40/40/20)
     - 50 câu → gợi ý 20 dễ / 20 TB / 10 khó
   - Validation: tổng 3 mức = tổng số câu tất cả chương

3. **Exam Builder** (`exam-builder.ts`)
   - Input: `ExamConfigV2`
   - Logic:
     ```
     Với mỗi chapter:
       → Tính số câu dễ/TB/khó theo tỷ lệ
       → Ưu tiên lấy từ Question Bank
       → Nếu thiếu → gọi AI sinh thêm
       → Nếu AI lỗi → dùng Template thay số
     Trộn tất cả → đánh số → tạo Exam
     ```

#### Files thay đổi:
- `[MỚI]` `src/components/ConfigFormV2.tsx`
- `[MỚI]` `src/components/DifficultyDistribution.tsx`
- `[MỚI]` `src/lib/exam-builder.ts`
- `[SỬA]` `src/lib/schemas.ts` — Thêm ExamConfigV2, ChapterConfig
- `[SỬA]` `src/app/page.tsx` — Thêm view mới, route đến ConfigFormV2

---

### Phase 3: Template Engine (Thay số tạo câu mới)
**Ưu tiên: 🟡 TRUNG BÌNH | Thời gian: ~2 giờ**

**Mục tiêu**: Khi API lỗi hoặc kho không đủ, tạo câu mới bằng cách thay số từ câu cũ.

#### Công việc:

1. **template-engine.ts**
   - `extractTemplate(question: BankQuestion)` → Phát hiện số trong nội dung, thay bằng biến `{{a}}`, `{{b}}`...
   - `generateFromTemplate(template: QuestionTemplate)` → Random số mới trong khoảng cho phép
   - `validateGenerated(question)` → Kiểm tra đáp án tính lại có đúng không
   - Đảm bảo: delta ≥ 0 (cho PT bậc 2), mẫu ≠ 0 (cho phân số)...

2. **Auto-extract templates**:
   - Mỗi khi câu mới vào Bank → tự thử extract template
   - Nếu thành công → lưu vào `data/templates.json`

3. **Tích hợp vào fallback chain**:
   ```
   Yêu cầu sinh câu → Gemini API
     ↓ (nếu lỗi / rate limit)
   Template Engine → Random số mới từ mẫu có sẵn
     ↓ (nếu không có mẫu phù hợp)
   Mock data (giữ nguyên)
   ```

#### Files thay đổi:
- `[MỚI]` `src/lib/template-engine.ts`
- `[MỚI]` `src/data/templates.json` — Khởi tạo `[]`
- `[SỬA]` `src/lib/ai-generator.ts` — Thêm template vào fallback chain

---

### Phase 4: Teacher Print Mode + Nhiều Mã Đề
**Ưu tiên: 🔴 CAO | Thời gian: ~3-4 giờ**

**Mục tiêu**: Giáo viên tạo đề → in giấy nhiều mã → phát cho học sinh thi.

#### Công việc:

1. **PrintExam.tsx** — Layout đề thi chuẩn THPT QG
   ```
   ┌──────────────────────────────────────────────┐
   │  SỞ GD&ĐT .............    ĐỀ KIỂM TRA      │
   │  TRƯỜNG .................  MÔN: TOÁN – LỚP 12│
   │                            Thời gian: 90 phút │
   │                            (Không kể phát đề) │
   │                            MÃ ĐỀ: 132         │
   │──────────────────────────────────────────────│
   │                                              │
   │  Câu 1. Cho hàm số y = x³ - 3x + 2.        │
   │  Hàm số đồng biến trên khoảng nào?          │
   │  A. (-1; 1)    B. (-∞; -1)                  │
   │  C. (1; +∞)    D. (-∞; -1) ∪ (1; +∞)       │
   │                                              │
   │  Câu 2. ...                                  │
   │                                              │
   │  ─────────── HẾT ───────────                 │
   └──────────────────────────────────────────────┘
   ```
   - CSS `@media print` cho layout 2 cột (A4)
   - Header tùy chỉnh: tên trường, sở, học kỳ, năm học
   - Công thức LaTeX render đẹp khi in
   - In nhiều trang nếu đề dài

2. **AnswerSheet.tsx** — Phiếu trả lời trắc nghiệm
   ```
   ┌───────────────────────────────────────┐
   │    PHIẾU TRẢ LỜI TRẮC NGHIỆM        │
   │                                       │
   │  Họ và tên: ________________________  │
   │  Lớp: ____  SBD: ____  Mã đề: ____   │
   │                                       │
   │  ┌─────────────────────────────────┐  │
   │  │ 01. ⓐ ⓑ ⓒ ⓓ │ 16. ⓐ ⓑ ⓒ ⓓ │  │
   │  │ 02. ⓐ ⓑ ⓒ ⓓ │ 17. ⓐ ⓑ ⓒ ⓓ │  │
   │  │ 03. ⓐ ⓑ ⓒ ⓓ │ 18. ⓐ ⓑ ⓒ ⓓ │  │
   │  │ ...            │ ...            │  │
   │  │ 15. ⓐ ⓑ ⓒ ⓓ │ 30. ⓐ ⓑ ⓒ ⓓ │  │
   │  └─────────────────────────────────┘  │
   └───────────────────────────────────────┘
   ```
   - Ô tròn (bubble) để tô
   - 2 cột cho đề 30 câu, 3 cột cho 50 câu
   - Có mã đề trên phiếu

3. **AnswerKeySheet.tsx** — Bảng đáp án cho giáo viên
   - Bảng: Mã đề | Câu 1 | Câu 2 | ... | Câu N
   - Tất cả mã đề trên 1 trang
   - **Chỉ in cho GV**, không phát cho HS

4. **Logic nhiều mã đề** (`print-utils.ts`):
   ```typescript
   function generateExamCodes(
     baseExam: Exam,
     codeCount: number         // VD: 4
   ): ExamCode[] {
     // Tạo mã: "132", "246", "358", "470"
     // Mỗi mã: shuffle câu hỏi + shuffle đáp án
     // Tính lại answer key cho từng mã
     // Return: 4 ExamCode objects
   }
   ```

5. **Print flow UI**:
   ```
   ConfigFormV2 (mode=print)
     → Chọn số mã đề (1-6)
     → Nhập thông tin trường, lớp
     → Tạo đề
     → Preview: Chọn mã đề để xem
     → Nút "In đề" / "In phiếu TL" / "In đáp án"
     → Ctrl+P → In từ browser
   ```

#### Files thay đổi:
- `[MỚI]` `src/components/PrintExam.tsx`
- `[MỚI]` `src/components/AnswerSheet.tsx`
- `[MỚI]` `src/components/AnswerKeySheet.tsx`
- `[MỚI]` `src/lib/print-utils.ts`
- `[SỬA]` `src/app/page.tsx` — Thêm print views
- `[SỬA]` `src/app/globals.css` — Thêm `@media print` styles

---

### Phase 5: Lưu đề & Quản lý
**Ưu tiên: 🟡 TRUNG BÌNH | Thời gian: ~2 giờ**

**Mục tiêu**: Lưu các đề đã tạo để sau này chấm bài hoặc in lại.

#### Công việc:

1. **ExamManager.tsx** — Dashboard quản lý
   - Danh sách đề đã tạo (card view)
   - Lọc: theo ngày, lớp, chuyên đề, trạng thái
   - Actions: xem lại / in lại / xóa / chấm bài

2. **API Routes** (`/api/exams`)
   - `POST /api/exams` → Lưu đề (bao gồm tất cả mã đề + answer keys)
   - `GET /api/exams` → Danh sách đề
   - `GET /api/exams/[id]` → Chi tiết 1 đề
   - `DELETE /api/exams/[id]` → Xóa

3. **Storage**: File JSON trong `data/exams/exam_[id].json`

4. **Tích hợp vào page.tsx**: View mới "exam-manager"

#### Files thay đổi:
- `[MỚI]` `src/components/ExamManager.tsx`
- `[MỚI]` `src/app/api/exams/route.ts`
- `[MỚI]` `src/app/api/exams/[id]/route.ts`
- `[SỬA]` `src/app/page.tsx` — Thêm view + nút truy cập

---

### Phase 6: OCR Nâng Cao (Quét đề + Quét phiếu)
**Ưu tiên: 🟢 THẤP | Thời gian: ~3-4 giờ**

**Mục tiêu**: 
- Upload ảnh **phiếu trả lời** đã tô → chấm điểm tự động.
- Upload ảnh **đề thi mới** → OCR → **lưu vào Question Bank** (giống cơ chế sinh đề AI).

#### Chế độ 1: Quét đề thi mới → Lưu vào Question Bank

Khi scan đề thi giấy, hệ thống sẽ xử lý **giống hệt pipeline sinh đề AI**:

```
Upload ảnh đề giấy
  ↓
Gemini Vision OCR → Nhận diện câu hỏi + đáp án
  ↓
Review / Chỉnh sửa → Gán metadata (chương, độ khó, chuyên đề)
  ↓
┌─────────────────────────────────────┐
│  TỰ ĐỘNG LƯU VÀO QUESTION BANK    │
│                                     │
│  Mỗi câu hỏi → BankQuestion {      │
│    source: "ocr",                   │
│    grade: ...,                      │
│    topic: ...,                      │
│    chapter: ...,                    │
│    difficulty: ...,                 │
│  }                                  │
│                                     │
│  → Có thể tái sử dụng khi tạo đề  │
│  → Có thể dùng Template thay số    │
└─────────────────────────────────────┘
  ↓
Tùy chọn: Làm bài ngay / Chỉ lưu vào kho
```

**Giao diện OCR scan đề sẽ thêm**:
- Bước gán metadata hàng loạt: chọn lớp, chuyên đề, chương cho toàn bộ đề
- Cho phép gán độ khó từng câu hoặc "auto" (AI tự đánh giá)
- Nút "Lưu vào kho" bên cạnh "Bắt đầu thi"
- Thông báo: "Đã lưu X câu vào Question Bank"

#### Chế độ 2: Quét phiếu trả lời → Chấm điểm

```
Chọn đề đã lưu → Upload ảnh phiếu → OCR nhận diện
  → Hiển thị: "Mã đề 132, 28/30 đúng, 9.3 điểm"
  → Chi tiết: câu nào đúng/sai
  → Lưu kết quả
```

#### Công việc:

1. **Nâng cấp OCRUpload.tsx** — 2 chế độ:
   - "Quét đề thi" → OCR + phân loại + lưu Question Bank
   - "Quét phiếu chấm" → Match đáp án + chấm điểm

2. **Metadata assignment UI** — Form gán thông tin cho câu hỏi scan:
   - Bulk assign: chọn lớp/chuyên đề/chương cho toàn đề
   - Per-question: gán độ khó riêng từng câu
   - AI auto-classify: gọi Gemini phân loại tự động

3. **Auto-save pipeline**: Sau khi confirm → `saveQuestions()` vào bank
   - Giống hệt luồng AI sinh đề (Phase 1)
   - Deduplicate (không lưu trùng)

4. **API**: Gọi Gemini Vision cho cả 2 chế độ

#### Files thay đổi:
- `[SỬA]` `src/components/OCRUpload.tsx` — Thêm 2 modes + metadata form
- `[MỚI]` `src/app/api/grade-scan/route.ts` — API chấm bài từ scan
- `[SỬA]` `src/lib/question-bank.ts` — Tích hợp auto-save từ OCR

---

### Phase 8: Smart Solve + Hệ thống Verification 5 Lớp
**Ưu tiên: 🔴 CAO | Thời gian: ~3 giờ**

**Mục tiêu**: Đảm bảo **mọi câu hỏi** (từ AI, OCR, template) đều có lời giải chính xác. Giảm tỷ lệ sai xuống mức thấp nhất.

#### Kiến trúc 5 lớp bảo vệ:

```
          CÂU HỎI MỚI (từ AI / OCR / Template)
                        ↓
  ┌─────────────────────────────────────────────┐
  │  LỚP 1: Self-Verify (2 lần gọi AI)         │
  │                                             │
  │  Gemini lần 1: "Giải bài này"               │
  │  Gemini lần 2: "Kiểm tra lời giải có đúng?" │
  │  → Khớp = ✅  |  Khác = ⚠️                  │
  │  → Giảm ~60-70% lỗi                        │
  └─────────────┬───────────────────────────────┘
                ↓
  ┌─────────────────────────────────────────────┐
  │  LỚP 2: Backward Check (thay ngược đáp án) │
  │                                             │
  │  PT: 2x²-5x+3=0 → AI: x=1, x=3/2          │
  │  Check: 2(1)²-5(1)+3 = 0 ✅                │
  │  Check: 2(3/2)²-5(3/2)+3 = 0 ✅            │
  │                                             │
  │  Áp dụng cho: PT, tính giới hạn, tích phân,│
  │  đạo hàm, hệ PT, tìm min/max...            │
  │  → Chính xác ~90% (dạng check được)         │
  └─────────────┬───────────────────────────────┘
                ↓
  ┌─────────────────────────────────────────────┐
  │  LỚP 3: So sánh Question Bank (RAG)        │
  │                                             │
  │  Tìm câu tương tự đã verified trong kho    │
  │  → Cùng dạng? Kết quả hợp lý?              │
  │  → Phát hiện bất thường (outlier)           │
  │  → Bank càng giàu → verify càng chính xác  │
  └─────────────┬───────────────────────────────┘
                ↓
  ┌─────────────────────────────────────────────┐
  │  LỚP 4: Rule-based Validation (code)       │
  │                                             │
  │  ✓ Đáp án đúng nằm trong 4 options          │
  │  ✓ 4 đáp án phải khác nhau                  │
  │  ✓ PT bậc 2: Δ ≥ 0 (nếu hỏi nghiệm thực)  │
  │  ✓ Phân số: mẫu ≠ 0                        │
  │  ✓ Lời giải chứa kết quả cuối              │
  │  ✓ LaTeX syntax hợp lệ                     │
  └─────────────┬───────────────────────────────┘
                ↓
  ┌─────────────────────────────────────────────┐
  │  LỚP 5: Confidence Score                   │
  │                                             │
  │  🟢 HIGH  (90%+): verify + backward + bank  │
  │  🟡 MEDIUM (60-90%): verify OK, no backward│
  │  🔴 LOW   (<60%): verify khác / dạng lạ    │
  │                                             │
  │  → 🟢 Dùng ngay                             │
  │  → 🟡 Dùng được, gợi ý GV kiểm tra         │
  │  → 🔴 Gắn tag "⚠️ Cần GV review"           │
  └─────────────────────────────────────────────┘
```

#### Data model bổ sung:

```typescript
// Thêm vào BankQuestion
interface BankQuestion {
  // ... (các field cũ)

  // --- Verification ---
  confidenceScore: number;              // 0-100
  confidenceLevel: "high" | "medium" | "low";
  verified: boolean;                    // GV đã xác nhận?
  verificationLog: VerificationStep[];  // Chi tiết 5 lớp
  needsReview: boolean;                 // Cần GV kiểm tra?
}

interface VerificationStep {
  layer: 1 | 2 | 3 | 4 | 5;
  name: string;                        // "Self-Verify", "Backward Check"...
  passed: boolean;
  detail: string;                      // Ghi chú chi tiết
}
```

#### Smart Solve (Tự giải câu mới):

```
Câu hỏi mới từ OCR (chưa có lời giải)
  ↓
Bước 1: Tìm câu tương tự trong Bank (có lời giải)
  ↓
Bước 2: Đưa câu tương tự làm few-shot example
         → Gemini: "Hãy giải câu mới theo phong cách này"
  ↓
Bước 3: Chạy 5 lớp verification
  ↓
Bước 4: Lưu câu + lời giải + confidence vào Bank
  ↓
📈 Bank càng giàu → few-shot càng tốt → giải càng chính xác
```

#### Công việc:

1. **verification.ts** — Engine kiểm tra 5 lớp
   - `selfVerify(question)` → Gọi Gemini lần 2 kiểm tra
   - `backwardCheck(question)` → Thay đáp án ngược vào đề
   - `bankComparison(question, bank)` → So sánh với câu tương tự
   - `ruleValidation(question)` → Kiểm tra quy tắc cố định
   - `calculateConfidence(results)` → Tính điểm tin cậy tổng

2. **smart-solve.ts** — Tự giải câu mới
   - `findSimilarQuestions(question, bank)` → Tìm câu giống trong kho
   - `generateSolution(question, examples)` → Gemini giải kèm few-shot
   - `solveAndVerify(question)` → Giải + verify đầy đủ 5 lớp

3. **UI hiển thị confidence**
   - Badge 🟢🟡🔴 trên mỗi câu hỏi
   - Tooltip: "Confidence 92% — Đã qua 5/5 lớp kiểm tra"
   - Trong Question Bank Browser: lọc theo confidence
   - Nút "GV Verify" → đánh dấu đã kiểm tra thủ công

#### Files thay đổi:
- `[MỚI]` `src/lib/verification.ts` — 5-layer verification engine
- `[MỚI]` `src/lib/smart-solve.ts` — Auto-solve + few-shot RAG
- `[SỬA]` `src/lib/ai-generator.ts` — Tích hợp verify sau khi sinh
- `[SỬA]` `src/lib/question-bank.ts` — Thêm confidence fields
- `[SỬA]` `src/components/OCRUpload.tsx` — Gọi Smart Solve cho câu chưa có lời giải
- `[SỬA]` `src/components/QuestionBankBrowser.tsx` — Hiển thị confidence badges

---

### Phase 7: Adaptive Difficulty (Tự điều chỉnh độ khó)
**Ưu tiên: 🟡 TRUNG BÌNH | Thời gian: ~2 giờ**

**Mục tiêu**: Hệ thống tự tăng/giảm độ khó dựa trên kết quả làm bài của người dùng. Người dùng có quyền bật/tắt tính năng này.

#### Cơ chế hoạt động:

```
┌─────────────────────────────────────────────────┐
│              ADAPTIVE DIFFICULTY                 │
│                                                 │
│  [ON/OFF Toggle] ← Người dùng chủ động bật/tắt │
│                                                 │
│  Khi BẬT:                                       │
│                                                 │
│  Làm bài xong → Hệ thống phân tích kết quả     │
│                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│  │ Score<40%│   │40%≤S≤75% │   │ Score>75%│    │
│  │ → Giảm 1 │   │ → Giữ    │   │ → Tăng 1 │    │
│  │   mức khó│   │ nguyên   │   │   mức khó│    │
│  └──────────┘   └──────────┘   └──────────┘    │
│                                                 │
│  Đề tiếp theo tự động áp dụng mức khó mới      │
│  Hiển thị badge: "Adaptive Mode ON"             │
└─────────────────────────────────────────────────┘
```

#### Data model:

```typescript
interface AdaptiveProfile {
  enabled: boolean;                    // Bật/tắt
  currentDifficulty: number;           // Mức khó hiện tại (1-5)
  history: AdaptiveEntry[];            // Lịch sử điều chỉnh
}

interface AdaptiveEntry {
  examId: string;
  score: number;                       // % đúng
  difficultyBefore: number;
  difficultyAfter: number;
  adjustedAt: string;
}
```

#### Quy tắc điều chỉnh:

| Điểm đạt được | Hành động | Ví dụ |
|---------------|-----------|-------|
| < 40% | Giảm 1 mức (min = 1) | Mức 3 → Mức 2 |
| 40% – 75% | Giữ nguyên | Mức 3 → Mức 3 |
| > 75% | Tăng 1 mức (max = 5) | Mức 3 → Mức 4 |
| > 90% liên tiếp 2 lần | Tăng 2 mức | Mức 2 → Mức 4 |
| < 30% liên tiếp 2 lần | Giảm 2 mức | Mức 4 → Mức 2 |

#### Công việc:

1. **AdaptiveToggle component** — Nút bật/tắt trên giao diện
   - Toggle switch đẹp, lưu trạng thái vào localStorage
   - Hiển thị mức khó hiện tại: "Mức 3 — Vận dụng"
   - History: "Lần trước: 82% → Tăng lên mức 4"

2. **Adaptive logic** (`src/lib/adaptive.ts`)
   - `evaluateAndAdjust(score, currentDifficulty, history)` → mức khó mới
   - `getAdaptiveProfile()` / `saveAdaptiveProfile()` — localStorage
   - Tích hợp vào luồng: Kết quả → Phân tích → Đề xuất mức khó mới

3. **Tích hợp vào ResultView.tsx**
   - Sau khi xem kết quả, hiển thị:
     ```
     📊 Adaptive Mode: Bạn đạt 85% → Đề tiếp theo sẽ ở mức 4 (Vận dụng cao)
     [Làm đề tiếp] ← auto-fill độ khó mới
     ```

4. **Tích hợp vào ConfigForm**
   - Khi Adaptive ON: ô "Độ khó" tự động fill theo profile
   - Tooltip: "Được điều chỉnh tự động dựa trên kết quả trước"
   - Người dùng vẫn có thể override thủ công

#### Files thay đổi:
- `[MỚI]` `src/lib/adaptive.ts` — Logic điều chỉnh độ khó
- `[MỚI]` `src/components/AdaptiveToggle.tsx` — Toggle UI
- `[SỬA]` `src/components/ResultView.tsx` — Hiển thị đề xuất mức khó mới
- `[SỬA]` `src/components/ConfigForm.tsx` — Auto-fill độ khó khi Adaptive ON
- `[SỬA]` `src/app/page.tsx` — Truyền adaptive state

---

### Phase 9: Cloud Database + Multi-user (Supabase)
**Ưu tiên: 🔴 CAO | Thời gian: ~3-4 giờ**

**Mục tiêu**: Chuyển từ local JSON sang **Supabase** (PostgreSQL cloud miễn phí) để nhiều người dùng cùng lúc, kho câu hỏi dùng chung và ngày càng giàu.

#### Tại sao Supabase?
- **Miễn phí**: 500MB database, 50K users/tháng, 1GB storage
- **Auth sẵn**: Đăng nhập Google/Email, 1 dòng code
- **PostgreSQL**: Mạnh, query linh hoạt, full-text search câu hỏi
- **Tích hợp Next.js**: SDK chính thức `@supabase/supabase-js`
- **Row Level Security**: GV chỉ sửa đề của mình, HS chỉ xem kết quả của mình

#### Database Schema:

```sql
-- Người dùng
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT UNIQUE NOT NULL,
  name TEXT,
  role TEXT CHECK (role IN ('teacher', 'student')) DEFAULT 'student',
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Kho câu hỏi (DÙNG CHUNG cho tất cả mọi người)
CREATE TABLE question_bank (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  content TEXT NOT NULL,
  options JSONB NOT NULL,               -- [{label, value}]
  correct_answer TEXT NOT NULL,
  solution TEXT,
  grade INTEGER NOT NULL,
  topic TEXT NOT NULL,
  chapter TEXT NOT NULL,
  difficulty INTEGER CHECK (difficulty BETWEEN 1 AND 5),
  skill_tags TEXT[],
  source TEXT CHECK (source IN ('ai', 'ocr', 'manual', 'template')),
  confidence_score INTEGER DEFAULT 0,
  confidence_level TEXT DEFAULT 'medium',
  verified BOOLEAN DEFAULT false,
  needs_review BOOLEAN DEFAULT false,
  contributed_by UUID REFERENCES users(id),
  usage_count INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Đề thi (đề của từng người dùng)
CREATE TABLE exams (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  title TEXT NOT NULL,
  config JSONB NOT NULL,                -- ExamConfigV2
  codes JSONB,                          -- ExamCode[]
  status TEXT DEFAULT 'draft',
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Kết quả thi (của từng người)
CREATE TABLE exam_results (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  exam_id UUID REFERENCES exams(id),
  score INTEGER,
  details JSONB,                        -- GradingResult
  time_taken INTEGER,
  tab_switches INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Adaptive profile (riêng từng người)
CREATE TABLE adaptive_profiles (
  user_id UUID PRIMARY KEY REFERENCES users(id),
  enabled BOOLEAN DEFAULT false,
  current_difficulty INTEGER DEFAULT 3,
  history JSONB DEFAULT '[]'
);
```

#### Phân quyền:

```
┌─────────────────────────────────────────┐
│  QUESTION BANK (Dùng chung)              │
│                                         │
│  GV: Đọc + Thêm + Sửa câu của mình     │
│  HS: Đọc + Thêm (qua OCR scan)          │
│  AI: Tự thêm khi sinh đề                │
│  → Càng nhiều người dùng = càng giàu   │
└─────────────────────────────────────────┘

┌───────────────────┐  ┌──────────────────┐
│  GIÁO VIÊN          │  │  HỌC SINH          │
│                   │  │                   │
│  ✓ Tạo đề + in   │  │  ✓ Làm bài online│
│  ✓ Nhiều mã đề   │  │  ✓ Xem kết quả  │
│  ✓ Chấm bài      │  │  ✓ Phân tích yếu  │
│  ✓ Quản lý đề    │  │  ✓ Adaptive mode│
│  ✓ Verify câu hỏi │  │  ✓ OCR scan đề  │
│  ✓ Xem thống kê  │  │  ✓ Lịch sử điểm │
└───────────────────┘  └──────────────────┘
```

#### Công việc:

1. **Setup Supabase project**
   - Tạo project miễn phí tại [supabase.com](https://supabase.com)
   - Tạo tables theo schema trên
   - Cấu hình Row Level Security

2. **Auth** (`src/lib/supabase.ts`)
   - Login với Google / Email
   - Middleware kiểm tra session
   - Redirect chưa login → trang đăng nhập

3. **Chuyển Question Bank từ JSON → Supabase**
   - Thay `fs.readFile/writeFile` bằng `supabase.from('question_bank').select/insert`
   - Thêm `contributed_by` = user_id của người tạo

4. **Chuyển Exams từ JSON → Supabase**
   - Lưu đề thi vào bảng `exams`
   - Kết quả vào bảng `exam_results`

5. **Chuyển Adaptive Profile từ localStorage → Supabase**
   - Bảng `adaptive_profiles` liên kết với user

6. **Auth UI** — Trang đăng nhập/đăng ký
   - "Tiếp tục với Google" button
   - Hoặc email/password
   - Chọn role: GV hay HS

#### Files thay đổi:
- `[MỚI]` `src/lib/supabase.ts` — Supabase client + helpers
- `[MỚI]` `src/components/AuthScreen.tsx` — Login/Register UI
- `[MỚI]` `src/middleware.ts` — Auth middleware
- `[SỬA]` `src/lib/question-bank.ts` — JSON → Supabase queries
- `[SỬA]` `src/lib/adaptive.ts` — localStorage → Supabase
- `[SỬA]` `src/app/page.tsx` — Thêm auth flow + role-based UI
- `[XÓA]` `src/data/` — Không cần file JSON local nữa

---

## 5. MVP Verification Gate

Trước khi một phase được coi là hoàn tất và được nối vào luồng chính, phải đi qua cổng xác minh MVP sau:

### 5.1. Gate bắt buộc
- `ConfigForm` validate đúng và không cho submit sai dữ liệu
- `/api/generate` trả response đúng schema hoặc fail rõ ràng
- Quiz timer chạy đúng, submit đúng, không vỡ khi refresh/đóng mở tab
- Kết quả chấm điểm đúng sau khi shuffle câu hỏi / shuffle đáp án
- Weakness view hiển thị đúng trạng thái heuristic hoặc AI mode
- OCR flow không được tự nhận là hoàn chỉnh nếu vẫn đang dùng mock

### 5.2. Manual E2E Checklist
1. Tạo đề từ form cấu hình cơ bản
2. Làm bài 1 phần, refresh trang, xác nhận recovery hoạt động
3. Submit bài, kiểm tra score và review đúng
4. Mở Weakness view, xác nhận khớp với câu sai
5. Upload ảnh đề, review OCR, confirm vào quiz
6. Kiểm tra retry / làm lại đề không làm hỏng state

### 5.3. Quy tắc release nội bộ
- Không merge phase mở rộng lớn nếu Phase 0 chưa pass gate
- Không đánh dấu `done` cho AI/OCR nếu vẫn dùng mock
- Mọi thay đổi lớn ở kiến trúc lưu trữ phải không làm hỏng flow MVP hiện tại

---

## 6. Dependencies cần thêm

```bash
npm install uuid @supabase/supabase-js @supabase/ssr
```

| Package | Mục đích | Có sẵn? |
|---------|---------|---------|
| `@ai-sdk/google` | Gemini API | ✅ Đã cài |
| `ai` | Vercel AI SDK | ✅ Đã cài |
| `zod` | Validation | ✅ Đã cài |
| `katex` | LaTeX render | ✅ Đã cài |
| `uuid` | Tạo ID unique | ❌ Cần cài |
| `@supabase/supabase-js` | Supabase client | ❌ Cần cài |
| `@supabase/ssr` | Supabase SSR (Next.js) | ❌ Cần cài |

> **Lưu trữ**: Supabase PostgreSQL (cloud, miễn phí 500MB)
> **Auth**: Supabase Auth (Google + Email)

---

## 7. Thứ tự triển khai

```
Phase 0 (Init Hardening)         ████░░░░░░░░░░░░░░░░░░░░░░  [Tuần 1] ← Chốt MVP trước
Phase 1+8 (API + Verify)         ░░░░██████████░░░░░░░░░░░░  [Tuần 1]
Phase 2 (Multi-chapter)          ░░░░░░░░██████████░░░░░░░░  [Tuần 1-2]
Phase 4 (Print + Mã đề)          ░░░░░░░░░░░░████████░░░░░░  [Tuần 2]
Phase 3 (Template)               ░░░░░░░░░░░░░░██████░░░░░░  [Tuần 2-3]
Phase 7 (Adaptive)               ░░░░░░░░░░░░░░░░██████░░░░  [Tuần 2-3]
Phase 5 (Exam Manager)           ░░░░░░░░░░░░░░░░░░░░██████  [Tuần 3]
Phase 6 (OCR Nâng Cao)           ░░░░░░░░░░░░░░░░░░░░░░████  [Tuần 3+]
Phase 9 (Supabase + Auth)        ░░░░░░░░░░░░░░░░░░████████  [Sau khi MVP pass gate]
```

**Thứ tự**: Phase 0 → 1+8 → 2 → 4 → 3 → 7 → 5 → 6 → 9

> Phase 0 phải làm **đầu tiên** để đóng hết gap của `init.md` và ổn định luồng MVP.
>
> Phase 9 chỉ nên triển khai sau khi MVP pass `MVP Verification Gate`, nhằm tránh việc mở rộng storage/auth quá sớm khi luồng cốt lõi còn chưa ổn định.

---

## 8. Quyết định đã xác nhận

| # | Câu hỏi | Trả lời |
|---|---------|---------|
| 1 | Gemini API Key | ✅ Có sẵn trong `.env` |
| 2 | Format đề in | ✅ Theo chuẩn THPT Quốc Gia |
| 3 | Số mã đề | ✅ Tùy người nhập (không giới hạn) |
| 4 | Adaptive Difficulty | ✅ Có nút bật/tắt, tự điều chỉnh |
| 5 | Verification | ✅ 5 lớp bảo vệ + Smart Solve |
| 6 | OCR scan đề | ✅ Câu scan tự lưu vào Bank |
| 7 | Lưu trữ | ✅ **Supabase cloud** — nhiều người dùng chung |
| 8 | Thứ tự | ✅ Phase 0 → 1+8 → 2 → 4 → 3 → 7 → 5 → 6 → 9 |
| 9 | Init gap handling | ✅ Bổ sung `Phase 0: Init Completion / MVP Hardening` + `MVP Verification Gate` |

> **Trạng thái: ĐÃ CẬP NHẬT — Sẵn sàng triển khai theo thứ tự mới**Ư
