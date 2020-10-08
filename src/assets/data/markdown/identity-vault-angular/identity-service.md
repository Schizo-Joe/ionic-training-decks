# Lab: Identity Service

In this lab, we will concentrate on refactoring the `IdentityService` to use Identity Vault to perform the same functionallity that it currently performs.

Identity Vault includes a class called `IonicIdentityVaultUser` which defines the currently logged in user and provides an interface with the identity vault plugin. By the end of this section, we will have updated our `IdentityService` to be a subclass of the `IonicIdentityVaultUser` class, giving us the access we need to the Identity Vault functionallity.

We will walk through adding the parent class one step at a time. As we do this, you should have two files open in your editor of choice:

- `src/app/core/identity/identity.service.spec.ts`
- `src/app/core/identity/identity.service.ts`

## Add the Platform Service

The `IonicIdentityVaultUser` takes the `Platorm` service as its first parameter of its constructor (actually, it takes any object with a `ready()` method). `IdentityService` will have the `Platform` service injected into it, so we need to set that up in our test. We will just use a mock `Platform` service. When we are done, the `TestBed` configuration should look something like this:

```typescript
TestBed.configureTestingModule({
  imports: [HttpClientTestingModule],
  providers: [{ provide: Platform, useFactory: createPlatformMock }],
});
```

Inject the `Platform` `BrowserAuthPlugin` services via the constructor in the `IdentityService` constructor:

```typescript
  constructor(
    private browserAuthPlugin: BrowserAuthPlugin,
    private http: HttpClient,
    platform: Platform,
  ) {
    this._changed = new Subject();
  }
```

## Inherit from `IonicIdentityVaultUser`

The way that our application will interface with Identity Vault is via a service that inherits from `IonicIdentityVaultUser`, so we will modify our `IdentityService` to do just that.

This process will essentially make all of our `IdentityService` tests fail. For this reason, we will first perform the initial inheritance tasks, and then we will concentrate on updating the service method by method to fix both the code and the tests. To start with:

- import several key items from `@ionic-enterprise/identity-vault`
- extend the `IonicIdentityVaultUser` using the `DefaultSession` type
- in the constructor, call `super(...)`
- add the `getPlugin()` method
- remove the `token` getter (the base class already has one)

```typescript
...
import {
  AuthMode,
  DefaultSession,
  IonicIdentityVaultUser,
  IonicNativeAuthPlugin,
} from '@ionic-enterprise/identity-vault';
...

@Injectable({
  providedIn: 'root',
})
export class IdentityService extends IonicIdentityVaultUser<DefaultSession> {
...

  constructor(
    private browserAuthPlugin: BrowserAuthPlugin,
    private http: HttpClient,
    platform: Platform,
  ) {
    super(platform, { authMode: AuthMode.SecureStorage });
    this._changed = new Subject();
  }

...

  getPlugin(): IonicNativeAuthPlugin {
    if ((this.platform as Platform).is('hybrid')) {
      return super.getPlugin();
    }
    return this.browserAuthPlugin;
  }
```

A further explanation of some of these change are in order.

### The `DefaultSession` Type

When we extended the base class, we need to provide a type for our session. Identity Vault is very flexible as to what session information can be stored. The `DefaultSession` includes `username` and `token`, which is more than adaquate for our application given that we are currently only storing the token. The `DefaultSession` can be extended if desired in order to store other information with your session data.

### The `super()` Call

Because of JavaScript's inheritance rules, the first thing we need to do in our contructor is call `super()`. The base class takes an instatiation of the `Platform` service as well as a configuration object. For our initial version of the application, the only configuration we will do is to set the `authMode` to `SecureStorage`. This will configure the vault to securely store the session information, but will not configure any other advanced features, such as locking the vault or using biometrics to unlock the vault.

### The `getPlugin()` Method

The base class' `getPlugin()` method returns an interface to the native plugin for Identity Vault. We also want this application to run in a web based context for testing and development. By overriding this method, we can have Identity Vault use the plugin when running in a hybrid mobild context and use our previously created web based service when running in any other conext.

One item of note here is that we cast the platform service (`this.platform as Platform`). This is required by TypeScript
because Identity Vault is not tightly coupled with `@ionic/angular`'s Platform service and we are calling a method here that Identity Vault does not know about.

## Convert the Methods

At this point we have made our `IdentityService` into an interface into the Identity Vault, but we aren't really using the vault. As a result we currently have three failing tests and our app does not work in that it cannot find access the token that comes back from a login, so our login goes nowhere.

Let's fix that one method at a time starting with the `init()` method.

### The `init()` Method

Currently, the `init()` method gets the token from Capacitor Storage and then tries to get the user if a token was obtained. Instead, we need to change it to restore the session. If after restoring the session a token exists then we can move forward with attempting to obtain information about the current user.

First change the "gets the stored token" test as such (including renaming it a bit):

```typescript
it('restores the session', async () => {
  spyOn(service, 'restoreSession');
  await service.init();
  expect(service.restoreSession).toHaveBeenCalledTimes(1);
});
```

Also change the setup for the "if there is a token" set of tests that immediatly follow the test case you just modified:

```typescript
    describe('if there is a token', () => {
      beforeEach(() => {
        spyOn(service, 'restoreSession').and.callFake(async () => {
          (service as any).session = {
            username: 'meh',
            token: '3884915llf950',
          };
          return (service as any).session;
        });
      });
      ...
    });
```

At this point, you should have four failing tests. Update the code to make them pass again:

```diff
   async init(): Promise<void> {
-    const { Storage } = Plugins;
-    const res = await Storage.get({ key: this.key });
-    this._token = res && res.value;
-    if (this._token) {
+    await this.restoreSession();
+    if (this.token) {
       this.http
         .get<User>(`${environment.dataService}/users/current`)
         .pipe(take(1))
@@ -62,4 +69,11 @@ export class IdentityService {
     await Storage.remove({ key: this.key });
     this._changed.next();
   }
```

#### The `set()` Method

Instead of saving the token using Capacitor Storage, the code will need to register the session information with a call to the base class' `login()` method.

Remove the "sets the token" and "saves the token in storage"

```diff
   async set(user: User, token: string): Promise<void> {
     this.user = user;
-    await this.setToken(token);
+    await this.login({ username: user.email, token });
     this.changed.next(this.user);
   }
```

#### `getToken()`

If the token is not currently set within the service, we need to obtain the token from the vault rather than from Ionic Storage. Call the base class's `restoreSession()` to accomplish this.

```diff
   async getToken(): Promise<string> {
     if (!this.token) {
-      await this.storage.ready();
-      this.token = await this.storage.get(this.tokenKey);
+      await this.restoreSession();
     }
     return this.token;
   }
```

#### `remove()`

Rather than removing the token using Ionic Storage, call the base class's `logout()` method which will remove the token from the vault.

```TypeScript
   async remove(): Promise<void> {
     this.user = undefined;
-    await this.setToken('');
+    await this.logout();
     this.changed.next(this.user);
   }
```

#### `restoreSession()`

It is possible for the vault to get into a state where it it locked but cannot be unlocked. For example if a user locks via touch ID and then removes their fingerprint. In this case, we will get a `VaultErrorCodes.VaultLocked` error indicating that we cannot unlock the vault. If this is the case, clear the vault so the user can log in again.

```TypeScript
  async restoreSession(): Promise<DefaultSession> {
    try {
      return await super.restoreSession();
    } catch (error) {
      if (error.code === VaultErrorCodes.VaultLocked) {
        const vault = await this.getVault();
        await vault.clear();
      }
    }
  }
```

#### Final Cleanup

At this point, there should be some dead code and unused parameters. If you are using VSCode these will show with a lighter font. Remove all of the dead code and unused parameters and/or imports.

## Conclusion

At this point, the application should no longer work in the browser and you will need to run it on a device (we will fix that soon). Try it out on your device:

- `npm run build`
- `npx cap open ios` or `npx cap open android`
- use Xcode or Android Studio to run the application on your attached device
