# Phase 3: Kế Hoạch Auth + Lớp học + Bài tập + Prime Billing

> **Phiên bản**: update2 — 2026-04-01
> **Trạng thái**: Draft for implementation

## 1. Mục tiêu

Mở rộng MathQuiz AI từ ứng dụng thi/luyện tập cá nhân thành hệ thống học tập nhiều người dùng với tài khoản thật, phân vai rõ ràng giữa học sinh và giáo viên, hỗ trợ tạo lớp, giao bài, lưu điểm, thống kê tiến độ học sinh, đồng thời chuyển cơ chế anti-cheat từ mặc định toàn hệ thống sang cấu hình do giáo viên quyết định trên từng bài tập.

Song song, hệ thống cần bổ sung phân tầng tài khoản để kiểm soát quota tạo đề: tài khoản thường chỉ được tạo tối đa 3 đề mỗi ngày, còn tài khoản Prime được tạo không giới hạn sau khi thanh toán online thành công và được hệ thống kích hoạt đúng theo gói tuần, tháng, hoặc năm.

Với giáo viên, hệ thống cũng cần giới hạn tính năng chấm bằng scan/OCR phiếu trả lời: tài khoản thường chỉ được chấm tối đa 40 bài thành công mỗi tháng, còn Prime được dùng không giới hạn.

## 2. Vấn đề hiện tại cần giải quyết

### 2.1 Auth mới ở mức nền tảng
- Đã có API `signup/login/logout` qua Supabase.
- Đã có session middleware.
- Chưa có màn hình đăng nhập/đăng ký thực tế cho người dùng.
- Chưa có role-based UI cho giáo viên và học sinh.

### 2.2 Chưa có mô hình lớp học
- Chưa có bảng lớp học, thành viên lớp, bài tập, lượt nộp.
- Kết quả hiện chủ yếu gắn với flow thi cá nhân, chưa phục vụ quản lý lớp.

### 2.3 Anti-cheat đang áp mặc định
- Logic `applyAntiCheat(...)` hiện được gọi trực tiếp khi tạo đề.
- Người dùng không có quyền chọn rõ bài nào bật anti-cheat, bài nào không.
- Yêu cầu mới: anti-cheat phải do giáo viên quyết định trên từng bài giao, không phải mặc định hệ thống.

### 2.4 Chưa có quota tạo đề và billing
- Hệ thống hiện chưa giới hạn số đề tạo mỗi ngày theo tài khoản.
- Chưa có khái niệm gói `free` / `prime`.
- Chưa có luồng thanh toán online, nhận diện giao dịch, hay tự động mở Prime.

### 2.5 Chưa có quota riêng cho scan grading của giáo viên
- Chưa có giới hạn số lượt chấm bằng scan/OCR cho giáo viên free.
- Chưa có định nghĩa rõ 1 lượt scan được tính như thế nào.
- Chưa có paywall hoặc cảnh báo khi giáo viên gần hết quota scan.

## 3. Nguyên tắc thiết kế mới

1. **Auth trước, lớp học sau**
   - Chỉ mở tính năng lớp/bài tập sau khi auth + role ổn định.

2. **Teacher-controlled anti-cheat**
   - Anti-cheat là metadata của assignment/exam instance.
   - Chỉ giáo viên được bật/tắt khi giao bài.

3. **Tách “đề gốc” và “bài được giao”**
   - Đề gốc có nội dung câu hỏi.
   - Assignment là lần giáo viên giao đề cho một lớp, kèm thời gian mở/đóng, anti-cheat, số lần làm, chế độ xem đáp án.

4. **Role-based product**
   - Giáo viên: tạo lớp, thêm học sinh, giao bài, xem thống kê.
   - Học sinh: tham gia lớp, làm bài, xem kết quả được phép xem.

5. **Không phá flow cá nhân hiện có**
   - Flow luyện tập cá nhân vẫn giữ được.
   - Flow assignment là lớp nghiệp vụ mới nằm trên nền quiz engine cũ.

6. **Quota và Prime phải enforce ở backend**
   - Client chỉ hiển thị trạng thái quota.
   - Backend là nơi quyết định user có được tạo đề hay không.
   - Các quota phải tách riêng theo loại tác vụ: tạo đề và scan grading.

7. **Thanh toán là nguồn chân lý cho Prime**
   - UI không được tự mở gói Prime.
   - Chỉ backend/service/webhook hợp lệ mới được cập nhật trạng thái thanh toán và subscription.

## 4. Mô hình dữ liệu đề xuất

### 4.1 Hồ sơ người dùng
```ts
interface UserProfile {
  id: string;                // auth.users.id
  email: string;
  displayName: string;
  role: "teacher" | "student";
  accountTier: "free" | "prime";
  grade?: number | null;
  schoolName?: string | null;
  createdAt: string;
  updatedAt: string;
}
```

### 4.2 Lớp học
```ts
interface Classroom {
  id: string;
  teacherId: string;
  name: string;
  description?: string;
  grade?: number | null;
  joinCode: string;
  status: "active" | "archived";
  createdAt: string;
}
```

### 4.3 Thành viên lớp
```ts
interface ClassroomMember {
  id: string;
  classroomId: string;
  studentId: string;
  joinedAt: string;
  status: "active" | "removed";
}
```

### 4.4 Đề thi gốc
```ts
interface ExamTemplate {
  id: string;
  ownerId: string;
  title: string;
  config: ExamConfigV2 | ExamConfig;
  questions: Question[];
  source: "ai" | "bank" | "mixed" | "manual";
  createdAt: string;
}
```

### 4.5 Cấu hình anti-cheat cho bài giao
```ts
interface AssignmentAntiCheatConfig {
  enabled: boolean;
  shuffleQuestions: boolean;
  shuffleOptions: boolean;
  detectTabSwitch: boolean;
  maxTabSwitches: number;
  autoSubmitOnExceed: boolean;
  disableCopy: boolean;
  showTabSwitchCount: boolean;
}
```

### 4.6 Bài tập được giao
```ts
interface Assignment {
  id: string;
  classroomId: string;
  teacherId: string;
  examTemplateId: string;
  title: string;
  instructions?: string;
  openAt: string;
  dueAt: string;
  durationMinutes: number;
  maxAttempts: number;
  antiCheatConfig: AssignmentAntiCheatConfig;
  showResultMode: "immediate" | "after_due" | "manual";
  status: "draft" | "published" | "closed";
  createdAt: string;
}
```

### 4.7 Lượt làm bài / nộp bài
```ts
interface AssignmentAttempt {
  id: string;
  assignmentId: string;
  studentId: string;
  attemptNo: number;
  startedAt: string;
  submittedAt?: string | null;
  status: "in_progress" | "submitted" | "auto_submitted" | "expired";
  score: number;
  correctCount: number;
  wrongCount: number;
  skippedCount: number;
  timeTaken: number;
  tabSwitches: number;
  antiCheatTriggered: boolean;
  answers: unknown;
  grading: GradingResult;
}
```

### 4.8 Tổng hợp phân tích học sinh
```ts
interface StudentAnalyticsSnapshot {
  studentId: string;
  classroomId: string;
  assignmentCount: number;
  submittedCount: number;
  avgScore: number;
  latestScore: number;
  weakTopics: string[];
  riskLevel: "low" | "medium" | "high";
  updatedAt: string;
}
```

### 4.9 Mô hình gói tài khoản
```ts
interface SubscriptionPlan {
  id: string;
  code: "free" | "prime_weekly" | "prime_monthly" | "prime_yearly";
  name: string;
  durationDays: number | null;
  examGenerateLimitPerDay: number | null;
  price: number;
  currency: "VND";
  active: boolean;
}

interface UserSubscription {
  id: string;
  userId: string;
  planCode: "free" | "prime_weekly" | "prime_monthly" | "prime_yearly";
  status: "active" | "expired" | "pending_payment" | "cancelled";
  startsAt: string | null;
  expiresAt: string | null;
  createdAt: string;
  updatedAt: string;
}

interface DailyUsage {
  id: string;
  userId: string;
  usageDate: string;             // YYYY-MM-DD
  generatedExamCount: number;
  scanGradingSuccessCount: number;
  updatedAt: string;
}

interface MonthlyUsage {
  id: string;
  userId: string;
  usageMonth: string;            // YYYY-MM
  successfulScanGradings: number;
  updatedAt: string;
}
```

### 4.10 Mô hình thanh toán
```ts
interface PaymentOrder {
  id: string;
  userId: string;
  planCode: "prime_weekly" | "prime_monthly" | "prime_yearly";
  amount: number;
  currency: "VND";
  paymentMethod: "bank_transfer" | "gateway";
  status: "pending" | "paid" | "failed" | "expired" | "cancelled";
  transferCode: string;
  bankAccountName?: string | null;
  bankAccountNumber?: string | null;
  bankName?: string | null;
  qrPayload?: string | null;
  paidAt?: string | null;
  expiresAt: string;
  rawPaymentData?: unknown;
  createdAt: string;
}

interface PaymentTransaction {
  id: string;
  paymentOrderId: string;
  provider: "manual_bank" | "gateway_webhook";
  providerTransactionId?: string | null;
  amountReceived: number;
  transferredAt?: string | null;
  payerInfo?: string | null;
  transferContent?: string | null;
  rawData?: unknown;
  matched: boolean;
  createdAt: string;
}
```

### 4.11 Mô hình scan grading
```ts
interface ScanGradingJob {
  id: string;
  teacherId: string;
  assignmentId?: string | null;
  classroomId?: string | null;
  studentId?: string | null;
  sourceType: "answer_sheet" | "uploaded_sheet";
  status: "pending_scan" | "scan_success" | "scan_failed" | "needs_manual_review";
  pagesCount: number;
  detectedAnswers?: unknown;
  score?: number | null;
  errorMessage?: string | null;
  createdAt: string;
  completedAt?: string | null;
}
```

### 4.12 Quy tắc quota sản phẩm
- Tất cả tài khoản `free`:
  - tối đa `3 đề/ngày`
- Tài khoản `teacher free`:
  - tối đa `40 lượt scan grading thành công/tháng`
- Tài khoản `prime`:
  - tạo đề không giới hạn
  - scan grading không giới hạn
- Định nghĩa 1 lượt scan:
  - `1 phiếu trả lời hoặc 1 bài làm của học sinh được chấm scan thành công = 1 lượt`
- Nếu OCR/scan thất bại hoặc job bị hủy trước khi chấm thành công:
  - không trừ quota scan
- Múi giờ quota khuyến nghị:
  - `Asia/Ho_Chi_Minh`

### 4.13 Quyết định sản phẩm đã chốt
- `3 đề/ngày` áp dụng cho mọi tài khoản `free`, không phân biệt học sinh hay giáo viên.
- `40 lượt scan grading thành công/tháng` chỉ áp dụng cho tài khoản `teacher free`.
- `1 lượt scan = 1 bài được chấm scan thành công`.
- OCR fail, scan fail, hoặc job cần review tay thì không trừ quota scan.
- Hệ thống có 3 mode tách biệt:
  - `practice mode`
  - `assignment mode`
  - `scan grading mode`
- Giáo viên chỉ xem dữ liệu lớp và assignment, không mặc định thấy dữ liệu luyện tập cá nhân của học sinh.
- Phải có `manual grading fallback` khi scan thất bại.
- Phải có `admin/manual override` cho Prime, payment, và quota nếu cần xử lý sự cố.
- Payment webhook và quota update phải xử lý idempotent để tránh trừ/ghi nhận trùng.
- Prime hết hạn thì user quay về `free`, nhưng không mất dữ liệu cũ.

## 5. Database schema Supabase cần bổ sung

### 5.1 Bảng nên có
- `user_profiles`
  - thêm cột `role`, `school_name`, `account_tier`
- `classrooms`
- `classroom_members`
- `exam_templates`
- `assignments`
- `assignment_attempts`
- `subscription_plans`
- `user_subscriptions`
- `daily_usage`
- `payment_orders`
- `payment_transactions`
- `monthly_usage`
- `scan_grading_jobs`
- có thể thêm `student_analytics_snapshots` nếu cần cache thống kê

### 5.2 Quan hệ chính
- `classrooms.teacher_id -> auth.users.id`
- `classroom_members.student_id -> auth.users.id`
- `exam_templates.owner_id -> auth.users.id`
- `assignments.classroom_id -> classrooms.id`
- `assignments.exam_template_id -> exam_templates.id`
- `assignment_attempts.assignment_id -> assignments.id`
- `assignment_attempts.student_id -> auth.users.id`
- `user_subscriptions.user_id -> auth.users.id`
- `daily_usage.user_id -> auth.users.id`
- `payment_orders.user_id -> auth.users.id`
- `payment_transactions.payment_order_id -> payment_orders.id`
- `monthly_usage.user_id -> auth.users.id`
- `scan_grading_jobs.teacher_id -> auth.users.id`

### 5.3 RLS đề xuất
- Giáo viên chỉ xem/sửa lớp của mình.
- Học sinh chỉ xem lớp mà mình là thành viên.
- Giáo viên chỉ tạo assignment cho lớp của mình.
- Học sinh chỉ tạo `assignment_attempts` cho assignment thuộc lớp mình tham gia.
- Học sinh chỉ xem bài nộp của chính mình.
- Giáo viên xem được toàn bộ bài nộp trong lớp mình phụ trách.
- Người dùng chỉ xem được subscription và payment của chính mình.
- Giáo viên chỉ xem được `scan_grading_jobs` của mình.
- Chỉ service role hoặc webhook backend mới được đánh dấu đơn hàng `paid` và kích hoạt Prime.
- Chỉ backend mới được tăng usage counters sau khi hành động thành công.

## 6. Luồng nghiệp vụ chính

### 6.1 Đăng ký
1. Người dùng chọn:
   - Học sinh
   - Giáo viên
2. Nhập email + mật khẩu + tên hiển thị.
3. `signup` qua Supabase Auth.
4. Tạo hoặc cập nhật `user_profiles`.
5. Tạo `user_subscriptions` mặc định với gói `free`.
6. Điều hướng theo role:
   - Giáo viên -> dashboard giáo viên
   - Học sinh -> dashboard học sinh

### 6.2 Đăng nhập
1. Email + mật khẩu.
2. Tải session hiện tại.
3. Lấy `user_profiles.role` và subscription hiện tại.
4. Render UI theo role.

### 6.3 Giáo viên tạo lớp
1. Nhập tên lớp, mô tả, khối lớp.
2. Hệ thống sinh `joinCode`.
3. Giáo viên thấy danh sách học sinh trong lớp và link/mã mời.

### 6.4 Học sinh tham gia lớp
1. Nhập `joinCode`.
2. Hệ thống kiểm tra lớp hợp lệ.
3. Tạo bản ghi `classroom_members`.
4. Lớp xuất hiện trong dashboard học sinh.

### 6.5 Giáo viên tạo đề và giao bài
1. Tạo đề từ AI/bank như hiện tại hoặc lưu đề sẵn thành `exam_templates`.
2. Chọn lớp cần giao.
3. Cấu hình:
   - thời gian mở/đóng
   - thời lượng làm bài
   - số lần làm
   - cách hiển thị kết quả
   - anti-cheat bật/tắt
   - nếu bật thì cấu hình chi tiết anti-cheat
4. Publish assignment.

### 6.6 Học sinh làm bài
1. Chỉ thấy assignment thuộc lớp của mình.
2. Vào bài làm -> hệ thống đọc `assignment.antiCheatConfig`.
3. Chỉ áp dụng `applyAntiCheat` nếu assignment bật anti-cheat.
4. Khi nộp bài:
   - chấm điểm
   - lưu `assignment_attempts`
   - cập nhật thống kê học sinh/lớp

### 6.7 Giáo viên xem thống kê
- Danh sách học sinh theo lớp
- Điểm trung bình, số bài đã nộp/chưa nộp
- Điểm theo từng assignment
- Chủ đề yếu của từng học sinh
- Tần suất vi phạm anti-cheat
- Top học sinh cần hỗ trợ

### 6.7.1 Quy tắc truy cập khi học sinh chưa có lớp
- Học sinh chưa tham gia lớp nào vẫn được quyền dùng hệ thống ở `practice mode`.
- Trong `practice mode`, học sinh có thể:
  - tự tạo đề
  - làm bài
  - xem kết quả cá nhân
  - xem phân tích điểm yếu cá nhân
- Trong `practice mode`, học sinh vẫn chịu giới hạn theo gói tài khoản:
  - `free` -> tối đa 3 đề/ngày
  - `prime` -> tạo đề không giới hạn
- Học sinh chưa có lớp sẽ không có quyền truy cập các tính năng phụ thuộc lớp học:
  - không thấy assignment do giáo viên giao
  - không có thống kê lớp
  - không có giáo viên theo dõi kết quả
  - không áp dụng anti-cheat theo assignment của giáo viên
- `Assignment mode` chỉ khả dụng khi học sinh là thành viên của ít nhất một lớp.

### 6.8 Luồng tạo đề có kiểm tra quota
1. User bấm tạo đề.
2. Backend kiểm tra `user_subscriptions`.
3. Nếu user đang có Prime active:
   - bỏ qua giới hạn số đề/ngày.
4. Nếu user là free:
    - kiểm tra `daily_usage.generated_exam_count`
    - nếu `< 3` thì cho tạo đề và tăng bộ đếm
    - nếu `>= 3` thì chặn, trả lỗi business rõ ràng
5. Quy tắc này áp dụng cho mọi tài khoản free, bao gồm cả học sinh free và giáo viên free.

### 6.8.1 Luồng scan grading có kiểm tra quota cho giáo viên
1. Giáo viên upload phiếu trả lời hoặc dùng màn hình scan chấm bài.
2. Backend kiểm tra role hiện tại phải là `teacher`.
3. Backend kiểm tra subscription hiện tại.
4. Nếu giáo viên đang có Prime active:
   - cho phép scan grading không giới hạn.
5. Nếu giáo viên là `free`:
   - kiểm tra `monthly_usage.successfulScanGradings`
   - nếu `< 40` thì cho thực hiện scan
   - nếu `>= 40` thì chặn và hiện paywall nâng cấp Prime
6. Chỉ khi job có trạng thái `scan_success` và chấm được bài hợp lệ thì mới tăng quota scan.
7. Nếu job rơi vào `scan_failed` hoặc `needs_manual_review`:
   - không trừ quota
   - cho phép giáo viên sửa tay hoặc scan lại

### 6.8.2 Luồng manual grading fallback
1. Nếu scan không đọc được phiếu trả lời chính xác, hệ thống tạo job với trạng thái `needs_manual_review`.
2. Giáo viên có thể nhập hoặc sửa đáp án bằng tay để hoàn tất chấm điểm.
3. Nếu giáo viên hoàn tất chấm tay từ job scan lỗi:
   - không tính là lượt scan grading thành công mới
   - vẫn lưu lịch sử job để đối soát và hỗ trợ người dùng.

### 6.9 Luồng thanh toán nâng cấp Prime

#### Cách 1 - Chuyển khoản ngân hàng có nhận diện nội dung
1. User chọn gói Prime tuần/tháng/năm.
2. Hệ thống tạo `payment_order`.
3. Sinh mã chuyển khoản duy nhất, ví dụ `MQA-USER123-MONTH-AB12`.
4. Hiển thị số tài khoản, ngân hàng, số tiền, nội dung chuyển khoản, QR thanh toán.
5. User chuyển khoản online.
6. Hệ thống nhận dữ liệu giao dịch bằng một trong hai hướng:
   - webhook từ cổng thanh toán / ngân hàng trung gian
   - hoặc luồng đối soát bán tự động/manual admin trong giai đoạn MVP
7. Nếu transaction khớp:
   - đúng số tiền
   - đúng mã chuyển khoản
   - đơn còn hiệu lực
   - chưa thanh toán trước đó
8. Hệ thống cập nhật:
   - `payment_orders.status = paid`
   - tạo `payment_transactions`
   - kích hoạt `user_subscriptions`
   - mở Prime theo đúng chu kỳ tuần/tháng/năm

#### Cách 2 - Payment gateway
- Tích hợp cổng thanh toán online có webhook xác nhận.
- Khi webhook báo thành công:
  - verify chữ ký
  - verify số tiền, mã đơn, trạng thái
  - cập nhật Prime tương tự như trên

### 6.10 Quy tắc kích hoạt Prime
- `prime_weekly` -> cộng 7 ngày
- `prime_monthly` -> cộng 30 ngày hoặc theo tháng lịch tùy quyết định sản phẩm
- `prime_yearly` -> cộng 365 ngày hoặc theo năm lịch tùy quyết định sản phẩm
- Nếu user đang có Prime active và mua tiếp:
  - khuyến nghị cộng dồn vào `expiresAt`
- Nếu user đã hết hạn:
  - bắt đầu từ thời điểm thanh toán thành công
- Khi Prime hết hạn:
  - user quay về `free`
  - giới hạn tạo đề quay lại 3 đề/ngày
  - giáo viên free quay lại giới hạn 40 lượt scan/tháng
  - không xóa lịch sử, lớp học, bài tập, kết quả cũ

### 6.11 Quy tắc cảnh báo quota và paywall
- Khi user free còn 1 lượt tạo đề trong ngày:
  - hiển thị cảnh báo nhẹ ở UI
- Khi giáo viên free còn 5 lượt scan trong tháng:
  - hiển thị cảnh báo quota sắp hết
- Khi vượt quota:
  - trả lỗi business từ backend
  - UI hiện paywall và gợi ý nâng cấp Prime
- Dashboard phải luôn hiển thị:
  - số đề đã tạo hôm nay
  - số lượt scan đã dùng trong tháng
  - ngày hết hạn Prime nếu có

### 6.13 Quy tắc admin/manual override
- Admin hoặc backend support nội bộ có thể:
  - mở Prime thủ công khi giao dịch đã nhận nhưng webhook lỗi
  - cộng lại quota nếu hệ thống trừ nhầm
  - đánh dấu payment order là `paid` sau khi đối soát thủ công hợp lệ
- Mọi thao tác override phải có audit log để truy vết.

### 6.12 Tách dữ liệu luyện tập cá nhân và dữ liệu lớp
- Điểm và kết quả trong `practice mode` là dữ liệu cá nhân của học sinh.
- Giáo viên không mặc định thấy dữ liệu luyện tập cá nhân này.
- Dashboard giáo viên chỉ tổng hợp dữ liệu từ assignment/classroom, trừ khi sau này có tính năng người học tự chia sẻ tiến độ.

## 7. Thay đổi kiến trúc anti-cheat

### 7.1 Trạng thái hiện tại
- `applyAntiCheat(exam, true, true)` đang được gọi cứng trong flow tạo đề.
- Điều này khiến anti-cheat trở thành mặc định hệ thống.

### 7.2 Thiết kế mới
- Anti-cheat không còn là mặc định ở `HomePage`.
- Anti-cheat phải lấy từ `AssignmentAntiCheatConfig` nếu là bài giáo viên giao.
- Với flow luyện tập cá nhân:
  - có thể giữ anti-cheat mặc định nhẹ
  - hoặc tách hẳn thành `practiceModeConfig`
- Với flow assignment:
  - chỉ anti-cheat theo cấu hình giáo viên

### 7.3 Quy tắc áp dụng
- `enabled = false` -> không shuffle, không tab-switch enforcement, không auto-submit
- `enabled = true` -> mới đọc các cờ chi tiết bên trong
- Dữ liệu anti-cheat phải được lưu cùng assignment và snapshot vào attempt khi học sinh bắt đầu làm bài

## 8. UI/UX cần bổ sung

### 8.1 Auth
- `AuthScreen`
  - tab Đăng nhập / Đăng ký
  - field họ tên, email, mật khẩu
  - chọn role khi đăng ký
- `AuthGuard`
  - route cần login
  - chặn role sai

### 8.2 Dashboard giáo viên
- Tổng quan lớp học
- Danh sách assignment
- Nút tạo lớp
- Nút giao bài
- Bảng điểm lớp
- Chi tiết từng học sinh

### 8.3 Dashboard học sinh
- Lớp đang tham gia
- Danh sách bài tập được giao
- Trạng thái: chưa mở / đang mở / đã nộp / quá hạn
- Điểm và phản hồi

### 8.4 Quản lý lớp
- Trang chi tiết lớp
- Danh sách học sinh
- Mã tham gia lớp
- Xóa/ẩn học sinh khỏi lớp

### 8.5 Tạo assignment
- Chọn đề nguồn
- Chọn lớp
- Chọn thời gian
- Chọn số lần làm
- Card anti-cheat:
  - bật/tắt anti-cheat
  - nếu bật thì mở advanced settings

### 8.6 Thống kê
- Biểu đồ điểm theo assignment
- Biểu đồ phân bố điểm lớp
- Bảng học sinh nguy cơ thấp/trung bình/cao
- Chủ đề yếu theo học sinh và toàn lớp

### 8.7 Billing / Subscription
- Màn hình hiển thị gói hiện tại: Free / Prime Weekly / Prime Monthly / Prime Yearly
- Hiển thị ngày hết hạn Prime
- Hiển thị số lượt tạo đề hôm nay và quota còn lại
- Nếu là giáo viên, hiển thị thêm số lượt scan còn lại trong tháng
- Nút nâng cấp Prime
- Danh sách gói + giá tiền
- Lịch sử thanh toán

### 8.8 Paywall khi hết quota
- Khi user free tạo quá 3 đề/ngày:
  - hiện modal hoặc page chặn
  - nêu rõ đã dùng hết lượt hôm nay
  - CTA nâng cấp Prime
  - CTA xem gói tuần/tháng/năm

### 8.9 Màn hình thanh toán
- Chọn gói
- Hiển thị QR chuyển khoản
- Hiển thị nội dung chuyển khoản bắt buộc
- Trạng thái đơn:
  - chờ thanh toán
  - đã nhận tiền
  - kích hoạt Prime thành công

### 8.10 UI quota scan cho giáo viên
- Hiển thị `Đã dùng 12/40 lượt scan tháng này` với tài khoản teacher free
- Hiển thị badge cảnh báo khi còn ít lượt scan
- Với job lỗi OCR, hiển thị rõ `Không trừ quota` nếu scan chưa thành công
- Có CTA `Nâng cấp Prime` khi chạm quota scan

## 9. API/Route cần có

### 9.1 Auth
- `POST /api/auth`
  - giữ login/signup/logout
  - mở rộng để lưu role/profile nếu signup
- `GET /api/me`
  - trả session + profile + subscription

### 9.2 Classroom
- `GET /api/classrooms`
- `POST /api/classrooms`
- `GET /api/classrooms/[id]`
- `POST /api/classrooms/join`
- `GET /api/classrooms/[id]/students`

### 9.3 Exam template
- `GET /api/exam-templates`
- `POST /api/exam-templates`
- `GET /api/exam-templates/[id]`

### 9.4 Assignment
- `GET /api/assignments`
- `POST /api/assignments`
- `GET /api/assignments/[id]`
- `POST /api/assignments/[id]/start`
- `POST /api/assignments/[id]/submit`

### 9.5 Analytics
- `GET /api/classrooms/[id]/analytics`
- `GET /api/students/[id]/analytics`
- `GET /api/assignments/[id]/results`

### 9.6 Billing
- `GET /api/billing/plans`
- `GET /api/billing/subscription`
- `GET /api/billing/usage`
- `POST /api/billing/create-order`
- `GET /api/billing/orders/[id]`
- `POST /api/billing/webhook`
- `POST /api/billing/recheck-order`

### 9.7 Scan grading usage
- `POST /api/scan-grade`
- `GET /api/scan-grade/jobs/[id]`
- `GET /api/billing/scan-usage`

### 9.8 Admin/support operations
- `POST /api/admin/subscriptions/override`
- `POST /api/admin/usage/adjust`
- `POST /api/admin/payments/mark-paid`

## 10. Danh sách file dự kiến thay đổi

### 10.1 File mới
- `app/src/components/AuthScreen.tsx`
- `app/src/components/TeacherDashboard.tsx`
- `app/src/components/StudentDashboard.tsx`
- `app/src/components/ClassroomList.tsx`
- `app/src/components/ClassroomDetail.tsx`
- `app/src/components/CreateClassroomForm.tsx`
- `app/src/components/JoinClassroomForm.tsx`
- `app/src/components/AssignmentBuilder.tsx`
- `app/src/components/AssignmentList.tsx`
- `app/src/components/AssignmentAnalytics.tsx`
- `app/src/components/PricingCard.tsx`
- `app/src/components/UpgradePrimeModal.tsx`
- `app/src/components/BillingDashboard.tsx`
- `app/src/components/PaymentQRCode.tsx`
- `app/src/components/UsageQuotaCard.tsx`
- `app/src/components/ScanUsageBanner.tsx`
- `app/src/lib/auth.ts`
- `app/src/lib/roles.ts`
- `app/src/lib/classrooms.ts`
- `app/src/lib/assignments.ts`
- `app/src/lib/analytics.ts`
- `app/src/lib/billing.ts`
- `app/src/lib/subscription.ts`
- `app/src/lib/payment-matching.ts`
- `app/src/lib/usage-tracker.ts`
- `app/src/lib/scan-grading.ts`
- `app/src/lib/admin-audit.ts`

### 10.2 File sửa
- `app/src/app/page.tsx`
  - thêm auth flow
  - tách teacher/student views
  - bỏ anti-cheat hardcoded khỏi flow assignment
  - hiển thị quota tạo đề và upgrade Prime
- `app/src/lib/schemas.ts`
  - thêm types cho profile, classroom, assignment, attempt, subscription, payment
- `app/src/lib/utils.ts`
  - sửa `applyAntiCheat` để nhận config thực tế thay vì boolean cứng
- `app/src/app/api/auth/route.ts`
  - ghi profile + role khi signup
- `app/src/app/api/generate/route.ts`
  - enforce quota backend
- `app/src/app/api/generate-v2/route.ts`
  - enforce quota backend
- `app/src/app/api/ocr/route.ts`
  - kiểm tra quota scan grading khi dùng flow chấm bài
- `app/src/lib/supabase/server.ts`
  - helper lấy current user/profile/subscription
- `app/src/lib/supabase/middleware.ts`
  - redirect theo session nếu cần
- `app/src/lib/supabase/bank.ts`
  - liên kết kết quả thi với assignment khi flow bài tập được dùng
- `app/supabase/migration.sql`
  - thêm các bảng lớp học, assignment, attempt, role, subscription, payment, usage
  - thêm audit log cho admin override nếu triển khai backend support

## 11. Phân phase triển khai

### Phase A: Auth UI + Role Foundation
**Ưu tiên: Rất cao**

- Hoàn thiện signup/login/logout UI
- Lưu role `teacher|student` vào `user_profiles`
- Tạo helper lấy current user + profile
- Tạo `user_subscriptions` mặc định là `free`
- Render dashboard khác nhau theo role

**Definition of Done**
- Đăng ký được bằng email/password
- Chọn được vai trò khi đăng ký
- Đăng nhập xong vào đúng dashboard
- Logout ổn định
- Refresh trang không mất session

### Phase B: Billing Foundation + Daily Quota
**Ưu tiên: Rất cao**

- Thêm bảng subscription, payment, daily usage
- Thêm bảng monthly usage cho scan grading
- Seed plan `free`, `prime_weekly`, `prime_monthly`, `prime_yearly`
- Thêm backend check quota
- Hiển thị quota trên UI

**Definition of Done**
- Free user chỉ tạo được tối đa 3 đề/ngày
- Teacher free chỉ scan grading được tối đa 40 bài thành công/tháng
- Prime user tạo đề không giới hạn
- Prime user scan grading không giới hạn
- Quota reset theo ngày
- Không bypass được quota từ client

### Phase C: Payment + Prime Activation
**Ưu tiên: Cao**

- Tạo order thanh toán
- Sinh mã chuyển khoản duy nhất
- Tích hợp webhook hoặc luồng đối soát giao dịch
- Tự động kích hoạt Prime khi thanh toán hợp lệ

**Definition of Done**
- User tạo được order thanh toán
- Hệ thống nhận biết giao dịch thanh toán đúng đơn
- Prime được kích hoạt đúng gói tuần/tháng/năm
- Không bị mở Prime sai user hoặc sai thời hạn

### Phase D: Classroom
**Ưu tiên: Rất cao**

- Tạo bảng `classrooms`, `classroom_members`
- Giáo viên tạo lớp
- Học sinh tham gia bằng mã lớp
- Danh sách lớp hiển thị đúng theo role

**Definition of Done**
- Giáo viên tạo được lớp
- Học sinh join được lớp
- Giáo viên thấy danh sách học sinh
- Học sinh chỉ thấy lớp của mình

### Phase E: Exam Template + Assignment
**Ưu tiên: Cao**

- Lưu đề thành `exam_templates`
- Tạo assignment từ đề
- Cấu hình thời gian, số lần làm, hiển thị kết quả
- Cấu hình anti-cheat ở mức assignment

**Definition of Done**
- Giáo viên giao được bài cho lớp
- Assignment lưu đầy đủ anti-cheat config
- Học sinh thấy bài được giao đúng thời gian mở/đóng

### Phase F: Attempt + Chấm điểm + Lưu điểm
**Ưu tiên: Cao**

- Tạo `assignment_attempts`
- Submit bài sẽ lưu điểm, đáp án, số lần chuyển tab, trạng thái auto-submit
- Đồng bộ kết quả với dashboard giáo viên/học sinh

**Definition of Done**
- Học sinh nộp bài thành công
- Điểm được lưu vào DB
- Giáo viên xem được danh sách kết quả
- Phân biệt được `submitted` và `auto_submitted`

### Phase G: Analytics + Đánh giá học sinh
**Ưu tiên: Cao**

- Thống kê theo lớp
- Thống kê theo học sinh
- Tổng hợp chủ đề yếu
- Tính risk level dựa trên số bài chưa nộp, điểm thấp, vi phạm anti-cheat

**Definition of Done**
- Giáo viên xem được bảng tổng hợp lớp
- Có drill-down theo từng học sinh
- Có thống kê theo assignment
- Có cảnh báo học sinh cần hỗ trợ

### Phase H: Anti-cheat Refactor Hoàn chỉnh
**Ưu tiên: Rất cao**

- Xóa việc gọi anti-cheat mặc định cứng trong flow assignment
- Chỉ bật anti-cheat khi assignment yêu cầu
- Snapshot anti-cheat config vào lúc bắt đầu attempt
- Log sự kiện vi phạm phục vụ thống kê

**Definition of Done**
- Bài không bật anti-cheat thì làm bài như bình thường
- Bài bật anti-cheat mới có shuffle/tab-switch enforcement
- Giáo viên quyết định anti-cheat ở màn giao bài
- Thống kê hiển thị được hành vi vi phạm

## 12. Kiểm thử bắt buộc

### 12.1 Unit
- Auth helpers
- Role guards
- Assignment anti-cheat config parser
- Analytics calculators
- Attempt status transitions
- Tính quota free vs prime
- Tính quota teacher free scan 40/tháng
- Tính `expiresAt`
- Match transaction theo transfer code
- Chặn duplicate payment processing
- Chỉ tăng scan quota khi `scan_success`
- Manual grading fallback không làm trừ thêm quota scan
- Admin override ghi đúng audit log

### 12.2 Integration
- Signup -> profile created
- Teacher creates classroom -> student joins
- Teacher publishes assignment -> student starts
- Student submits -> score stored -> teacher sees result
- Free user generate 3 lần thành công, lần 4 bị chặn
- Prime user generate không giới hạn
- Teacher free scan 40 bài thành công, lần 41 bị chặn
- Scan lỗi không làm tăng quota teacher free
- Manual grading từ scan lỗi không tăng thêm quota scan
- Tạo payment order -> nhận transaction hợp lệ -> Prime active
- Prime hết hạn -> quay lại free

### 12.3 Manual E2E
1. Đăng ký giáo viên
2. Tạo lớp
3. Đăng ký học sinh
4. Học sinh join lớp
5. Giáo viên tạo assignment có anti-cheat OFF
6. Học sinh làm bài, xác nhận không có enforcement anti-cheat
7. Giáo viên tạo assignment có anti-cheat ON
8. Học sinh làm bài, xác nhận enforcement hoạt động
9. Tạo tài khoản free và generate 3 đề thành công
10. Tạo đề lần 4 bị chặn
11. Đăng nhập tài khoản giáo viên free và scan 40 bài thành công
12. Scan bài thứ 41 bị chặn và hiện paywall
13. Thử scan lỗi OCR, xác nhận không trừ quota
14. Mua Prime tháng
15. Hệ thống xác nhận thanh toán
16. Tài khoản được mở Prime
17. Tạo đề tiếp không giới hạn và scan không giới hạn
18. Giáo viên mở dashboard xem điểm, thống kê, và usage quota

## 13. Rủi ro và cách giảm thiểu

### 13.1 RLS sai
- Có thể làm học sinh thấy dữ liệu của lớp khác.
- Giảm thiểu: test policy ngay từ đầu bằng nhiều tài khoản.

### 13.2 Flow assignment và flow luyện tập cá nhân chồng chéo
- Giảm thiểu: tách rõ `practice mode` và `assignment mode`.

### 13.3 Anti-cheat ảnh hưởng chấm điểm
- Giảm thiểu: snapshot question order/option order theo attempt.

### 13.4 Tăng độ phức tạp UI quá nhanh
- Giảm thiểu: triển khai dashboard teacher/student theo phase, không làm tất cả cùng lúc.

### 13.5 Rủi ro riêng của thanh toán
- Nhận diện sai giao dịch chuyển khoản
  - Giảm thiểu: mã chuyển khoản duy nhất, kiểm tra đúng số tiền, đúng trạng thái đơn, chống xử lý trùng
- Webhook giả mạo
  - Giảm thiểu: verify signature/IP whitelist nếu nhà cung cấp hỗ trợ
- Double activation
  - Giảm thiểu: idempotency key, transaction lock
- Lệch múi giờ khi tính hết hạn
  - Giảm thiểu: dùng UTC trong DB, format lại ở UI

### 13.6 Rủi ro riêng của quota scan grading
- OCR fail nhiều lần gây khó chịu cho giáo viên free
  - Giảm thiểu: chỉ trừ quota khi scan thành công
- Người dùng không hiểu vì sao bị trừ quota
  - Giảm thiểu: hiển thị lịch sử scan jobs và trạng thái rõ ràng
- Scan lặp cùng một bài gây tranh chấp quota
  - Giảm thiểu: lưu `scan_grading_jobs`, có đối soát lịch sử và giới hạn chống submit trùng

### 13.7 Rủi ro riêng của manual override
- Support mở Prime hoặc cộng quota sai tài khoản
  - Giảm thiểu: xác thực nội bộ, audit log, yêu cầu đối soát theo order id / user id rõ ràng
- Override bị lạm dụng làm sai lệch doanh thu hoặc usage
  - Giảm thiểu: phân quyền chặt và log toàn bộ thao tác admin

## 14. Thứ tự triển khai khuyến nghị

1. Auth UI + profile role
2. Daily quota + subscription foundation
3. Scan grading quota + usage tracking cho teacher free
4. Payment order + Prime activation
5. Classroom + join code
6. Exam template persistence
7. Assignment creation
8. Attempt submission + score storage
9. Teacher analytics
10. Anti-cheat refactor hoàn chỉnh cho assignment mode
11. Tối ưu UX và bổ sung test

## 15. Quyết định sản phẩm cần khóa trước khi code

1. Học sinh có được tự đăng ký tài khoản không, hay giáo viên tạo trước?
2. Học sinh tham gia lớp bằng mã mời là đủ hay cần giáo viên duyệt?
3. Một học sinh có được làm lại assignment nhiều lần không?
4. Học sinh có được xem đáp án/lời giải ngay sau khi nộp không?
5. Flow luyện tập cá nhân có tiếp tục giữ adaptive riêng hay chỉ assignment mới lưu thống kê lớp?
6. Giá Prime tuần / tháng / năm là bao nhiêu?
7. Reset quota theo múi giờ Việt Nam hay UTC?
8. Chỉ hỗ trợ chuyển khoản ngân hàng hay tích hợp thêm cổng thanh toán ngay từ đầu?
9. Kích hoạt Prime tự động hoàn toàn hay giai đoạn đầu cho phép admin duyệt giao dịch lỗi?
10. Prime có áp dụng cho cả giáo viên và học sinh, hay chỉ một nhóm người dùng?
11. Manual grading fallback có được mở ngay ở phase đầu của scan grading không?
12. Admin override sẽ dùng tool nội bộ đơn giản hay cần giao diện admin ngay từ đầu?

## 16. Kết luận

Phase update2 biến MathQuiz AI thành nền tảng học tập nhiều người dùng thực thụ:
- có đăng nhập/đăng ký thật,
- có vai trò giáo viên/học sinh,
- có lớp học,
- có bài tập và lưu điểm,
- có thống kê đánh giá học sinh,
- có quota tạo đề theo gói tài khoản,
- có quota scan grading riêng cho giáo viên free,
- có Prime tuần/tháng/năm với thanh toán online và nhận diện giao dịch,
- và anti-cheat được chuyển thành quyền cấu hình của giáo viên theo từng bài giao thay vì mặc định toàn hệ thống.
