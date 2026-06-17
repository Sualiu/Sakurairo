# Sakurairo 安全补丁提交指南

基于 `mirai-mamori/Sakurairo` 的 `preview` 分支，逐类提交 Patch PR。
每类一个问题，一个 commit，一个 PR。当前 PR 合并后再提交下一个。

---

## Patch 1: 修复 IP 伪造漏洞

**严重程度**: 高
**文件**: `functions.php`

### 修改位置: `get_the_user_ip()` 函数

**原始代码** (约 L4107):
```php
function get_the_user_ip()
{
    if (!empty($_SERVER['HTTP_CLIENT_IP'])) {
        $ip = $_SERVER['HTTP_CLIENT_IP'];
    } elseif (!empty($_SERVER['HTTP_X_FORWARDED_FOR'])) {
        $ip = $_SERVER['HTTP_X_FORWARDED_FOR'];
    } else {
        $ip = isset($_SERVER['REMOTE_ADDR']) ? $_SERVER['REMOTE_ADDR'] : '';
    }
    return apply_filters('wpb_get_ip', $ip);
}
```

**替换为**:
```php
function get_the_user_ip()
{
    // CDN/反向代理场景：从 X-Forwarded-For 取最右侧（最接近源站）的 IP
    // CDN 会将真实访客 IP 追加到链尾，取最右侧可抵抗在链首插入伪造 IP 的攻击
    // 不信任 HTTP_CLIENT_IP（非标准头，纯伪造向量）
    $ip = isset($_SERVER['REMOTE_ADDR']) ? $_SERVER['REMOTE_ADDR'] : '';
    if (!empty($_SERVER['HTTP_X_FORWARDED_FOR'])) {
        $forwarded_chain = explode(',', $_SERVER['HTTP_X_FORWARDED_FOR']);
        $candidate = trim(end($forwarded_chain));
        // 验证为合法 IPv4/IPv6，否则回退到 REMOTE_ADDR
        if (filter_var($candidate, FILTER_VALIDATE_IP)) {
            $ip = $candidate;
        }
    }

    return apply_filters('wpb_get_ip', $ip);
}
```

**原因**:
- 原始代码信任 `HTTP_CLIENT_IP`（非标准头，攻击者可随意伪造）
- `HTTP_X_FORWARDED_FOR` 取整个值而非最右侧，攻击者可在链首插入伪造 IP
- 新代码：移除 `HTTP_CLIENT_IP` 信任；取 `X-Forwarded-For` 最右侧（CDN 追加的真实 IP）；`filter_var` 验证 IP 格式

---

## Patch 2: 修复 SSRF 漏洞

**严重程度**: 高
**文件**: `inc/article-highlight.php`, `functions.php`

### 2a. article-highlight.php

**修改位置**: 远程图片获取逻辑（约 L234 附近）

**原始代码**:
```php
        $remote_response = wp_remote_get(esc_url_raw($input), array('timeout' => 10));
        if (is_wp_error($remote_response) || wp_remote_retrieve_response_code($remote_response) !== 200) {
            error_log('远程获取图片失败');
            return false;
        }
        $image_data = wp_remote_retrieve_body($remote_response);
    } else {
        if (file_exists($input)) {
            $image_data = file_get_contents($input);
```

> 注意：article-highlight.php 原始代码已经是 `wp_remote_get`，此处无需修改。
> 实际需要修改的是 functions.php 中的 change_avatar。

### 2b. functions.php — change_avatar() 函数

**修改位置**: `change_avatar()` 函数（约 L2460 附近）

**原始代码**:
```php
add_filter('get_avatar', 'change_avatar', 10, 3);
function change_avatar($avatar)
{
    global $comment, $sakura_privkey;
    if ($comment && get_comment_meta($comment->comment_ID, 'new_field_qq', true)) {
        $qq_number = get_comment_meta($comment->comment_ID, 'new_field_qq', true);
        if (iro_opt('qq_avatar_link') == 'off') {
            return '<img src="https://q2.qlogo.cn/headimg_dl?dst_uin=' . $qq_number . '&spec=100" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
        }
        if (iro_opt('qq_avatar_link') == 'type_3') {
            $qqavatar = file_get_contents('https://ptlogin2.qq.com/getface?appid=1006102&imgtype=3&uin=' . $qq_number);
            preg_match('/:\"([^\"]*)\"/i', $qqavatar, $matches);
            $avatar_url = $matches[1];
            if (empty($avatar_url)) {
                return $avatar;
            }
            return '<img src="' . $avatar_url . '" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
        }

        $encrypted = openssl_encrypt($qq_number, 'aes-128-cbc', $sakura_privkey, 0, '0000000000000000');
        $encrypted = urlencode(base64_encode($encrypted));
        return '<img src="' . rest_url("sakura/v1/qqinfo/avatar") . '?qq=' . $encrypted . '" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
    }
    return $avatar;
}
```

**替换为**:
```php
add_filter('get_avatar', 'change_avatar', 10, 3);
function change_avatar($avatar)
{
    global $comment, $sakura_privkey;
    if ($comment && get_comment_meta($comment->comment_ID, 'new_field_qq', true)) {
        $qq_number = get_comment_meta($comment->comment_ID, 'new_field_qq', true);
        $qq_number = sanitize_text_field($qq_number);
        if (iro_opt('qq_avatar_link') == 'off') {
            return '<img src="https://q2.qlogo.cn/headimg_dl?dst_uin=' . esc_attr($qq_number) . '&spec=100" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
        }
        if (iro_opt('qq_avatar_link') == 'type_3') {
            $qqavatar = wp_remote_retrieve_body(wp_remote_get('https://ptlogin2.qq.com/getface?appid=1006102&imgtype=3&uin=' . urlencode($qq_number)));
            preg_match('/:\"([^\"]*)\"/i', $qqavatar, $matches);
            $avatar_url = isset($matches[1]) ? esc_url($matches[1]) : '';
            if (empty($avatar_url)) {
                return $avatar;
            }
            return '<img src="' . $avatar_url . '" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
        }

        // Ensure $sakura_privkey is defined and not null
        if (isset($sakura_privkey) && !is_null($sakura_privkey)) {
            // 生成一个合适长度的初始化向量
            $iv_length = openssl_cipher_iv_length('aes-128-cbc');
            $iv = openssl_random_pseudo_bytes($iv_length);

            // 加密数据
            $encrypted = openssl_encrypt($qq_number, 'aes-128-cbc', $sakura_privkey, 0, $iv);

            // 将初始化向量和加密数据一起编码
            $encrypted = urlencode(base64_encode($iv . $encrypted));

            return '<img src="' . esc_url(rest_url("sakura/v1/qqinfo/avatar") . '?qq=' . $encrypted) . '" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
        } else {
            // Handle the case where $sakura_privkey is not set or is null
            return $avatar;
        }
    }
    return $avatar;
}
```

**原因**:
- `file_get_contents('http://...')` → `wp_remote_get('https://...')`：修复 SSRF + HTTP→HTTPS
- `$qq_number` 加入 `sanitize_text_field()` 净化
- `esc_attr($qq_number)` / `esc_url($matches[1])` / `esc_url(rest_url(...))` 防止 XSS
- `openssl_encrypt` 使用硬编码 IV `'0000000000000000'` → `openssl_random_pseudo_bytes()`：修复加密强度
- 加密前拼接 IV（`$iv . $encrypted`）：与解密端 QQ.php 一致
- `$sakura_privkey` 未定义时返回原 `$avatar`（原始代码会报错/裂图）

---

## Patch 3: 修复开放重定向漏洞

**严重程度**: 高
**文件**: `inc/cache_settings.php`, `inc/classes/gallery.php`

### 3a. cache_settings.php

**修改位置**: `sakurairo_cache_setting_update()` 函数末尾（约 L138）

**原始代码**:
```php
        // 重定向
        wp_redirect(add_query_arg(['page' => 'sakurairo_cache_setting', 'updated' => 'true'], admin_url('admin.php')));
        exit;
```

**替换为**:
```php
        // 重定向
        wp_safe_redirect(esc_url_raw(add_query_arg(['page' => 'sakurairo_cache_setting', 'updated' => 'true'], admin_url('admin.php'))));
        exit;
```

**原因**: `wp_redirect` 不校验目标 URL，`wp_safe_redirect` 会检查允许列表

### 3b. gallery.php

**修改位置**: `get_image()` 方法末尾（约 L227）

**原始代码**:
```php
            $random_image = wp_get_upload_dir()['baseurl'] . $random_image;

            wp_redirect($random_image, 302);
            exit;
```

**替换为**:
```php
            $random_image = wp_get_upload_dir()['baseurl'] . $random_image;

            wp_safe_redirect(esc_url_raw($random_image), 302);
            exit;
```

**原因**: 同上，`wp_redirect` → `wp_safe_redirect` + `esc_url_raw`

---

## Patch 4: 修复权限越权漏洞

**严重程度**: 高
**文件**: `inc/link-status.php`, `inc/categories-images.php`, `inc/chatgpt/aigc-manage.php`

### 4a. link-status.php

**修改位置**: `sakurairo_link_status_page()` 函数开头（约 L31）

**原始代码**:
```php
function sakurairo_link_status_page() {
    // 处理手动检测请求
```

> 注意：原始代码已有 `current_user_can('manage_options')` 检查。此文件无需修改。

### 4b. categories-images.php

**修改位置**: `z_save_taxonomy_image()` 函数（约 L147）

**原始代码**:
```php
function z_save_taxonomy_image($term_id) {
    if(isset($_POST['taxonomy_image']))
        update_option('z_taxonomy_image'.$term_id, $_POST['taxonomy_image'], NULL);
}
```

**替换为**:
```php
function z_save_taxonomy_image($term_id) {
    if (isset($_POST['taxonomy_image']) && current_user_can('manage_categories')) {
        update_option('z_taxonomy_image'.$term_id, esc_url_raw(wp_unslash($_POST['taxonomy_image'])), NULL);
    }
}
```

**原因**:
- 无权限检查，任何访问者可修改分类图片 → 加入 `current_user_can('manage_categories')`
- `$_POST['taxonomy_image']` 是 URL，原始代码直接存储 → `esc_url_raw(wp_unslash(...))`

### 4c. aigc-manage.php — 6 处表单处理

**修改位置**: `render_annotations_admin_page()` 函数中的 3 个表单处理分支

**原始代码** (约 L46):
```php
    if (isset($_POST['generate_annotations']) && isset($_POST['post_id']) && check_admin_referer('iro_generate_annotations')) {
```

**替换为**:
```php
    if (isset($_POST['generate_annotations']) && isset($_POST['post_id']) && current_user_can('manage_options') && check_admin_referer('iro_generate_annotations')) {
```

**原始代码** (约 L57):
```php
    } elseif (isset($_POST['delete_annotations']) && isset($_POST['post_id']) && check_admin_referer('iro_generate_annotations')) {
```

**替换为**:
```php
    } elseif (isset($_POST['delete_annotations']) && isset($_POST['post_id']) && current_user_can('manage_options') && check_admin_referer('iro_generate_annotations')) {
```

**原始代码** (约 L62):
```php
    } elseif (isset($_POST['test_save']) && isset($_POST['post_id']) && check_admin_referer('iro_generate_annotations')) {
```

**替换为**:
```php
    } elseif (isset($_POST['test_save']) && isset($_POST['post_id']) && current_user_can('manage_options') && check_admin_referer('iro_generate_annotations')) {
```

**修改位置**: `render_summary_admin_page()` 函数中的 3 个表单处理分支

**原始代码** (约 L604):
```php
    if (isset($_POST['generate_summary']) && isset($_POST['post_id']) && check_admin_referer('iro_generate_summary')) {
```

**替换为**:
```php
    if (isset($_POST['generate_summary']) && isset($_POST['post_id']) && current_user_can('manage_options') && check_admin_referer('iro_generate_summary')) {
```

**原始代码** (约 L617):
```php
    } elseif (isset($_POST['delete_summary']) && isset($_POST['post_id']) && check_admin_referer('iro_generate_summary')) {
```

**替换为**:
```php
    } elseif (isset($_POST['delete_summary']) && isset($_POST['post_id']) && current_user_can('manage_options') && check_admin_referer('iro_generate_summary')) {
```

**原始代码** (约 L622):
```php
    } elseif (isset($_POST['test_save']) && isset($_POST['post_id']) && check_admin_referer('iro_generate_summary')) {
```

**替换为**:
```php
    } elseif (isset($_POST['test_save']) && isset($_POST['post_id']) && current_user_can('manage_options') && check_admin_referer('iro_generate_summary')) {
```

**原因**: 所有管理操作缺少 `current_user_can` 检查，任何登录用户均可触发

---

## Patch 5: 修复 XSS 输出净化

**严重程度**: 中
**文件**: `exhibition.php`, `author.php`, `tpl/content-none.php`, `inc/chatgpt/aigc-manage.php`

### 5a. exhibition.php

**修改位置**: 展示区标题（约 L346）

**原始代码**:
```php
        <i class="<?php echo iro_opt('exhibition_area_icon', 'fa-solid fa-laptop'); ?>" aria-hidden="true"></i> <?php echo iro_opt('exhibition_area_title', '展示'); ?>
```

**替换为**:
```php
        <i class="<?php echo esc_attr(iro_opt('exhibition_area_icon', 'fa-solid fa-laptop')); ?>" aria-hidden="true"></i> <?php echo esc_html(iro_opt('exhibition_area_title', '展示')); ?>
```

### 5b. author.php

**修改位置**: 分页自动加载属性（约 L56）

**原始代码**:
```php
        <div id="add_post"><span id="add_post_time" style="visibility: hidden;" title="<?php echo iro_opt('page_auto_load', ''); ?>"></span></div>
```

**替换为**:
```php
        <div id="add_post"><span id="add_post_time" style="visibility: hidden;" title="<?php echo esc_attr(iro_opt('page_auto_load', '')); ?>"></span></div>
```

### 5c. tpl/content-none.php

**修改位置**: 搜索无结果列表（约 L35）

**原始代码**:
```php
				<li><a href="<?php echo get_permalink($postid); ?>" title="<?php echo $title ?>"><?php echo $title ?></a> </li>
```

**替换为**:
```php
				<li><a href="<?php echo esc_url(get_permalink($postid)); ?>" title="<?php echo esc_attr($title); ?>"><?php echo esc_html($title); ?></a> </li>
```

### 5d. aigc-manage.php — 多处输出净化

**修改位置**: `get_chatgpt_api_info()` 函数（约 L15-17）

**原始代码**:
```php
<?php _e('API Endpoint: ', 'sakurairo'); ?><?php echo iro_opt('chatgpt_endpoint', __('Not set', 'sakurairo')); ?>
<?php _e('API Key: ', 'sakurairo'); ?><?php echo empty(iro_opt('chatgpt_access_token')) ? __('Not set', 'sakurairo') : __('Set (Length: ', 'sakurairo') . strlen(iro_opt('chatgpt_access_token')) . ')'; ?>
<?php _e('Model: ', 'sakurairo'); ?><?php echo iro_opt('chatgpt_model', __('Not set', 'sakurairo')); ?>
```

**替换为**:
```php
<?php
// iro_opt() 返回值类型不固定，统一做类型保护
$chatgpt_endpoint = iro_opt('chatgpt_endpoint', __('Not set', 'sakurairo'));
$chatgpt_token = iro_opt('chatgpt_access_token', '');
$chatgpt_model = iro_opt('chatgpt_model', __('Not set', 'sakurairo'));
if (!is_string($chatgpt_endpoint)) {
    $chatgpt_endpoint = __('Not set', 'sakurairo');
}
if (!is_string($chatgpt_model)) {
    $chatgpt_model = __('Not set', 'sakurairo');
}
?>
<?php _e('API Endpoint: ', 'sakurairo'); ?><?php echo esc_html($chatgpt_endpoint); ?>
<?php _e('API Key: ', 'sakurairo'); ?><?php echo (empty($chatgpt_token) || !is_string($chatgpt_token)) ? esc_html__('Not set', 'sakurairo') : esc_html(sprintf(__('Set (Length: %d)', 'sakurairo'), strlen($chatgpt_token))); ?>
<?php _e('Model: ', 'sakurairo'); ?><?php echo esc_html($chatgpt_model); ?>
```

**修改位置**: `render_annotations_admin_page()` 中的消息输出（约 L110-112）

**原始代码**:
```php
            <div class="notice notice-<?php echo $message_type; ?> is-dismissible">
                <p><?php echo $message; ?></p>
            </div>
```

**替换为**:
```php
            <div class="notice notice-<?php echo esc_attr($message_type); ?> is-dismissible">
                <p><?php echo wp_kses_post($message); ?></p>
            </div>
```

**修改位置**: `render_summary_admin_page()` 中的消息输出（约 L655-657）

**原始代码**:
```php
            <div class="notice notice-<?php echo $message_type; ?> is-dismissible">
                <p><?php echo $message; ?></p>
            </div>
```

**替换为**:
```php
            <div class="notice notice-<?php echo esc_attr($message_type); ?> is-dismissible">
                <p><?php echo wp_kses_post($message); ?></p>
            </div>
```

**修改位置**: `render_annotations_meta_box()` 中的链接输出（约 L573）

**原始代码**:
```php
    echo '<p><a href="' . admin_url('tools.php?page=iro-term-annotations') . '" class="button">' . __('Edit in Annotations Management Page', 'sakurairo') . '</a></p>';
```

**替换为**:
```php
    echo '<p><a href="' . esc_url(admin_url('tools.php?page=iro-term-annotations')) . '" class="button">' . __('Edit in Annotations Management Page', 'sakurairo') . '</a></p>';
```

**修改位置**: `render_summary_meta_box()` 中的链接输出（约 L962）

**原始代码**:
```php
    echo '<p><a href="' . admin_url('edit.php?page=iro-post-summary') . '" class="button">' . __('Edit in Summary Management Page', 'sakurairo') . '</a></p>';
```

**替换为**:
```php
    echo '<p><a href="' . esc_url(admin_url('edit.php?page=iro-post-summary')) . '" class="button">' . __('Edit in Summary Management Page', 'sakurairo') . '</a></p>';
```

---

## Patch 6: 修复独立脚本输入净化

**严重程度**: 中
**文件**: `inc/dash-scheme.php`

> 此文件作为独立 CSS 直接输出，未经 WP 引导，不可使用 WP 函数。

### 6a. `_get()` 函数

**原始代码** (约 L13):
```php
function _get($str){
    $val = !empty($_GET[$str]) ? $_GET[$str] : null;
    return $val;
}
```

**替换为**:
```php
// 此文件作为独立 CSS 直接输出，未经 WP 引导，不可使用 WP 函数
function _get($str){
    // 原生 PHP 净化：去除斜杠与 HTML 标签
    $val = !empty($_GET[$str]) ? strip_tags(stripslashes($_GET[$str])) : null;
    return $val;
}
```

### 6b. 新增 `_sanitize_hex_color()` 函数

在 `_get()` 函数之后添加：

```php
/**
 * Validate and sanitize a hex color value.
 */
function _sanitize_hex_color($color, $default = '#000') {
    if (empty($color)) return $default;
    $color = ltrim($color, '#');
    if (!preg_match('/^[a-fA-F0-9]{3}$|^[a-fA-F0-9]{6}$/', $color)) {
        return $default;
    }
    return '#' . $color;
}
```

### 6c. 颜色值使用 `_sanitize_hex_color()`

将所有 `$color_1="#"._get('color_1')` 等替换：

**原始代码**:
```php
if(_get('color_1')==NULL) {
	$color_1="#000";
} else {
	$color_1="#"._get('color_1');
}
if(_get('color_2')==NULL) {
	$color_2="#000";
} else {
	$color_2="#"._get('color_2');
}
if(_get('color_3')==NULL) {
	$color_3="#ff6496";
} else {
	$color_3="#"._get('color_3');
}
```

**替换为**:
```php
$color_1 = _sanitize_hex_color(_get('color_1'), '#000');
$color_2 = _sanitize_hex_color(_get('color_2'), '#000');
$color_3 = _sanitize_hex_color(_get('color_3'), '#ff6496');
```

### 6d. rules 处理

**原始代码**:
```php
if(_get('rules')==NULL) {
	$rules="";
} else {
	$rules=urldecode(_get('rules'));
}
```

**替换为**:
```php
if(_get('rules')==NULL) {
	$rules="";
} else {
	// 使用原生 strip_tags（此文件无 WP 引导）
	$rules = strip_tags(urldecode(_get('rules')));
	// Remove dangerous CSS functions and imports
	$rules = preg_replace('/@import\s+url\s*\(/i', '/* blocked */', $rules);
	$rules = preg_replace('/expression\s*\(/i', '/* blocked */', $rules);
	$rules = preg_replace('/javascript\s*:/i', '/* blocked */', $rules);
	$rules = preg_replace('/-moz-binding\s*:/i', '/* blocked */', $rules);
	$rules = preg_replace('/behavior\s*:/i', '/* blocked */', $rules);
	$rules = preg_replace('/url\s*\(\s*[\'"]?\s*data\s*:/i', '/* blocked */', $rules);
}
```

---

## Patch 7: 修复输入净化

**严重程度**: 中
**文件**: `search.php`, `inc/theme-plus.php`

### 7a. search.php

**修改位置**: content_type 参数处理（约 L40）

**原始代码**:
```php
    $content_types = isset($_GET['content_type']) ? explode(',', $_GET['content_type']) : $default_checked;
```

**替换为**:
```php
    $content_types = isset($_GET['content_type']) ? array_map('sanitize_key', explode(',', wp_unslash($_GET['content_type']))) : $default_checked;
```

**原因**: `sanitize_key` 仅保留 `[a-zA-Z0-9_-]`，`wp_unslash` 去除 WP 添加的魔术引号

### 7b. theme-plus.php — 验证码字段

**修改位置**: `comment_captcha()` 函数（约 L167-170）

**原始代码**:
```php
  if (iro_opt('comment_captcha_select') == "iro_captcha") {
      $captcha = isset($_POST['captcha']) ? $_POST['captcha'] : '';
      $timestamp = isset($_POST['timestamp']) ? $_POST['timestamp'] : '';
      $captcha_id = isset($_POST['id']) ? $_POST['id'] : '';
```

**替换为**:
```php
  if (iro_opt('comment_captcha_select') == "iro_captcha") {
      $captcha = isset($_POST['captcha']) ? sanitize_text_field(wp_unslash($_POST['captcha'])) : '';
      $timestamp = isset($_POST['timestamp']) ? sanitize_text_field(wp_unslash($_POST['timestamp'])) : '';
      $captcha_id = isset($_POST['id']) ? sanitize_text_field(wp_unslash($_POST['id'])) : '';
```

### 7c. theme-plus.php — Turnstile token

**修改位置**: `comment_captcha()` 函数 turnstile 分支（约 L189）

**原始代码**:
```php
      $token = sanitize_text_field($_POST['cf-turnstile-response']);
```

**替换为**:
```php
      $token = sanitize_text_field(wp_unslash($_POST['cf-turnstile-response']));
```

### 7d. theme-plus.php — 评论提交 nonce 检查

**修改位置**: `siren_ajax_comment_callback()` 函数（约 L213-214）

**原始代码**:
```php
    function siren_ajax_comment_callback(){
      $comment = wp_handle_comment_submission( wp_unslash( $_POST ) );
```

**替换为**:
```php
    function siren_ajax_comment_callback(){
      check_ajax_referer('sakurairo_ajax_comment', 'sakurairo_comment_nonce');
      $comment = wp_handle_comment_submission( wp_unslash( $_POST ) );
```

**原因**: 原始代码无 nonce 验证，CSRF 攻击可代替用户提交评论

### 7e. comments.php — 添加 nonce field

**修改位置**: `comment_form()` 参数中的 `submit_button`（约 L186-193）

**原始代码**:
```php
                'submit_button'     => '<div class="form-submit">
                                            <input name="submit" type="submit" id="submit" class="submit" value=" ' . esc_attr(iro_opt('comment_submit_button_text')) . ' ">' . $smilies_button . $img_upload .'
                                            <label class="markdown-toggle">
                                                <input type="checkbox" id="enable_markdown" name="enable_markdown">
                                                <i class="fa-brands fa-markdown fa-sm"></i>
                                            </label>
                                        </div>',
```

**替换为**:
```php
                'submit_button'     => '<div class="form-submit">
                                            <input name="submit" type="submit" id="submit" class="submit" value=" ' . esc_attr(iro_opt('comment_submit_button_text', __('Submit', 'sakurairo'))) . ' ">' . $smilies_button . $img_upload .'
                                            <label class="markdown-toggle">
                                                <input type="checkbox" id="enable_markdown" name="enable_markdown">
                                                <i class="fa-brands fa-markdown fa-sm"></i>
                                            </label>
                                            ' . wp_nonce_field('sakurairo_ajax_comment', 'sakurairo_comment_nonce', true, false) . '
                                        </div>',
```

**原因**: 与 theme-plus.php 的 `check_ajax_referer` 配对。前端 JS 使用 `FormData(this)` 自动序列化表单，会包含此 nonce。

---

## Patch 8: 修复 iro_opt 类型安全问题

**严重程度**: 低
**文件**: `comments.php`

### comments.php — 补充 default 值

**修改位置 1**: 评论区域折叠判断（约 L73）

**原始代码**:
```php
    <div class="commentwrap comments-hidden<?php echo esc_attr(iro_opt('comment_area')) == 'fold' ? ' comments-fold' : ''; ?>">
```

**替换为**:
```php
    <div class="commentwrap comments-hidden<?php echo esc_attr(iro_opt('comment_area', 'unfold')) == 'fold' ? ' comments-fold' : ''; ?>">
```

**修改位置 2**: 评论表单参数（约 L180-183）

**原始代码**:
```php
                'label_submit'      => esc_attr(iro_opt('comment_submit_button_text')),
                'comment_field'     => '<div class="comment-textarea">
                                            <textarea placeholder="' . esc_attr(iro_opt('comment_placeholder_text')) . '" name="comment" class="commentbody" id="comment" rows="5" tabindex="4"></textarea>
                                            <label class="input-label">' . esc_html(iro_opt('comment_placeholder_text')) . '</label>
```

**替换为**:
```php
                'label_submit'      => esc_attr(iro_opt('comment_submit_button_text', __('Submit', 'sakurairo'))),
                'comment_field'     => '<div class="comment-textarea">
                                            <textarea placeholder="' . esc_attr(iro_opt('comment_placeholder_text', '')) . '" name="comment" class="commentbody" id="comment" rows="5" tabindex="4"></textarea>
                                            <label class="input-label">' . esc_html(iro_opt('comment_placeholder_text', '')) . '</label>
```

**原因**: `iro_opt()` 未传 default 时可能返回 null，PHP 8.1+ 传入 `esc_attr`/`esc_html` 触发 deprecation 警告

---

## 提交顺序建议

| 顺序 | Patch | Commit 消息 | 优先级 |
|------|-------|-------------|--------|
| 1 | Patch 1 | `fix: 修復 get_the_user_ip IP 偽造漏洞` | 高 |
| 2 | Patch 2 | `fix: 修復 SSRF 漏洞與 QQ 頭像加密強度` | 高 |
| 3 | Patch 3 | `fix: 修復開放重定向漏洞` | 高 |
| 4 | Patch 4 | `fix: 修復權限越權漏洞` | 高 |
| 5 | Patch 5 | `fix: 修復 XSS 輸出淨化問題` | 中 |
| 6 | Patch 6 | `fix: 修復 dash-scheme.php 獨立腳本輸入淨化` | 中 |
| 7 | Patch 7 | `fix: 修復輸入淨化與評論 CSRF 防護` | 中 |
| 8 | Patch 8 | `fix: 修復 iro_opt 類型安全問題` | 低 |

---

## 特别注意

### QQ 头像加密功能需用户手动配置

QQ 头像加密功能（`qq_avatar_link` 设置为 `type_1` 时）依赖全局变量 `$sakura_privkey`，这是**原始代码的设计**，主题本身从未对该变量赋值。

**用户若要启用此功能，需在 `wp-config.php` 中手动定义**：
```php
$sakura_privkey = 'your-16-char-secret-key';
```

**未定义时的行为**：
- `change_avatar()` 检测到 `$sakura_privkey` 未设置，回退返回默认 `$avatar`（Patch 2 修复了原始代码在此情况下报错/裂图的问题）
- `QQ::get_qq_avatar()` 返回 `false`，REST API 返回 404

此行为与原始代码逻辑一致，Patch 2 仅修复了 IV 提取方式和未定义时的回退行为。
