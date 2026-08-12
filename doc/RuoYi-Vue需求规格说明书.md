# 若依管理系统（RuoYi-Vue）需求规格说明书

> **版本号**：V1.0  
> **日期**：2026-07-31  
> **来源项目**：https://gitee.com/y_project/RuoYi-Vue  
> **当前版本**：v3.9.2  
> **文档密级**：内部公开  

---

## 修订记录

| 版本 | 日期 | 修订人 | 修订内容 |
|------|------|--------|----------|
| V1.0 | 2026-07-31 | PM | 初稿，覆盖全部现有功能模块 |

---

## 1. 引言

### 1.1 项目背景

若依（RuoYi）是一套基于 Spring Boot 4.x + Vue 2.x 的前后端分离企业级后台管理系统快速开发框架。当前项目为 RuoYi-Vue 版本，面向中小型企业管理场景，提供用户权限管理、系统监控、代码生成等通用后台功能，可作为各类业务系统的开发底座。

### 1.2 目标用户

| 用户类型 | 说明 |
|----------|------|
| 超级管理员（admin） | 拥有全部权限，管理所有模块 |
| 普通管理员 | 按角色分配权限，管理指定模块 |
| 开发人员 | 使用代码生成器快速生成 CRUD 代码 |

### 1.3 产品定位

企业级后台管理基础平台，核心实现"用户-角色-菜单"三级权限管控体系，搭配系统监控、代码生成等提效工具。

### 1.4 技术栈一览

| 层级 | 技术选型 |
|------|----------|
| 后端框架 | Spring Boot 4.0.6 |
| 安全认证 | Spring Security + JWT |
| 数据库 | MySQL 8.0 |
| ORM | MyBatis / MyBatis-Spring-Boot 4.0.1 |
| 连接池 | Alibaba Druid 1.2.28 |
| 缓存 | Redis（Lettuce 客户端） |
| 定时任务 | Quartz |
| 接口文档 | Springdoc (Swagger 3) |
| 前端框架 | Vue 2.6.12 + Element UI 2.15.14 |
| 构建工具 | Maven 3.x / Vue CLI |

---

## 2. 系统架构概览

### 2.1 项目模块划分

```
RuoYi-Vue
├── ruoyi-admin       # 应用入口层（Controller + 启动类）
├── ruoyi-common       # 公共模块（实体类、工具类、基础定义）
├── ruoyi-framework    # 框架层（Security、配置、AOP、拦截器）
├── ruoyi-system       # 系统管理业务层（Service + Mapper）
├── ruoyi-quartz       # 定时任务模块
├── ruoyi-generator    # 代码生成模块
└── ruoyi-ui           # 前端 Vue 工程
```

### 2.2 运行环境要求

| 依赖项 | 版本要求 |
|--------|----------|
| JDK | 17+ |
| Maven | 3.9+ |
| MySQL | 8.0+ |
| Redis | 6.0+ |
| Node.js | ≥8.9（开发时） |

---

## 3. 功能需求详述

---

### 3.1 登录认证模块

**入口**：`/login`  
**默认账号**：admin / admin123  

#### 3.1.1 登录功能

**功能描述**：用户输入账号、密码、验证码，系统校验通过后签发 JWT Token，前端保存后用于后续请求认证。

**业务规则**：
- BR-001：登录需输入验证码（数学计算型或字符型，可配置切换）
- BR-002：密码连续错误次数达到上限（默认5次）后账号锁定（默认10分钟）
- BR-003：登录成功/失败均记录至 `sys_logininfor` 表
- BR-004：Token 有效期默认 30 分钟（可配置）
- BR-005：支持多终端同时登录，不互斥

**前后端数据流**：
```
前端 → POST /login { username, password, code, uuid } 
     → 后端校验验证码 → 校验密码 → 记录登录日志 
     → 生成 JWT Token → 存入 Redis 
     → 返回 { token }
```

**可配置项**（`application.yml`）：
| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| captchaType | math | 验证码类型（math/char） |
| user.password.maxRetryCount | 5 | 密码最大错误次数 |
| user.password.lockTime | 10 | 锁定时间（分钟） |
| token.expireTime | 30 | Token 有效期（分钟） |

#### 3.1.2 用户注册（可选）

**功能描述**：支持用户自主注册（默认关闭），由 `sys_config` 表中 `registerUser` 参数控制开关。

**入口**：`/register`  
**业务规则**：
- BR-006：注册开关由 `sys_config.registerUser` 控制（true/false）
- BR-007：注册成功后分配"普通角色"

#### 3.1.3 获取用户信息/路由

**功能描述**：登录后前端调用 `/getInfo` 获取用户基本信息及权限集合，调用 `/getRouters` 获取动态菜单路由。

**业务规则**：
- BR-008：根据用户角色查询权限集合（`sys_role_menu` → `sys_menu.perms`）
- BR-009：前端根据权限集合控制按钮级显示

#### 3.1.4 锁屏

**功能描述**：用户可临时锁定屏幕，解锁需重新输入密码。

**入口**：`/lock` → `/unlockscreen`  

---

### 3.2 用户管理模块

**模块路径**：`/system/user`  
**权限前缀**：`system:user:*`

#### 功能清单

| 功能 | 权限标识 | 说明 |
|------|----------|------|
| 用户列表查询 | system:user:query | 分页查询、支持多条件模糊搜索 |
| 新增用户 | system:user:add | 填写用户信息、分配角色和岗位 |
| 编辑用户 | system:user:edit | 修改用户信息，不可改密码 |
| 删除用户 | system:user:remove | 支持批量删除，admin 不可删自己 |
| 重置密码 | system:user:resetPwd | 重置为初始密码（默认 123456） |
| 修改状态 | system:user:edit | 启用/停用用户 |
| 导出 | system:user:export | 用户数据导出为 Excel |
| 导入 | system:user:import | 按模板批量导入用户 |

#### 用户实体关键字段

| 字段 | 类型 | 说明 |
|------|------|------|
| userName | varchar(30) | 登录账号，唯一 |
| nickName | varchar(30) | 用户昵称 |
| email | varchar(50) | 邮箱 |
| phonenumber | varchar(11) | 手机号 |
| sex | char(1) | 性别（0男/1女/2未知） |
| status | char(1) | 状态（0正常/1停用） |
| deptId | bigint | 所属部门 |
| delFlag | char(1) | 删除标记（0正常/2删除） |

#### 业务规则

- BR-010：用户名唯一，删除为逻辑删除（delFlag=2）
- BR-011：admin 用户不可删除、不可停用
- BR-012：批量删除时排除自己
- BR-013：导入功能需先下载模板，按模板格式填写

---

### 3.3 角色管理模块

**模块路径**：`/system/role`  
**权限前缀**：`system:role:*`

#### 功能清单

| 功能 | 权限标识 | 说明 |
|------|----------|------|
| 角色列表查询 | system:role:query | 分页查询 |
| 新增角色 | system:role:add | 配置角色权限范围 |
| 编辑角色 | system:role:edit | 修改角色信息 |
| 删除角色 | system:role:remove | 支持批量删除 |
| 分配用户 | system:role:edit | 为角色分配/移除已授权用户 |
| 数据权限 | — | 设置角色数据范围 |

#### 数据权限范围（data_scope）

| 值 | 含义 | 说明 |
|----|------|------|
| 1 | 全部数据权限 | 可查看所有部门数据 |
| 2 | 自定数据权限 | 指定可查看的部门 |
| 3 | 本部门数据权限 | 仅查看本部门 |
| 4 | 本部门及以下 | 本部门+所有子部门 |
| 5 | 仅本人 | 仅查看自己的数据 |

#### 业务规则

- BR-014：角色编码（roleKey）唯一
- BR-015：角色关联菜单权限（`sys_role_menu`）
- BR-016：admin 角色不可删除

---

### 3.4 菜单管理模块

**模块路径**：`/system/menu`  
**权限前缀**：`system:menu:*`

#### 功能清单

| 功能 | 权限标识 | 说明 |
|------|----------|------|
| 菜单列表 | system:menu:query | 树形结构展示 |
| 新增菜单 | system:menu:add | M/C/F 三种类型 |
| 编辑菜单 | system:menu:edit | 修改菜单属性 |
| 删除菜单 | system:menu:remove | 存在子菜单时不可删除 |
| 排序 | — | 拖拽排序 |

#### 菜单类型说明

| 类型 | 值 | 说明 |
|------|----|------|
| 目录 | M | 顶级导航目录，无实际页面 |
| 菜单 | C | 可点击的实际页面 |
| 按钮 | F | 页面内按钮级权限标识 |

#### 关键字段

| 字段 | 说明 |
|------|------|
| perms | 权限标识字符串，如 `system:user:list` |
| path | 前端路由路径 |
| component | 前端组件路径 |
| icon | 菜单图标 |
| isFrame | 是否外链（1跳新标签） |
| isCache | 页面是否缓存 |
| visible | 是否显示（0显示/1隐藏） |

#### 业务规则

- BR-017：类型 M 可包含 M/C，类型 C 可包含 F
- BR-018：存在子菜单时父菜单不可删除
- BR-019：权限标识 perms 全局唯一

---

### 3.5 部门管理模块

**模块路径**：`/system/dept`  
**权限前缀**：`system:dept:*`

#### 功能清单

| 功能 | 说明 |
|------|------|
| 部门列表 | 树形结构展示 |
| 新增/编辑/删除 | 标准 CRUD |
| 排序 | 同层内排序 |

#### 关键字段

| 字段 | 说明 |
|------|------|
| parentId | 父部门 ID，顶级为 0 |
| ancestors | 祖先链，如 `0,100,200` |
| orderNum | 排序号 |
| leader | 负责人 |
| phone / email | 联系方式 |

#### 业务规则

- BR-020：存在子部门时父部门不可删除
- BR-021：存在分配用户时部门不可删除

---

### 3.6 岗位管理模块

**模块路径**：`/system/post`  
**权限前缀**：`system:post:*`

#### 功能清单

| 功能 | 说明 |
|------|------|
| 岗位列表 | 分页查询 |
| 新增/编辑/删除 | 标准 CRUD |
| 导出 | 导出为 Excel |

#### 业务规则

- BR-022：岗位编码（postCode）唯一
- BR-023：岗位通过 `sys_user_post` 关联用户（多对多）

---

### 3.7 字典管理模块

**模块路径**：`/system/dict/type`、`/system/dict/data`  
**权限前缀**：`system:dict:*`

#### 功能清单

| 功能 | 说明 |
|------|------|
| 字典类型管理 | 定义字典类型（如 `sys_user_sex`） |
| 字典数据管理 | 为每种类型定义键值对（如 0=男, 1=女） |
| 缓存刷新 | 手动刷新字典缓存 |

#### 业务规则

- BR-024：字典类型编码（dictType）唯一
- BR-025：同一类型下字典标签（dictLabel）唯一
- BR-026：字典数据修改后需刷新缓存生效

---

### 3.8 参数管理模块

**模块路径**：`/system/config`  
**权限前缀**：`system:config:*`

#### 功能清单

| 功能 | 说明 |
|------|------|
| 参数列表 | 分页查询 |
| 新增/编辑/删除 | 标准 CRUD |
| 缓存刷新 | 手动刷新配置缓存 |

#### 预设系统参数

| 参数键 | 默认值 | 说明 |
|--------|--------|------|
| sys.user.initPassword | 123456 | 用户初始密码 |
| captchaEnabled | true | 验证码开关 |
| registerUser | false | 用户注册开关 |
| skinName | skin-blue | 默认皮肤主题 |
| sideTheme | theme-dark | 侧边栏主题 |
| initPasswordModify | false | 首次登录强制修改密码 |
| passwordValidateDays | 0 | 密码有效期（天），0=永不过期 |
| chrtype | 3 | 密码字符规则（位掩码） |
| login.blackIPList | — | 登录IP黑名单 |

#### 业务规则

- BR-027：参数键名（configKey）唯一
- BR-028：修改后需刷新缓存生效

---

### 3.9 通知公告模块

**模块路径**：`/system/notice`  
**权限前缀**：`system:notice:*`

#### 功能清单

| 功能 | 说明 |
|------|------|
| 公告列表 | 分页查询 |
| 新增/编辑/删除 | 标准 CRUD |
| **置顶通知** | 首页铃铛展示的未读通知 |
| **标记已读** | 单条/全部标记为已读 |
| **已读用户查询** | 查看哪些用户已读（自定义功能） |

#### 关键字段

| 字段 | 说明 |
|------|------|
| noticeId | 公告 ID |
| noticeTitle | 标题 |
| noticeType | 类型（1通知/2公告） |
| noticeContent | 富文本内容 |
| status | 状态（0正常/1关闭） |

#### 自定义扩展：已读追踪

本项目新增 `sys_notice_read` 表，记录每位用户对每条通知的阅读状态：

| 字段 | 说明 |
|------|------|
| noticeId | 通知 ID |
| userId | 用户 ID |
| readTime | 阅读时间 |

**对应接口**：
- `GET /system/notice/listTop` — 获取首页铃铛展示的置顶通知（含已读/未读状态）
- `POST /system/notice/markRead/{noticeId}` — 标记某条为已读
- `POST /system/notice/markReadAll` — 全部标记已读
- `GET /system/notice/readUsers/list?noticeId=xx` — 查询已阅读用户列表

---

### 3.10 系统监控模块

#### 3.10.1 在线用户（`/monitor/online`）

| 功能 | 权限 | 说明 |
|------|------|------|
| 在线用户列表 | monitor:online:query | 查看当前登录用户 |
| 强退用户 | monitor:online:forceLogout | 强制踢出指定用户 |
| 批量强退 | monitor:online:batchLogout | 批量踢出 |

#### 3.10.2 操作日志（`/monitor/operlog`）

| 功能 | 权限 | 说明 |
|------|------|------|
| 日志列表 | monitor:operlog:query | 分页查询，支持多条件筛选 |
| 删除 | monitor:operlog:remove | 支持批量删除 |
| 清空 | monitor:operlog:remove | 一键清空全部日志 |
| 导出 | monitor:operlog:export | 导出 Excel |
| **操作详情** | — | 查看单条日志的请求参数和返回结果（自定义） |

**记录字段**：操作模块、业务类型、请求方法、请求URL、操作人IP、操作时间，**请求参数 JSON**、**返回结果 JSON**、消耗时间(ms)

#### 3.10.3 登录日志（`/monitor/logininfor`）

| 功能 | 权限 | 说明 |
|------|------|------|
| 日志列表 | monitor:logininfor:query | 含登录结果（成功/失败）、失败原因 |
| 删除/清空 | monitor:logininfor:remove | |
| 导出 | monitor:logininfor:export | |
| **解锁账户** | monitor:logininfor:unlock | 解锁被锁定的用户 |

#### 3.10.4 服务监控（`/monitor/server`）

展示服务器实时运行状态：CPU 使用率、JVM 内存、磁盘 I/O、网络流量、系统信息（OS、JDK 版本、启动时间）

#### 3.10.5 缓存监控（`/monitor/cache`）

| 功能 | 说明 |
|------|------|
| 缓存列表 | 按 cacheName 分组展示 Redis 缓存键 |
| 查看缓存值 | 点击某个 key 查看 JSON 值 |
| 清除单个 key | 删除指定缓存项 |
| 清除某个 CacheName | 删除某个缓存组 |
| 清除全部缓存 | 一键清空 Redis |

**权限前缀**：`monitor:cache:*`

#### 3.10.6 数据监控（Druid 内置）

Druid 连接池监控控制台，地址 `/druid/*`：
- SQL 执行统计
- 慢 SQL 记录（>1s）
- 活跃连接数 / 连接池状态
- URI 请求监控
- Session 监控

**登录账号**：ruoyi / 123456（`application-druid.yml` 配置）

---

### 3.11 定时任务模块

**模块路径**：`/monitor/job`  
**权限前缀**：`monitor:job:*`

#### 功能清单

| 功能 | 权限 | 说明 |
|------|------|------|
| 任务列表 | monitor:job:query | 分页查询 |
| 新增任务 | monitor:job:add | 配置调用目标字符串、cron 表达式 |
| 编辑任务 | monitor:job:edit | 修改任务配置 |
| 删除任务 | monitor:job:remove | |
| 立即执行 | monitor:job:changeStatus | 手动触发一次 |
| 暂停/恢复 | monitor:job:changeStatus | 修改任务状态 |
| 任务日志 | monitor:job:query | 查看每次执行日志（结果、耗时、异常信息） |

#### 关键字段

| 字段 | 说明 |
|------|------|
| jobName | 任务名称 |
| jobGroup | 任务组（默认 DEFAULT） |
| invokeTarget | 调用目标字符串（Spring Bean 方法） |
| cronExpression | CRON 表达式 |
| misfirePolicy | 错失执行策略（1立即/2执行一次/3放弃） |
| concurrent | 是否并发执行（0允许/1禁止） |
| status | 状态（0正常/1暂停） |

#### 预置示例任务

| 任务 | invokeTarget | 说明 |
|------|-------------|------|
| 无参任务 | ryTask.ryNoParams | 示例：无参数方法 |
| 带参任务 | ryTask.ryParams('ry') | 示例：带字符串参数 |
| 多参任务 | ryTask.ryMultipleParams('ry', true, 100L, ...) | 示例：多类型参数 |

---

### 3.12 代码生成模块

**模块路径**：`/tool/gen`  
**权限前缀**：`tool:gen:*`

#### 功能清单

| 功能 | 权限 | 说明 |
|------|------|------|
| 表列表 | tool:gen:query | 展示已导入的数据库表 |
| 导入表 | tool:gen:import | 从数据库选择表导入 |
| 创建表 | — | 在 UI 中直接编写建表 DDL |
| 编辑配置 | tool:gen:edit | 配置生成参数（作者、包名、模块名等） |
| 预览代码 | tool:gen:preview | 预览即将生成的代码 |
| 生成代码 | tool:gen:code | 打包下载生成的 ZIP |
| 批量生成 | — | 多表同时生成 |
| 同步表结构 | — | 同步数据库表字段变更 |
| 删除 | tool:gen:remove | |

#### 生成内容

对每张表自动生成以下文件：
- `entity.java` — 实体类
- `mapper.java` + `mapper.xml` — 数据访问层
- `service.java` + `serviceImpl.java` — 业务层
- `controller.java` — 控制器
- Vue 文件（`index.vue` + `api.js`）— 前端页面

**生成模板路径**：`ruoyi-generator/src/main/resources/vm/`

---

### 3.13 其他工具模块

#### 3.13.1 表单构建器（`/tool/build`）

拖拽式可视化表单设计器，从左侧组件面板拖放组件到画布，生成 JSON 格式配置，可导出 Vue 页面文件。

#### 3.13.2 系统接口文档（`/swagger-ui.html`）

Swagger 接口文档，展示所有 API 接口的请求/响应结构，支持在线调试。分组包含：
- `default` 组：test 模块接口
- 其他分组由 `springdoc.group-configs` 配置

#### 3.13.3 文件上传/下载（`/common`）

- `POST /common/upload` — 单文件上传
- `POST /common/uploads` — 多文件上传
- `GET /common/download` — 文件下载（防目录穿越）
- `GET /common/download/resource` — 资源下载

---

## 4. 非功能性需求

### 4.1 安全需求

| 需求项 | 实现方式 |
|--------|----------|
| 身份认证 | JWT Token + Redis 存储 |
| 接口鉴权 | Spring Security + @PreAuthorize 注解校验 |
| XSS 防护 | 全局过滤器，默认开启 |
| SQL 注入防护 | MyBatis 参数化查询 |
| 密码加密 | BCrypt 加密存储 |
| 防盗链 | Referer 校验（默认关闭） |
| 文件上传限制 | 单文件 10MB，总请求 20MB |
| 登录失败锁定 | 5 次锁定 10 分钟 |
| 验证码 | 数学/字符验证码，防暴力破解 |

### 4.2 性能需求

| 指标 | 要求 |
|------|------|
| 页面加载时间 | 首屏 < 2s（开发环境） |
| 列表分页查询 | 响应时间 < 500ms |
| 最大并发线程 | 800（Tomcat 配置） |
| 数据库连接池 | 最大活跃连接 20 |
| 慢 SQL 告警 | > 1000ms 记录日志 |

### 4.3 兼容性

| 项 | 要求 |
|----|------|
| 浏览器 | Chrome、Firefox、Edge 最新两个大版本 |
| 数据库 | MySQL 8.0+ |
| JDK | 17+ |

### 4.4 可维护性

- 代码结构按功能模块拆分（Maven 多模块）
- 日志按模块输出（DEBUG 级别开发环境）
- 前后端分离，独立部署
- Swagger 文档自动生成

---

## 5. 数据库核心 ER 关系

```
sys_dept ──┬── sys_user ──┬── sys_user_role ─── sys_role ──┬── sys_role_menu ─── sys_menu
           │              │                                  │
           │              └── sys_user_post ─── sys_post      └── sys_role_dept ─── sys_dept
           │
           ├── sys_notice ─── sys_notice_read ─── sys_user（已读追踪扩展）
           │
           ├── sys_oper_log（操作日志）
           │
           ├── sys_logininfor（登录日志）
           │
           ├── sys_dict_type ──┬─ sys_dict_data
           │
           ├── sys_config（系统参数）
           │
           └── gen_table ──┬─ gen_table_column
                           │
           sys_job ─── sys_job_log
```

---

## 6. 接口总览

### 6.1 公共接口（无需认证）

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /login | 用户登录 |
| POST | /register | 用户注册 |
| GET | /captchaImage | 获取验证码 |
| GET | /v3/api-docs/** / swagger-ui.html | Swagger 文档 |
| GET | /druid/** | Druid 监控控制台 |

### 6.2 需认证接口（按模块统计）

| 模块 | 接口数 | 含权限控制 |
|------|--------|----|
| 用户管理 | ~15 | 是 |
| 角色管理 | ~13 | 是 |
| 菜单管理 | ~8 | 是 |
| 部门管理 | ~7 | 是 |
| 岗位管理 | ~6 | 是 |
| 字典管理 | ~11 | 是 |
| 参数管理 | ~9 | 是 |
| 通知公告 | ~10 | 是 |
| 文件上传下载 | 4 | 否 |
| 个人中心 | 4 | 否 |
| 在线用户 | 3 | 是 |
| 操作日志 | 6 | 是 |
| 登录日志 | 7 | 是 |
| 服务监控 | 1 | 是 |
| 缓存监控 | 7 | 是 |
| 定时任务 | 9 | 是 |
| 代码生成 | ~12 | 是 |
| 表单构建 | — | 纯前端 |
| **合计** | **~130+** | — |

---

## 7. 名词解释

| 术语 | 说明 |
|------|------|
| 角色（Role） | 权限的集合，一个用户可以拥有多个角色 |
| 权限标识（Perms） | 细粒度的接口访问控制字符串，如 `system:user:list` |
| 菜单路由 | 前端 Vue Router 的动态路由，由后端从 sys_menu 生成 |
| 数据权限 | 控制用户在指定模块能查看的数据范围（全部/部门/仅自己） |
| JWT | JSON Web Token，无状态身份认证令牌 |
| CRON 表达式 | 定时任务的调度时间规则表达式 |

---

> **附录**：本文档基于 RuoYi-Vue v3.9.2 现有功能整理，如需后续扩展功能，请在修订记录中追加。
