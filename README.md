## Appium with support for Crosswalk

Note: even the lastest xwalkdriver release has this bug: https://bugs.chromium.org/p/chromedriver/issues/detail?id=1225, better solution at https://github.com/eduramiba/appium

This is a Appium fork based on version 1.5.3 with a custom [appium-android-driver](https://github.com/eduramiba/appium-android-driver/tree/1.5.crosswalk) that is able to connect to Crosswalk applications using the [crosswalk webdriver](https://github.com/crosswalk-project/crosswalk-web-driver/tree/xwalkdriver_for_crosswalk_16/bin) as chrome driver executable.

Package available at https://www.npmjs.com/package/appium-crosswalk

You can connect to this version of appium with WebdriverIO for example with the following code:

```javascript
var client = webdriverio.remote({
    host: 'localhost',
    port: 4723,
    desiredCapabilities: {
        'appiumVersion': '1.5.3',
        'platformName': 'Android',
        'deviceName': 'Android Emulator',
        'device-orientation': 'portrait',
        'autoWebview': true,
        'autoWebviewTimeout': 15000,
        'chromedriverExecutable': '/path/to/xwalkdriver64_xwalk_15',
        'appPackage': package,
        'app': apkPath
    }
});
```

This has been tested with a Cordova application and crosswalk 19+ plugin.

Based on the solution at https://github.com/hamsterksu/appium-xwalk but with a more recent version of appium.

This patch does not break support for normal Android Webview support, it should work with both types of webviews.
