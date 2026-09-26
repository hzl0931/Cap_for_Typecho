# Cap for Typecho

一个为 Typecho 博客系统提供 Cap 人机验证功能的插件。

---

![Static Badge](https://img.shields.io/badge/Apache_License-V2.0-green)
![GitHub commit activity](https://img.shields.io/github/commit-activity/m/cc2562/Cap_for_Typecho)

Cap 是一个现代、轻量级的开源 SHA-256 工作量证明 CAPTCHA 替代方案。

与传统的 CAPTCHA 不同，Cap：
- 快速且不显眼
- 不使用跟踪或 cookies
- 使用工作量证明而非令人烦恼的视觉谜题
- 完全可访问且可自托管

## 功能特性

- 支持在评论和登录页面启用 Cap 验证码
- 支持自定义 Cap API 端点
- 支持亮色/暗色主题切换
- 支持 cURL 和 file_get_contents 两种请求方式
- 提供救援模式，便于故障排查
- 基于最新的 Cap API 规范

## 安装方法

1. 将 `Cap` 文件夹上传到 Typecho 的 `usr/plugins/` 目录下
2. 在 Typecho 后台的"插件管理"中启用 Cap 插件
3. 进入插件设置页面配置相关参数

## 配置说明

### 必需配置

- **Cap API 端点**: Cap 验证服务的 API 端点地址，默认：`https://captcha.gurl.eu.org/api/`
- **Cap 脚本地址**: Cap 客户端脚本的 URL 地址，默认：`https://captcha.gurl.eu.org/cap.min.js`

### 可选配置

- **启用位置**: 选择在登录页面和/或评论页面启用验证
- **主题**: 选择亮色或暗色主题
- **使用 cURL**: 推荐启用，需要 PHP cURL 扩展
- **Cloudflare Access Client ID / Client Secret**: 可选，使用 Cloudflare Access Service Token 保护 Cap 接口时填写，详见下文

## 使用 Cloudflare Access Service Token 保护 Cap 服务

如果 Cap 服务部署在 Cloudflare 上，并希望用 Cloudflare Access 限制访问来源，可以在插件中填写 Service Token。
插件在服务端校验时会自动为请求附加 Cloudflare 要求的认证请求头：

```
CF-Access-Client-Id: <CLIENT_ID>
CF-Access-Client-Secret: <CLIENT_SECRET>
```

配置步骤：

1. 在 Cloudflare Zero Trust 控制台的 **Access → Service Auth → Service Tokens** 创建 Service Token，记下 Client ID 与 Client Secret
2. 在 Access 应用的策略（Policy）中新增一条 **Service Auth** 规则，仅允许该 Service Token 通过
3. 在插件设置中填入 Client ID 和 Client Secret，保存后即可生效

注意事项：

- Access 应用应**只覆盖服务端校验路径** `{apiEndpoint}/validate`。浏览器中的验证组件会直接请求 `{apiEndpoint}/challenge`
  和 `{apiEndpoint}/redeem`，这两个路径若被 Access 拦截，验证组件将无法加载。请将 Access 应用的路径限定为 `validate`，
  或使用 Bypass 策略放行这两个路径。
- Service Token 属于机密信息，只会保存在服务端数据库中。请勿将其写入主题模板或其他前端可见的位置。
- 两个字段留空时插件不会发送上述请求头，行为与未启用该功能时完全一致。
- 若 Service Token 配置错误，日志中会出现 `Request was intercepted by Cloudflare Access` 提示。

## 自托管Cap
Cap支持自托管，你可以查看官方文档自行建立服务器[https://capjs.js.org/guide/server.html](https://capjs.js.org/guide/server.html)

注意本插件不支持Cap Standalone模式。自托管推荐使用Cloudflare一键部署：[https://github.com/xyTom/cap-worker](https://github.com/xyTom/cap-worker)

## 主题集成

### 评论表单集成

如果要在评论表单中显示验证码，需要在主题的评论表单中添加以下代码：

```php
<?php if (class_exists('Cap_Plugin')): ?>
    <?php Cap_Plugin::output(); ?>
<?php endif; ?>
```

通常添加在评论表单的提交按钮之前。

### 示例代码

```php
<form method="post" action="<?php $this->commentUrl() ?>" id="comment-form" role="form">
    <!-- 其他表单字段 -->
    
    <?php if (class_exists('Cap_Plugin')): ?>
        <?php Cap_Plugin::output(); ?>
    <?php endif; ?>
    
    <button type="submit" class="submit"><?php _e('提交评论'); ?></button>
</form>
```

## Cap 验证服务

本插件基于新的 Cap API 规范，使用以下端点：

### 客户端集成
- 脚本地址: `https://captcha.gurl.eu.org/api/`
- Widget 配置: `data-cap-api-endpoint="https://captcha.gurl.eu.org/cap.min.js"`

### 服务端验证
- 验证端点: `POST /api/validate`
- 请求格式:
  ```json
  {
    "token": "验证token",
    "keepToken": false
  }
  ```
- 响应格式:
  ```json
  {
    "success": true
  }
  ```


## 故障排查

### 救援模式

如果登录验证出现问题导致无法登录后台，可以：

1. 编辑 `Cap/Plugin.php` 文件
2. 将 `private static $rescueMode = false;` 改为 `private static $rescueMode = true;`
3. 这将临时跳过登录验证，允许你进入后台调整设置

### 常见问题

1. **验证码不显示**: 检查 API 端点和脚本地址是否正确
2. **验证失败**: 检查网络连接是否正常，API 服务是否可用
3. **JavaScript 错误**: 检查浏览器控制台是否有错误信息

### 调试信息

插件会在浏览器控制台输出调试信息：
- 验证完成时会显示 "Cap 验证完成" 或 "Cap 登录验证完成"
- 可以检查表单中是否正确添加了 `cap-token` 隐藏字段

## 鸣谢

- Cap 项目: https://github.com/prosopo/captcha

## 许可证

本插件基于 Apache License Version 2.0 许可证发布。

## 更新日志

### v1.1.0
- 支持 Cloudflare Access Service Token：可在插件配置中填写 Client ID / Client Secret，服务端校验请求会自动附加认证请求头
- 请求被 Cloudflare Access 拦截时给出明确提示，便于排查配置问题

### v1.0.0
- 初始版本
- 支持评论和登录验证
- 支持自定义 Cap Standalone 服务器