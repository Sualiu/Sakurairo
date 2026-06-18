# 类别 5: REST API 安全 — 规范化 + nonce 绕过 + 输入验证

> 包含 Patch: 17-18, 20
> 审查依据: wp-rest-api/SKILL.md, wp-rest-api/references/routes-and-endpoints.md, wp-rest-api/references/authentication.md, wp-rest-api/references/schema.md
> 风险等级: 严重（认证绕过 / 功能损坏）

---

## WordPress Skills 最佳实践

来自 `wp-rest-api/references/routes-and-endpoints.md`:

> **permission_callback (required)**
> - Always provide `permission_callback`.
> - Public endpoints should use `__return_true`.
> - For restricted endpoints, use capability checks (`current_user_can`) or object-level authorization.
> - Missing `permission_callback` emits a `_doing_it_wrong` notice in modern WP.
>
> **Arguments**
> - Register `args` to validate and sanitize inputs.
> - Use `type`, `required`, `default`, `validate_callback`, `sanitize_callback`.
> - Access params via the `WP_REST_Request` object, not `$_GET`/`$_POST`.
>
> **Return values**
> - Return data via `rest_ensure_response()` or a `WP_REST_Response`.
> - Return `WP_Error` with a `status` in `data` for error responses.
> - Do not call `wp_send_json()` in REST callbacks.

来自 `wp-rest-api/references/authentication.md`:

> **Cookie authentication (in-dashboard / same-site)**
> - Requires a REST nonce (`wp_rest`) sent as `X-WP-Nonce` header or `_wpnonce` param.
> - If the nonce is missing, the request is treated as unauthenticated even if cookies exist.

来自 `wp-rest-api/references/schema.md`:

> **Validation + sanitization**
> - Use `rest_validate_value_from_schema( $value, $schema )` then `rest_sanitize_value_from_schema( $value, $schema )`.
> - Common formats: `date-time`, `uri`, `email`, `ip`, `uuid`, `hex-color`.

REST API 安全核心原则：
1. **禁止直接访问 `$_GET`/`$_POST`** — 必须通过 `$request->get_param()` 获取参数
2. **必须提供 `permission_callback`** — 即使是公开端点也要 `__return_true`
3. **注册 `args` schema** — 声明 `type`/`required`/`default`/`sanitize_callback`
4. **错误返回 `WP_Error`** — 带有 `status` 的 `WP_Error` 是标准错误响应
5. **nonce 验证逻辑必须严格** — 不能"传了才验证"，必须"必须提供有效 nonce"

---

## 主题存在的问题分析

### 问题 1: REST 回调直接访问 $_GET/$_POST

`inc/api.php` 中所有 REST 回调函数直接访问 `$_GET`/`$_POST`，而非使用 `WP_REST_Request` 对象。

**违规代码:**
```php
function bgm_bilibili()  // 无 $request 参数
{
    $page = $_GET["page"] ?: 2;  // 直接访问 $_GET
```

**危害:**
1. 绕过 REST API 的参数验证和净化机制
2. 无法利用 REST schema 自动验证
3. `$_GET` 可能包含未净化的恶意数据

### 问题 2: nonce 验证逻辑绕过

`meting_aplayer()` 的 nonce 验证存在逻辑漏洞：

**违规代码:**
```php
if (in_array('_wpnonce', $_GET))
    $wpnonce = $_GET['_wpnonce'];
if (in_array('meting_nonce', $_GET))
    $meting_nonce = $_GET['meting_nonce'];
// 如果两个 nonce 都不传，$wpnonce 和 $meting_nonce 都未定义
// if 条件中 isset() 返回 false，整个条件为 false → 不拒绝请求！
if ((isset($wpnonce) && !check_ajax_referer('wp_rest', $wpnonce, false)) || (isset($meting_nonce) && !wp_verify_nonce($meting_nonce, $type . '#:' . $id))) {
```

**攻击方式:** 不传任何 nonce 参数 → `isset($wpnonce)` 和 `isset($meting_nonce)` 都为 `false` → 整个 if 条件为 `false` → 请求被放行。

### 问题 3: sakura_verify_rest_request_nonce() 未定义

PR 中所有 REST 回调都改用了 `sakura_verify_rest_request_nonce($request)` 函数，但该函数在代码库中不存在。如果不定义此函数，所有 REST API 端点将 500 报错。

### 问题 4: get_qq_avatar 输入未净化 + type_2 二进制输出损坏

```php
function get_qq_avatar()
{
    $encrypted = $_GET["qq"];  // 直接访问 $_GET，无净化
    $imgurl = QQ::get_qq_avatar($encrypted);
    if (iro_opt('qq_avatar_link') == 'type_2') {
        $imgdata = file_get_contents($imgurl);  // file_get_contents SSRF
        $response = new WP_REST_Response();
        $response->set_headers(array('Content-Type' => 'image/jpeg', ...));
        echo $imgdata;  // 二进制数据通过 REST 框架输出会被 JSON 编码损坏
    }
```

**问题:**
1. `$_GET["qq"]` 直接访问，无 `sanitize_text_field`
2. `file_get_contents($imgurl)` SSRF 风险
3. `echo $imgdata` 在 REST 回调中会被框架 JSON 编码，二进制数据损坏
4. 错误响应用 `WP_REST_Response` 而非 `WP_Error`

### 问题 5: REST 参数未注册 schema

所有 `register_rest_route()` 调用都缺少 `args` 定义，参数验证仅在回调中手动进行，不符合 WordPress skills 要求。

---

## Patch 清单

### Patch 17: REST API 规范化 — WP_REST_Request + nonce 函数定义

**文件:** [inc/api.php](file:///workspace/inc/api.php), [inc/classes/gallery.php](file:///workspace/inc/classes/gallery.php)

```diff
--- a/inc/api.php
+++ b/inc/api.php
@@ 文件顶部，新增函数定义 @@
+/**
+ * Verify REST request nonce from WP_REST_Request object.
+ * Supports both 'wp_rest' nonce and the legacy check_ajax_referer pattern.
+ */
+function sakura_verify_rest_request_nonce(WP_REST_Request $request) {
+    $nonce = $request->get_param('_wpnonce');
+    return $nonce && wp_verify_nonce($nonce, 'wp_rest');
+}

@@ -258,1 +258,1 @@
-function cache_search_json()
+function cache_search_json(WP_REST_Request $request)
 {
-    if (!check_ajax_referer('wp_rest', '_wpnonce', false)) {
+    if (!sakura_verify_rest_request_nonce($request)) {

@@ -286,7 +286,9 @@
     if (!check_ajax_referer('wp_rest', '_wpnonce', false)) {
         $output = array(
             'status' => 403,
             'success' => false,
             'message' => 'Unauthorized client.'
         );
-    } elseif ($_GET['qq']) {
-        $qq = $_GET['qq'];
-        $output = QQ::get_qq_info($qq);
     } else {
-        $output = array(
-            'status' => 400,
-            'success' => false,
-            'message' => 'Bad Request'
-        );
+        $qq = sanitize_text_field($request->get_param('qq'));
+        if (empty($qq)) {
+            $output = array(
+                'status' => 400,
+                'success' => false,
+                'message' => 'Bad Request'
+            );
+        } else {
+            $output = QQ::get_qq_info($qq);
+        }
     }

@@ -334,1 +336,1 @@
-function bgm_bangumi($request)
+function bgm_bangumi(WP_REST_Request $request)
 {
-    if (!check_ajax_referer('wp_rest', '_wpnonce', false)) {
+    if (!sakura_verify_rest_request_nonce($request)) {
         $response = array(
             'status' => 418,
             'success' => false,
             'message' => 'Unauthorized client.'
         );
         return new WP_REST_Response($response, 418);
-    } else {
-        $userID = $request->get_param('userID');
-        $page = $request->get_param('page') ?: 1;
-        $bgmList = new \Sakura\API\BangumiList();
     }
-    return $bgmList->get_bgm_items($userID, (int)$page);
+    $userID = sanitize_text_field($request->get_param('userID'));
+    $page = intval($request->get_param('page')) ?: 1;
+    $bgmList = new \Sakura\API\BangumiList();
+    return $bgmList->get_bgm_items($userID, $page);

@@ -351,1 +353,1 @@
-function bgm_bilibili()
+function bgm_bilibili(WP_REST_Request $request)
 {
-    $response = null;
-    if (!check_ajax_referer('wp_rest', '_wpnonce', false)) {
+    if (!sakura_verify_rest_request_nonce($request)) {
         $output = array(
             'status' => 403,
             'success' => false,
             'message' => 'Unauthorized client.'
         );
-        $response = new WP_REST_Response($output, 403);
-    } else {
-        $page = $_GET["page"] ?: 2;
-        $bgm = new \Sakura\API\Bilibili();
-        $html = preg_replace("/\s+|\n+|\r/", ' ', $bgm->get_bgm_items($page));
-        $response = new WP_REST_Response($html, 200);
+        return new WP_REST_Response($output, 403);
     }
-    return $response;
+    $page = intval($request->get_param('page')) ?: 2;
+    $bgm = new \Sakura\API\Bilibili();
+    $html = preg_replace("/\s+|\n+|\r/", ' ', $bgm->get_bgm_items($page));
+    return new WP_REST_Response($html, 200);

@@ -370,1 +372,1 @@
-function bfv_bilibili()
+function bfv_bilibili(WP_REST_Request $request)
 {
-    $response = null;
-    if (!check_ajax_referer('wp_rest', '_wpnonce', false)) {
+    if (!sakura_verify_rest_request_nonce($request)) {
         $output = array(
             'status' => 403,
             'success' => false,
             'message' => 'Unauthorized client.'
         );
-        $response = new WP_REST_Response($output, 403);
-    } else {
-        $page = $_GET["page"] ?: 2;
-        $bgm = new \Sakura\API\Bilibili();
-        $html = preg_replace("/\s+|\n+|\r/", ' ', $bgm->get_bfv_items($page));
-        $response = new WP_REST_Response($html, 200);
+        return new WP_REST_Response($output, 403);
     }
-    return $response;
+    $page = intval($request->get_param('page')) ?: 2;
+    $bgm = new \Sakura\API\Bilibili();
+    $html = preg_replace("/\s+|\n+|\r/", ' ', $bgm->get_bfv_items($page));
+    return new WP_REST_Response($html, 200);

@@ -389,1 +391,1 @@
-function steam_library ($request)
+function steam_library(WP_REST_Request $request)
 {
-    if (!check_ajax_referer('wp_rest', '_wpnonce', false)) {
+    if (!sakura_verify_rest_request_nonce($request)) {
         $response = array(
             'status' => 418,
             'success' => false,
             'message' => 'Unauthorized client.'
         );
         return new WP_REST_Response($response, 418);
-    } else {
-        $page = $request->get_param('page') ?: 1;
-        $SteamList = new \Sakura\API\Steam();
     }
-    return $SteamList->get_steam_items((int)$page);
+    $page = intval($request->get_param('page')) ?: 1;
+    $SteamList = new \Sakura\API\Steam();
+    return $SteamList->get_steam_items($page);

@@ -408,1 +410,1 @@
-    if (!check_ajax_referer('wp_rest', '_wpnonce', false)) {
+    if (!sakura_verify_rest_request_nonce($request)) {

@@ -474,1 +476,1 @@
-    if (!check_ajax_referer('wp_rest', '_wpnonce', false)) {
+    if (!sakura_verify_rest_request_nonce($request)) {

--- a/inc/classes/gallery.php
+++ b/inc/classes/gallery.php
@@ -192,2 +192,2 @@
-    public function get_image() {
-        $imgParam = isset($_GET['img']) ? sanitize_text_field($_GET['img']) : '';
+    public function get_image(\WP_REST_Request $request) {
+        $imgParam = sanitize_text_field($request->get_param('img')) ?: '';
```

**修复要点:**
1. **新增 `sakura_verify_rest_request_nonce()` 函数** — 阻塞性修复，不定义则所有 REST API 500
2. **所有回调加入 `WP_REST_Request $request` 参数** — 符合 REST API 规范
3. **`$_GET` → `$request->get_param()`** — 禁止直接访问超全局变量
4. **`sanitize_text_field()` 净化字符串参数** — userID 等
5. **`intval()` 转换整数参数** — page 等
6. **简化控制流** — nonce 失败直接 `return`，移除 `else` 嵌套

---

### Patch 18: REST API nonce 绕过修复 — meting_aplayer

**文件:** [inc/api.php](file:///workspace/inc/api.php)
**前置依赖:** Patch 17（函数签名已改为 WP_REST_Request）

```diff
--- a/inc/api.php
+++ b/inc/api.php
@@ -527,1 +527,1 @@
-function meting_aplayer()
+function meting_aplayer(WP_REST_Request $request)
 {
-    $type = $_GET['type'];
-    $id = $_GET['id'];
-    if (in_array('_wpnonce', $_GET))
-        $wpnonce = $_GET['_wpnonce'];
-    if (in_array('meting_nonce', $_GET))
-        $meting_nonce = $_GET['meting_nonce'];
-    if ((isset($wpnonce) && !check_ajax_referer('wp_rest', $wpnonce, false)) || (isset($meting_nonce) && !wp_verify_nonce($meting_nonce, $type . '#:' . $id))) {
+    $type = sanitize_text_field($request->get_param('type'));
+    $id = sanitize_text_field($request->get_param('id'));
+    $wpnonce = sanitize_text_field($request->get_param('_wpnonce')) ?: null;
+    $meting_nonce = sanitize_text_field($request->get_param('meting_nonce')) ?: null;
+
+    // 必须提供至少一个有效 nonce，否则拒绝
+    $wpnonce_valid = $wpnonce && wp_verify_nonce($wpnonce, 'wp_rest');
+    $meting_nonce_valid = $meting_nonce && wp_verify_nonce($meting_nonce, $type . '#:' . $id);
+
+    if (!$wpnonce_valid && !$meting_nonce_valid) {
         $output = array(
             'status' => 403,
             'success' => false,
             'message' => 'Unauthorized client.'
         );
```

**逻辑修复对比:**

| 场景 | 修复前 | 修复后 |
|------|--------|--------|
| 不传任何 nonce | ✅ 放行（漏洞） | ❌ 拒绝 |
| 传无效 wpnonce | ❌ 拒绝 | ❌ 拒绝 |
| 传有效 wpnonce | ✅ 放行 | ✅ 放行 |
| 传有效 meting_nonce | ✅ 放行 | ✅ 放行 |
| 传无效 meting_nonce | ❌ 拒绝 | ❌ 拒绝 |

**修复逻辑:** `!$wpnonce_valid && !$meting_nonce_valid` — 两个 nonce 都无效才拒绝（OR 逻辑），即至少一个有效就放行。

---

### Patch 20: 输入验证 + 重定向安全 — get_qq_avatar

**文件:** [inc/api.php](file:///workspace/inc/api.php)
**前置依赖:** Patch 17（函数签名已改为 WP_REST_Request）

```diff
--- a/inc/api.php
+++ b/inc/api.php
@@ -312,8 +312,15 @@
-function get_qq_avatar()
+function get_qq_avatar(WP_REST_Request $request)
 {
-    $encrypted = $_GET["qq"];
+    $encrypted = sanitize_text_field($request->get_param('qq'));
+    if (empty($encrypted)) {
+        return new WP_Error('rest_qq_missing', 'Missing qq parameter', array('status' => 400));
+    }
     $imgurl = QQ::get_qq_avatar($encrypted);
+    if (!$imgurl) {
+        return new WP_Error('rest_qq_avatar_not_found', 'Avatar not found', array('status' => 404));
+    }
     if (iro_opt('qq_avatar_link') == 'type_2') {
-        $imgdata = file_get_contents($imgurl);
+        $remote = wp_remote_get(esc_url_raw($imgurl));
+        if (is_wp_error($remote) || wp_remote_retrieve_response_code($remote) !== 200) {
+            return new WP_Error('rest_qq_avatar_fetch_failed', 'Failed to fetch avatar', array('status' => 500));
+        }
+        $imgdata = wp_remote_retrieve_body($remote);
+        // 二进制数据必须绕过 REST 框架直接输出
+        header('Content-Type: image/jpeg');
+        header('Cache-Control: max-age=86400');
+        echo $imgdata;
+        exit;
     } else {
         $response = new WP_REST_Response();
-        $response->set_status(301);
-        $response->header('Location', $imgurl);
+        $response->set_status(302);
+        $response->header('Location', esc_url_raw($imgurl));
     }
     return $response;
 }
```

**修复要点:**

1. **输入净化:** `sanitize_text_field($request->get_param('qq'))` 替代 `$_GET["qq"]`
2. **错误响应改用 `WP_Error`:** 符合 WordPress skills "Return `WP_Error` with a `status` in `data`"
3. **`file_get_contents` → `wp_remote_get`:** SSRF 修复
4. **type_2 二进制直接输出:** `header()` + `echo` + `exit` 绕过 REST 框架
   - REST 框架会对返回值进行 JSON 编码，二进制图片数据会被损坏
   - 必须直接设置 header 并输出后 `exit`
5. **301 → 302:** 302 临时重定向语义正确（头像 URL 可能变化）
6. **`esc_url_raw($imgurl)`:** 重定向 URL 净化

---

## 审查结论

| Patch | WP_REST_Request | get_param | sanitize | WP_Error | 符合最佳实践 |
|-------|----------------|-----------|----------|----------|-------------|
| 17 | ✅ | ✅ | ✅ | N/A | ✅ |
| 18 | ✅ | ✅ | ✅ | N/A | ✅ |
| 20 | ✅ | ✅ | ✅ | ✅ | ✅ |

**改进建议（非阻塞）:**
- 建议在 `register_rest_route()` 中添加 `args` schema 定义，实现参数自动验证
- 建议将 nonce 验证移至 `permission_callback`，而非在回调中手动验证
- `sakura_verify_rest_request_nonce()` 可扩展支持 `X-WP-Nonce` header

**关键教训:**
- REST 回调中禁止直接访问 `$_GET`/`$_POST`，必须用 `$request->get_param()`
- nonce 验证逻辑必须是"必须提供有效 nonce"，不能是"传了才验证"
- 二进制数据（图片）必须绕过 REST 框架直接输出
- 错误响应应使用 `WP_Error` 而非 `WP_REST_Response`
