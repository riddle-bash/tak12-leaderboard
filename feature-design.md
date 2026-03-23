# Feature Design Document

---

## I. Leaderboard

### Filters

| Filter | Options | Default |
|--------|---------|---------|
| Cá nhân / nhóm | TAK12 · Nhóm của bạn | TAK12 |
| Thời gian | Tuần · Tháng · Tất cả | Tuần |
| Phạm vi | Tỉnh · Trường · Nhóm | — |

> **Lưu ý:** Filter được lưu theo lựa chọn cuối cùng của người dùng.

---

### API

#### Request

```ts
{
  userId:    number
  schoolId?: number
  groupId?:  number
  timeRange: string
}
```

#### Response — `Student[]`

```ts
Student {
  id:            number
  email:         string
  point:         float
  schoolId:      number
  schoolName:    string
  rank:          number
  rankShifted:   number
  pointToRankUp: number
}
```

---

## II. Group

### FE — Danh sách nhóm

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
  memberCount:      number
}
```

---

### FE — CRUD nhóm

#### `GET` — Danh sách
```ts
Request:  { userId, groups: [] }
```

#### `GET ?id` — Chi tiết
```ts
Request:  { userId, groupId, groupName, groupDescription, imageUrl, group_members[] }
```

#### `POST` — Tạo mới
```ts
Request:  { userId, groupName, memberIds, imageBase64 }
```

#### `PUT` — Cập nhật
```ts
Request:  { userId, groupId, groupName, memberIds, imageBase64 }
```

#### `DELETE`
```ts
Request:  { userId, groupId }
```

#### Schema — `group_members`
```ts
{
  userId:       number
  emailAddress: string
  imageUrl:     string
  schoolName:   string
  schoolId:     number
  joinedDate:   string
}
```

---

### FE — Chi tiết nhóm

#### `GET ?id&countTotalOnly=true` — Overview / lời mời
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

#### `GET getDetailedGroupMembers?groupId=`

**Request**
```ts
{ timeRange, examId, groupId, page }
```

**Response — `students[]`**
```ts
{
  userId:       number
  emailAddress: string
  points:       number
  rank:         number
  rankShifted:  number
  streakCount:  number
  joinedDate:   string
}
```

---

### Admin — Quản lý nhóm

Tái sử dụng các API FE. Admin có thêm:

- Xem danh sách **tất cả** nhóm
- Tìm kiếm theo `groupId`
- Tìm kiếm theo `userId` / `email` của người tạo

---

## III. Streak

### FE — Dashboard

#### `getUserStreakInfo`

```ts
Response: {
  streak:                   number
  maxStreak:                number
  topStreak:                number
  maxLearningTimeInMinutes: number
  topLearningTime:          number
  totalPoints:              number
  topTotalPoints:           number
  streakDays:               string[]
  frozenStreakDays:          string[]
  nextStreakAchievement: {
    id:        number
    name:      string
    imagePath: string
    goal:      number
  }
}
```

#### `getUserStreakFreezeInfo`

```ts
Response: {
  streakFreezeCount:     number
  creditBalance:         number
  creditPerStreakFreeze: number
}
```

#### `redeemStreakFreeze`

```ts
Request:  { userId: number }
Response: { remainCredit: number, streakFreezeCount: number }
```

---

### Admin — Cài đặt nhắc nhở

Bổ sung `applicationSettingKey`:

```ts
streakReminderOffsetHours: number
```

> Số giờ trước khi kết thúc ngày học — TAK12 sẽ gửi nhắc nhở để học sinh không bị mất chuỗi streak.

---

## IV. Achievement

### Data Models

#### `Achievement`

```ts
{
  name:                     string
  description:              string
  type:                     1 | 2 | 3 | 4  // 1=streak · 2=leaderboard · 3=completion · 4=manual
  isActive:                 boolean
  imagePath:                string
  creditReward:             number
  tierId:                   number
  requiredStreakDays?:       number
  requiredLeaderboardRank?: number
}
```

#### `AchievementTier`

```ts
{ id: number, name: string }
```

#### `AchievementExams`

```ts
{
  examId:                       number
  achievementId:                number
  requiredCompletionPercentage: number
  awardingInterval:             number
  requiredCompletionCount:      number
  countType:                    number
}
```

#### `UserAchievements`

```ts
{
  userId:        number
  achievementId: number
  receivedDate:  string
  note:          string
}
```

---

### FE — Tủ danh hiệu

#### `getUserAchievements`

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

> Người dùng có thể trưng bày tối đa **3 danh hiệu** trên profile.  
> Lưu vào `[UserSettings].[ShowcaseAchievementIds]`.

---

### Admin — Quản lý danh hiệu

| Method | Endpoint |
|--------|----------|
| `GET` | `getAllAchievements` |
| `POST` | `achievement` |
| `PUT` | `achievement` |
| `DELETE` | `achievement` |

#### `grantAchievementToUser`

```ts
Request: {
  achievementId: number
  userId:        number
  note:          string
}
```

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
| `requiredStreakDays` | `INT` | `NULL` | Chỉ dùng khi `type = 1` |
| `requiredLeaderboardRank` | `INT` | `NULL` | Chỉ dùng khi `type = 2` |

> **Index gợi ý:** `IX_Achievement_Type` trên cột `type` để lọc nhanh theo loại.

---

### Table: `AchievementExams`

Liên kết giữa danh hiệu loại `completion` (`type = 3`) và các bài thi cụ thể.

| Column | Type | Constraints | Ghi chú |
|--------|------|-------------|---------|
| `id` | `INT` | `PK`, `IDENTITY` | |
| `achievementId` | `INT` | `FK → Achievement(id)`, `NOT NULL` | |
| `examId` | `INT` | `NOT NULL` | Tham chiếu tới bảng Exam |
| `requiredCompletionPercentage` | `DECIMAL(5,2)` | `NOT NULL` | Tỉ lệ % hoàn thành tối thiểu (0–100) |
| `requiredCompletionCount` | `INT` | `NOT NULL`, `DEFAULT 1` | Số lần phải hoàn thành |
| `awardingInterval` | `INT` | `NULL` | Khoảng thời gian (ngày) giữa các lần trao |
| `countType` | `NVARCHAR(50)` | `NOT NULL` | Cách đếm lần hoàn thành, vd: `"all_time"`, `"monthly"` |

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
