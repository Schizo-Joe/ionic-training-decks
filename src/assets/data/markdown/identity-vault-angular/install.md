# Lab: Installation and Basic Setup

## Install Identity Vault

Before we can use Identity Vault, we need to install it and update the native projects.

```Bash
$ ionic enterprise register --key=YOURPRODUCTKEY
$ npm i @ionic-enterprise/identity-vault
$ npx cap update
```

**Note:** if you have already registered your product key using a different app, you can just copy the `.npmrc` file from the other application to this training application. Since this is just a training app, we do not care that you are using your key here and in your production application.

## Browser Auth Plugin and Service

Identity Vault uses hardware and APIs that are only available when we are running on a device. That limits our application to only being run on a device or emulator. That is less than ideal. We would like to be able to also run our application in a web context in order to support the following scenarios:

- Unit testing
- Running in the development server as we build out our application
- Deploying the application to the web for use as a PWA

In order to support this, we need to add a service that will perform the "Identity Vault" related tasks but store the token using the Capacitor Storage API, similar to what our app currently does by default.

### The Services

Two services need to be created. They both will reside in the `src/app/services/browser-auth` folder, so create that folder now.

#### BrowserAuthService

The `BrowserAuthService` is based on the JavaScript interface for the Identity Vault itself. It contains all of the same methods that the actual Vault does. Many of these methods are just stubs. A few need to perform actions, however. For the most part, these actions involve storing and retrieving the authentication token using the Capacitor Storage API. A common implementation looks like this:

```TypeScript
import { Injectable } from '@angular/core';
import { Plugins } from '@capacitor/core';
import {
  BiometricType,
  IdentityVault,
  PluginConfiguration,
  AuthMode,
  SupportedBiometricType,
} from '@ionic-enterprise/identity-vault';

@Injectable({
  providedIn: 'root',
})
export class BrowserAuthService implements IdentityVault {
  constructor() {}

  config = {
    authMode: AuthMode.SecureStorage,
    descriptor: {
      username: '',
      vaultId: '',
    },
    isBiometricsEnabled: false,
    isPasscodeEnabled: false,
    isPasscodeSetupNeeded: false,
    isSecureStorageModeEnabled: true,
    hideScreenOnBackground: false,
    lockAfter: 50000,
  };

  async unsubscribe(): Promise<void> {}

  async clear(): Promise<void> {
    const { Storage } = Plugins;
    await Storage.clear();
  }

  async lock(): Promise<void> {}

  async isLocked(): Promise<boolean> {
    return false;
  }

  async isInUse(): Promise<boolean> {
    const { Storage } = Plugins;
    return !!(await Storage.get({ key: 'session' }));
  }

  async getConfig(): Promise<PluginConfiguration> {
    return this.config;
  }

  async remainingAttempts(): Promise<number> {
    return 5;
  }

  async getUsername(): Promise<string> {
    return 'MyUsername';
  }

  async storeToken(token: any): Promise<void> {}

  async getToken(): Promise<any> {
    return 'MyToken';
  }

  async storeValue(key: string, value: any): Promise<void> {
    const { Storage } = Plugins;
    await Storage.set({ key, value: JSON.stringify(value) });
  }

  async getValue(key: string): Promise<any> {
    const { Storage } = Plugins;
    const { value } = await Storage.get({ key });
    return JSON.parse(value);
  }

  async removeValue(key: string): Promise<void> {
    const { Storage } = Plugins;
    await Storage.remove({ key });
  }

  async getKeys(): Promise<Array<string>> {
    const { Storage } = Plugins;
    const { keys } = await Storage.keys();
    return keys;
  }

  // tslint:disable-next-line
  async getBiometricType(): Promise<BiometricType> {
    return 'none';
  }

  async getAvailableHardware(): Promise<Array<SupportedBiometricType>> {
    return [];
  }

  async setBiometricsEnabled(isBiometricsEnabled: boolean): Promise<void> {}

  async isBiometricsEnabled(): Promise<boolean> {
    return false;
  }

  async isBiometricsAvailable(): Promise<boolean> {
    return false;
  }

  async isBiometricsSupported(): Promise<boolean> {
    return false;
  }

  async isLockedOutOfBiometrics(): Promise<boolean> {
    return false;
  }

  async isPasscodeSetupNeeded(): Promise<boolean> {
    return false;
  }

  async setPasscode(passcode?: string): Promise<void> {}

  async isPasscodeEnabled(): Promise<boolean> {
    return false;
  }

  async isSecureStorageModeEnabled(): Promise<boolean> {
    return true;
  }

  async setPasscodeEnabled(isPasscodeEnabled: boolean): Promise<void> {}

  async setSecureStorageModeEnabled(enabled: boolean): Promise<void> {}

  async unlock(usingPasscode?: boolean, passcode?: string): Promise<void> {}
}
```

Create a `src/app/core/browser-auth/browser-auth.service.ts` file with the above contents.

#### BrowserAuthPlugin

The `BrowserAuthPlugin` service mimics the Indentity Vault plugin's JavaScript interface that gets us access to the vault. Rather than returning an object that accesses the Vault plugin, the `BrowserAuthPlugin` service returns an instance of the `BrowserAuthService`, which is our browser-based service the implements the same API as the plugin. This code is very simple:

```TypeScript
import { Injectable } from '@angular/core';
import {
  IdentityVault,
  PluginOptions,
  IonicNativeAuthPlugin
} from '@ionic-enterprise/identity-vault';
import { BrowserAuthService } from './browser-auth.service';

@Injectable({ providedIn: 'root' })
export class BrowserAuthPlugin implements IonicNativeAuthPlugin {
  constructor(private browserAuthService: BrowserAuthService) {}

  getVault(config: PluginOptions): IdentityVault {
    config.onReady(this.browserAuthService);
    return this.browserAuthService;
  }
}
```

## Conclusion

Now that Identity Vault has been installed, it is time to modify the `IdentityService` to use Identity Vault. We will do this in baby steps. Our first step will just be to get the application working essetially as it is today, except using Idenity Vault for the storage of the session information.
