# Fix Plan - bat buoc truoc `update2`

## 1. Muc tieu

File nay chi gom nhung viec bat buoc phai xong truoc khi chot va mo scope `update2`.

Nguyen tac tach scope:
- `fix` = sua bug logic, data, flow MVP, product honesty, va quality gate
- `update2` = auth, classroom, assignment, quota, billing, community solutions
- `update3` = nang cap sau `update2`, khong phai blocker de on dinh san pham hien tai

---

## 2. Dieu kien xong `fix`

Chi duoc xem la xong khi dat du cac nhom sau:
- khong con bug blocker gay sai dap an, sai source, sai state, sai route co ban
- local bank/template da duoc lam sach toi thieu va co gate chan du lieu loi moi
- flow MVP hien tai chay on dinh, khong overclaim so voi code that
- docs va UI wording khop voi muc do hoan thien that su
- quality gate pass: `npm test`, `npm run lint`, `npm run build`

---

## 3. Thu tu thuc hien de xuat

Lam theo thu tu nay de giam rui ro:

1. Sua blocker logic
2. Them validation gate va lam sach data local
3. Hardening flow quiz / recovery / OCR / weakness
4. Chot product honesty trong UI va README
5. Chay quality gate va test tay flow chinh

---

## 4. Nhom A - Blocker logic phai sua ngay

### A1. Template engine sinh sai dap an

Bat buoc fix:
- sua `app/src/lib/template-engine.ts`
- bo logic co dinh `correctAnswer: "A"`
- khi sinh option moi, phai xac dinh lai option nao trung voi dap an dung that su
- neu khong map duoc dap an an toan thi khong duoc tra ve cau hoi
- bo sung test cho truong hop option bi thay doi gia tri sau khi substitute

Vi sao nam trong `fix`:
- day la bug logic anh huong truc tiep do dung cua de thi

### A2. Exam builder bo qua `questionSource`

Bat buoc fix:
- sua `app/src/lib/exam-builder.ts`
- sua `app/src/components/ConfigFormV2.tsx`
- ton trong day du `questionSource`:
  - `ai`
  - `bank`
  - `mixed`
- chi save vao bank cac cau moi duoc tao that su, khong duoc danh dau cau bank thanh `source: "ai"`
- neu mode `bank` khong du cau thi phai bao loi ro rang, khong silently fallback sang AI

Vi sao nam trong `fix`:
- day la bug logic va bug metadata, anh huong truc tiep chat luong kho cau hoi

### A3. Loi ky thuat tren bank API

Bat buoc fix:
- sua `app/src/app/api/bank/route.ts`
- import dung `crypto` neu tiep tuc dung `crypto.randomUUID()`
- validate ky du lieu incoming, tranh route loi runtime va tranh luu cau hoi sai co ban

### A4. Quiz state / retry / submit can on dinh

Bat buoc fix:
- sua `app/src/components/QuizEngine.tsx`
- sua `app/src/app/page.tsx`
- dam bao auto-submit, timer, recovery, retry, beforeunload, tab-switch khong gay state loi hoac submit lap
- dam bao retry bat dau sach state moi, khong day lai exam da shuffle nhieu lan theo cach kho kiem soat

---

## 5. Nhom B - Validation gate va data cleanup bat buoc

### B1. Lam sach Question Bank

Bat buoc fix:
- audit `app/src/data/question-bank.json`
- sua cac ban ghi mau thuan giua:
  - `correctAnswer`
  - options
  - `solution`
- loai bo cac cau demo khong dang tin cay neu khong sua an toan duoc

### B2. Lam sach Template store

Bat buoc fix:
- audit `app/src/data/templates.json`
- sua/loai bo template sai do duoc trich tu bank loi
- dam bao `solutionTemplate` khop `contentTemplate` va dap an thuc te

### B3. Chan du lieu loi di vao pipeline

Bat buoc fix:
- bo sung validation truoc khi save vao bank trong `app/src/lib/question-bank.ts` va `app/src/app/api/bank/route.ts`
- bo sung validation truoc khi luu template trong `app/src/lib/template-engine.ts`
- OCR -> bank -> template extraction phai co gate toi thieu, khong tai su dung du lieu da biet la sai

Vi sao nam trong `fix`:
- neu khong chan du lieu sai tu goc, moi mo rong sau `update2` deu co nguy co xay tren nen sai

---

## 6. Nhom C - Hardening flow MVP hien tai

### C1. Session recovery / retry / anti-cheat

Bat buoc fix:
- hoan thien recovery flow trong `app/src/components/QuizEngine.tsx` va `app/src/app/page.tsx`
- refresh/mo lai tab phai khoi phuc dung state can thiet
- retry khong hong state cu va khong tu tang sai lich su adaptive
- xac nhan shuffle question/options khong lam sai grading

### C2. OCR va Weakness phai trung thuc voi thuc te

Bat buoc fix:
- cap nhat `app/src/components/OCRUpload.tsx` de co review/edit MVP that su, hoac ghi ro la review/delete only neu chua edit du
- OCR khong nen auto-day cau hoi chua verify vao bank theo cach overclaim
- cap nhat `app/src/components/WeaknessView.tsx` de tach ro heuristic va AI mode
- UI khong noi qua muc do hoan thien cua he thong

### C3. Adaptive chi thuoc practice mode

Bat buoc fix:
- flow hien tai phai ghi adaptive theo huong `self_practice`, khong de logic nay mo rong sai khi sang `update2`
- cap nhat wording va code lien quan neu can de tach ro practice flow voi flow khac

### C4. Print metadata flow va docs implementation

Bat buoc fix:
- noi metadata in de that su tu form vao preview in
- xac nhan thong tin truong/lop/hoc ky/nam hoc di vao print layout neu da claim co
- neu chua co field nao thi README/UI khong duoc claim qua muc thuc te

---

## 7. Nhom D - Product honesty, docs, va scope cleanup

### D1. README phai khop code that

Bat buoc fix:
- cap nhat `app/README.md` cho dung voi code that
- bo huong dan sai ve `.env.example` neu file khong ton tai
- khong liet ke feature da implement neu no moi la partial/demo

### D2. UI wording phai khop tinh trang that

Bat buoc fix:
- cap nhat wording tren home, OCR, weakness, dashboard, print neu can
- phan nao la fallback/mock/heuristic phai duoc noi ro

### D3. Route/component scope cleanup

Nguyen tac:
- chi giu trong `fix` nhung muc that su can de MVP chay on va de test
- `DifficultyDistribution.tsx` la refactor tu chon, khong con xem la blocker neu logic trong `ConfigFormV2` da du
- `app/src/app/api/bank/[id]/route.ts` la muc nen co de chuan hoa CRUD va test, nhung khong uu tien cao hon bug blocker neu flow hien tai van on

---

## 8. Nhom E - Quality gate bat buoc

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

Neu chua dat, khong duoc xem la da xong `fix`.

---

## 9. Danh sach file chinh trong `fix`

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

## 10. Dinh nghia xong `fix`

Chi duoc xem la xong `fix` khi:
- blocker logic da duoc sua
- local data da duoc lam sach toi thieu va co validation gate moi
- flow MVP chinh da on dinh
- docs/UI khong con overclaim
- quality gate va test tay flow chinh deu dat

Sau moc nay moi nen di tiep sang `update2`.
