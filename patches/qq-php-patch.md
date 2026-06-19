# QQ.php 修复 Patch 文档

> 涉及文件: inc/classes/QQ.php, inc/api.php, functions.php
> 审查依据: WordPress Skills 最佳实践
> 状态: 待审核，未应用

---

## Patch A: inc/classes/QQ.php — 完整重写

**修复问题:**
1. 🔴 SSRF: `file_get_contents()` → `wp_remote_get()`
2. 🔴 加解密 IV 不匹配：固定 IV → 从密文提取 IV
3. 🔴 `$sakura_privkey` 无 isset 检查
4. 🔴 `preg_match` 失败后访问 `$matches[0]`
5. 🟠 `$output` 逻辑分支不完整
6. 🟠 QQ 号未 `urlencode()`
7. 🟠 `$name['code']`/`$name['name']` 无 isset 检查

```diff
--- a/inc/classes/QQ.php
+++ b/inc/classes/QQ.php
@@ -1,40 +1,85 @@
 <?php

 namespace Sakura\API;

 class QQ
 {
-    public static function get_qq_info($qq) {
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
-        } else {
-            $output = array(
-                'status' => 404,
-                'success' => false,
-                'message' => 'QQ number not exist.'
-            );
-        }
-        return $output;
+    /**
+     * Get QQ user info from external API.
+     *
+     * @param string $qq QQ number (digits only, 3+ chars)
+     * @return array{status: int, success: bool, message: string, avatar?: string, name?: string}
+     */
+    public static function get_qq_info($qq) {
+        // 输入验证
+        $qq = sanitize_text_field($qq);
+        if (empty($qq) || !preg_match('/^\d{3,}$/', $qq)) {
+            return array(
+                'status' => 400,
+                'success' => false,
+                'message' => 'Invalid QQ number format.'
+            );
+        }
+
+        // 使用 WP HTTP API 替代 file_get_contents（修复 SSRF）
+        $response = wp_remote_get(
+            'https://api.qjqq.cn/api/qqinfo?qq=' . urlencode($qq),
+            array('timeout' => 5)
+        );
+
+        // 错误检查
+        if (is_wp_error($response) || wp_remote_retrieve_response_code($response) !== 200) {
+            return array(
+                'status' => 404,
+                'success' => false,
+                'message' => 'QQ number not exist.'
+            );
+        }
+
+        // JSON 解析 + 类型验证
+        $name = json_decode(wp_remote_retrieve_body($response), true);
+        if (!is_array($name) || !isset($name['code']) || $name['code'] != 200) {
+            return array(
+                'status' => 404,
+                'success' => false,
+                'message' => 'QQ number not exist.'
+            );
+        }
+
+        return array(
+            'status' => 200,
+            'success' => true,
+            'message' => 'success',
+            'avatar' => 'https://q2.qlogo.cn/headimg_dl?dst_uin=' . urlencode($qq) . '&spec=100',
+            'name' => isset($name['name']) ? $name['name'] : '',
+        );
     }

-    public static function get_qq_avatar($encrypted) {
-        global $sakura_privkey;
-        if (isset($encrypted)) {
-            $iv = str_repeat($sakura_privkey, 2);
-            $encrypted = base64_decode(urldecode($encrypted));
-            $qq_number = openssl_decrypt($encrypted, 'aes-128-cbc', $sakura_privkey, 0, $iv);
-            preg_match('/^\d{3,}$/', $qq_number, $matches);
-            return 'https://q2.qlogo.cn/headimg_dl?dst_uin=' . $matches[0] . '&spec=100';
-        }
+    /**
+     * Decrypt QQ number from encrypted string and return avatar URL.
+     *
+     * Encryption format (from change_avatar): base64( IV(16bytes) + AES-CBC-ciphertext )
+     *
+     * @param string $encrypted Base64-encoded encrypted QQ number (IV + ciphertext)
+     * @return string|false Avatar URL on success, false on failure
+     */
+    public static function get_qq_avatar($encrypted) {
+        global $sakura_privkey;
+
+        // 密钥验证
+        if (!isset($sakura_privkey) || empty($sakura_privkey)) {
+            return false;
+        }
+
+        // 输入验证
+        if (empty($encrypted)) {
+            return false;
+        }
+
+        // Base64 解码
+        $decoded = base64_decode(urldecode($encrypted));
+        if ($decoded === false) {
+            return false;
+        }
+
+        // IV 提取（与加密端一致：IV 拼接在密文前）
+        $iv_length = openssl_cipher_iv_length('aes-128-cbc');
+        if (strlen($decoded) <= $iv_length) {
+            return false;
+        }
+        $iv = substr($decoded, 0, $iv_length);
+        $ciphertext = substr($decoded, $iv_length);
+
+        // 解密
+        $qq_number = openssl_decrypt($ciphertext, 'aes-128-cbc', $sakura_privkey, 0, $iv);
+        if ($qq_number === false) {
+            return false;
+        }
+
+        // QQ 号格式验证
+        if (!preg_match('/^\d{3,}$/', $qq_number)) {
+            return false;
+        }
+
+        return 'https://q2.qlogo.cn/headimg_dl?dst_uin=' . urlencode($qq_number) . '&spec=100';
     }
 }
```

---

## Patch B: inc/api.php — 路由注册 + get_qq_info + get_qq_avatar

**修复问题:**
1. 🟠 REST 路由无 args schema
2. 🟠 REST 回调直接访问 `$_GET`
3. 🟠 `get_qq_avatar` 无 `WP_REST_Request` 参数
4. 🔴 `file_get_contents($imgurl)` SSRF
5. 🟠 type_2 二进制输出通过 REST 框架损坏
6. 🟠 301 → 302 语义修正
7. 🟠 错误响应应使用 `WP_Error`
8. 🟠 `Location` header 未 `esc_url_raw`

### B-1: 路由注册（api.php:57-68）

```diff
--- a/inc/api.php
+++ b/inc/api.php
@@ -57,12 +57,24 @@
     register_rest_route('sakura/v1', '/qqinfo/json', array(
         'methods' => 'GET',
         'callback' => 'get_qq_info',
-        'permission_callback' => '__return_true'
+        'permission_callback' => '__return_true',
+        'args' => array(
+            'qq' => array(
+                'type' => 'string',
+                'required' => true,
+                'sanitize_callback' => 'sanitize_text_field',
+            ),
+        ),
     )
     );
     register_rest_route('sakura/v1', '/qqinfo/avatar', array(
         'methods' => 'GET',
         'callback' => 'get_qq_avatar',
-        'permission_callback' => '__return_true'
+        'permission_callback' => '__return_true',
+        'args' => array(
+            'qq' => array(
+                'type' => 'string',
+                'required' => true,
+                'sanitize_callback' => 'sanitize_text_field',
+            ),
+        ),
     )
     );
```

### B-2: get_qq_info 回调（api.php:284-306）

```diff
--- a/inc/api.php
+++ b/inc/api.php
@@ -284,18 +284,20 @@
 function get_qq_info(WP_REST_Request $request)
 {
-    if (!check_ajax_referer('wp_rest', '_wpnonce', false)) {
+    if (!sakura_verify_rest_request_nonce($request)) {
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

     $result = new WP_REST_Response($output, $output['status']);
```

### B-3: get_qq_avatar 回调（api.php:312-332）

```diff
--- a/inc/api.php
+++ b/inc/api.php
@@ -312,21 +312,27 @@
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
-        $response = new WP_REST_Response();
-        $response->set_headers(
-            array(
-                'Content-Type' => 'image/jpeg',
-                'Cache-Control' => 'max-age=86400'
-            )
-        );
-        echo $imgdata;
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

---

## Patch C: functions.php — change_avatar 函数

**修复问题:**
1. 🟠 `$qq_number` 未 `sanitize_text_field`
2. 🟠 QQ 号拼接到 URL 未 `urlencode` + 未 `esc_attr`
3. 🔴 `file_get_contents('http://...')` SSRF + 明文
4. 🟠 `$matches[1]` 未 isset 检查 + 未 `esc_url`
5. 🟠 `rest_url()` + `$encrypted` 输出未 `esc_url`
6. 🟠 密钥不存在时回退 `default_avatar_url` 硬编码 → 应回退原 `$avatar`

```diff
--- a/functions.php
+++ b/functions.php
@@ -2418,12 +2418,15 @@
     global $comment, $sakura_privkey;
     if ($comment && get_comment_meta($comment->comment_ID, 'new_field_qq', true)) {
-        $qq_number = get_comment_meta($comment->comment_ID, 'new_field_qq', true);
+        $qq_number = sanitize_text_field(get_comment_meta($comment->comment_ID, 'new_field_qq', true));
         if (iro_opt('qq_avatar_link') == 'off') {
-            return '<img src="https://q2.qlogo.cn/headimg_dl?dst_uin=' . $qq_number . '&spec=100" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
+            return '<img src="https://q2.qlogo.cn/headimg_dl?dst_uin=' . esc_attr(urlencode($qq_number)) . '&spec=100" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
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
        
         // Ensure $sakura_privkey is defined and not null
@@ -2442,1 +2445,1 @@
-            return '<img src="' . rest_url("sakura/v1/qqinfo/avatar") . '?qq=' . $encrypted . '" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
+            return '<img src="' . esc_url(rest_url("sakura/v1/qqinfo/avatar") . '?qq=' . $encrypted) . '" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
         } else {
-            // Handle the case where $sakura_privkey is not set or is null
-            return '<img src="default_avatar_url" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
+            return $avatar;
         }
     }
```

---

## 修复问题与 Patch 映射

| # | 问题 | 严重程度 | Patch | 文件 |
|---|------|---------|-------|------|
| 1 | SSRF: file_get_contents | 🔴 | A, B-3, C | QQ.php, api.php, functions.php |
| 2 | 加解密 IV 不匹配 | 🔴 | A | QQ.php |
| 3 | $sakura_privkey 无 isset | 🔴 | A | QQ.php |
| 4 | $matches[0] 未定义索引 | 🔴 | A | QQ.php |
| 5 | $output 逻辑分支不完整 | 🟠 | A | QQ.php |
| 6 | QQ 号未 urlencode | 🟠 | A, C | QQ.php, functions.php |
| 7 | REST 直接访问 $_GET | 🟠 | B-2, B-3 | api.php |
| 8 | REST 路由无 args schema | 🟠 | B-1 | api.php |
| 9 | type_2 二进制输出损坏 | 🟠 | B-3 | api.php |
| 10 | 301 → 302 语义 | 🟠 | B-3 | api.php |
| 11 | 错误响应应用 WP_Error | 🟠 | B-3 | api.php |
| 12 | Location 未 esc_url_raw | 🟠 | B-3 | api.php |
| 13 | XSS: QQ 号输出未转义 | 🟡 | C | functions.php |
| 14 | $matches[1] 无 isset | 🟡 | C | functions.php |
| 15 | 密钥不存在回退硬编码 URL | 🟡 | C | functions.php |

---

## 前置依赖

- Patch B-2/B-3 依赖 `sakura_verify_rest_request_nonce()` 函数已定义（见 Patch 17）
- 如果 Patch 17 尚未应用，需先在 api.php 顶部添加该函数定义

## 应用顺序

1. **Patch A** — QQ.php（核心修复，无依赖）
2. **Patch B-1** — 路由注册（无依赖）
3. **Patch B-2** — get_qq_info 回调（依赖 sakura_verify_rest_request_nonce）
4. **Patch B-3** — get_qq_avatar 回调（依赖 Patch A 的 QQ::get_qq_avatar 返回 false）
5. **Patch C** — change_avatar（无依赖）
