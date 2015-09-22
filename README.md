# Cordova Universal Links Plugin
Plugin adds support for opening application from the browser when user clicks on some link. Better known as:
- [Universal Links on iOS](https://developer.apple.com/library/ios/documentation/General/Conceptual/AppSearch/UniversalLinks.html)
- [Deep Linking on Android](https://developer.android.com/training/app-indexing/deep-linking.html)

Basically, you can have a single link that will either open your app (if it is installed) or your website (if it's not).

Integration process is simple:
1. Add plugin to your project.
2. Define supported hosts and paths in Cordova's `config.xml`.
3. Write some JavaScript code to listen for application launch by the links.
4. Build project from the CLI.
5. Add web hooks to your website.

It is important not only to redirect user to your app from the web, but also provide him with the information he was looking for. For example, if he clicks on `http://mysite.com/news` and get redirected in the application - he probably hope to see the `news` page in it. Plugin will help developer with that. In `config.xml` you can specify event name that is dispatched when user opens the app from the certain link. This way, appropriate method of your web project will be called, and you can show user the requested content.

**Note:** At the moment plugin doesn't support custom url schemes, but may be added later if lots of people would want that.

## Supported Platforms
- Android 4.0.0 or above.
- iOS 9.0 or above. Xcode 7 is required.

## Documentation
- [Installation](#installation)
- [Cordova config preferences](#cordova-config-preferences)
- [Application launch handling](#application-launch-handling)
- [Configuring website](#configuring-website)
  - [Configure for Android](#configure-for-android)
  - [Configure for iOS](#configure-for-ios)

- [How to test your application](#how-to-test-your-application)
  - [Testing on Android](#testing-on-android)
  - [Testing on iOS](#testing-on-ios)

- [Links for additional documentation](#links-for-additional-documentation)

### Installation
This requires cordova 5.0+ (current stable 1.0.0)

```sh
cordova plugin add cordova-universal-links-plugin
```

It is also possible to install via repo url directly (**unstable**)

```sh
cordova plugin add https://github.com/nordnet/cordova-universal-links-plugin.git
```

### Cordova config preferences
Cordova uses `config.xml` file to set different project preferences: name, description, starting page and so on. Using this config file you can also set options for the plugin.

Those preferences are specified inside the `<universal-links>` block. For example:

```xml
<universal-links>
    <host name="example.com">
        <path url="/some/path" />
    </host>
</universal-links>
```

In it you define hosts and paths that application should handle. You can have as many hosts and paths as you like.

#### host
`<host />` tag lets you describe hosts, that your application supports. It can have three attributes:
- `name` - hostname. **This is a required attribute.**
- `scheme` - supported url scheme. Should be either `http` or `https`. If not set - `http` is used.
- `event` - name of the event that is dispatched on JavaScript side when application is launched from the link with the given hostname. If not set - `ul_didLaunchAppFromLink` event name is used.

For example,

```xml
<universal-links>
    <host name="example.com" scheme="https" event="ul_myExampleEvent" />
</universal-links>
```

defines, that when user clicks on any `https://example.com` link - `ul_myExampleEvent` is dispatched to the JavaScript side. You can subscribe to it and act properly. More details regarding event handling can be found [below](#application-launch-handling).

#### path
In `<path />` tag you define which paths for the given host you want to support. If no `<path />` is set - then we want to handle all host links. If paths are defined - then application will handle only links with those paths.

There are two supported attributes:
- `url` - path component of the url; should be relative to the host name. **This is a required attribute.**
- `event` - name of the event that is dispatched on JavaScript side when application is launched from the link with the given hostname and path. If not set - `ul_didLaunchAppFromLink` event name is used.

For example,

```xml
<universal-links>
    <host name="example.com">
        <path url="/some/path" />
    </host>
</universal-links>
```

defines, that when user click on `http://example.com/some/path` - application will be launched and event `ul_didLaunchAppFromLink` is send to JavaScript side. All other links from that host will be ignored.

Query parameters are not used for link matching. For example, `http://example.com/some/path?foo=bar#some_tag` will work the same way as `http://example.com/some/path` does.

In order to support all links inside `/some/path/` you can use `*` like so:

```xml
<universal-links>
    <host name="example.com">
        <path url="/some/path/*" />
    </host>
</universal-links>
```

`*` can be used only for paths, but you can place it anywhere. For example,

```xml
<universal-links>
    <host name="example.com">
        <path url="*mypath*" />
    </host>
</universal-links>
```

states, that application can handle any link from `http://example.com` which has `mypath` string in his path component: `http://example.com/some/long/mypath/test.html`, `http://example.com/testmypath.html` and so on.

**Note:** Following configuration

```xml
<universal-links>
    <host name="example.com" />
</universal-links>
```

is the same as:

```xml
<universal-links>
    <host name="example.com">
      <path url="*" />
    </host>
</universal-links>
```

### Application launch handling
As we have already mentioned - it is not enough just to redirect user into your app, you need to show him content that he was looking for. In order to help you with that - plugin will send appropriate event with url data to the JavaScript side. By default, event name is `ul_didLaunchAppFromLink`, but you can specify any name for any host/path combination by using `event` attribute.

To subscribe for default UL event in JavaScript - use `document.addEventListener` like so:

```js
document.addEventListener('ul_didLaunchAppFromLink', didLaunchAppFromLink, false);

function didLaunchAppFromLink(event) {
  var urlData = event.detail;
  console.log('Did launch application from the link: ' + urlData.url);
  // do some work
}
```

`event.detail` holds information about the launching url. For example, for `http://myhost.com/news/ul-plugin-released.html?foo=bar#cordova-news` it will be:

```json
{
  "url": "http://myhost.com/news/ul-plugin-released.html?foo=bar#cordova-news",
  "scheme": "http",
  "host": "myhost.com",
  "path": "/news/ul-plugin-released.html",
  "params": {
    "foo": "bar"
  },
  "hash": "cordova-news"
}
```

- `url` - original launch url;
- `scheme` - url scheme;
- `host` - hostname from the url;
- `path` - path component of the url;
- `params` - dictionary with query parameters; the ones that after `?` character;
- `hash` - content after `#` character.

Now it's time for some examples. In here we are gonna use Android, because it is easier to test (see [Testing on Android ](#testing-on-android) section). JavaScript side is platform independent, so all the example code below will also work for iOS.

1. Create new Cordova application and add Android platform.

  ```sh
  cordova create TestAndroidApp com.example.ul TestAndroidApp
  cd ./TestAndroidApp
  cordova platform add android
  ```

2. Add UL plugin:

  ```sh
  cordova plugin add cordova-universal-links-plugin
  ```

3. Add `<universal-links>` preference into `config.xml`:

  ```xml
  <!-- some other data from config.xml -->
  <universal-links>
   <host name="myhost.com">
     <path url="/news/" event="openNewsListPage" />
     <path url="/news/*" event="openNewsDetailedPage" />
   </host>
  </universal-links>
  ```

  As you can see - we want our application to be launched, when user goes to the `news` section of our website. And for that - we are gonna dispatch different events to understand, what has happened.

4. Subscribe to `openNewsListPage` and `openNewsDetailedPage` events. For that - open `www/js/index.js` and make it look like that:

  ```js
  var app = {
    // Application Constructor
    initialize: function() {
      this.bindEvents();
    },

    // Bind Event Listeners
    bindEvents: function() {
      document.addEventListener('deviceready', this.onDeviceReady, false);
      document.addEventListener('openNewsListPage', this.onNewsListPageRequested, false);
      document.addEventListener('openNewsDetailedPage', this.onNewsDetailedPageRequested, false);
    },

    // deviceready Event Handler
    onDeviceReady: function() {
      console.log('Handle deviceready event if you need.');
    },

    // openNewsListPage Event Handler
    onNewsListPageRequested: function(event) {
      console.log('Showing to user list of awesome news.');

      // do some work to show list of news
    },

    // openNewsDetailedPage Event Handler
    onNewsDetailedPageRequested: function(event) {
      var linkData = event.detail;
      console.log('Showing to user details page: ' + linkData.path);

      // do some work to show detailed page
    }
  };

  app.initialize();
  ```

  With that, if user clicks on `http://myhost.com/news/` link - method `onNewsListPageRequested` will be called, and for every link like `http://myhost.com/news/*` - `onNewsDetailedPageRequested`. Basically, we created a mapping between the links and JavaScript methods.

5. Build and run your application:

  ```sh
  cordova run android
  ```

6. Close your app.

7. Execute in the terminal:

  ```sh
  adb shell am start -W -a android.intent.action.VIEW -d "http://myhost.com/news/" com.example.ul
  ```

  As a result, your application will be launched, and in JavaScript console you will see message:

  ```
  Showing to user list of awesome news.
  ```

  Repeat operation, but this time with the command:

  ```sh
  adb shell am start -W -a android.intent.action.VIEW -d "http://myhost.com/news/ul-plugin-released.html" com.example.ul
  ```

  Application will be launched and you will see in JS console:

  ```
  Showing to user details page: /news/ul-plugin-released.html
  ```

Now, let's say, you want your app to handle all links from `myhost.com`, but you need to keep the mapping for the paths. For that you just need to modify your `config.xml` and add default event handler on JavaScript side:

1. Open `config.xml` and change `<universal-links>` block like so:

  ```xml
  <universal-links>
   <host name="myhost.com">
     <path url="/news/" event="openNewsListPage" />
     <path url="/news/*" event="openNewsDetailedPage" />
     <path url="*" />
   </host>
  </universal-links>
  ```

  As you can see - we added `*` as `path`. This way we declared, that application should be launched from any `http://myhost.com` link.

2. Add handling for default UL event in the `www/js/index.js`:

  ```js
  var app = {
    // Application Constructor
    initialize: function() {
      this.bindEvents();
    },

    // Bind Event Listeners
    bindEvents: function() {
      document.addEventListener('deviceready', this.onDeviceReady, false);
      document.addEventListener('openNewsListPage', this.onNewsListPageRequested, false);
      document.addEventListener('openNewsDetailedPage', this.onNewsDetailedPageRequested, false);
      document.addEventListener('ul_didLaunchAppFromLink', this.onApplicationDidLaunchFromLink, false);
    },

    // deviceready Event Handler
    onDeviceReady: function() {
      console.log('Handle deviceready event if you need');
    },

    // openNewsListPage Event Handler
    onNewsListPageRequested: function(event) {
      console.log('Showing to user list of awesome news');

      // do some work to show list of news
    },

    // openNewsDetailedPage Event Handler
    onNewsDetailedPageRequested: function(event) {
      console.log('Showing to user details page for some news');

      // do some work to show detailed page
    },

    // ul_didLaunchAppFromLink Event Handler
    onApplicationDidLaunchFromLink: function(event) {
      var linkData = event.detail;
      console.log('Did launch app from the link: ' + linkData.url);
    }
  };

  app.initialize();
  ```

That's it! Now, by default for `myhost.com` links `onApplicationDidLaunchFromLink` method will be called, but for `news` section - `onNewsListPageRequested` and `onNewsDetailedPageRequested`.

### Configuring website
#### Configure for Android
The problem is that, when you start testing this in the field, the chooser dialogue appears because the content can be handled by multiple things, including your app and browsers. The way to avoid doing that is by using app indexing. App indexing is the second part of deep linking, where you link that URI/URL between Google and your app. Even when users do a Google search, search results can bring them back to the app.

#### Configure for iOS
### How to test your application
Testing Universal Links can be a little tricky. For Android you can easily do that from console, using `adb`. But for iOS it can be painful, if you want just to try it out. Luckily, there is a way to do that by using [branch.io](http://branch.io). Read further to get more details.

#### Testing on Android
To test Android application for Deep Linking support you just need to execute the following command in the console:

```sh
adb shell am start
        -W -a android.intent.action.VIEW
        -d <URI> <PACKAGE>
```

where
- `<URI>` - url that you want to test;
- `<PACKAGE>` - your application's package name.

Let's create new application to play with:
1. Create new Cordova project and add Android platform to it:

  ```sh
  cordova create TestAndroidApp com.example.ul TestAndroidApp
  cd ./TestAndroidApp
  cordova platform add android
  ```

2. Add UL plugin:

  ```sh
  cordova plugin add cordova-universal-links-plugin
  ```

3. Add `<universal-links>` preference into `config.xml` (`TestAndroidApp/config.xml`):

  ```xml
  <!-- some other data from config.xml -->
  <universal-links>
   <host name="myhost.com" />
  </universal-links>
  ```

4. Build and run the app:

  ```sh
  cordova run android
  ```

5. Close your application and return to console.
6. Enter in console:

  ```sh
  adb shell am start -W -a android.intent.action.VIEW -d "http://myhost.com/any/path" com.example.ul
  ```

  As a result, your application will be launched and you will see in console:

  ```
  Starting: Intent { act=android.intent.action.VIEW dat=http://myhost.com/any/path pkg=com.example.ul }
  Status: ok
  Activity: com.example.ul/.MainActivity
  ThisTime: 52
  TotalTime: 52
  Complete
  ```

  If you'll try to use host (or path), that is not defined in `config.xml` - you'll get a following error:

  ```
  Starting: Intent { act=android.intent.action.VIEW dat=http://anotherhost.com/path pkg=com.example.ul }
  Error: Activity not started, unable to resolve Intent { act=android.intent.action.VIEW dat=http://anotherhost.com/path flg=0x10000000 pkg=com.example.ul }
  ```

This way you can experiment with your Android application and check how it corresponds to different links.

#### Testing on iOS


### Links for additional documentation
Android:
- [Enable Deep Linking on Android](https://developer.android.com/training/app-indexing/deep-linking.html)
- [Specifying App Content for Indexing](https://developer.android.com/training/app-indexing/enabling-app-indexing.html)
- [Video tutorial on Android App Indexing](https://realm.io/news/juan-gomez-android-app-indexing/)

iOS:
- [Apple documentation on Universal Links](https://developer.apple.com/library/ios/documentation/General/Conceptual/AppSearch/UniversalLinks.html)
- [How to setup universal links on iOS 9](https://blog.branch.io/how-to-setup-universal-links-to-deep-link-on-apple-ios-9)
- [Branch.io documentation on universal links](https://dev.branch.io/recipes/branch_universal_links/#enable-universal-links-on-the-branch-dashboard)
