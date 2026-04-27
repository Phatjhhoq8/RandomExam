# Phase 3: Kế Hoạch Auth + Lớp học + Bài tập + Prime Billing

> **Phiên bản**: update2 — 2026-04-01
> **Trạng thái**: Draft for implementation

## 1. Mục tiêu

Mở rộng MathQuiz AI từ ứng dụng thi/luyện tập cá nhân thành hệ thống học tập nhiều người dùng với tài khoản thật, phân vai rõ ràng giữa học sinh và giáo viên, hỗ trợ tạo lớp, giao bài, lưu điểm, thống kê tiến độ học sinh, đồng thời chuyển cơ chế anti-cheat từ mặc định toàn hệ thống sang cấu hình do giáo viên quyết định trên từng bài tập.

Ngoài luồng lớp học và billing, hệ thống cần mở thêm lớp cộng đồng cho phần lời giải: cho phép mọi người dùng gửi lời giải thay thế, báo lời giải sai, vote các cách giải đã được duyệt, hiển thị danh sách "các cách giải khác" theo độ hữu ích, nhưng chỉ admin mới được quyền phê duyệt và đưa đóng góp vào dữ liệu hiển thị chính thức.

Song song, hệ thống cần bổ sung phân tầng tài khoản để kiểm soát quota tạo đề: tài khoản thường chỉ được tạo tối đa 3 đề mỗi ngày, còn tài khoản Prime được tạo không giới hạn sau khi thanh toán online thành công và được hệ thống kích hoạt đúng theo gói tuần, tháng, hoặc năm.

Với giáo viên, hệ thống cũng cần giới hạn tính năng chấm bằng scan/OCR phiếu trả lời: tài khoản thường chỉ được chấm tối đa 40 bài thành công mỗi tuần, còn Prime được dùng không giới hạn.

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

### 2.6 Chưa có cơ chế cộng đồng sửa và xếp hạng lời giải
- Mỗi câu hiện chỉ có một `solution` chuẩn, chưa có danh sách lời giải thay thế.
- Người dùng chưa thể báo lời giải sai hoặc đề xuất cách giải khác ngay tại màn hình kết quả.
- Chưa có cơ chế vote để đẩy các cách giải hữu ích lên đầu.
- Chưa có quy trình moderation để admin duyệt trước khi công khai lời giải cộng đồng.

## 3. Nguyên tắc thiết kế mới

1. **Auth trước, lớp học sau**
   - Chỉ mở tính năng lớp/bài tập sau khi auth + role ổn định.

2. **Teacher-controlled anti-cheat**
   - Anti-cheat là metadata của assignment/exam instance.
   - Chỉ giáo viên được bật/tắt khi giao bài.

3. **Tách “cấu trúc đề”, “bài được giao”, và “mã đề”**
   - Cấu trúc đề chỉ mô tả cách sinh đề: lớp, chương, số câu, độ khó, thời gian, nguồn câu hỏi.
   - Assignment là lần giáo viên giao bài cho một lớp, kèm thời gian mở/đóng, anti-cheat, số lần làm, chế độ xem đáp án.
   - Một assignment có thể sinh ra nhiều variant/mã đề khác nhau từ cùng một cấu trúc để giảm trao đổi đáp án khi học sinh làm ở nhà.

4. **Role-based product**
   - Giáo viên: tạo lớp, thêm học sinh, giao bài, xem thống kê.
   - Học sinh: tham gia lớp, làm bài, xem kết quả được phép xem.

5. **Không phá flow cá nhân hiện có**
   - Flow luyện tập cá nhân vẫn giữ được, bao gồm adaptive difficulty riêng cho người học.
   - Flow assignment là lớp nghiệp vụ mới nằm trên nền quiz engine cũ.

6. **Adaptive chỉ thuộc flow tự luyện cá nhân**
   - Adaptive difficulty chỉ áp dụng cho `practice mode` / `self_practice`.
   - Kết quả từ assignment/lớp học không được cập nhật adaptive profile cá nhân.
   - Độ khó của assignment do giáo viên hoặc cấu trúc đề quyết định, không bị adaptive cá nhân tự override.
   - Nếu cần insight cho lớp học thì làm bằng classroom analytics riêng, không dùng lại adaptive profile cá nhân.

7. **Quota và Prime phải enforce ở backend**
   - Client chỉ hiển thị trạng thái quota.
   - Backend là nơi quyết định user có được tạo đề hay không.
   - Các quota phải tách riêng theo loại tác vụ: tạo đề và scan grading.

8. **Thanh toán là nguồn chân lý cho Prime**
   - UI không được tự mở gói Prime.
   - Chỉ backend/service/webhook hợp lệ mới được cập nhật trạng thái thanh toán và subscription.

9. **Lời giải cộng đồng phải qua moderation**
   - Ai cũng có thể gửi đóng góp, kể cả không cần auth chặt trong giai đoạn đầu.
   - Mọi lời giải cộng đồng đều đi vào hàng chờ `pending` trước khi hiển thị công khai.
   - Vote chỉ áp dụng cho lời giải đã `approved`.
   - Lời giải chuẩn và lời giải cộng đồng là hai lớp dữ liệu khác nhau; admin có thể nâng một lời giải cộng đồng thành lời giải chuẩn khi cần.

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

### 4.4 Cấu trúc đề
```ts
interface ExamTemplate {
  id: string;
  ownerId: string;
  title: string;
  config: ExamConfigV2 | ExamConfig;
  description?: string;
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
  variantMode: "per_student" | "limited_pool";
  variantCount?: number | null;
  structureSnapshot: ExamConfigV2 | ExamConfig;
  questionGenerationSeed?: string | null;
  visibleToFutureMembers: boolean;
  allowFutureMembersToAttempt: boolean;
  showResultMode: "immediate" | "after_due" | "manual";
  status: "draft" | "published" | "closed";
  createdAt: string;
  updatedAt: string;
  lastScheduleEditedAt?: string | null;
  lastScheduleEditedBy?: string | null;
  scheduleVersion: number;
}
```

### 4.7 Mã đề / variant của assignment
```ts
interface AssignmentVariant {
  id: string;
  assignmentId: string;
  variantCode: string;
  examSnapshot: Exam;
  generatedFromTemplateId?: string | null;
  generatedAt: string;
  createdAt: string;
}
```

### 4.8 Gán assignment cho học sinh
```ts
interface AssignmentRecipient {
  id: string;
  assignmentId: string;
  studentId: string;
  assignmentVariantId?: string | null;
  status: "assigned" | "in_progress" | "submitted" | "overdue" | "view_only";
  seatConsumed: boolean;
  seatConsumedAt?: string | null;
  assignedAt: string;
  startedAt?: string | null;
  submittedAt?: string | null;
  bestAttemptId?: string | null;
}
```

### 4.9 Lượt làm bài / nộp bài
```ts
interface AssignmentAttempt {
  id: string;
  assignmentId: string;
  studentId: string;
  assignmentVariantId: string;
  attemptNo: number;
  startedAt: string;
  submittedAt?: string | null;
  status: "in_progress" | "submitted" | "auto_submitted" | "force_submitted" | "expired";
  score: number;
  correctCount: number;
  wrongCount: number;
  skippedCount: number;
  timeTaken: number;
  tabSwitches: number;
  antiCheatTriggered: boolean;
  durationSnapshotMinutes: number;
  antiCheatConfigSnapshot: AssignmentAntiCheatConfig;
  questionOrderSnapshot?: unknown;
  optionOrderSnapshot?: unknown;
  answers: unknown;
  grading: GradingResult;
}
```

### 4.10 Lịch sử chỉnh thời gian assignment
```ts
interface AssignmentScheduleHistory {
  id: string;
  assignmentId: string;
  oldOpenAt: string;
  oldDueAt: string;
  oldDurationMinutes: number;
  newOpenAt: string;
  newDueAt: string;
  newDurationMinutes: number;
  reason?: string | null;
  changedBy: string;
  changedAt: string;
}
```

### 4.11 Lịch sử cưỡng ép nộp bài
```ts
interface AssignmentForceSubmitLog {
  id: string;
  assignmentId: string;
  studentId?: string | null;
  attemptId: string;
  forcedBy: string;
  reason?: string | null;
  createdAt: string;
}
```

### 4.12 Tổng hợp phân tích học sinh
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

### 4.13 Mô hình gói tài khoản
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
  updatedAt: string;
}

interface UsageWindow {
  id: string;
  userId: string;
  quotaType: "assignment_student_seat" | "scan_grading";
  limitCount: number;
  usedCount: number;
  windowStartedAt?: string | null;
  windowExpiresAt?: string | null;
  assignmentPublishedCount?: number;
  updatedAt: string;
}
```

### 4.14 Mô hình thanh toán
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

### 4.15 Mô hình scan grading
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

### 4.15b Mô hình đóng góp lời giải cộng đồng
```ts
interface CommunitySolution {
  id: string;
  questionRef: string;
  type: "alternative" | "correction";
  content: string;
  authorName: string;
  submittedByUserId?: string | null;
  status: "pending" | "approved" | "rejected";
  voteScore: number;
  upvoteCount: number;
  downvoteCount: number;
  promotedToCanonical: boolean;
  createdAt: string;
  reviewedAt?: string | null;
  reviewedBy?: string | null;
}

interface CommunitySolutionVote {
  id: string;
  communitySolutionId: string;
  voterKey: string;
  value: 1 | -1;
  createdAt: string;
  updatedAt: string;
}

interface SolutionIssueReport {
  id: string;
  questionRef: string;
  reportType: "wrong_solution" | "wrong_answer" | "unclear_explanation";
  note?: string | null;
  suggestedCorrectAnswer?: "A" | "B" | "C" | "D" | null;
  reporterName: string;
  reporterUserId?: string | null;
  status: "pending" | "resolved" | "rejected";
  createdAt: string;
  reviewedAt?: string | null;
}
```

### 4.15c Quy tắc hiển thị và xếp hạng lời giải cộng đồng
- `solution` hiện tại vẫn là lời giải chuẩn mặc định của hệ thống.
- Nút `Các cách giải khác` mở danh sách `CommunitySolution` đã `approved` theo từng câu.
- Danh sách lời giải cộng đồng phải sắp theo:
  - `voteScore` giảm dần
  - nếu hòa thì ưu tiên lời giải được duyệt gần hơn
  - nếu vẫn hòa thì ưu tiên lời giải mới hơn
- Mỗi lời giải cần hiển thị tối thiểu:
  - nội dung lời giải
  - tên người đóng góp
  - điểm vote
  - badge `Đã duyệt` và badge `Lời giải thay thế` hoặc `Đề xuất sửa`
- Admin có thể chọn một lời giải cộng đồng để `promotedToCanonical = true`; khi đó lời giải này trở thành lời giải chuẩn và lời giải chuẩn cũ phải được lưu vào lịch sử hoặc chuyển thành lời giải thay thế đã duyệt.
- Vote chỉ được tính trên lời giải đã `approved`; lời giải `pending` và `rejected` không được hiện công khai.
- MVP cho phép người dùng nhập `authorName` công khai; nếu bỏ trống thì backend tự gán nhãn ẩn danh.

### 4.16 Quy tắc tạo variant/mã đề cho assignment
- Mặc định assignment dùng `variantMode = "per_student"`.
- Ở mode này, hệ thống sinh một đề riêng cho từng học sinh trong lớp từ cùng `structureSnapshot`.
- Giáo viên có thể chọn `variantMode = "limited_pool"` và nhập `variantCount` nếu muốn chỉ tạo một số lượng mã đề giới hạn cho cả lớp.
- Mỗi `AssignmentVariant` phải được snapshot cố định ngay khi publish assignment hoặc theo một quy tắc generate nhất quán ở lần start đầu tiên.
- Việc gán học sinh -> variant phải đi qua `AssignmentRecipient`, không gắn cứng trực tiếp trong `AssignmentVariant`.
- Một học sinh luôn làm đúng variant đã được gán cho mình; reload hoặc vào lại không được sinh đề khác.
- Giáo viên phải xem được `variantCode` trong màn chi tiết bài làm để đối chiếu khi cần.

### 4.17 Quy tắc hiển thị assignment cũ cho học sinh mới vào lớp
- Học sinh vào lớp sau vẫn nhìn thấy các assignment đã publish trước đó.
- Việc `nhìn thấy assignment` và `được phép làm assignment` là hai quyền khác nhau.
- Mặc định:
  - `visibleToFutureMembers = true`
  - `allowFutureMembersToAttempt = false`
- Giáo viên được quyền bật/tắt `allowFutureMembersToAttempt` trên từng assignment.
- Nếu `allowFutureMembersToAttempt = true` và assignment vẫn còn hiệu lực, hệ thống có thể tạo `AssignmentRecipient` và gán `AssignmentVariant` bổ sung cho học sinh vào sau.
- Nếu assignment đã quá hạn hoặc đã đóng, học sinh vào sau chỉ xem được metadata và trạng thái, không bắt đầu làm mới.

### 4.18 Quy tắc chỉnh thời gian assignment
- Giáo viên được phép chỉnh `openAt`, `dueAt`, `durationMinutes` sau khi publish assignment.
- Việc chỉnh thời gian không được làm đổi `structureSnapshot`, `assignment_variant`, hoặc bài đã nộp.
- `durationMinutes` mới chỉ áp dụng cho các attempt bắt đầu sau khi chỉnh.
- Attempt đang `in_progress` phải giữ `durationSnapshotMinutes` đã chụp lúc start.
- Mọi thay đổi thời gian phải được lưu vào `AssignmentScheduleHistory` để audit và hiển thị cho học sinh nếu cần.
- Giáo viên không được đóng assignment nếu vẫn còn học sinh đang ở trạng thái `in_progress`.
- Nếu cần kết thúc sớm, giáo viên phải dùng thao tác `force submit` thay vì đóng assignment trực tiếp.

### 4.19 Quy tắc cưỡng ép nộp bài
- Giáo viên được phép `force submit` một học sinh hoặc toàn bộ học sinh đang làm bài.
- Khi `force submit`, hệ thống chấm điểm theo đáp án hiện tại và lưu attempt với trạng thái `force_submitted`.
- Mọi thao tác `force submit` phải có audit log để biết ai thực hiện, lúc nào, và vì sao.
- UI học sinh và giáo viên phải hiển thị rõ bài đó bị giáo viên kết thúc cưỡng ép.

### 4.20 Quy tắc quota sản phẩm
- Tất cả tài khoản `free`:
  - tối đa `3 đề/ngày`
- Tài khoản `teacher free`:
  - tối đa `40 student seats/tuần` cho assignment mode
  - tối đa `40 lượt scan grading thành công/tuần`
- Tài khoản `prime`:
  - tạo đề không giới hạn
  - assignment/student seats không giới hạn
  - scan grading không giới hạn
- Assignment mode không trừ quota của học sinh; quota assignment được tính về giáo viên nhưng chỉ bị trừ khi học sinh bắt đầu làm bài lần đầu.
- Định nghĩa `1 student seat`:
  - `1 học sinh bắt đầu làm 1 assignment lần đầu = 1 seat`
- Ví dụ:
  - giao 1 assignment cho 20 học sinh, nhưng chỉ 12 em bấm vào làm -> dùng 12 seats
  - một em trong 12 em đó làm lại lần 2, lần 3 -> không dùng thêm seat
- Định nghĩa 1 lượt scan:
  - `1 phiếu trả lời hoặc 1 bài làm của học sinh được chấm scan thành công = 1 lượt`
- Nếu OCR/scan thất bại hoặc job bị hủy trước khi chấm thành công:
  - không trừ quota scan
- Publish assignment thành công không tự trừ `student seat quota`.
- Chỉ trừ `student seat quota` khi học sinh bắt đầu attempt đầu tiên của assignment và `seatConsumed = false`.
- Một học sinh chỉ bị trừ tối đa `1 seat` cho mỗi assignment, kể cả khi được phép làm lại nhiều lần.
- Học sinh vào lớp sau nếu được phép làm assignment cũ cũng chỉ bị trừ seat ở lần start đầu tiên.
- Xóa học sinh khỏi lớp hoặc đóng assignment sau đó không hoàn lại `student seat quota` đã dùng.
- Múi giờ quota khuyến nghị:
  - `Asia/Ho_Chi_Minh`
- `student seat quota` và `scan quota` dùng rolling window `7 ngày`.
- Bộ đếm quota bắt đầu từ lần sử dụng bị trừ đầu tiên sau khi quota đã hồi hoàn toàn.
- Khi cửa sổ quota hết hạn, quota trở về trạng thái sẵn sàng và chỉ bắt đầu đếm lại khi có lần sử dụng bị trừ đầu tiên tiếp theo.

### 4.21 Quyết định sản phẩm đã chốt
- `3 đề/ngày` áp dụng cho mọi tài khoản `free`, không phân biệt học sinh hay giáo viên.
- Assignment mode không trừ quota học sinh; quota assignment được tính cho giáo viên.
- `teacher free` có tối đa `40 student seats/tuần`.
- `1 student seat = 1 học sinh bắt đầu làm 1 assignment lần đầu`.
- `40 lượt scan grading thành công/tuần` chỉ áp dụng cho tài khoản `teacher free`.
- `student seat quota` và `scan quota` chạy theo rolling window `7 ngày` tính từ lần dùng bị trừ đầu tiên sau khi quota hồi.
- `1 lượt scan = 1 bài được chấm scan thành công`.
- OCR fail, scan fail, hoặc job cần review tay thì không trừ quota scan.
- Assignment mặc định dùng chế độ `mỗi học sinh một đề riêng` để giảm trao đổi đáp án khi làm ở nhà.
- Giáo viên có thể chọn số lượng `mã đề/variant` khi giao bài nếu không muốn sinh riêng hoàn toàn cho từng học sinh.
- Dù dùng `per_student` hay `limited_pool`, mỗi bài làm phải gắn cố định với một `assignment_variant` để tránh đổi đề khi học sinh vào lại.
- Học sinh vào lớp sau vẫn nhìn thấy assignment cũ đã publish.
- `allowFutureMembersToAttempt` là cờ do giáo viên chọn để quyết định học sinh vào lớp sau có được làm assignment cũ hay không.
- Giáo viên được chỉnh lại thời gian assignment; thay đổi thời lượng chỉ áp dụng cho attempt bắt đầu sau thời điểm chỉnh.
- Giáo viên không được đóng assignment nếu còn học sinh đang làm; nếu cần kết thúc sớm thì dùng `force submit`.
- `assignmentPublishedCount` hiện dùng cho analytics/billing nội bộ, chưa phải quota chặn chính.
- Hệ thống có 3 mode tách biệt:
  - `practice mode`
  - `assignment mode`
  - `scan grading mode`
- Mọi exam/attempt cần có `source` / `origin` rõ ràng giữa `self_practice` và `classroom_assignment` để chặn logic adaptive đi nhầm luồng.
- `practice mode` mới được ghi lịch sử adaptive để tăng/giảm độ khó cho lần sinh đề sau.
- `assignment mode` chỉ phục vụ chấm điểm, tiến độ, và analytics lớp; không được cập nhật adaptive profile cá nhân.
- Giáo viên chỉ xem dữ liệu lớp và assignment, không mặc định thấy dữ liệu luyện tập cá nhân của học sinh.
- Phải có `manual grading fallback` khi scan thất bại.
- Phải có `admin/manual override` cho Prime, payment, và quota nếu cần xử lý sự cố.
- Payment webhook và quota update phải xử lý idempotent để tránh trừ/ghi nhận trùng.
- Prime hết hạn thì user quay về `free`, nhưng không mất dữ liệu cũ.
- Ai cũng có thể gửi đóng góp lời giải và báo lỗi lời giải.
- Chỉ admin mới được duyệt lời giải cộng đồng hoặc nâng thành lời giải chuẩn.
- Màn hình kết quả phải có nút `Các cách giải khác` và hiển thị danh sách lời giải đã duyệt.
- Danh sách lời giải cộng đồng phải sắp theo vote cao xuống thấp.
- Mỗi lời giải công khai phải hiện người đóng góp và điểm vote.
- MVP cho vote có thể dùng `voterKey` theo user hoặc device; nhưng backend vẫn phải enforce mỗi `voterKey` chỉ có 1 phiếu trên 1 lời giải.

## 5. Database schema Supabase cần bổ sung

### 5.1 Bảng nên có
- `user_profiles`
  - thêm cột `role`, `school_name`, `account_tier`
- `classrooms`
- `classroom_members`
- `exam_templates`
- `assignments`
- `assignment_variants`
- `assignment_recipients`
- `assignment_attempts`
- `assignment_schedule_history`
- `assignment_force_submit_logs`
- `subscription_plans`
- `user_subscriptions`
- `daily_usage`
- `payment_orders`
- `payment_transactions`
- `usage_windows`
- `scan_grading_jobs`
- `community_solutions`
- `community_solution_votes`
- `solution_issue_reports`
- có thể thêm `student_analytics_snapshots` nếu cần cache thống kê

### 5.2 Quan hệ chính
- `classrooms.teacher_id -> auth.users.id`
- `classroom_members.student_id -> auth.users.id`
- `exam_templates.owner_id -> auth.users.id`
- `assignments.classroom_id -> classrooms.id`
- `assignments.exam_template_id -> exam_templates.id`
- `assignment_variants.assignment_id -> assignments.id`
- `assignment_recipients.assignment_id -> assignments.id`
- `assignment_recipients.student_id -> auth.users.id`
- `assignment_recipients.assignment_variant_id -> assignment_variants.id`
- `assignment_attempts.assignment_id -> assignments.id`
- `assignment_attempts.assignment_variant_id -> assignment_variants.id`
- `assignment_attempts.student_id -> auth.users.id`
- `assignment_schedule_history.assignment_id -> assignments.id`
- `assignment_force_submit_logs.assignment_id -> assignments.id`
- `assignment_force_submit_logs.attempt_id -> assignment_attempts.id`
- `user_subscriptions.user_id -> auth.users.id`
- `daily_usage.user_id -> auth.users.id`
- `payment_orders.user_id -> auth.users.id`
- `payment_transactions.payment_order_id -> payment_orders.id`
- `usage_windows.user_id -> auth.users.id`
- `scan_grading_jobs.teacher_id -> auth.users.id`
- `community_solutions.submitted_by_user_id -> auth.users.id` (nullable)
- `community_solutions.reviewed_by -> auth.users.id` (nullable, admin)
- `community_solution_votes.community_solution_id -> community_solutions.id`
- `solution_issue_reports.reporter_user_id -> auth.users.id` (nullable)

### 5.3 RLS đề xuất
- Giáo viên chỉ xem/sửa lớp của mình.
- Học sinh chỉ xem lớp mà mình là thành viên.
- Giáo viên chỉ tạo assignment cho lớp của mình.
- Giáo viên chỉ xem được `assignment_variants` thuộc assignment của lớp mình.
- Giáo viên chỉ xem/sửa được `assignment_recipients` thuộc assignment của lớp mình.
- Giáo viên chỉ tạo được `force submit log` cho assignment thuộc lớp mình.
- Học sinh chỉ tạo `assignment_attempts` cho assignment thuộc lớp mình tham gia.
- Học sinh chỉ đọc được `assignment_variant` đã gán cho mình.
- Học sinh chỉ đọc được `assignment_recipient` của chính mình.
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
1. Tạo hoặc chọn `cấu trúc đề` từ AI/bank như hiện tại và lưu thành `exam_templates`.
2. Chọn lớp cần giao.
3. Cấu hình:
    - thời gian mở/đóng
    - thời lượng làm bài
    - số lần làm
    - cách hiển thị kết quả
    - anti-cheat bật/tắt
    - cách sinh đề:
      - `mỗi học sinh một đề riêng`
      - hoặc `số lượng mã đề giới hạn`
    - số lượng mã đề nếu chọn `limited_pool`
    - assignment cũ có hiển thị cho học sinh vào lớp sau hay không
    - học sinh vào lớp sau có được làm assignment cũ hay không
    - nếu bật thì cấu hình chi tiết anti-cheat
4. Khi publish assignment, hệ thống sinh các `assignment_variants` tương ứng theo cấu trúc đã chọn.
5. Hệ thống tạo `assignment_recipients` cho danh sách học sinh hiện có trong lớp.
6. Mỗi học sinh được gán cố định một variant trước khi bắt đầu làm bài.
7. Publish assignment không tự trừ `student seat quota`; quota chỉ bị trừ khi học sinh bắt đầu làm bài lần đầu.

### 6.6 Học sinh làm bài
1. Chỉ thấy assignment thuộc lớp của mình.
2. Nếu là học sinh vào lớp sau, vẫn thấy assignment cũ theo rule hiển thị của assignment.
3. Chỉ được bắt đầu làm khi có `assignment_recipient` hợp lệ hoặc assignment cho phép generate recipient cho thành viên vào sau.
4. Nếu `assignment_recipient.seatConsumed = false`, hệ thống check quota giáo viên và chỉ cho start khi còn seat; sau đó mới set `seatConsumed = true`.
5. Vào bài làm -> hệ thống lấy đúng `assignment_variant` đã gán cho học sinh.
6. Học sinh luôn quay lại đúng đề cũ nếu đang làm dở hoặc vào lại assignment.
7. Hệ thống đọc `assignment.antiCheatConfig`.
8. Chỉ áp dụng `applyAntiCheat` nếu assignment bật anti-cheat.
9. Khi nộp bài:
    - chấm điểm
    - lưu `assignment_attempts`
    - cập nhật `assignment_recipients.status`
    - cập nhật thống kê học sinh/lớp

### 6.6.1 Giáo viên chỉnh thời gian assignment
1. Giáo viên mở màn chi tiết assignment và sửa `openAt`, `dueAt`, `durationMinutes`.
2. Hệ thống validate quyền sửa và tính hợp lệ của lịch mới.
3. Hệ thống tăng `scheduleVersion`, cập nhật `lastScheduleEditedAt`, `lastScheduleEditedBy`.
4. Hệ thống ghi `AssignmentScheduleHistory` để audit.
5. `durationMinutes` mới chỉ áp dụng cho các attempt bắt đầu sau khi thay đổi.
6. Attempt đang `in_progress` giữ nguyên `durationSnapshotMinutes` cũ.

### 6.6.2 Giáo viên cưỡng ép nộp bài
1. Giáo viên mở màn hình assignment và chọn một học sinh hoặc toàn bộ học sinh đang `in_progress`.
2. Hệ thống lấy đáp án hiện tại đã autosave của từng attempt đang chạy.
3. Hệ thống chấm điểm với dữ liệu hiện có và cập nhật trạng thái `force_submitted`.
4. Hệ thống ghi `AssignmentForceSubmitLog` để audit.
5. UI học sinh và giáo viên hiển thị rõ bài bị giáo viên kết thúc cưỡng ép.

### 6.7 Giáo viên xem thống kê
- Danh sách học sinh theo lớp
- Điểm trung bình, số bài đã nộp/chưa nộp
- Phân biệt được `đã giao`, `đang làm`, `đã nộp`, `quá hạn`, `chỉ xem`, `vào lớp sau khi giao bài`
- Phân biệt được `force_submitted` với các trạng thái nộp bài khác
- Điểm theo từng assignment
- Xem được học sinh đang gắn với `variantCode` nào
- Chủ đề yếu của từng học sinh
- Tần suất vi phạm anti-cheat
- Top học sinh cần hỗ trợ

### 6.7b Luồng cộng đồng đóng góp lời giải và admin duyệt
1. Người dùng mở màn hình kết quả và bấm `Đóng góp lời giải` hoặc `Báo lời giải sai`.
2. Hệ thống tạo `CommunitySolution` hoặc `SolutionIssueReport` với trạng thái `pending`.
3. Admin mở dashboard moderation để đối chiếu đề bài, lời giải chuẩn hiện tại, và nội dung đóng góp.
4. Nếu admin duyệt lời giải loại `alternative`:
   - lời giải được đưa vào danh sách `Các cách giải khác`
   - lời giải bắt đầu cho phép vote công khai
5. Nếu admin duyệt lời giải loại `correction`:
   - admin có thể giữ nó như lời giải thay thế đã duyệt
   - hoặc nâng nó thành lời giải chuẩn mới của câu hỏi
6. Khi người dùng mở `Các cách giải khác`, API chỉ trả về các lời giải `approved` của câu đó và sắp theo `voteScore` giảm dần.

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

### 6.8.0 Luồng publish assignment với seat quota trừ khi học sinh bắt đầu làm
1. Giáo viên cấu hình assignment và bấm publish.
2. Backend kiểm tra role hiện tại phải là `teacher`.
3. Backend kiểm tra subscription hiện tại.
4. Nếu giáo viên đang có Prime active:
   - cho phép publish assignment và sinh recipients/variants không giới hạn.
5. Nếu giáo viên là `free`:
   - cho phép publish assignment nếu metadata và quyền hợp lệ, không trừ seat ở bước này.
6. Chỉ khi publish assignment thành công mới tăng `assignmentPublishedCount` để phục vụ analytics/billing nội bộ.
7. Nếu học sinh vào lớp sau và assignment cho phép các thành viên vào sau được làm bài:
   - khi tạo thêm `assignment_recipient` mới cũng chưa trừ seat ngay.

### 6.8.0.1 Luồng start attempt có kiểm tra student seat quota
1. Học sinh bấm bắt đầu làm assignment.
2. Backend lấy `assignment_recipient` tương ứng.
3. Nếu `seatConsumed = true`:
   - cho phép vào bài mà không trừ thêm seat.
4. Nếu `seatConsumed = false` và giáo viên đang có Prime active:
   - set `seatConsumed = true`, lưu `seatConsumedAt`, cho phép start.
5. Nếu `seatConsumed = false` và giáo viên là `free`:
   - kiểm tra `usage_windows` của quota `assignment_student_seat`.
   - nếu chưa có cửa sổ đang chạy hoặc cửa sổ cũ đã hết hạn thì lần trừ đầu tiên sẽ mở cửa sổ mới `7 ngày`.
   - nếu `usedCount < 40` thì tăng `usedCount`, set `seatConsumed = true`, lưu `seatConsumedAt`, và cho phép start.
   - nếu `usedCount >= 40` thì chặn start, hiển thị paywall/nội dung hết quota của giáo viên.
6. Một học sinh làm lại assignment nhiều lần không trừ thêm seat nếu `seatConsumed` đã là `true`.

### 6.8.1 Luồng scan grading có kiểm tra quota cho giáo viên
1. Giáo viên upload phiếu trả lời hoặc dùng màn hình scan chấm bài.
2. Backend kiểm tra role hiện tại phải là `teacher`.
3. Backend kiểm tra subscription hiện tại.
4. Nếu giáo viên đang có Prime active:
   - cho phép scan grading không giới hạn.
5. Nếu giáo viên là `free`:
    - kiểm tra `usage_windows` của quota `scan_grading`
    - nếu chưa có cửa sổ đang chạy hoặc cửa sổ cũ đã hết hạn thì lần trừ đầu tiên sẽ mở cửa sổ mới `7 ngày`.
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
  - giáo viên free quay lại giới hạn 40 lượt scan/tuần
  - không xóa lịch sử, lớp học, bài tập, kết quả cũ

### 6.11 Quy tắc cảnh báo quota và paywall
- Khi user free còn 1 lượt tạo đề trong ngày:
  - hiển thị cảnh báo nhẹ ở UI
- Khi giáo viên free còn 5 lượt scan trong cửa sổ quota hiện tại:
  - hiển thị cảnh báo quota sắp hết
- Khi vượt quota:
  - trả lỗi business từ backend
  - UI hiện paywall và gợi ý nâng cấp Prime
- Dashboard phải luôn hiển thị:
  - số đề đã tạo hôm nay
  - số lượt scan đã dùng trong cửa sổ quota hiện tại
  - ngày hết hạn Prime nếu có

### 6.12 Tách dữ liệu luyện tập cá nhân và dữ liệu lớp
- Điểm và kết quả trong `practice mode` là dữ liệu cá nhân của học sinh.
- Giáo viên không mặc định thấy dữ liệu luyện tập cá nhân này.
- Dashboard giáo viên chỉ tổng hợp dữ liệu từ assignment/classroom, trừ khi sau này có tính năng người học tự chia sẻ tiến độ.

### 6.13 Quy tắc admin/manual override
- Admin hoặc backend support nội bộ có thể:
  - mở Prime thủ công khi giao dịch đã nhận nhưng webhook lỗi
  - cộng lại quota nếu hệ thống trừ nhầm
  - đánh dấu payment order là `paid` sau khi đối soát thủ công hợp lệ
- Mọi thao tác override phải có audit log để truy vết.

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
- Chọn cấu trúc đề / đề nguồn
- Chọn lớp
- Chọn thời gian
- Chọn số lần làm (`maxAttempts`)
- Chọn cách hiển thị kết quả (`showResultMode`)
- Chọn chế độ sinh đề:
  - mỗi học sinh một đề riêng
  - hoặc số lượng mã đề giới hạn
- Nếu chọn số lượng mã đề giới hạn:
  - nhập `variantCount`
- Chọn học sinh vào lớp sau có được làm assignment cũ hay không (`allowFutureMembersToAttempt`)
- Card anti-cheat:
  - bật/tắt anti-cheat
  - nếu bật thì mở advanced settings

### 8.6 Thống kê
- Biểu đồ điểm theo assignment
- Biểu đồ phân bố điểm lớp
- Bảng học sinh nguy cơ thấp/trung bình/cao
- Chủ đề yếu theo học sinh và toàn lớp
- Hiển thị trạng thái `force submitted` nếu giáo viên cưỡng ép nộp bài

### 8.7 Billing / Subscription
- Màn hình hiển thị gói hiện tại: Free / Prime Weekly / Prime Monthly / Prime Yearly
- Hiển thị ngày hết hạn Prime
- Hiển thị số lượt tạo đề hôm nay và quota còn lại
- Nếu là giáo viên, hiển thị thêm số `student seats` đã dùng trong cửa sổ quota hiện tại
- Nếu là giáo viên, hiển thị thêm số lượt scan còn lại trong cửa sổ quota hiện tại
- Nút nâng cấp Prime
- Danh sách gói + giá tiền
- Lịch sử thanh toán

### 8.8 Paywall khi hết quota
- Khi user free tạo quá 3 đề/ngày:
  - hiện modal hoặc page chặn
  - nêu rõ đã dùng hết lượt hôm nay
  - CTA nâng cấp Prime
  - CTA xem gói tuần/tháng/năm
- Khi giáo viên free vượt `40 student seats` trong cửa sổ quota `7 ngày` hiện tại:
  - chặn học sinh bắt đầu assignment mới nếu `seatConsumed = false`
  - nêu rõ số seat đã dùng và thời điểm quota hồi lại
  - CTA nâng cấp Prime

### 8.9 Màn hình thanh toán
- Chọn gói
- Hiển thị QR chuyển khoản
- Hiển thị nội dung chuyển khoản bắt buộc
- Trạng thái đơn:
  - chờ thanh toán
  - đã nhận tiền
  - kích hoạt Prime thành công

### 8.10 UI quota scan cho giáo viên
- Hiển thị `Đã dùng 12/40 lượt scan trong cửa sổ hiện tại` với tài khoản teacher free
- Hiển thị `Đã dùng 18/40 student seats trong cửa sổ hiện tại` với tài khoản teacher free
- Hiển thị thời gian còn lại tới lúc quota hồi, ví dụ `Còn 2 ngày 4 giờ`
- Hiển thị badge cảnh báo khi còn ít lượt scan
- Với job lỗi OCR, hiển thị rõ `Không trừ quota` nếu scan chưa thành công
- Có CTA `Nâng cấp Prime` khi chạm quota scan

### 8.11 Cộng đồng lời giải
- `ResultView`
  - nút `Đóng góp lời giải`
  - nút `Báo lời giải sai`
  - nút `Các cách giải khác`
  - drawer/modal hiển thị danh sách lời giải đã duyệt theo vote
  - mỗi item hiển thị: người đóng góp, điểm vote, badge loại lời giải
  - cho phép vote lên/xuống với lời giải đã duyệt
- `TeacherDashboard` hoặc admin moderation screen
  - tab `Kiểm duyệt lời giải`
  - danh sách lời giải `pending`, `approved`, `rejected`
  - khối so sánh `lời giải chuẩn hiện tại` vs `lời giải được đề xuất`
  - action `Duyệt`, `Từ chối`, `Đặt làm lời giải chuẩn`

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
- `PATCH /api/assignments/[id]`
- `PATCH /api/assignments/[id]/schedule`
- `POST /api/assignments/[id]/force-submit`
- `POST /api/assignments/[id]/start`
- `POST /api/assignments/[id]/submit`
- `GET /api/assignments/[id]/variants`
- `GET /api/assignments/[id]/recipients`

### 9.5 Analytics
- `GET /api/classrooms/[id]/analytics`
- `GET /api/students/[id]/analytics`
- `GET /api/assignments/[id]/results`

### 9.6 Billing
- `GET /api/billing/plans`
- `GET /api/billing/subscription`
- `GET /api/billing/usage`
- `GET /api/billing/assignment-usage`
- `GET /api/billing/usage-windows`
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

### 9.9 Community solutions
- `POST /api/community-solutions`
  - gửi lời giải thay thế hoặc đề xuất sửa lời giải
- `GET /api/community-solutions?questionRef=...`
  - lấy danh sách lời giải `approved` của một câu, sort theo vote
- `POST /api/community-solutions/[id]/vote`
  - upvote/downvote một lời giải đã duyệt
- `POST /api/solution-issue-reports`
  - gửi báo cáo lời giải sai, đáp án sai, hoặc diễn giải chưa rõ
- `GET /api/admin/community-solutions`
  - lấy danh sách lời giải chờ duyệt / đã duyệt / bị từ chối
- `PATCH /api/admin/community-solutions/[id]`
  - duyệt, từ chối, hoặc nâng lời giải thành canonical
- `GET /api/admin/solution-issue-reports`
  - lấy danh sách báo lỗi lời giải
- `PATCH /api/admin/solution-issue-reports/[id]`
  - resolve hoặc reject báo cáo

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
- `app/src/components/AssignmentScheduleEditor.tsx`
- `app/src/components/AssignmentForceSubmitDialog.tsx`
- `app/src/components/PricingCard.tsx`
- `app/src/components/UpgradePrimeModal.tsx`
- `app/src/components/BillingDashboard.tsx`
- `app/src/components/PaymentQRCode.tsx`
- `app/src/components/UsageQuotaCard.tsx`
- `app/src/components/ScanUsageBanner.tsx`
- `app/src/components/CommunitySolutionsList.tsx`
- `app/src/components/CommunitySolutionComposer.tsx`
- `app/src/components/SolutionModerationPanel.tsx`
- `app/src/lib/auth.ts`
- `app/src/lib/roles.ts`
- `app/src/lib/classrooms.ts`
- `app/src/lib/assignments.ts`
- `app/src/lib/assignment-recipients.ts`
- `app/src/lib/assignment-force-submit.ts`
- `app/src/lib/analytics.ts`
- `app/src/lib/billing.ts`
- `app/src/lib/subscription.ts`
- `app/src/lib/payment-matching.ts`
- `app/src/lib/usage-tracker.ts`
- `app/src/lib/scan-grading.ts`
- `app/src/lib/community-solutions.ts`
- `app/src/lib/admin-audit.ts`

### 10.2 File sửa
- `app/src/app/page.tsx`
  - thêm auth flow
  - tách teacher/student views
  - bỏ anti-cheat hardcoded khỏi flow assignment
  - chỉ ghi adaptive history khi bài thuộc `self_practice`
  - hiển thị quota tạo đề và upgrade Prime
- `app/src/lib/adaptive.ts`
  - chỉ lưu/đọc record của `self_practice`
  - loại `classroom_assignment` khỏi suggested difficulty và performance summary
- `app/src/lib/schemas.ts`
  - thêm types cho profile, classroom, assignment, assignment variant, assignment recipient, attempt, schedule history, force-submit log, subscription, payment
  - thêm metadata `source` / `origin` để tách `self_practice` khỏi `classroom_assignment`
- `app/src/components/ConfigForm.tsx`
  - tiếp tục dùng adaptive suggestion cho flow tự luyện
- `app/src/components/ResultView.tsx`
  - chỉ hiện adaptive card cho kết quả tự luyện, không hiện trong assignment
  - thêm CTA đóng góp lời giải, báo lỗi lời giải, và danh sách `Các cách giải khác`
- `app/src/components/TeacherDashboard.tsx`
  - thêm tab moderation cho lời giải cộng đồng và báo lỗi lời giải
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
  - thêm các bảng lời giải cộng đồng, vote, và báo lỗi lời giải
  - thêm audit log cho admin override nếu triển khai backend support

### 10.3 Chi tiết cần sửa trong `app/supabase/migration.sql`
- Mở rộng `user_profiles`:
  - thêm `role`, `school_name`, `account_tier`
  - cập nhật trigger `handle_new_user()` để lưu `display_name`, `role` nếu có trong metadata signup
- Thêm bảng lớp học:
  - `classrooms`
  - `classroom_members`
  - index cho `teacher_id`, `join_code`, `classroom_id`, `student_id`
- Thêm bảng cấu trúc đề và assignment:
  - `exam_templates`
  - `assignments`
  - cột quan trọng trong `assignments`: `variant_mode`, `variant_count`, `structure_snapshot`, `anti_cheat_config`, `visible_to_future_members`, `allow_future_members_to_attempt`, `max_attempts`, `show_result_mode`, `open_at`, `due_at`, `duration_minutes`, `schedule_version`, `last_schedule_edited_at`, `last_schedule_edited_by`
- Thêm bảng variant và phân phối assignment:
  - `assignment_variants`
  - `assignment_recipients`
  - `assignment_attempts`
  - `assignment_schedule_history`
  - `assignment_force_submit_logs`
- Trong `assignment_attempts`, cần có các cột snapshot:
  - `duration_snapshot_minutes`
  - `anti_cheat_config_snapshot`
  - `question_order_snapshot`
  - `option_order_snapshot`
  - `status` phải hỗ trợ `force_submitted`
- Thêm quota/billing tables:
  - giữ `daily_usage` cho `practice_generate`
  - thêm `usage_windows` cho rolling window `7 ngày` của `assignment_student_seat` và `scan_grading`
  - `usage_windows` cần có `quota_type`, `limit_count`, `used_count`, `window_started_at`, `window_expires_at`, `assignment_published_count`
  - giữ `subscription_plans`, `user_subscriptions`, `payment_orders`, `payment_transactions`
- Thêm bảng scan/audit nếu chưa đủ:
  - `scan_grading_jobs`
  - `community_solutions`
  - `community_solution_votes`
  - `solution_issue_reports`
  - bảng audit cho admin override nếu làm trong phase này
- Thêm đầy đủ foreign keys, unique constraints và check constraints:
  - unique `classrooms.join_code`
  - unique `(classroom_id, student_id)` cho `classroom_members`
  - unique `(assignment_id, student_id)` cho `assignment_recipients`
  - unique `(community_solution_id, voter_key)` cho `community_solution_votes`
  - check enum text cho `role`, `account_tier`, `variant_mode`, `show_result_mode`, `attempt.status`, `recipient.status`, `subscription.status`, `payment.status`, `scan.status`
- Thêm index cho các query chính:
  - danh sách lớp của giáo viên/học sinh
  - danh sách assignment theo lớp
  - recipients/attempts theo assignment
  - usage window theo `user_id + quota_type`
  - payment theo `user_id`, `status`
  - community solutions theo `question_ref + status + vote_score`
- Bật RLS cho toàn bộ bảng mới và thêm policy:
  - giáo viên chỉ thấy/sửa lớp của mình
  - học sinh chỉ thấy lớp mình tham gia
  - học sinh chỉ đọc recipient/variant/attempt của chính mình
  - giáo viên xem được toàn bộ recipients/attempts/variants trong lớp mình dạy
  - chỉ backend/service role được cập nhật quota, payment paid, force submit log hệ thống nếu cần
  - mọi user đều có thể insert `community_solutions` và `solution_issue_reports`
  - chỉ admin/service role được `approve/reject/promote` community solutions
  - chỉ cho phép insert/update vote của chính `voter_key` qua backend wrapper để tránh ghi bừa
  - với `assignment_recipients`, cần thêm cột `seat_consumed boolean default false` và `seat_consumed_at timestamptz null`

### 10.4 Chi tiết cần sửa trong `app/src/lib/schemas.ts`
- Thêm schema/type cho auth và profile:
  - `UserRole`
  - `AccountTier`
  - `UserProfile`
- Thêm schema/type cho classroom:
  - `Classroom`
  - `ClassroomMember`
- Thêm schema/type cho assignment domain:
  - `ExamTemplate`
  - `AssignmentAntiCheatConfig`
  - `Assignment`
  - `AssignmentVariant`
  - `AssignmentRecipient`
  - `AssignmentAttempt`
  - `AssignmentScheduleHistory`
  - `AssignmentForceSubmitLog`
- Trong `Assignment`, cần đủ các field đã chốt:
  - `maxAttempts`
  - `showResultMode`
  - `variantMode`
  - `variantCount`
  - `structureSnapshot`
  - `visibleToFutureMembers`
  - `allowFutureMembersToAttempt`
  - `scheduleVersion`
- Trong `AssignmentAttempt`, cần đủ:
  - `status` gồm `submitted`, `auto_submitted`, `force_submitted`, `expired`
  - `durationSnapshotMinutes`
  - `antiCheatConfigSnapshot`
  - `questionOrderSnapshot`
  - `optionOrderSnapshot`
- Thêm schema/type cho billing/quota:
  - `SubscriptionPlan`
  - `UserSubscription`
  - `DailyUsage`
  - `UsageWindow`
  - `PaymentOrder`
  - `PaymentTransaction`
- Thêm schema/type cho scan grading:
  - `ScanGradingJob`
- Thêm schema/type cho lời giải cộng đồng:
  - `CommunitySolution`
  - `CommunitySolutionVote`
  - `SolutionIssueReport`
  - `CommunitySolutionType`
  - `CommunitySolutionStatus`
  - `SolutionIssueReportType`
- Thêm enum/type dùng chung để tránh string rời rạc:
  - `AssignmentStatus`
  - `AssignmentRecipientStatus`
  - `AssignmentAttemptStatus`
  - `ShowResultMode`
  - `VariantMode`
  - `UsageQuotaType`
- Cập nhật `AppView` nếu UI role-based và classroom/assignment screens vẫn đi qua state machine ở `page.tsx`
- Trong `AssignmentRecipient`, cần thêm:
  - `seatConsumed`
  - `seatConsumedAt`

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
- Thêm bảng usage window cho scan grading và assignment seats
- Thêm tracking `student seat quota` cho assignment mode
- Seed plan `free`, `prime_weekly`, `prime_monthly`, `prime_yearly`
- Thêm backend check quota
- Hiển thị quota trên UI

**Definition of Done**
- Free user chỉ tạo được tối đa 3 đề/ngày
- Teacher free chỉ có tối đa `40 student seats/tuần`
  - theo nghĩa tối đa 40 học sinh bắt đầu assignment lần đầu trong mỗi rolling window 7 ngày
- Teacher free chỉ scan grading được tối đa 40 bài thành công/tuần
- Prime user tạo đề không giới hạn
- Prime teacher không giới hạn student seats
- Prime user scan grading không giới hạn
- Quota rolling window cho giáo viên hoạt động đúng từ lần dùng đầu tiên
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

- Lưu cấu trúc đề thành `exam_templates`
- Tạo assignment từ cấu trúc đề
- Cấu hình thời gian, số lần làm, hiển thị kết quả
- Cấu hình mode sinh đề `per_student` hoặc `limited_pool`
- Cho giáo viên chọn số lượng mã đề nếu cần
- Tạo `assignment_recipients` cho học sinh hiện có
- Hỗ trợ rule hiển thị assignment cũ cho học sinh vào lớp sau
- Cho giáo viên chọn `maxAttempts`, `showResultMode`, `allowFutureMembersToAttempt`
- Cấu hình anti-cheat ở mức assignment

**Definition of Done**
- Giáo viên giao được bài cho lớp
- Assignment lưu đầy đủ anti-cheat config
- Assignment sinh đúng `assignment_variants` theo cấu hình đã chọn
- Assignment tạo đúng `assignment_recipients` và map đúng học sinh -> variant
- `AssignmentRecipient.seatConsumed` chỉ chuyển sang `true` ở lần start đầu tiên, không phải lúc publish
- Học sinh thấy bài được giao đúng thời gian mở/đóng

### Phase F: Attempt + Chấm điểm + Lưu điểm
**Ưu tiên: Cao**

- Tạo `assignment_attempts`
- Gắn mỗi attempt với đúng `assignment_variant`
- Snapshot `durationMinutes`, anti-cheat, question order, option order khi start attempt
- Submit bài sẽ lưu điểm, đáp án, số lần chuyển tab, trạng thái auto-submit
- Hỗ trợ trạng thái `force_submitted`
- Đồng bộ kết quả với dashboard giáo viên/học sinh

**Definition of Done**
- Học sinh nộp bài thành công
- Điểm được lưu vào DB
- Giáo viên xem được danh sách kết quả
- Giáo viên xem được học sinh làm mã đề nào
- Giáo viên phân biệt rõ ai được giao nhưng chưa làm, ai chỉ được xem, ai vào lớp sau
- Phân biệt được `submitted`, `auto_submitted`, và `force_submitted`

### Phase F2: Schedule Management
**Ưu tiên: Cao**

- Cho giáo viên chỉnh `openAt`, `dueAt`, `durationMinutes`
- Lưu `assignment_schedule_history`
- Giữ attempt đang chạy không bị đổi thời lượng giữa chừng

**Definition of Done**
- Giáo viên sửa được lịch assignment sau publish
- Học sinh thấy lịch mới trên UI
- Attempt đang làm giữ snapshot cũ
- Có audit lịch sử thay đổi

### Phase F3: Force Submit Control
**Ưu tiên: Cao**

- Chặn thao tác đóng assignment nếu còn attempt `in_progress`
- Cho giáo viên `force submit` một hoặc nhiều học sinh đang làm
- Lưu `assignment_force_submit_logs`

**Definition of Done**
- Giáo viên không thể đóng assignment sai rule khi còn học sinh đang làm
- Giáo viên force submit được đúng dữ liệu hiện có
- Attempt chuyển sang `force_submitted`
- UI hiển thị rõ bài bị cưỡng ép nộp

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

### Phase I: Community Solutions + Moderation
**Ưu tiên: Trung bình - cao**

- Thêm bảng/community store cho lời giải cộng đồng, vote, và báo lỗi lời giải
- Thêm UI trong `ResultView` để người dùng gửi lời giải thay thế và báo lỗi lời giải
- Thêm danh sách `Các cách giải khác` theo thứ tự vote
- Thêm moderation panel cho admin/teacher dashboard
- Cho phép admin duyệt, từ chối, hoặc nâng lời giải cộng đồng thành lời giải chuẩn

**Definition of Done**
- Người dùng gửi được lời giải thay thế mà không sửa dữ liệu chính ngay lập tức
- Người dùng báo được lời giải sai
- Chỉ lời giải `approved` mới xuất hiện công khai
- Danh sách lời giải công khai được sort đúng theo vote cao xuống thấp
- Mỗi lời giải công khai hiển thị người đóng góp và điểm vote
- Admin duyệt/từ chối/promote được từ UI moderation

## 12. Kiểm thử bắt buộc

### 12.1 Unit
- Auth helpers
- Role guards
- Assignment anti-cheat config parser
- Analytics calculators
- Attempt status transitions
- Quy tắc chặn đóng assignment khi còn attempt `in_progress`
- Quy tắc `force submit` chuyển trạng thái đúng sang `force_submitted`
- Tính quota free vs prime
- Tính `student seat quota` cho teacher free
- Tính quota teacher free scan 40/tuần
- Tính đúng `windowStartedAt` và `windowExpiresAt` từ lần trừ đầu tiên
- Tính đúng rule `seatConsumed` chỉ trừ 1 lần cho mỗi `studentId + assignmentId`
- Tính `expiresAt`
- Match transaction theo transfer code
- Chặn duplicate payment processing
- Chỉ tăng scan quota khi `scan_success`
- Manual grading fallback không làm trừ thêm quota scan
- Admin override ghi đúng audit log
- Tính `voteScore` đúng khi upvote/downvote và khi user đổi phiếu
- Chặn một `voterKey` vote trùng trên cùng một lời giải
- Sort danh sách lời giải công khai đúng theo `voteScore` rồi thời gian
- Promote lời giải cộng đồng thành canonical không làm mất lịch sử lời giải cũ

### 12.2 Integration
- Signup -> profile created
- Teacher creates classroom -> student joins
- Teacher publishes assignment -> student starts
- Student submits -> score stored -> teacher sees result
- Teacher free publish assignment -> thành công dù chưa trừ seat
- Teacher free student start first attempt trong giới hạn 40 seats/tuần -> thành công
- Teacher free student start first attempt vượt 40 seats/tuần -> bị chặn
- Teacher free dùng seat/scans lần đầu sau khi quota hồi -> mở cửa sổ quota mới đúng thời điểm
- Student joins class after publish -> still sees old assignment
- Student joins class after publish -> only attempts old assignment when `allowFutureMembersToAttempt = true`
- Teacher edits assignment schedule -> future attempts use new duration, active attempts keep old snapshot
- Teacher cannot close assignment while students are still `in_progress`
- Teacher force submits a running attempt -> result stored with `force_submitted`
- Free user generate 3 lần thành công, lần 4 bị chặn
- Prime user generate không giới hạn
- Teacher free scan 40 bài thành công, lần 41 bị chặn
- Scan lỗi không làm tăng quota teacher free
- Manual grading từ scan lỗi không tăng thêm quota scan
- Tạo payment order -> nhận transaction hợp lệ -> Prime active
- Prime hết hạn -> quay lại free
- User gửi community solution -> trạng thái `pending`
- Admin duyệt community solution -> lời giải xuất hiện trong `Các cách giải khác`
- User vote lời giải đã duyệt -> thứ tự hiển thị thay đổi đúng theo vote
- User báo lời giải sai -> admin resolve được từ moderation panel

### 12.3 Manual E2E
1. Đăng ký giáo viên
2. Tạo lớp
3. Đăng ký học sinh
4. Học sinh join lớp
5. Giáo viên tạo assignment có anti-cheat OFF
6. Học sinh làm bài, xác nhận không có enforcement anti-cheat
7. Giáo viên tạo assignment có anti-cheat ON
8. Học sinh làm bài, xác nhận enforcement hoạt động
9. Một học sinh mới join lớp sau khi assignment đã publish vẫn nhìn thấy assignment cũ
10. Xác nhận học sinh mới không làm được assignment cũ nếu `allowFutureMembersToAttempt = false`
11. Giáo viên bật cho phép thành viên vào sau được làm bài, xác nhận học sinh mới được tạo recipient/variant đúng
12. Giáo viên chỉnh lại `dueAt` và `durationMinutes`
13. Xác nhận học sinh chưa bắt đầu thấy lịch mới, học sinh đang làm giữ `durationSnapshotMinutes` cũ
14. Mở kết quả bài làm và gửi một lời giải thay thế
15. Xác nhận lời giải mới chưa hiện công khai khi chưa duyệt
16. Admin vào moderation panel và duyệt lời giải đó
17. Quay lại `ResultView`, mở `Các cách giải khác`, xác nhận lời giải đã xuất hiện kèm tên người đóng góp
18. Vote cho 2 lời giải khác nhau và xác nhận danh sách được sắp theo vote giảm dần
19. Thử đóng assignment khi còn học sinh đang làm, xác nhận hệ thống chặn thao tác đó
20. Giáo viên dùng `force submit`, xác nhận bài chuyển sang `force_submitted` và học sinh thấy rõ lý do bị kết thúc bài
21. Giáo viên free publish assignment, xác nhận chưa bị trừ seat nếu chưa có học sinh nào bấm làm
22. Đủ 40 học sinh bắt đầu attempt đầu tiên trong cửa sổ quota hiện tại
23. Học sinh thứ 41 bấm làm lần đầu bị chặn và hiện paywall
24. Một học sinh đã dùng seat làm lại lần 2/lần 3, xác nhận không trừ thêm seat
25. Chờ quota window hết hạn hoặc mô phỏng hết hạn, xác nhận lần dùng tiếp theo mở cửa sổ mới từ thời điểm bị trừ đầu tiên
26. Tạo tài khoản free và generate 3 đề thành công
27. Tạo đề lần 4 bị chặn
28. Đăng nhập tài khoản giáo viên free và scan 40 bài thành công
29. Scan bài thứ 41 bị chặn và hiện paywall
30. Thử scan lỗi OCR, xác nhận không trừ quota
31. Mua Prime tháng
32. Hệ thống xác nhận thanh toán
33. Tài khoản được mở Prime
34. Tạo đề tiếp không giới hạn, học sinh bắt đầu assignment không giới hạn seat, và scan không giới hạn
35. Giáo viên mở dashboard xem điểm, thống kê, và usage quota

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

### 13.8 Rủi ro riêng của student seat quota
- Giáo viên khó hiểu vì sao một assignment cho lớp đông hết quota nhanh
  - Giảm thiểu: hiển thị rõ `1 student seat = 1 học sinh bắt đầu làm 1 assignment lần đầu`
- Học sinh vào lớp sau làm phát sinh thêm seat ngoài dự kiến
  - Giảm thiểu: hiển thị cảnh báo trước khi bật `allowFutureMembersToAttempt`
- Publish lỗi nhưng vẫn trừ quota
  - Giảm thiểu: seat không bị trừ lúc publish; chỉ trừ ở lần start đầu tiên
- Cửa sổ quota bị tính sai thời điểm bắt đầu hoặc hết hạn
  - Giảm thiểu: chuẩn hóa `windowStartedAt/windowExpiresAt` ở backend và test edge case hết hạn kỹ
- Học sinh bấm bắt đầu làm nhưng bị chặn do giáo viên hết seat
  - Giảm thiểu: hiển thị quota còn lại cho giáo viên từ trước và cảnh báo rõ ở UI học sinh khi bị chặn

### 13.9 Rủi ro riêng của force submit
- Giáo viên kết thúc bài nhầm khi học sinh vẫn đang làm
  - Giảm thiểu: cần confirm dialog + hiển thị rõ số học sinh bị ảnh hưởng
- Học sinh không hiểu vì sao bài bị nộp đột ngột
  - Giảm thiểu: hiển thị banner/trạng thái `Bài đã bị giáo viên kết thúc` trong UI và kết quả

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

## 15. Quyết định sản phẩm đã khóa và phần còn cần chốt

### 15.1 Đã khóa
1. Assignment mặc định dùng `mỗi học sinh một đề riêng`.
2. Giáo viên có thể chọn `số lượng mã đề/variant` nếu không muốn sinh riêng hoàn toàn cho từng học sinh.
3. Học sinh vào lớp sau vẫn nhìn thấy assignment cũ đã publish.
4. Mặc định học sinh vào lớp sau không được làm assignment cũ trừ khi assignment cho phép rõ ràng.
5. Giáo viên được chỉnh `openAt`, `dueAt`, `durationMinutes` sau khi publish.
6. Thời lượng mới chỉ áp dụng cho attempt bắt đầu sau khi chỉnh; attempt đang chạy giữ snapshot cũ.
7. `maxAttempts` là cấu hình do giáo viên chọn trên từng assignment.
8. `showResultMode` là cấu hình do giáo viên chọn trên từng assignment.
9. `allowFutureMembersToAttempt` là cấu hình do giáo viên chọn trên từng assignment.
10. Giáo viên không được đóng assignment nếu còn học sinh đang làm; nếu cần kết thúc sớm thì dùng `force submit`.
11. `teacher free` có giới hạn `40 student seats/tuần` và đây là chủ ý sản phẩm để phân biệt gói trả phí.
12. `assignmentPublishedCount` hiện chỉ dùng cho analytics/billing nội bộ, chưa phải quota chặn chính.
13. `student seat` chỉ bị trừ khi học sinh bắt đầu làm assignment lần đầu, không trừ lúc publish.
14. Học sinh làm lại cùng assignment không bị trừ thêm seat.
15. Flow luyện tập cá nhân tiếp tục giữ adaptive riêng; assignment/lớp học không cập nhật adaptive difficulty cá nhân, chỉ lưu thống kê và analytics lớp.

### 15.2 Còn cần chốt
1. Học sinh có được tự đăng ký tài khoản không, hay giáo viên tạo trước?
2. Học sinh tham gia lớp bằng mã mời là đủ hay cần giáo viên duyệt?
3. Giá Prime tuần / tháng / năm là bao nhiêu?
4. Reset quota theo múi giờ Việt Nam hay UTC?
5. Chỉ hỗ trợ chuyển khoản ngân hàng hay tích hợp thêm cổng thanh toán ngay từ đầu?
6. Kích hoạt Prime tự động hoàn toàn hay giai đoạn đầu cho phép admin duyệt giao dịch lỗi?
7. Prime có áp dụng cho cả giáo viên và học sinh, hay chỉ một nhóm người dùng?
8. Manual grading fallback có được mở ngay ở phase đầu của scan grading không?
9. Admin override sẽ dùng tool nội bộ đơn giản hay cần giao diện admin ngay từ đầu?

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
