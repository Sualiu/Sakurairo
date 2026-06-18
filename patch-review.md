# PR #1407 Patch 最佳实践审查报告

> 审查依据: ~/.trae/skills/ 中的 WordPress skills 最佳实践
> - wp-plugin-development/references/security.md
> - wp-rest-api/SKILL.md
> - wp-rest-api/references/authentication.md
> - wp-rest-api/references/routes-and-endpoints.md
> - wp-rest-api/references/schema.md

---

## 审查核心原则（来自 WordPress Skills）

1. **sanitize/validate on input, escape on output** — 输入净化，输出转义
2. **Nonces help prevent CSRF, not authorization** — nonce 防 CSRF，不等于授权；必须配对 `current_user_can()`
3. **Never read `$_GET`/`$_POST` directly inside REST endpoints; use `WP_REST_Request`** — REST 回调中禁止直接访问超全局变量
4. **Always provide `permission_callback`** — REST 路由必须声明权限回调
5. **Define `args` with `type`, `default`, `required`, `validate_callback`, `sanitize_callback`** — REST 参数应注册 schema 验证
6. **Use `$wpdb->prepare()` for SQL; avoid building SQL with string concatenation** — SQL 必须参数化
7. **Return errors via `WP_Error` with an explicit `status`** — REST 错误应使用 WP_Error
8. **Do not call `wp_send_json()` in REST callbacks** — REST 回调中不要用 wp_send_json

---

## 逐 Patch 审查

### Patch 01: XSS — exhibition.php ✅ 符合最佳实践

**审查项:** 输出转义
- `esc_attr()` 用于 HTML 属性（class、data-* 属性） ✅
- `esc_html()` 用于 HTML 正文文本 ✅
- 符合 "escape on output" 原则

**无问题。**

---

### Patch 02: XSS — author.php ✅ 符合最佳实践

**审查项:** 输出转义
- `wp_kses_post(nl2br($description))` — 用户描述允许 HTML，使用 `wp_kses_post` 是正确的 ✅
- `esc_html__()` 用于翻译字符串输出 ✅
- `esc_attr()` 用于 HTML 属性 ✅

**无问题。**

---

### Patch 03: XSS — tpl/content-none.php ✅ 符合最佳实践

**审查项:** 输出转义
- `esc_url()` 用于 URL ✅
- `esc_attr()` 用于 title 属性 ✅
- `esc_html()` 用于正文文本 ✅

**无问题。**

---

### Patch 04: XSS — layouts/imgbox.php ✅ 符合最佳实践

**审查项:** 输出转义
- `esc_attr()` 用于 onclick、title、class 属性 ✅
- `esc_url()` 用于 img src、a href ✅
- `wp_kses_post()` 用于 inner HTML（可能包含 img 标签） ✅

**无问题。**

---

### Patch 05: XSS — inc/chatgpt/aigc-manage.php ✅ 符合最佳实践

**审查项:** 输出转义
- `esc_html()` 用于 API 端点、模型名等纯文本 ✅
- `esc_attr()` 用于 HTML 属性（value、class、data-*） ✅
- `esc_url()` 用于链接 URL ✅
- `wp_kses_post()` 用于消息文本（可能含 HTML） ✅

**无问题。**

---

### Patch 06: XSS — inc/theme-plus.php 头图 URL ✅ 符合最佳实践

**审查项:** 输出转义
- `esc_url()` 用于 background-image URL 和 data-src ✅

**无问题。**

---

### Patch 07: CSRF — AJAX 评论 nonce ⚠️ 部分符合，有改进空间

**审查项:** nonce + capability 配对

**符合项:**
- `wp_nonce_field('sakurairo_ajax_comment', 'sakurairo_comment_nonce')` 在表单中生成 nonce ✅
- `check_ajax_referer('sakurairo_ajax_comment', 'sakurairo_comment_nonce')` 在回调中验证 ✅

**问题:**
- WordPress skills 明确要求 **"Always pair nonces with capability checks"**。当前仅验证 nonce，未检查用户权限。
- 但评论提交是前台功能，未登录用户也应能评论，所以不加 `current_user_can()` 是合理的。
- **建议:** 对于已登录用户，可考虑在 `siren_ajax_comment_callback()` 中验证 `comment_author` 相关权限，但非必须。

**结论:** 当前实现可接受，评论场景下 nonce 已足够。

---

### Patch 08: 权限检查 — AIGC 管理表单 ✅ 符合最佳实践

**审查项:** nonce + capability 配对

- `current_user_can('manage_options')` 配对 `check_admin_referer()` ✅
- 6 处表单全部覆盖 ✅
- 完全符合 WordPress skills "nonces + permissions" 原则

**无问题。**

---

### Patch 09: 权限检查 — 分类图片保存 ⚠️ 部分符合，有改进空间

**审查项:** nonce + capability + 输入净化

**符合项:**
- `current_user_can('manage_categories')` ✅
- `wp_unslash()` + `sanitize_text_field()` ✅

**问题:**
- 缺少 nonce 验证。WordPress skills 要求 **nonce 和 capability 必须配对**。`z_save_taxonomy_image()` 钩在 `edit_term`/`create_term` 上，但未验证 nonce。
- **注意:** 此函数通过 WordPress 后台分类编辑表单触发，WordPress 核心在 term 编辑表单中已包含 nonce，但 `z_save_taxonomy_image` 是独立钩子，不自动继承该 nonce 验证。
- **建议:** 添加 nonce 验证，或在分类编辑页面中加入自定义 nonce field。

**结论:** 权限检查正确，但缺少 nonce 配对，存在 CSRF 风险。

---

### Patch 10: 权限检查 — 链接分类优先级 ⚠️ 同 Patch 09 问题

**审查项:** nonce + capability 配对

**符合项:**
- `current_user_can('manage_categories')` ✅

**问题:**
- 同 Patch 09，缺少 nonce 验证。`$_POST['term_priority']` 的保存没有 CSRF 保护。
- **建议:** 添加 nonce 验证。

**结论:** 权限检查正确，但缺少 nonce 配对。

---

### Patch 11: CSS 注入 — dash-scheme.php ⚠️ 部分符合，有改进空间

**审查项:** 输入净化 + 输出安全

**符合项:**
- `_sanitize_hex_color()` 使用正则验证 3/6 位 hex ✅
- `$rules` 过滤 `@import`、`expression()`、`javascript:` 等危险模式 ✅

**问题:**
1. **`_get()` 函数未净化** — 最后一条 commit 将其改为 `strip_tags(stripslashes())`，但我们已忽略该 commit。当前 `_get()` 直接返回 `$_GET[$str]`，未经任何净化。`_sanitize_hex_color()` 虽然会二次验证颜色值，但 `$rules` 的 `urldecode(_get('rules'))` 依赖 `_get()` 的原始值。
   - **建议:** 应同时采纳最后 commit 中对 `_get()` 的修复（`strip_tags(stripslashes())`），否则 `$rules` 的过滤可能被绕过。

2. **`strip_tags()` 在 `$rules` 中使用** — 此文件作为独立 CSS 输出，未经 WP 引导，使用 `strip_tags()` 是正确的（最后 commit 的逻辑）。但当前 patch 中 `$rules` 的 `strip_tags()` 是新增的，而 `_get()` 的 `strip_tags()` 被忽略了，存在不一致。

**结论:** 颜色验证和危险模式过滤正确，但 `_get()` 函数缺少净化是遗漏。建议将最后 commit 中 `_get()` 的修复也纳入此 patch。

---

### Patch 12: 输入验证 — theme-plus.php 评论验证码 ✅ 符合最佳实践

**审查项:** 输入净化

- `sanitize_text_field(wp_unslash($_POST['captcha']))` ✅
- `sanitize_text_field(wp_unslash($_POST['timestamp']))` ✅
- `sanitize_text_field(wp_unslash($_POST['id']))` ✅
- 符合 "never process the entire `$_POST` array; read explicit keys" 原则 ✅
- 符合 "use `wp_unslash()` before sanitizing" 原则 ✅
- 移除 WP 4.4 版本检查合理（style.css 声明 Requires 6.0） ✅

**无问题。**

---

### Patch 13: 输入验证 — search.php ✅ 符合最佳实践

**审查项:** 输入净化

- `array_map('sanitize_key', explode(',', wp_unslash($_GET['content_type'])))` ✅
- `sanitize_key` 适用于 WordPress 的 key 类参数（仅允许小写字母数字、连字符、下划线） ✅
- `wp_unslash()` 先于净化 ✅

**无问题。**

---

### Patch 14: 开放重定向 — wp_redirect → wp_safe_redirect ✅ 符合最佳实践

**审查项:** 重定向安全

- `wp_safe_redirect()` 替代 `wp_redirect()` ✅
- `admin_url()` 替代 `get_option('siteurl')` 拼接 ✅
- `esc_url_raw()` 用于重定向目标 ✅

**无问题。**

---

### Patch 15: SSRF — file_get_contents → wp_remote_get ✅ 符合最佳实践

**审查项:** 外部请求安全

- `wp_remote_get()` 替代 `file_get_contents()` ✅
- `esc_url_raw()` 用于请求 URL ✅
- 超时限制（5s/10s）防止 DoS ✅
- `urlencode($qq_number)` 避免 URL 注入 ✅
- `isset($matches[1])` 检查避免未定义索引 ✅
- 空值回退原 `$avatar` 而非硬编码无效路径 ✅

**无问题。**

---

### Patch 16: SSRF + 加解密 IV — QQ.php ✅ 符合最佳实践

**审查项:** SSRF + 功能 Bug 修复

- `wp_remote_get()` + 5s 超时 ✅
- `is_wp_error()` 检查 ✅
- `isset()` 检查避免未定义索引 ✅
- `urlencode($qq)` ✅
- IV 不匹配修复：从密文前 16 字节提取 IV，与加密端一致 ✅
- 解密失败检查 + QQ 号格式验证 ✅
- 密钥/参数空值检查 ✅

**无问题。** 这是整个 PR 中最关键的修复。

---

### Patch 17: REST API 规范化 — WP_REST_Request ⚠️ 部分符合，有重大改进空间

**审查项:** REST API 最佳实践

**符合项:**
- 回调函数加入 `WP_REST_Request $request` 参数 ✅
- `$_GET` 改为 `$request->get_param()` ✅
- `$_FILES` 改为 `$request->get_file_params()` ✅
- `intval()` 用于 page 参数 ✅
- `sanitize_text_field()` 用于 userID ✅

**问题（严重）:**

1. **`sakura_verify_rest_request_nonce()` 未定义** — PR 中所有 REST 回调都改用了这个函数，但当前代码库中不存在此函数。必须在应用此 patch 前定义它，否则所有 REST API 端点将 500 报错。
   - **建议:** 在 `inc/api.php` 或 `functions.php` 中添加：
     ```php
     function sakura_verify_rest_request_nonce(WP_REST_Request $request) {
         $nonce = $request->get_param('_wpnonce');
         return $nonce && wp_verify_nonce($nonce, 'wp_rest');
     }
     ```

2. **REST 参数未注册 schema 验证** — WordPress skills 明确要求：
   > "Define `args` with `type`, `default`, `required`, `validate_callback`, `sanitize_callback`"
   
   当前 `register_rest_route()` 的 `args` 全部缺失。例如 `bgm_bangumi` 的 `userID` 和 `page` 参数应在路由注册时声明类型和验证回调，而非仅在回调中手动 `sanitize_text_field`。
   - **建议:** 在 `register_rest_route()` 中添加 `args` 定义，如：
     ```php
     register_rest_route('sakura/v1', '/bangumi', array(
         'methods' => 'POST',
         'callback' => 'bgm_bangumi',
         'permission_callback' => '__return_true',
         'args' => array(
             'userID' => array(
                 'type' => 'string',
                 'required' => true,
                 'sanitize_callback' => 'sanitize_text_field',
             ),
             'page' => array(
                 'type' => 'integer',
                 'default' => 1,
                 'minimum' => 1,
             ),
         ),
     ));
     ```

3. **`permission_callback` 仍为 `__return_true`** — 所有路由的权限回调都是 `__return_true`（公开端点），但 nonce 验证在回调内部进行。WordPress skills 建议：
   > "Use capability checks in `permission_callback` (authorization), not just 'logged in'"
   
   当前模式（nonce 在回调中验证）是 WordPress 主题/插件中的常见做法，但不是最佳实践。理想情况下，nonce 验证应通过 `permission_callback` 实现。
   - **注意:** 这是一个架构层面的改进建议，不阻塞当前 patch 的应用。

**结论:** 核心逻辑正确，但 `sakura_verify_rest_request_nonce()` 必须先定义，否则会导致所有 REST API 崩溃。REST args schema 注册是重要改进项。

---

### Patch 18: nonce 绕过 — meting_aplayer ✅ 符合最佳实践

**审查项:** nonce 逻辑修复

**符合项:**
- 修复逻辑绕过：原 "传了 nonce 才验证" → 改为 "必须提供至少一个有效 nonce" ✅
- 使用 `wp_verify_nonce()` 直接验证而非 `check_ajax_referer()` ✅
- `$request->get_param()` 替代 `$_GET` ✅
- `sanitize_text_field()` 净化参数 ✅

**问题:**
- 同 Patch 17，依赖 `sakura_verify_rest_request_nonce()` 未定义的问题。

**结论:** 逻辑修复正确，符合 WordPress nonce 最佳实践。

---

### Patch 19: IP 伪造防护 — get_the_user_ip ⚠️ 部分符合，有风险

**审查项:** IP 获取安全

**符合项:**
- 移除 `HTTP_CLIENT_IP` 信任 ✅
- `X-Forwarded-For` 取最右侧 IP ✅

**问题:**

1. **缺少 IP 格式验证** — 最后一条 commit 添加了 `filter_var($candidate, FILTER_VALIDATE_IP)` 验证，但我们已忽略该 commit。当前 patch 直接信任 `end($forwarded_chain)` 的结果，如果 X-Forwarded-For 最右侧值不是合法 IP（如被注入恶意字符串），将导致 `get_the_user_ip()` 返回非法值。
   - **建议:** 必须纳入 `filter_var(FILTER_VALIDATE_IP)` 验证，否则 IP 伪造防护不完整。这是安全修复，不应忽略。

2. **`REMOTE_ADDR` 回退缺少 `isset` 检查** — 在 CLI 环境下 `$_SERVER['REMOTE_ADDR']` 可能不存在。当前 patch 已添加 `isset($_SERVER['REMOTE_ADDR'])` 检查 ✅。

**结论:** 方向正确，但缺少 IP 格式验证是安全隐患。建议将最后 commit 中的 `filter_var` 验证纳入此 patch。

---

### Patch 20: 输入验证 + 重定向 — get_qq_avatar ⚠️ 部分符合，有改进空间

**审查项:** REST API 输入验证 + 响应处理

**符合项:**
- `sanitize_text_field($request->get_param('qq'))` ✅
- 空值检查返回 400 ✅
- `esc_url_raw($imgurl)` 用于重定向 URL ✅
- 301 → 302（语义正确：302 临时重定向） ✅
- `wp_remote_get(esc_url_raw($imgurl))` 替代 `file_get_contents()` ✅
- 解密失败返回 404 ✅

**问题:**

1. **type_2 分支的响应方式不符合 REST 规范** — 当前 patch 中 type_2 分支仍使用 `WP_REST_Response` + `echo $imgdata`，这会导致 REST 框架对二进制数据进行 JSON 编码，损坏响应。最后一条 commit 将其改为直接 `header()` + `echo` + `exit`，这是正确的做法。
   - WordPress skills: "Return data via `rest_ensure_response()` or a `WP_REST_Response`" — 但二进制数据是例外，必须绕过 REST 框架。
   - **建议:** 应将最后 commit 中 type_2 的直接输出修复纳入此 patch，否则 QQ 头像 type_2 模式将返回损坏数据。

2. **错误响应应使用 `WP_Error`** — WordPress skills 建议 "Return errors via `WP_Error` with an explicit `status`"。当前使用 `WP_REST_Response` 返回错误，虽然功能正确，但不符合最佳实践。
   - **建议:** 改为 `return new WP_Error('rest_qq_missing', 'Missing qq parameter', array('status' => 400));`

3. **`qq` 参数应在路由注册时声明 args** — 同 Patch 17 的问题。

**结论:** 输入验证和重定向修复正确，但 type_2 二进制输出问题必须修复，否则功能损坏。

---

## 汇总

### ✅ 完全符合最佳实践（无需修改）

| Patch | 类型 | 说明 |
|-------|------|------|
| 01 | XSS | exhibition.php 输出转义 |
| 02 | XSS | author.php 输出转义 |
| 03 | XSS | tpl/content-none.php 输出转义 |
| 04 | XSS | layouts/imgbox.php 输出转义 |
| 05 | XSS | aigc-manage.php 输出转义 |
| 06 | XSS | theme-plus.php 头图 URL |
| 08 | 权限 | AIGC 管理 current_user_can + nonce |
| 12 | 输入 | 评论验证码参数净化 |
| 13 | 输入 | search.php content_type 净化 |
| 14 | 重定向 | wp_safe_redirect |
| 15 | SSRF | file_get_contents → wp_remote_get |
| 16 | SSRF+Bug | QQ.php IV 不匹配 + wp_remote_get |

### ⚠️ 部分符合，需要补充修改

| Patch | 问题 | 建议 |
|-------|------|------|
| **07** | 评论 nonce 无 capability 配对 | 可接受（评论是公开功能），但可考虑对已登录用户做额外验证 |
| **09** | 分类图片保存有 capability 但无 nonce | **添加 nonce 验证**，与 capability 配对 |
| **10** | 链接分类优先级有 capability 但无 nonce | **添加 nonce 验证**，与 capability 配对 |
| **11** | `_get()` 函数未净化 | **纳入最后 commit 的 `_get()` 修复**（`strip_tags(stripslashes())`） |
| **17** | `sakura_verify_rest_request_nonce()` 未定义；REST args 缺 schema | **必须先定义该函数**；建议在路由注册时添加 args schema |
| **19** | 缺少 IP 格式验证 | **纳入最后 commit 的 `filter_var(FILTER_VALIDATE_IP)` 验证** |
| **20** | type_2 二进制输出损坏；错误响应应用 WP_Error | **纳入最后 commit 的 type_2 直接输出修复**；建议改用 WP_Error |

### 🔴 阻塞性问题（必须修复才能应用）

1. **Patch 17: `sakura_verify_rest_request_nonce()` 未定义** — 应用 Patch 17/18/20 前必须先定义此函数，否则所有 REST API 端点 500 报错
2. **Patch 20: type_2 二进制输出损坏** — 必须纳入直接输出修复，否则 QQ 头像 type_2 模式功能损坏

### 📋 建议从"已忽略"中恢复的修复

以下 3 项原被忽略的修复，经最佳实践审查后发现**必须纳入**：

1. **dash-scheme.php `_get()` 函数净化** → 纳入 Patch 11
2. **get_the_user_ip `filter_var` 验证** → 纳入 Patch 19
3. **get_qq_avatar type_2 二进制直接输出** → 纳入 Patch 20

以下 2 项可继续忽略（属于增强而非必须）：

4. ~~upload_image() 上传文件存在性验证~~ — 增强项，非安全漏洞
5. ~~categories-images.php esc_url_raw() 替代 sanitize_text_field()~~ — 改进项，sanitize_text_field 对 URL 不够精确但不会导致安全问题
