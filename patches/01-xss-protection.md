# 类别 1: XSS 防护 — 输出转义

> 包含 Patch: 01-06
> 审查依据: wp-plugin-development/references/security.md
> 风险等级: 高（存储型/反射型 XSS）

---

## WordPress Skills 最佳实践

来自 `wp-plugin-development/references/security.md`:

> **Golden rule: sanitize/validate on input, escape on output.**
>
> - never process the entire `$_POST` / `$_GET` array; read explicit keys
> - use `wp_unslash()` before sanitizing when needed
> - use prepared statements for SQL; avoid interpolating user input into queries

WordPress 提供以下输出转义函数，必须根据上下文选择：

| 上下文 | 函数 | 用途 |
|--------|------|------|
| HTML 正文 | `esc_html()` | 转义 `< > & " '` |
| HTML 属性 | `esc_attr()` | 转义属性值中的特殊字符 |
| URL | `esc_url()` | 输出侧 URL 净化（允许 http/https/mailto 等） |
| URL（存储） | `esc_url_raw()` | 存入数据库前的 URL 净化（不输出实体） |
| 富文本 HTML | `wp_kses_post()` | 允许安全 HTML 标签，过滤危险脚本 |
| JavaScript | `esc_js()` | JS 字符串上下文转义 |
| 翻译字符串 | `esc_html__()` / `esc_attr__()` | 翻译 + 转义一体化 |

---

## 主题存在的问题分析

### 问题 1: 主题选项值直接输出未经转义

Sakurairo 主题大量使用 `iro_opt()` 函数读取主题选项值并直接 echo 到 HTML 中。这些选项值存储在数据库中，虽然通常由管理员设置，但存在以下风险：

1. **管理员账号被入侵** → 攻击者可修改选项值注入 XSS
2. **多管理员场景** → 低权限管理员若能编辑选项，可攻击高权限管理员
3. **选项导入功能** → 如果主题支持选项导入/导出，恶意选项包可传播 XSS

**违规代码模式（全主题普遍存在）:**
```php
<?php echo iro_opt('exhibition_area_icon', 'fa-solid fa-laptop'); ?>
<?php echo iro_opt('signature_text', 'Hi, Mashiro?'); ?>
```

### 问题 2: 动态数组数据直接输出

`exhibition.php` 中的 `$square_cards`、`$medal` 等数组数据直接输出到 HTML 属性和正文中，未经任何转义。这些数据虽然来自主题选项，但结构复杂，可能包含用户可控内容。

### 问题 3: 用户描述输出未经 wp_kses_post

`author.php` 中用户个人描述直接 `nl2br()` 输出，未经过 `wp_kses_post()` 过滤。用户描述是用户可控输入，可能包含 `<script>` 标签。

### 问题 4: URL 输出未经 esc_url

多处 `get_permalink()`、`get_edit_post_link()`、图片 URL 直接输出到 `href`/`src` 属性，未经 `esc_url()` 转义。虽然 WordPress 核心函数返回值通常安全，但防御性编程要求仍需转义。

---

## Patch 清单

### Patch 01: exhibition.php 展示页输出转义

**文件:** [exhibition.php](file:///workspace/exhibition.php)
**问题:** `iro_opt()` 值、`$square_cards`/`$medal` 数组值直接输出

```diff
--- a/exhibition.php
+++ b/exhibition.php
@@ -346,1 +346,1 @@
-        <i class="<?php echo iro_opt('exhibition_area_icon', 'fa-solid fa-laptop'); ?>" aria-hidden="true"></i> <?php echo iro_opt('exhibition_area_title', '展示'); ?>
+        <i class="<?php echo esc_attr(iro_opt('exhibition_area_icon', 'fa-solid fa-laptop')); ?>" aria-hidden="true"></i> <?php echo esc_html(iro_opt('exhibition_area_title', '展示')); ?>

@@ -367,3 +367,3 @@
-                          <i class="<?php echo $square_cards[$component]['icon']; ?>"></i>
+                          <i class="<?php echo esc_attr($square_cards[$component]['icon']); ?>"></i>
                           <div class="capsule-content">
-                            <span class="capsule-label"><?php echo $square_cards[$component]['label']; ?></span>
-                            <span class="capsule-value"><?php echo $square_cards[$component]['value']; ?></span>
+                            <span class="capsule-label"><?php echo esc_html($square_cards[$component]['label']); ?></span>
+                            <span class="capsule-value"><?php echo esc_html($square_cards[$component]['value']); ?></span>

@@ -383,3 +383,3 @@
-                            <div class="stat-capsule medal-capsule <?php echo $medal['type']; ?>"
-                                 data-medal-type="<?php echo $component; ?>"
-                                 data-medal-level="<?php echo $medal['type']; ?>"
+                            <div class="stat-capsule medal-capsule <?php echo esc_attr($medal['type']); ?>"
+                                 data-medal-type="<?php echo esc_attr($component); ?>"
+                                 data-medal-level="<?php echo esc_attr($medal['type']); ?>"

@@ -400,2 +400,2 @@
-                                    <span class="capsule-label"><?php echo $medal['label']; ?></span>
+                                    <span class="capsule-label"><?php echo esc_html($medal['label']); ?></span>
                                     <span class="capsule-value">
-                                    <?php echo $square_cards[$component]['value']; ?>
+                                    <?php echo esc_html($square_cards[$component]['value']); ?>

@@ -407,2 +407,2 @@
-                                        <?php echo isset($medal['achievement']) ? $medal['achievement'] : ''; ?>
+                                        <?php echo isset($medal['achievement']) ? esc_html($medal['achievement']) : ''; ?>
                                     </div>
                                     <?php if (isset($medal['next_level'])) : ?>
                                     <div class="medal-next-level">
-                                        <?php echo $medal['next_level']; ?>
+                                        <?php echo esc_html($medal['next_level']); ?>

@@ -418,3 +418,3 @@
-                                <i class="<?php echo $square_cards[$component]['icon']; ?>"></i>
+                                <i class="<?php echo esc_attr($square_cards[$component]['icon']); ?>"></i>
                                 <div class="capsule-content">
-                                <span class="capsule-label"><?php echo $square_cards[$component]['label']; ?></span>
-                                <span class="capsule-value"><?php echo $square_cards[$component]['value']; ?></span>
+                                <span class="capsule-label"><?php echo esc_html($square_cards[$component]['label']); ?></span>
+                                <span class="capsule-value"><?php echo esc_html($square_cards[$component]['value']); ?></span>

@@ -459,3 +459,3 @@
-                            <i class="<?php echo $square_cards['announcement']['icon']; ?>"></i>
+                            <i class="<?php echo esc_attr($square_cards['announcement']['icon']); ?>"></i>
                             <div class="capsule-content">
-                                <span class="announcement-line first-line"><?php echo $first; ?></span>
-                                <span class="announcement-line second-line"><?php echo $second; ?></span>
+                                <span class="announcement-line first-line"><?php echo esc_html($first); ?></span>
+                                <span class="announcement-line second-line"><?php echo esc_html($second); ?></span>
```

**转义函数选择依据:**
- `class` 属性 → `esc_attr()`（属性上下文）
- `data-*` 属性 → `esc_attr()`（属性上下文）
- `<span>` 正文文本 → `esc_html()`（HTML 正文上下文）

---

### Patch 02: author.php 作者页输出转义

**文件:** [author.php](file:///workspace/author.php)
**问题:** 用户描述直接 `nl2br()` 输出，未过滤 HTML；`page_auto_load` 选项未转义

```diff
--- a/author.php
+++ b/author.php
@@ -15,1 +15,1 @@
-            echo $description ? nl2br($description) : __("No personal profile set yet", "sakurairo");
+            echo $description ? wp_kses_post(nl2br($description)) : esc_html__("No personal profile set yet", "sakurairo");

@@ -56,1 +56,1 @@
-        <div id="add_post"><span id="add_post_time" style="visibility: hidden;" title="<?php echo iro_opt('page_auto_load', ''); ?>"></span></div>
+        <div id="add_post"><span id="add_post_time" style="visibility: hidden;" title="<?php echo esc_attr(iro_opt('page_auto_load', '')); ?>"></span></div>
```

**转义函数选择依据:**
- 用户描述允许 HTML（`<br>`、`<a>` 等）→ `wp_kses_post()`（允许安全标签，过滤 `<script>`/`onerror` 等）
- `title` 属性 → `esc_attr()`
- 翻译字符串 → `esc_html__()`（翻译 + 转义一体化）

---

### Patch 03: tpl/content-none.php 输出转义 + null 安全

**文件:** [tpl/content-none.php](file:///workspace/tpl/content-none.php)
**问题:** URL/标题直接输出未转义；`$wpdb->get_results()` 可能返回 null

```diff
--- a/tpl/content-none.php
+++ b/tpl/content-none.php
@@ -28,2 +28,4 @@
-				$result = $wpdb->get_results("SELECT ID,post_title FROM $wpdb->posts where post_status='publish' and post_type='post' ORDER BY ID DESC LIMIT 0 , 20");
-				foreach ($result as $post) {
+				$result = $wpdb->get_results("SELECT ID,post_title FROM $wpdb->posts where post_status='publish' and post_type='post' ORDER BY ID DESC LIMIT 0 , 20");
+				if (!empty($result)) {
+				foreach ($result as $post) {
@@ -35,1 +37,1 @@
-				<li><a href="<?php echo get_permalink($postid); ?>" title="<?php echo $title ?>"><?php echo $title ?></a> </li>
+				<li><a href="<?php echo esc_url(get_permalink($postid)); ?>" title="<?php echo esc_attr($title); ?>"><?php echo esc_html($title); ?></a> </li>
+				}
```

**双重修复:**
1. XSS: `esc_url()` + `esc_attr()` + `esc_html()` 分别用于 URL/属性/正文
2. 类型安全: `!empty($result)` 防御 `get_results()` 返回 null 时 foreach 报错

---

### Patch 04: layouts/imgbox.php 社交图标区输出转义

**文件:** [layouts/imgbox.php](file:///workspace/layouts/imgbox.php)
**问题:** 社交图标 link/img_url/action/inner/class/title 全部直接输出

```diff
--- a/layouts/imgbox.php
+++ b/layouts/imgbox.php
@@ -63,1 +63,1 @@
-                <a href="javascript:void(0);" title="<?= strtolower($icon_data['icon']) ?>" onclick="<?= $icon_data['action'] ?>">
+                <a href="javascript:void(0);" title="<?= esc_attr(strtolower($icon_data['icon'])) ?>" onclick="<?= esc_attr($icon_data['action']) ?>">
@@ -67,1 +67,1 @@
-                        <img loading="lazy" src="<?= iro_opt('vision_resource_basepath') . iro_opt('social_display_icon') . '/' . esc_attr($icon_data['icon']) . '.webp' ?>" />
+                        <img loading="lazy" src="<?= esc_url(iro_opt('vision_resource_basepath') . iro_opt('social_display_icon') . '/' . $icon_data['icon'] . '.webp') ?>" />
@@ -71,1 +71,1 @@
-                <?= $icon_data['inner'] ?>
+                <?= wp_kses_post($icon_data['inner']) ?>
@@ -85,1 +85,1 @@
-            <li><a href="<?= $value['link']; ?>" target="_blank" class="social-<?= $value['class'] ?? $key ?>" title="<?= $title ?>">
+            <li><a href="<?= esc_url($value['link']); ?>" target="_blank" class="social-<?= esc_attr($value['class'] ?? $key) ?>" title="<?= esc_attr($title) ?>">
@@ -89,1 +89,1 @@
-                    <img alt="<?= $title ?>" loading="lazy" src="<?= $img_url ?>" />
+                    <img alt="<?= esc_attr($title) ?>" loading="lazy" src="<?= esc_url($img_url) ?>" />
@@ -104,1 +104,1 @@
-                <img loading="lazy" alt="E-mail" src="<?php echo iro_opt('vision_resource_basepath').iro_opt('social_display_icon').'/' . 'mail.webp'; ?>" />
+                <img loading="lazy" alt="E-mail" src="<?php echo esc_url(iro_opt('vision_resource_basepath').iro_opt('social_display_icon').'/' . 'mail.webp'); ?>" />
@@ -337,1 +337,1 @@
-                    <p><?php echo iro_opt('signature_text', 'Hi, Mashiro?'); ?></p>
+                    <p><?php echo esc_html(iro_opt('signature_text', 'Hi, Mashiro?')); ?></p>
```

**转义函数选择依据:**
- `onclick` 属性 → `esc_attr()`（防止属性注入）
- `src`/`href` URL → `esc_url()`（URL 上下文）
- `inner` HTML 内容 → `wp_kses_post()`（允许 `<img>` 等安全标签）
- `class`/`title`/`alt` 属性 → `esc_attr()`
- 签名文本 → `esc_html()`

---

### Patch 05: inc/chatgpt/aigc-manage.php 管理页输出转义

**文件:** [inc/chatgpt/aigc-manage.php](file:///workspace/inc/chatgpt/aigc-manage.php)
**问题:** 管理页面 API 配置、文章列表、消息提示等全部直接输出

```diff
--- a/inc/chatgpt/aigc-manage.php
+++ b/inc/chatgpt/aigc-manage.php
@@ -15,3 +15,3 @@
-<?php _e('API Endpoint: ', 'sakurairo'); ?><?php echo iro_opt('chatgpt_endpoint', __('Not set', 'sakurairo')); ?>
-<?php _e('API Key: ', 'sakurairo'); ?><?php echo empty(iro_opt('chatgpt_access_token')) ? __('Not set', 'sakurairo') : __('Set (Length: ', 'sakurairo') . strlen(iro_opt('chatgpt_access_token')) . ')'; ?>
-<?php _e('Model: ', 'sakurairo'); ?><?php echo iro_opt('chatgpt_model', __('Not set', 'sakurairo')); ?>
+<?php _e('API Endpoint: ', 'sakurairo'); ?><?php echo esc_html(iro_opt('chatgpt_endpoint', __('Not set', 'sakurairo'))); ?>
+<?php _e('API Key: ', 'sakurairo'); ?><?php echo empty(iro_opt('chatgpt_access_token')) ? esc_html__('Not set', 'sakurairo') : esc_html(__('Set (Length: ', 'sakurairo') . strlen(iro_opt('chatgpt_access_token')) . ')'); ?>
+<?php _e('Model: ', 'sakurairo'); ?><?php echo esc_html(iro_opt('chatgpt_model', __('Not set', 'sakurairo'))); ?>

@@ -110,2 +110,2 @@
-            <div class="notice notice-<?php echo $message_type; ?> is-dismissible">
-                <p><?php echo $message; ?></p>
+            <div class="notice notice-<?php echo esc_attr($message_type); ?> is-dismissible">
+                <p><?php echo wp_kses_post($message); ?></p>

@@ -144,1 +144,1 @@
-                                    <option value="<?php echo $p->ID; ?>"><?php echo $p->post_title; ?> (ID: <?php echo $p->ID; ?>)</option>
+                                    <option value="<?php echo esc_attr($p->ID); ?>"><?php echo esc_html($p->post_title); ?> (ID: <?php echo esc_html($p->ID); ?>)</option>

@@ -180,5 +180,5 @@
-                                    <a href="<?php echo get_permalink($post->ID); ?>" target="_blank">
-                                        <?php echo $post->post_title; ?>
+                                    <a href="<?php echo esc_url(get_permalink($post->ID)); ?>" target="_blank">
+                                        <?php echo esc_html($post->post_title); ?>
                                     </a>
                                 </td>
-                                <td><?php echo $annotation_count; ?></td>
+                                <td><?php echo esc_html($annotation_count); ?></td>
                                 <td>
-                                    <a href="#" class="view-annotations" data-post-id="<?php echo $post->ID; ?>"><?php _e('View Annotations', 'sakurairo'); ?></a> |
-                                    <a href="<?php echo get_edit_post_link($post->ID); ?>" target="_blank"><?php _e('Edit Post', 'sakurairo'); ?></a> |
+                                    <a href="#" class="view-annotations" data-post-id="<?php echo esc_attr($post->ID); ?>"><?php _e('View Annotations', 'sakurairo'); ?></a> |
+                                    <a href="<?php echo esc_url(get_edit_post_link($post->ID)); ?>" target="_blank"><?php _e('Edit Post', 'sakurairo'); ?></a> |
                                     <form method="post" style="display:inline;">
                                         <?php wp_nonce_field('iro_generate_annotations'); ?>
-                                        <input type="hidden" name="post_id" value="<?php echo $post->ID; ?>">
+                                        <input type="hidden" name="post_id" value="<?php echo esc_attr($post->ID); ?>">

--- render_summary_admin_page 中的相同模式 ---

@@ -655,2 +655,2 @@
-            <div class="notice notice-<?php echo $message_type; ?> is-dismissible">
-                <p><?php echo $message; ?></p>
+            <div class="notice notice-<?php echo esc_attr($message_type); ?> is-dismissible">
+                <p><?php echo wp_kses_post($message); ?></p>

@@ -688,1 +688,1 @@
-                                    <option value="<?php echo $p->ID; ?>"><?php echo $p->post_title; ?> (ID: <?php echo $p->ID; ?>)</option>
+                                    <option value="<?php echo esc_attr($p->ID); ?>"><?php echo esc_html($p->post_title); ?> (ID: <?php echo esc_html($p->ID); ?>)</option>

@@ -724,5 +724,5 @@
-                                    <a href="<?php echo get_permalink($post->ID); ?>" target="_blank">
-                                        <?php echo $post->post_title; ?>
+                                    <a href="<?php echo esc_url(get_permalink($post->ID)); ?>" target="_blank">
+                                        <?php echo esc_html($post->post_title); ?>
                                     </a>
                                 </td>
                                 <td><?php echo esc_html($preview); ?></td>
                                 <td>
-                                    <a href="#" class="view-summary" data-post-id="<?php echo $post->ID; ?>"><?php _e('View/Edit Summary', 'sakurairo'); ?></a> |
-                                    <a href="<?php echo get_edit_post_link($post->ID); ?>" target="_blank"><?php _e('Edit Post', 'sakurairo'); ?></a> |
+                                    <a href="#" class="view-summary" data-post-id="<?php echo esc_attr($post->ID); ?>"><?php _e('View/Edit Summary', 'sakurairo'); ?></a> |
+                                    <a href="<?php echo esc_url(get_edit_post_link($post->ID)); ?>" target="_blank"><?php _e('Edit Post', 'sakurairo'); ?></a> |
                                     <form method="post" style="display:inline;">
                                         <?php wp_nonce_field('iro_generate_summary'); ?>
-                                        <input type="hidden" name="post_id" value="<?php echo $post->ID; ?>">
+                                        <input type="hidden" name="post_id" value="<?php echo esc_attr($post->ID); ?>">
```

**管理页面 XSS 特别说明:**
管理页面 XSS 风险更高，因为攻击可影响管理员会话。即使 `current_user_can('manage_options')` 已检查，仍需转义输出——因为：
1. 选项值可能来自外部 API 返回（如 ChatGPT API）
2. 防御性编程原则：不信任任何数据源
3. `wp_kses_post()` 用于消息提示因为可能包含 HTML 链接

---

### Patch 06: inc/theme-plus.php 头图 URL 输出转义

**文件:** [inc/theme-plus.php](file:///workspace/inc/theme-plus.php)
**问题:** 头图 URL 直接输出到 `background-image` 和 `data-src`

```diff
--- a/inc/theme-plus.php
+++ b/inc/theme-plus.php
@@ -294,1 +294,1 @@
-    <div class="pattern-attachment bg lazyload" style="background-image: url(<?php echo iro_opt('load_out_svg'); ?>)" data-src="<?php echo $full_image_url; ?>"> </div>
+    <div class="pattern-attachment bg lazyload" style="background-image: url(<?php echo esc_url(iro_opt('load_out_svg')); ?>)" data-src="<?php echo esc_url($full_image_url); ?>"> </div>
```

**注意:** `esc_url()` 用于 CSS `url()` 上下文也是正确的，因为它会过滤掉 `javascript:`、`data:` 等危险协议。

---

## 审查结论

| Patch | 符合最佳实践 | 说明 |
|-------|-------------|------|
| 01 | ✅ | 属性用 esc_attr，正文用 esc_html |
| 02 | ✅ | 用户描述用 wp_kses_post 允许安全 HTML |
| 03 | ✅ | URL/属性/正文分别用对应函数 |
| 04 | ✅ | inner HTML 用 wp_kses_post 允许 img 标签 |
| 05 | ✅ | 管理页面全面转义，消息用 wp_kses_post |
| 06 | ✅ | URL 用 esc_url 过滤危险协议 |
