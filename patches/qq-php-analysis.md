# QQ.php 深度安全分析报告

> 审查对象: inc/classes/QQ.php + 调用链（functions.php change_avatar、inc/api.php）
> 审查依据: WordPress Skills 最佳实践
> - wp-plugin-development/references/security.md
> - wp-rest-api/references/routes-and-endpoints.md
> - wp-rest-api/references/authentication.md
> - wp-plugin-development/references/data-and-cron.md

---

## 一、完整调用链梳理

```
用户评论 → 保存 QQ 号到 comment meta (new_field_qq)
    ↓
get_avatar filter → change_avatar()
    ├── type_off:  直接拼接 QQ 号到 qlogo.cn URL
    ├── type_3:   file_get_contents(ptlogin2.qq.com) → 正则提取头像 URL
    └── type_2:   openssl_encrypt(QQ号) → 输出 <img src="REST API URL?qq=密文">
                        ↓
                  REST: /sakura/v1/qqinfo/avatar?qq=密文
                        ↓
                  get_qq_avatar() → QQ::get_qq_avatar(密文)
                        ↓
                  openssl_decrypt(密文) → 拼接 qlogo.cn URL
                        ├── type_2: file_get_contents(qlogo.cn) → 输出二进制图片
                        └── 其他: 301 重定向到 qlogo.cn URL

用户输入 QQ 号 → REST: /sakura/v1/qqinfo/json?qq=QQ号
                        ↓
                  get_qq_info() → QQ::get_qq_info(QQ号)
                        ↓
                  file_get_contents(api.qjqq.cn) → 返回 QQ 昵称+头像
```

---

## 二、逐行问题分析

### 问题 1: 🔴 SSRF — file_get_contents 发起外部 HTTP 请求

**位置:** QQ.php:8, QQ.php:37（间接）, api.php:317, functions.php:2425

```php
// QQ.php:8 — get_qq_info()
$get_info = file_get_contents('https://api.qjqq.cn/api/qqinfo?qq=' . $qq);

// api.php:317 — get_qq_avatar() type_2 分支
$imgdata = file_get_contents($imgurl);

// functions.php:2425 — change_avatar() type_3 分支
$qqavatar = file_get_contents('http://ptlogin2.qq.com/getface?appid=1006102&imgtype=3&uin=' . $qq_number);
```

**WordPress Skills 违规:**
- `wp-plugin-development/references/security.md`: "sanitize/validate on input, escape on output"
- `wp-plugin-development/references/data-and-cron.md`: "avoid building queries with concatenated user input"

**危害:**
1. `file_get_contents()` 无超时控制 → 恶意 URL 可导致请求挂起，DoS
2. `file_get_contents('file:///etc/passwd')` 可读取本地文件（SSRF）
3. `file_get_contents('http://...')` 明文传输，可被中间人攻击
4. QQ 号直接拼接到 URL，未经 `urlencode()` → URL 注入

**修复:** 全部改用 `wp_remote_get()` + `is_wp_error()` 检查 + 超时设置

---

### 问题 2: 🔴 加解密 IV 不匹配 — 功能完全失效

**位置:** QQ.php:33-35（解密端） vs functions.php:2434-2440（加密端）

**加密端 (functions.php:2434-2440):**
```php
$iv = openssl_random_pseudo_bytes($iv_length);       // 随机 IV（16 字节）
$encrypted = openssl_encrypt($qq_number, 'aes-128-cbc', $sakura_privkey, 0, $iv);
$encrypted = urlencode(base64_encode($iv . $encrypted)); // IV 拼接在密文前
```

**解密端 (QQ.php:33-35):**
```php
$iv = str_repeat($sakura_privkey, 2);                // 固定 IV（错误！）
$encrypted = base64_decode(urldecode($encrypted));    // 整个 base64 解码（包含 IV 前缀）
$qq_number = openssl_decrypt($encrypted, 'aes-128-cbc', $sakura_privkey, 0, $iv);
```

**问题:**
1. 加密端用随机 IV，解密端用固定 IV → **解密永远失败** → QQ 头像 type_2 功能完全失效
2. 解密端未从密文中分离 IV → 整个 base64_decode 结果（包含 IV 前缀）被当作密文
3. 这是长期存在的 Bug，意味着 type_2 模式从未正常工作过

**修复:** 从密文前 16 字节提取 IV，与加密端一致

---

### 问题 3: 🔴 $sakura_privkey 未定义 — 全局变量无来源

**位置:** QQ.php:31, functions.php:2418

```php
global $sakura_privkey;
```

**问题:**
- 全局搜索代码库，`$sakura_privkey` **从未被赋值**
- 仅在 `functions.php:2431` 有 `if (isset($sakura_privkey) && !is_null($sakura_privkey))` 检查
- 但 QQ.php:31 的 `global $sakura_privkey` 之后直接使用，**无 isset 检查**
- 如果 `$sakura_privkey` 未定义，`str_repeat(null, 2)` 返回空字符串 → IV 为空 → 解密失败
- `openssl_decrypt(..., '', 0, '')` 可能返回 false 或产生 warning

**WordPress Skills 违规:**
- `wp-plugin-development/references/security.md`: "sanitize/validate on input"
- 密钥是加解密的核心输入，必须验证其存在性和有效性

**修复:** 添加 `!isset($sakura_privkey)` 检查，密钥不存在时返回 false

---

### 问题 4: 🔴 未定义索引 — preg_match 失败后访问 $matches[0]

**位置:** QQ.php:36-37

```php
preg_match('/^\d{3,}$/', $qq_number, $matches);
return 'https://q2.qlogo.cn/headimg_dl?dst_uin=' . $matches[0] . '&spec=100';
```

**问题:**
1. `openssl_decrypt` 失败时返回 `false`，`preg_match('/^\d{3,}$/', false, $matches)` 不匹配 → `$matches` 为空数组
2. `$matches[0]` 访问空数组 → **PHP Notice: Undefined offset: 0**
3. 拼接后 URL 变成 `https://q2.qlogo.cn/headimg_dl?dst_uin=&spec=100` → 请求无效 QQ 号
4. 即使解密成功但 QQ 号不匹配正则（如包含字母），`$matches[0]` 同样未定义

**修复:** 先检查 `openssl_decrypt` 返回值，再验证 QQ 号格式，失败时返回 false

---

### 问题 5: 🟠 $output 未定义 — get_qq_info 逻辑分支不完整

**位置:** QQ.php:10-27

```php
if ($name) {
    if ($name['code'] == 200){
        $output = array(...);
    }
    // ← 如果 $name['code'] != 200，$output 未定义！
} else {
    $output = array(...);
}
return $output;  // ← $output 可能未定义
```

**问题:**
1. 如果 API 返回有效 JSON 但 `code != 200`，`$output` 未定义 → PHP Notice + 返回 null
2. 调用方 `api.php:303` 执行 `$output['status']` → null 数组访问 → Fatal Error
3. `$name['code']` 和 `$name['name']` 无 `isset` 检查 → 未定义索引 Notice

**修复:** 补全所有分支，添加 isset 检查

---

### 问题 6: 🟠 URL 参数注入 — QQ 号未 urlencode

**位置:** QQ.php:8, QQ.php:16, QQ.php:37, functions.php:2422, functions.php:2425

```php
// QQ.php:8
$get_info = file_get_contents('https://api.qjqq.cn/api/qqinfo?qq=' . $qq);

// QQ.php:16
'avatar' => 'https://q2.qlogo.cn/headimg_dl?dst_uin=' . $qq . '&spec=100',

// QQ.php:37
return 'https://q2.qlogo.cn/headimg_dl?dst_uin=' . $matches[0] . '&spec=100';

// functions.php:2422
return '<img src="https://q2.qlogo.cn/headimg_dl?dst_uin=' . $qq_number . '&spec=100" ...';

// functions.php:2425
$qqavatar = file_get_contents('http://ptlogin2.qq.com/getface?appid=1006102&imgtype=3&uin=' . $qq_number);
```

**WordPress Skills 违规:**
- `wp-plugin-development/references/security.md`: "sanitize/validate on input"
- 用户输入拼接到 URL 前必须净化

**攻击向量:**
- QQ 号输入 `123&spec=1` → 修改 spec 参数
- QQ 号输入 `123#<script>alert(1)</script>` → URL 片段注入
- QQ 号输入 `../../etc/passwd` → 路径遍历（取决于 API 实现）

**修复:** 所有 QQ 号拼接到 URL 前使用 `urlencode()`

---

### 问题 7: 🟠 REST API 回调直接访问 $_GET

**位置:** api.php:292-293, api.php:314

```php
// api.php:292 — get_qq_info()
} elseif ($_GET['qq']) {
    $qq = $_GET['qq'];

// api.php:314 — get_qq_avatar()
$encrypted = $_GET["qq"];
```

**WordPress Skills 违规:**
- `wp-rest-api/references/routes-and-endpoints.md`: "Access params via the `WP_REST_Request` object, not `$_GET`/`$_POST`."

**问题:**
1. 直接访问 `$_GET` 绕过 REST API 的参数验证和净化机制
2. `get_qq_info()` 已声明 `WP_REST_Request $request` 参数但未使用
3. `get_qq_avatar()` 甚至未声明 `$request` 参数
4. 无 `sanitize_text_field()` 净化

**修复:** 改用 `$request->get_param('qq')` + `sanitize_text_field()`

---

### 问题 8: 🟠 REST 路由无 args schema 验证

**位置:** api.php:57-67

```php
register_rest_route('sakura/v1', '/qqinfo/json', array(
    'methods' => 'GET',
    'callback' => 'get_qq_info',
    'permission_callback' => '__return_true'
    // ← 缺少 args 定义
));

register_rest_route('sakura/v1', '/qqinfo/avatar', array(
    'methods' => 'GET',
    'callback' => 'get_qq_avatar',
    'permission_callback' => '__return_true'
    // ← 缺少 args 定义
));
```

**WordPress Skills 违规:**
- `wp-rest-api/references/routes-and-endpoints.md`: "Register `args` to validate and sanitize inputs. Use `type`, `required`, `default`, `validate_callback`, `sanitize_callback`."
- `wp-rest-api/references/schema.md`: "Use `rest_validate_value_from_schema()` then `rest_sanitize_value_from_schema()`"

**修复建议:**
```php
register_rest_route('sakura/v1', '/qqinfo/json', array(
    'methods' => 'GET',
    'callback' => 'get_qq_info',
    'permission_callback' => '__return_true',
    'args' => array(
        'qq' => array(
            'type' => 'string',
            'required' => true,
            'sanitize_callback' => 'sanitize_text_field',
            'validate_callback' => function($value) {
                return preg_match('/^\d{3,}$/', $value);
            },
        ),
    ),
));

register_rest_route('sakura/v1', '/qqinfo/avatar', array(
    'methods' => 'GET',
    'callback' => 'get_qq_avatar',
    'permission_callback' => '__return_true',
    'args' => array(
        'qq' => array(
            'type' => 'string',
            'required' => true,
            'sanitize_callback' => 'sanitize_text_field',
        ),
    ),
));
```

---

### 问题 9: 🟡 XSS — QQ 号直接输出到 HTML 属性

**位置:** functions.php:2422, functions.php:2442

```php
// functions.php:2422 — type_off
return '<img src="https://q2.qlogo.cn/headimg_dl?dst_uin=' . $qq_number . '&spec=100" ...';

// functions.php:2442 — type_2
return '<img src="' . rest_url("sakura/v1/qqinfo/avatar") . '?qq=' . $encrypted . '" ...';
```

**WordPress Skills 违规:**
- `wp-plugin-development/references/security.md`: "escape on output"
- QQ 号来自 comment meta（用户输入），输出到 HTML `src` 属性必须转义

**修复:**
- `src` 属性中的 URL 用 `esc_url()`
- `$qq_number` 用 `esc_attr()` 或 `urlencode()`

---

### 问题 10: 🟡 加密密钥管理不当

**位置:** 全局 `$sakura_privkey` 变量

**问题:**
1. **密钥来源不明** — 全局搜索未找到赋值位置，可能来自外部配置或已删除的代码
2. **使用 global 声明** — 密钥作为全局变量，任何代码都可读取或修改
3. **无密钥轮换机制** — 密钥固定后无法更换（更换会导致所有已加密的 QQ 号无法解密）
4. **密钥长度未验证** — AES-128-CBC 要求 16 字节密钥，但 `$sakura_privkey` 长度未验证

**WordPress Skills 最佳实践:**
- `wp-plugin-development/references/data-and-cron.md`: "Prefer Options API for small config/state"
- 密钥应存储在 WordPress options 中，通过 `get_option()` 读取
- 应添加密钥长度验证

**修复建议:**
```php
// 从 options 读取密钥
$sakura_privkey = get_option('sakura_encryption_key');

// 验证密钥长度
if (strlen($sakura_privkey) !== 16) {
    return false;
}
```

---

### 问题 11: 🟡 AES-CBC 模式无认证 — 密文可被篡改

**位置:** QQ.php:35, functions.php:2437

```php
openssl_encrypt($qq_number, 'aes-128-cbc', $sakura_privkey, 0, $iv);
openssl_decrypt($encrypted, 'aes-128-cbc', $sakura_privkey, 0, $iv);
```

**问题:**
- AES-CBC 模式不提供完整性验证（无 HMAC/MAC）
- 攻击者可修改密文（如翻转某些位），解密后得到不同的 QQ 号
- 虽然 `preg_match('/^\d{3,}$/', $qq_number)` 提供了一定的格式验证，但无法保证解密结果与原始加密值一致

**修复建议（长期）:**
- 改用 AES-256-GCM（提供认证加密）
- 或在密文后附加 HMAC-SHA256 签名

---

### 问题 12: 🟢 get_qq_info 返回值类型不一致

**位置:** QQ.php:7-28

```php
public static function get_qq_info($qq) {
    // ...
    if ($name) {
        if ($name['code'] == 200){
            $output = array(...);  // 成功时返回 array
        }
        // 失败时 $output 未定义
    } else {
        $output = array(...);      // 失败时返回 array
    }
    return $output;  // 可能返回 null（未定义变量）
}
```

**问题:** 函数声明返回 `array`，但可能返回 `null`。调用方 `api.php:303` 执行 `$output['status']` 时，如果 `$output` 为 null 则 Fatal Error。

---

## 三、问题汇总（按严重程度排序）

| # | 严重程度 | 问题 | 位置 | WordPress Skills 违规 |
|---|---------|------|------|---------------------|
| 1 | 🔴 严重 | SSRF: file_get_contents 发起 HTTP 请求 | QQ.php:8, api.php:317, functions.php:2425 | security.md: sanitize on input |
| 2 | 🔴 严重 | 加解密 IV 不匹配，功能完全失效 | QQ.php:33-35 vs functions.php:2434-2440 | data-and-cron.md: 数据一致性 |
| 3 | 🔴 严重 | $sakura_privkey 未定义，无 isset 检查 | QQ.php:31 | security.md: validate on input |
| 4 | 🔴 严重 | preg_match 失败后访问 $matches[0] | QQ.php:36-37 | 类型安全 |
| 5 | 🟠 高 | $output 未定义，逻辑分支不完整 | QQ.php:10-27 | 类型安全 |
| 6 | 🟠 高 | URL 参数注入: QQ 号未 urlencode | QQ.php:8,16,37, functions.php:2422,2425 | security.md: sanitize on input |
| 7 | 🟠 高 | REST 回调直接访问 $_GET | api.php:292-293, api.php:314 | routes-and-endpoints.md: use WP_REST_Request |
| 8 | 🟠 高 | REST 路由无 args schema | api.php:57-67 | routes-and-endpoints.md: register args |
| 9 | 🟡 中 | XSS: QQ 号直接输出到 HTML | functions.php:2422,2442 | security.md: escape on output |
| 10 | 🟡 中 | 加密密钥管理不当 | 全局 $sakura_privkey | data-and-cron.md: Options API |
| 11 | 🟡 中 | AES-CBC 无认证加密 | QQ.php:35, functions.php:2437 | 安全最佳实践 |
| 12 | 🟢 轻微 | 返回值类型不一致 | QQ.php:7-28 | 类型安全 |

---

## 四、完整修复方案

```php
<?php

namespace Sakura\API;

class QQ
{
    /**
     * Get QQ user info from external API.
     *
     * @param string $qq QQ number (digits only, 3+ chars)
     * @return array{status: int, success: bool, message: string, avatar?: string, name?: string}
     */
    public static function get_qq_info($qq) {
        // 1. 输入验证
        $qq = sanitize_text_field($qq);
        if (empty($qq) || !preg_match('/^\d{3,}$/', $qq)) {
            return array(
                'status' => 400,
                'success' => false,
                'message' => 'Invalid QQ number format.'
            );
        }

        // 2. 使用 WP HTTP API 替代 file_get_contents
        $response = wp_remote_get(
            'https://api.qjqq.cn/api/qqinfo?qq=' . urlencode($qq),
            array('timeout' => 5)
        );

        // 3. 错误检查
        if (is_wp_error($response) || wp_remote_retrieve_response_code($response) !== 200) {
            return array(
                'status' => 404,
                'success' => false,
                'message' => 'QQ number not exist.'
            );
        }

        // 4. JSON 解析 + 类型验证
        $name = json_decode(wp_remote_retrieve_body($response), true);
        if (!is_array($name) || !isset($name['code']) || $name['code'] != 200) {
            return array(
                'status' => 404,
                'success' => false,
                'message' => 'QQ number not exist.'
            );
        }

        return array(
            'status' => 200,
            'success' => true,
            'message' => 'success',
            'avatar' => 'https://q2.qlogo.cn/headimg_dl?dst_uin=' . urlencode($qq) . '&spec=100',
            'name' => isset($name['name']) ? $name['name'] : '',
        );
    }

    /**
     * Decrypt QQ number from encrypted string and return avatar URL.
     *
     * @param string $encrypted Base64-encoded encrypted QQ number (IV + ciphertext)
     * @return string|false Avatar URL on success, false on failure
     */
    public static function get_qq_avatar($encrypted) {
        global $sakura_privkey;

        // 1. 密钥验证
        if (!isset($sakura_privkey) || empty($sakura_privkey)) {
            return false;
        }

        // 2. 输入验证
        if (empty($encrypted)) {
            return false;
        }

        // 3. Base64 解码
        $decoded = base64_decode(urldecode($encrypted));
        if ($decoded === false) {
            return false;
        }

        // 4. IV 提取（与加密端一致：IV 拼接在密文前）
        $iv_length = openssl_cipher_iv_length('aes-128-cbc');
        if (strlen($decoded) <= $iv_length) {
            return false;
        }

        $iv = substr($decoded, 0, $iv_length);
        $ciphertext = substr($decoded, $iv_length);

        // 5. 解密
        $qq_number = openssl_decrypt($ciphertext, 'aes-128-cbc', $sakura_privkey, 0, $iv);
        if ($qq_number === false) {
            return false;
        }

        // 6. QQ 号格式验证
        if (!preg_match('/^\d{3,}$/', $qq_number)) {
            return false;
        }

        return 'https://q2.qlogo.cn/headimg_dl?dst_uin=' . urlencode($qq_number) . '&spec=100';
    }
}
```

---

## 五、调用方配套修复

### api.php — get_qq_info()

```php
function get_qq_info(WP_REST_Request $request)
{
    if (!sakura_verify_rest_request_nonce($request)) {
        $output = array(
            'status' => 403,
            'success' => false,
            'message' => 'Unauthorized client.'
        );
    } else {
        $qq = sanitize_text_field($request->get_param('qq'));
        if (empty($qq)) {
            $output = array(
                'status' => 400,
                'success' => false,
                'message' => 'Bad Request'
            );
        } else {
            $output = QQ::get_qq_info($qq);
        }
    }

    $result = new WP_REST_Response($output, $output['status']);
    $result->set_headers(array('Content-Type' => 'application/json'));
    return $result;
}
```

### api.php — get_qq_avatar()

```php
function get_qq_avatar(WP_REST_Request $request)
{
    $encrypted = sanitize_text_field($request->get_param('qq'));
    if (empty($encrypted)) {
        return new WP_Error('rest_qq_missing', 'Missing qq parameter', array('status' => 400));
    }

    $imgurl = QQ::get_qq_avatar($encrypted);
    if (!$imgurl) {
        return new WP_Error('rest_qq_avatar_not_found', 'Avatar not found', array('status' => 404));
    }

    if (iro_opt('qq_avatar_link') == 'type_2') {
        $remote = wp_remote_get(esc_url_raw($imgurl));
        if (is_wp_error($remote) || wp_remote_retrieve_response_code($remote) !== 200) {
            return new WP_Error('rest_qq_avatar_fetch_failed', 'Failed to fetch avatar', array('status' => 500));
        }
        $imgdata = wp_remote_retrieve_body($remote);
        header('Content-Type: image/jpeg');
        header('Cache-Control: max-age=86400');
        echo $imgdata;
        exit;
    } else {
        $response = new WP_REST_Response();
        $response->set_status(302);
        $response->header('Location', esc_url_raw($imgurl));
        return $response;
    }
}
```

### api.php — 路由注册（添加 args schema）

```php
register_rest_route('sakura/v1', '/qqinfo/json', array(
    'methods' => 'GET',
    'callback' => 'get_qq_info',
    'permission_callback' => '__return_true',
    'args' => array(
        'qq' => array(
            'type' => 'string',
            'required' => true,
            'sanitize_callback' => 'sanitize_text_field',
            'validate_callback' => function($value) {
                return is_string($value) && preg_match('/^\d{3,}$/', $value);
            },
        ),
    ),
));

register_rest_route('sakura/v1', '/qqinfo/avatar', array(
    'methods' => 'GET',
    'callback' => 'get_qq_avatar',
    'permission_callback' => '__return_true',
    'args' => array(
        'qq' => array(
            'type' => 'string',
            'required' => true,
            'sanitize_callback' => 'sanitize_text_field',
        ),
    ),
));
```

### functions.php — change_avatar()

```php
function change_avatar($avatar)
{
    global $comment, $sakura_privkey;
    if ($comment && get_comment_meta($comment->comment_ID, 'new_field_qq', true)) {
        $qq_number = sanitize_text_field(get_comment_meta($comment->comment_ID, 'new_field_qq', true));
        if (iro_opt('qq_avatar_link') == 'off') {
            return '<img src="https://q2.qlogo.cn/headimg_dl?dst_uin=' . esc_attr(urlencode($qq_number)) . '&spec=100" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
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

        if (isset($sakura_privkey) && !is_null($sakura_privkey)) {
            $iv_length = openssl_cipher_iv_length('aes-128-cbc');
            $iv = openssl_random_pseudo_bytes($iv_length);
            $encrypted = openssl_encrypt($qq_number, 'aes-128-cbc', $sakura_privkey, 0, $iv);
            $encrypted = urlencode(base64_encode($iv . $encrypted));

            return '<img src="' . esc_url(rest_url("sakura/v1/qqinfo/avatar") . '?qq=' . $encrypted) . '" class="lazyload avatar avatar-24 photo" alt="😀" width="24" height="24" onerror="imgError(this,1)">';
        } else {
            return $avatar;
        }
    }
    return $avatar;
}
```
