# 类别 2: 认证与授权 — CSRF + 权限检查

> 包含 Patch: 07-10
> 审查依据: wp-plugin-development/references/security.md
> 风险等级: 严重（认证绕过 / 权限提升）

---

## WordPress Skills 最佳实践

来自 `wp-plugin-development/references/security.md`:

> **Nonces + permissions**
> - Nonces help prevent CSRF, not authorization.
> - Always pair nonces with capability checks (`current_user_can()` or a more specific capability).

核心原则：
1. **Nonce 防 CSRF，不防授权** — nonce 只能证明请求来自站点内，不能证明用户有权限
2. **Nonce 必须与 capability 配对** — 缺一不可
3. **`check_admin_referer()` / `check_ajax_referer()`** — 验证 nonce，失败则 die
4. **`current_user_can()`** — 验证用户权限，是授权的核心
5. **`wp_verify_nonce()`** — 仅验证 nonce，不 die（用于条件判断）

WordPress 能力层级（capability）参考：

| 能力 | 适用角色 | 场景 |
|------|---------|------|
| `manage_options` | 管理员 | 主题/插件设置 |
| `manage_categories` | 编辑+管理员 | 分类管理 |
| `edit_posts` | 作者+ | 内容编辑 |
| `moderate_comments` | 编辑+ | 评论管理 |

---

## 主题存在的问题分析

### 问题 1: AJAX 评论无 CSRF 保护

`inc/theme-plus.php` 中的 `siren_ajax_comment_callback()` 处理 AJAX 评论提交，但完全没有 nonce 验证。攻击者可构造恶意页面，诱导已登录用户访问后自动提交评论。

**违规代码:**
```php
function siren_ajax_comment_callback(){
    $comment = wp_handle_comment_submission( wp_unslash( $_POST ) );
    // 无 nonce 验证，无 CSRF 保护
```

### 问题 2: AIGC 管理表单有 nonce 但无权限检查

`inc/chatgpt/aigc-manage.php` 中 6 处表单处理都使用了 `check_admin_referer()` 验证 nonce，但缺少 `current_user_can('manage_options')` 权限检查。

**违规代码:**
```php
if (isset($_POST['generate_annotations']) && isset($_POST['post_id']) && check_admin_referer('iro_generate_annotations')) {
    // 有 nonce 但无 capability 检查
    // 任何已登录用户（如订阅者）只要拿到 nonce 就能执行
```

**风险分析:**
- nonce 会通过 `wp_nonce_field()` 输出到页面 HTML 中
- 任何能访问该页面的用户都能获取 nonce
- 如果页面权限控制不严（如仅检查 `is_admin()` 而非 `current_user_can()`），低权限用户可获取 nonce
- 即使 nonce 有效，也不代表用户有权限执行操作

### 问题 3: 分类图片保存无权限检查 + 无 nonce

`inc/categories-images.php` 的 `z_save_taxonomy_image()` 钩子在 `edit_term`/`create_term` 上触发，但：
1. 无 `current_user_can('manage_categories')` 检查
2. 无 nonce 验证
3. `$_POST['taxonomy_image']` 未经 `wp_unslash()` 处理

**违规代码:**
```php
function z_save_taxonomy_image($term_id) {
    if(isset($_POST['taxonomy_image']))
        update_option('z_taxonomy_image'.$term_id, $_POST['taxonomy_image'], NULL);
}
```

### 问题 4: 链接分类优先级保存无权限检查 + 无 nonce

`inc/link-status.php` 的 `created_link_category`/`edited_link_category` 钩子同样缺少权限检查和 nonce 验证。

---

## Patch 清单

### Patch 07: CSRF 防护 — AJAX 评论 nonce 验证

**文件:** [comments.php](file:///workspace/comments.php), [inc/theme-plus.php](file:///workspace/inc/theme-plus.php)
**问题:** AJAX 评论无 CSRF 保护

```diff
--- a/comments.php
+++ b/comments.php
@@ -192,0 +192,1 @@
                                             </label>
+                                            ' . wp_nonce_field('sakurairo_ajax_comment', 'sakurairo_comment_nonce', true, false) . '
                                         </div>',

--- a/inc/theme-plus.php
+++ b/inc/theme-plus.php
@@ -211,0 +211,1 @@
     function siren_ajax_comment_callback(){
+      check_ajax_referer('sakurairo_ajax_comment', 'sakurairo_comment_nonce');
       $comment = wp_handle_comment_submission( wp_unslash( $_POST ) );
```

**审查说明:**
- 评论是公开功能，未登录用户也应能评论，因此不加 `current_user_can()` 是合理的
- `check_ajax_referer()` 在 nonce 无效时会 `wp_die()`，阻止 CSRF 攻击
- `wp_nonce_field()` 第四个参数 `false` 表示返回字符串而非直接 echo（因为此处是在字符串拼接中）

---

### Patch 08: 权限检查 — AIGC 管理表单加入 current_user_can

**文件:** [inc/chatgpt/aigc-manage.php](file:///workspace/inc/chatgpt/aigc-manage.php)
**问题:** 6 处表单有 nonce 但无 capability 配对

```diff
--- a/inc/chatgpt/aigc-manage.php
+++ b/inc/chatgpt/aigc-manage.php
@@ -46,1 +46,1 @@
-    if (isset($_POST['generate_annotations']) && isset($_POST['post_id']) && check_admin_referer('iro_generate_annotations')) {
+    if (isset($_POST['generate_annotations']) && isset($_POST['post_id']) && current_user_can('manage_options') && check_admin_referer('iro_generate_annotations')) {

@@ -57,1 +57,1 @@
-    } elseif (isset($_POST['delete_annotations']) && isset($_POST['post_id']) && check_admin_referer('iro_generate_annotations')) {
+    } elseif (isset($_POST['delete_annotations']) && isset($_POST['post_id']) && current_user_can('manage_options') && check_admin_referer('iro_generate_annotations')) {

@@ -62,1 +62,1 @@
-    } elseif (isset($_POST['test_save']) && isset($_POST['post_id']) && check_admin_referer('iro_generate_annotations')) {
+    } elseif (isset($_POST['test_save']) && isset($_POST['post_id']) && current_user_can('manage_options') && check_admin_referer('iro_generate_annotations')) {

@@ -204,1 +204,1 @@
-        if (isset($_POST['debug_annotations']) && isset($_POST['post_id']) && check_admin_referer('iro_generate_annotations')) {
+        if (isset($_POST['debug_annotations']) && isset($_POST['post_id']) && current_user_can('manage_options') && check_admin_referer('iro_generate_annotations')) {

@@ -604,1 +604,1 @@
-    if (isset($_POST['generate_summary']) && isset($_POST['post_id']) && check_admin_referer('iro_generate_summary')) {
+    if (isset($_POST['generate_summary']) && isset($_POST['post_id']) && current_user_can('manage_options') && check_admin_referer('iro_generate_summary')) {

@@ -617,1 +617,1 @@
-    } elseif (isset($_POST['delete_summary']) && isset($_POST['post_id']) && check_admin_referer('iro_generate_summary')) {
+    } elseif (isset($_POST['delete_summary']) && isset($_POST['post_id']) && current_user_can('manage_options') && check_admin_referer('iro_generate_summary')) {

@@ -622,1 +622,1 @@
-    } elseif (isset($_POST['test_save']) && isset($_POST['post_id']) && check_admin_referer('iro_generate_summary')) {
+    } elseif (isset($_POST['test_save']) && isset($_POST['post_id']) && current_user_can('manage_options') && check_admin_referer('iro_generate_summary')) {
```

**审查说明:**
- `current_user_can('manage_options')` 放在 `check_admin_referer()` 之前，先检查权限再验证 nonce
- `manage_options` 是管理员专属能力，适合 AIGC 管理操作
- 完全符合 WordPress skills "nonce + capability 必须配对" 原则

---

### Patch 09: 权限检查 + CSRF — 分类图片保存

**文件:** [inc/categories-images.php](file:///workspace/inc/categories-images.php)
**问题:** 无权限检查 + 无 nonce + 输入未净化

```diff
--- a/inc/categories-images.php
+++ b/inc/categories-images.php
@@ -147,2 +147,5 @@
 function z_save_taxonomy_image($term_id) {
-    if(isset($_POST['taxonomy_image']))
-        update_option('z_taxonomy_image'.$term_id, $_POST['taxonomy_image'], NULL);
+    if (isset($_POST['taxonomy_image']) && current_user_can('manage_categories')) {
+        if (!isset($_POST['z_taxonomy_image_nonce']) || !wp_verify_nonce($_POST['z_taxonomy_image_nonce'], 'z_save_taxonomy_image_' . $term_id)) {
+            return;
+        }
+        update_option('z_taxonomy_image'.$term_id, esc_url_raw(wp_unslash($_POST['taxonomy_image'])), false);
+    }
 }
```

**配套修改:** 需在分类编辑表单 `z_taxonomy_image_edit_form_field()` 中添加：
```php
wp_nonce_field('z_save_taxonomy_image_' . $term_id, 'z_taxonomy_image_nonce');
```

**三重防护:**
1. `current_user_can('manage_categories')` — 授权检查
2. `wp_verify_nonce()` — CSRF 防护
3. `esc_url_raw(wp_unslash())` — 输入净化（URL 存储用 esc_url_raw）

---

### Patch 10: 权限检查 + CSRF — 链接分类优先级保存

**文件:** [inc/link-status.php](file:///workspace/inc/link-status.php)
**问题:** 无权限检查 + 无 nonce

```diff
--- a/inc/link-status.php
+++ b/inc/link-status.php
@@ -324,2 +324,4 @@
-    if (isset($_POST['term_priority'])) {
-        update_option('_link_category_priority_' . $term_id, intval($_POST['term_priority']));
+    if (isset($_POST['term_priority']) && current_user_can('manage_categories')) {
+        if (!isset($_POST['_link_priority_nonce']) || !wp_verify_nonce($_POST['_link_priority_nonce'], 'save_link_priority_' . $term_id)) {
+            return;
+        }
+        update_option('_link_category_priority_' . $term_id, intval(wp_unslash($_POST['term_priority'])));
     }

@@ -330,2 +332,4 @@
-    if (isset($_POST['term_priority'])) {
-        update_option('_link_category_priority_' . $term_id, intval($_POST['term_priority']));
+    if (isset($_POST['term_priority']) && current_user_can('manage_categories')) {
+        if (!isset($_POST['_link_priority_nonce']) || !wp_verify_nonce($_POST['_link_priority_nonce'], 'save_link_priority_' . $term_id)) {
+            return;
+        }
+        update_option('_link_category_priority_' . $term_id, intval(wp_unslash($_POST['term_priority'])));
     }
```

**配套修改:** 需在链接分类编辑表单中添加 nonce field。

---

## 审查结论

| Patch | nonce | capability | 输入净化 | 符合最佳实践 |
|-------|-------|-----------|---------|-------------|
| 07 | ✅ | N/A（公开功能） | N/A | ✅ |
| 08 | ✅（已有） | ✅（新增） | N/A | ✅ |
| 09 | ✅（新增） | ✅（新增） | ✅ | ✅ |
| 10 | ✅（新增） | ✅（新增） | ✅ | ✅ |

**关键教训:**
- WordPress skills 明确要求 nonce 和 capability 必须配对
- 仅检查 nonce 不够——nonce 可被同站任何用户获取
- 仅检查 capability 不够——无法防御 CSRF
- `wp_unslash()` 必须在 `sanitize`/`esc_url_raw` 之前调用，因为 WP 会自动给 `$_POST` 添加斜杠
