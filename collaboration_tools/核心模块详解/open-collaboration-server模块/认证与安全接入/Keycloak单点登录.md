# Keycloak 单点登录（企业环境的 OAuth2/OIDC 接入）

Keycloak 在这个项目里本质上也是一个 `OAuthEndpoint` 的实现：

- client 仍然拿 `/api/login/initial` 的 pollToken。
- server 暴露 `/api/login/keycloak` 与 `/api/login/keycloak-callback`。
- 授权完成后从 Keycloak 的 userinfo 拉用户信息，转成 `userInfo`，fire 给 `CredentialsManager.confirmUser()` 统一签 JWT。

这一页只讲“在本项目里 Keycloak 怎么接”，不展开 Keycloak 的部署与权限模型。

## 1) 初始化：从配置里读 Keycloak 参数

Keycloak endpoint 在 `init()` 里读取配置并拼出 realm base URL：

```ts
this.host = this.configuration.getValue('keycloak-host');
this.realm = this.configuration.getValue('keycloak-realm');
this.clientID = this.configuration.getValue('keycloak-client-id');
this.clientSecret = this.configuration.getValue('keycloak-client-secret');
this.userNameClaim = this.configuration.getValue('keycloak-username-claim');
this.label = this.configuration.getValue('keycloak-client-label') ?? 'Keycloak';

this.keycloakBaseUrl = `${this.host}/realms/${this.realm}`;
super.initialize();
```

这段代码解决的问题是：“让同一套代码能接入不同 Keycloak 实例”。二开时如果你们有多个 realm 或多个 client，建议不要把逻辑写死在代码里，继续沿用配置驱动。

## 2) 何时启用：`shouldActivate()` 是最小启用条件

Keycloak provider 不要求 clientSecret（可选），但至少要有 host/realm/clientId：

```ts
shouldActivate(): boolean {
    return !!this.host && !!this.realm && !!this.clientID;
}
```

这段代码解决的问题是：“不要把不可用 provider 暴露给客户端”。

## 3) 策略实现：授权/换 token/userinfo，并映射成 `userInfo`

Keycloak 的 strategy 配置了 authorization/token/userinfo 三个 URL，并在 verify 回调里把 profile 映射成 `userInfo`：

```ts
return new KeycloakStrategy({
    authorizationURL: `${this.keycloakBaseUrl}/protocol/openid-connect/auth`,
    tokenURL: `${this.keycloakBaseUrl}/protocol/openid-connect/token`,
    userInfoURL: `${this.keycloakBaseUrl}/protocol/openid-connect/userinfo`,
    clientID: this.clientID!,
    clientSecret: this.clientSecret ?? '',
    scope: ['openid', 'email', 'profile'],
    callbackURL: this.createRedirectUrl(host, port, this.redirectPath),
}, (accessToken: string, refreshToken: string, profile: any, done: VerifyCallback) => {
    const userInfo = {
        name: profile[this.userNameClaim ?? 'preferred_username'],
        email: profile.email,
        authProvider: this.label,
    };
    done(undefined, userInfo);
});
```

这段代码解决的问题是：“把 Keycloak 的用户信息转成项目内部需要的 `UserInfo`”。你如果要在内部环境里把更多字段写进 JWT（例如部门/工号），建议从 `userInfo` 里扩展，然后在 `UserManager.registerUser` / JWT claim 里控制存储范围。

`KeycloakStrategy.userProfile()` 会去 userinfoURL 拉 profile：

```ts
fetch(this.options.userInfoURL, {
    headers: { Authorization: `Bearer ${accessToken}` },
}).then(async response => {
    if (!response.ok) {
        throw new Error(`Failed to fetch user profile: ${response.statusText}`);
    }
    done(undefined, await response.json());
}).catch(err => done(err));
```

## Summary

- Keycloak 在本项目里就是一个 OAuthEndpoint：依赖配置拼 URL，授权后拉 userinfo，映射成 `userInfo` 并 fire 给统一 JWT 签发链路。
- 最小启用条件：host/realm/clientId 配齐。
- 二开推荐做法：扩展 `userInfo` 字段，但保持 `confirmUser()` 作为签 JWT 的统一收口。
- 下一步阅读建议：
  - OAuth 通用骨架： [OAuth 集成认证](/collaboration_tools/核心模块详解/open-collaboration-server模块/认证与安全接入/OAuth 集成认证.md)
  - 认证主线： [认证与安全接入](/collaboration_tools/核心模块详解/open-collaboration-server模块/认证与安全接入/认证与安全接入.md)

