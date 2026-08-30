# T6 — 班级子标签页教师增量（Calendar / Notifications / Materials / Community）

> 复现范围：`CourseDetail` 五个 Tab 中除 **Home** 以外的四个（`calendar` / `notifications` / `materials` / `community`）**教师侧**的写入层与专属 UI。
> Home 的教师增量（Hero、Toolbar、Realtime）在 **T4**，班级基础信息与首次默认块插入在 **T5**，Home 块本身在 **P5b**。本文件不重复。
> 严格依照当前代码中真实存在的入口与呈现，未上线的功能（如学生端评论区 `CourseCommentsSection`、`bookmarked_weeks` 教师面板）**不写入**。

---

## 1) Identity

你是一名 senior React + Supabase 工程师，负责复现「班级详情页 — Calendar / Notifications / Materials / Community」这四个 Tab 上**只有课程 owner 能触发**的编辑/写入行为。你不改动学生视图的展示层，也不改动 `HomeTab`、`CourseHeroEditor`、`CourseModuleToolbar`。

判定「教师权限」的唯一依据（沿用 `CourseDetail.tsx`）：
```ts
const isOwner = !!(course && user && course.owner_id === user.id);
const isStudentView = isShared || (!!course && !isOwner);
const effectiveEditMode = isOwner && editMode; // 仅 Home 使用；其余 Tab 不看 editMode
```
> 除 Community 的「학생 카드直接输入 / 修改」用到 `editMode` 之外，Calendar / Notifications / Materials 三 Tab **不使用** `editMode`，而是用「行级 owner 校验」和 `canManage = !isStudentView` 直接控制按钮出现与否。

---

## 2) Instructions

### 2.1 Calendar Tab（`CalendarTab.tsx`）— 教师增量

固定入口与呈现（当前代码即为最终形态）：

- 顶部日历卡片，右侧「일정 추가」按钮**登录即可见**（不是教师专属），但删除按钮受下列行级判定约束：
  ```ts
  user && (item.owner_id === user.id || item.user_id === user.id)
  ```
  即：**只有事件创建者本人**（不论是教师还是学生）看得到该事件的垃圾桶图标；其他 owner 事件对当前用户只读。这是刻意的：教师不能替学生删事件、学生不能删教师日程。
- 表单字段 = `title / event_type(공지/과제/시험/수업/기타) / all_day / time / description`；写入表 `course_calendar_items`，附带 `item_type: "lesson"`（保留字段）。
- 「수업일」条纹与「공휴일」（`@hyunbinseo/holidays-kr`）与「사용자 일정」环形高亮完全由 `parseClassTime(course.class_time)` + `course.start_date` + `getHolidayNames` 客户端合成，**不写库**。
- 数据窗口固定为当前显示月：`gte(starts_at, startOfMonth) / lte(endOfMonth)`，切月即重取。

> 与 P5c 的区别：**P5c 讲的是 Calendar 的公共骨架（月视图、修课日/公休日渲染、右侧行程面板）**；本节只重申「删除按钮的行级 owner 判定」和「教师视角下事件添加对话框仍走同一 `insert` 路径」，不重画 UI。

### 2.2 Notifications Tab（`NotificationsTab.tsx` + `notifications/*`）— 教师增量

判定：
```ts
const isOwner = !!(user && course.owner_id && course.owner_id === user.id);
const effectiveStudentView = isStudentView || !isTeacher || !isOwner;
// 教师视角 = !effectiveStudentView
```

两栏布局 `grid lg:grid-cols-2`：
- 左栏渲染 `TeacherNoticeList` + `StudentPostList`，参数 `canManage = !effectiveStudentView`；
- 右栏根据 `effectiveStudentView` 二选一：`TeacherComposer` 或 `StudentComposer`。

**教师专属组件与行为（本文件唯一负责）：**

1. **`TeacherComposer`**
   - 类型：`공지 / 과제 / 일정`（固定 3 类，无「전체」）。
   - 6 个内置模板按钮：`교실 변경 / 수업 취소 / 퀴즈 공지 / 과제 마감 / 자료 업로드 / 일정 변경`，点击一键填入 textarea。
   - **AI 다듬기**：调用 edge function `polish-notice`，body `{ role: "teacher", type, content }`；402/429 由 Edge 返回 body 里的 `error` 字段读出；成功后弹 `AIPolishPanel`，可「이 내용으로 교체」或「취소」。
   - **미리보기**：切换本地面板，显示「유형 / 수신: 전체 학생」以及 `content`。
   - **발송**：`insert course_notices { course_id, author_id: user.id, type, content }`；发送后进入「발송 완료」视图并提供「새 공지 작성」重置。
   - **Toolbar 桥**：监听 `window` 上的 `COURSE_TOOLBAR_EVENT_NAME`；若 `detail.tab === "notifications"` 且 `detail.type ∈ {공지,과제,일정}`，则设为当前类型并重置 `sent/previewing`。**注意**：Toolbar 只在 Home Tab 显示（见 T4），此监听是为了教师从 Home 切到 Notifications 后类型自动带入。

2. **`TeacherNoticeList`（`canManage=true` 分支）**
   - 顶部筛选 `전체 / 공지 / 과제 / 일정`；卡片右上角三枚按钮：`답글`（展开 `RepliesThread`）/`Pencil` 编辑 / `Trash2` 删除。
   - 编辑对话框字段仅 `content` + `type`（选择 3 类），写回 `update course_notices ... where id=`。删除走 `delete course_notices`。
   - **不再有其它教师操作**：没有置顶、没有已读回执统计、没有导出（不要添加）。

3. **`StudentPostList`（`canManage=true` 分支）**
   - 教师额外筛选 `전체 / 전체공개 / 교사전용` 三项（映射到 `visibility = public / private`）。
   - 每条卡片教师可 `update course_student_posts.status` 至 `pending / approved / rejected / answered` 与 `delete`；状态芯片配色见 `statusChip` 字典。
   - `RepliesThread` 组件供双方回复：教师身份下写入 `notification_replies` 表并允许删除自己写的回复。

4. **Realtime**：`NotificationsTab` 订阅一个 channel `notif-${course.id}`，同时监听 `course_notices` 与 `course_student_posts` 的 `course_id=eq.${course.id}` 变更，每次触发 `fetchAll()`。学生名头像/emoji 从 `course_student_profiles`（按 `member_user_id` 建 map）合并。

> 与 P5d 的区别：**P5d 讲两栏结构、学生 Composer、可见性规则、状态语义与 UI 骨架**；本节只写「教师 Composer 六模板 / `polish-notice` 调用契约 / 教师端 List 的筛选与状态操作 / Toolbar 事件桥」这些**教师独占**逻辑，不重复公共两栏骨架和学生侧组件。

### 2.3 Lesson Materials Tab（`LessonMaterialsTab.tsx` + `materials/*`）— 教师增量

三个 sub-tab：`plans(강의안) / songs(노래) / attachments(자료)`，各挂一个面板并把 `count` 回吐给父组件用于 `Badge` 与「학생 시야空态自动切换」逻辑（学生态下空 sub-tab 直接隐藏；教师态永远显示三个）。

**`LessonPlanPanel`（教师增量）**
- 顶部 `Select` 只在 `plans.length > 1` 时出现，切换所选 `lesson_plans` 行；数据来源 `lesson_plans where course_id=` + `lesson_weeks in (plan_ids)`。
- Realtime 通道 `course-lesson-plans-${course.id}` 监听 `lesson_plans (course_id filter)` 与 `lesson_weeks (全表)` 的变更。
- 空态显示「연결된 강의안이 없습니다」，文案引导去「강의안 제작」页面创建，本 Panel **不做**创建入口（避免与 `Lessons` 页重复）。
- Week Accordion 内嵌 `WeekDetailView`（详细的 AI 生成、编辑、部分再生成走 T3a/T3b 已描述的流程）；本 Panel 不做额外的教师批量操作。

**`CourseSongPanel`（教师增量）**
- 教师态右上角显示 `Plus`「노래 추가」，打开 `SongPickerDialog`；学生态不显示。
- 选中一首歌 → `insert course_material_items { material_type:"song", song_id, title, description: "artist|hsk|video_id", url:youtube_url }`；`sort_order` 追加末尾。
- 每张卡片教师可删（`delete course_material_items where id=`），学生仅可点击跳到 `SongAnalysisDialog`（沿用 P3j 逻辑，不在本 Panel 内实现）。
- Realtime 通道 `course-songs-${course.id}` 监听 `course_material_items (course_id filter)`。

**`CourseAttachmentPanel`（教师增量）**
- 顶部 `DropdownMenu` 提供四类添加：`파일 업로드 / 링크 / 텍스트 메모 / 임베드 (YouTube)`，仅教师可见。
- 文件上传：走 `supabase.storage.from("materials").upload(...)`，**路径必须以 `${user.id}/courses/${course.id}/` 开头**（Storage RLS 强制）；文件名 `replace(/[^\w.\-]+/g, "_")` 保 ASCII。大小上限 `MAX_FILE_BYTES` 与后缀白名单 `ACCEPTED_FILE_EXT` 来自 `@/lib/file-icons`。写入 `course_material_items { material_type:"file", url:publicUrl, description:"mime|size|path" }`。
- Link/Text/Embed 统一走一个对话框 `handleAddByDialog`，`material_type ∈ {link,text,embed}`；`text` 允许空 URL 但要求 `description`；`embed` 主要用于 YouTube（用 `getYoutubeId` 抽 id 渲染 iframe）。
- 删除：教师确认 `confirm(...)` → 若是 `file` 且 `description` 里能解出 storage path 则先 `storage.from("materials").remove([path])`（失败静默），再 `delete course_material_items`。
- Realtime 通道 `course-attachments-${course.id}` 监听 `course_material_items (course_id filter)`。

> 与 P5c 的区别：**P5c 讲三 sub-tab 的 shell、`count` badge、学生空态自动切换与卡片渲染**；本节只写「三个 Panel 的教师专属写入路径 + Storage 命名约束 + 删除时的清理」。P5c 里不出现 `polish-notice`、不出现 storage upload 代码。

### 2.4 Community Tab（`CommunityTab.tsx`）— 教师增量

数据源：`course_student_profiles where course_id=` + `courses.share_token`；渲染 CSS Grid `repeat(auto-fill, minmax(160px,1fr))`。

**教师专属操作：**

1. **「학생 초대」按钮**：始终对 owner 可见，复制 `${origin}/shared/course/${share_token}`，toast 提示学生走「Google 로그인 → 정보 입력 → 자동 명단」流程。
2. **`editMode=true` 时**（沿用 Home 编辑开关的同一状态；`CourseDetail` 通过 `editMode` prop 传入）：
   - 空占位卡（`GhostAvatar`）出现「직접 입력」按钮，允许教师无学生登录时手工填卡片；
   - 已有卡片右上角显示铅笔按钮（hover 显现），教师可编辑他人卡；学生自己看到的自己卡片有「나」徽章且铅笔打开 `StudentOnboardingDialog`（学生自编）。
3. `renderEditorCard` 内联表单字段固定：`full_name / student_number / department / avatar_url`；保存走 `insert / update course_student_profiles`（含 `sort_order`）；不动 `language_level / study_years / gender / emoji`（这些走 `StudentOnboardingDialog` 完整表单）。
4. 卡片最少呈现 `MIN_PLACEHOLDER_COUNT = 6`；`editMode` 下至少再补 2 张占位卡以便快速新增。

> 与 P5d 的区别：**P5d 没有覆盖 Community**（P5d 只写 Notifications）。Community 的**公共渲染**（头像三态：图片 / emoji / 首字母；语言等级/学习年数/性别芯片）在其它 P5 系列不重复；本文件不重写渲染，只列教师增量。

---

## 3) Examples

### 3.1 `polish-notice` 调用最小样例（TeacherComposer）
```ts
const { data, error } = await supabase.functions.invoke("polish-notice", {
  body: { role: "teacher", type, content },
});
const errMsg = (data as any)?.error;
if (errMsg) return toast({ title: "AI 다듬기 실패", description: errMsg, variant: "destructive" });
if (error)  return toast({ title: "AI 다듬기 실패", description: error.message, variant: "destructive" });
setPolished((data as any).polished);
```

### 3.2 教师侧 List 状态更新
```ts
await supabase.from("course_student_posts").update({ status: "approved" }).eq("id", id);
```

### 3.3 附件上传路径约束
```ts
const path = `${user.id}/courses/${course.id}/${Date.now()}_${safeName}`;
await supabase.storage.from("materials").upload(path, file, { contentType: file.type });
```

### 3.4 Community 初始化 fetch
```ts
const [studentResult, courseResult] = await Promise.all([
  supabase.from("course_student_profiles")
    .select("id, full_name, student_number, department, avatar_url, sort_order, member_user_id, language_level, study_years, gender, emoji")
    .eq("course_id", course.id).order("sort_order"),
  supabase.from("courses").select("share_token").eq("id", course.id).maybeSingle(),
]);
```

---

## 4) Context

- 相关表：`course_calendar_items`、`course_notices`、`course_student_posts`、`notification_replies`、`course_material_items`、`lesson_plans`、`lesson_weeks`、`course_student_profiles`、`courses(share_token)`。
- Storage bucket：`materials`（public 读，写路径必须 `auth.uid()/…`）。
- Edge function：`polish-notice`（`role/type/content` → `{ polished }`；402/429 时 body 带 `error`）。
- Realtime 通道命名统一 `${resource}-${course.id}`，每个 Panel 独立订阅，unmount 时 `supabase.removeChannel(ch)`。
- 事件桥：`COURSE_TOOLBAR_EVENT_NAME`（来自 `CourseModuleToolbar`），只有 TeacherComposer 在监听。
- 权限判定：`isOwner` 由 `courses.owner_id === auth.uid()`；`canManage = !effectiveStudentView`；行级删除按钮由 `item.owner_id === auth.uid() || item.user_id === auth.uid()` 判断。RLS 已在库侧对齐（本文件不重写 RLS）。

---

## 5) Acceptance

- 教师身份下：
  - Notifications：可看到 TeacherComposer + AI 다듬기 + 미리보기 + 발송；NoticeList 显示编辑/删除/답글；StudentPostList 显示三档筛选 + 状态更新 + 删除。
  - Materials：三 sub-tab 恒亮；`강의안` 无创建入口但有多 plan 切换；`노래` 有「노래 추가」并可删；`자료` 有四类 Dropdown 添加、文件走 `materials` bucket 且 path 以 uid 开头、删除时清理 storage。
  - Community：可复制 `/shared/course/:token` 邀请链接；`editMode=true` 时占位卡出现「직接 입력」并允许编辑他人卡；卡数不足 6 自动补齐。
  - Calendar：`일정 추가` 可写入 `course_calendar_items`；只对**自己创建**的事件显示删除。
- 学生 / 分享链接身份下：
  - Notifications：仅看到 StudentComposer + 公开或自己的帖子；无筛选下拉、无 approve/reject。
  - Materials：空 sub-tab 隐藏，无添加/删除按钮，无上传 Dropdown。
  - Community：无 Ghost 卡「직접 입력」；只能编辑自己那张卡（走 `StudentOnboardingDialog`）。
  - Calendar：可自己添加事件，但不能删他人事件。
- 三个 Realtime 通道均在 unmount 后 `removeChannel`，无泄漏；类型切换、月份切换、行删除后 UI 立即反映。

---

## 6) 与 P5b / P5c / P5d / T4 / T5 的差异速查

| 文件 | 覆盖对象 | 与 T6 的重叠 | T6 的差异 |
|---|---|---|---|
| **P5b** | 7 个 Home Block 的渲染 + 编辑弹窗 | 无 | T6 不涉及 Home |
| **P5c** | Calendar 公共骨架 + Materials 三 sub-tab shell 与学生态显隐 | Calendar 与 Materials 的**呈现层** | T6 只写**教师写入路径**：`insert/update/delete`、Storage 上传、`SongPickerDialog`、多 plan `Select`、行级 owner 校验 |
| **P5d** | Notifications 两栏结构 + 学生 Composer + 可见性/状态语义 | Notifications 骨架 | T6 只写 TeacherComposer 六模板 + `polish-notice` 调用 + 教师筛选 + 状态更新 + Toolbar 事件桥 |
| **T4** | Home 编辑态、右侧 Toolbar、Hero、`buildBlockInsertPrefilled`、3 条 Home Realtime | Toolbar 事件的**发送方** | T6 是 Toolbar 事件的**接收方**（仅 TeacherComposer 监听 `tab === "notifications"`），且完全不动 Home |
| **T5** | `CreateCourseDialog / EditCourseDialog / Courses.tsx` 卡片 + 首次插入 7 默认块 | 无 | T6 不涉及 courses 行本身、不涉及默认块插入 |

不要在 T6 里出现的东西：`CourseCommentsSection`（学生视图未启用）、Home Block 编辑、`user_roles` 授予、`fork_public_item`、AI 강의안 생성、`bookmarked_weeks` 教师面板。

