# Báo cáo toàn bộ những gì đã làm trong repo hiện tại

## Phạm vi

- Báo cáo này chỉ quét trong phạm vi repo hiện tại: `C:\Users\Admin\Desktop\RandomEx`.
- Phần được tính gồm cả repo con `app/` vì đây là nơi chứa toàn bộ mã nguồn ứng dụng.
- Mục tiêu của file này là ghi lại **từ lúc khởi tạo đến trạng thái hiện nay** những gì có thể xác minh được bằng lịch sử git, file kế hoạch, cấu trúc mã nguồn, test, docs, migration, dữ liệu và trạng thái workspace.

## Kết luận ngắn

- Dự án bắt đầu từ một app Next.js khởi tạo cơ bản.
- Sau đó repo được mở rộng thành một hệ thống MathQuiz AI khá lớn, gồm: sinh đề, làm bài online, OCR, question bank, template engine, adaptive, verification, auth, classroom, assignment, billing, quota, community solutions, Supabase, analytics, test và docs vận hành.
- Repo ngoài `RandomEx` chủ yếu lưu lịch sử **quy hoạch sản phẩm, tách phase và tiêu chí hoàn thành**.
- Repo con `app/` chứa phần lớn phần mềm đang phát triển; hiện tại phần lớn thay đổi ở đây vẫn là **chưa commit**.

## 1. Dòng thời gian từ khởi tạo tới hiện nay

### Giai đoạn khởi tạo ứng dụng

Nguồn:
- `app/.git`
- `C:\Users\Admin\Desktop\RandomEx\.git\logs\HEAD`

Xác minh được:
- `2026-03-31 12:54:26 +0700` - repo `app/` có commit `bcbd38c` với message `Initial commit from Create Next App`.
- Trạng thái này cho thấy dự án bắt đầu từ bộ khởi tạo Next.js mặc định, với các file nền như `package.json`, `src/app/page.tsx`, `src/app/layout.tsx`, `src/app/globals.css`, `public/*`.

### Giai đoạn tạo repo quản lý kế hoạch bên ngoài

Nguồn:
- `C:\Users\Admin\Desktop\RandomEx\.git\logs\HEAD`
- `git log` của repo ngoài

Xác minh được:
- `2026-04-01 01:03:56 +0700` - commit `026eb81` tạo repo ngoài để quản lý kế hoạch và workspace tổng.
- Commit này thêm:
  - `.cursorrules`
  - `.gitignore`
  - `plan/init.md`
  - `plan/update1.md`
  - gitlink `app`

Ý nghĩa:
- Từ thời điểm này dự án không còn chỉ là một app demo, mà đã có lớp quản lý roadmap và định nghĩa sản phẩm riêng.

### Giai đoạn mở rộng kế hoạch và tách phase lớn

Nguồn:
- `git log` repo ngoài
- `plan/update1.md`
- `plan/update2.md`
- `plan/update3.md`
- `plan/fix.md`

Xác minh được:
- `2026-04-01 03:48:59 +0700` - commit `938f9ec`
- `2026-04-02 22:24:34 +0700` - commit `91ba59b`
- `2026-04-03 00:51:32 +0700` - commit `81b74ef`

Những gì đã xảy ra ở lớp kế hoạch:
- bổ sung và mở rộng mạnh `update1`, `update2`, `update3`
- xây `fix` như một phase riêng để xử lý blocker trước khi mở rộng tiếp
- đổi `fix.md` cũ ở root sang `plan/fix.md`

### Giai đoạn phát triển mã nguồn ứng dụng hiện tại

Nguồn:
- `git status` của repo `app/`
- cấu trúc thư mục hiện tại trong `app/src/app/api/`, `app/src/components/`, `app/src/lib/`, `app/src/__tests__/`

Xác minh được:
- so với commit khởi tạo, repo `app/` hiện đã có rất nhiều route API, component, thư viện nghiệp vụ, test, dữ liệu, docs, migration và log phát triển
- phần lớn thay đổi này vẫn đang ở trạng thái làm việc hiện tại, chưa có commit mới trong repo `app/`

## 2. Những gì đã làm ở cấp kế hoạch và định nghĩa sản phẩm

### `plan/init.md` - xác lập MVP gốc

Đã định nghĩa:
- hệ thống tạo đề toán bằng AI
- quiz online có timer
- auto grading
- weakness detection
- OCR từ ảnh đề giấy
- anti-cheat
- chiến lược verification, cache, testing, data strategy

### `plan/update1.md` - mở rộng từ MVP sang nền tảng ra đề hoàn chỉnh hơn

Đã bổ sung kế hoạch cho:
- Gemini API thật
- question bank
- template engine thay số
- multi-chapter config
- difficulty distribution
- print mode và nhiều mã đề
- exam manager
- Supabase auth/storage
- verification 5 lớp
- adaptive difficulty
- migration từ local sang cloud

### `plan/update2.md` - mở rộng sang nền tảng nhiều người dùng

Đã xác lập scope cho:
- auth thật
- teacher/student role
- classroom
- assignment
- anti-cheat theo assignment
- billing và quota
- Prime subscription
- payment order / transaction
- scan grading quota
- community solutions
- admin override và audit

### `plan/update3.md` - tách các hạng mục nâng cao sau `update2`

Đã tách riêng:
- trust layer nâng cao
- smart solve
- OCR nâng cao
- adaptive polish
- classroom analytics nâng cao
- docs / ops / release maturity

### `plan/fix.md` - gom các blocker phải xử lý trước khi đi tiếp

Đã chốt các nhóm việc bắt buộc:
- sửa blocker logic
- làm sạch bank/template data
- chặn dữ liệu lỗi vào pipeline
- hardening quiz / retry / recovery / OCR / weakness
- đồng bộ docs và wording với thực tế
- quality gate: test, lint, build

## 3. Những gì đã làm trong ứng dụng `app/` theo từng lớp chức năng

## 3.1. Lớp giao diện và luồng người dùng chính

Nguồn:
- `app/src/components/`
- `app/src/app/page.tsx`

Các màn hình và component đã có:
- `ConfigForm.tsx`
- `ConfigFormV2.tsx`
- `QuizEngine.tsx`
- `ResultView.tsx`
- `WeaknessView.tsx`
- `OCRUpload.tsx`
- `PrintExam.tsx`
- `AnswerSheet.tsx`
- `AnswerKeySheet.tsx`
- `QuestionBankBrowser.tsx`
- `Dashboard.tsx`
- `LoadingScreen.tsx`
- `MathRenderer.tsx`
- `AdaptiveToggle.tsx`

Điều này cho thấy đã làm:
- form cấu hình đề cơ bản và nâng cao
- luồng làm bài online
- màn hình kết quả
- màn hình phân tích điểm yếu
- upload OCR
- giao diện in đề, phiếu trả lời, đáp án
- duyệt kho câu hỏi
- adaptive UI

## 3.2. Sinh đề, ngân hàng câu hỏi, template, lịch sử đề

Nguồn:
- `app/src/app/api/generate/route.ts`
- `app/src/app/api/generate-v2/route.ts`
- `app/src/app/api/bank/route.ts`
- `app/src/lib/ai-generator.ts`
- `app/src/lib/question-bank.ts`
- `app/src/lib/template-engine.ts`
- `app/src/lib/exam-builder.ts`
- `app/src/lib/exam-history.ts`
- `app/src/data/question-bank.json`
- `app/src/data/templates.json`

Đã làm:
- API sinh đề bản cơ bản và bản V2
- question bank local
- template engine
- exam builder
- lưu lịch sử đề
- data store cục bộ cho bank/template

## 3.3. Verification, weakness, trust, smart solve

Nguồn:
- `app/src/app/api/weakness/route.ts`
- `app/src/lib/verification.ts`
- `app/src/lib/weakness-ai.ts`
- `app/src/lib/smart-solve.ts`
- `app/docs/data-cleanup-rules.md`
- `app/docs/product-scope.md`

Đã làm:
- route weakness analysis
- module verification
- module weakness AI
- module smart solve
- quy tắc cleanup dữ liệu và tách canonical/community solutions

## 3.4. Auth, phân quyền, hồ sơ người dùng

Nguồn:
- `app/src/app/api/auth/route.ts`
- `app/src/app/api/me/route.ts`
- `app/src/lib/auth.ts`
- `app/src/lib/roles.ts`
- `app/src/components/AuthScreen.tsx`

Đã làm:
- signup / login / logout
- route lấy session hiện tại
- logic role teacher/student
- UI đăng nhập/đăng ký

## 3.5. Classroom, assignment, teacher/student dashboard

Nguồn:
- `app/src/app/api/classrooms/route.ts`
- `app/src/app/api/classrooms/join/route.ts`
- `app/src/app/api/classrooms/[id]/route.ts`
- `app/src/app/api/classrooms/[id]/analytics/route.ts`
- `app/src/app/api/exam-templates/route.ts`
- `app/src/app/api/assignments/route.ts`
- `app/src/app/api/assignments/[id]/route.ts`
- `app/src/app/api/assignments/[id]/start/route.ts`
- `app/src/app/api/assignments/[id]/submit/route.ts`
- `app/src/app/api/assignments/[id]/schedule/route.ts`
- `app/src/app/api/assignments/[id]/force-submit/route.ts`
- `app/src/components/TeacherDashboard.tsx`
- `app/src/components/StudentDashboard.tsx`
- `app/src/components/ClassroomList.tsx`
- `app/src/components/ClassroomDetail.tsx`
- `app/src/components/CreateClassroomForm.tsx`
- `app/src/components/JoinClassroomForm.tsx`
- `app/src/components/AssignmentBuilder.tsx`
- `app/src/components/AssignmentList.tsx`
- `app/src/components/AssignmentAnalytics.tsx`
- `app/src/components/AssignmentScheduleEditor.tsx`
- `app/src/components/AssignmentForceSubmitDialog.tsx`
- `app/src/lib/classrooms.ts`
- `app/src/lib/assignments.ts`
- `app/src/lib/analytics.ts`

Đã làm:
- quản lý lớp học
- tham gia lớp bằng mã
- tạo exam template
- tạo và publish assignment
- start / submit assignment
- sửa lịch assignment
- force submit assignment
- teacher dashboard / student dashboard
- analytics lớp và assignment

## 3.6. Billing, quota, Prime, payment, usage tracking

Nguồn:
- `app/src/app/api/billing/plans/route.ts`
- `app/src/app/api/billing/subscription/route.ts`
- `app/src/app/api/billing/usage/route.ts`
- `app/src/app/api/billing/create-order/route.ts`
- `app/src/app/api/billing/recheck-order/route.ts`
- `app/src/app/api/billing/webhook/route.ts`
- `app/src/app/api/admin/subscriptions/override/route.ts`
- `app/src/app/api/admin/usage/adjust/route.ts`
- `app/src/app/api/admin/payments/mark-paid/route.ts`
- `app/src/components/BillingDashboard.tsx`
- `app/src/components/PricingCard.tsx`
- `app/src/components/UpgradePrimeModal.tsx`
- `app/src/components/UsageQuotaCard.tsx`
- `app/src/lib/billing.ts`
- `app/src/lib/subscription.ts`
- `app/src/lib/usage-tracker.ts`
- `app/src/lib/admin-audit.ts`

Đã làm:
- xem plan/subscription/usage
- tạo đơn hàng nâng cấp Prime
- recheck/webhook thanh toán
- admin override cho subscription, usage và mark-paid
- UI quota và paywall
- audit cho thao tác admin

## 3.7. OCR, scan grading, manual review path

Nguồn:
- `app/src/app/api/ocr/route.ts`
- `app/src/app/api/scan-grade/route.ts`
- `app/src/app/api/grade-scan/route.ts`
- `app/src/components/OCRUpload.tsx`
- `app/src/lib/ocr-engine.ts`
- `app/src/lib/scan-grading.ts`
- `app/docs/manual-test-checklist.md`
- `app/docs/release-gate.md`

Đã làm:
- OCR route
- scan grade route
- grade-scan route
- component OCR upload
- thư viện scan grading
- checklist cho success path và manual review path

## 3.8. Community solutions và moderation

Nguồn:
- `app/src/app/api/community-solutions/route.ts`
- `app/src/app/api/community-solutions/[id]/vote/route.ts`
- `app/src/app/api/solution-issue-reports/route.ts`
- `app/src/app/api/admin/community-solutions/route.ts`
- `app/src/app/api/admin/community-solutions/[id]/route.ts`
- `app/src/app/api/admin/solution-issue-reports/route.ts`
- `app/src/app/api/admin/solution-issue-reports/[id]/route.ts`
- `app/src/components/CommunitySolutionsList.tsx`
- `app/src/components/CommunitySolutionComposer.tsx`
- `app/src/components/SolutionModerationPanel.tsx`
- `app/src/lib/community-solutions.ts`

Đã làm:
- gửi lời giải cộng đồng
- vote lời giải
- báo lỗi lời giải
- moderation dashboard/panel
- workflow admin duyệt hoặc từ chối

## 3.9. Supabase, platform store, repository abstraction, migration

Nguồn:
- `app/src/lib/supabase/client.ts`
- `app/src/lib/supabase/server.ts`
- `app/src/lib/supabase/middleware.ts`
- `app/src/lib/supabase/bank.ts`
- `app/src/lib/platform-model.ts`
- `app/src/lib/platform-repository.ts`
- `app/src/lib/platform-store.ts`
- `app/src/data/platform-store.json`
- `app/supabase/migration.sql`
- `app/supabase/migrations/20260402_update2_update3.sql`
- `app/docs/supabase-setup.md`
- `app/docs/migration-plan.md`

Đã làm:
- helper client/server/middleware cho Supabase
- abstraction repository/platform store
- local platform store JSON
- migration SQL cho schema lớn
- tài liệu setup và kế hoạch chuyển nguồn dữ liệu từ local sang Supabase

## 3.10. Test, checklist, release gate, docs vận hành

Nguồn:
- `app/src/__tests__/adaptive.test.ts`
- `app/src/__tests__/exam-builder.test.ts`
- `app/src/__tests__/session.test.ts`
- `app/src/__tests__/template-engine.test.ts`
- `app/src/__tests__/update2-platform.test.ts`
- `app/src/__tests__/utils.test.ts`
- `app/src/__tests__/verification.test.ts`
- `app/docs/manual-test-checklist.md`
- `app/docs/release-gate.md`
- `app/docs/data-cleanup-rules.md`
- `app/docs/product-scope.md`
- `app/docs/migration-plan.md`
- `app/docs/supabase-setup.md`

Đã làm:
- unit test cho adaptive, exam builder, session, template engine, verification, utils, platform/update2
- checklist test tay cho auth, teacher flow, student flow, billing, community solutions, scan grading, practice mode
- release gate yêu cầu lint, test, build và verify các luồng chính
- docs cleanup, product scope, migration và setup Supabase

## 4. Những file và thư mục lớn đã xuất hiện thêm so với lúc khởi tạo

### API routes hiện có

Nguồn:
- `app/src/app/api`

Đã có các nhóm route:
- `admin/`
- `assignments/`
- `auth/`
- `bank/`
- `billing/`
- `classrooms/`
- `community-solutions/`
- `exam-templates/`
- `generate/`
- `generate-v2/`
- `grade-scan/`
- `me/`
- `ocr/`
- `scan-grade/`
- `solution-issue-reports/`
- `weakness/`

### Component hiện có

Nguồn:
- `app/src/components`

Đã có 33 component, gồm các nhóm chính:
- quiz / result / weakness / OCR
- config / adaptive / print
- auth / billing
- classroom / assignment / dashboard
- community solutions / moderation

### Thư viện nghiệp vụ hiện có

Nguồn:
- `app/src/lib`

Đã có 29 module hoặc nhóm module, gồm:
- AI generation
- adaptive
- analytics
- assignments
- auth
- billing
- classrooms
- community solutions
- exam builder
- OCR / scan grading
- platform repository/store
- question bank
- template engine
- subscription / usage tracking
- verification / weakness / smart solve
- Supabase helpers

### Test hiện có

Nguồn:
- `app/src/__tests__`

Đã có 7 file test:
- `adaptive.test.ts`
- `exam-builder.test.ts`
- `session.test.ts`
- `template-engine.test.ts`
- `update2-platform.test.ts`
- `utils.test.ts`
- `verification.test.ts`

## 5. Dấu vết vận hành và kiểm tra chất lượng đã làm

Nguồn:
- `app/dev-server.log`
- `app/dev-server-3001.log`
- `app/eslint-output.txt`
- `app/eslint_errors.txt`
- `app/eslint_errors2.txt`
- `app/eslint_out.txt`
- `app/eslint_quiet.txt`
- `app/eslint_readable.txt`
- `app/eslint_short.txt`
- `app/lint_output.txt`
- `app/lint_filtered.txt`
- `app/lint_report.json`
- `app/lint_results.json`
- `app/lint_summary.json`
- `app/eslint.json`

Xác minh được:
- đã từng chạy dev server ở ít nhất hai cổng hoặc hai lần chạy khác nhau
- đã từng chạy lint/eslint và xuất báo cáo ra nhiều định dạng khác nhau
- đã có ý thức kiểm tra chất lượng và đọc lỗi lint thay vì chỉ viết code

## 6. Trạng thái hiện tại của workspace

### Repo ngoài `RandomEx`

Nguồn:
- `git status --short`

Hiện tại:
- `app` đang modified
- `fix.md` ở root đã bị xóa
- `plan/fix.md` đang untracked
- file báo cáo hiện tại `repo-activity-report.md` cũng là sản phẩm mới trong workspace

### Repo con `app`

Nguồn:
- `git -C app status --short`

Hiện tại đang có:
- sửa các file nền:
  - `.gitignore`
  - `README.md`
  - `package.json`
  - `package-lock.json`
  - `src/app/globals.css`
  - `src/app/layout.tsx`
  - `src/app/page.tsx`
- thêm mới hàng loạt file trong:
  - `docs/`
  - `src/__tests__/`
  - `src/app/api/`
  - `src/components/`
  - `src/data/`
  - `src/lib/`
  - `supabase/`
- thêm file vận hành cục bộ như:
  - `cookies.txt`
  - `student-cookies.txt`
  - các file log và report lint

Ý nghĩa:
- phần lớn công việc phát triển ứng dụng đang nằm trong working tree hiện tại
- repo `app/` chưa phản ánh khối lượng công việc này bằng commit lịch sử riêng

## 7. Tổng hợp “đã làm gì từ khởi tạo tới nay”

- Khởi đầu từ một app Next.js mặc định.
- Tạo repo quản lý kế hoạch bên ngoài để theo dõi roadmap và scope.
- Viết tài liệu gốc cho MVP MathQuiz AI.
- Mở rộng kế hoạch sang bank, template, print, adaptive, verification, Supabase.
- Mở rộng tiếp sang auth, lớp học, assignment, billing, quota, Prime, payment, community solutions, scan grading.
- Tách tiếp các hạng mục nâng cao như smart solve, OCR nâng cao, analytics nâng cao, release maturity.
- Thêm nhiều route API phục vụ generate, bank, auth, classrooms, assignments, billing, OCR, scan, community và admin.
- Thêm nhiều component cho quiz, result, weakness, OCR, print, auth, billing, classroom, assignment, moderation.
- Thêm nhiều module nghiệp vụ cho AI generation, verification, question bank, template engine, assignments, billing, usage, analytics, Supabase, community.
- Thêm test, checklist và docs vận hành.
- Tạo dữ liệu local, migration SQL, log dev và báo cáo lint.
- Hiện tại hệ thống vẫn đang trong trạng thái phát triển tiếp, với rất nhiều thay đổi chưa commit ở repo `app/`.

## 8. Danh sách nguồn chính đã dùng để lập báo cáo

- `plan/init.md`
- `plan/update1.md`
- `plan/update2.md`
- `plan/update3.md`
- `plan/fix.md`
- `app/README.md`
- `app/docs/manual-test-checklist.md`
- `app/docs/release-gate.md`
- `app/docs/data-cleanup-rules.md`
- `app/docs/product-scope.md`
- `app/docs/migration-plan.md`
- `app/docs/supabase-setup.md`
- `app/src/app/api/`
- `app/src/components/`
- `app/src/lib/`
- `app/src/__tests__/`
- `app/supabase/migrations/20260402_update2_update3.sql`
- `app/src/data/question-bank.json`
- `app/src/data/templates.json`
- `app/src/data/platform-store.json`

## 9. Các file chính và vai trò của từng file/thư mục

### 9.1. Repo ngoài `RandomEx`

- `plan/init.md` - tài liệu kế hoạch MVP gốc của MathQuiz AI.
- `plan/update1.md` - kế hoạch mở rộng sau MVP: bank, template, print, adaptive, Supabase, verification.
- `plan/update2.md` - kế hoạch mở rộng sang auth, classroom, assignment, quota, billing, community solutions.
- `plan/update3.md` - kế hoạch cho các hạng mục nâng cao sau `update2` như trust, OCR nâng cao, analytics, release maturity.
- `plan/fix.md` - danh sách bug blocker, cleanup và quality gate phải xử lý trước khi đi tiếp.
- `.cursorrules` - luật/định hướng làm việc cho agent trong repo ngoài.
- `.gitignore` - cấu hình các file/thư mục không đưa vào git.
- `repo-activity-report.md` - báo cáo tổng hợp những gì đã làm trong repo hiện tại.

### 9.2. File gốc của ứng dụng `app/`

- `app/package.json` - khai báo dependencies, scripts và cấu hình package của ứng dụng.
- `app/package-lock.json` - khóa phiên bản dependency đã cài.
- `app/README.md` - mô tả sản phẩm, tính năng, cách chạy và stack hiện tại.
- `app/next.config.ts` - cấu hình Next.js.
- `app/tsconfig.json` - cấu hình TypeScript.
- `app/eslint.config.mjs` - cấu hình ESLint.
- `app/postcss.config.mjs` - cấu hình PostCSS.
- `app/vitest.config.ts` - cấu hình Vitest.
- `app/AGENTS.md` - hướng dẫn/ghi chú dành cho agent trong repo `app/`.
- `app/CLAUDE.md` - file hướng dẫn ngắn cho môi trường agent.

### 9.3. App shell và entrypoint

- `app/src/app/page.tsx` - trang chính, điểm vào của SPA/luồng ứng dụng.
- `app/src/app/layout.tsx` - layout gốc của ứng dụng.
- `app/src/app/globals.css` - CSS toàn cục và style nền.
- `app/src/app/favicon.ico` - favicon của ứng dụng.

### 9.4. API routes và mục đích

- `app/src/app/api/generate/route.ts` - API sinh đề bản cơ bản.
- `app/src/app/api/generate-v2/route.ts` - API sinh đề nâng cao/multi-chapter.
- `app/src/app/api/bank/route.ts` - API thao tác với question bank.
- `app/src/app/api/weakness/route.ts` - API phân tích điểm yếu/weakness.
- `app/src/app/api/auth/route.ts` - API đăng ký, đăng nhập, đăng xuất.
- `app/src/app/api/me/route.ts` - API lấy session/profile hiện tại.
- `app/src/app/api/classrooms/route.ts` - API danh sách/tạo lớp học.
- `app/src/app/api/classrooms/join/route.ts` - API tham gia lớp bằng mã.
- `app/src/app/api/classrooms/[id]/route.ts` - API chi tiết một lớp học.
- `app/src/app/api/classrooms/[id]/analytics/route.ts` - API analytics cho lớp học.
- `app/src/app/api/exam-templates/route.ts` - API lưu/lấy template cấu trúc đề.
- `app/src/app/api/assignments/route.ts` - API danh sách/tạo assignment.
- `app/src/app/api/assignments/[id]/route.ts` - API chi tiết assignment.
- `app/src/app/api/assignments/[id]/start/route.ts` - API bắt đầu làm assignment.
- `app/src/app/api/assignments/[id]/submit/route.ts` - API nộp bài assignment.
- `app/src/app/api/assignments/[id]/schedule/route.ts` - API chỉnh lịch assignment.
- `app/src/app/api/assignments/[id]/force-submit/route.ts` - API cưỡng ép nộp bài.
- `app/src/app/api/billing/plans/route.ts` - API danh sách gói billing.
- `app/src/app/api/billing/subscription/route.ts` - API subscription hiện tại.
- `app/src/app/api/billing/usage/route.ts` - API usage/quota hiện tại.
- `app/src/app/api/billing/create-order/route.ts` - API tạo đơn hàng nâng cấp Prime.
- `app/src/app/api/billing/recheck-order/route.ts` - API kiểm tra lại trạng thái đơn hàng.
- `app/src/app/api/billing/webhook/route.ts` - API nhận webhook thanh toán.
- `app/src/app/api/ocr/route.ts` - API OCR đề/bài từ ảnh.
- `app/src/app/api/scan-grade/route.ts` - API scan grading.
- `app/src/app/api/grade-scan/route.ts` - API chấm bài từ scan/grade-scan flow.
- `app/src/app/api/community-solutions/route.ts` - API gửi/lấy lời giải cộng đồng.
- `app/src/app/api/community-solutions/[id]/vote/route.ts` - API vote một lời giải cộng đồng.
- `app/src/app/api/solution-issue-reports/route.ts` - API báo lỗi lời giải hoặc đáp án.
- `app/src/app/api/admin/community-solutions/route.ts` - API admin duyệt danh sách lời giải cộng đồng.
- `app/src/app/api/admin/community-solutions/[id]/route.ts` - API admin xử lý một lời giải cộng đồng cụ thể.
- `app/src/app/api/admin/solution-issue-reports/route.ts` - API admin xem danh sách báo lỗi lời giải.
- `app/src/app/api/admin/solution-issue-reports/[id]/route.ts` - API admin xử lý một báo lỗi cụ thể.
- `app/src/app/api/admin/subscriptions/override/route.ts` - API admin override subscription.
- `app/src/app/api/admin/usage/adjust/route.ts` - API admin điều chỉnh quota/usage.
- `app/src/app/api/admin/payments/mark-paid/route.ts` - API admin đánh dấu đơn hàng đã thanh toán.

### 9.5. Components và mục đích

- `app/src/components/ConfigForm.tsx` - form cấu hình đề cơ bản.
- `app/src/components/ConfigFormV2.tsx` - form cấu hình đề nâng cao/multi-chapter.
- `app/src/components/QuizEngine.tsx` - engine làm bài, timer, submit, anti-cheat, recovery.
- `app/src/components/ResultView.tsx` - màn hình kết quả sau khi nộp bài.
- `app/src/components/WeaknessView.tsx` - giao diện hiển thị phân tích điểm yếu.
- `app/src/components/OCRUpload.tsx` - giao diện upload ảnh OCR / scan.
- `app/src/components/MathRenderer.tsx` - render công thức toán.
- `app/src/components/LoadingScreen.tsx` - màn hình chờ khi hệ thống xử lý.
- `app/src/components/Dashboard.tsx` - dashboard/tổng quan chung của ứng dụng.
- `app/src/components/QuestionBankBrowser.tsx` - giao diện duyệt và xem kho câu hỏi.
- `app/src/components/AdaptiveToggle.tsx` - nút bật/tắt adaptive difficulty.
- `app/src/components/PrintExam.tsx` - giao diện preview/in đề thi.
- `app/src/components/AnswerSheet.tsx` - giao diện phiếu trả lời trắc nghiệm.
- `app/src/components/AnswerKeySheet.tsx` - giao diện bảng đáp án cho giáo viên.
- `app/src/components/AuthScreen.tsx` - màn hình đăng nhập/đăng ký.
- `app/src/components/TeacherDashboard.tsx` - dashboard cho giáo viên.
- `app/src/components/StudentDashboard.tsx` - dashboard cho học sinh.
- `app/src/components/ClassroomList.tsx` - danh sách lớp học.
- `app/src/components/ClassroomDetail.tsx` - màn chi tiết lớp học.
- `app/src/components/CreateClassroomForm.tsx` - form tạo lớp học.
- `app/src/components/JoinClassroomForm.tsx` - form tham gia lớp học.
- `app/src/components/AssignmentBuilder.tsx` - UI tạo/giao assignment.
- `app/src/components/AssignmentList.tsx` - danh sách assignment.
- `app/src/components/AssignmentAnalytics.tsx` - giao diện analytics cho assignment/lớp.
- `app/src/components/AssignmentScheduleEditor.tsx` - UI sửa lịch assignment.
- `app/src/components/AssignmentForceSubmitDialog.tsx` - hộp thoại force submit.
- `app/src/components/BillingDashboard.tsx` - dashboard billing/quota/subscription.
- `app/src/components/PricingCard.tsx` - card hiển thị gói Prime/giá.
- `app/src/components/UpgradePrimeModal.tsx` - modal nâng cấp Prime.
- `app/src/components/UsageQuotaCard.tsx` - card hiển thị quota/usage.
- `app/src/components/CommunitySolutionsList.tsx` - danh sách lời giải cộng đồng.
- `app/src/components/CommunitySolutionComposer.tsx` - form gửi lời giải cộng đồng.
- `app/src/components/SolutionModerationPanel.tsx` - panel moderation cho admin/giáo viên phụ trách.

### 9.6. Lib modules và mục đích

- `app/src/lib/ai-generator.ts` - logic gọi AI để sinh đề.
- `app/src/lib/question-bank.ts` - CRUD và xử lý kho câu hỏi.
- `app/src/lib/template-engine.ts` - sinh câu mới từ template/thay số.
- `app/src/lib/exam-builder.ts` - ghép câu hỏi thành đề thi hoàn chỉnh.
- `app/src/lib/exam-history.ts` - lưu/lấy lịch sử đề đã tạo.
- `app/src/lib/verification.ts` - kiểm tra chất lượng/tính đúng đắn câu hỏi.
- `app/src/lib/weakness-ai.ts` - logic phân tích điểm yếu bằng AI/heuristic.
- `app/src/lib/adaptive.ts` - quản lý adaptive difficulty.
- `app/src/lib/smart-solve.ts` - logic trust/smart solve cho câu hỏi.
- `app/src/lib/utils.ts` - helper dùng chung.
- `app/src/lib/schemas.ts` - schema dữ liệu và validation.
- `app/src/lib/auth.ts` - logic auth/session local hoặc Supabase-related.
- `app/src/lib/roles.ts` - helper phân quyền teacher/student/admin.
- `app/src/lib/classrooms.ts` - nghiệp vụ lớp học.
- `app/src/lib/assignments.ts` - nghiệp vụ assignment.
- `app/src/lib/analytics.ts` - tính toán analytics lớp và bài tập.
- `app/src/lib/billing.ts` - nghiệp vụ billing/thanh toán/quota.
- `app/src/lib/subscription.ts` - xử lý gói thuê bao Prime/free.
- `app/src/lib/usage-tracker.ts` - theo dõi số lượt dùng/quota window.
- `app/src/lib/community-solutions.ts` - nghiệp vụ lời giải cộng đồng và moderation.
- `app/src/lib/admin-audit.ts` - ghi audit log cho thao tác admin.
- `app/src/lib/ocr-engine.ts` - xử lý OCR engine.
- `app/src/lib/scan-grading.ts` - xử lý scan grading.
- `app/src/lib/question-identity.ts` - helper nhận diện/deduplicate câu hỏi.
- `app/src/lib/print-utils.ts` - helper cho in đề và mã đề.
- `app/src/lib/platform-model.ts` - mô hình dữ liệu platform tổng.
- `app/src/lib/platform-repository.ts` - abstraction repository cho platform data.
- `app/src/lib/platform-store.ts` - truy cập local platform store JSON.
- `app/src/lib/supabase/client.ts` - Supabase client phía client.
- `app/src/lib/supabase/server.ts` - Supabase helper phía server.
- `app/src/lib/supabase/middleware.ts` - middleware/helper Supabase cho request flow.
- `app/src/lib/supabase/bank.ts` - adapter bank qua Supabase.

### 9.7. Test files và mục đích

- `app/src/__tests__/adaptive.test.ts` - test logic adaptive difficulty.
- `app/src/__tests__/exam-builder.test.ts` - test exam builder.
- `app/src/__tests__/session.test.ts` - test session/recovery-related logic.
- `app/src/__tests__/template-engine.test.ts` - test template engine.
- `app/src/__tests__/update2-platform.test.ts` - test platform store/schema cho scope update2.
- `app/src/__tests__/utils.test.ts` - test các helper chung.
- `app/src/__tests__/verification.test.ts` - test verification logic.

### 9.8. Docs và mục đích

- `app/docs/manual-test-checklist.md` - checklist test tay các luồng chính.
- `app/docs/release-gate.md` - tiêu chí trước khi coi là đủ để release.
- `app/docs/data-cleanup-rules.md` - quy tắc lọc và làm sạch dữ liệu sai.
- `app/docs/product-scope.md` - tài liệu chốt ranh giới giữa `update2` và `update3`.
- `app/docs/migration-plan.md` - kế hoạch chuyển từ local store sang Supabase.
- `app/docs/supabase-setup.md` - hướng dẫn setup Supabase và migration hiện tại.

### 9.9. Data và migration files

- `app/src/data/question-bank.json` - dữ liệu question bank cục bộ.
- `app/src/data/templates.json` - dữ liệu template cục bộ.
- `app/src/data/platform-store.json` - local store tổng cho platform state.
- `app/supabase/migration.sql` - migration SQL nền hoặc bản đầu.
- `app/supabase/migrations/20260402_update2_update3.sql` - migration schema lớn cho scope update2/update3.

### 9.10. Log, report và file vận hành cục bộ

- `app/dev-server.log` - log chạy dev server.
- `app/dev-server-3001.log` - log dev server ở lần/cổng khác.
- `app/eslint-output.txt` - output ESLint.
- `app/eslint_errors.txt` - bản lỗi ESLint.
- `app/eslint_errors2.txt` - bản lỗi ESLint khác/lần chạy khác.
- `app/eslint_out.txt` - output ESLint dạng khác.
- `app/eslint_quiet.txt` - output ESLint ở mode quiet.
- `app/eslint_readable.txt` - output ESLint dễ đọc.
- `app/eslint_short.txt` - output ESLint dạng rút gọn.
- `app/lint_output.txt` - output lint tổng quát.
- `app/lint_filtered.txt` - output lint đã lọc.
- `app/lint_report.json` - báo cáo lint JSON.
- `app/lint_results.json` - kết quả lint JSON.
- `app/lint_summary.json` - tóm tắt lint JSON.
- `app/eslint.json` - dữ liệu kết quả eslint ở dạng JSON.
- `app/cookies.txt` - file cookie dùng cho kiểm thử cục bộ.
- `app/student-cookies.txt` - file cookie học sinh dùng cho kiểm thử cục bộ.

## 10. Kết luận cuối cùng

- Nếu chỉ xét trong **repo hiện tại**, thì từ lúc khởi tạo đến nay dự án đã đi từ một khung Next.js cơ bản thành một nền tảng MathQuiz AI nhiều module và nhiều lớp nghiệp vụ.
- Phần “đã làm” không chỉ nằm ở code, mà còn nằm ở quy hoạch phase, tiêu chí fix, release gate, migration plan, test checklist và dữ liệu/migration đi kèm.
- Điểm đặc biệt của trạng thái hiện tại là: **khối lượng công việc rất lớn đã tồn tại trong workspace, nhưng chưa được phản ánh đầy đủ bằng commit history của repo `app/`**.
