# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

彩虹易支付：基于 PHP 的第三方聚合支付平台（当前版本 3075）。商户通过 API 发起收款，平台按渠道生成二维码/跳转收银台，用户完成支付后通过异步回调/主动查询回填订单。所有对外业务由 `index.php`、`api.php`、`pay.php`、`submit.php`、`cron.php` 等根目录入口处理；核心逻辑集中在 `includes/lib/`，支付渠道通过 `plugins/` 插件化扩展。

无需构建、编译或单元测试。运行即部署（宝塔面板 PHP 7.4+ / MySQL 5.7+），详见 `README.md`。目录级 `AGENTS.md`（`user/` `admin/` `includes/` `template/` `plugins/`）是本仓库的结构知识库，遇到具体目录先读对应文件的 `AGENTS.md`。

## 关键常量与全局引导

`includes/common.php` 是唯一引导文件，所有入口（除 `config.php`、`install/` 外）都 `include` 它。它依次：
1. 定义常量 `SYSTEM_ROOT`、`ROOT`、`PLUGIN_ROOT`、`TEMPLATE_ROOT`、`DBQZ`（表前缀）、`SYS_KEY`。
2. 注册 `Autoloader`（`includes/autoloader.php`）——把 `lib\Foo` 类名映射到 `includes/lib/Foo.php`。核心类统一在 `lib\` 命名空间。
3. 引入 `config.php`（`$dbconfig`，数据库连接 + `dbqz` 前缀），建立全局 `$DB`（`lib\PdoHelper`）、`$CACHE`（`lib\Cache`，文件缓存）、`$conf`（`pre_config` 配置数组）。
4. 导入 `functions.php`（业务函数，40KB，命名空间外的全局函数）和 `member.php`（登录态 `$user`）。

全局变量约定（函数内用 `global $DB, $conf, $user, $order, $channel;`）：`$DB` 数据库、`$CACHE` 缓存、`$conf` 系统配置、`$user` 当前商户、`$order` 当前订单、`$channel` 当前支付通道。数据表统一 `pre_` 前缀（默认 `pay_`），核心表：`pre_user`（商户）、`pre_order`（订单）、`pre_type`（支付类型）、`pre_channel`（支付通道）、`pre_config`、`pre_settle`、`pre_site`。

## HTTP 入口与路由约定

不同入口用不同方式定位处理逻辑，勿混淆：

| 入口 | 路由 | 处理 |
|------|------|------|
| `index.php` 前端 | `?mod=<name>` | `\lib\Template::load($mod)` 输出页面；`?doc=` 用于文档页 |
| `submit.php` / `submit2.php` 商户取支付链接 | `pid` + `trade_no` 等 | `\lib\api\Pay::submit()`（商户 API 的核心调用点） |
| `pay.php` 用户支付页 | `?s=<func>/<trade_no>/` | `\lib\Plugin::loadForPay($s)` 调插件方法并 `\lib\Payment::echoDefault` |
| `cashier.php` 收银台 | `?trade_no=...` | 展示订单 + 可选支付通道 |
| `api.php` 商户 API | `?s=<ep>` → `\lib\ApiHelper::load_api`；否则 `?act=` JSON 端点 | 签名验签（`md5`/RSA） |
| `cron.php` 计划任务 | `?key=<cronkey>&do=<settle\|query...>` | 回调、主动查单、结算等定时任务 |
| 管理后台 `admin/` | `?mod=` | 独立后台入口与登录验证 |

插件方法的 `$s` 参数格式固定为 `/<func>/<trade_no>/`（正则 `^(.[a-zA-Z0-9]+)\/([0-9]+)\/$`），`func` 即插件方法名（如 `submit`、`qrcode`、`notify`）。

## 支付插件系统

每个渠道一个目录 `plugins/<name>/`，主文件 `<name>_plugin.php` 定义一个**全局类** `class <name>_plugin`（无命名空间），全部为 `static` 方法。核心接口方法（各渠道选择性实现）：
- 收款：`submit()`、`qrcode()`、`mapi()`、`submitpc()`、`submitwap()`、`apppay()` 等，返回「二维码/跳转表单/App 拉起」等载荷。
- 回调：`notify()`（异步回调）、`return()`（同步跳转）、`preauthnotify()`。
- 提现/查询：`refund($order)`、`transfer($channel,$bizParam)`、`transfer_query()`、`transfer_proof()`、`balance_query()`。

统一走 `lib\Channel`（取通道）、`lib\Plugin`（调度 `loadForPay`/`loadClass`/`loadForSubmit`/`loadForAdmin`）、`lib\Payment`（签名 `makeSign`/`verifySign`、订单处理 `processOrder`/`updateOrder`、回显 `echoDefault`/`echoJson`）。任何提交都应先查订单再调用插件，避免重复处理。

新增渠道 = 新建 `plugins/<name>/<name>_plugin.php` 实现上述接口，无需注册其它地方；通过后台启用、配置存库。参考现成插件（如 `plugins/alipay/`、`plugins/epay/`）保持一致写法。

## 常改的横跨文件逻辑

- 订单金额计算、回调回填、签名校验：`includes/functions.php` + `includes/lib/Payment.php` + 对应插件 `notify()`。
- 商户 API 新增接口：在 `api.php`（`?act=` 分支）或 `includes/lib/ApiHelper.php` 添加，注意 `pre_user.key` 的 `md5`/RSA 签名及 `keytype` 校验。
- 前端页面/模板：路由在 `index.php`/`includes/pages/`，展示在 `template/<name>/`，用 `\lib\Template::load` 装载。

## 部署与运行注意事项

- 需要 PHP 7.4+、MySQL 5.7；面板中必须**关闭跨站攻击防护**；伪静态规则见根目录 `nginx.txt`（Apache 见 `.htaccess`）。
- 首次访问 `/install` 安装（生成 `install/install.lock` 后站点才工作）；升级走 `/install/update.php`，版本由 `DB_VERSION` 控制。
- 支付回调依赖 URL 计划任务定时访问 `cron.php?key=<cronkey>&do=query`，未配置则支付不回调，详见后台。
- 数据库连接在 `config.php`（当前为空占位，安装时写入）。默认后台账号 `admin/123456`。
