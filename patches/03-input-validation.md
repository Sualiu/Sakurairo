# 类别 3: 输入验证与净化 — CSS 注入 + 参数净化

> 包含 Patch: 11-13
> 审查依据: wp-plugin-development/references/security.md
> 风险等级: 高（CSS 注入 / 输入注入）

---

## WordPress Skills 最佳实践

来自 `wp-plugin-development/references/security.md`:

> **Golden rule: sanitize/validate on input, escape on output.**
>
> Practical rules:
> - never process the entire `$_POST` / `$_GET` array; read explicit keys
> - use `wp_unslash()` before sanitizing when needed
> - use prepared statements for SQL; avoid interpolating user input into queries

WordPress 输入净化函数层级：

| 函数 | 用途 | 允许字符 |
|------|------|---------|
| `sanitize_text_field()` | 纯文本 | 去除 HTML 标签、额外空白 |
| `sanitize_key()` | WordPress key | 仅小写 a-z 0-9 - _ |
| `sanitize_email()` | 邮箱 | RFC 邮箱格式 |
| `esc_url_raw()` | URL（存储） | 过滤危险协议，存入数据库 |
| `sanitize_textarea_field()` | 多行文本 | 保留换行，去除 HTML |
| `wp_unslash()` | 去除 WP 斜杠 | 必须在 sanitize 前调用 |

**关键原则:** WordPress 会自动给 `$_GET`/`$_POST` 添加 magic quotes 斜杠，因此必须先 `wp_unslash()` 再 sanitize。

---

## 主题存在的问题分析

### 问题 1: dash-scheme.php CSS 注入 — _get() 函数无净化

`inc/dash-scheme.php` 是一个独立输出的 CSS 文件（不经过 WordPress 引导），直接从 `$_GET` 读取参数并拼接到 CSS 输出中。

**违规代码:**
```php
function _get($str){
    return $_GET[$str];  // 完全无净化
}

$color_1 = "#"._get('color_1');  // 直接拼接
$rules = urldecode(_get('rules'));  // 直接输出用户 CSS
```

**攻击向量:**
1. **颜色值注入** — `color_1` 参数可注入 `000;} body{display:none} .x{` 等恶意 CSS
2. **rules 参数注入** — `rules` 参数直接 urldecode 后输出，可注入 `@import url(javascript:...)`、`expression()`、`behavior:` 等
3. **`_get()` 无 `isset` 检查** — 未定义 key 会触发 PHP Notice

### 问题 2: 评论验证码参数直接使用 $_POST

`inc/theme-plus.php` 的 `comment_captcha()` 直接访问 `$_POST['captcha']`、`$_POST['timestamp']`、`$_POST['id']`，未经 `wp_unslash()` 和 `sanitize_text_field()` 处理。

**违规代码:**
```php
if (!(isset($_POST['captcha']) && !empty(trim($_POST['captcha'])))) {
    // 直接使用 $_POST['captcha']，未经 wp_unslash + sanitize
$check = $img->check_captcha($_POST['captcha'], $_POST['timestamp'], $_POST['id']);
```

### 问题 3: search.php content_type 参数直接使用 $_GET

```php
$content_types = isset($_GET['content_type']) ? explode(',', $_GET['content_type']) : $default_checked;
```

`$_GET['content_type']` 未经 `wp_unslash()` 和 `sanitize_key()` 处理，直接 explode 后传入 WP_Query 的 `post_type` 参数。

### 问题 4: 过时的 WP 4.4 版本检查

```php
if ( version_compare( $GLOBALS['wp_version'], '4.4-alpha', '<' ) ) { wp_die(...); }
```

`style.css` 已声明 `Requires at least: 6.0`，此检查完全无用且增加代码噪音。

---

## Patch 清单

### Patch 11: CSS 注入防护 — dash-scheme.php

**文件:** [inc/dash-scheme.php](file:///workspace/inc/dash-scheme.php)
**问题:** `_get()` 无净化 + 颜色值无验证 + rules 无过滤

```diff
--- a/inc/dash-scheme.php
+++ b/inc/dash-scheme.php
@@ _get() 函数修复 @@
-function _get($str){
-	return $_GET[$str];
+function _get($str){
+	return strip_tags(stripslashes($_GET[$str] ?? ''));
 }

@@ -17,16 +17,14 @@
-if(_get('color_1')==NULL) {
-	$color_1="#000";
-} else {
-	$color_1="#"._get('color_1');
-}
-
-if(_get('color_2')==NULL) {
-	$color_2="#000";
-} else {
-	$color_2="#"._get('color_2');
-}
-
-if(_get('color_3')==NULL) {
-	$color_3="#ff6496";
-} else {
-	$color_3="#"._get('color_3');
-}
+/**
+ * Validate and sanitize a hex color value.
+ */
+function _sanitize_hex_color($color, $default = '#000') {
+    if (empty($color)) return $default;
+    $color = ltrim($color, '#');
+    if (!preg_match('/^[a-fA-F0-9]{3}$|^[a-fA-F0-9]{6}$/', $color)) {
+        return $default;
+    }
+    return '#' . $color;
+}
+
+$color_1 = _sanitize_hex_color(_get('color_1'), '#000');
+$color_2 = _sanitize_hex_color(_get('color_2'), '#000');
+$color_3 = _sanitize_hex_color(_get('color_3'), '#ff6496');

@@ -83,2 +83,10 @@
 } else {
-	$rules=urldecode(_get('rules'));
+	$rules = strip_tags(urldecode(_get('rules')));
+	// Remove dangerous CSS functions and imports
+	$rules = preg_replace('/@import\s+url\s*\(/i', '/* blocked */', $rules);
+	$rules = preg_replace('/expression\s*\(/i', '/* blocked */', $rules);
+	$rules = preg_replace('/javascript\s*:/i', '/* blocked */', $rules);
+	$rules = preg_replace('/-moz-binding\s*:/i', '/* blocked */', $rules);
+	$rules = preg_replace('/behavior\s*:/i', '/* blocked */', $rules);
+	$rules = preg_replace('/url\s*\(\s*[\'"]?\s*data\s*:/i', '/* blocked */', $rules);
 }
```

**三层防护:**

1. **`_get()` 函数净化** — `strip_tags(stripslashes())` 去除 HTML 标签和斜杠
   - 此文件独立输出 CSS，未经 WordPress 引导，无法使用 `wp_unslash()`/`sanitize_text_field()`
   - `stripslashes()` 等效于 `wp_unslash()`
   - `strip_tags()` 防止 HTML 标签注入

2. **颜色值 hex 验证** — `_sanitize_hex_color()` 使用正则验证 3/6 位 hex
   - 只允许 `a-f A-F 0-9` 字符
   - 无效值回退默认色

3. **rules 危险模式过滤** — 6 个正则替换阻止 CSS 攻击向量
   - `@import url(` — 防止外部 CSS 导入
   - `expression()` — IE 特有的 CSS 表达式执行
   - `javascript:` — CSS 中的 JS 协议
   - `-moz-binding:` — Firefox XBL 绑定
   - `behavior:` — IE 行为绑定
   - `url(data:` — data URL 可包含恶意内容

---

### Patch 12: 输入验证 — 评论验证码参数净化

**文件:** [inc/theme-plus.php](file:///workspace/inc/theme-plus.php)
**问题:** `$_POST` 直接使用 + 过时 WP 版本检查

```diff
--- a/inc/theme-plus.php
+++ b/inc/theme-plus.php
@@ -147,1 +147,0 @@
-if ( version_compare( $GLOBALS['wp_version'], '4.4-alpha', '<' ) ) { wp_die(__('Please upgrade wordpress to version 4.4+','sakurairo')); }/*请升级到4.4以上版本*/
 // 提示

@@ -169,7 +169,9 @@
   if (iro_opt('comment_captcha_select') == "iro_captcha") {
-      if (!(isset($_POST['captcha']) && !empty(trim($_POST['captcha'])))) {
+      $captcha = isset($_POST['captcha']) ? sanitize_text_field(wp_unslash($_POST['captcha'])) : '';
+      $timestamp = isset($_POST['timestamp']) ? sanitize_text_field(wp_unslash($_POST['timestamp'])) : '';
+      $captcha_id = isset($_POST['id']) ? sanitize_text_field(wp_unslash($_POST['id'])) : '';
+      if (empty(trim($captcha))) {
           return siren_ajax_comment_err(__('Please fill in the captcha answer','sakurairo'));
       }
-      if (!isset($_POST['timestamp']) || !isset($_POST['id']) || !preg_match('/^[\w$.\/]+$/', $_POST['id']) || !ctype_digit($_POST['timestamp'])) {
+      if (empty($timestamp) || empty($captcha_id) || !preg_match('/^[\w$.\/]+$/', $captcha_id) || !ctype_digit($timestamp)) {
           return siren_ajax_comment_err(__('Have you modified the captcha code data? Or refresh the captcha and try again?','sakurairo'));
       }
       include_once( get_template_directory() . '/inc/classes/Captcha.php');
       $img = new Sakura\API\Captcha;
-      $check = $img->check_captcha($_POST['captcha'], $_POST['timestamp'], $_POST['id']);
+      $check = $img->check_captcha($captcha, $timestamp, $captcha_id);
```

**修复要点:**
1. **`wp_unslash()` 在前** — 去除 WP 自动添加的斜杠
2. **`sanitize_text_field()` 在后** — 去除 HTML 标签和多余空白
3. **先净化后验证** — 净化后的值再用于 `preg_match`/`ctype_digit` 验证
4. **移除 WP 4.4 检查** — `style.css` 已声明 `Requires at least: 6.0`

---

### Patch 13: 输入验证 — search.php content_type 净化

**文件:** [search.php](file:///workspace/search.php)
**问题:** `$_GET['content_type']` 直接 explode 传入 WP_Query

```diff
--- a/search.php
+++ b/search.php
@@ -40,1 +40,1 @@
-    $content_types = isset($_GET['content_type']) ? explode(',', $_GET['content_type']) : $default_checked;
+    $content_types = isset($_GET['content_type']) ? array_map('sanitize_key', explode(',', wp_unslash($_GET['content_type']))) : $default_checked;
```

**修复要点:**
1. **`wp_unslash()`** — 去除 WP 斜杠
2. **`explode(',', ...)`** — 按逗号分割为数组
3. **`array_map('sanitize_key', ...)`** — 对每个元素应用 `sanitize_key`
   - `sanitize_key` 仅允许小写 `a-z 0-9 - _`
   - 适合 `post_type` 参数（如 `post`、`page`、`shuoshuo`）
   - 防止注入非法 post_type 值

---

## 审查结论

| Patch | wp_unslash | sanitize | 验证 | 符合最佳实践 |
|-------|-----------|----------|------|-------------|
| 11 | stripslashes（等效） | strip_tags + 正则 | hex 正则 | ✅ |
| 12 | ✅ | sanitize_text_field | preg_match + ctype_digit | ✅ |
| 13 | ✅ | sanitize_key | — | ✅ |

**关键教训:**
- `wp_unslash()` 必须在 sanitize 之前调用
- CSS 上下文需要特殊过滤（`@import`、`expression()` 等），不能仅依赖 `strip_tags`
- `sanitize_key` 适合 key 类参数（post_type、meta_key 等），比 `sanitize_text_field` 更严格
