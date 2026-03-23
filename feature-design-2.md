# Feature Design Document

---

## I. Leaderboard

Tính năng Leaderboard cho phép học sinh theo dõi thứ hạng của mình so với các học sinh khác trong cùng trường, nhóm, hoặc toàn hệ thống. Mục tiêu là tạo động lực học tập thông qua sự cạnh tranh lành mạnh — học sinh có thể thấy mình đang đứng ở đâu, cần bao nhiêu điểm để vươn lên thứ hạng tiếp theo, và thứ hạng của mình đang tăng hay giảm so với kỳ trước.

### Filters

| Filter | Options | Default |
|--------|---------|---------|
| Cá nhân / nhóm | `TAK12` · `Nhóm của bạn` | TAK12 |
| Thời gian | `Tuần` · `Tháng` · `Tất cả` | Tuần |
| Phạm vi | Tỉnh · Trường · Nhóm | — |

> **Lưu ý:** Filter được lưu theo lựa chọn cuối cùng của người dùng, giúp học sinh không phải thiết lập lại mỗi lần vào xem.

---

### API

Leaderboard sử dụng một endpoint duy nhất. Các filter được truyền qua request để server tính toán và trả về danh sách xếp hạng phù hợp. `schoolId` và `groupId` là tuỳ chọn — nếu không truyền, hệ thống mặc định trả về bảng xếp hạng toàn quốc theo `timeRange`.

#### Request

```ts
{
  userId:    number   // Dùng để xác định vị trí của người dùng hiện tại trong bảng xếp hạng
  schoolId?: number   // Lọc theo trường — chỉ hiển thị học sinh cùng trường
  groupId?:  number   // Lọc theo nhóm — chỉ hiển thị thành viên trong nhóm đó
  timeRange: string   // "week" | "month" | "all" — phạm vi thời gian tính điểm
}
```

#### Response — `Student[]`

```ts
Student {
  id:            number   // ID học sinh
  email:         string
  point:         float    // Tổng điểm tích lũy trong khoảng timeRange
  schoolId:      number
  schoolName:    string
  rank:          number   // Thứ hạng hiện tại
  rankShifted:   number   // Thay đổi thứ hạng so với kỳ trước (dương = tăng, âm = giảm, 0 = giữ nguyên)
  pointToRankUp: number   // Số điểm còn thiếu để vượt lên thứ hạng liền trên
}
```

---

### Review

- ✅ `userId` không nên truyền từ client — server tự lấy từ session (`AbpSession.GetUserId()` hoặc `CthSession.UserGuid` tùy API).
- **Phân trang?** Không cần — leaderboard chỉ hiển thị khoảng 10–14 user trên UI.
- ✅ Filter theo Tỉnh đã có trong thiết kế.
- ⚠️ `rankShifted` — chưa thiết kế cơ chế lưu trữ lịch sử. Cần làm rõ "kỳ trước" là gì (tuần trước? tháng trước?) và dữ liệu lịch sử lưu ở đâu. **Chưa có giải pháp.**
- **Nguồn dữ liệu `point`:** Tính trực tiếp từ `QuizAttemptQuestions`.

---

## II. Group

Tính năng Group cho phép học sinh tạo và tham gia nhóm học tập riêng — ví dụ nhóm bạn cùng lớp, cùng trường, hoặc nhóm ôn thi. Sau khi có nhóm, học sinh có thể xem leaderboard nội bộ và theo dõi tiến độ của từng thành viên. Đây là lớp xã hội hoá quan trọng giúp tăng gắn kết và động lực học tập.

---

### FE — Danh sách nhóm

Màn hình đầu tiên khi vào mục Group. Hiển thị tất cả nhóm mà người dùng hiện đang là thành viên, kèm thông tin tóm tắt để học sinh chọn nhóm muốn xem chi tiết.

**Request**
```ts
{ userId: number }
```

**Response — `groups[]`**
```ts
{
  groupId:          number
  groupName:        string
  groupDescription: string
  imageUrl:         string
  memberCount:      number   // Hiển thị số thành viên trực tiếp trên thẻ nhóm
}
```

---

### FE — CRUD nhóm

Cho phép người dùng tự quản lý nhóm của mình: tạo nhóm mới, mời thành viên, cập nhật thông tin, hoặc giải tán nhóm. Người tạo nhóm mặc định là admin của nhóm đó.

#### `GET` — Danh sách nhóm của user
```ts
Request: { userId: number }
```

#### `GET ?id` — Chi tiết một nhóm (dùng để hiển thị form chỉnh sửa)
```ts
Request: {
  userId:           number
  groupId:          number
  groupName:        string
  groupDescription: string
  imageUrl:         string
  group_members:    GroupMember[]
}
```

#### `POST` — Tạo nhóm mới
```ts
Request: {
  userId:      number    // Người tạo — tự động trở thành admin nhóm
  groupName:   string
  memberIds:   number[]  // Danh sách userId được mời vào nhóm ngay lúc tạo
  imageBase64: string    // Ảnh đại diện nhóm, encode base64
}
```

#### `PUT` — Cập nhật thông tin nhóm
```ts
Request: {
  userId:      number    // Phải là admin của nhóm
  groupId:     number
  groupName:   string
  memberIds:   number[]  // Danh sách thành viên sau khi cập nhật (thêm hoặc xoá)
  imageBase64: string
}
```

#### `DELETE` — Giải tán nhóm
```ts
Request: { userId: number, groupId: number }
```

#### Schema — `GroupMember`
```ts
{
  userId:       number
  emailAddress: string
  imageUrl:     string
  schoolName:   string
  schoolId:     number
  joinedDate:   string   // Ngày thành viên gia nhập nhóm
}
```

---

### FE — Chi tiết nhóm

Màn hình chính khi vào một nhóm cụ thể. Có hai lớp dữ liệu được tải riêng: thông tin tổng quan của nhóm (nhẹ, tải ngay), và bảng xếp hạng thành viên (nặng hơn, có phân trang).

#### `GET ?id&countTotalOnly=true` — Thông tin tổng quan nhóm

Endpoint này trả về thông tin tóm tắt của nhóm mà không tải toàn bộ danh sách thành viên. Cũng được tái sử dụng khi hiển thị trang xác nhận lời mời tham gia nhóm — người được mời có thể xem tên, mô tả và số thành viên trước khi chấp nhận.

```ts
Response: {
  userId:           number
  userEmail:        string
  groupId:          number
  groupDescription: string
  imageUrl:         string
  memberCount:      number
}
```

#### `GET getDetailedGroupMembers?groupId=` — Bảng xếp hạng thành viên

Trả về danh sách thành viên kèm điểm số, thứ hạng và streak — tương tự leaderboard nhưng giới hạn trong phạm vi nhóm. Hỗ trợ lọc theo bài thi và thời gian, phân trang để tránh tải quá nhiều dữ liệu một lúc.

**Request**
```ts
{
  timeRange: string   // "week" | "month" | "all"
  examId:    number   // Lọc điểm theo bài thi cụ thể (tuỳ chọn)
  groupId:   number
  page:      number   // Phân trang
}
```

**Response — `students[]`**
```ts
{
  userId:       number
  emailAddress: string
  points:       number
  rank:         number
  rankShifted:  number   // Thay đổi thứ hạng so với kỳ trước
  streakCount:  number   // Chuỗi ngày học liên tiếp hiện tại của thành viên
  joinedDate:   string
}
```

---

### Admin — Quản lý nhóm

Admin tái sử dụng toàn bộ các API của FE để thêm, sửa, xoá nhóm. Ngoài ra, admin có thêm khả năng giám sát toàn bộ nhóm trên hệ thống — hữu ích khi cần kiểm tra nhóm có nội dung không phù hợp hoặc hỗ trợ người dùng gặp sự cố.

Quyền bổ sung của admin:

- Xem danh sách **tất cả** nhóm trên hệ thống (không giới hạn theo userId)
- Tìm kiếm nhóm theo `groupId`
- Tìm kiếm nhóm theo `userId` hoặc `email` của người tạo

---

### Database Schema

#### Table: `Groups`

| Column | Type | Constraints | Ghi chú |
|--------|------|-------------|---------|
| `id` | `INT` | `PK`, `IDENTITY` | |
| `name` | `NVARCHAR(200)` | `NOT NULL` | |
| `description` | `NVARCHAR(MAX)` | `NULL` | |
| `adminUserId` | `INT` | `FK → Users(id)`, `NOT NULL` | Người tạo — mặc định là admin nhóm |
| `imageUrl` | `NVARCHAR(500)` | `NULL` | |
| `creationTime` | `DATETIME2` | `NOT NULL`, `DEFAULT SYSUTCDATETIME()` | |

#### Table: `GroupUsers`

| Column | Type | Constraints | Ghi chú |
|--------|------|-------------|---------|
| `userId` | `INT` | `FK → Users(id)`, `NOT NULL` | |
| `groupId` | `INT` | `FK → Groups(id)`, `NOT NULL` | |
| `creationTime` | `DATETIME2` | `NOT NULL`, `DEFAULT SYSUTCDATETIME()` | |

> **Primary key:** `PK_GroupUsers` composite trên `(userId, groupId)`.

---

### Review

- ✅ Database schema đã được thiết kế (xem bảng `Groups` và `GroupUsers` ở trên).
- ✅ Tách API cập nhật thành viên để tránh race condition khi nhiều client gọi đồng thời:
  - `POST addGroupMembers    { groupId, userIds: number[] }`
  - `POST removeGroupMembers { groupId, userIds: number[] }`
- **Cơ chế mời thành viên:** Tự động thêm ngay, không cần accept invitation — tương tự AZVocab.
- **Giới hạn hệ thống** (số nhóm tối đa có thể tạo, số thành viên tối đa/nhóm, số nhóm tối đa user có thể tham gia): Chưa có trong business requirement, sẽ bổ sung sau.
- **Group admin bị xóa tài khoản:** Không xảy ra — trên TAK12 không có chức năng xoá tài khoản.

---

## III. Streak

Streak là số ngày học liên tiếp của học sinh. Đây là một trong những cơ chế giữ chân người dùng mạnh nhất — khi học sinh đã có chuỗi dài, họ sẽ không muốn để nó bị đứt. Tính năng này bao gồm: hiển thị thống kê streak trên dashboard, cơ chế "đóng băng" streak (streak freeze) để bảo vệ chuỗi trong những ngày bận, và hệ thống nhắc nhở tự động.

---

### FE — Dashboard

#### `getUserStreakInfo` — Thông tin streak tổng quan

API chính của màn hình dashboard streak. Trả về cả dữ liệu cá nhân lẫn dữ liệu so sánh với toàn hệ thống (`top*`), giúp học sinh thấy mình đang đứng ở đâu so với những người học tốt nhất.

```ts
Response: {
  streak:                   number    // Chuỗi hiện tại (ngày)
  maxStreak:                number    // Chuỗi dài nhất mà user từng đạt được
  topStreak:                number    // Chuỗi dài nhất trên toàn hệ thống (để so sánh)
  maxLearningTimeInMinutes: number    // Thời gian học dài nhất trong 1 ngày của user
  topLearningTime:          number    // Thời gian học dài nhất trong 1 ngày của toàn hệ thống
  totalPoints:              number    // Tổng điểm tích lũy của user
  topTotalPoints:           number    // Tổng điểm cao nhất trên toàn hệ thống
  streakDays:               string[]  // Derived: [lastLearnedDate - streak + 1 … lastLearnedDate] — không lưu DB
  frozenStreakDays:          string[]  // Query từ UserStreakFreezes — chỉ ngày freeze đã tiêu thụ (30 ngày gần nhất)
  nextStreakAchievement: {
    id:        number
    name:      string
    imagePath: string
    goal:      number   // Số ngày streak cần đạt để mở khoá danh hiệu tiếp theo
  }
}
```

#### `getUserStreakFreezeInfo` — Thông tin streak freeze

Streak freeze cho phép học sinh "bảo vệ" chuỗi streak trong một ngày không học, bằng cách đổi credit lấy freeze. API này trả về số lượng freeze đang có và số dư credit, giúp UI hiển thị nút "Mua freeze" với đầy đủ thông tin.

```ts
Response: {
  streakFreezeCount:     number   // Số lượt freeze còn lại
  creditBalance:         number   // Số credit hiện tại của user
  creditPerStreakFreeze: number   // Chi phí mỗi lần mua freeze (để hiển thị giá)
}
```

#### `redeemStreakFreeze` — Đổi credit lấy streak freeze

Được gọi khi học sinh bấm "Mua streak freeze". Server kiểm tra số dư credit, trừ credit, và cộng thêm một lượt freeze. Response trả về số dư mới để UI cập nhật ngay mà không cần reload.

```ts
Request:  { userId: number }
Response: {
  remainCredit:      number   // Số credit còn lại sau khi đổi
  streakFreezeCount: number   // Số lượt freeze sau khi cộng thêm
}
```

---

### Admin — Cài đặt nhắc nhở

Bổ sung một `applicationSettingKey` mới để cấu hình thời điểm gửi thông báo nhắc học sinh trước khi ngày học kết thúc. Thay vì hardcode giờ nhắc, admin có thể điều chỉnh linh hoạt mà không cần deploy lại code.

| Key | Kiểu | Mô tả |
|-----|------|-------|
| `streakReminderOffsetHours` | `number` | Số giờ trước khi hết ngày học, hệ thống gửi push notification nhắc học sinh học bài để không mất streak. Ví dụ: giá trị `3` nghĩa là nhắc lúc 21:00 nếu ngày học kết thúc lúc 00:00. |

---

### Database Schema

#### Table: `UserStreak`

Một hàng trên mỗi user. Lưu trạng thái streak pre-computed — đọc O(1), không cần aggregate.

| Column | Type | Constraints | Ghi chú |
|--------|------|-------------|----------|
| `userId` | `INT` | `PK`, `FK → Users(id)` | |
| `streak` | `INT` | `NOT NULL`, `DEFAULT 0` | Chuỗi ngày học hiện tại |
| `maxStreak` | `INT` | `NOT NULL`, `DEFAULT 0` | Chuỗi dài nhất từng đạt |
| `lastLearnedDate` | `DATE` | `NULL` | Ngày học gần nhất — dùng để suy ra `streakDays[]` và detect miss |
| `streakFreezeCount` | `INT` | `NOT NULL`, `DEFAULT 0` | Số lượt freeze còn lại |

> **Write path:** `streak`, `maxStreak`, `lastLearnedDate` cập nhật inline khi nộp quiz. `streakFreezeCount` tăng khi user mua freeze (`redeemStreakFreeze`), giảm khi `StreakFreezeApplyJob` tự động tiêu thụ.

#### Table: `UserStreakFreezes`

Chỉ lưu các ngày freeze **đã được tiêu thụ** — không lưu mọi ngày học. Theo kiến trúc Duolingo: `streakDays[]` được **suy ra** từ `lastLearnedDate + streak`, còn `frozenStreakDays[]` query từ bảng này.

| Column | Type | Constraints | Ghi chú |
|--------|------|-------------|----------|
| `userId` | `INT` | `FK → Users(id)`, `NOT NULL` | |
| `frozenDate` | `DATE` | `NOT NULL` | Ngày được bảo vệ bởi freeze (ngày hôm qua khi job chạy) |

> **Primary key:** Composite `(userId, frozenDate)` — tự enforce idempotency, không cần check trước khi INSERT.
> **Retention:** Xoá hàng cũ hơn 90 ngày — calendar chỉ hiển thị 30 ngày, 90 ngày là đủ headroom. `UserStreakCleanupJob` chạy hàng tuần.

**Calendar derivation at read time:**
```sql
-- streakDays[]: suy ra từ hai cột trong UserStreak — không cần bảng log
streakDays = [lastLearnedDate - streak + 1  →  lastLearnedDate]

-- frozenStreakDays[]: query từ UserStreakFreezes
SELECT frozenDate FROM UserStreakFreezes
WHERE userId = @userId
  AND frozenDate >= DATEADD(day, -30, GETUTCDATE())
```

Không lưu log từng ngày học → **không có bảng phình to theo DAU**.
Tại 500k users, 10% dùng freeze mỗi ngày → ~50k rows/ngày, retention 90 ngày → tối đa ~4.5M rows.

---

### Review

- ✅ **Database schema** đã thiết kế — xem `UserStreak` và `UserStreakFreezes` ở trên.
- ⚠️ **Timezone:** Streak tính theo ngày — cần thống nhất timezone (UTC hay local time?) trước khi implement.
- ⚠️ **Credit conflict:** Hệ thống credit hiện tại đang xung đột với AI credit — cần tạo hệ thống credit/coin riêng cho gamification.
- **Streak freeze mechanism:** Job tự động chạy vào đầu mỗi ngày — nếu user không học ngày hôm trước và còn lượt freeze, job tự động trừ một lượt để bảo vệ streak (không cần user bấm thủ công).

---

## IV. Achievement

Hệ thống Achievement (danh hiệu) là lớp gamification chính của nền tảng. Học sinh nhận danh hiệu khi đạt các cột mốc cụ thể — duy trì streak dài, leo lên top leaderboard, hoàn thành bài thi với tỉ lệ cao, hoặc được admin trao thủ công. Danh hiệu vừa là phần thưởng, vừa là tín hiệu xã hội khi học sinh trưng bày trên profile.

---

### Data Models

#### `Achievement` — Định nghĩa danh hiệu

```ts
{
  name:         string
  description:  string
  type:         1 | 2 | 3 | 4
  // 1 = streak      — trao khi đạt số ngày streak liên tiếp
  // 2 = leaderboard — trao khi đạt thứ hạng nhất định
  // 3 = completion  — trao khi hoàn thành bài thi với điều kiện cụ thể (dùng AchievementExams)
  // 4 = manual      — admin trao tay, không có điều kiện tự động
  isActive:     boolean   // Danh hiệu không active sẽ không được trao mới, nhưng user đã có vẫn giữ
  imagePath:    string
  creditReward: number    // Số credit thưởng khi đạt danh hiệu
  tierId:       number    // Xếp danh hiệu vào tier (Bronze, Silver, Gold, ...)
  requirements: {
    requiredStreakDays?:           number   // type = 1 — số ngày streak liên tiếp cần đạt
    requiredLeaderboardRank?:     number   // type = 2 — thứ hạng leaderboard cần đạt
    requiredCompletionPercentage?: number  // type = 3 — % hoàn thành tối thiểu
    requiredCompletionCount?:     number   // type = 3 — số lần hoàn thành đủ điều kiện
    countType?:                   number   // 1 = by quizzes, 2 = by questions
    awardingInterval?:            number   // 1 = Day, 2 = Week, 3 = Month, 4 = Quarter, 5 = Year
  }
}
```

#### `AchievementTier` — Phân cấp danh hiệu

```ts
{ id: number, name: string }
// Ví dụ: { id: 1, name: "Bronze" }, { id: 2, name: "Silver" }, { id: 3, name: "Gold" }
```

#### `AchievementExams` — Liên kết danh hiệu với bài thi

Chỉ áp dụng cho danh hiệu `type = 3`. Mỗi hàng liên kết một danh hiệu với một bài thi cụ thể. Toàn bộ điều kiện hoàn thành được lưu trong trường `requirements` của bảng `Achievement`.

```ts
{
  examId:        number
  achievementId: number
}
```

#### `UserAchievements` — Danh hiệu đã trao

Mỗi hàng là một lần trao danh hiệu cho một học sinh. Không xoá record khi danh hiệu bị deactivate — lịch sử luôn được giữ nguyên.

```ts
{
  userId:        number
  achievementId: number
  receivedDate:  string   // Timestamp UTC khi danh hiệu được trao
  note:          string   // Lý do khi admin trao thủ công (type = 4), để trống nếu trao tự động
}
```

---

### FE — Tủ danh hiệu

#### `getUserAchievements` — Lấy danh sách danh hiệu của học sinh

Hiển thị toàn bộ danh hiệu mà học sinh đã nhận được, sắp xếp theo `receivedDate` mới nhất. Học sinh có thể chọn tối đa 3 danh hiệu để ghim lên profile — những danh hiệu này hiển thị công khai với các học sinh khác.

```ts
Response: {
  achievements: {
    id:           number
    name:         string
    description:  string
    imagePath:    string
    receivedDate: string
  }[]
}
```

> Danh hiệu được ghim lên profile lưu vào `[UserSettings].[ShowcaseAchievementIds]` (tối đa 3 giá trị).

---

### Admin — Quản lý danh hiệu

Admin có toàn quyền tạo, chỉnh sửa, ẩn/hiện và trao tay danh hiệu. Việc xoá danh hiệu cần cẩn thận — nên dùng `isActive = false` thay vì xoá thật để không ảnh hưởng đến lịch sử của học sinh đã nhận.

| Method | Endpoint | Mô tả |
|--------|----------|-------|
| `GET` | `getAllAchievements` | Lấy toàn bộ danh hiệu (kể cả inactive) |
| `POST` | `achievement` | Tạo danh hiệu mới |
| `PUT` | `achievement` | Cập nhật thông tin hoặc điều kiện danh hiệu |
| `DELETE` | `achievement` | Xoá vĩnh viễn (chỉ dùng khi chưa có user nào nhận) |

#### `grantAchievementToUser` — Trao danh hiệu thủ công

Dùng cho các trường hợp đặc biệt mà hệ thống không thể tự động phát hiện — ví dụ: học sinh đạt thành tích trong sự kiện offline, hoặc được thưởng danh hiệu theo chương trình khuyến mãi. Trường `note` bắt buộc để lưu lý do kiểm tra sau.

```ts
Request: {
  achievementId: number
  userId:        number
  note:          string   // Lý do trao — bắt buộc khi trao thủ công
}
```

---

### Review

- ✅ `type` nên dùng Enum thay vì số trực tiếp để tránh magic number.
- **Cơ chế tự động gán:** Trigger ngay sau khi kết thúc phiên ôn luyện hoặc nộp bài quiz — kiểm tra điều kiện và trao danh hiệu tại thời điểm đó.
- ⚠️ **One-time vs repeatable:** Cần làm rõ achievement nào có thể nhận nhiều lần (ví dụ: leaderboard hàng tuần), achievement nào chỉ nhận một lần (ví dụ: streak milestone). Chưa có cơ chế phân biệt trong thiết kế hiện tại.

---

## V. Database Schema — Achievement Module

### Table: `AchievementTier`

Phân cấp danh hiệu (Bronze, Silver, Gold, …).

| Column | Type | Constraints | Ghi chú |
|--------|------|-------------|---------|
| `id` | `INT` | `PK`, `IDENTITY` | |
| `name` | `NVARCHAR(100)` | `NOT NULL` | Tên tier, vd: "Bronze", "Gold" |

---

### Table: `Achievement`

Định nghĩa từng loại danh hiệu.

| Column | Type | Constraints | Ghi chú |
|--------|------|-------------|---------|
| `id` | `INT` | `PK`, `IDENTITY` | |
| `name` | `NVARCHAR(200)` | `NOT NULL` | |
| `description` | `NVARCHAR(MAX)` | `NULL` | |
| `type` | `TINYINT` | `NOT NULL` | `1`=streak · `2`=leaderboard · `3`=completion · `4`=manual |
| `isActive` | `BIT` | `NOT NULL`, `DEFAULT 1` | |
| `imagePath` | `NVARCHAR(500)` | `NULL` | |
| `creditReward` | `INT` | `NOT NULL`, `DEFAULT 0` | Số credit thưởng khi đạt danh hiệu |
| `tierId` | `INT` | `FK → AchievementTier(id)`, `NULL` | |
| `requirements` | `NVARCHAR(MAX)` | `NULL` | JSON chứa điều kiện: `requiredStreakDays`, `requiredLeaderboardRank`, `requiredCompletionPercentage`, `requiredCompletionCount`, `countType`, `awardingInterval` |

> **Index gợi ý:** `IX_Achievement_Type` trên cột `type` để lọc nhanh theo loại.

---

### Table: `AchievementExams`

Liên kết giữa danh hiệu loại `completion` (`type = 3`) và các bài thi cụ thể. Điều kiện hoàn thành được lưu trong trường `requirements` của bảng `Achievement`.

| Column | Type | Constraints | Ghi chú |
|--------|------|-------------|---------|
| `id` | `INT` | `PK`, `IDENTITY` | |
| `achievementId` | `INT` | `FK → Achievement(id)`, `NOT NULL` | |
| `examId` | `INT` | `NOT NULL` | Tham chiếu tới bảng Exam |

> **Unique constraint gợi ý:** `UQ_AchievementExams_Achievement_Exam` trên `(achievementId, examId)` để tránh cấu hình trùng.

---

### Table: `UserAchievements`

Lưu danh hiệu đã được trao cho từng học sinh.

| Column | Type | Constraints | Ghi chú |
|--------|------|-------------|---------|
| `id` | `INT` | `PK`, `IDENTITY` | |
| `userId` | `INT` | `FK → Users(id)`, `NOT NULL` | |
| `achievementId` | `INT` | `FK → Achievement(id)`, `NOT NULL` | |
| `receivedDate` | `DATETIME` | `NOT NULL`, `DEFAULT GETUTCDATE()` | |
| `note` | `NVARCHAR(500)` | `NULL` | Ghi chú khi admin trao thủ công (`type = 4`) |

> **Index gợi ý:** `IX_UserAchievements_UserId` trên `userId` để query nhanh tủ danh hiệu của một user.

---

### Quan hệ giữa các bảng

```
AchievementTier (1) ──────── (N) Achievement
Achievement     (1) ──────── (N) AchievementExams
Achievement     (1) ──────── (N) UserAchievements
Users           (1) ──────── (N) UserAchievements
Exams           (1) ──────── (N) AchievementExams
```

---

## VI. Hangfire Jobs — Achievement Module

Achievement granting uses two models depending on type:

- **Event-driven (inline):** Called directly from the quiz submission service the moment a condition is met. No scheduling needed.
- **Scheduled (Hangfire):** Requires data across all users or the full period to be closed before evaluation. Registered as `RecurringJob` with a cron expression.

### Event-driven — Triggered inline after quiz submission

| Handler | Achievement Types | Repeatable | `periodStart` |
|---------|-------------------|------------|---------------|
| `AchievementGranter` (called in quiz submit service) | `1` (streak) | No — one-time per milestone | `NULL` |
| `AchievementGranter` (called in quiz submit service) | `5` (mastery_topic), `6` (mastery_program) | No — one-time per topic/program | `NULL` |

> These types have conditions that are fully deterministic at the moment of submission — no need to wait for period end.

### Scheduled — Hangfire RecurringJob

| Job | Cron | Purpose | Note |
|-----|------|---------|------|
| `StreakFreezeApplyJob` | Daily, start-of-day (00:05) | Auto-consume one freeze for users who didn't learn yesterday and have freeze count > 0 | Protects streak without user action |
| `TopStreakAchievementJob` | Daily, end-of-day (23:50) | Grant type `2` (top_streak) achievements | Needs full-day data to rank all users |
| `LeaderboardAchievementJob` | End of each `awardingInterval` period | Grant type `3` (leaderboard) achievements | Runs after period closes |
| `HighScoreAchievementJob` | End of each `awardingInterval` period | Grant type `4` (high_score) achievements | Runs after period closes |

> Type `2` requires ranking all users by streak — not knowable until end-of-day. Types `3` and `4` require the period to close before final standings are determined.

---

### `StreakFreezeApplyJob` — Design Detail

**Logic:**
1. Query `UserStreak` where: `streak > 0` AND `lastLearnedDate < today` AND `streakFreezeCount > 0`
2. For each matching user: decrement `streakFreezeCount` by 1, `INSERT INTO UserStreakFreezes (userId, frozenDate = yesterday)`
3. Composite PK `(userId, frozenDate)` on `UserStreakFreezes` enforces idempotency — duplicate INSERT is a no-op on retry

**At scale — batch processing:**

Querying and updating all users in a single transaction will cause lock contention and timeouts at large user counts. Instead:

- **Chunk by userId range:** Job queries users in pages (e.g. 500 at a time) ordered by `userId`. Each page is dispatched as a child `BackgroundJob` via `BackgroundJob.Enqueue(...)`. The parent job only enqueues — it does not process directly.

```
StreakFreezeApplyJob (RecurringJob)
  └─ Enqueues N × StreakFreezeApplyBatchJob(minUserId, maxUserId)

StreakFreezeApplyBatchJob (BackgroundJob, processes one page)
  └─ Bulk UPDATE within that userId range
```

- **Bulk UPDATE instead of row-by-row:** Within each batch, use a single `UPDATE` + `INSERT` instead of looping per user — reduces round trips significantly.
- **Idempotency key:** Store `processedDate` on the freeze record. Job checks `WHERE frozenDate != today` before processing, so safe to re-run on retry.
- **Queue isolation:** Run freeze jobs on a dedicated Hangfire queue (e.g. `"streak"`) separate from web-triggered jobs to avoid starving user-facing requests.

---

### Shared Rules

- **Idempotency:** Before inserting into `UserAchievements`, check uniqueness on `(userId, achievementId, periodStart)` to prevent duplicate grants on job retry or re-submission.
- **Active check:** Skip any achievement where `isActive = false`.
- **Credit grant:** Add `creditReward` to the user's credit balance in the same transaction as the `UserAchievements` insert.
- **`awardingInterval` on the achievement record** controls the evaluation window for scheduled jobs, not the job schedule itself — jobs run on a fixed cadence and filter which achievements to process.
