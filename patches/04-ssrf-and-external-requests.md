# 类别 4: SSRF 与外部请求安全

> 包含 Patch: 15-16
> 审查依据: wp-plugin-development/references/security.md, wp-plugin-development/references/data-and-cron.md
> 风险等级: 严重（SSRF / 功能失效）

---

## WordPress Skills 最佳实践

来自 `wp-plugin-development/references/data-and-cron.md`:

> **Database safety note:** If using `$wpdb->prepare()`, avoid building queries with concatenated user input.

虽然此条针对 SQL，但同样适用于 HTTP 请求——**避免将用户输入直接拼接到 URL 中**。

WordPress HTTP API 最佳实践：

| 函数 | 用途 | 安全特性 |
|------|------|---------|
| `wp_remote_get()` | GET 请求 | 支持 timeout、redirection 限制 |
| `wp_remote_post()` | POST 请求 | 同上 |
| `wp_remote_retrieve_body()` | 提取 body | 安全提取，避免直接数组访问 |
| `wp_remote_retrieve_response_code()` | 提取状态码 | 安全提取 |
| `is_wp_error()` | 错误检查 | 必须检查，WP_Error 不可 ArrayAccess |

**关键原则:**
1. **禁止使用 `file_get_contents()` 发起 HTTP 请求** — 无超时控制、无错误处理、可被 SSRF 利用
2. **必须检查 `is_wp_error()`** — `wp_remote_*` 失败返回 `WP_Error`，直接访问 `$response['body']` 会致命错误
3. **用户输入必须 `urlencode()`** — 拼接到 URL 前必须编码
4. **必须设置 timeout** — 防止恶意 URL 导致 DoS

---

## 主题存在的问题分析

### 问题 1: file_get_contents 发起 HTTP 请求（SSRF）

主题中多处使用 `file_get_contents()` 发起外部 HTTP 请求：

| 文件 | 行号 | URL 来源 | 风险 |
|------|------|---------|------|
| `functions.php` | 2420 | `http://ptlogin2.qq.com/...?uin=' . $qq_number` | QQ 号来自评论 meta，可注入 URL 参数 |
| `inc/article-highlight.php` | 234 | `$input`（用户可控 URL） | 完全用户可控，SSRF |
| `inc/classes/QQ.php` | 7 | `https://api.qjqq.cn/...?qq=' . $qq` | QQ 号来自用户输入 |
| `inc/api.php` | 312 | `$imgurl`（来自 QQ::get_qq_avatar） | 间接用户可控 |

**`file_get_contents()` 的危害:**
1. **无超时控制** — 恶意 URL 可导致请求挂起，DoS
2. **无协议限制** — `file_get_contents('file:///etc/passwd')` 可读取本地文件
3. **无错误处理** — 失败返回 `false`，后续代码可能出错
4. **HTTP 明文** — `http://` 无加密，可被中间人攻击

### 问题 2: QQ 加解密 IV 不匹配（功能完全失效）

`inc/classes/QQ.php` 的 `get_qq_avatar()` 存在长期 Bug：

**加密端** (`functions.php` change_avatar type_2):
```php
$iv = openssl_random_pseudo_bytes(16);  // 随机 IV
$encrypted = openssl_encrypt($qq_number, 'aes-128-cbc', $sakura_privkey, 0, $iv);
$encrypted = base64_encode($iv . $encrypted);  // IV 拼接在密文前
```

**解密端** (`QQ.php` get_qq_avatar — 有 Bug):
```php
$iv = str_repeat($sakura_privkey, 2);  // 固定 IV（错误！）
$encrypted = base64_decode(urldecode($encrypted));
$qq_number = openssl_decrypt($encrypted, 'aes-128-cbc', $sakura_privkey, 0, $iv);
```

**问题:** 加密端用随机 IV，解密端用固定 IV → 解密永远失败 → QQ 头像 type_2 功能完全失效。

### 问题 3: wp_remote_post 未检查 is_wp_error

`inc/classes/Images.php` 中 `Chevereto_API()` 和 `Imgur_API()` 调用 `wp_remote_post()` 后直接 `$response['body']`，未检查 `is_wp_error()`。如果请求失败（网络错误、DNS 解析失败等），`WP_Error` 对象不支持数组访问，将触发 Fatal Error。

> 注: 此问题在类型安全类别（Patch 22）中修复，此处仅记录关联。

---

## Patch 清单

### Patch 15: SSRF 修复 — file_get_contents → wp_remote_get

**文件:** [functions.php](file:///workspace/functions.php), [inc/article-highlight.php](file:///workspace/inc/article-highlight.php)

```diff
--- a/functions.php
+++ b/functions.php
@@ -2420,1 +2420,1 @@
         $qq_number = get_comment_meta($comment->comment_ID, 'new_field_qq', true);
+        $qq_number = sanitize_text_field($qq_number);
         if (iro_opt('qq_avatar_link') == 'off') {
-            return '<img src="https://q2.qlogo.cn/headimg_dl?dst_uin=' . $qq_number . '&spec=100" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
+            return '<img src="https://q2.qlogo.cn/headimg_dl?dst_uin=' . esc_attr($qq_number) . '&spec=100" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
         }
         if (iro_opt('qq_avatar_link') == 'type_3') {
-            $qqavatar = file_get_contents('http://ptlogin2.qq.com/getface?appid=1006102&imgtype=3&uin=' . $qq_number);
+            $qqavatar = wp_remote_retrieve_body(wp_remote_get('https://ptlogin2.qq.com/getface?appid=1006102&imgtype=3&uin=' . urlencode($qq_number)));
             preg_match('/:\"([^\"]*)\"/i', $qqavatar, $matches);
-            return '<img src="' . $matches[1] . '" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
+            $avatar_url = isset($matches[1]) ? esc_url($matches[1]) : '';
+            if (empty($avatar_url)) {
+                return $avatar;
+            }
+            return '<img src="' . $avatar_url . '" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
         }

@@ -2440,1 +2440,1 @@
-            return '<img src="' . rest_url("sakura/v1/qqinfo/avatar") . '?qq=' . $encrypted . '" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
+            return '<img src="' . esc_url(rest_url("sakura/v1/qqinfo/avatar") . '?qq=' . $encrypted) . '" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
         } else {
-            return '<img src="default_avatar_url" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
+            return $avatar;
         }

--- a/inc/article-highlight.php
+++ b/inc/article-highlight.php
@@ -234,4 +234,5 @@
         error_log('编码结果为' . $input);
-        if (function_exists('wp_get_remote_content')) {
-            $image_data = wp_get_remote_content($input);
-        } else {
-            $image_data = file_get_contents($input);
+        $remote_response = wp_remote_get(esc_url_raw($input), array('timeout' => 10));
+        if (is_wp_error($remote_response) || wp_remote_retrieve_response_code($remote_response) !== 200) {
+            error_log('远程获取图片失败');
+            return false;
         }
+        $image_data = wp_remote_retrieve_body($remote_response);
```

**修复要点:**
1. `file_get_contents('http://...')` → `wp_remote_get('https://...')` — 升级为 HTTPS + WP HTTP API
2. `urlencode($qq_number)` — 防止 URL 参数注入
3. `isset($matches[1])` — 防止正则未匹配时未定义索引
4. `esc_url($matches[1])` — 过滤返回的 URL
5. `esc_url_raw($input)` — 请求 URL 净化
6. `is_wp_error()` + 状态码检查 — 完整错误处理
7. `'timeout' => 10` — 防止 DoS

---

### Patch 16: SSRF 修复 + 加解密 IV 不匹配 — QQ.php

**文件:** [inc/classes/QQ.php](file:///workspace/inc/classes/QQ.php)

```diff
--- a/inc/classes/QQ.php
+++ b/inc/classes/QQ.php
@@ -7,12 +7,17 @@
     public static function get_qq_info($qq) {
-        $get_info = file_get_contents('https://api.qjqq.cn/api/qqinfo?qq=' . $qq);
-        $name = json_decode($get_info, true);
-        if ($name) {
-            if ($name['code'] == 200){
-                $output = array(
-                    'status' => 200,
-                    'success' => true,
-                    'message' => 'success',
-                    'avatar' => 'https://q2.qlogo.cn/headimg_dl?dst_uin=' . $qq . '&spec=100',
-                    'name' => $name['name'],
-                );
-            }
+        $response = wp_remote_get('https://api.qjqq.cn/api/qqinfo?qq=' . urlencode($qq), array('timeout' => 5));
+        if (is_wp_error($response) || wp_remote_retrieve_response_code($response) !== 200) {
+            return array(
+                'status' => 404,
+                'success' => false,
+                'message' => 'QQ number not exist.'
+            );
+        }
+        $name = json_decode(wp_remote_retrieve_body($response), true);
+        if ($name && isset($name['code']) && $name['code'] == 200) {
+            $output = array(
+                'status' => 200,
+                'success' => true,
+                'message' => 'success',
+                'avatar' => 'https://q2.qlogo.cn/headimg_dl?dst_uin=' . $qq . '&spec=100',
+                'name' => isset($name['name']) ? $name['name'] : '',
+            );
         } else {

@@ -30,9 +35,33 @@
     public static function get_qq_avatar($encrypted) {
         global $sakura_privkey;
-        if (isset($encrypted)) {
-            $iv = str_repeat($sakura_privkey, 2);
-            $encrypted = base64_decode(urldecode($encrypted));
-            $qq_number = openssl_decrypt($encrypted, 'aes-128-cbc', $sakura_privkey, 0, $iv);
-            preg_match('/^\d{3,}$/', $qq_number, $matches);
-            return 'https://q2.qlogo.cn/headimg_dl?dst_uin=' . $matches[0] . '&spec=100';
-        }
+        if (!isset($sakura_privkey) || empty($encrypted)) {
+            return false;
+        }
+
+        $decoded = base64_decode(urldecode($encrypted));
+        if ($decoded === false) {
+            return false;
+        }
+
+        $iv_length = openssl_cipher_iv_length('aes-128-cbc');
+        if (strlen($decoded) <= $iv_length) {
+            return false;
+        }
+
+        // 提取前 16 字节作为 IV，与 change_avatar 加密端一致
+        $iv = substr($decoded, 0, $iv_length);
+        $ciphertext = substr($decoded, $iv_length);
+
+        $qq_number = openssl_decrypt($ciphertext, 'aes-128-cbc', $sakura_privkey, 0, $iv);
+        if ($qq_number === false) {
+            return false;
+        }
+
+        if (!preg_match('/^\d{3,}$/', $qq_number)) {
+            return false;
+        }
+
+        return 'https://q2.qlogo.cn/headimg_dl?dst_uin=' . $qq_number . '&spec=100';
     }
```

**get_qq_info() 修复:**
1. `file_get_contents` → `wp_remote_get` + 5s 超时
2. `is_wp_error()` 检查
3. `urlencode($qq)` 防止 URL 注入
4. `isset($name['code'])` 防止未定义索引

**get_qq_avatar() IV 修复（关键 Bug）:**

| 步骤 | 修复前（Bug） | 修复后 |
|------|-------------|--------|
| IV 来源 | `str_repeat($sakura_privkey, 2)` 固定值 | `substr($decoded, 0, 16)` 从密文提取 |
| 密文 | 整个 base64_decode 结果 | `substr($decoded, 16)` 跳过 IV |
| 解密 | 永远失败（IV 不匹配） | 成功（IV 与加密端一致） |

**额外安全加固:**
1. `!isset($sakura_privkey)` — 密钥空值检查
2. `empty($encrypted)` — 输入空值检查
3. `strlen($decoded) <= $iv_length` — 密文长度验证
4. `$qq_number === false` — 解密失败检查
5. `preg_match('/^\d{3,}$/', $qq_number)` — QQ 号格式验证（纯数字，至少 3 位）

---

## 审查结论

| Patch | wp_remote_get | is_wp_error | urlencode | esc_url | 符合最佳实践 |
|-------|--------------|------------|-----------|---------|-------------|
| 15 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 16 | ✅ | ✅ | ✅ | N/A | ✅ |

**关键教训:**
- `file_get_contents()` 绝不能用于 HTTP 请求——无超时、无协议限制、无错误处理
- `wp_remote_get()` 返回 `WP_Error|array`，必须先 `is_wp_error()` 检查
- 用户输入拼接到 URL 前必须 `urlencode()`
- AES-CBC 加解密必须确保 IV 一致——加密端随机 IV 时，解密端必须从密文中提取 IV
