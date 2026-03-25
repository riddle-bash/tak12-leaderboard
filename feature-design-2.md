# Feature Design Document

---

## I. Leaderboard

Tính năng Leaderboard cho phép học sinh theo dõi thứ hạng của mình so với các học sinh khác trong cùng trường, nhóm, hoặc toàn hệ thống. Mục tiêu là tạo động lực học tập thông qua sự cạnh tranh lành mạnh — học sinh có thể thấy mình đang đứng ở đâu, cần bao nhiêu điểm để vươn lên thứ hạng tiếp theo, và thứ hạng của mình đang tăng hay giảm so với kỳ trước.

### Filters

| Filter | Options | Default |
|--------|---------|---------|
| Cá nhân / nhóm | `TAK12` · `Nhóm của bạn` | TAK12 |
| Thời gian | `Tuần` · `Tháng` · `Học kỳ` | Tuần |
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
  timeRange: string   // "week" | "month" | "semester" — phạm vi thời gian tính điểm
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

### API Changes

#### `UpdateUserExamRecordJob.cs`

Thay đổi signature để nhận thêm `examId` làm input — tránh việc job phải tự query qua các bảng `QuizQuestions`, `QuizCategories`, và `Exams` để tra cứu exam.

| | Before | After |
|---|--------|-------|
| **Inputs** | `questionId`, `customerId` | `questionId`, `customerId`, `examId` |

> `examId` được truyền vào từ caller (quiz submission service) tại thời điểm nộp bài — lúc này `examId` đã có sẵn trong context, không cần join thêm bảng.

---

### Leaderboard Calculation

#### Nguồn dữ liệu — `UserExamRecords`

Điểm leaderboard được tính từ bảng `UserExamRecords`. Bảng này là bản tổng hợp pre-computed theo từng (user, exam, ngày) — được `UpdateUserExamRecordJob` cập nhật mỗi khi học sinh nộp một câu hỏi. Nhờ phân đoạn sẵn theo năm/tháng/ngày, việc tính tổng điểm trong một khoảng thời gian chỉ cần `SUM` trên tập hàng đã lọc, không phải scan toàn bộ raw quiz data.

```sql
-- Schema tham khảo
SELECT [Id], [CustomerGuid], [ExamId],
       [Year], [Month], [Day],
       [CorrectCount], [WrongCount],
       [UpdateTime], [DataDate]
FROM [QuizSystem].[dbo].[UserExamRecords]
```

> **Điểm của một hàng** = `CorrectCount` (hoặc công thức tính điểm cụ thể theo business rule — cần xác nhận). Leaderboard dùng tổng `CorrectCount` trong khoảng `timeRange`.

---

#### Global Leaderboard

Tính tổng điểm của **tất cả user** trên **tất cả exam**, lọc theo `timeRange`. Không filter theo `examId`.

```sql
-- Xác định khoảng ngày dựa vào timeRange
-- "week"     → DataDate >= DATEADD(day, -7,   CAST(GETUTCDATE() AS DATE))
-- "month"    → DataDate >= DATEADD(day, -30,  CAST(GETUTCDATE() AS DATE))
-- "semester" → DataDate >= đầu học kỳ hiện tại (ngày bắt đầu học kỳ — cần xác định ranh giới học kỳ theo lịch nhà trường)

SELECT
    uer.CustomerGuid                    AS userId,
    SUM(uer.CorrectCount)               AS totalPoint
FROM UserExamRecords uer
WHERE
    -- timeRange = "week"
    uer.DataDate >= DATEADD(day, -7, CAST(GETUTCDATE() AS DATE))
    -- (thay điều kiện ngày tương ứng với timeRange)
GROUP BY
    uer.CustomerGuid
ORDER BY
    totalPoint DESC
```

Rank, `rankShifted`, và `pointToRankUp` được tính ở application layer sau khi có danh sách đã sắp xếp.

Nếu cần thêm filter **Tỉnh / Trường**, join thêm bảng `Users` (hoặc bảng profile tương ứng) để lọc `schoolId` / `provinceId`:

```sql
SELECT
    uer.CustomerGuid                    AS userId,
    SUM(uer.CorrectCount)               AS totalPoint
FROM UserExamRecords uer
INNER JOIN Users u ON u.CustomerGuid = uer.CustomerGuid
WHERE
    uer.DataDate >= DATEADD(day, -7, CAST(GETUTCDATE() AS DATE))
    AND u.SchoolId = @schoolId          -- filter tỉnh/trường (tuỳ chọn)
GROUP BY
    uer.CustomerGuid
ORDER BY
    totalPoint DESC
```

---

#### Group Leaderboard

Chỉ tính điểm cho các **thành viên đang active trong nhóm** (`GroupUsers.isDeleted = 0`). Hỗ trợ hai chế độ:

**Tất cả exam (không filter examId):**

```sql
SELECT
    uer.CustomerGuid                    AS userId,
    SUM(uer.CorrectCount)               AS totalPoint
FROM UserExamRecords uer
INNER JOIN GroupUsers gu
    ON  gu.UserId    = uer.CustomerGuid
    AND gu.GroupId   = @groupId
    AND gu.isDeleted = 0                -- chỉ thành viên đang active
WHERE
    uer.DataDate >= DATEADD(day, -7, CAST(GETUTCDATE() AS DATE))
    -- (thay điều kiện ngày tương ứng với timeRange)
GROUP BY
    uer.CustomerGuid
ORDER BY
    totalPoint DESC
```

**Lọc theo exam cụ thể (`examId` được truyền):**

```sql
SELECT
    uer.CustomerGuid                    AS userId,
    SUM(uer.CorrectCount)               AS totalPoint
FROM UserExamRecords uer
INNER JOIN GroupUsers gu
    ON  gu.UserId    = uer.CustomerGuid
    AND gu.GroupId   = @groupId
    AND gu.isDeleted = 0
WHERE
    uer.ExamId   = @examId              -- chỉ tính điểm bài thi này
    AND uer.DataDate >= DATEADD(day, -7, CAST(GETUTCDATE() AS DATE))
GROUP BY
    uer.CustomerGuid
ORDER BY
    totalPoint DESC
```

> **Lưu ý:** Group leaderboard sử dụng cùng logic rank/rankShifted/pointToRankUp với global — chỉ khác tập user được tính.

---

### Tradeoff — Hiệu năng tính điểm Leaderboard

#### Vấn đề

Mỗi lần load leaderboard hiện tại phải chạy `GROUP BY + SUM` toàn bộ `UserExamRecords` trong `timeRange`. Với 500k user, mỗi user làm 10 câu/ngày → ~3.5M hàng trong window 7 ngày. Query này chạy trên mỗi request của mọi học sinh trên cùng lúc — đây là bottleneck rõ ràng trên read path.

#### So sánh các hướng tiếp cận

| Hướng | Read | Write | Độ tươi | Độ phức tạp |
|-------|------|-------|---------|-------------|
| **A — Live query** (hiện tại) | Nặng — full aggregation mỗi request | Không có | Real-time | Thấp |
| **B — Pre-computed cache** (khuyến nghị) | Nhẹ — `SELECT` đơn giản | Trung bình — job refresh định kỳ | Trễ tối đa N phút | Trung bình |
| **C — Incremental write-time update** | Rất nhẹ | Nặng — mỗi submission cập nhật cache | Gần real-time | Cao |

> Hướng C có vấn đề với **rolling window**: khi một ngày "rơi ra" khỏi cửa sổ (ví dụ: ngày thứ 8 trong window 7 ngày), cần trừ điểm của ngày đó khỏi tổng — phức tạp và dễ sai. Không khuyến nghị.

#### Giải pháp — Hướng B: Pre-computed cache với job refresh định kỳ

Thêm bảng `UserLeaderboardScores` lưu tổng điểm đã tính sẵn theo từng `(userId, timeRange)`. Job `LeaderboardRefreshJob` chạy mỗi 5 phút, re-compute toàn bộ và UPSERT vào bảng này. Leaderboard read chỉ cần `SELECT ... ORDER BY score DESC` — không aggregation.

##### Bảng `UserLeaderboardScores`

| Column | Type | Constraints | Ghi chú |
|--------|------|-------------|----------|
| `id` | `INT` | `PK`, `IDENTITY` | |
| `userId` | `INT` | `FK → Users(id)`, `NOT NULL` | |
| `timeRange` | `NVARCHAR(10)` | `NOT NULL` | `"week"` \| `"month"` \| `"semester"` |
| `score` | `FLOAT` | `NOT NULL` | Tổng `CorrectCount` trong `timeRange` |
| `updatedAt` | `DATETIME2` | `NOT NULL`, `DEFAULT SYSUTCDATETIME()` | Timestamp của lần refresh gần nhất |

> **Unique constraint:** `(userId, timeRange)` — mỗi user chỉ có 1 hàng mỗi loại `timeRange`.
> **Index gợi ý:** `IX_UserLeaderboardScores_TimeRange_Score` trên `(timeRange, score DESC)` để sort nhanh.

##### Job — `LeaderboardRefreshJob`

| Property | Value |
|----------|---------|
| **Cron** | Mỗi 5 phút (có thể cấu hình qua `applicationSetting`: `leaderboardRefreshIntervalMinutes`) |
| **Logic** | MERGE (upsert) điểm tất cả user cho từng `timeRange` vào `UserLeaderboardScores` |
| **Queue** | Dedicated Hangfire queue `"leaderboard"` — tránh block web requests |

```sql
-- Ví dụ refresh cho timeRange = "week"
MERGE UserLeaderboardScores AS target
USING (
    SELECT
        uer.CustomerGuid                AS userId,
        SUM(uer.CorrectCount)           AS score
    FROM UserExamRecords uer
    WHERE uer.DataDate >= DATEADD(day, -7, CAST(GETUTCDATE() AS DATE))
    GROUP BY uer.CustomerGuid
) AS source
ON  target.userId    = source.userId
AND target.timeRange = 'week'
WHEN MATCHED THEN
    UPDATE SET
        target.score     = source.score,
        target.updatedAt = SYSUTCDATETIME()
WHEN NOT MATCHED THEN
    INSERT (userId, timeRange, score, updatedAt)
    VALUES (source.userId, 'week', source.score, SYSUTCDATETIME());
```

##### Leaderboard read query (sau khi có cache)

```sql
SELECT
    ls.userId,
    ls.score                            AS totalPoint
FROM UserLeaderboardScores ls
INNER JOIN Users u ON u.Id = ls.userId
WHERE
    ls.timeRange = @timeRange
    AND u.SchoolId = @schoolId          -- filter tỉnh/trường (tuỳ chọn)
ORDER BY
    ls.score DESC
```

> **Chấp nhận được độ trễ:** Leaderboard gamification không yêu cầu real-time tuyệt đối — trễ 5 phút là chấp nhận được và giảm đáng kể load database trên hot read path.

---

### RankShifted — Thiết kế

#### Vấn đề

Leaderboard cập nhật liên tục theo từng lần nộp câu hỏi — thứ hạng có thể thay đổi nhiều lần trong một ngày. Nếu tính `rankShifted` so với thời điểm hiện tại thì không có ý nghĩa. Cần một mốc tham chiếu cố định và rõ ràng.

#### Cơ chế — Mốc tham chiếu cố định

`rankShifted` luôn so sánh thứ hạng hiện tại với thứ hạng vào **cuối kỳ trước đó** (snapshot). Mốc này giữ nguyên trong suốt kỳ đang chạy — mọi thay đổi trong tuần/tháng đều đo từ cùng một điểm tham chiếu, không thay đổi theo ngày hay giờ.

**Ví dụ — `timeRange = "week"`:**

| Thời điểm | Peter A | Harry C | John B |
|-----------|---------|---------|--------|
| Cuối tuần trước *(snapshot — mốc tham chiếu)* | 100đ · **1st** | 79đ · **3rd** | 81đ · **2nd** |
| Thứ Tư tuần này | 102đ · 1st · `0` | 95đ · 2nd · `+1` | 93đ · 3rd · `−1` |
| Thứ Năm tuần này | 103đ · 3rd · `−2` | 105đ · 1st · `+2` | 104đ · 2nd · `0` |

> **Công thức:** `rankShifted = snapshotRank − currentRank`
> Dương = tăng hạng (Harry: snapshot 3rd → hiện tại 1st → `3 − 1 = +2`).
> Âm = giảm hạng (Peter: snapshot 1st → hiện tại 3rd → `1 − 3 = −2`).
> Snapshot chỉ được cập nhật khi kỳ kết thúc bởi `LeaderboardSnapshotJob` — không cập nhật giữa kỳ.

#### Hai hướng tiếp cận

**Hướng 1 — Rolling window (không cần lưu thêm dữ liệu)**

Tính `rank` cho khoảng thời gian hiện tại và khoảng thời gian liền trước bằng cách thay đổi điều kiện ngày trên `UserExamRecords`:

| `timeRange` | Kỳ hiện tại | Kỳ trước |
|-------------|-------------|----------|
| `week` | DataDate trong 7 ngày gần nhất | DataDate từ ngày 8–14 trước |
| `month` | DataDate trong 30 ngày gần nhất | DataDate từ ngày 31–60 trước |
| `semester` | DataDate từ đầu học kỳ hiện tại | DataDate từ đầu học kỳ trước |

Ưu điểm: không cần bảng thêm, không cần job. Nhược điểm: **chi phí gấp đôi** — mỗi lần load leaderboard phải chạy 2 lần aggregation toàn bộ user. Với 500k user, đây là bottleneck nghiêm trọng.

**Hướng 2 — Snapshot table (khuyến nghị)**

Một job chạy định kỳ vào cuối mỗi kỳ, lưu snapshot thứ hạng của mỗi user vào bảng `UserRankSnapshots`. Khi load leaderboard, `rankShifted` = `snapshotRank - currentRank` — chỉ cần một lần đọc bảng snapshot, không cần tính lại.

#### Giải pháp — Snapshot table

##### Bảng `UserRankSnapshots`

| Column | Type | Constraints | Ghi chú |
|--------|------|-------------|---------|
| `id` | `INT` | `PK`, `IDENTITY` | |
| `userId` | `INT` | `FK → Users(id)`, `NOT NULL` | |
| `timeRange` | `NVARCHAR(10)` | `NOT NULL` | `"week"` hoặc `"month"` hoặc `"semester"` |
| `periodEnd` | `DATE` | `NOT NULL` | Ngày cuối kỳ vừa kết thúc (ví dụ: Chủ nhật cuối tuần) |
| `rank` | `INT` | `NOT NULL` | Thứ hạng tại thời điểm kết thúc kỳ |

> **Index gợi ý:** `IX_UserRankSnapshots_UserId_TimeRange_PeriodEnd` trên `(userId, timeRange, periodEnd DESC)` để lookup nhanh snapshot gần nhất.
> **Retention:** Chỉ cần giữ 1–2 snapshot gần nhất mỗi loại `timeRange` mỗi user — các snapshot cũ hơn có thể xóa định kỳ.

##### Job — `LeaderboardSnapshotJob`

| Property | Value |
|----------|-------|
| **Cron** | Cuối tuần (ví dụ: Chủ nhật 23:55) cho `week`; ngày cuối tháng 23:55 cho `month`; ngày cuối học kỳ 23:55 cho `semester` |
| **Logic** | Chạy query tổng hợp điểm toàn bộ user cho kỳ vừa kết thúc, tính rank theo `DENSE_RANK()`, INSERT hàng loạt vào `UserRankSnapshots` |
| **Idempotency** | Unique constraint trên `(userId, timeRange, periodEnd)` — chạy lại an toàn |

```sql
-- Ví dụ snapshot cuối tuần
INSERT INTO UserRankSnapshots (userId, timeRange, periodEnd, rank)
SELECT
    uer.CustomerGuid,
    'week',
    CAST(GETUTCDATE() AS DATE),   -- ngày snapshot
    DENSE_RANK() OVER (ORDER BY SUM(uer.CorrectCount) DESC)
FROM UserExamRecords uer
WHERE uer.DataDate >= DATEADD(day, -7, CAST(GETUTCDATE() AS DATE))
GROUP BY uer.CustomerGuid
```

##### Cách tính `rankShifted` khi load leaderboard

```sql
-- Lấy snapshot gần nhất cho mỗi user theo timeRange
WITH LatestSnapshot AS (
    SELECT userId, rank,
           ROW_NUMBER() OVER (PARTITION BY userId ORDER BY periodEnd DESC) AS rn
    FROM UserRankSnapshots
    WHERE timeRange = @timeRange
)
SELECT
    current.userId,
    current.totalPoint,
    DENSE_RANK() OVER (ORDER BY current.totalPoint DESC)   AS currentRank,
    ISNULL(snap.rank, 0)                                   AS previousRank,
    ISNULL(snap.rank, 0)
        - DENSE_RANK() OVER (ORDER BY current.totalPoint DESC) AS rankShifted
    -- dương = tăng hạng, âm = giảm hạng, 0 = chưa có snapshot trước
FROM (... -- current period aggregation) current
LEFT JOIN LatestSnapshot snap
    ON snap.userId = current.userId AND snap.rn = 1
ORDER BY currentRank
```

---

### Review

- ✅ `userId` không nên truyền từ client — server tự lấy từ session (`AbpSession.GetUserId()` hoặc `CthSession.UserGuid` tùy API).
- **Phân trang?** Không cần — leaderboard chỉ hiển thị khoảng 10–14 user trên UI.
- ✅ Filter theo Tỉnh đã có trong thiết kế.
- ✅ `rankShifted` — xem thiết kế `UserRankSnapshots` + `LeaderboardSnapshotJob` ở trên.
- **Semester boundary:** Ngày bắt đầu/kết thúc học kỳ cần được cấu hình theo lịch nhà trường (không phải rolling window cố định). Cần xác định đầu vào cấu hình (applicationSetting hoặc enum học kỳ) trước khi implement.
- **Nguồn dữ liệu `point`:** Tổng hợp từ `UserExamRecords.CorrectCount` trong `timeRange` — không query raw `QuizAttemptQuestions` trực tiếp.

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
  timeRange: string   // "week" | "month" | "semester"
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
| `isDeleted` | `BIT` | `NOT NULL`, `DEFAULT 0` | Soft delete — nhóm bị giải tán không xoá vật lý, giữ lại để leaderboard có thể tra cứu dữ liệu lịch sử liên quan |

#### Table: `GroupUsers`

| Column | Type | Constraints | Ghi chú |
|--------|------|-------------|---------|
| `userId` | `INT` | `FK → Users(id)`, `NOT NULL` | |
| `groupId` | `INT` | `FK → Groups(id)`, `NOT NULL` | |
| `creationTime` | `DATETIME2` | `NOT NULL`, `DEFAULT SYSUTCDATETIME()` | |
| `isDeleted` | `BIT` | `NOT NULL`, `DEFAULT 0` | Soft delete — giữ lại lịch sử membership để leaderboard có thể tra cứu điểm đóng góp của thành viên cũ |

> **Primary key:** `PK_GroupUsers` composite trên `(userId, groupId)`.
> **Lưu ý query:** Các truy vấn membership hiện tại phải lọc thêm `WHERE isDeleted = 0`. Leaderboard/analytics có thể đọc toàn bộ (kể cả `isDeleted = 1`) để tính điểm lịch sử.

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
  type:         1 | 2 | 3 | 4 | 5 | 6 | 7
  // 1 = streak          — Bạn đã học liên tiếp {X} ngày không gián đoạn
  // 2 = top_streak      — Chuỗi học tập thuộc top {X}% streak cao nhất
  // 3 = leaderboard     — Lọt Top {N} bảng xếp hạng {kỳ}
  // 4 = high_score      — Đạt điểm số rất cao trong nhiều bài luyện tập
  // 5 = mastery_topic   — Nắm vững chủ điểm
  // 6 = mastery_program — Nắm vững chương trình {tên chương trình}
  // 7 = manual          — Danh hiệu trao thủ công
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
  examId?:       number   // FK → Exams — bài thi cụ thể đã trigger danh hiệu (chỉ có với type 3/4)
                          // NULL với các type không liên quan đến exam (1, 2, 5, 6, 7)
  receivedDate:  string   // Timestamp UTC khi danh hiệu được trao
  note:          string   // Lý do khi admin trao thủ công (type = 7), để trống nếu trao tự động
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
| `type` | `TINYINT` | `NOT NULL` | `1`=streak · `2`=top_streak · `3`=leaderboard · `4`=high_score · `5`=mastery_topic · `6`=mastery_program · `7`=manual |
| `isActive` | `BIT` | `NOT NULL`, `DEFAULT 1` | |
| `isDeleted` | `BIT` | `NOT NULL`, `DEFAULT 0` | Soft delete — ẩn danh hiệu khỏi hệ thống mà không xoá vật lý, giữ nguyên lịch sử `UserAchievements` |
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
| `examId` | `INT` | `FK → Exams(id)`, `NULL` | Bài thi cụ thể đã trigger danh hiệu — `NULL` với type không liên quan đến exam (1, 2, 5, 6, 7) |
| `note` | `NVARCHAR(500)` | `NULL` | Ghi chú khi admin trao thủ công (`type = 7`) |

> **Index gợi ý:** `IX_UserAchievements_UserId` trên `userId` để query nhanh tủ danh hiệu của một user.
> **Traceability:** Với type 3/4, `(userId, achievementId, examId, periodStart)` xác định duy nhất một lần trao — dùng cho idempotency check và audit trail.

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

Achievement granting is split into two execution models based on the nature of the achievement:

- **Self-awarded (event-driven):** Condition is fully deterministic for a single user at the moment they submit a quiz/topic/program — no need to compare against others or wait for a period to close. Evaluated inline in the submission service.
- **Competitive (scheduled):** Requires ranking across all users, or the full period to close before final standings can be determined. Registered as `RecurringJob` with a cron expression.

---

### Self-awarded — Event-driven (inline after submission)

Triggered directly inside the quiz/topic/program submission service. No Hangfire job is enqueued — `AchievementGranter` is called synchronously before the response is returned.

| Type | Trigger point | Repeatable | `periodStart` |
|------|---------------|------------|---------------|
| `1` (streak) | After quiz submitted — streak counter updated | No — one-time per milestone | `NULL` |
| `5` (mastery_topic) | After topic marked complete | No — one-time per topic | `NULL` |
| `6` (mastery_program) | After program marked complete | No — one-time per program | `NULL` |
| `7` (manual) | Admin calls `grantAchievementToUser` directly | Admin-controlled | `NULL` |

> These types check conditions that belong entirely to the submitting user — streak length, topic completion rate, program completion rate. No cross-user comparison needed.

---

### Competitive — Scheduled (Hangfire RecurringJob)

These types require data from all users or the full period to close before a winner can be determined. Each job is registered as a `RecurringJob`.

| Job | Cron | Achievement Type | Trigger condition |
|-----|------|-----------------|-------------------|
| `StreakFreezeApplyJob` | Daily 00:05 | — (streak protection, not achievement) | User didn't learn yesterday AND `streakFreezeCount > 0` |
| `LeaderboardRefreshJob` | Every 5 min (configurable) | — (leaderboard infrastructure) | Re-computes `SUM(CorrectCount)` per user per `timeRange`, upserts into `UserLeaderboardScores` cache table |
| `LeaderboardSnapshotJob` | End of each period | — (leaderboard infrastructure) | Week ends (e.g. Sun 23:58) → snapshot `"week"` rank; Month ends → snapshot `"month"` rank; Semester ends → snapshot `"semester"` rank |
| `TopStreakAchievementJob` | Daily 23:50 | `2` (top_streak) | Full day of data needed to rank all users by streak |
| `LeaderboardAchievementJob` | End of each `awardingInterval` | `3` (leaderboard) | Period closes — final rank determined from leaderboard standings |
| `HighScoreAchievementJob` | End of each `awardingInterval` | `4` (high_score) | Period closes — `requiredCompletionCount` checked across the closed period |

> `awardingInterval` on the achievement record controls the evaluation window. Jobs run on a fixed cadence and filter which achievements to process based on `awardingInterval`.

---

### Job Activation — Achievement-driven

Jobs and inline granters are not unconditionally active. Their execution is gated on whether at least one active, non-deleted achievement of the corresponding type exists. This prevents wasted compute when no achievement has been configured for a type yet.

#### Scheduled jobs — early-exit guard

Each `RecurringJob` begins with a guard query. If no qualifying achievement exists, the job exits immediately without doing any work:

```csharp
// Example: TopStreakAchievementJob
var hasActive = await _achievementRepo.AnyAsync(a =>
    a.Type == AchievementType.TopStreak &&
    a.IsActive && !a.IsDeleted);

if (!hasActive) return; // nothing to evaluate
```

This means **all scheduled jobs are always registered** in Hangfire — no need to add/remove `RecurringJob` entries at runtime. The guard makes them no-ops when there is nothing to process.

#### Event-driven granters — inline guard

`AchievementGranter` (called inline in quiz/topic/program submission) applies the same guard per type before evaluating conditions:

```csharp
// Called after quiz submitted — checks type 1 (streak)
var streakAchievements = await _achievementRepo.GetActiveByTypeAsync(
    AchievementType.Streak); // filters IsActive && !IsDeleted

if (!streakAchievements.Any()) return;
// proceed to check streak milestones for this user
```

#### Admin create / activate flow

When admin creates or re-activates an achievement:

1. Row is inserted / `isActive` set to `true`, `isDeleted` set to `false`.
2. No additional job registration is needed — the guard in the corresponding job/granter will now pass on its next execution.
3. For **scheduled types** (2, 3, 4): the `RecurringJob` is already registered; it will pick up the new achievement on its next cron tick.
4. For **event-driven types** (1, 5, 6): the inline granter is already wired into the submission service; it will evaluate the new achievement on the next quiz/topic/program submission.

#### Admin deactivate / delete flow

When admin sets `isActive = false` or `isDeleted = true` on the last achievement of a type:

1. The guard query will return `false` on the next job run or granter call.
2. The job/granter exits early — effectively disabled without removing the `RecurringJob` from Hangfire.
3. No manual Hangfire job removal is required.

> **Summary:** Jobs are always registered; achievements act as the on/off switch. One active achievement of a type = job runs. Zero active achievements of a type = job is a no-op.

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

- **Idempotency:** Before inserting into `UserAchievements`, check uniqueness on `(userId, achievementId, examId, periodStart)` to prevent duplicate grants on job retry or re-submission. `examId` is `NULL` for non-exam types — use `IS NULL` in the uniqueness check accordingly.
- **Active check:** Skip any achievement where `isActive = false`.
- **Credit grant:** Add `creditReward` to the user's credit balance in the same transaction as the `UserAchievements` insert.
- **`awardingInterval` on the achievement record** controls the evaluation window for scheduled jobs, not the job schedule itself — jobs run on a fixed cadence and filter which achievements to process.
