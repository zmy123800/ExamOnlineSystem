# 学生注册功能设计

日期：2026-10-12
状态：待评审

## 1. 背景与目标

当前系统只有管理员能建账号：管理员登录后台 → 用户管理 → 填写账号/姓名/角色/专业/班级 → 保存。普通学生想用系统，必须先找管理员开账号。

目标是让**学生可以自助注册**，并且注册入口就放在学生每天要用的那个登录页上——学生打开系统首页，看到登录框，底下就有「还没有账号？立即注册」。

## 2. 范围

**做什么**

- 用户端登录页（`http://localhost:8080/`）底部增加注册入口
- 新增注册页，学生填写账号、姓名、密码、确认密码、专业→班级（级联选择）
- 新增后端注册接口，注册出来的账号角色固定为学生
- 预置示例专业和班级数据，让注册功能开箱可用
- 修复新学生首次登录后首页 500 的缺陷（否则注册出来的账号一登录就报错，功能等于白做）

**不做什么**

- 不开放教师注册。教师仍由管理员在用户管理里创建（教师涉及专业归属，需要人工审核）
- 不做邮箱验证、手机验证、图形验证码（系统的登录接口本来就没有验证码）
- 不修改任何数据库表结构
- 不做"忘记密码"、"修改个人资料"
- 不改动管理员登录页

## 3. 现状勘察（关键事实）

这些是设计的前提，均已实际核对过：

| 事实 | 出处 | 对设计的影响 |
|---|---|---|
| `user.account` 上有 `UNIQUE KEY`，**全局唯一**（不分角色） | `SHOW CREATE TABLE user` | 账号占用检查必须覆盖所有角色；不能靠"同角色内唯一" |
| `major` 表 0 行，`clazz` 表 0 行 | `SELECT COUNT(*) FROM major/clazz` | **不预置数据，注册页的班级下拉就是空的，功能不可用** |
| `user` 表**没有** `clazz_id` / `major_id` 列，学生-班级关系存在 `user_clazz` 表 | `SHOW CREATE TABLE user` | `User.clazzId` 是 `@TableField(select=false)` 的传输字段，写库前必须置 null |
| `WebSecurityConfig` 第 73-77 行授权配置**整段被注释** | [WebSecurityConfig.java:73-77](../../../backend/src/main/java/com/zj/examsystem/config/WebSecurityConfig.java) | 所有接口当前无需 token 即可访问，注册接口不需要改安全配置 |
| 前端 axios `baseURL: '/api'`，带全局 loading 拦截器 | [loading.js:7-10](../../../frontend/src/services/loading.js) | 注册页复用 `this.$axios` 即可 |
| `dealSelect()` 失败分支写的是 `data.essage`（少一个 m，是 typo） | [response.js:8](../../../frontend/src/services/response.js) | 注册页**不复用** `dealSelect`，自己读 `response.data`，否则失败时提示为空 |
| `/clazz/loadMajorClazzCascader` 返回 `[{label, value, children:[{label, value}]}]` | [ClazzServiceImpl.java:34-54](../../../backend/src/main/java/com/zj/examsystem/service/impl/ClazzServiceImpl.java) | 正好匹配 Element Plus `el-cascader` 的默认 props，不用改后端 |
| `saveUser()` 中 `case 2`/`case 3` 分支结尾没有 `break` | [UserServiceImpl.java:40-53](../../../backend/src/main/java/com/zj/examsystem/service/impl/UserServiceImpl.java) | 依赖"insert 成功即 return"来兜住；注册接口只走 `case 3`，本次不动它 |
| `loadStudentData()` 直接 `correctList.get(0)` | [UserServiceImpl.java:226](../../../backend/src/main/java/com/zj/examsystem/service/impl/UserServiceImpl.java) | 新学生没有任何测验记录 → 空列表 → `IndexOutOfBoundsException` → 首页 500 |

## 4. 后端设计

新增一个接口：`POST /user/register`

**请求体**（与 `/user/login` 一致的 JSON 风格）：

```json
{ "account": "20250101", "password": "123456", "username": "张三", "clazzId": 1 }
```

**响应**：复用项目统一包装 `BaseResponseEntity<Boolean>`

| 情况 | 响应 |
|---|---|
| 成功 | `BaseResponseEntity.ok("注册成功", true)` |
| 账号或姓名或密码为空 | `error(FAIL, "请填写完整的注册信息")` |
| 账号已被占用 | `error(FAIL, "该账号已被占用，请换一个")` |
| 班级不存在 | `error(FAIL, "所选班级不存在，请重新选择")` |
| `saveUser` 返回 false | `error(FAIL, "注册失败，请稍后重试")` |

**实现要点**

1. 签名用 `@RequestBody User user`——与同文件的 `authenticateUser` 保持一致的接收风格，不新建 DTO 类。
2. **服务端强制 `user.setRoleId(3)`**，请求体里传什么角色都覆盖掉。这是安全边界：没有它，任何人 POST 一个 `roleId=1` 就能给自己造管理员账号。
3. 同时 `user.setUserId(null)`，防止客户端指定主键覆盖已有用户（`insert` 对已存在主键会失败，但显式置空更稳妥）。
4. 校验顺序（早失败，避免无谓的加密封装）：
   1. `account` / `password` / `username` / `clazzId` 任一为空 → 直接返回
   2. 查 `account` 是否已存在（`QueryWrapper.eq("account", ...)`）→ 已存在则返回
   3. 查 `clazzId` 在 `clazz` 表是否存在 → 不存在则返回
   4. 调用现成的 `userService.saveUser(user)`
5. 不在 Controller 里写业务，校验逻辑放进 `UserService.registerStudent(User user)`，与既有的 `saveUser` / `updateUser` 并列。Controller 只做协议转换。
6. `saveUser` 内部会把 `clazzId` 取出来存进 `user_clazz` 再置 null，注册接口不重复处理这件事。
7. 密码由 `saveUser` 里的 `passwordEncoder.encode()` 处理，**存库的是 BCrypt 哈希，不是明文**。

**改动文件**（3 个，均为改动，无新增 Java 文件）

- [UserController.java](../../../backend/src/main/java/com/zj/examsystem/controller/UserController.java) — 加 `/register` 端点
- [UserService.java](../../../backend/src/main/java/com/zj/examsystem/service/UserService.java) — 加 `registerStudent` 声明
- [UserServiceImpl.java](../../../backend/src/main/java/com/zj/examsystem/service/impl/UserServiceImpl.java) — 实现 `registerStudent` + 修复 `loadStudentData`

关于"账号已被占用"的竞态：并发下两个请求可能同时通过第 2 步检查，由数据库唯一索引兜底，`saveUser` 已经 catch 了 `DuplicateKeyException` 并返回 false，最终仍返回"注册失败，请稍后重试"。不需要额外的锁。

## 5. 前端设计

**新增页面** `frontend/src/views/Register.vue`

- 视觉完全复用用户端登录页：[Login.vue](../../../frontend/src/views/Login.vue) 的背景图 `user_login_bg.png`、`login-box` 卡片、`input-group` 输入框样式和 `@import "../assets/login/user_login_style.css"` 全部照搬，保证从登录页点进来视觉是连贯的
- 标题「注册账号」，底部一行「已有账号？返回登录」链接回 `/`
- 表单字段（自上而下）：
  1. 账号（`type="text"`）
  2. 姓名（`type="text"`）
  3. 密码（`type="password"`）
  4. 确认密码（`type="password"`）
  5. 专业 / 班级（`el-cascader`，数据来自 `GET /clazz/loadMajorClazzCascader`）
- `created()` 时拉取级联数据
- `el-cascader` 的 props 照抄 [User.vue:99-108](../../../frontend/src/views/admin/User.vue) 的用法（`:options` + `:show-all-levels="false"` + `:props="{ expandTrigger: 'hover' }"`）。注意那里写的 `filterable1` 是个无效属性名（拼写错误），照抄时不要带上
- 组件默认样式与登录页的自定义 CSS 输入框不一致，需要用 `:deep()` 覆盖宽度和圆角，让它和上方四个 `input-group` 对齐
- **无可用班级时**（返回空数组，或所有专业的 `children` 都为空）：禁用提交按钮，并在表单内显示「暂无可选班级，请联系管理员创建」——这正是当前数据库的真实状态，所以这条分支会被真实触发，不是摆设
- 提交前前端校验：五个字段都非空 → 两次密码一致 → 不满足则 `this.$message.warning(...)` 并中止
- 提交 `POST /user/register`，自己读 `response.data.success` / `response.data.message`（**不用 `dealSelect`**，原因见第 3 节）
- 成功后自动登录：紧接着 `POST /user/login`，body 为 `{roleId: 3, account, password}`，拿到 token 后 `this.$storage.setStorageSync("user", response.data, 10800000)` 再 `this.$router.push("/home")`，与 [Login.vue:100-106](../../../frontend/src/views/Login.vue) 的行为一致。学生不需要注册完再手动登一次
- 若自动登录意外失败（例如后端刚重启），提示「注册成功，请返回登录页登录」并跳回 `/`，保证注册结果不丢失

**改动文件**

- [Login.vue](../../../frontend/src/views/Login.vue) — 在「登录」按钮下方加一行 `<router-link to="/register">还没有账号？立即注册</router-link>`，用 `login-box` 内部已有样式，不引入新配色
- [router/index.js](../../../frontend/src/router/index.js) — 新增 `{ path: '/register', component: () => import('../views/Register.vue') }`

**路由守卫**：`/register` 必须加进 [router/index.js:139](../../../frontend/src/router/index.js) 的 `noAuthPath` 白名单。否则未登录用户访问 `/register` 会被 `next('/')` 弹回登录页——那就正好把新加的入口给堵死了。

## 6. 预置数据

`major` 和 `clazz` 目前都是空表，没有这份数据注册功能无法使用。需要插入 2 个专业 + 4 个班级：

```sql
INSERT INTO major (major_name) VALUES ('计算机科学与技术'), ('软件工程');
-- 假设上一步得到 major_id = 1, 2
INSERT INTO clazz (clazz_name, major_id) VALUES
  ('计科2101班', 1), ('计科2102班', 1),
  ('软工2101班', 2), ('软工2102班', 2);
```

实施时直接对当前数据库 `exam_online_system` 执行，并用 `SELECT LAST_INSERT_ID()` 或按名称回查确定真实的 `major_id`，**不硬编码 1/2**（两张表的 `AUTO_INCREMENT` 分别是 13 和 55，不是从 1 开始）。

这份 SQL 记录在本设计文档中，不往仓库塞额外的 `.sql` 文件——仓库是从别人那里 clone 的，保持干净。代价是重新导入数据库后需要再跑一次，实施时会向用户说明。

## 7. 附带缺陷修复

**`loadStudentData` 越界**（[UserServiceImpl.java:217-226](../../../backend/src/main/java/com/zj/examsystem/service/impl/UserServiceImpl.java)）

新注册的学生测验记录为空，`findCorrectnessByStudentId` 返回空列表，然后 `correctList.get(0)` 抛 `IndexOutOfBoundsException`，首页接口 500。

修复：列表为空时 `average` 和 `max` 都取 `0.0f`。

```java
List<Float> correctList = testHistoryMapper.findCorrectnessByStudentId(userId);
Float sum = 0.0f;
for (Float f : correctList) {
    sum += f;
}
result.put("average", correctList.isEmpty() ? 0.0f : sum / correctList.size());
result.put("max", correctList.isEmpty() ? 0.0f : correctList.get(0));
```

`sum / correctList.size()` 在列表为空时是 `0.0f / 0` = `NaN`，所以 average 也必须走同一个判空分支，只修 `max` 不够。

这不是"顺手重构"，而是注册功能的必要组成部分：不修它，注册出来的每一个学生账号首次登录都会看到报错的首页。已有学生（有历史记录）的行为完全不变。

## 8. 测试策略

**后端** 新增 `UserRegisterTest extends BaseSpringBootTest`（[BaseSpringBootTest.java](../../../backend/src/test/java/com/zj/examsystem/test/BaseSpringBootTest.java)）

| 用例 | 断言 |
|---|---|
| 正常注册 | 返回 true；`user` 表新增一行且 `role_id=3`；`user_clazz` 里 (user_id, clazz_id) 关联正确 |
| 密码加密 | 从库里查出的 password ≠ 明文，且以 `$2a$` 开头 |
| 账号重复 | 返回失败；`user` 表没有多出脏数据；**原账号的密码没有被改动** |
| 越权注册 | 请求体传 `roleId=2`，落库后 `role_id` 仍是 3 |
| 班级不存在 | `clazzId=999999` 时返回失败且不建用户 |
| 新学生首页数据 | 对刚注册的用户调 `loadStudentData`，不抛异常，`average`/`max` 均为 `0.0f` |

测试数据用时间戳生成唯一账号（如 `test_` + `System.currentTimeMillis()`），**不加类级 `@Transactional`**：`saveUser` 内部的 `DuplicateKeyException` 被 catch 后会把外层事务标记为 rollback-only，加事务注解反而会让"重复注册"用例行为失真。改为在 `@AfterEach` 里按账号前缀清理测试产生的数据。

**端到端** 用 Playwright（沿用已有的 `D:\tools\pw` 驱动，`channel: 'msedge'`）走一遍真实流程：

1. 打开 `http://localhost:8080/`
2. 断言页面上能看到「立即注册」入口 → 点击
3. 填写账号/姓名/密码/确认密码，选专业 → 班级
4. 点注册 → 断言自动跳转到 `/home` 且首页正常渲染（不是白屏或报错页）
5. 截图留证，同时收集浏览器控制台错误

**回归**：手动用 admin/123 登录管理员后台，确认用户管理页能看到新注册的学生、其角色显示为「学生」、班级正确——即注册出来的账号和管理员建的账号在后台完全等价。

## 9. 已否决的备选方案

| 方案 | 否决理由 |
|---|---|
| 复用 `/user/save` 做注册 | 该接口用表单参数接收 `User user`，且 `status` 参数直接拼进提示语，请求体里还能带任意 `roleId`——直接暴露它等于开放管理员自助注册 |
| 注册后跳回登录页手动登录 | 用户明确要求入口在登录页，说明希望流程尽量短；多一次输入账号密码纯属多余 |
| 注册不选班级（后期管理员分配） | 已确认不可行：学生首页、测验、成绩查询都以 `user_clazz` 关联为前提，没有班级的学生进系统什么都看不到 |
| 开放教师自助注册 | 教师绑定专业、能建科目和题目，属于需要审核的权限，保持管理员创建 |
| 注册页做独立视觉设计 | 从登录页点进去却换了一套风格会显得突兀，复用登录页样式成本也更低 |
