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
change_avatar() → <img src="REST?comment_id=123">
                        ↓
                get_qq_avatar() → get_comment_meta(comment_id, 'new_field_qq') → qlogo URL
```

优势：
- QQ 号**完全不出现在任何 URL 中**
- 无需加密/解密，无密钥管理问题
- 利用 WordPress 已有的 comment meta 存储机制
- `comment_id` 是公开信息，不存在隐私泄露

### 昵称查询不变

用户在评论表单中**主动输入** QQ 号 → 调用 `/qqinfo/json?qq=123456789` → 返回昵称+头像。这是用户自愿行为，不属于隐私泄露。

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
4. 新增 `get_qq_avatar_data($comment_id)` — 通过 comment_id 读取 QQ 号返回头像二进制数据
5. 新增 `get_qq_avatar_url_ptlogin2($comment_id)` — 通过 ptlogin2 接口获取头像 URL

```diff
--- a/inc/classes/QQ.php
+++ b/inc/classes/QQ.php
@@ -1,40 +1,122 @@
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
+     * Get QQ avatar binary data by comment ID.
+     * Fetches avatar image and returns raw JPEG data.
+     *
+     * @param int $comment_id WordPress comment ID
+     * @return string|false Binary image data on success, false on failure
+     */
+    public static function get_qq_avatar_data($comment_id) {
+        $imgurl = self::get_qq_avatar_url($comment_id);
+        if (!$imgurl) {
+            return false;
+        }
+
+        $response = wp_remote_get(esc_url_raw($imgurl), array('timeout' => 10));
+        if (is_wp_error($response) || wp_remote_retrieve_response_code($response) !== 200) {
+            return false;
+        }
+
+        return wp_remote_retrieve_body($response);
     }
 }
```

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

`proxy` 模式下 REST 端点只有一种行为：通过 comment_id 代理头像数据。

```diff
--- a/inc/api.php
+++ b/inc/api.php
@@ -308,24 +308,22 @@
 /**
- * QQ头像链接解密
+ * QQ avatar proxy by comment ID
  * https://sakura.2heng.xin/wp-json/sakura/v1/qqinfo/avatar
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
+        return new WP_Error('rest_invalid_comment_id', 'Invalid comment ID', array('status' => 400));
+    }
+
+    $imgdata = QQ::get_qq_avatar_data($comment_id);
+    if (!$imgdata) {
+        return new WP_Error('rest_qq_avatar_not_found', 'Avatar not found', array('status' => 404));
+    }
+
+    // 二进制数据必须绕过 REST 框架直接输出
+    header('Content-Type: image/jpeg');
+    header('Cache-Control: max-age=86400');
+    echo $imgdata;
+    exit;
 }
```

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
+        return '<img src="' . esc_url(rest_url("sakura/v1/qqinfo/avatar") . '?comment_id=' . $comment->comment_ID) . '" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
     }
     return $avatar;
 }
```

**变更:**
1. 移除 `global $sakura_privkey` — 不再需要加密
2. `$qq_number` 加 `sanitize_text_field()`
3. 使用新选项值 `direct`/`ptlogin2`/`proxy`
4. `direct` 模式：QQ 号加 `esc_attr(urlencode())`
5. `ptlogin2` 模式：调用 `QQ::get_qq_avatar_url_ptlogin2()` + `esc_url()` + 空值回退 `$avatar`
6. `proxy` 模式：通过 `comment_id` 代理头像
7. 删除整个加密逻辑和硬编码 `default_avatar_url`

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

---

## 前置依赖

- Patch B-2/B-3 依赖 `sakura_verify_rest_request_nonce()` 函数已定义（见 Patch 17）

## 应用顺序

1. **Patch A** — QQ.php（核心重写，无依赖）
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
