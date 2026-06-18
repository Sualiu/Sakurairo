# 类别 6: 重定向与 IP 安全

> 包含 Patch: 14, 19
> 审查依据: wp-plugin-development/references/security.md
> 风险等级: 中（开放重定向 / IP 伪造）

---

## WordPress Skills 最佳实践

来自 `wp-plugin-development/references/security.md`:

> **Sanitization and escaping**
> - sanitize/validate on input, escape on output.
> - use `wp_unslash()` before sanitizing when needed

WordPress 重定向安全函数：

| 函数 | 用途 | 安全特性 |
|------|------|---------|
| `wp_safe_redirect()` | 安全重定向 | 检查重定向 URL 是否在允许主机列表中 |
| `wp_redirect()` | 普通重定向 | 无主机检查，有开放重定向风险 |
| `admin_url()` | 管理后台 URL | 始终返回站点内 URL |
| `esc_url_raw()` | URL 净化（存储/重定向） | 过滤危险协议，用于非输出上下文 |

**关键原则:**
1. **重定向必须用 `wp_safe_redirect()`** — 防止开放重定向攻击
2. **`wp_redirect()` 仅用于确定安全的内部 URL** — 如 `admin_url()`
3. **用户可控 URL 必须先 `esc_url_raw()`** — 过滤 `javascript:`、`data:` 等协议
4. **IP 获取不信任 HTTP_CLIENT_IP** — 非标准头，纯伪造向量
5. **X-Forwarded-For 取最右侧 IP** — CDN 场景下最右侧最接近源站

---

## 主题存在的问题分析

### 问题 1: 开放重定向 — wp_redirect 用于用户可控 URL

主题中多处使用 `wp_redirect()` 重定向到可能包含用户输入的 URL：

| 文件 | 行号 | 代码 | 风险 |
|------|------|------|------|
| `functions.php` | 1769 | `wp_redirect(get_option('siteurl') . '/wp-admin/profile.php')` | `get_option('siteurl')` 可被修改 |
| `functions.php` | 4521 | `wp_redirect(admin_url(), 302)` | `admin_url()` 安全，但应用 `wp_safe_redirect` |
| `inc/cache_settings.php` | 138 | `wp_redirect(add_query_arg(...))` | `add_query_arg` 可被注入 |
| `inc/classes/gallery.php` | 227 | `wp_redirect($random_image, 302)` | `$random_image` 来自数据库 |
| `inc/api.php` | 556 | `$response->header('Location', $data)` | REST 重定向未净化 |

**开放重定向危害:**
攻击者可构造恶意链接 `https://victim.com/redirect?url=https://evil.com`，利用站点信任诱导用户访问钓鱼网站。

### 问题 2: IP 伪造 — 信任 HTTP_CLIENT_IP

`functions.php` 的 `get_the_user_ip()` 信任 `HTTP_CLIENT_IP` 头：

```php
$ip = $_SERVER['HTTP_CLIENT_IP'] ?? $_SERVER['HTTP_X_FORWARDED_FOR'] ?? $_SERVER['REMOTE_ADDR'];
```

**问题:**
1. **`HTTP_CLIENT_IP` 是非标准头** — 不在任何 RFC 中，纯伪造向量
2. **`HTTP_X_FORWARDED_FOR` 取整个字符串** — 攻击者可在链首插入伪造 IP
   - X-Forwarded-For 格式: `client, proxy1, proxy2`
   - 攻击者发送: `X-Forwarded-For: 1.2.3.4, real_ip`
   - 代码取整个字符串 → IP 含逗号和空格
3. **无 IP 格式验证** — 返回值可能是任意字符串

**IP 伪造危害:**
- 绕过 IP 限制（评论频率限制、登录尝试限制等）
- 日志污染（注入恶意 IP 字符串）
- 如果 IP 用于 SQL 查询（虽已参数化），仍可能导致逻辑错误

---

## Patch 清单

### Patch 14: 开放重定向修复 — wp_redirect → wp_safe_redirect

**文件:** [functions.php](file:///workspace/functions.php), [inc/cache_settings.php](file:///workspace/inc/cache_settings.php), [inc/classes/gallery.php](file:///workspace/inc/classes/gallery.php), [inc/api.php](file:///workspace/inc/api.php)

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

**修复要点:**

1. **`wp_redirect()` → `wp_safe_redirect()`** — 检查重定向目标是否在 `allowed_redirect_hosts` 列表中
2. **`get_option('siteurl') . '/wp-admin/...'` → `admin_url('profile.php')`** — 使用 WordPress 核心 URL 构建函数
   - `admin_url()` 始终返回当前站点管理 URL
   - 避免手动拼接 URL
3. **`esc_url_raw()` 用于重定向 URL** — 过滤危险协议
   - `esc_url_raw()` 用于存储/重定向上下文（不输出 HTML 实体）
   - `esc_url()` 用于输出上下文（输出 HTML 实体）
4. **REST API Location header 也需净化** — `$response->header('Location', esc_url_raw($data))`

**`wp_safe_redirect()` 工作原理:**
- 检查目标 URL 的 host 是否在 `allowed_redirect_hosts` 列表中
- 默认仅允许当前站点 host
- 可通过 `allowed_redirect_hosts` filter 添加额外 host
- 如果目标不在列表中，重定向到 `wp_login_url()` 或当前站点首页

---

### Patch 19: IP 伪造防护 — get_the_user_ip 加固

**文件:** [functions.php](file:///workspace/functions.php)

```diff
--- a/functions.php
+++ b/functions.php
@@ -3611,14 +3611,13 @@
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
+        if (filter_var($candidate, FILTER_VALIDATE_IP)) {
+            $ip = $candidate;
+        }
+    }

     return apply_filters('wpb_get_ip', $ip);
```

**修复要点:**

1. **移除 `HTTP_CLIENT_IP` 信任** — 非标准头，纯伪造向量
   - `HTTP_CLIENT_IP` 不在任何 RFC 中
   - 任何客户端都可在请求中设置此头
   - 不应作为 IP 来源

2. **X-Forwarded-For 取最右侧 IP** — `end($forwarded_chain)`
   - X-Forwarded-For 格式: `client, proxy1, proxy2`
   - CDN/反向代理会将真实访客 IP **追加到链尾**
   - 取最右侧 = 取最接近源站的 IP
   - 攻击者在链首插入伪造 IP 无效（被忽略）

3. **`filter_var($candidate, FILTER_VALIDATE_IP)` 验证** — IP 格式验证
   - 验证候选值是否为合法 IPv4 或 IPv6
   - 防止注入非 IP 字符串
   - 如果验证失败，回退到 `REMOTE_ADDR`

4. **`isset($_SERVER['REMOTE_ADDR'])` 检查** — CLI 环境防御
   - CLI 环境下 `$_SERVER['REMOTE_ADDR']` 可能不存在
   - 回退到空字符串

**IP 获取优先级对比:**

| 优先级 | 修复前 | 修复后 |
|--------|--------|--------|
| 1 | `HTTP_CLIENT_IP`（可伪造） | `REMOTE_ADDR`（不可伪造） |
| 2 | `HTTP_X_FORWARDED_FOR`（整个字符串） | X-Forwarded-For 最右侧（经 filter_var 验证） |
| 3 | `REMOTE_ADDR` | — |

---

## 审查结论

| Patch | wp_safe_redirect | esc_url_raw | IP 验证 | 符合最佳实践 |
|-------|-----------------|------------|---------|-------------|
| 14 | ✅ | ✅ | N/A | ✅ |
| 19 | N/A | N/A | ✅ filter_var | ✅ |

**关键教训:**
- `wp_redirect()` 有开放重定向风险，应始终使用 `wp_safe_redirect()`
- `admin_url()` 比 `get_option('siteurl')` 拼接更安全
- `HTTP_CLIENT_IP` 是非标准头，绝不应信任
- X-Forwarded-For 应取最右侧 IP（CDN 追加方向），并验证 IP 格式
- `filter_var(FILTER_VALIDATE_IP)` 是 PHP 内置的 IP 验证函数，比正则更可靠
