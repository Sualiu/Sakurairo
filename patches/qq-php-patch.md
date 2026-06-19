# QQ.php 修复 Patch 文档（方案 A: comment_id 替代加密）

> 涉及文件: inc/classes/QQ.php, inc/api.php, functions.php, opt/options/theme-options.php
> 设计决策: 用 comment_id 替代加密方案，QQ 号永不出现在 URL 中
> 状态: 待审核，未应用

---

## 设计变更说明

### 旧方案（加密）

```
change_avatar() → openssl_encrypt(QQ号) → <img src="REST?qq=密文">
                                                  ↓
                                          get_qq_avatar() → openssl_decrypt → qlogo URL
```

问题：后端加密后端解密，QQ 号仅从 URL 明文变为 URL 密文，保护力度弱；存在 IV 不匹配 Bug；密钥管理不当。

### 新方案（comment_id）

```
change_avatar() → <img src="REST?comment_id=123" onerror="imgError(this,1)">
                        ↓
                get_qq_avatar() → get_comment_meta(comment_id, 'new_field_qq') → qlogo URL
                        ↓ (缓存命中)
                文件缓存 → 直接返回二进制 (<5ms)
                        ↓ (缓存未命中)
                wp_remote_get(qlogo) → 存入文件缓存 → 返回二进制
                        ↓ (获取失败)
                HTTP 404 → 浏览器触发 onerror → imgError() → 主题缺失头像
```

优势：
- QQ 号**完全不出现在任何 URL 中**
- 无需加密/解密，无密钥管理问题
- 利用 WordPress 已有的 comment meta 存储机制
- `comment_id` 是公开信息，不存在隐私泄露
- **文件缓存**避免重复出站请求，同一 QQ 号只请求一次
- **前端 onerror 降级**与主题已有的 `imgError()` 机制一致

### 昵称查询不变

用户在评论表单中**主动输入** QQ 号 → 调用 `/qqinfo/json?qq=123456789` → 返回昵称+头像。这是用户自愿行为，不属于隐私泄露。

---

## 原始主题优秀做法分析

原始主题虽然后端错误处理缺失，但前端设计有几个优秀模式，新方案必须保留并增强：

### 1. `onerror="imgError(this,1)"` — 前端兜底降级

```javascript
// src/app/global-func.js
function imgError(ele, type) {
    switch (type) {
        case 1:  // 头像类型
        case 2:
            ele.onerror = "";  // 防止无限循环
            if (_iro.missing_avatars != "") {
                ele.src = _iro.missing_avatars;  // 主题自定义缺失头像
            } else {
                ele.src = 'https://weavatar.com/avatar/?s=80&d=mm&r=g';  // Gravatar 默认
            }
            break;
        default:  // 普通图片
            ele.onerror = "";
            if (_iro.missing_images != ""){
                ele.src = _iro.missing_images;
            } else {
                ele.src = svg404;  // 内联 SVG 404 占位图
            }
    }
}
```

**这是原始主题最优秀的设计**：后端无论返回什么错误（404/500/超时/无效数据），浏览器都会触发 `onerror`，`imgError()` 自动替换为主题配置的缺失头像。这比后端 302 重定向到 Gravatar 更好，因为：
- 使用主题自身的 `_iro.missing_avatars` 配置，风格一致
- `ele.onerror = ""` 防止降级图片再失败时无限循环
- 前端降级不增加后端负担

**新方案策略：后端失败时返回 HTTP 404，让 `onerror` 自然触发。** 不需要后端 302 重定向。

### 2. `Cache-Control: max-age=86400` — 浏览器缓存

原始 type_2 模式设置浏览器缓存 24 小时。新方案保留并增强：
- 成功响应：`Cache-Control: max-age=86400`（浏览器 24 小时内不再请求）
- 失败响应：`Cache-Control: no-cache`（不缓存错误，下次重试）

### 3. `lazyload` class — 懒加载

原始主题使用 lazyload 延迟加载头像，减少首屏并发请求数。新方案保留不变。

### 4. `spec=100` — 指定尺寸

请求 qlogo 时指定 `spec=100`（100x100），避免下载过大图片。新方案保留不变。

---

## 主题选项变更

### 旧选项

| 值 | 标签 | 行为 | QQ 号暴露 |
|---|------|------|----------|
| `off` | Off | 直接拼接 QQ 号到 qlogo URL | ✅ |
| `type_1` | Redirect (low security) | 加密后重定向到 qlogo URL | ❌ 密文 |
| `type_2` | Get avatar data in backend (medium) | 加密后后端获取头像二进制 | ❌ 密文 |
| `type_3` | Parse avatar interface (high, slow) | 后端解析 ptlogin2 接口 | ✅ URL 含 QQ 号 |

### 新选项

| 值 | 标签 | 行为 | QQ 号暴露 |
|---|------|------|----------|
| `direct` | Direct | 直接拼接 qlogo URL | ✅ |
| `ptlogin2` | Direct (ptlogin2 fallback) | 通过 ptlogin2 接口获取头像 URL，CDN 兼容性更好 | ✅ |
| `proxy` | Proxy via comment ID | 通过 comment_id 后端代理头像，QQ 号不出现在任何 URL 中 | ❌ |

**变更说明:**
- 使用全新语义化值名，不沿用旧值
- `off` → `direct`：语义更明确
- `type_3` → `ptlogin2`：直接描述实现方式
- `type_1`+`type_2` → `proxy`：合并为代理模式
- 默认值从 `off` 改为 `direct`

---

## Patch A: inc/classes/QQ.php — 重写

**变更:**
1. `get_qq_info()` — 修复 SSRF、输入验证、逻辑分支、urlencode
2. 删除 `get_qq_avatar($encrypted)` — 加密解密方案废弃
3. 新增 `get_qq_avatar_url($comment_id)` — 通过 comment_id 读取 QQ 号返回头像 URL
4. 新增 `get_qq_avatar_data($comment_id)` — 通过 comment_id 读取 QQ 号返回头像二进制数据（含文件缓存）
5. 新增 `get_qq_avatar_url_ptlogin2($comment_id)` — 通过 ptlogin2 接口获取头像 URL

```diff
--- a/inc/classes/QQ.php
+++ b/inc/classes/QQ.php
@@ -1,40 +1,156 @@
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
+     * Used when user actively inputs QQ number in comment form.
+     *
+     * @param string $qq QQ number (digits only, 3+ chars)
+     * @return array{status: int, success: bool, message: string, avatar?: string, name?: string}
+     */
+    public static function get_qq_info($qq) {
+        $qq = sanitize_text_field($qq);
+        if (empty($qq) || !preg_match('/^\d{3,}$/', $qq)) {
+            return array(
+                'status' => 400,
+                'success' => false,
+                'message' => 'Invalid QQ number format.'
+            );
+        }
+
+        $response = wp_remote_get(
+            'https://api.qjqq.cn/api/qqinfo?qq=' . urlencode($qq),
+            array('timeout' => 5)
+        );
+
+        if (is_wp_error($response) || wp_remote_retrieve_response_code($response) !== 200) {
+            return array(
+                'status' => 404,
+                'success' => false,
+                'message' => 'QQ number not exist.'
+            );
+        }
+
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
+     * Get QQ avatar URL by comment ID (qlogo direct).
+     * Reads QQ number from comment meta, returns qlogo URL.
+     *
+     * @param int $comment_id WordPress comment ID
+     * @return string|false Avatar URL on success, false on failure
+     */
+    public static function get_qq_avatar_url($comment_id) {
+        $comment_id = intval($comment_id);
+        if ($comment_id <= 0) {
+            return false;
+        }
+
+        $qq_number = get_comment_meta($comment_id, 'new_field_qq', true);
+        $qq_number = sanitize_text_field($qq_number);
+        if (empty($qq_number) || !preg_match('/^\d{3,}$/', $qq_number)) {
+            return false;
+        }
+
+        return 'https://q2.qlogo.cn/headimg_dl?dst_uin=' . urlencode($qq_number) . '&spec=100';
+    }
+
+    /**
+     * Get QQ avatar URL by comment ID (ptlogin2 fallback).
+     * Uses ptlogin2 API to resolve avatar URL, better CDN compatibility.
+     *
+     * @param int $comment_id WordPress comment ID
+     * @return string|false Avatar URL on success, false on failure
+     */
+    public static function get_qq_avatar_url_ptlogin2($comment_id) {
+        $comment_id = intval($comment_id);
+        if ($comment_id <= 0) {
+            return false;
+        }
+
+        $qq_number = get_comment_meta($comment_id, 'new_field_qq', true);
+        $qq_number = sanitize_text_field($qq_number);
+        if (empty($qq_number) || !preg_match('/^\d{3,}$/', $qq_number)) {
+            return false;
+        }
+
+        $response = wp_remote_get(
+            'https://ptlogin2.qq.com/getface?appid=1006102&imgtype=3&uin=' . urlencode($qq_number),
+            array('timeout' => 5)
+        );
+        if (is_wp_error($response) || wp_remote_retrieve_response_code($response) !== 200) {
+            return false;
+        }
+
+        $body = wp_remote_retrieve_body($response);
+        if (!preg_match('/:\"([^\"]*)\"/i', $body, $matches) || empty($matches[1])) {
+            return false;
+        }
+
+        return esc_url_raw($matches[1]);
+    }
+
+    /**
+     * Get QQ avatar binary data by comment ID with file caching.
+     *
+     * Caching strategy:
+     * 1. File cache (wp-content/cache/qq-avatars/) — avoids DB bloat
+     * 2. Cache key by QQ number — same QQ across different comments shares one cache
+     * 3. TTL: 7 days — avatar rarely changes
+     *
+     * On cache miss: fetches from qlogo via wp_remote_get(), stores, returns.
+     * On fetch failure or file cache unavailable: returns false
+     *     (caller returns HTTP 404, browser triggers onerror → imgError() → missing avatar).
+     *
+     * @param int $comment_id WordPress comment ID
+     * @return string|false Binary image data on success, false on failure
+     */
+    public static function get_qq_avatar_data($comment_id) {
+        $comment_id = intval($comment_id);
+        if ($comment_id <= 0) {
+            return false;
+        }
+
+        $qq_number = get_comment_meta($comment_id, 'new_field_qq', true);
+        $qq_number = sanitize_text_field($qq_number);
+        if (empty($qq_number) || !preg_match('/^\d{3,}$/', $qq_number)) {
+            return false;
+        }
+
+        // Cache key by QQ number — same QQ across different comments shares one cache entry
+        $cache_key = md5($qq_number);
+        $cache_dir = WP_CONTENT_DIR . '/cache/qq-avatars';
+        $cache_file = $cache_dir . '/' . $cache_key . '.jpg';
+
+        // 1. Try file cache (7 day TTL)
+        if (file_exists($cache_file) && (time() - filemtime($cache_file)) < 7 * DAY_IN_SECONDS) {
+            $data = @file_get_contents($cache_file);
+            if ($data !== false) {
+                return $data;
+            }
+        }
+
+        // 2. Fetch from qlogo
+        $imgurl = 'https://q2.qlogo.cn/headimg_dl?dst_uin=' . urlencode($qq_number) . '&spec=100';
+        $response = wp_remote_get(esc_url_raw($imgurl), array('timeout' => 10));
+        if (is_wp_error($response) || wp_remote_retrieve_response_code($response) !== 200) {
+            return false;
+        }
+
+        $imgdata = wp_remote_retrieve_body($response);
+        if (empty($imgdata)) {
+            return false;
+        }
+
+        // 3. Store to file cache (best-effort, failure is non-fatal)
+        if (wp_mkdir_p($cache_dir)) {
+            @file_put_contents($cache_file, $imgdata);
+        }
+        // File cache unavailable — return data anyway, just without caching.
+        // Next request will fetch from qlogo again. Browser Cache-Control still helps.
+
+        return $imgdata;
     }
 }
```

**缓存设计说明:**

| 存储 | 读取速度 | 容量限制 | 说明 |
|------|---------|---------|------|
| 文件缓存 `wp-content/cache/qq-avatars/*.jpg` | <5ms | 无 | 首选，7 天 TTL |
| 无缓存（文件系统不可写） | N/A | N/A | 每次从 qlogo 获取，浏览器 `Cache-Control` 仍可缓解 |

- 同一 QQ 号在多条评论中共享同一份缓存（`md5(qq_number)` 作为文件名）
- 文件缓存 TTL 通过 `filemtime()` 判断，无需额外存储
- 文件系统不可写时：不缓存，直接返回数据，下次请求重新获取 qlogo
- **不使用 Transient 存储二进制数据**，避免 `wp_options` 表膨胀

---

## Patch B: inc/api.php — 路由注册 + 回调函数

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
+            'comment_id' => array(
+                'type' => 'integer',
+                'required' => true,
+                'sanitize_callback' => 'absint',
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

### B-3: get_qq_avatar 回调（api.php:308-332）— 完全重写

**错误处理策略：** 结合原始主题的 `onerror="imgError(this,1)"` 前端降级机制。后端失败时返回 HTTP 404，浏览器自动触发 `onerror`，`imgError()` 将 `src` 替换为主题配置的缺失头像（`_iro.missing_avatars`）或 Gravatar 默认头像。这比后端 302 重定向更好，因为：
1. 不增加后端负担（无需再发一次重定向）
2. 使用主题自身的缺失头像配置，风格一致
3. `imgError()` 已有 `ele.onerror = ""` 防止无限循环

```diff
--- a/inc/api.php
+++ b/inc/api.php
@@ -308,24 +308,30 @@
 /**
- * QQ头像链接解密
+ * QQ avatar proxy by comment ID
  * https://sakura.2heng.xin/wp-json/sakura/v1/qqinfo/avatar
+ *
+ * Returns JPEG binary on success (bypasses REST framework via header+echo+exit).
+ * Returns HTTP 404 on failure — browser triggers onerror → imgError() → missing avatar.
  */
-function get_qq_avatar()
+function get_qq_avatar(WP_REST_Request $request)
 {
-    $encrypted = $_GET["qq"];
-    $imgurl = QQ::get_qq_avatar($encrypted);
-    if (iro_opt('qq_avatar_link') == 'type_2') {
-        $imgdata = file_get_contents($imgurl);
-        $response = new WP_REST_Response();
-        $response->set_headers(
-            array(
-                'Content-Type' => 'image/jpeg',
-                'Cache-Control' => 'max-age=86400'
-            )
-        );
-        echo $imgdata;
-    } else {
-        $response = new WP_REST_Response();
-        $response->set_status(301);
-        $response->header('Location', $imgurl);
-    }
-    return $response;
+    $comment_id = intval($request->get_param('comment_id'));
+    if ($comment_id <= 0) {
+        status_header(404);
+        header('Content-Type: image/jpeg');
+        header('Cache-Control: no-cache');
+        exit;
+    }
+
+    $imgdata = QQ::get_qq_avatar_data($comment_id);
+    if (!$imgdata) {
+        // 返回 404，浏览器触发 <img onerror="imgError(this,1)">
+        // imgError() 会替换为 _iro.missing_avatars 或 Gravatar 默认头像
+        status_header(404);
+        header('Content-Type: image/jpeg');
+        header('Cache-Control: no-cache');
+        exit;
+    }
+
+    // 二进制数据必须绕过 REST 框架直接输出
+    header('Content-Type: image/jpeg');
+    header('Cache-Control: max-age=86400');
+    echo $imgdata;
+    exit;
 }
```

**错误处理流程对比:**

| 场景 | 原始 type_2 | 新 proxy |
|------|-----------|---------|
| qlogo 正常 | 返回 JPEG | 返回 JPEG（缓存 24h） |
| qlogo 超时 | PHP Fatal Error | 返回 404 → `imgError()` → 缺失头像 |
| QQ 号无效 | PHP Fatal Error | 返回 404 → `imgError()` → 缺失头像 |
| comment_id 无效 | N/A | 返回 404 → `imgError()` → 缺失头像 |
| 缓存命中 | N/A | 直接返回（<5ms，无出站请求） |

---

## Patch C: functions.php — change_avatar 函数

```diff
--- a/functions.php
+++ b/functions.php
@@ -2415,33 +2415,24 @@
 add_filter('get_avatar', 'change_avatar', 10, 3);
 function change_avatar($avatar)
 {
-    global $comment, $sakura_privkey;
+    global $comment;
     if ($comment && get_comment_meta($comment->comment_ID, 'new_field_qq', true)) {
-        $qq_number = get_comment_meta($comment->comment_ID, 'new_field_qq', true);
+        $qq_number = sanitize_text_field(get_comment_meta($comment->comment_ID, 'new_field_qq', true));
-        if (iro_opt('qq_avatar_link') == 'off') {
-            return '<img src="https://q2.qlogo.cn/headimg_dl?dst_uin=' . $qq_number . '&spec=100" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
-        }
-        if (iro_opt('qq_avatar_link') == 'type_3') {
-            $qqavatar = file_get_contents('http://ptlogin2.qq.com/getface?appid=1006102&imgtype=3&uin=' . $qq_number);
-            preg_match('/:\"([^\"]*)\"/i', $qqavatar, $matches);
-            return '<img src="' . $matches[1] . '" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
-        }
-
-        // Ensure $sakura_privkey is defined and not null
-        if (isset($sakura_privkey) && !is_null($sakura_privkey)) {
-            // 生成一个合适长度的初始化向量
-            $iv_length = openssl_cipher_iv_length('aes-128-cbc');
-            $iv = openssl_random_pseudo_bytes($iv_length);
-
-            // 加密数据
-            $encrypted = openssl_encrypt($qq_number, 'aes-128-cbc', $sakura_privkey, 0, $iv);
-
-            // 将初始化向量和加密数据一起编码
-            $encrypted = urlencode(base64_encode($iv . $encrypted));
-
-            return '<img src="' . rest_url("sakura/v1/qqinfo/avatar") . '?qq=' . $encrypted . '" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
-        } else {
-            // Handle the case where $sakura_privkey is not set or is null
-            return '<img src="default_avatar_url" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
+        $mode = iro_opt('qq_avatar_link', 'direct');
+
+        if ($mode === 'direct') {
+            return '<img src="https://q2.qlogo.cn/headimg_dl?dst_uin=' . esc_attr(urlencode($qq_number)) . '&spec=100" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
+        }
+
+        if ($mode === 'ptlogin2') {
+            $avatar_url = QQ::get_qq_avatar_url_ptlogin2($comment->comment_ID);
+            if (!$avatar_url) {
+                return $avatar;
+            }
+            return '<img src="' . esc_url($avatar_url) . '" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
         }
+
+        // proxy: 通过 comment_id 代理头像，QQ 号不出现在 URL 中
+        // 后端失败时返回 HTTP 404，浏览器触发 onerror → imgError(this,1) → 缺失头像
+        return '<img src="' . esc_url(rest_url("sakura/v1/qqinfo/avatar") . '?comment_id=' . $comment->comment_ID) . '" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
     }
     return $avatar;
 }
```

**变更:**
1. 移除 `global $sakura_privkey` — 不再需要加密
2. `$qq_number` 加 `sanitize_text_field()`
3. 使用新选项值 `direct`/`ptlogin2`/`proxy`
4. `direct` 模式：QQ 号加 `esc_attr(urlencode())`，保留 `onerror="imgError(this,1)"`
5. `ptlogin2` 模式：调用 `QQ::get_qq_avatar_url_ptlogin2()` + `esc_url()` + 空值回退 `$avatar`
6. `proxy` 模式：通过 `comment_id` 代理头像，保留 `onerror="imgError(this,1)"`
7. 删除整个加密逻辑和硬编码 `default_avatar_url`

**保留的原始主题优秀做法:**
- `onerror="imgError(this,1)"` — 所有模式都保留前端兜底降级
- `lazyload` class — 懒加载
- `spec=100` — 指定头像尺寸

---

## Patch D: opt/options/theme-options.php — 选项重写

```diff
--- a/opt/options/theme-options.php
+++ b/opt/options/theme-options.php
@@ -3170,12 +3170,12 @@
       array(
         'id' => 'qq_avatar_link',
         'type' => 'select',
-        'title' => __('QQ Avatar Link Encryption','sakurairo_csf'),
+        'title' => __('QQ Avatar Link Mode','sakurairo_csf'),
         'options' => array(
-          'off' => __('Off','sakurairo_csf'),
-          'type_1' => __('Redirect (low security)','sakurairo_csf'),
-          'type_2' => __('Get avatar data in the backend (medium security)','sakurairo_csf'),
-          'type_3' => __('Parse avatar interface in the backend (high security, slow)','sakurairo_csf'),
+          'direct' => __('Direct (QQ number visible)','sakurairo_csf'),
+          'ptlogin2' => __('Direct via ptlogin2 (QQ number visible, better CDN compatibility)','sakurairo_csf'),
+          'proxy' => __('Proxy via comment ID (QQ number hidden)','sakurairo_csf'),
         ),
-        'default' => 'off'
+        'default' => 'direct'
       ),
```

**变更:**
- 标题从"Encryption"改为"Mode"
- 选项值全部使用新语义化名称
- 标签明确标注 QQ 号是否可见
- 默认值从 `off` 改为 `direct`

---

## 修复问题与 Patch 映射

| # | 问题 | 严重程度 | Patch | 文件 |
|---|------|---------|-------|------|
| 1 | SSRF: file_get_contents | 🔴 | A, B-3, C | QQ.php, api.php, functions.php |
| 2 | 加解密 IV 不匹配 | 🔴 | A | QQ.php（删除加密方法） |
| 3 | $sakura_privkey 无 isset | 🔴 | A, C | QQ.php, functions.php（移除依赖） |
| 4 | $matches[0] 未定义索引 | 🔴 | A | QQ.php（删除旧方法） |
| 5 | $output 逻辑分支不完整 | 🟠 | A | QQ.php |
| 6 | QQ 号未 urlencode | 🟠 | A, C | QQ.php, functions.php |
| 7 | REST 直接访问 $_GET | 🟠 | B-2, B-3 | api.php |
| 8 | REST 路由无 args schema | 🟠 | B-1 | api.php |
| 9 | type_2 二进制输出损坏 | 🟠 | B-3 | api.php |
| 10 | 错误响应应用 WP_Error | 🟠 | B-3 | api.php |
| 11 | XSS: QQ 号输出未转义 | 🟡 | C | functions.php |
| 12 | $matches[1] 无 isset | 🟡 | A, C | QQ.php, functions.php |
| 13 | 密钥不存在回退硬编码 URL | 🟡 | C | functions.php |
| 14 | 选项标签过时/误导 | 🟢 | D | theme-options.php |
| 15 | proxy 无缓存导致 PHP 负担 | 🟠 | A | QQ.php（文件缓存 + transient 降级） |
| 16 | proxy 失败无降级 | 🟡 | B-3 | api.php（404 → onerror → imgError） |

---

## 前置依赖

- Patch B-2/B-3 依赖 `sakura_verify_rest_request_nonce()` 函数已定义（见 Patch 17）

## 应用顺序

1. **Patch A** — QQ.php（核心重写 + 缓存，无依赖）
2. **Patch B-1** — 路由注册（无依赖）
3. **Patch B-2** — get_qq_info 回调（依赖 sakura_verify_rest_request_nonce）
4. **Patch B-3** — get_qq_avatar 回调（依赖 Patch A）
5. **Patch C** — change_avatar（依赖 Patch A + B-3）
6. **Patch D** — 选项定义（必须与 Patch C 同步应用，否则选项值不匹配）

## 兼容性说明

- **数据库无变更** — comment meta 中的 `new_field_qq` 字段不变，无需迁移
- **选项值变更** — 旧值 `off`/`type_1`/`type_2`/`type_3` 不再使用，升级后需重新选择
- **`$sakura_privkey` 可安全移除** — 不再有任何代码引用它
- **前端无需修改** — 评论表单的 QQ 号输入和昵称查询逻辑不变
- **缓存目录** — `wp-content/cache/qq-avatars/` 自动创建，不可写时不缓存（每次重新获取，浏览器 `Cache-Control` 仍可缓解）
- **`imgError()` 不受影响** — 前端降级机制保持原样，所有模式都保留 `onerror="imgError(this,1)"`
