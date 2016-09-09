# How to use standard appium 1.5.3 (unpatched) with Crosswalk

Start Appium 1.5.3.

Download the patched chromedriver from this repository releases (https://github.com/eduramiba/appium/releases/tag/1.5.3 linux only). This driver has an additional `androidDeviceSocket` capability.

Then run a test using `androidDeviceSocket` capability in both appium and `chromeOptions`.

## Example test with Webdriverio 4.2.1 and Mocha

```javascript
var chromeDriverPath = path.resolve('/path/to/chromedriver_2_23_patched');
var apkPath = path.resolve('/path/to/apk/with/crosswalk/app.apk');

var package = 'your.app.package';
var socketName = package+ '_devtools_remote';
var client = webdriverio.remote({
    host: 'localhost',
    port: 4723,
    desiredCapabilities: {
        'appiumVersion': '1.5.3',
        'platformName': 'Android',
        'deviceName': 'Android Emulator',
        'device-orientation': 'portrait',
        'chromedriverExecutable': chromeDriverPath,
        'appPackage': package,
        'app': apkPath,
        'androidDeviceSocket': socketName,
        'chromeOptions': {
            'androidDeviceSocket': socketName
        }
    }
});

// Similar to appium's autoWebview but for any context name
function autoContext(contextName, timeout, pollInterval) {
    if (!pollInterval) {
        pollInterval = 100;
    }

    if (!timeout) {
        timeout = 15000;
    }
    
    var startTime = Date.now();
    var client = this;

    return new Promise(function (done, reject) {
        function checkContexts() {
            client.contexts().then(function (contexts) {
                if (contexts.value.indexOf(contextName) !== -1) {
                    done();
                } else {
                    if (Date.now() - startTime >= timeout) {
                        reject('Could not switch to context ' + contextName + ' after ' + timeout + ' milliseconds');
                    } else {
                        setTimeout(checkContexts, pollInterval);
                    }
                }
            });
        }
        
        checkContexts();
    }).then(function () {
        return client.context(contextName);
    });
}

client.addCommand('autoContext', autoContext);

describe('Cordova Crosswalk Application', function () {
    before(function () {
        return client.init().getCurrentDeviceActivity().then(function (activity) {
            console.log('Current activity = ' + activity);
        }).autoContext('CHROMIUM');
    });

    after(function () {
        return client.end();
    });

    it('should have the right title', function () {
        return client.getTitle().then(function (title) {
            assert.equal(title, 'Cordova Crosswalk Example');
        });
    });

    it('Device ready', function () {
        return client.isVisible('p.event.received').then(function (visible) {
            assert(visible);
        }).getText('p.event.received').then(function (text) {
            assert.equal(text, "DEVICE IS READY");
        });
    });
});
```

Then run `mocha --timeout=60000 test.js` for example.

# How to build a patched chromedriver

We will follow the instructions on https://www.chromium.org/developers/how-tos/get-the-code and https://chromium.googlesource.com/chromium/src/+/master/tools/gn/docs/quick_start.md

On linux, I run this:

```bash
# Install depot tools
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH=$PATH:/path/to/depot_tools #You can add this to your .bashrc or similar

# Clone chromium code (takes some time...)
mkdir chromium
cd chromium

fetch chromium
cd src
./build/install-build-deps.sh # Install packages needed to compile

gclient runhooks

# Generate out directory and default config
gn gen out/Default "--args=is_debug=false" # Configure a non-debug, statically linked build profile

# Apply the patch (first copy it to src directory):
git apply androidDeviceSocket.patch

# Compile chromedriver
ninja -C out/Default chromedriver
```

Finally, chromedriver executable will be available at ```out/Default/chromedriver``` :)

**The build is OS dependant**

# NOTES

Always use a 64 bit OS to compile chromedriver, or you will run out of memory.

If you want to compile a 32 bit version of chromedriver from a 64 bit OS, change the `gn gen ...` command to:
```bash
gn gen out/Default "--args=is_debug=false target_cpu=\"x86\""
```

## The patch (androidDeviceSocket.patch)

```diff
From a3a60514cd32f578a7d8458c51e33c880096d96d Mon Sep 17 00:00:00 2001
From: Eduardo Ramos <eduramiba@gmail.com>
Date: Mon, 8 Aug 2016 16:36:51 +0200
Subject: [PATCH] Apply androidDeviceSocket patch

---
 chrome/test/chromedriver/capabilities.cc          |  2 ++
 chrome/test/chromedriver/capabilities.h           |  2 ++
 chrome/test/chromedriver/chrome/device_manager.cc | 14 ++++++++------
 chrome/test/chromedriver/chrome/device_manager.h  |  1 +
 chrome/test/chromedriver/chrome_launcher.cc       |  1 +
 5 files changed, 14 insertions(+), 6 deletions(-)

diff --git a/chrome/test/chromedriver/capabilities.cc b/chrome/test/chromedriver/capabilities.cc
index 820cf7d..0d54360 100644
--- a/chrome/test/chromedriver/capabilities.cc
+++ b/chrome/test/chromedriver/capabilities.cc
@@ -425,6 +425,8 @@ Status ParseChromeOptions(
         base::Bind(&ParseString, &capabilities->android_package);
     parser_map["androidProcess"] =
         base::Bind(&ParseString, &capabilities->android_process);
+    parser_map["androidDeviceSocket"] =
+        base::Bind(&ParseString, &capabilities->android_device_socket);
     parser_map["androidUseRunningApp"] =
         base::Bind(&ParseBoolean, &capabilities->android_use_running_app);
     parser_map["args"] = base::Bind(&ParseSwitches);
diff --git a/chrome/test/chromedriver/capabilities.h b/chrome/test/chromedriver/capabilities.h
index c11bd4b..5e71844 100644
--- a/chrome/test/chromedriver/capabilities.h
+++ b/chrome/test/chromedriver/capabilities.h
@@ -108,6 +108,8 @@ struct Capabilities {
 
   std::string android_process;
 
+  std::string android_device_socket;
+
   bool android_use_running_app;
 
   base::FilePath binary;
diff --git a/chrome/test/chromedriver/chrome/device_manager.cc b/chrome/test/chromedriver/chrome/device_manager.cc
index 7627f3f..ac1c8b0 100644
--- a/chrome/test/chromedriver/chrome/device_manager.cc
+++ b/chrome/test/chromedriver/chrome/device_manager.cc
@@ -35,6 +35,7 @@ Device::~Device() {
 Status Device::SetUp(const std::string& package,
                      const std::string& activity,
                      const std::string& process,
+                     const std::string& device_socket,
                      const std::string& args,
                      bool use_running_app,
                      int port) {
@@ -48,19 +49,19 @@ Status Device::SetUp(const std::string& package,
 
   std::string known_activity;
   std::string command_line_file;
-  std::string device_socket;
+  std::string known_device_socket = device_socket;
   std::string exec_name;
   if (package.compare("org.chromium.content_shell_apk") == 0) {
     // Chromium content shell.
     known_activity = ".ContentShellActivity";
-    device_socket = "content_shell_devtools_remote";
+    known_device_socket = "content_shell_devtools_remote";
     command_line_file = "/data/local/tmp/content-shell-command-line";
     exec_name = "content_shell";
   } else if (package.find("chrome") != std::string::npos &&
              package.find("webview") == std::string::npos) {
     // Chrome.
     known_activity = "com.google.android.apps.chrome.Main";
-    device_socket = "chrome_devtools_remote";
+    known_device_socket = "chrome_devtools_remote";
     command_line_file = kChromeCmdLineFileBeforeM33;
     exec_name = "chrome";
     status = adb_->SetDebugApp(serial_, package);
@@ -75,9 +76,10 @@ Status Device::SetUp(const std::string& package,
 
     if (!known_activity.empty()) {
       if (!activity.empty() ||
-          !process.empty())
+          !process.empty() ||
+          !device_socket.empty())
         return Status(kUnknownError, "known package " + package +
-                      " does not accept activity/process");
+                      " does not accept activity/process/device_socket");
     } else if (activity.empty()) {
       return Status(kUnknownError, "WebView apps require activity name");
     }
@@ -109,7 +111,7 @@ Status Device::SetUp(const std::string& package,
 
     active_package_ = package;
   }
-  this->ForwardDevtoolsPort(package, process, port, &device_socket);
+  this->ForwardDevtoolsPort(package, process, port, &known_device_socket);
 
   return status;
 }
diff --git a/chrome/test/chromedriver/chrome/device_manager.h b/chrome/test/chromedriver/chrome/device_manager.h
index cce404e..d72cc0c 100644
--- a/chrome/test/chromedriver/chrome/device_manager.h
+++ b/chrome/test/chromedriver/chrome/device_manager.h
@@ -26,6 +26,7 @@ class Device {
   Status SetUp(const std::string& package,
                const std::string& activity,
                const std::string& process,
+               const std::string& device_socket,
                const std::string& args,
                bool use_running_app,
                int port);
diff --git a/chrome/test/chromedriver/chrome_launcher.cc b/chrome/test/chromedriver/chrome_launcher.cc
index 1a74e44..3d83809 100644
--- a/chrome/test/chromedriver/chrome_launcher.cc
+++ b/chrome/test/chromedriver/chrome_launcher.cc
@@ -492,6 +492,7 @@ Status LaunchAndroidChrome(
   status = device->SetUp(capabilities.android_package,
                          capabilities.android_activity,
                          capabilities.android_process,
+                         capabilities.android_device_socket,
                          switches.ToString(),
                          capabilities.android_use_running_app,
                          port);
-- 
2.7.4
## 
```
