# OAuth 集成认证（GitHub/Google：第三方登录在这里接入）

你们在日常开发里大概率不会第一时间用 OAuth（内部环境 simple login 更省事），但如果要对外部署、或要把用户体系接到 GitHub/Google，那么这条链路就必须清楚：

- 客户端仍然从 `/api/login/initial` 拿到 pollToken。
- server 暴露 `/api/login/github`、`/api/login/google` 这样的入口。
- 用户在第三方完成授权后回调到 `*-callback` 路由。
- server 将 third-party profile 转成 `userInfo`，fire `onDidAuthenticate`，最终由 `CredentialsManager.confirmUser()` 签发 JWT。

这一页只讲“这个项目里 OAuth 是怎么接进来的”，不展开 OAuth 教科书内容。

## 1) 通用骨架：`OAuthEndpoint.onStart()` 把两条路由挂到 Express

`OAuthEndpoint` 负责把“开始登录的入口路由”和“回调路由”都挂上，并且把最终认证结果 fire 出去。

下面这段代码里，你能看到三件很实用的工程细节：

- `token` 从 query 里拿（它就是 pollToken/confirmToken）。
- 回调时用 `state` 传回 token，把 OAuth 回调与轮询 token 绑在一起。
- 支持 redirect，并且做 whitelist 校验，避免开放重定向。

```ts
app.get(this.path, async (req, res) => {
    const token = req.query.token;
    if (!token) {
        res.status(400);
        res.send('Error: Missing token parameter in request');
        return;
    }
    if (req.query.redirect) {
        this.loginRedirectRequests.set(token.toString(), req.query.redirect.toString());
    }
    passport.authenticate(this.id, { state: `${token}`, scope: this.scope })(req, res);
});

const redirectUriWhitelist = this.configuration.getValue('oct-redirect-url-whitelist')?.split(',');
app.get(this.redirectPath, async (req, res) => {
    const token = (req.query.state as string);
    passport.authenticate(this.id, { state: token, scope: this.scope }, async (err: any, userInfo?: UserInfo) => {
        if (err || !userInfo) {
            res.status(400);
            res.send('Error retrieving user info');
            return;
        }
        await Promise.all(this.authSuccessEmitter.fire({ token, userInfo }));
        // ... redirect with whitelist check
    })(req, res);
});
```

这段代码解决的问题是：“OAuth 跳转与回调是第三方的，但 token 的收口仍然回到本项目的 pollToken 机制”。

## 2) 何时启用 GitHub/Google：`shouldActivate()` 是配置开关

GitHub/Google 的 endpoint 只有在配置齐全时才会激活（避免把不可用 provider 暴露给客户端）：

```ts
shouldActivate(): boolean {
    return Boolean(
        this.configuration.getValue('oct-oauth-github-clientid')
        && this.configuration.getValue('oct-oauth-github-clientsecret')
    );
}
```

这段代码解决的问题是：“不用把环境差异写死在前端”。你们部署时要注意把这些值放到正确的 config/env 里。

## 3) 生成 callback URL：`createRedirectUrl()` 依赖 `oct-base-url`

OAuth 最容易踩的坑之一是 callback URL 不一致（本地、内网、外网、反代）。

项目里用 `oct-base-url` 来决定 callback 的 base URL，如果没有就用 `host/port` 拼一个默认：

```ts
protected createRedirectUrl(host: string, port: number, path: string): string {
    const baseURL = this.baseURL ?? `http://${host === '0.0.0.0' ? 'localhost' : host}:${port}`;
    return new URL(path, baseURL).toString();
}
```

这段代码解决的问题是：“同一套 server 代码能在不同部署环境下稳定生成 callback URL”。如果你们走反向代理（HTTPS/域名），建议务必显式配置 `oct-base-url`。

## Summary

- OAuthEndpoint 的关键是把 `pollToken`（confirmToken）通过 query/state 串进第三方 OAuth 回调，再 fire `userInfo` 给 server 统一签 JWT。
- provider 的启用条件在 `shouldActivate()`：配置不全就不要暴露给客户端。
- callback URL 生成依赖 `oct-base-url`，部署时建议显式配置，避免反代环境下回调错。
- 下一步阅读建议：
  - 认证总主线： [认证与安全接入](/collaboration_tools/核心模块详解/open-collaboration-server模块/认证与安全接入/认证与安全接入.md)
  - Keycloak： [Keycloak 单点登录](/collaboration_tools/核心模块详解/open-collaboration-server模块/认证与安全接入/Keycloak 单点登录.md)

