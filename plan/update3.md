# Ke hoach `update3` - Phan tach ra sau `update2`

## 1. Muc dich

File nay gom cac hang muc duoc tach ra khoi `fix` vi khong phai blocker truoc `update2`, nhung van can de san pham day du hon sau `update2`.

Nguyen tac:
- nhung gi lam sai ket qua, sai data, vo flow chinh thi khong dua vao day
- nhung gi la nang cap trust, intelligence, cloud, va van hanh mo rong thi dua vao `update3`

---

## 2. Pham vi `update3`

`update3` se tap trung vao 5 nhom lon:

- verification va trust layer nang cao
- OCR nang cao va grade scan
- adaptive difficulty hoan chinh
- Supabase + multi-user + auth UI
- docs/ops/platform maturity

---

## 3. Nhom A - Verification 5 lop day du

Muc tieu:
- dua he thong tu muc "co generate" len muc "dang tin" hon

Cong viec:
- nang cap `app/src/lib/verification.ts` thanh 5-layer theo ke hoach:
  - Self-Verify
  - Backward Check
  - Bank Comparison
  - Rule Validation
  - Confidence Score
- luu metadata verify vao bank:
  - `confidenceScore`
  - `confidenceLevel`
  - `verified`
  - `needsReview`
  - `verificationLog`
- gan confidence vao pipeline AI/OCR/template

File lien quan:
- `app/src/lib/verification.ts`
- `app/src/lib/question-bank.ts`
- `app/src/lib/ai-generator.ts`

---

## 4. Nhom B - Smart Solve va trust UI

Cong viec:
- tao `app/src/lib/smart-solve.ts`
- tim cau tuong tu trong bank
- few-shot giai cau moi
- verify sau khi giai
- hien confidence badges trong UI
- them luong "GV verify"

File lien quan:
- `app/src/lib/smart-solve.ts`
- `app/src/components/QuestionBankBrowser.tsx`
- `app/src/components/OCRUpload.tsx`

---

## 5. Nhom C - OCR nang cao

Cong viec:
- nang `app/src/components/OCRUpload.tsx` thanh 2 mode day du:
  - quet de thi
  - quet phieu cham
- tao `app/src/app/api/grade-scan/route.ts`
- metadata assignment UI day du
- AI auto-classify neu tiep tuc scope nay
- save vao bank bang pipeline hoan chinh hon

Luu y:
- neu `fix` da chi can OCR MVP on dinh, thi phan OCR nang cao nay thuoc `update3`

---

## 6. Nhom D - Adaptive Difficulty hoan chinh

Cong viec:
- tao `app/src/components/AdaptiveToggle.tsx`
- nang cap `app/src/lib/adaptive.ts` theo rule day du hon
- hien history/rationale ro hon trong ket qua
- cho phep auto-fill/override ro rang hon trong config

File lien quan:
- `app/src/components/AdaptiveToggle.tsx`
- `app/src/lib/adaptive.ts`
- `app/src/components/ConfigForm.tsx`
- `app/src/components/ResultView.tsx`

---

## 7. Nhom E - Supabase + Multi-user + Auth

Cong viec:
- tao `app/src/components/AuthScreen.tsx`
- noi auth flow vao `app/src/app/page.tsx`
- chuyen bank/exams/adaptive len Supabase neu chot huong cloud
- role-based UI cho teacher/student
- chot middleware/session/ownership rules

File lien quan:
- `app/src/components/AuthScreen.tsx`
- `app/src/lib/supabase/*`
- `app/src/middleware.ts`
- `app/supabase/migration.sql`
- `app/src/lib/question-bank.ts`
- `app/src/lib/adaptive.ts`

---

## 8. Nhom F - Docs, migration, release maturity

Cong viec:
- them docs rieng:
  - `docs/manual-test-checklist.md`
  - `docs/release-gate.md`
  - `docs/data-cleanup-rules.md`
  - `docs/migration-plan.md`
  - `docs/product-scope.md`
- logging/ops/release checklist ro rang hon
- ke hoach migrate du lieu local -> cloud neu can

Luu y:
- nhom nay quan trong cho do truong thanh cua product, nhung khong can la blocker truoc `update2`

---

## 9. File/component/route thuoc `update3`

- `app/src/lib/smart-solve.ts`
- `app/src/components/QuestionBankBrowser.tsx`
- `app/src/components/AdaptiveToggle.tsx`
- `app/src/components/AuthScreen.tsx`
- `app/src/app/api/grade-scan/route.ts`
- docs bo sung trong `docs/*`

Va cac file can nang cap sau `update2`:
- `app/src/lib/verification.ts`
- `app/src/lib/question-bank.ts`
- `app/src/lib/ai-generator.ts`
- `app/src/components/OCRUpload.tsx`
- `app/src/lib/adaptive.ts`
- `app/src/app/page.tsx`
- `app/src/lib/supabase/*`

---

## 10. Dinh nghia xong `update3`

Chi duoc xem la xong `update3` khi:

- verification 5 lop da duoc trien khai that su, khong chi heuristic toi gian
- trust metadata da di xuyen suot qua bank va UI lien quan
- OCR nang cao/grade scan da chay duoc neu van giu trong scope
- adaptive da co toggle, history, va rule nang cao ro rang
- auth/cloud flow hoan chinh neu tiep tuc Phase 9
- docs van hanh/release/migration da co bo khung ro rang
