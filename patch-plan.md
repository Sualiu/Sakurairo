# PR #1407 Patch 分解清单

> 基于 PR: https://github.com/mirai-mamori/Sakurairo/pull/1407
> 本地基线: commit 24bc70b (已修复 SQL 注入)
> 已忽略最后一条 commit (78b69012) 中的 5 项修复:
>   1. dash-scheme.php 改用原生 PHP 函数替代 WP 函数
>   2. api.php upload_image() 上传文件存在性验证
>   3. api.php get_qq_avatar() type_2 二进制直接输出
>   4. functions.php get_the_user_ip() filter_var 验证
>   5. categories-images.php URL 存储改用 esc_url_raw()

---

## Patch 01: XSS 防护 — exhibition.php 展示页输出转义

**类型:** XSS 输出转义
**文件:** `exhibition.php`
**来源 Commit:** 5d7212d2

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

---

## Patch 02: XSS 防护 — author.php 作者页输出转义

**类型:** XSS 输出转义
**文件:** `author.php`
**来源 Commit:** 5d7212d2

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

---

## Patch 03: XSS 防护 — tpl/content-none.php 输出转义

**类型:** XSS 输出转义
**文件:** `tpl/content-none.php`
**来源 Commit:** 5d7212d2

```diff
--- a/tpl/content-none.php
+++ b/tpl/content-none.php
@@ -35,1 +35,1 @@
-				<li><a href="<?php echo get_permalink($postid); ?>" title="<?php echo $title ?>"><?php echo $title ?></a> </li>
+				<li><a href="<?php echo esc_url(get_permalink($postid)); ?>" title="<?php echo esc_attr($title); ?>"><?php echo esc_html($title); ?></a> </li>
```

---

## Patch 04: XSS 防护 — layouts/imgbox.php 社交图标区输出转义

**类型:** XSS 输出转义
**文件:** `layouts/imgbox.php`
**来源 Commit:** 5d7212d2

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

---

## Patch 05: XSS 防护 — inc/chatgpt/aigc-manage.php 管理页输出转义

**类型:** XSS 输出转义
**文件:** `inc/chatgpt/aigc-manage.php`
**来源 Commit:** 5d7212d2

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

---

## Patch 06: XSS 防护 — inc/theme-plus.php 头图 URL 输出转义

**类型:** XSS 输出转义
**文件:** `inc/theme-plus.php`
**来源 Commit:** 5d7212d2

```diff
--- a/inc/theme-plus.php
+++ b/inc/theme-plus.php
@@ -294,1 +294,1 @@
-    <div class="pattern-attachment bg lazyload" style="background-image: url(<?php echo iro_opt('load_out_svg'); ?>)" data-src="<?php echo $full_image_url; ?>"> </div>
+    <div class="pattern-attachment bg lazyload" style="background-image: url(<?php echo esc_url(iro_opt('load_out_svg')); ?>)" data-src="<?php echo esc_url($full_image_url); ?>"> </div>
```

---

## Patch 07: CSRF 防护 — AJAX 评论 nonce 验证

**类型:** CSRF nonce 验证
**文件:** `comments.php`, `inc/theme-plus.php`
**来源 Commit:** 5d7212d2

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

---

## Patch 08: 权限检查 — AIGC 管理表单加入 current_user_can

**类型:** 权限检查
**文件:** `inc/chatgpt/aigc-manage.php`
**来源 Commit:** 5d7212d2

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

---

## Patch 09: 权限检查 — 分类图片保存加入 current_user_can + wp_unslash

**类型:** 权限检查 + 输入验证
**文件:** `inc/categories-images.php`
**来源 Commit:** 5d7212d2
**注意:** 忽略最后 commit 中的 esc_url_raw() 替换，保留 sanitize_text_field

```diff
--- a/inc/categories-images.php
+++ b/inc/categories-images.php
@@ -147,2 +147,3 @@
 function z_save_taxonomy_image($term_id) {
-    if(isset($_POST['taxonomy_image']))
-        update_option('z_taxonomy_image'.$term_id, $_POST['taxonomy_image'], NULL);
+    if (isset($_POST['taxonomy_image']) && current_user_can('manage_categories')) {
+        update_option('z_taxonomy_image'.$term_id, sanitize_text_field(wp_unslash($_POST['taxonomy_image'])), NULL);
+    }
 }
```

---

## Patch 10: 权限检查 — 链接分类优先级保存加入 current_user_can

**类型:** 权限检查
**文件:** `inc/link-status.php`
**来源 Commit:** ca2390fe

```diff
--- a/inc/link-status.php
+++ b/inc/link-status.php
@@ -324,1 +324,1 @@
-    if (isset($_POST['term_priority'])) {
+    if (isset($_POST['term_priority']) && current_user_can('manage_categories')) {

@@ -330,1 +330,1 @@
-    if (isset($_POST['term_priority'])) {
+    if (isset($_POST['term_priority']) && current_user_can('manage_categories')) {
```

---

## Patch 11: CSS 注入防护 — dash-scheme.php 颜色值验证与 rules 过滤

**类型:** CSS 注入防护
**文件:** `inc/dash-scheme.php`
**来源 Commit:** 5d7212d2
**注意:** 忽略最后 commit 中的 strip_tags/stripslashes 替换（_get() 函数保持原样）

```diff
--- a/inc/dash-scheme.php
+++ b/inc/dash-scheme.php
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

---

## Patch 12: 输入验证 — inc/theme-plus.php 评论验证码参数净化 + 移除 WP 4.4 检查

**类型:** 输入验证 + 代码清理
**文件:** `inc/theme-plus.php`
**来源 Commit:** add745e8, 5d7212d2

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

---

## Patch 13: 输入验证 — search.php content_type 净化

**类型:** 输入验证
**文件:** `search.php`
**来源 Commit:** add745e8

```diff
--- a/search.php
+++ b/search.php
@@ -40,1 +40,1 @@
-    $content_types = isset($_GET['content_type']) ? explode(',', $_GET['content_type']) : $default_checked;
+    $content_types = isset($_GET['content_type']) ? array_map('sanitize_key', explode(',', wp_unslash($_GET['content_type']))) : $default_checked;
```

---

## Patch 14: 开放重定向修复 — wp_redirect → wp_safe_redirect

**类型:** 开放重定向
**文件:** `functions.php`, `inc/cache_settings.php`, `inc/classes/gallery.php`, `inc/api.php`
**来源 Commit:** ca2390fe, 5d7212d2, add745e8

```diff
--- a/functions.php
+++ b/functions.php
@@ -1769,1 +1769,1 @@
-            wp_redirect(get_option('siteurl') . '/wp-admin/profile.php');
+            wp_safe_redirect(admin_url('profile.php'));

@@ -4521,2 +4521,2 @@
-                wp_redirect(admin_url(), 302); //重载theme_folder_check_on_admin_init流程
+                wp_safe_redirect(admin_url(), 302); //重载theme_folder_check_on_admin_init流程
             } else {
-                wp_redirect(admin_url(), 302);
+                wp_safe_redirect(admin_url(), 302);

--- a/inc/cache_settings.php
+++ b/inc/cache_settings.php
@@ -138,1 +138,1 @@
-        wp_redirect(add_query_arg(['page' => 'sakurairo_cache_setting', 'updated' => 'true'], admin_url('admin.php')));
+        wp_safe_redirect(esc_url_raw(add_query_arg(['page' => 'sakurairo_cache_setting', 'updated' => 'true'], admin_url('admin.php'))));

--- a/inc/classes/gallery.php
+++ b/inc/classes/gallery.php
@@ -227,1 +227,1 @@
-            wp_redirect($random_image, 302);
+            wp_safe_redirect(esc_url_raw($random_image), 302);

--- a/inc/api.php
+++ b/inc/api.php
@@ -556,1 +556,1 @@
-            $response->header('Location', $data);
+            $response->header('Location', esc_url_raw($data));
```
> 注: api.php meting_aplayer 的 Location 修复。此 patch 仅包含 esc_url_raw 修复，
> meting_aplayer 的 nonce 绕过修复和 WP_REST_Request 重构分别在 Patch 18 和 Patch 17 中。

---

## Patch 15: SSRF 修复 — file_get_contents → wp_remote_get

**类型:** SSRF
**文件:** `functions.php`, `inc/article-highlight.php`
**来源 Commit:** be405f75, add745e8

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

---

## Patch 16: SSRF 修复 + 加解密 IV 不匹配 — inc/classes/QQ.php

**类型:** SSRF + Bug 修复
**文件:** `inc/classes/QQ.php`
**来源 Commit:** 716feb9c, be405f75

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
+        // 解码：urldecode → base64_decode → 分离 IV 和密文
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
+        // 验证解密结果为纯数字 QQ 号
+        if (!preg_match('/^\d{3,}$/', $qq_number)) {
+            return false;
+        }
+
+        return 'https://q2.qlogo.cn/headimg_dl?dst_uin=' . $qq_number . '&spec=100';
     }
```

---

## Patch 17: REST API 规范化 — 回调改用 WP_REST_Request + sakura_verify_rest_request_nonce

**类型:** REST API 规范化
**文件:** `inc/api.php`, `inc/classes/gallery.php`
**来源 Commit:** 33350a80, add745e8
**注意:** 此 patch 不包含 meting_aplayer 的 nonce 绕过修复（见 Patch 18），
      也不包含 get_qq_avatar 的输入验证修复（见 Patch 20），
      也不包含 upload_image 的文件验证（已忽略）。

```diff
--- a/inc/api.php
+++ b/inc/api.php
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

---

## Patch 18: REST API nonce 绕过修复 — meting_aplayer

**类型:** nonce 绕过修复
**文件:** `inc/api.php`
**来源 Commit:** be405f75
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

---

## Patch 19: IP 伪造防护 — get_the_user_ip 加固

**类型:** IP 伪造防护
**文件:** `functions.php`
**来源 Commit:** 2585b28f
**注意:** 忽略最后 commit 中的 filter_var(FILTER_VALIDATE_IP) 验证

```diff
--- a/functions.php
+++ b/functions.php
@@ -3611,14 +3611,11 @@
 function get_the_user_ip()
 {
-    // if (!empty($_SERVER['HTTP_CLIENT_IP'])) {
-    //     //check ip from share internet
-    //     $ip = $_SERVER['HTTP_CLIENT_IP'];
-    // } elseif (!empty($_SERVER['HTTP_X_FORWARDED_FOR'])) {
-    //     //to check ip is pass from proxy
-    //     $ip = $_SERVER['HTTP_X_FORWARDED_FOR'];
-    // } else {
-    //     $ip = $_SERVER['REMOTE_ADDR'];
-    // }
-    // 简略版
-    // $ip = $_SERVER['HTTP_CLIENT_IP'] ?: ($_SERVER['HTTP_X_FORWARDED_FOR'] ?: $_SERVER['REMOTE_ADDR']);
-    $ip = $_SERVER['HTTP_CLIENT_IP'] ?? $_SERVER['HTTP_X_FORWARDED_FOR'] ?? $_SERVER['REMOTE_ADDR'];
+    // CDN/反向代理场景：从 X-Forwarded-For 取最右侧（最接近源站）的 IP
+    // CDN 会将真实访客 IP 追加到链尾，取最右侧可抵抗在链首插入伪造 IP 的攻击
+    // 不信任 HTTP_CLIENT_IP（非标准头，纯伪造向量）
+    $ip = isset($_SERVER['REMOTE_ADDR']) ? $_SERVER['REMOTE_ADDR'] : '';
+    if (!empty($_SERVER['HTTP_X_FORWARDED_FOR'])) {
+        $forwarded_chain = explode(',', $_SERVER['HTTP_X_FORWARDED_FOR']);
+        $candidate = trim(end($forwarded_chain));
+        $ip = $candidate;
+    }

     return apply_filters('wpb_get_ip', $ip);
```

---

## Patch 20: 输入验证 + 重定向安全 — api.php get_qq_avatar

**类型:** 输入验证 + 重定向安全
**文件:** `inc/api.php`
**来源 Commit:** be405f75
**前置依赖:** Patch 17（函数签名已改为 WP_REST_Request）
**注意:** 忽略最后 commit 中的 type_2 二进制直接输出修复

```diff
--- a/inc/api.php
+++ b/inc/api.php
@@ -312,8 +312,10 @@
-function get_qq_avatar()
+function get_qq_avatar(WP_REST_Request $request)
 {
-    $encrypted = $_GET["qq"];
+    $encrypted = sanitize_text_field($request->get_param('qq'));
+    if (empty($encrypted)) {
+        return new WP_REST_Response(array('status' => 400, 'message' => 'Missing qq parameter'), 400);
+    }
     $imgurl = QQ::get_qq_avatar($encrypted);
+    if (!$imgurl) {
+        return new WP_REST_Response(array('status' => 404, 'message' => 'Avatar not found'), 404);
+    }
     if (iro_opt('qq_avatar_link') == 'type_2') {
-        $imgdata = file_get_contents($imgurl);
+        $imgdata = wp_remote_retrieve_body(wp_remote_get(esc_url_raw($imgurl)));
+        if (empty($imgdata)) {
+            return new WP_REST_Response(array('status' => 500, 'message' => 'Failed to fetch avatar'), 500);
+        }
         $response = new WP_REST_Response();
         $response->set_headers(
             array(
                 'Content-Type' => 'image/jpeg',
                 'Cache-Control' => 'max-age=86400'
             )
         );
         echo $imgdata;
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

## 附录: Patch 依赖关系

```
Patch 01-06: 互相独立，可并行应用
Patch 07:    独立
Patch 08:    独立（与 Patch 05 同文件但不同行）
Patch 09:    独立
Patch 10:    独立
Patch 11:    独立
Patch 12:    独立（与 Patch 06/07 同文件但不同行）
Patch 13:    独立
Patch 14:    独立
Patch 15:    独立
Patch 16:    独立
Patch 17:    独立（api.php + gallery.php REST 规范化）
Patch 18:    依赖 Patch 17（meting_aplayer 签名已改）
Patch 19:    独立
Patch 20:    依赖 Patch 17（get_qq_avatar 签名已改）
```

## 附录: 建议应用顺序

按风险等级从高到低排序：

1. **Patch 16** — QQ 加解密 IV 不匹配（功能完全失效的长期 Bug）
2. **Patch 07** — CSRF 评论 nonce（无认证即可提交评论）
3. **Patch 08** — AIGC 权限检查（低权限用户可执行管理操作）
4. **Patch 09** — 分类图片权限检查
5. **Patch 10** — 链接分类权限检查
6. **Patch 17** — REST API 规范化（前置依赖）
7. **Patch 18** — meting nonce 绕过（依赖 17）
8. **Patch 20** — get_qq_avatar 输入验证（依赖 17）
9. **Patch 15** — SSRF 修复
10. **Patch 14** — 开放重定向修复
11. **Patch 19** — IP 伪造防护
12. **Patch 11** — CSS 注入防护
13. **Patch 12** — 评论验证码输入净化
14. **Patch 13** — search.php 输入净化
15. **Patch 01-06** — XSS 输出转义（可并行）
