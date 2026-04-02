# Fix Plan - bắt buộc trước `update2`

## 1. Mục tiêu

File này chỉ gồm những việc bắt buộc phải xong trước khi chốt và mở scope `update2`.

Nguyên tắc tách scope:
- `fix` = sửa bug logic, data, flow MVP, product honesty, và quality gate
- `update2` = auth, classroom, assignment, quota, billing, community solutions
- `update3` = nâng cấp sau `update2`, không phải blocker để ổn định sản phẩm hiện tại

---

## 2. Điều kiện xong `fix`

Chỉ được xem là xong khi đạt đủ các nhóm sau:
- không còn bug blocker gây sai đáp án, sai source, sai state, sai route cơ bản
- local bank/template đã được làm sạch tối thiểu và có gate chặn dữ liệu lỗi mới
- flow MVP hiện tại chạy ổn định, không overclaim so với code thật
- docs và UI wording khớp với mức độ hoàn thiện thật sự
- quality gate pass: `npm test`, `npm run lint`, `npm run build`

---

## 3. Thứ tự thực hiện đề xuất

Làm theo thứ tự này để giảm rủi ro:

1. Sửa blocker logic
2. Thêm validation gate và làm sạch data local
3. Hardening flow quiz / recovery / OCR / weakness
4. Chốt product honesty trong UI và README
5. Chạy quality gate và test tay flow chính

---

## 4. Nhóm A - Blocker logic phải sửa ngay

### A1. Template engine sinh sai đáp án

Bắt buộc fix:
- sửa `app/src/lib/template-engine.ts`
- bỏ logic cố định `correctAnswer: "A"`
- khi sinh option mới, phải xác định lại option nào trùng với đáp án đúng thật sự
- nếu không map được đáp án an toàn thì không được trả về câu hỏi
- bổ sung test cho trường hợp option bị thay đổi giá trị sau khi substitute

Vì sao nằm trong `fix`:
- đây là bug logic ảnh hưởng trực tiếp độ đúng của đề thi

### A2. Exam builder bỏ qua `questionSource`

Bắt buộc fix:
- sửa `app/src/lib/exam-builder.ts`
- sửa `app/src/components/ConfigFormV2.tsx`
- tôn trọng đầy đủ `questionSource`:
  - `ai`
  - `bank`
  - `mixed`
- chỉ save vào bank các câu mới được tạo thật sự, không được đánh dấu câu bank thành `source: "ai"`
- nếu mode `bank` không đủ câu thì phải báo lỗi rõ ràng, không silently fallback sang AI

Vì sao nằm trong `fix`:
- đây là bug logic và bug metadata, ảnh hưởng trực tiếp chất lượng kho câu hỏi

### A3. Lỗi kỹ thuật trên bank API

Bắt buộc fix:
- sửa `app/src/app/api/bank/route.ts`
- import đúng `crypto` nếu tiếp tục dùng `crypto.randomUUID()`
- validate kỹ dữ liệu incoming, tránh route lỗi runtime và tránh lưu câu hỏi sai cơ bản

### A4. Quiz state / retry / submit cần ổn định

Bắt buộc fix:
- sửa `app/src/components/QuizEngine.tsx`
- sửa `app/src/app/page.tsx`
- đảm bảo auto-submit, timer, recovery, retry, beforeunload, tab-switch không gây state lỗi hoặc submit lặp
- đảm bảo retry bắt đầu sạch state mới, không đẩy lại exam đã shuffle nhiều lần theo cách khó kiểm soát

---

## 5. Nhóm B - Validation gate và data cleanup bắt buộc

### B1. Làm sạch Question Bank

Bắt buộc fix:
- audit `app/src/data/question-bank.json`
- sửa các bản ghi mâu thuẫn giữa:
  - `correctAnswer`
  - options
  - `solution`
- loại bỏ các câu demo không đáng tin cậy nếu không sửa an toàn được

### B2. Làm sạch Template store

Bắt buộc fix:
- audit `app/src/data/templates.json`
- sửa/loại bỏ template sai do được trích từ bank lỗi
- đảm bảo `solutionTemplate` khớp `contentTemplate` và đáp án thực tế

### B3. Chặn dữ liệu lỗi đi vào pipeline

Bắt buộc fix:
- bổ sung validation trước khi save vào bank trong `app/src/lib/question-bank.ts` và `app/src/app/api/bank/route.ts`
- bổ sung validation trước khi lưu template trong `app/src/lib/template-engine.ts`
- OCR -> bank -> template extraction phải có gate tối thiểu, không tái sử dụng dữ liệu đã biết là sai

Vì sao nằm trong `fix`:
- nếu không chặn dữ liệu sai từ gốc, mọi mở rộng sau `update2` đều có nguy cơ xây trên nền sai

---

## 6. Nhóm C - Hardening flow MVP hiện tại

### C1. Session recovery / retry / anti-cheat

Bắt buộc fix:
- hoàn thiện recovery flow trong `app/src/components/QuizEngine.tsx` và `app/src/app/page.tsx`
- refresh/mở lại tab phải khôi phục đúng state cần thiết
- retry không hỏng state cũ và không tự tăng sai lịch sử adaptive
- xác nhận shuffle question/options không làm sai grading

### C2. OCR và Weakness phải trung thực với thực tế

Bắt buộc fix:
- cập nhật `app/src/components/OCRUpload.tsx` để có review/edit MVP thật sự, hoặc ghi rõ là review/delete only nếu chưa edit đủ
- OCR không nên auto-đẩy câu hỏi chưa verify vào bank theo cách overclaim
- cập nhật `app/src/components/WeaknessView.tsx` để tách rõ heuristic và AI mode
- UI không nói quá mức độ hoàn thiện của hệ thống

### C3. Adaptive chỉ thuộc practice mode

Bắt buộc fix:
- flow hiện tại phải ghi adaptive theo hướng `self_practice`, không để logic này mở rộng sai khi sang `update2`
- cập nhật wording và code liên quan nếu cần để tách rõ practice flow với flow khác

### C4. Print metadata flow va docs implementation

Bắt buộc fix:
- nối metadata in đề thật sự từ form vào preview in
- xác nhận thông tin trường/lớp/học kỳ/năm học đi vào print layout nếu đã claim có
- nếu chưa có field nào thì README/UI không được claim quá mức thực tế

---

## 7. Nhóm D - Product honesty, docs, và scope cleanup

### D1. README phai khop code that

Bat buoc fix:
- cập nhật `app/README.md` cho đúng với code thật
- bỏ hướng dẫn sai về `.env.example` nếu file không tồn tại
- không liệt kê feature đã implement nếu nó mới là partial/demo

### D2. UI wording phai khop tinh trang that

Bat buoc fix:
- cập nhật wording trên home, OCR, weakness, dashboard, print nếu cần
- phần nào là fallback/mock/heuristic phải được nói rõ

### D3. Route/component scope cleanup

Nguyên tắc:
- chỉ giữ trong `fix` những mục thật sự cần để MVP chạy ổn và để test
- `DifficultyDistribution.tsx` là refactor tùy chọn, không còn xem là blocker nếu logic trong `ConfigFormV2` đã đủ
- `app/src/app/api/bank/[id]/route.ts` là mục nên có để chuẩn hóa CRUD và test, nhưng không ưu tiên cao hơn bug blocker nếu flow hiện tại vẫn ổn

---

## 8. Nhóm E - Quality gate bắt buộc

Phải đạt được:
- `npm test` pass
- `npm run lint` pass
- `npm run build` pass

Phải kiểm tra tay tối thiểu:
- config -> generate -> quiz -> submit -> result -> weakness
- upload ảnh -> review -> confirm -> quiz
- refresh/reopen -> session recovery
- retry quiz không hỏng state
- shuffle question/options không sai grading
- generate print -> preview -> in đề / in đáp án

Nếu chưa đạt, không được xem là đã xong `fix`.

---

## 9. Danh sách file chính trong `fix`

- `app/src/lib/template-engine.ts`
- `app/src/lib/exam-builder.ts`
- `app/src/lib/question-bank.ts`
- `app/src/app/api/bank/route.ts`
- `app/src/components/QuizEngine.tsx`
- `app/src/components/ConfigFormV2.tsx`
- `app/src/components/OCRUpload.tsx`
- `app/src/components/WeaknessView.tsx`
- `app/src/app/page.tsx`
- `app/src/lib/utils.ts`
- `app/src/data/question-bank.json`
- `app/src/data/templates.json`
- `app/README.md`

---

## 10. Định nghĩa xong `fix`

Chỉ được xem là xong `fix` khi:
- blocker logic đã được sửa
- local data đã được làm sạch tối thiểu và có validation gate mới
- flow MVP chính đã ổn định
- docs/UI không còn overclaim
- quality gate và test tay flow chính đều đạt

Sau mốc này mới nên đi tiếp sang `update2`.
