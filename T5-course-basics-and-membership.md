# T5 · 班级基础信息与成员入场（教师增量）

> **归属**：教师端（Teacher-side）
> **配对文件**：P5a / P5b / P5c / P5d（公共骨架、7 大 Home 块、Calendar/Materials、Notifications/Community）与 T4（Hero 编辑 / Toolbar / Prefill / Realtime）
> **本文件范围**：`过程`（course）本身的**基础信息生命周期**与**成员进场机制**。即：教师如何**创建 / 编辑 / 排序 / 移入回收站** 一个班级卡片，以及学生如何通过链接**加入 / 完成首入场档案**。不涉及 Home 块内部渲染、Realtime 同步细节（已在 T4 覆盖）。

---

## 1 · Identity（模块身份）

- **入口位置**：
  - 侧边栏 `과정 관리` → `/courses`（`src/pages/Courses.tsx`）。
  - Workspace 首屏 `내 반` 卡片列表 → 「새 반 만들기」按钮（`MyClassesSection.tsx`）。
  - 学生端：任意页面通过 `학생 홈` 上的「반 참여」按钮 → `JoinCourseDialog`。
- **目的**：
  1. 让教师用一次 AI 调用即可从既有强义案（lesson plan）**孵化出完整班级**（含 7 个默认 Home 块）。
  2. 让教师能在**不进入班级详情页**的情况下修改「卡片可见信息 + 学生进入后看到的第一段介绍」。
  3. 让教师能拖拽卡片排序（`sort_order`）、软删除（`move_to_trash` RPC，7 日回收）。
  4. 让学生通过**链接 / 分享 token / UUID** 三种方式命中同一个班级，并在首次进入时补齐 `course_student_profiles`。

---

## 2 · Instructions（组件与业务规则）

### 2.1 「新 반 만들기」— `CreateCourseDialog`

> 位置：`src/components/courses/CreateCourseDialog.tsx`
> 触发 props：`open`、`onCreated()`、可选 `preselectPlanId`（从 Workspace 的强义案卡片 →「이 강의안으로 반 만들기」传入）。

**表单字段**（唯一强制项：강의안）：

| 字段 | 组件 | 默认 | 说明 |
| --- | --- | --- | --- |
| 강의안 | `Select`（列出 `lesson_plans` 全量，含 level 徽章、`course_id` 存在时显示「연결됨」徽章） | `preselectPlanId ?? ""` | 必填。选中后展示一个 muted 卡片显示 title + level + status。 |
| 학기 | `SemesterPicker`（年份 ±3、学期枚举 `1학기 / 여름학기 / 2학기 / 겨울학기`） | 当年 1학기 | 可空。 |
| 수업 시간 | `ClassTimePicker`（7 星期多选按钮 + 起止时 15 分钟粒度） | 空 | 可空。 |
| 수업 시작일 | `<input type="date">` | 空 | 决定 Home 首页「이번 주」计算基准。 |

**保存动作 `handleSave` 严格顺序**（任何一步失败即 rollback toast 展示 message）：
1. `supabase.from("lesson_weeks").select("id, lesson_plan_id, week_number, title, week_type")` 拉取周次列表用作 AI 上下文。
2. `supabase.functions.invoke("generate-course-content", { body: { lesson_plan: {title, level, weeks}, semester, class_time } })` → 返回 `{ courseTitle, courseSubtitle, introduction, duration, studentsTarget, goals[], weeks[], songs[], vocab[] }`。
3. `supabase.from("courses").insert({...})`：`name = aiContent.courseTitle || plan.title`，`introduction = aiContent.introduction || fallback`，`description = aiContent.courseSubtitle`。
4. **强义案归属检查 + 分叉**：若所选 plan 是公用模板（`owner_id === null`）或属于他人，先调用 `rpc("fork_public_item", { _table: "lesson_plans", _id })` 获得 `effectivePlanId`，再 `update({ course_id }).eq("id", effectivePlanId)`。若 `linked.length === 0` 抛 `강의안 연결에 실패했습니다.`。
5. **一次性写入 7 个默认 Home 块**（sort_order 0–6）：`course_header, course_info, goals, weekly_preview, curriculum, songs_list, vocab_grid`。各块 `content` 直接落 AI 的对应字段；`weekly_preview.selected_plan_id = effectivePlanId`。
6. Toast「과정 생성 완료」→ `onCreated()`。

**UI 状态**：`saving = true` 时锁按钮显示「AI가 과정 콘텐츠를 생성 중... (약 15초)」，Dialog 的 `onOpenChange` 也拦截关闭。

### 2.2 「과정 정보 수정」— `EditCourseDialog`

> 位置：`src/components/courses/EditCourseDialog.tsx`
> 触发：`SortableCourseCard` 的 kebab 菜单 → `수정`（或 Home 页 Hero editor 中的「메타 편집」入口）。

**字段与规则**：
- `name`（必填，去首尾空格）、`level`（초급 / 중급 / 고급）、`semester`（同 SemesterPicker）、`class_time`（同 ClassTimePicker）、`description`（一行卡片副标题）、`introduction`（进入班级时看到的段落）。
- 保存：`update({...}).eq("id", course.id)` 后广播 `window.dispatchEvent(new CustomEvent("course:updated", { detail: { id } }))`，Courses 页监听后 `fetchCourses()` 局部刷新（避免依赖 Realtime 延迟）。

### 2.3 卡片网格与排序 — `Courses.tsx` + `SortableCourseCard`

- 数据：`courses.select("*").is("deleted_at", null).order("sort_order").order("created_at desc")`，包 `fetchWithRetry`（Exponential Backoff + AbortController）。
- 隐藏公用项：`fetchHiddenIds("courses")` 集合过滤，实现「不喜欢的公用样例可从我的视图移除但不删除」。
- 拖拽：`@dnd-kit` + `rectSortingStrategy`；`onDragEnd` → `arrayMove` → `upsert([{id, sort_order:index}, ...])`，失败回滚 `fetchCourses()`。
- 复制分享链接：`${origin}/shared/course/${share_token}` → `navigator.clipboard.writeText`，2 秒后 `copiedId` 复位显示 √。
- 删除 = 软删除：`rpc("move_to_trash", { _table: "courses", _id })`，toast「7 일 후 자동 삭제됩니다」；从本地 state 立即移除。
- **公用课程的编辑 fork**：若 `!course.owner_id && !isAdmin`，`handleEditClick` 先 `forkPublicItem("courses", id)`，等 `fetchCourses()` 后再从新 id 读取记录并打开 `EditCourseDialog`。

### 2.4 「반 참여」— `JoinCourseDialog`

> 位置：`src/components/courses/JoinCourseDialog.tsx`

- **输入解析**：`UUID_RE = /[0-9a-f]{8}-[0-9a-f]{4}-…/i`，从「粘贴的整段链接 / 纯 ID / share_token」中提取第一个 UUID 命中。
- **预览**：`rpc("get_course_preview_by_token", { _token })` 匿名可读，返回 `{ id, name, level, introduction }`；未命中显示「수업을 찾을 수 없어요.」。
- **加入**：
  1. `select("id").eq("course_id", preview.id).eq("user_id", user.id).maybeSingle()` 判重；已存在则 toast「이미 참여 중인 반이에요.」并 `navigate`。
  2. `insert({ course_id, user_id, role: "student" })` into `course_members`；失败 toast 显示 e.message。
  3. 成功广播 `window.dispatchEvent(new Event("student-courses:refresh"))` 让 `StudentHome` 主动重取，随后 `navigate(/courses/${id})`。

### 2.5 「首入场档案」— `StudentOnboardingDialog`

> 位置：`src/components/courses/StudentOnboardingDialog.tsx`
> 触发：`CourseDetail` 进入时若 `course_student_profiles` 中 `member_user_id` 无记录 → `open=true`；「내 정보 수정」再次打开时 `existing` 有值。

**核心机制**：
- **首入场 prefill**：若 `existing == null`，从 `profiles.select("full_name, department, student_id, hsk_level, topik_level, learning_duration, gender, avatar_url")` 读入并按下表映射（保证下拉框合法值）：
  - `profileHskToBucket("HSK 3") → "HSK 1-3"`（1-3 / 4-6 / 7-9 三档）
  - `profileTopikToBucket("TOPIK 5") → "TOPIK 5-6"`（1-2 / 3-4 / 5-6 三档）
  - `profileDurationToYears("6개월") → "0.5"`，否则匹配整数（≥5 → "5"）
  - 若无 `avatar_url`，从 `PRESET_STUDENT_AVATARS` 随机取一张。
- **头像来源**：`PRESET_STUDENT_AVATARS`（12 张固定 PNG）+ 「내 사진 업로드」→ `storage.from("avatars").upload("${user.id}/student-avatar-${Date.now()}.${ext}")`，限制 5MB 且 `image/*`。**已弃用** emoji 头像，仅保留 `STUDENT_EMOJIS` 常量给 CSV 导入面板做兜底。
- **表单字段**：이름*, 학번, 학과, HSK, TOPIK, 학습 기간（6개월 미만 / 1~5년 이상）, 성별（RadioGroup: 남 / 여 / 기타）。
- **保存**：
  - 组合 `language_level = [hsk!=없음, topik!=없음].join(" / ") || null`。
  - `existing?` → `update(...).eq("id", existing.id)`；否则先 `select("sort_order").order desc limit 1 maybeSingle` 拿 `nextSort = (max ?? -1) + 1`，再 `insert({ ..., sort_order: nextSort, emoji: null })`。
  - toast「${courseName}에 오신 것을 환영합니다」，`onSaved(id)` 关闭对话框。

### 2.6 `SemesterPicker` / `ClassTimePicker` 复用规则

- 两者都是**受控组件**：接收字符串 value（例如 `"2025년 1학기"`、`"월/수 10:00~11:30"`），内部 `parseXxxString` 拆分为结构再展示；每次改变都 `onChange(formatXxxString(...))` 回传字符串——**保持数据库里存的仍是纯文本，不要为它们建结构化字段**。
- CreateCourseDialog / EditCourseDialog / 后续任何要输入学期或上课时间的地方**都必须复用它们**，避免出现自定义 `<input>` 打破解析格式。

---

## 3 · Examples（可复现输入 / 输出对照）

- 输入：教师在 Workspace `내 강의안` 卡片点击「이 강의안으로 반 만들기」，`preselectPlanId="plan_1"`（公用样例，owner_id=null）；表单填 `2025년 2학기 / 월·수 10:00~11:30 / 2025-09-01`。
- 期望：
  1. Edge function 返回 `{ courseTitle:"test기초 中国语", weeks:[...15], songs:[...], vocab:[...] }`。
  2. `fork_public_item("lesson_plans","plan_1")` → `plan_1_forked`；`courses` 插入后拿 `courseId`；`update({course_id:courseId}).eq("id","plan_1_forked")` 成功。
  3. `course_home_blocks` 插入 7 行（sort_order 0..6，types 严格上面列出的顺序）。
  4. Courses 列表刷新，新卡片出现在末尾（`sort_order` 为最大值 +1 由 Realtime 拉回）。

- 输入：学生粘贴 `https://melodyclass.app/shared/course/9b1c…` 到 JoinCourseDialog。
- 期望：`extractCourseId` 命中 UUID → 预览「test기초 中国语 · 초급」；点击「참여하기」→ `course_members.insert` → 广播 refresh → `navigate("/courses/9b1c…")` → `CourseDetail` 检测无 profile → 打开 `StudentOnboardingDialog`，头像位默认展示注册时的信息。

---

## 4 · Context（外部依赖）

- **表**：`courses`（含 `share_token`, `sort_order`, `deleted_at`, `owner_id`），`course_members`（教师 = owner_id 自动、学生 = 通过 dialog 插入 role="student"），`course_student_profiles`（唯一键 `(course_id, member_user_id)`），`course_home_blocks`（7 默认块由 CreateCourseDialog 一次插入），`lesson_plans` + `lesson_weeks`（AI 上下文与分叉源）。
- **RPC**：`get_course_preview_by_token(_token uuid)`（`SECURITY DEFINER`，允许匿名读取受限字段），`fork_public_item(_table, _id)`（返回新 id），`move_to_trash(_table, _id)`（软删除，与 `Trash.tsx` 页面对齐）。
- **Storage**：`avatars`（公开 bucket，路径以 `user.id` 前缀写入）。
- **Edge Function**：`generate-course-content`（gpt-4o-mini，10 秒左右返回；捕获 429/402 → Korean toast）。
- **事件**：`course:updated`（本地窗口事件，触发列表刷新），`student-courses:refresh`（学生端首页监听）。
- **样式**：卡片 hover 显示 kebab 菜单，dnd-kit `distance: 8px` 激活阈值防止误拖。

---

## 5 · Acceptance（验收清单）

- [ ] 无强义案时 CreateCourseDialog 显示「아직 생성된 강의안이 없습니다」并禁用保存按钮。
- [ ] AI 调用中关闭 Dialog 被拦截；调用失败 toast 显示 `e.message`，不产生残留 course。
- [ ] 公用强义案被选中时，插入 `courses` 后 `lesson_plans.course_id` 一定指向**新 forked id**，原公用 plan 不被改动。
- [ ] EditCourseDialog 保存后 Courses 列表卡片名字 / 副标题即时更新（通过 `course:updated` 事件，无需等 Realtime）。
- [ ] 卡片拖拽后刷新页面顺序保持一致；upsert 失败自动 `fetchCourses` 回滚。
- [ ] 删除卡片走 `move_to_trash`，7 日内可在 `/trash` 恢复。
- [ ] 非管理员点击公用课程「수정」时先 fork 再打开 Edit（toast 提示「개인 사본 생성 중...」）。
- [ ] JoinCourseDialog 支持三种输入：完整链接、`/courses/UUID` 尾段、纯 UUID。重复加入不报错但跳转。
- [ ] StudentOnboardingDialog 首入场时 HSK/TOPIK/성별/头像**全部预填**；未注册档案时随机分配 12 张头像之一。
- [ ] 上传自定义头像时非图片或 >5MB 立即 toast 报错，不发起 upload 请求。

---

## 6 · 与 P5b / P5c / T4 的分工区别（必读）

| 关注点 | 本文件 (T5) | P5b (Home Blocks) | P5c (Calendar / Materials) | T4 (Home 教师增量) |
| --- | --- | --- | --- | --- |
| `courses` 表的行级 CRUD（name/level/semester/class_time/description/introduction） | ✅ 唯一权威 | ✗ | ✗ | ✗（只涉及 Hero 图片列） |
| 7 个默认 Home 块**首次插入**（AI 生成） | ✅ 由 CreateCourseDialog 完成 | ✗（只讲已有块的渲染） | ✗ | ✗ |
| 7 个 Home 块的**结构 / 渲染 / 编辑弹窗** | ✗ | ✅ | ✗ | ✗ |
| Hero 图上传 / Pixabay 检索 / 移除 | ✗ | ✗ | ✗ | ✅ |
| 编辑态开关、右侧 Toolbar 追加块、`buildBlockInsertPrefilled` | ✗ | ✗ | ✗ | ✅ |
| 3 条 Realtime 通道（courses 行 / course_header block / blocks 列表） | ✗ | ✗ | ✗ | ✅ |
| 加入班级（`JoinCourseDialog` + `course_members.insert`） | ✅ | ✗ | ✗ | ✗ |
| 首入场档案 `StudentOnboardingDialog` + `course_student_profiles` | ✅ | ✗ | ✗ | ✗ |
| Calendar Tab（`schedule_events` 事件、周视图、教师/学生权限区分） | ✗ | ✗ | ✅ | ✗ |
| Materials Tab（三 Panel 教师写入层：LessonPlan / Song / Attachment） | ✗ | ✗ | ✅ | ✗ |
| Notifications / Community | ✗ | ✗ | ✗（在 P5d） | ✗ |
| 卡片列表 + 拖拽排序 + 软删除 + 公用课程 fork | ✅ | ✗ | ✗ | ✗ |

一句话记忆：**T5 = 「班级这张卡片本身 + 学生怎么进」；P5b/T4 = 「进去之后 Home 页发生了什么」；P5c/P5d = 「Home 之外的 Calendar/Materials/Notice/Community」。**
