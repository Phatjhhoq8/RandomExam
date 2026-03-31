# Fix Plan - Bat buoc truoc `update2`

## 1. Muc dich

File nay chi gom nhung viec bat buoc phai xong truoc khi bat dau hoac chot `update2`.

Nguyen tac tach scope:
- `fix` = sua nen tang, bug, data, flow, gate, va cac phan dang do anh huong truc tiep product hien tai
- `update3` = cac nang cap sau `update2`, khong phai blocker de dua product len muc on dinh tiep theo

---

## 2. Dieu kien bat buoc truoc `update2`

Phai dat duoc toan bo cac nhom sau:

- khong con bug blocker gay sai dap an, sai state, sai luong chinh
- du lieu local khong con mau thuan ro rang trong bank/template
- code, README, va plan khong con overclaim lech nhau
- luong core cua hoc sinh va giao vien chay on dinh
- quality gate pass: `npm test`, `npm run lint`, `npm run build`

---

## 3. Nhom A - Bug blocker phai sua ngay

### A1. Template engine sinh sai dap an

Bat buoc fix:
- sua `app/src/lib/template-engine.ts`
- khong duoc co logic co dinh `correctAnswer: "A"`
- phai map lai dap an dung theo option da sinh that su
- phai co validation toi thieu de tranh sinh cau sai

Vi sao nam trong `fix`:
- day la bug logic anh huong truc tiep do dung cua he thong

### A2. Exam builder bo qua config nguon cau hoi

Bat buoc fix:
- sua `app/src/lib/exam-builder.ts`
- ton trong `questionSource`:
  - `ai`
  - `bank`
  - `mixed`
- khong duoc luu moi cau vao bank voi `source: "ai"` neu cau do den tu bank

Vi sao nam trong `fix`:
- day la bug logic va bug metadata, anh huong truc tiep de sinh ra va chat luong bank

### A3. Loi ky thuat tren bank API

Bat buoc fix:
- sua `app/src/app/api/bank/route.ts`
- import dung `crypto` neu tiep tuc dung `crypto.randomUUID()`
- dam bao route chay on dinh, khong loi runtime co ban

### A4. Loi flow/lint trong quiz va config

Bat buoc fix:
- sua `app/src/components/QuizEngine.tsx`
- sua `app/src/components/ConfigForm.tsx`
- loai bo logic de gay loi effect, submit flow, timer purity, setState khong dung cho

Vi sao nam trong `fix`:
- day la luong product chinh, khong the de loi roi nhay sang `update2`

---

## 4. Nhom B - Data cleanup bat buoc

### B1. Lam sach Question Bank

Bat buoc fix:
- audit `app/src/data/question-bank.json`
- sua cac ban ghi mau thuan giua:
  - `correctAnswer`
  - `solution`
  - options
- loai bo hoac danh dau cac cau khong the tin cay

### B2. Lam sach Template store

Bat buoc fix:
- audit `app/src/data/templates.json`
- sua/loai bo template duoc sinh tu du lieu sai
- dam bao `solutionTemplate` khop `contentTemplate`

### B3. Chan du lieu loi di vao pipeline

Bat buoc fix:
- bo sung validation truoc khi save vao bank
- bo sung validation truoc khi luu template
- khong tai su dung du lieu da biet la sai

Vi sao nam trong `fix`:
- neu khong lam sach data truoc `update2`, moi mo rong sau do deu co nguy co xay tren nen sai

---

## 5. Nhom C - Hardening luong MVP hien tai

### C1. Quiz recovery / retry / anti-cheat

Bat buoc fix:
- hoan thien recovery flow trong `app/src/components/QuizEngine.tsx` va `app/src/app/page.tsx`
- dam bao refresh/mo lai tab co the khoi phuc dung state can thiet
- dam bao retry khong hong state cu
- rasoat `beforeunload`, tab-switch warning, auto-submit
- xac nhan shuffle question/options khong lam sai grading

### C2. OCR va Weakness phai trung thuc voi thuc te

Bat buoc fix:
- cap nhat `app/src/components/OCRUpload.tsx` de:
  - co review/edit that su o muc MVP toi thieu, hoac
  - neu chua du thi ghi ro la partial/demo
- cap nhat `app/src/components/WeaknessView.tsx` de tach ro heuristic va AI mode
- khong de UI noi qua muc do hoan thien cua he thong

### C3. Math renderer / UI stack / docs phai chot 1 huong

Bat buoc fix:
- quyet dinh 1 huong duy nhat giua `react-katex` va KaTeX custom
- cap nhat `app/README.md` va ke hoach lien quan cho khop code that
- neu khong dung `shadcn/ui` thi khong nen tiep tuc mo ta nhu trong spec cu

Vi sao nam trong `fix`:
- day la MVP hardening, phai xong truoc khi mo them `update2`

---

## 6. Nhom D - Route/component dang do nhung can cho `update2`

Nhung muc nay du thuoc `update1`, nhung vi anh huong truc tiep product scope truoc `update2`, nen phai xong trong `fix`.

### D1. Difficulty distribution tach thanh component rieng

Bat buoc them:
- `app/src/components/DifficultyDistribution.tsx`

### D2. Exam management

Bat buoc them:
- `app/src/components/ExamManager.tsx`
- `app/src/app/api/exams/route.ts`
- `app/src/app/api/exams/[id]/route.ts`
- luu de vao storage dung nhu ke hoach dang ap dung

### D3. Bank single-item route

Bat buoc them:
- `app/src/app/api/bank/[id]/route.ts`

### D4. Print metadata flow

Bat buoc fix:
- noi form voi metadata in de that su
- xac nhan thong tin truong/lop/hoc ky/nam hoc di vao print layout

Vi sao nam trong `fix`:
- day la nhung phan truoc `update2` nen xong de product flow hien tai khong con dang do

---

## 7. Nhom E - Quality gate bat buoc

Phai dat duoc:

- `npm test` pass
- `npm run lint` pass
- `npm run build` pass

Phai kiem tra tay toi thieu:
- config -> generate -> quiz -> submit -> result -> weakness
- upload anh -> review -> confirm -> quiz
- refresh/reopen -> session recovery
- retry quiz khong hong state
- shuffle question/options khong sai grading
- generate print -> preview -> in de / in dap an

Neu chua dat, khong duoc xem la da xong `fix` va khong nen nhay sang `update2`.

---

## 8. Nhom F - Tai lieu va product honesty bat buoc

Bat buoc fix:
- cap nhat `app/README.md` cho dung voi code that
- bo huong dan sai ve `.env.example` neu file khong ton tai
- khong liet ke feature da implement neu no moi la partial/demo
- cap nhat lai wording tren UI neu can

Muc tieu:
- nguoi dung va team khong bi hieu nham rang he thong da hoan tat hon muc thuc te

---

## 9. Danh sach file chinh nam trong `fix`

- `app/src/lib/template-engine.ts`
- `app/src/lib/exam-builder.ts`
- `app/src/app/api/bank/route.ts`
- `app/src/app/api/bank/[id]/route.ts`
- `app/src/components/QuizEngine.tsx`
- `app/src/components/ConfigForm.tsx`
- `app/src/components/OCRUpload.tsx`
- `app/src/components/WeaknessView.tsx`
- `app/src/components/DifficultyDistribution.tsx`
- `app/src/components/ExamManager.tsx`
- `app/src/app/api/exams/route.ts`
- `app/src/app/api/exams/[id]/route.ts`
- `app/src/app/page.tsx`
- `app/src/lib/utils.ts`
- `app/src/data/question-bank.json`
- `app/src/data/templates.json`
- `app/README.md`

---

## 10. Dinh nghia xong `fix`

Chi duoc xem la xong `fix` khi:

- bug blocker da duoc sua
- data local da duoc lam sach toi thieu
- luong MVP chinh da on dinh
- route/component dang do can thiet truoc `update2` da co
- docs/UI khong con overclaim
- `npm test`, `npm run lint`, `npm run build` deu pass

Sau moc nay moi nen di tiep sang `update2`.
