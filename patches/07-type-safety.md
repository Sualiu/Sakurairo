# 类别 7: 类型安全 — 数据库操作与返回值类型

> 包含 Patch: 21-26
> 审查依据: wp-plugin-development/references/data-and-cron.md, wp-rest-api/references/responses-and-fields.md
> 风险等级: 严重（Fatal Error）到轻微（类型不一致）

---

## WordPress Skills 最佳实践

来自 `wp-plugin-development/references/data-and-cron.md`:

> **Database safety note:** If using `$wpdb->prepare()`, avoid building queries with concatenated user input.

来自 `wp-rest-api/references/responses-and-fields.md`:

> **register_meta / register_post_meta / register_term_meta**
> - Use when the data is stored as meta.
> - Set `show_in_rest => true` to expose under `.meta`.
> - For `object` or `array` types, provide a JSON schema in `show_in_rest.schema`.

WordPress 数据库 API 返回值类型参考：

| 函数 | 返回类型 | 失败时 | 注意事项 |
|------|---------|--------|---------|
| `$wpdb->get_results()` | `array\|null` | `null`（数据库错误） | foreach null 在 PHP 8+ 报 warning |
| `$wpdb->get_var()` | `string\|null` | `null`（无结果） | 需 `(int)` 转换用于整数上下文 |
| `$wpdb->get_row()` | `object\|null` | `null` | 访问属性前需 null 检查 |
| `get_option()` | `mixed` | `false`（选项不存在） | `??` 只对 null 生效，对 false 无效 |
| `get_post_meta($id, $key, true)` | `string\|array\|false` | `''`（meta 不存在） | `$single=true` 时序列化数组返回 array |
| `get_post_meta($id, $key, false)` | `array` | `[]`（空数组） | 总是返回数组 |
| `get_post()` | `WP_Post\|null` | `null` | 访问属性前需 null 检查 |
| `wp_remote_get()` | `array\|WP_Error` | `WP_Error` | WP_Error 不可 ArrayAccess |
| `wp_remote_post()` | `array\|WP_Error` | `WP_Error` | 同上 |
| `json_decode()` | `mixed\|null` | `null`（解析失败） | 需 `is_array` 检查 |

**关键原则:**
1. **PHP 8+ 对 null 的操作更严格** — `foreach(null)`、`count(null)`、`null['key']` 都会报 warning 或 error
2. **`??` 运算符只对 null 生效** — `false['key'] ?? 'default'` 不会回退到 default
3. **`WP_Error` 不可 ArrayAccess** — `$response['body']` 在 WP_Error 上会 Fatal Error
4. **`get_post_meta($id, $key, true)` 可能返回数组** — 当存储的是序列化数组时

---

## 主题存在的问题分析

### 问题 1: get_option 返回 false 当作数组访问（严重 — Fatal Error）

**位置:** [opt/options/theme-options.php:23](file:///workspace/opt/options/theme-options.php#L23)

```php
$vision_resource_basepath = get_option('iro_options')['vision_resource_basepath'] ?? 'https://s.nmxc.ltd/sakurairo_vision/@3.0/';
```

**问题:** `get_option('iro_options')` 在选项未设置时返回 `false`。`false['vision_resource_basepath']` 在 PHP 8+ 触发 Fatal Error。`??` 运算符只对 `null` 生效，对 `false` 无效——因此当选项不存在时仍会报错。

### 问题 2: wp_remote_post 返回 WP_Error 时直接访问 body（严重 — Fatal Error）

**位置:** [inc/classes/Images.php:93-94](file:///workspace/inc/classes/Images.php#L93-L94) 和 [Images.php:133-134](file:///workspace/inc/classes/Images.php#L133-L134)

```php
$response = wp_remote_post($upload_url, $args);
$reply = json_decode($response['body']);  // WP_Error 时 Fatal Error
```

**问题:** `wp_remote_post` 失败返回 `WP_Error`，`WP_Error` 不实现 `ArrayAccess` 接口，`$response['body']` 会触发 Fatal Error。

### 问题 3: $wpdb->get_results 返回 null 直接 foreach（中等 — PHP 8+ Warning）

**位置:** [tpl/content-none.php:29-30](file:///workspace/tpl/content-none.php#L29-L30)

```php
$result = $wpdb->get_results("SELECT ...");
foreach ($result as $post) {  // null 时 PHP 8+ warning
```

**问题:** `get_results` 在数据库错误时返回 `null`，`foreach(null)` 在 PHP 8+ 触发 warning。

### 问题 4: $wpdb->get_var 返回 string|null 未类型转换（中等 — 类型不一致）

**位置:** [inc/theme-plus.php:660](file:///workspace/inc/theme-plus.php#L660)

```php
$author_id = $wpdb->get_var( $wpdb->prepare("...") );
if ( $author_id ) {
    $query_vars['author'] = $author_id;  // string 赋给应为 int 的字段
```

**问题:** `get_var` 返回 `string|null`，赋值给 `author` 后 WordPress 内部期望 `int`。虽然 PHP 弱类型下通常工作，但严格模式下可能出问题。

### 问题 5: get_option 返回 false 的防御性处理缺失（中等 — 类型不一致）

**位置:** [inc/swicher.php:53](file:///workspace/inc/swicher.php#L53), [tpl/content-thumb.php:17](file:///workspace/tpl/content-thumb.php#L17), [search.php:18](file:///workspace/search.php#L18)

```php
'order' => get_option('comment_order'),  // 可能返回 false
$sticky_posts = get_option('sticky_posts');  // 可能返回 false
```

**问题:** `get_option` 在选项不存在时返回 `false`，后续代码可能将 `false` 传入期望 `string` 或 `array` 的函数。

### 问题 6: get_post_meta 返回值类型处理不一致（中等 — 类型不一致）

**位置:** [inc/link-status.php:119-123](file:///workspace/inc/link-status.php#L119-L123), [inc/chatgpt/aigc-manage.php:719](file:///workspace/inc/chatgpt/aigc-manage.php#L719)

```php
$check_status = get_post_meta($link_id, '_link_check_status', true);   // string|false
$status_code = get_post_meta($link_id, '_link_status_code', true);     // string|false，应为 int
$summary = get_post_meta($post->ID, 'ai_summon_excerpt', true);        // string|array|false
$preview = mb_substr(strip_tags($summary), 0, 100);  // 如果 $summary 是 array，strip_tags 报错
```

### 问题 7: wp_remote_get + json_decode 返回值未验证（轻微 — 潜在 null 访问）

**位置:** [inc/classes/Steam.php:43,53](file:///workspace/inc/classes/Steam.php#L43-L53)

```php
$response = json_decode(wp_remote_retrieve_body($response), true);
// 后续直接 $data['response']['games']
```

**问题:** `json_decode` 失败返回 `null`，后续 `isset($data['response']['games'])` 虽然安全，但类型从 `array|null` 不明确。

---

## Patch 清单

### Patch 21: 类型安全 — get_option 返回 false 当作数组访问

**文件:** [opt/options/theme-options.php](file:///workspace/opt/options/theme-options.php)
**严重程度:** 🔴 严重 — PHP 8+ Fatal Error

```diff
--- a/opt/options/theme-options.php
+++ b/opt/options/theme-options.php
@@ -23,1 +23,4 @@
-$vision_resource_basepath = get_option('iro_options')['vision_resource_basepath'] ?? 'https://s.nmxc.ltd/sakurairo_vision/@3.0/';
+$iro_options = get_option('iro_options');
+$vision_resource_basepath = (is_array($iro_options) && !empty($iro_options['vision_resource_basepath']))
+    ? $iro_options['vision_resource_basepath']
+    : 'https://s.nmxc.ltd/sakurairo_vision/@3.0/';
```

**修复说明:**
- `get_option('iro_options')` 返回 `false` 时，`false['key']` 在 PHP 8+ 是 Fatal Error
- `??` 运算符只对 `null` 生效，对 `false` 无效
- 必须先 `is_array()` 检查，再访问数组键

---

### Patch 22: 类型安全 — wp_remote_post 返回 WP_Error 时直接访问 body

**文件:** [inc/classes/Images.php](file:///workspace/inc/classes/Images.php)
**严重程度:** 🔴 严重 — Fatal Error

```diff
--- a/inc/classes/Images.php
+++ b/inc/classes/Images.php
@@ -93,2 +93,6 @@
         $response = wp_remote_post($upload_url, $args);
-        $reply = json_decode($response['body']);
+        if (is_wp_error($response)) {
+            return array('status' => 500, 'success' => false, 'message' => $response->get_error_message(), 'link' => $this->getDefaultErrorImage(), 'proxy' => '');
+        }
+        $reply = json_decode(wp_remote_retrieve_body($response));
+        if (!$reply) {
+            return array('status' => 500, 'success' => false, 'message' => 'Invalid response from Chevereto', 'link' => $this->getDefaultErrorImage(), 'proxy' => '');
+        }

@@ -133,2 +137,6 @@
         $response = wp_remote_post($upload_url, $args);
-        $reply = json_decode($response['body']);
+        if (is_wp_error($response)) {
+            return array('status' => 500, 'success' => false, 'message' => $response->get_error_message(), 'link' => $this->getDefaultErrorImage(), 'proxy' => '');
+        }
+        $reply = json_decode(wp_remote_retrieve_body($response));
+        if (!$reply) {
+            return array('status' => 500, 'success' => false, 'message' => 'Invalid response from Imgur', 'link' => $this->getDefaultErrorImage(), 'proxy' => '');
+        }
```

**修复说明:**
- `WP_Error` 不实现 `ArrayAccess`，`$response['body']` 会 Fatal Error
- 必须先 `is_wp_error()` 检查
- 使用 `wp_remote_retrieve_body()` 安全提取 body
- `json_decode` 失败返回 `null`，需检查 `$reply` 是否有效

---

### Patch 23: 类型安全 — $wpdb->get_var 返回值类型转换

**文件:** [inc/theme-plus.php](file:///workspace/inc/theme-plus.php), [inc/categories-images.php](file:///workspace/inc/categories-images.php)
**严重程度:** ⚠️ 中等 — 类型不一致

```diff
--- a/inc/theme-plus.php
+++ b/inc/theme-plus.php
@@ -660,2 +660,2 @@
         $author_id = $wpdb->get_var( $wpdb->prepare( "SELECT user_id FROM {$wpdb->usermeta} WHERE meta_key='nickname' AND meta_value = %s", $query_vars['author_name'] ) );
         if ( $author_id ) {
-            $query_vars['author'] = $author_id;
+            $query_vars['author'] = (int) $author_id;

--- a/inc/categories-images.php
+++ b/inc/categories-images.php
@@ -156,1 +156,1 @@
-    return (!empty($id)) ? $id : NULL;
+    return (!empty($id)) ? (int) $id : NULL;
```

**修复说明:**
- `get_var` 返回 `string|null`
- WordPress 的 `author` query var 期望 `int`
- `(int)` 转换确保类型一致

---

### Patch 24: 类型安全 — get_option 返回 false 的防御性处理

**文件:** [inc/swicher.php](file:///workspace/inc/swicher.php), [tpl/content-thumb.php](file:///workspace/tpl/content-thumb.php), [search.php](file:///workspace/search.php)
**严重程度:** ⚠️ 中等 — 类型不一致

```diff
--- a/inc/swicher.php
+++ b/inc/swicher.php
@@ -53,1 +53,1 @@
-        'order' => get_option('comment_order'), // ajax comments
+        'order' => get_option('comment_order') ?: 'desc', // ajax comments

@@ -75,1 +75,3 @@
-        'have_annotation' => check(get_post_meta(get_the_ID(), 'iro_chatgpt_annotations', true)), // 检查是否有注释
+        $annotations = get_post_meta(get_the_ID(), 'iro_chatgpt_annotations', true);
+        $has_annotations = !empty($annotations) && (is_array($annotations) ? count($annotations) > 0 : true);
+        'have_annotation' => check($has_annotations), // 检查是否有注释

--- a/tpl/content-thumb.php
+++ b/tpl/content-thumb.php
@@ -17,1 +17,1 @@
-$sticky_posts = get_option('sticky_posts');
+$sticky_posts = get_option('sticky_posts') ?: array();

--- a/search.php
+++ b/search.php
@@ -18,1 +18,1 @@
-$sticky_posts = get_option('sticky_posts');
+$sticky_posts = get_option('sticky_posts') ?: array();
```

**修复说明:**
- `get_option('comment_order')` 可能返回 `false`，`?:` 运算符回退到 `'desc'`
- `get_post_meta(..., true)` 可能返回 `array`（序列化数组），`is_array` 检查后 `count`
- `get_option('sticky_posts')` 可能返回 `false`，`?:` 回退到空数组

---

### Patch 25: 类型安全 — get_post_meta 返回值类型处理

**文件:** [inc/link-status.php](file:///workspace/inc/link-status.php), [inc/chatgpt/aigc-manage.php](file:///workspace/inc/chatgpt/aigc-manage.php)
**严重程度:** ⚠️ 中等 — 类型不一致

```diff
--- a/inc/link-status.php
+++ b/inc/link-status.php
@@ -119,5 +119,5 @@
-        $check_status = get_post_meta($link_id, '_link_check_status', true);
-        $check_time = get_post_meta($link_id, '_link_check_time', true);
+        $check_status = (string) get_post_meta($link_id, '_link_check_status', true);
+        $check_time = (string) get_post_meta($link_id, '_link_check_time', true);
         $failure_count = intval(get_post_meta($link_id, '_link_failure_count', true));
-        $status_code = get_post_meta($link_id, '_link_status_code', true);
-        $error_message = get_post_meta($link_id, '_link_error_message', true);
+        $status_code = (int) get_post_meta($link_id, '_link_status_code', true);
+        $error_message = (string) get_post_meta($link_id, '_link_error_message', true);

--- a/inc/chatgpt/aigc-manage.php
+++ b/inc/chatgpt/aigc-manage.php
@@ -719,2 +719,2 @@
         $summary = get_post_meta($post->ID, 'ai_summon_excerpt', true);
-        $preview = mb_substr(strip_tags($summary), 0, 100) . (mb_strlen($summary) > 100 ? '...' : '');
+        $summary = is_scalar($summary) ? $summary : '';
+        $preview = mb_substr(strip_tags($summary), 0, 100) . (mb_strlen($summary) > 100 ? '...' : '');

@@ -84,1 +84,1 @@
-            echo '<p>' . sprintf(__('Database query result: %s', 'sakurairo'), (count($result) > 0 ? __('Found ', 'sakurairo') . count($result) . __(' records', 'sakurairo') : __('No records found', 'sakurairo'))) . '</p>';
+            echo '<p>' . sprintf(__('Database query result: %s', 'sakurairo'), (!empty($result) && is_array($result) ? __('Found ', 'sakurairo') . count($result) . __(' records', 'sakurairo') : __('No records found', 'sakurairo'))) . '</p>';
```

**修复说明:**
- `(string)` / `(int)` 强制类型转换确保 meta 值类型一致
- `is_scalar($summary)` 检查防止 `strip_tags(array)` 报错
- `!empty($result) && is_array($result)` 防止 `count(null)` 报错

---

### Patch 26: 类型安全 — wp_remote_get + json_decode 返回值验证

**文件:** [inc/classes/Steam.php](file:///workspace/inc/classes/Steam.php)
**严重程度:** ⚠️ 轻微 — 潜在 null 访问

```diff
--- a/inc/classes/Steam.php
+++ b/inc/classes/Steam.php
@@ -43,1 +43,1 @@
-                $response = json_decode(wp_remote_retrieve_body($response), true);
+                $decoded = json_decode(wp_remote_retrieve_body($response), true);
+                $response = is_array($decoded) ? $decoded : ['response' => ['games' => []]];

@@ -53,1 +53,1 @@
-            $response = json_decode(wp_remote_retrieve_body($response), true);
+            $decoded = json_decode(wp_remote_retrieve_body($response), true);
+            $response = is_array($decoded) ? $decoded : ['response' => ['games' => []]];
```

**修复说明:**
- `json_decode` 失败返回 `null`
- `is_array($decoded)` 检查后回退到空游戏列表
- 确保后续 `$data['response']['games']` 访问安全

---

## 审查结论

| Patch | 严重程度 | 问题类型 | 修复方式 | 符合最佳实践 |
|-------|---------|---------|---------|-------------|
| 21 | 🔴 严重 | false 数组访问 | is_array 检查 | ✅ |
| 22 | 🔴 严重 | WP_Error 数组访问 | is_wp_error 检查 | ✅ |
| 23 | ⚠️ 中等 | string→int 类型不一致 | (int) 转换 | ✅ |
| 24 | ⚠️ 中等 | false 传入期望 string/array | ?: 回退 | ✅ |
| 25 | ⚠️ 中等 | mixed 类型未统一 | (string)/(int)/is_scalar | ✅ |
| 26 | ⚠️ 轻微 | null 数组访问 | is_array 检查 | ✅ |

**关键教训:**
- PHP 8+ 对 null/false 的操作更严格，必须显式检查
- `??` 运算符只对 null 生效，对 false 无效——用 `?:` 或 `is_array` 检查
- `WP_Error` 不可 ArrayAccess——`wp_remote_*` 后必须 `is_wp_error()` 检查
- `get_post_meta($id, $key, true)` 可能返回 `array`（序列化数组），需 `is_scalar`/`is_array` 检查
- `json_decode` 失败返回 `null`，需 `is_array` 检查
- 数据库函数返回值类型固定，应使用 `(int)`/`(string)` 转换确保类型一致
