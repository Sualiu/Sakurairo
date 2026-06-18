# Project Rules

## Git 提交规则
- 提交 commit 时，直接提交到当前仓库分支（security202606），不要创建或推送到 trae 分支

## WordPress 代码审查规则
- 对 WordPress 代码的审查和修改必须结合 ~/.trae/skills/ 中的 WordPress skills 最佳实践，特别是：
  - wp-plugin-development: 插件架构、钩子、安全（nonce/capabilities/sanitization/escaping）
  - wp-rest-api: REST API 路由/端点/认证/schema
  - wp-performance: 性能优化、缓存、数据库
  - wp-phpstan: PHPStan 静态分析
