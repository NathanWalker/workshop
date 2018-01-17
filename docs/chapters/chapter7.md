## Code sharing in Nx with web and NativeScript mobile apps

Now that you’ve got the NativeScript basics down, it’s time to put your skills to the test. Your task in the next three chapters is to utilize these new skills to maximize code sharing between a web app and a NativeScrip mobile app.

### Create an Nx workspace containing (web + mobile) apps + shared lib

Let’s get started building by creating a new Nx workspace.

<h4 class="exercise-start">
  <b>Exercise</b>: Create an Nx workspace containing (web + mobile) apps
</h4>

Navigate to a folder where you’d like your new workspace to live in your file system, and run the following commands.

```
npm install -g @nrwl/schematics
npm install -g @angular/cli
create-nx-workspace mycompany
```

After that completes, `cd` into your newly created workspace.

```
cd mycompany
```

#### Create web app

Let's generate our web app which will end up being a standard Angular CLI web application.

```
ng generate app web
```

We are going to want to use `SASS` for all our styling so let's make the following adjustments:

1. Open `.angular-cli.json` and change `"defaults"."styleExt"` to `scss`.
2. Open `apps/web/src` and rename `styles.css` to `styles.scss`.

Just add a `body` style to our `styles.scss` so we know it's working:

```
body {
    background-color: lightgrey;
}
```

We can now run `npm start` and navigate to `http://localhost:4200` to see our web app.

#### Create mobile app

Let's now create a NativeScript mobile app.

```
cd apps
tns create mobile --template https://github.com/NathanWalker/template-nativescript-nx
cd ..
```

#### Followup steps to satisfy Nx workspace setup

We now have our web and mobile apps setup ready to develop but we want to add some scripts to run these specifically. Open `package.json` and add the following in between `ng` and `build`:

```
--- "ng": "ng", ---
"start": "npm run start.web",
"start.web": "ng serve --app=web",
"start.ios": "cd apps/mobile && tns run ios --emulator --syncAllFiles",
"start.android": "cd apps/mobile && tns run android --emulator --syncAllFiles",
--- "build": "ng build", --- 
```

To ensure the workspace unit tests can still be run against the web code open `tsconfig.spec.json` and add `apps/mobile` to the `exclude` block.

Before going further you will absolutely want to add a `.gitignore` inside the `apps/mobile` directory containing the following:

```
# system files
.DS_Store
Thumbs.db

# nativescript
/hooks
/node_modules
/platforms
/lib

app/**/*.js
app/**/*.d.ts
app/**/*.map
app/**/*.css

# misc
npm-debug.log

# app
!app/assets/font-awesome.min.css
/report/
.nsbuildinfo
/temp/
/app/tns_modules/

# app uses platform specific scss which can inadvertently get renamed which will cause problems
app/app.scss
```

<div class="exercise-end"></div>

You can now run `npm start` to develop the web app and `npm run start.ios` or `npm run start.android` to develop the iOS or Android apps.

### 7.1 Create a shared lib

#### Background on why we are about to build a foundational layer of services?

The browser global `window` object does not exist in a native mobile app since you are not working with browser api's however NativeScript provides a few common browser like api's which help provide a familiar developer experience to web developers.

Things like `setTimeout`, `setInterval`, `alert`, `confirm` and several others are provided in the global context as you can see [here](https://github.com/NativeScript/NativeScript/blob/master/tns-core-modules/globals/globals.ts#L153-L179). You can see each is mapped to an according NativeScript module which correlates to iOS or Android platform specific api's.

One important thing to note is that the browser `alert` returns `void` however in NativeScript it returns a `Promise`. When sharing code we want all of our core (low-level) api's to have common return types and behavior. This provides more opportunities for greater code sharing and consistency.

With Angular's powerful dependency injection we have the chance to enrich even the browser `alert` to return a `Promise` as well via a configurable service.

<h4 class="exercise-start">
  <b>Exercise</b>: Build our first foundational service
</h4>

To get started sharing code we want to create our first shared lib. Since we will build a foundational layer of services which will serve as the `core` to our companies stategic code sharing objective we will aptly name this `core`.

```
ng generate lib core
```

This will generate a `CoreModule` we will use to provide various foundational services to enrich our entire workspace for our entire company.

For the most part you will be building out your company's workspace on your own without any copy-and-paste guidance from us, but we are going to provide a few things. 

Create `libs/core/src/services/window.service.ts` with the following:

```
import { Injectable } from '@angular/core';

@Injectable()
export class PlatformWindowService {
  public alert(msg: any) { };
  public confirm(msg: any) { };
}

@Injectable()
export class WindowService {

  private _dialogOpened = false;

  constructor(
    private _platformWindow: PlatformWindowService,
  ) { }

  public alert(msg: any): Promise<any> {
    return new Promise((resolve, reject) => {
      if (!this._dialogOpened) {
        this._dialogOpened = true;
        this._resultHandler(this._platformWindow.alert(msg), resolve, reject, true);
      }
    });
  }

  public confirm(msg: any): Promise<any> {
    return new Promise((resolve, reject) => {
      if (!this._dialogOpened) {
        this._dialogOpened = true;
        this._resultHandler(this._platformWindow.confirm(msg), resolve, reject);
      }
    });
  }

  private _resultHandler(result: any, resolve: (result?: any) => void, reject: (reason?: any) => void, alwaysResolve?: boolean) {
    if (typeof result === 'object' && result.then) {
      result.then((result) => {
        if (alwaysResolve || result) {
          resolve(result);
        } else {
          reject();
        }
        this._dialogOpened = false;
      }, (err) => {
        reject(err);
        this._dialogOpened = false;
      });
    } else {
      if (alwaysResolve || result) {
        resolve(result);
      } else {
        reject();
      }
      this._dialogOpened = false;
    }
  }
}
```

Now create `libs/core/src/services/index.ts` with the following:

```
import { WindowService } from './window.service';

export const PROVIDERS = [
    WindowService
];

export * from './window.service';
```

Now modify our `CoreModule` to look like this:

```
import { NgModule, ModuleWithProviders, Optional, SkipSelf } from '@angular/core';
import { CommonModule } from '@angular/common';

// app
import { throwIfAlreadyLoaded } from './helpers';
import { PROVIDERS } from './services';

export const BASE_PROVIDERS: any[] = [
  ...PROVIDERS,
];

@NgModule({
  imports: [CommonModule],
})
export class CoreModule {
  static forRoot(configuredProviders: Array<any>): ModuleWithProviders {
    return {
      ngModule : CoreModule,
      providers : [
        ...BASE_PROVIDERS,
        ...configuredProviders
      ],
    };
  }

  constructor(@Optional() @SkipSelf() parentModule: CoreModule) {
    throwIfAlreadyLoaded(parentModule, 'CoreModule');
  }
}
```

There's quite a bit to unpack here so we'll discuss this however you'll notice we are using a helper we haven't created yet! So let's create our first shared helper.

Create `libs/core/src/helpers/angular.ts` with the following:

```
export function throwIfAlreadyLoaded(
  parentModule: any,
  moduleName: string,
) {
  if ( parentModule ) {
    throw new Error(`${moduleName} has already been loaded. Import ${moduleName} in the AppModule only.`);
  }
}
```

Now following our code standard also create `libs/core/src/helpers/index.ts` with the following:

```
export * from './angular';
```

<div class="exercise-end"></div>

Now let's talk about this.

### 7.2 Use shared service in web and mobile

#### Use platform provider for web

It's now time to reap the benefits of what we've setup here.

Before we use our shared service in our apps let's ensure we have exported everything we just created.
Open `libs/core/src/index.ts` and make following changes:

```
export * from './src/helpers';
export * from './src/services';
export { CoreModule } from './src/core.module';
```

Now we can share everything we created with our web and mobile apps.

Each of our apps will have their own `CoreModule`'s so we will setup our web app module now.

<h4 class="exercise-start">
  <b>Exercise</b>: Use shared lib in web app
</h4>

Create `apps/web/src/app/modules/core/core.module`:

```
import { NgModule, Optional, SkipSelf } from '@angular/core';
// libs
import {
  CoreModule as LibCoreModule,
  PlatformWindowService,
  throwIfAlreadyLoaded
} from '@mycompany/core';

// factories
export function platformWindow() {
  return window;
}

@NgModule({
  imports: [
    LibCoreModule.forRoot([
      {
        provide: PlatformWindowService,
        useFactory: platformWindow,
      }
    ])
  ]
})
export class CoreModule {
  constructor( @Optional() @SkipSelf() parentModule: CoreModule) {
    throwIfAlreadyLoaded(parentModule, 'CoreModule');
  }
}
```

Open `apps/web/src/app/app.module` and let's use this:

```
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';

// libs
import { NxModule } from '@nrwl/nx';

// app
import { CoreModule } from './modules/core/core.module';
import { AppComponent } from './app.component';

@NgModule({
  imports: [
    BrowserModule,
    NxModule.forRoot(),
    CoreModule
  ],
  declarations: [AppComponent],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

To try this out let's add some buttons and logic to our app component. Modify `apps/web/src/app/app.component.html` as follows:

```
<p>
  <button type="button" (click)="alert('Hello')">Show Alert</button>
</p>
<p>
  <button type="button" (click)="confirm('Are you sure?')">Show Confirm</button>
</p>
```

Now modify `apps/web/src/app/app.component.ts` as follows:

```
import { Component } from '@angular/core';

import { WindowService } from '@mycompany/core';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {

  constructor(
    private _win: WindowService
  ) { }

  public alert(msg: string) {
    this._win.alert(msg).then(_ => {
      console.log('alert dismissed.');
    });
  }

  public confirm(msg: string) {
    this._win.confirm(msg).then((confirmed) => {
      console.log('confirm:', confirmed);
    }, _ => {
      console.log('confirm canceled.');
    });
  }
}
```

Now try this out in the browser: `npm start`

<div class="exercise-end"></div>

Hopefully you are starting to see the benefits of what we are building.

#### Use platform provider for mobile

We now have a consistent way to provide `alert` and `confirm` dialogs with easy to use `Promise` resolution handling whether the code exists in shared code or in our web and mobile apps.

Let's now apply these same practices to mobile.

<h4 class="exercise-start">
  <b>Exercise</b>: Use shared lib in mobile app
</h4>

You will find that the NativeScript Nx template already set you up with a `CoreModule` so we can just import our shared module there. However we will want to create a slim service to help provide for our  `PlatformWindowService` on mobile.

Create `apps/mobile/app/modules/core/services/window-mobile.service.ts` with the following:

```
import { Injectable } from '@angular/core';

// libs
import { alert as tnsAlert, confirm as tnsConfirm } from 'tns-core-modules/ui/dialogs';
import { PlatformWindowService } from '@mycompany/core';

@Injectable()
export class WindowMobileService extends PlatformWindowService {

  public alert(msg: any) {
    return tnsAlert(msg);
  }

  public confirm(msg: any) {
    return tnsConfirm(msg);
  }
}
```

Modify `apps/mobile/app/modules/core/services/index.ts` with:

```
import { AppService } from './app.service';
import { WindowMobileService } from './window-mobile.service';

export const CORE_PROVIDERS = [
  AppService,
  WindowMobileService
];
```

Now modify `apps/mobile/app/modules/core/core.module.ts` as follows:

```
import { NgModule } from '@angular/core';

// libs
import { TNSFontIconModule } from 'nativescript-ngx-fonticon';
import {
  CoreModule as LibCoreModule,
  PlatformWindowService,
} from '@mycompany/core';

// app
import { CORE_PROVIDERS } from './services';
import { WindowMobileService } from './services/window-mobile.service';
import { ITEMS_PROVIDERS } from '../items/services';

@NgModule({
  imports: [
    LibCoreModule.forRoot([
      {
        provide: PlatformWindowService,
        useClass: WindowMobileService,
      },
    ]),
    TNSFontIconModule.forRoot({
      fa: './assets/font-awesome.min.css'
    })
  ],
  providers: [
    ...CORE_PROVIDERS,
    ...ITEMS_PROVIDERS
  ]
})
export class CoreModule { }
```

Then we will setup our main component much like we did for web. Modify `apps/mobile/app/modules/items/components/items/items.component.ts` since this represents our home view as follows:

```
import { Component, OnInit } from '@angular/core';

// libs
import { WindowService } from '@mycompany/core';
// app
import { Item } from '../../models';
import { ItemService } from '../../services/item.service';

@Component({
  selector: 'ns-items',
  moduleId: module.id,
  templateUrl: './items.component.html'
})
export class ItemsComponent implements OnInit {
  items: Item[];

  constructor(
    private _itemService: ItemService,
    private _win: WindowService
  ) { }

  ngOnInit(): void {
    this.items = this._itemService.getItems();
  }

  public alert(msg: string) {
    this._win.alert(msg).then(_ => {
      console.log('alert dismissed.');
    });
  }

  public confirm(msg: string) {
    this._win.confirm(msg).then((confirmed) => {
      console.log('confirm:', confirmed);
    }, _ => {
      console.log('confirm canceled.');
    });
  }
}
```

And update the template to:

```
<StackLayout class="page">
    <Button text="Show alert" (tap)="alert('Hello')" class="btn btn-primary"></Button>
    <Button text="Show confirm" (tap)="confirm('Are you sure?')" class="btn btn-primary"></Button>

    <ListView [items]="items" class="list-group">
        <ng-template let-item="item">
            <StackLayout orientation="horizontal" class="p-x-10" [nsRouterLink]="['/items', item.id]">
                <Label class="fa" [text]="'fa-futbol-o' | fonticon"></Label>
                <Label [text]="item.name" class="list-group-item"></Label>
            </StackLayout>
        </ng-template>
    </ListView>
</StackLayout>
```

With this in place you may now try running the iOS or Android script `npm run start.ios` or `npm run start.android` and would see this error:

![](images/chapter7/error1.png?raw=true)

This is because NativeScript needs libs added as a package dependency to know what to build into the app. To resolve this, create `libs/core/package.json` with the following:

```
{
  "name": "@mycompany/core",
  "version": "1.0.0",
  "main": "index",
  "typings": "index.d.ts"
}
```

We can now simply install this shared lib like anything else in our mobile app:

```
cd apps/mobile
npm i ../../libs/core --save
```

Now we can run our mobile app `npm run start.ios` and check out `WindowService` with it's own platform provider at work.

<div class="exercise-end"></div>

#### Cleanup git

If you ran `git status` now you would see `.js` files coming from `libs`. This occurs because the NativeScript app is building those out when the code is shared. We want to make sure those never end up in git history.

Open `.gitignore` and add the following to the bottom:

```
# libs
libs/**/*.js
libs/**/*.map
libs/**/*.d.ts
libs/**/*.metadata.json
libs/**/*.ngfactory.ts
libs/**/*.ngsummary.json
```

### 7.3 Locale platform provider :)

Let's continue building out our foundational service layer with a nice way to handle the default platform locale of the user.

Create `libs/core/src/helpers/tokens.ts` with the following:

```
import { InjectionToken } from '@angular/core';

export const PlatformLanguageToken = new InjectionToken<string>('PlatformLanguageToken');
```

Ensure we export our tokens from the `helpers` barrel (`libs/core/src/helpers/index.ts`):

```
export * from './angular';
export * from './tokens';
```

#### Provide web locale 

Modify `apps/web/src/app/modules/core/core.module.ts` as follows:

```
import { NgModule, Optional, SkipSelf } from '@angular/core';
// libs
import {
  CoreModule as LibCoreModule,
  PlatformWindowService,
  PlatformLanguageToken,
  throwIfAlreadyLoaded
} from '@mycompany/core';

// factories
export function platformWindow() {
  return window;
}

export function platformLanguage() {
  return window.navigator.language;
}

@NgModule({
  imports: [
    LibCoreModule.forRoot([
      {
        provide: PlatformWindowService,
        useFactory: platformWindow,
      },
      {
        provide: PlatformLanguageToken,
        useFactory: platformLanguage,
      }
    ])
  ]
})
export class CoreModule {
  constructor( @Optional() @SkipSelf() parentModule: CoreModule) {
    throwIfAlreadyLoaded(parentModule, 'CoreModule');
  }
}
```

To see this working we can just inject the token into our app component for instance (`apps/web/src/app/app.component.ts`):

```
...
import { WindowService, PlatformLanguageToken } from '@mycompany/core';

...
export class AppComponent implements OnInit {

  constructor(
    private _win: WindowService,
    @Inject(PlatformLanguageToken) private _lang: string
  ) {
    console.log('platformLanguage:', this._lang);
  }

  ...
}
```

You can run `npm start` and see the default browser locale.

#### Provide mobile device locale

Modify `apps/mobile/app/modules/core/core.module.ts` as follows:

```
...
// libs
import { TNSFontIconModule } from 'nativescript-ngx-fonticon';
import { device } from 'tns-core-modules/platform';
import {
  CoreModule as LibCoreModule,
  PlatformWindowService,
  PlatformLanguageToken,
} from '@mycompany/core';

...

// factories
export function platformLanguage() {
  return device.language;
}

@NgModule({
  imports: [
    LibCoreModule.forRoot([
      {
        provide: PlatformWindowService,
        useClass: WindowMobileService,
      },
      {
        provide: PlatformLanguageToken,
        useFactory: platformLanguage,
      },
    ]),
    ...
  ],
  ...
})
export class CoreModule { }
```

We will inject our token into the mobile app component to try this out (`apps/mobile/app/app.component.ts`):

```
...
// libs
import { PlatformLanguageToken } from '@mycompany/core';

...
export class AppComponent {
  constructor(
    // ensure singleton construction on app boot)
    private _appService: AppService,
    @Inject(PlatformLanguageToken) private _lang: string,
  ) {
    console.log('platformLanguage:', this._lang);
  }
}
```

We will now see the device locale when the iOS and Android apps start via the console. 

