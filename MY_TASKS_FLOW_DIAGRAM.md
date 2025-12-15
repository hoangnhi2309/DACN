# Sơ đồ luồng My Tasks - Tóm tắt

## 📊 Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    USER MỞ TRANG MY TASKS                   │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  FRONTEND: AllTaskComponent.ngOnInit()                      │
│  - Gọi loadTasks()                                          │
│  - Setup realtime subscription                              │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  FRONTEND: loadTasks()                                       │
│  1. Lấy currentUserId từ Supabase Auth                     │
│  2. Build auth headers (JWT token)                         │
│  3. Gọi API: GET /boards/tasks/my                           │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  BACKEND: BoardsController.getMyTasks()                      │
│  - Extract userId từ JWT token                              │
│  - Gọi boardsService.getMyAssignedTasks(userId)             │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  BACKEND: BoardsService.getMyAssignedTasks()                │
│                                                              │
│  BƯỚC 1: Lấy boards user có quyền                           │
│  ├─ Query: SELECT * FROM boards WHERE owner_id = userId     │
│  └─ Query: SELECT * FROM board_members WHERE user_id = ... │
│                                                              │
│  BƯỚC 2: Query tasks được ASSIGN                            │
│  └─ SELECT * FROM tasks                                      │
│     WHERE project_id IN (board_ids)                          │
│     AND assignee = userId                                    │
│                                                              │
│  BƯỚC 3: Query tasks được TẠO                              │
│  └─ SELECT * FROM tasks                                      │
│     WHERE project_id IN (board_ids)                         │
│     AND created_by = userId                                  │
│                                                              │
│  BƯỚC 4: Merge và loại bỏ duplicate                         │
│  └─ taskMap.set() để tránh duplicate                        │
│                                                              │
│  BƯỚC 5: Lấy thông tin bổ sung                              │
│  ├─ Query profiles (creator info)                            │
│  ├─ Query columns (list names)                               │
│  └─ Query boards (board names)                               │
│                                                              │
│  BƯỚC 6: Format response                                    │
│  └─ Return array of TaskWithBoard                            │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  FRONTEND: Nhận response                                     │
│  - tasks$ Observable emit data                               │
│  - Template hiển thị tasks                                   │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  REALTIME SUBSCRIPTION (Chạy song song)                     │
│                                                              │
│  Channel 1: tasks-assigned-changes                           │
│  ├─ Filter: assignee = userId                                │
│  ├─ Event: UPDATE → reload tasks                             │
│  └─ Event: INSERT → reload tasks                             │
│                                                              │
│  Channel 2: tasks-created-changes                            │
│  ├─ Filter: created_by = userId                              │
│  ├─ Event: UPDATE → reload tasks                             │
│  └─ Event: INSERT → reload tasks                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔄 Flow khi ASSIGN TASK

```
┌─────────────────────────────────────────────────────────────┐
│  USER A assign task cho USER B                              │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  FRONTEND: BoardComponent.save()                             │
│  - Gọi API: PATCH /boards/cards/:id                          │
│  - Body: { assignee: "user-b-id" }                          │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  BACKEND: BoardsService.updateCard()                         │
│  1. Lấy oldAssignee từ database                             │
│  2. Update task.assignee = newAssignee                       │
│  3. So sánh: oldAssignee !== newAssignee?                    │
│  4. Nếu có → gọi createTaskAssignmentNotification()         │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  BACKEND: createTaskAssignmentNotification()                 │
│  1. Lấy thông tin assigner (User A)                          │
│  2. Lấy tên board                                            │
│  3. Tạo notification record:                                 │
│     {                                                         │
│       user_id: "user-b-id",  // User B nhận                  │
│       type: "task_assigned",                                  │
│       content: "{ message: '...', ... }",                    │
│       task_id: "...",                                         │
│       sender_id: "user-a-id"  // User A gửi                  │
│     }                                                         │
│  4. INSERT INTO notifications                                │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  SUPABASE REALTIME TRIGGER                                   │
│                                                              │
│  Event 1: tasks table UPDATE                                 │
│  ├─ Filter: assignee = user-b-id                             │
│  └─ → User B nhận realtime update                            │
│                                                              │
│  Event 2: notifications table INSERT                         │
│  ├─ Filter: user_id = user-b-id                              │
│  └─ → User B nhận notification                               │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  USER B FRONTEND                                             │
│                                                              │
│  Realtime Handler 1:                                         │
│  ├─ Nhận tasks UPDATE event                                  │
│  └─ → loadTasks() → Task hiển thị trong My Tasks            │
│                                                              │
│  Realtime Handler 2:                                         │
│  ├─ Nhận notifications INSERT event                          │
│  └─ → reloadInvitations() → Hiển thị notification           │
└─────────────────────────────────────────────────────────────┘
```

---

## 📋 Database Queries Chi Tiết

### Query 1: Lấy boards user có quyền

```sql
-- Boards user là owner
SELECT id, name, visibility, owner_id 
FROM boards 
WHERE owner_id = 'user-id';

-- Boards user là member
SELECT bm.board_id, b.id, b.name, b.visibility, b.owner_id
FROM board_members bm
JOIN boards b ON b.id = bm.board_id
WHERE bm.user_id = 'user-id';
```

### Query 2: Lấy tasks được ASSIGN

```sql
SELECT 
  id, column_id, title, status, description, 
  position, assignee, project_id, created_by
FROM tasks
WHERE project_id IN ('board-id-1', 'board-id-2', ...)
  AND assignee = 'user-id'
  AND assignee IS NOT NULL
  AND assignee != '';
```

### Query 3: Lấy tasks được TẠO

```sql
SELECT 
  id, column_id, title, status, description, 
  position, assignee, project_id, created_by
FROM tasks
WHERE project_id IN ('board-id-1', 'board-id-2', ...)
  AND created_by = 'user-id'
  AND created_by IS NOT NULL;
```

### Query 4: Lấy thông tin creator

```sql
SELECT id, email, full_name, avatar_url
FROM profiles
WHERE id IN ('creator-id-1', 'creator-id-2', ...);
```

### Query 5: Lấy tên columns

```sql
SELECT id, title
FROM columns
WHERE id IN ('column-id-1', 'column-id-2', ...);
```

---

## 🔑 Key Components

### Frontend Files:
- `quanli/src/app/pages/all-task/all-task.ts` - Component chính
- `quanli/src/app/pages/all-task/all-task.html` - Template
- `quanli/src/app/pages/all-task/all-task.css` - Styles

### Backend Files:
- `task-backend/src/boards/boards.controller.ts` - API endpoint
- `task-backend/src/boards/boards.service.ts` - Business logic

### Supabase:
- Bảng `tasks` - Lưu tasks với `assignee` và `created_by`
- Bảng `boards` - Lưu boards
- Bảng `board_members` - Lưu members của boards
- Bảng `notifications` - Lưu notifications
- Realtime - Trigger events khi có thay đổi

---

## ✅ Checklist Implementation

- [x] Frontend: Component load tasks từ API
- [x] Frontend: Realtime subscription cho assigned tasks
- [x] Frontend: Realtime subscription cho created tasks
- [x] Backend: Endpoint `/boards/tasks/my`
- [x] Backend: Query tasks assigned cho user
- [x] Backend: Query tasks created bởi user
- [x] Backend: Merge và loại bỏ duplicate
- [x] Backend: Format response với đầy đủ thông tin
- [x] Supabase: Schema với `assignee` và `created_by`
- [x] Supabase: Realtime enabled cho bảng `tasks`
- [x] Notification: Tạo notification khi assign task
- [x] Notification: Realtime subscription cho notifications

