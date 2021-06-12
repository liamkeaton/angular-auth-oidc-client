---
sidebar_label: Useful Features of this library
sidebar_position: 6
---

# Useful Features of this library

## Public Events

The library exposes several events which are happening during the runtime. You can subscribe to those events by using the `PublicEventsService`.

Currently the events

```ts
{
    ConfigLoaded,
    CheckSessionReceived,
    UserDataChanged,
    NewAuthorizationResult,
    TokenExpired,
    IdTokenExpired,
}
```

are supported.

> Notice that the `ConfigLoaded` event only runs inside the `AppModule`s constructor as the config is loaded with the `APP_INITIALIZER` of Angular inside of the lib.

You can inject the service and use the events like this:

```ts
import { PublicEventsService } from 'angular-auth-oidc-client';

constructor(private eventService: PublicEventsService) {
    this.eventService
            .registerForEvents()
            .pipe(filter((notification) => notification.type === EventTypes.CheckSessionReceived))
            .subscribe((value) => console.log('CheckSessionChanged with value ', value));
}
```

The `Notification` being sent out comes with a `type` and a `value`.

```ts
export interface OidcClientNotification<T> {
  type: EventTypes;
  value?: T;
}
```

Pass inside the `filter` the type of event you are interested in and subscribe to it.

## Custom Storage

If you need, you can create a custom storage (for example to use cookies).

Implement `AbstractSecurityStorage` and the `read`, `write` and `remove` methods:

```ts
@Injectable()
export class CustomStorage implements AbstractSecurityStorage {

    public read(key: string): any {
        ...
        return ...
    }

    public write(key: string, value: any): void {
        ...
    }

    public remove(key: string): void {
        ...
    }

    public clear(): void {
        ...
    }
}
```

Then provide the class in the module:

```ts
@NgModule({
    imports: [
        ...
        AuthModule.forRoot({ config: { storage: CustomStorage } })
    ],
    ...
})
```

## Auto Login

If you want to have your app being redirected to the sts automatically without the user clicking any login button only by accessing a specific you can use the `AutoLoginGuard` provided by the lib. Use it for all the routes you want automatic login to be enabled.

In case you are using multiple configs the guard currently uses the first config fix to perform a login!

The guard handles `canActivate` and `canLoad` for you.

Here are two use cases to distinguish:

1. Redirect route from Security Token Server has a guard in `canLoad` or `canActivate`
2. Redirect route from Token server does _not_ have a guard.

### Redirect route from Token server has a guard

If your redirect route from the Security Token Server to your app has the `AutoLoginGuard` activated already, like this:

```ts
import { AutoLoginGuard } from 'angular-auth-oidc-client';

const appRoutes: Routes = [
  { path: '', pathMatch: 'full', redirectTo: 'home' },
  { path: 'home', component: HomeComponent, canActivate: [AutoLoginGuard] }, <<<< Redirect Route from STS has the guard
  {...
];
```

Then _make sure_ to _*not*_ call the `checkAuth()` method in your `app.component.ts`. This will be done by the guard automatically for you.

### Redirect route from the Token server is public / Does not have a guard

If the redirect route from the STS is publicly available, you _have to_ call the `checkAuth()` by yourself in the `app.component.ts` to proceed the url when getting redirected. The lib redirects you to the route the user entered before he was sent to the login page on the sts automatically for you.

```ts
import { AutoLoginGuard } from 'angular-auth-oidc-client';

const appRoutes: Routes = [
  { path: '', pathMatch: 'full', redirectTo: 'home' },
  { path: 'home', component: HomeComponent },
  { path: 'protected', component: ProtectedComponent, canActivate: [AutoLoginGuard] },
  { path: 'forbidden', component: ForbiddenComponent, canActivate: [AutoLoginGuard] },
  { path: 'unauthorized', component: UnauthorizedComponent },
];
```

```ts
export class AppComponent implements OnInit {
  constructor(public oidcSecurityService: OidcSecurityService) {}

  ngOnInit() {
    this.oidcSecurityService.checkAuth().subscribe(({ isAuthenticated, userData, accessToken, errorMessage }) => {
      console.log(isAuthenticated);
      console.log(userData);
      console.log(accessToken);
      console.log(errorMessage);
    });
  }
}
```

[src code](../projects/sample-code-flow-auto-login)

## Custom parameters

Custom parameters can be added to the auth request by adding them to the config. They are provided by

```ts
customParamsAuthRequest?: {
    [key: string]: string | number | boolean;
};
```

so you can pass them as an object like this:

```ts
AuthModule.forRoot({
      config: {
        stsServer: '<your sts address here>',
        customParamsAuthRequest: {
          response_mode: 'fragment',
          prompt: 'consent',
        },
      },
    }),
```

## Dynamic custom parameters

If you want to pass dynamic custom parameters with the request url to the sts you can do that by passing the parameters into the `authorize` method.

```ts
login() {
    this.oidcSecurityService.authorize(null, { customParams: { ui_locales: 'de-CH' }});
}

```

> If you want to pass static parameters to the sts every time please use the custom parameters in the [Configuration](configuration.md) instead!

## Using the OIDC package in a module or a Angular lib

This example shows how you could set the configuration when loading a child module.

> This is not recommended. Please use the initialization on root level.

```ts
import { NgModule } from '@angular/core';
import { AuthModule, LogLevel } from 'angular-auth-oidc-client';
import { HttpClientModule } from '@angular/common/http';
import { CommonModule } from '@angular/common';
import { RouterModule } from '@angular/router';

@NgModule({
  declarations: [
    /* */
  ],
  imports: [
    AuthModule.forRoot({
      config: {
        stsServer: '<your sts address here>',
        redirectUrl: window.location.origin,
        postLogoutRedirectUri: window.location.origin,
        clientId: 'angularClient',
        scope: 'openid profile email',
        responseType: 'code',
        silentRenew: true,
        silentRenewUrl: `${window.location.origin}/silent-renew.html`,
        renewTimeBeforeTokenExpiresInSeconds: 10,
        logLevel: LogLevel.Debug,
      },
    }),
    HttpClientModule,
    CommonModule,
    RouterModule,
  ],
  exports: [
    /* */
  ],
})
export class ChildModule {}
```

The components code is the same then as using it in the main or any other module.

## Delay the loading or pass an existing `.well-known/openid-configuration` configuration

The secure token server `.well-known/openid-configuration` configuration can be requested via an HTTPS call when starting the application. This HTTPS call may affect your first page loading time. You can disable this and configure the loading of the `.well-known/openid-configuration` later, just before you start the authentication process. You as a user, can decide when you want to request the well known endpoints.

The property `eagerLoadAuthWellKnownEndpoints` in the configuration sets exactly this. The default is set to `true`, so the `.well-known/openid-configuration` is loaded at the start as in previous versions. Setting this to `false` the `.well-known/openid-configuration` will be loaded when the user starts the authentication.

You also have the option to pass the already existing `.well-known/openid-configuration` into the module as a second parameter. In this case no HTTPS call to load the `.well-known/openid-configuration` will be made.

```ts
 AuthModule.forRoot({
    config: {
      // ...
      eagerLoadAuthWellKnownEndpoints: true | false
    },

    authWellknownEndpoints: {
      // ...
    }
  }),
```