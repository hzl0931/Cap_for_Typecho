# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A single-file Typecho plugin that adds [Cap](https://github.com/prosopo/captcha) proof-of-work human verification to the login page and comment form. Targets Typecho 1.2+ (namespaced API — `Typecho\Plugin\PluginInterface`, not the legacy `Typecho_Plugin_Interface`).

## Development

There is no build, lint, test, or dependency tooling. This is a plain PHP plugin distributed by copying the directory into a Typecho install.

The directory **must be named `Cap`** once deployed — Typecho derives the plugin's config key (`plugin:Cap`) and the `Cap_Plugin` class name from the folder name.

Syntax check the only source file:

```bash
php -l Plugin.php
```

Manual test loop: copy this directory to `<typecho>/usr/plugins/Cap/`, enable it under 后台 → 插件管理, then exercise the login page and comment form. There is no automated coverage — verify changes in a running Typecho instance, checking both the browser console and the PHP `error_log` output.

## Architecture

`Plugin.php` is the entire plugin. `cap.min.js` is a vendored copy of the upstream Cap client script, included as a fallback for self-hosting the frontend asset; it is not built or modified here.

### Hook wiring

All hooks are registered in `activate()`. Understanding the plugin means understanding these four entry points:

| Hook | Handler | Role |
| --- | --- | --- |
| `Widget\Archive`->header | `header()` | Injects `<script src="{scriptUrl}">` site-wide |
| `Widget\Feedback`->comment | `verifyCap_comment()` | Server-side token check on comment submit |
| `Widget\Login`->login | `verifyCap_beforeLogin()` | Server-side token check on login submit |
| `admin/footer.php`->end | `output_login()` | Injects the login-page widget markup + client JS |

`output()` is *not* a hook — it is a public static method that theme authors call manually from the comment form template (`Cap_Plugin::output()`). This asymmetry is documented in the README and is the reason comment verification cannot work without a theme edit.

### Configuration

Config lives in Typecho's `table.options` under the key `plugin:Cap`, read everywhere via `Options::alloc()->plugin('Cap')`. Since the plugin defines no `Config` form class, `activate()` writes a default row directly into the DB and falls back to `Options::__set()` if the insert fails. Keys: `apiEndpoint`, `scriptUrl`, `theme`, `enableActions` (array, values `login` / `comment`), `useCurl`.

Several handlers wrap config reads in `try/catch` and silently return on failure, so a missing or corrupt config degrades to "plugin does nothing" rather than a fatal error. Do not remove those guards without checking that `Options::plugin()` still tolerates an unconfigured plugin.

### Token flow

1. Cap's `<cap-widget>` element is rendered with `data-cap-api-endpoint`.
2. On the widget's `solve` event, the client JS injects a hidden `input[name="cap-token"]` into the form. Form submit handlers block submission if that field is missing or short.
3. Server side, `verifyCap_comment()` / `verifyCap_beforeLogin()` read `$_POST['cap-token']` and call `validateCapToken()`.
4. `validateCapToken()` POSTs `{"token": ..., "keepToken": false}` to `{apiEndpoint}/validate` and requires a JSON body of `{"success": true}`. Tokens are one-shot and must contain a `:` separator — the colon check is an explicit early-reject, not an incidental parse.

Only the API's `/validate` endpoint is used; there is no site/secret key exchange (Cap v2 API). Self-hosted Cap **Standalone** mode is explicitly unsupported — see the README for the recommended Cloudflare worker deployment.

### Failure semantics

These differ per path and are easy to break accidentally:

- **Comment**: user errors (missing/empty token, validation failure) throw `\Typecho\Plugin\Exception` and block the comment. Internal errors (network, JSON parse, config) are caught and the comment is *allowed* through — the plugin fails open on comment.
- **Login**: failures call `loginFailed()`, which sets a `Widget\Notice` error and redirects back. The `catch` swallows config errors and lets login proceed.

### Rescue mode

`private static $rescueMode` at the top of `Plugin.php` is a hardcoded kill switch. Setting it to `true` skips login verification entirely, allowing recovery when a misconfigured plugin locks the admin out. This is the documented recovery path — keep it working, and do not make it depend on the plugin config it is meant to bypass.

## Debugging

The plugin logs verbosely via `error_log` on both verification paths, including raw POST keys and full API responses. Client-side, it logs to the browser console and exposes `window.checkCapStatus()` on the login page for inspecting the current token state.
