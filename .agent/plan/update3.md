# Kế hoạch `update3` - Phân tách ra sau `update2`

## 1. Mục đích

File này gồm các hạng mục được tách ra khỏi `fix` vì không phải blocker trước `update2`, nhưng vẫn cần để sản phẩm đầy đủ hơn sau `update2`.

Nguyên tắc:
- những gì làm sai kết quả, sai data, vỡ flow chính thì không đưa vào đây
- những gì là nâng cấp trust, intelligence, cloud, và vận hành mở rộng thì đưa vào `update3`

---

## 2. Phạm vi `update3`

`update3` sẽ tập trung vào 5 nhóm lớn:

- verification và trust layer nâng cao
- OCR nâng cao và grade scan
- adaptive difficulty hoàn chỉnh
- classroom intelligence và teaching analytics nâng cao
- docs/ops/platform maturity

---

## 3. Nhóm A - Verification 5 lớp đầy đủ

Mục tiêu:
- đưa hệ thống từ mức "có generate" lên mức "đáng tin" hơn

Công việc:
- nâng cấp `app/src/lib/verification.ts` thành 5-layer theo kế hoạch:
  - Self-Verify
  - Backward Check
  - Bank Comparison
  - Rule Validation
  - Confidence Score
- lưu metadata verify vào bank:
  - `confidenceScore`
  - `confidenceLevel`
  - `verified`
  - `needsReview`
  - `verificationLog`
- gán confidence vào pipeline AI/OCR/template

File liên quan:
- `app/src/lib/verification.ts`
- `app/src/lib/question-bank.ts`
- `app/src/lib/ai-generator.ts`

---

## 4. Nhóm B - Smart Solve và trust UI

Công việc:
- tạo `app/src/lib/smart-solve.ts`
- tìm câu tương tự trong bank
- few-shot giải câu mới
- verify sau khi giải
- hiện confidence badges trong UI
- thêm luồng "GV verify"

File liên quan:
- `app/src/lib/smart-solve.ts`
- `app/src/components/QuestionBankBrowser.tsx`
- `app/src/components/OCRUpload.tsx`

---

## 5. Nhóm C - OCR nâng cao

Công việc:
- nâng `app/src/components/OCRUpload.tsx` thành 2 mode đầy đủ:
  - quét đề thi
  - quét phiếu chấm
- tạo `app/src/app/api/grade-scan/route.ts`
- metadata assignment UI đầy đủ
- AI auto-classify nếu tiếp tục scope này
- save vào bank bằng pipeline hoàn chỉnh hơn

Lưu ý:
- nếu `fix` đã chỉ cần OCR MVP ổn định, thì phần OCR nâng cao này thuộc `update3`

---

## 6. Nhóm D - Adaptive Difficulty hoàn chỉnh

Phạm vi:
- nhóm này chỉ mở rộng adaptive cho flow tự luyện cá nhân (`practice mode` / `self_practice`)
- không mở rộng adaptive để tự động chèn vào assignment/lớp học
- nếu cần intelligence cho lớp học thì thuộc nhóm classroom analytics, tách riêng với adaptive profile cá nhân

Công việc:
- tạo `app/src/components/AdaptiveToggle.tsx`
- nâng cấp `app/src/lib/adaptive.ts` theo rule đầy đủ hơn
- hiện history/rationale rõ hơn trong kết quả
- cho phép auto-fill/override rõ ràng hơn trong config
- bổ sung rule/filter rõ ràng để chỉ đọc và ghi adaptive records từ `self_practice`

File liên quan:
- `app/src/components/AdaptiveToggle.tsx`
- `app/src/lib/adaptive.ts`
- `app/src/components/ConfigForm.tsx`
- `app/src/components/ResultView.tsx`

---

## 7. Nhóm E - Classroom intelligence và analytics nâng cao

Phạm vi:
- nhóm này chỉ làm analytics cho classroom/assignment sau `update2`
- không được dùng classroom attempts để cập nhật adaptive difficulty cá nhân
- nếu cần xếp hạng học sinh, cảnh báo nguy cơ, hay gợi ý dạy học thì dùng snapshot/analytics riêng của lớp

Công việc:
- nâng cấp teacher dashboard sau `update2` với analytics sâu hơn
- thêm cảnh báo học sinh nguy cơ trượt deadline/điểm thấp
- thêm so sánh giữa các assignment, xu hướng theo chủ đề, và nhóm học sinh yếu
- cân nhắc notification/reminder cho assignment gần đến hạn
- tối ưu các snapshot/summary để dashboard lớp tải nhanh hơn

File liên quan:
- `app/src/components/TeacherDashboard.tsx`
- `app/src/components/AssignmentAnalytics.tsx`
- `app/src/lib/analytics.ts`
- `app/src/lib/classrooms.ts`

---

## 8. Nhóm F - Docs, migration, release maturity

Công việc:
- thêm docs riêng:
  - `docs/manual-test-checklist.md`
  - `docs/release-gate.md`
  - `docs/data-cleanup-rules.md`
  - `docs/migration-plan.md`
  - `docs/product-scope.md`
- logging/ops/release checklist rõ ràng hơn
- kế hoạch migrate dữ liệu local -> cloud nếu cần

Lưu ý:
- nhóm này quan trọng cho độ trưởng thành của product, nhưng không cần là blocker trước `update2`

---

## 9. File/component/route thuộc `update3`

- `app/src/lib/smart-solve.ts`
- `app/src/components/QuestionBankBrowser.tsx`
- `app/src/components/AdaptiveToggle.tsx`
- `app/src/app/api/grade-scan/route.ts`
- docs bổ sung trong `docs/*`

Và các file cần nâng cấp sau `update2`:
- `app/src/lib/verification.ts`
- `app/src/lib/question-bank.ts`
- `app/src/lib/ai-generator.ts`
- `app/src/components/OCRUpload.tsx`
- `app/src/lib/adaptive.ts`
- `app/src/app/page.tsx`
- `app/src/lib/supabase/*`

Lưu ý tách scope sau `update2`:
- `app/src/lib/adaptive.ts` tiếp tục phục vụ adaptive cho tự luyện cá nhân
- classroom analytics nếu cần thêm rule riêng thì ưu tiên đặt trong `app/src/lib/analytics.ts` hoặc module classroom liên quan, tránh trộn logic với adaptive profile cá nhân

---

## 10. Định nghĩa xong `update3`

Chỉ được xem là xong `update3` khi:

- verification 5 lớp đã được triển khai thật sự, không chỉ heuristic tối giản
- trust metadata đã đi xuyên suốt qua bank và UI liên quan
- OCR nâng cao/grade scan đã chạy được nếu vẫn giữ trong scope
- adaptive đã có toggle, history, và rule nâng cao rõ ràng cho flow tự luyện cá nhân
- classroom analytics nâng cao đã tách rõ khỏi MVP `update2`
- classroom analytics không làm thay đổi adaptive difficulty cá nhân
- docs vận hành/release/migration đã có bộ khung rõ ràng
