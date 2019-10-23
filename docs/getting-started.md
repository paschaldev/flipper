---
id: getting-started
title: Getting Started
---

Flipper helps you debug Android and iOS apps running in an emulator/simulator or connected physical development devices. Flipper consists of two parts:

- The desktop app
- The native mobile SDKs for Android and iOS

To use Flipper, you need to add the mobile SDK to your app.

## Setup

### Desktop app

The desktop part of Flipper doesn't need any particular setup. Simply download the latest build for [Mac](https://www.facebook.com/fbflipper/public/mac), [Linux](https://www.facebook.com/fbflipper/public/linux) or [Windows](https://www.facebook.com/fbflipper/public/windows) and launch it. In order to work properly, Flipper requires a working installation of the Android and (if where applicable) iOS development tools on your system, as well as the [OpenSSL](https://www.openssl.org) binary on your `$PATH`.

Once you start Flipper and launch an emulator/simulator or connect a device, you will already be able to see the device logs in Flipper. To see app specific data, you need to integrate our native SDKs with your app.

![Logs plugin](/docs/assets/initial.png)

## Setup your Android app

It's recommended that you add the following activity to the manifest, which can help diagnose integration issues and other problems:

```xml
<activity android:name="com.facebook.flipper.android.diagnostics.FlipperDiagnosticActivity"
        android:exported="true"/>
```

Flipper is distributed via JCenter. Add the dependencies to your `build.gradle` file.
You should also explicitly depend on [`soloader`](https://github.com/facebook/soloader)
instead of relying on transitive dependency resolution which is getting deprecated
with Gradle 5.

We provide a "no-op" implementation of some oft-used Flipper interfaces you can
use to make it easier to strip Flipper from your release builds.

```groovy
repositories {
  jcenter()
}

dependencies {
  debugImplementation 'com.facebook.flipper:flipper:0.26.0'
  debugImplementation 'com.facebook.soloader:soloader:0.5.1'

  releaseImplementation 'com.facebook.flipper:flipper-noop:0.26.0'
}
```

Now you can initialize Flipper in your Application's `onCreate` method, which involves
initializing SoLoader (for loading the C++ part of Flipper) and starting a `FlipperClient`.

```java
import com.facebook.flipper.android.AndroidFlipperClient;
import com.facebook.flipper.android.utils.FlipperUtils;
import com.facebook.flipper.core.FlipperClient;

public class MyApplication extends Application {

  @Override
  public void onCreate() {
    super.onCreate();
    SoLoader.init(this, false);

    if (BuildConfig.DEBUG && FlipperUtils.shouldEnableFlipper(this)) {
      final FlipperClient client = AndroidFlipperClient.getInstance(this);
      client.addPlugin(new InspectorFlipperPlugin(this, DescriptorMapping.withDefaults()));
      client.start();
    }
  }
}
```

### Android Snapshots

Feeling adventurous? We publish Android snapshot releases directly off of `master`.

You can get the latest version by adding the Maven Snapshot repository to your sources
and pointing to the most recent `-SNAPSHOT` version.

```groovy
repositories {
  maven { url 'https://oss.sonatype.org/content/repositories/snapshots/' }
}

dependencies {
  debugImplementation 'com.facebook.flipper:flipper:0.26.1-SNAPSHOT'
  debugImplementation 'com.facebook.soloader:soloader:0.5.1'

  releaseImplementation 'com.facebook.flipper:flipper-noop:0.26.1-SNAPSHOT'
}
```

## Setup your iOS app

We support both Swift and Objective-C for Flipper with CocoaPods as build and distribution mechanism. For CocoaPods 1.7+ following is the configuration.

### CocoaPods

<!--DOCUSAURUS_CODE_TABS-->
<!--Objective-C-->

```ruby
project 'MyApp.xcodeproj'
flipperkit_version = '0.26.0'

target 'MyApp' do
  platform :ios, '9.0'
  # use_framework!
  pod 'FlipperKit', '~>' + flipperkit_version
  pod 'FlipperKit/FlipperKitLayoutComponentKitSupport', '~>' + flipperkit_version
  pod 'FlipperKit/SKIOSNetworkPlugin', '~>' + flipperkit_version
  pod 'FlipperKit/FlipperKitUserDefaultsPlugin', '~>' + flipperkit_version

  # If you use `use_frameworks!` in your Podfile,
  # uncomment the below $static_framework array and also
  # the pre_install section.  This will cause Flipper and
  # it's dependencies to be built as a static library and all other pods to
  # be dynamic.
  # $static_framework = ['FlipperKit', 'Flipper', 'Flipper-Folly',
  #   'CocoaAsyncSocket', 'ComponentKit', 'DoubleConversion',
  #   'glog', 'Flipper-PeerTalk', 'Flipper-RSocket', 'Yoga', 'YogaKit',
  #   'CocoaLibEvent', 'OpenSSL-Universal', 'boost-for-react-native']
  #
  # pre_install do |installer|
  #   Pod::Installer::Xcode::TargetValidator.send(:define_method, :verify_no_static_framework_transitive_dependencies) {}
  #   installer.pod_targets.each do |pod|
  #       if $static_framework.include?(pod.name)
  #         def pod.build_type;
  #           Pod::Target::BuildType.static_library
  #         end
  #       end
  #     end
  # end

  # This post_install hook adds the -DFB_SONARKIT_ENABLED=1 flag to OTHER_CFLAGS, necessary to expose Flipper classes in the header files
  post_install do |installer|
    installer.pods_project.targets.each do |target|
      if target.name == 'YogaKit'
        target.build_configurations.each do |config|
          config.build_settings['SWIFT_VERSION'] = '4.1'
        end
      end
    end
    file_name = Dir.glob("*.xcodeproj")[0]
    app_project = Xcodeproj::Project.open(file_name)
    app_project.native_targets.each do |target|
      target.build_configurations.each do |config|
        cflags = config.build_settings['OTHER_CFLAGS'] || '$(inherited) '
        unless cflags.include? '-DFB_SONARKIT_ENABLED=1'
          puts 'Adding -DFB_SONARKIT_ENABLED=1 in OTHER_CFLAGS...'
          cflags << '-DFB_SONARKIT_ENABLED=1'
        end
        config.build_settings['OTHER_CFLAGS'] = cflags
      end
      app_project.save
    end
    installer.pods_project.save
   end
end
```

<!--Swift-->

```ruby
project 'MyApp.xcodeproj'
flipperkit_version = '0.26.0'

target 'MyApp' do
  platform :ios, '9.0'

  pod 'FlipperKit', '~>' + flipperkit_version
  # Layout and network plugins are not yet supported for swift projects
  pod 'FlipperKit/FlipperKitLayoutComponentKitSupport', '~>' + flipperkit_version
  pod 'FlipperKit/SKIOSNetworkPlugin', '~>' + flipperkit_version
  pod 'FlipperKit/FlipperKitUserDefaultsPlugin', '~>' + flipperkit_version

  # If you use `use_frameworks!` in your Podfile,
  # uncomment the below $static_framework array and also
  # the pre_install section.  This will cause Flipper and
  # it's dependencies to be built as a static library and all other pods to
  # be dynamic.
  # $static_framework = ['FlipperKit', 'Flipper', 'Flipper-Folly',
  #   'CocoaAsyncSocket', 'ComponentKit', 'DoubleConversion',
  #   'glog', 'Flipper-PeerTalk', 'Flipper-RSocket', 'Yoga', 'YogaKit',
  #   'CocoaLibEvent', 'OpenSSL-Universal', 'boost-for-react-native']
  #
  # pre_install do |installer|
  #   Pod::Installer::Xcode::TargetValidator.send(:define_method, :verify_no_static_framework_transitive_dependencies) {}
  #   installer.pod_targets.each do |pod|
  #       if $static_framework.include?(pod.name)
  #         def pod.build_type;
  #           Pod::Target::BuildType.static_library
  #         end
  #       end
  #     end
  # end


  # This post_install hook adds the -DFB_SONARKIT_ENABLED flag to OTHER_SWIFT_FLAGS, necessary to build swift target
  post_install do |installer|
    installer.pods_project.targets.each do |target|
      if target.name == 'YogaKit'
        target.build_configurations.each do |config|
          config.build_settings['SWIFT_VERSION'] = '4.1'
        end
      end
    end
    file_name = Dir.glob("*.xcodeproj")[0]
    app_project = Xcodeproj::Project.open(file_name)
    app_project.native_targets.each do |target|
        target.build_configurations.each do |config|
          if (config.build_settings['OTHER_SWIFT_FLAGS'])
            unless config.build_settings['OTHER_SWIFT_FLAGS'].include? '-DFB_SONARKIT_ENABLED'
              puts 'Adding -DFB_SONARKIT_ENABLED ...'
              swift_flags = config.build_settings['OTHER_SWIFT_FLAGS']
              if swift_flags.split.last != '-Xcc'
                config.build_settings['OTHER_SWIFT_FLAGS'] << ' -Xcc'
              end
              config.build_settings['OTHER_SWIFT_FLAGS'] << ' -DFB_SONARKIT_ENABLED'
            end
          else
            puts 'OTHER_SWIFT_FLAGS does not exist thus assigning it to `$(inherited) -Xcc -DFB_SONARKIT_ENABLED`'
            config.build_settings['OTHER_SWIFT_FLAGS'] = '$(inherited) -Xcc -DFB_SONARKIT_ENABLED'
          end
          app_project.save
        end
      end
      installer.pods_project.save
  end
end
```

<!--END_DOCUSAURUS_CODE_TABS-->

You need to compile your project with the `FB_SONARKIT_ENABLED=1` compiler flag. The above `post_install` hook adds this compiler flag to your project settings.

<div class="warning">

On the first run of `pod install`, `FB_SONARKIT_ENABLED=1` may not be added in the "Build Settings" of your project, but in all the subsequent runs of `pod install`, the above `post_install` hook successfully adds the compiler flag. So before running your app, make sure that `FB_SONARKIT_ENABLED=1` is present in `OTHER_CFLAGS` and `OTHER_SWIFT_FLAGS` for Objective-C and Swift projects respectively.

</div>

Install the dependencies by running `pod install`. You can now import and initialize Flipper in your
AppDelegate.

<!--DOCUSAURUS_CODE_TABS-->

<!--Objective-C-->

```objective-c
#import <FlipperKit/FlipperClient.h>
#import <FlipperKitLayoutPlugin/FlipperKitLayoutPlugin.h>
#import <FlipperKitLayoutComponentKitSupport/FlipperKitLayoutComponentKitSupport.h>
#import <FlipperKitUserDefaultsPlugin/FKUserDefaultsPlugin.h>
#import <FlipperKitNetworkPlugin/FlipperKitNetworkPlugin.h>
#import <SKIOSNetworkPlugin/SKIOSNetworkAdapter.h>

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
  FlipperClient *client = [FlipperClient sharedClient];
  SKDescriptorMapper *layoutDescriptorMapper = [[SKDescriptorMapper alloc] initWithDefaults];
  [FlipperKitLayoutComponentKitSupport setUpWithDescriptorMapper: layoutDescriptorMapper];
  [client addPlugin: [[FlipperKitLayoutPlugin alloc] initWithRootNode: application
                                                 withDescriptorMapper: layoutDescriptorMapper]];

  [client addPlugin:[[FKUserDefaultsPlugin alloc] initWithSuiteName:nil]];
  [client addPlugin: [[FlipperKitNetworkPlugin alloc] initWithNetworkAdapter:[SKIOSNetworkAdapter new]]];
  [client start];
  ...
}
@end
```

<!--Swift-->

```swift
import UIKit
import FlipperKit
@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {

  var window: UIWindow?


  func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    // Override point for customization after application launch.
    let client = FlipperClient.shared()
    let layoutDescriptorMapper = SKDescriptorMapper(defaults: ())
    FlipperKitLayoutComponentKitSupport.setUpWith(layoutDescriptorMapper)
    client?.add(FlipperKitLayoutPlugin(rootNode: application, with: layoutDescriptorMapper!))
    client?.start()
    return true
  }
}
```

<!--END_DOCUSAURUS_CODE_TABS-->

## Ready for takeoff

Finally, you need to add plugins to your Flipper client. Above we have only added the Layout Inspector plugin to get you started. See [Network Plugin](setup/network-plugin.md) and [Layout Inspector Plugin](setup/layout-plugin.md) for information on how to add them, and also enable Litho or ComponentKit support. You can check the sample apps in the [GitHub repo](https://github.com/facebook/flipper) for examples of integrating other plugins.

## Setup your React Native app

_Inspired by [a blog post by Ram N](http://blog.nparashuram.com/2019/09/using-flipper-with-react-native.html)._

Integrating Flipper with React Native is a bit different than with a native app.

### Android

First, add this line to your `android/gradle.properties`:

```groovy
# On Android, React Native currently has issues with higher versions
FLIPPER_VERSION=0.23.4
```

Add the following permissions to your `AndroidManifest.xml`. The SDK needs these to communicate with the desktop app on localhost via adb. It won't make any external internet requests.

`android/app/src/main/AndroidManifest.xml`

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
```

It's recommended that you add the following activity to the manifest too, which can help diagnose integration issues and other problems:

```xml
<activity android:name="com.facebook.flipper.android.diagnostics.FlipperDiagnosticActivity"
        android:exported="true"/>
```

Flipper is distributed via JCenter. Add the dependencies to your `build.gradle` file.
You should also explicitly depend on [`soloader`](https://github.com/facebook/soloader)
instead of relying on transitive dependency resolution which is getting deprecated
with Gradle 5.

We provide a "no-op" implementation of some oft-used Flipper interfaces you can
use to make it easier to strip Flipper from your release builds.

`android/app/build.gradle`

```groovy
android {
  packagingOptions {
    ...
    // This line is required to prevent React Native from crash
    pickFirst '**/libc++_shared.so'
  }
}

dependencies {
  ...
  debugImplementation("com.facebook.flipper:flipper:${FLIPPER_VERSION}") {
    exclude group:'com.facebook.yoga'
    exclude group:'com.facebook.flipper', module: 'fbjni'
    exclude group:'com.facebook.litho', module: 'litho-annotations'
    exclude group:'com.squareup.okhttp3'
  }
}
```

Now, we create a new file inside `android/app/src/debug/java/com/yourappname/ReactNativeFlipper.java`.

These are the suggested plugins integrations:

- Layout Inspector
- Network
- Databases
- Images
- Shared Preferences
- Crash Reporter
- React devtools

```java
package com.yourappname;

import android.content.Context;
import com.facebook.flipper.android.AndroidFlipperClient;
import com.facebook.flipper.android.utils.FlipperUtils;
import com.facebook.flipper.core.FlipperClient;
import com.facebook.flipper.plugins.crashreporter.CrashReporterPlugin;
import com.facebook.flipper.plugins.databases.DatabasesFlipperPlugin;
import com.facebook.flipper.plugins.fresco.FrescoFlipperPlugin;
import com.facebook.flipper.plugins.inspector.DescriptorMapping;
import com.facebook.flipper.plugins.inspector.InspectorFlipperPlugin;
import com.facebook.flipper.plugins.network.FlipperOkhttpInterceptor;
import com.facebook.flipper.plugins.network.NetworkFlipperPlugin;
import com.facebook.flipper.plugins.react.ReactFlipperPlugin;
import com.facebook.flipper.plugins.sharedpreferences.SharedPreferencesFlipperPlugin;
import com.facebook.react.ReactInstanceManager;
import com.facebook.react.bridge.ReactContext;
import com.facebook.react.modules.network.NetworkingModule;
import okhttp3.OkHttpClient;

public class ReactNativeFlipper {

  public static void initializeFlipper(Context context, final ReactInstanceManager reactInstanceManager) {
    if (!FlipperUtils.shouldEnableFlipper(context)) {
      return;
    }
    final FlipperClient client = AndroidFlipperClient.getInstance(context);

    client.addPlugin(new InspectorFlipperPlugin(context, DescriptorMapping.withDefaults()));
    client.addPlugin(new ReactFlipperPlugin());
    client.addPlugin(new DatabasesFlipperPlugin(context));
    client.addPlugin(new SharedPreferencesFlipperPlugin(context));
    client.addPlugin(CrashReporterPlugin.getInstance());

    final NetworkFlipperPlugin networkFlipperPlugin = new NetworkFlipperPlugin();
    NetworkingModule.setCustomClientBuilder(
        new NetworkingModule.CustomClientBuilder() {
          @Override
          public void apply(OkHttpClient.Builder builder) {
            builder.addNetworkInterceptor(new FlipperOkhttpInterceptor(networkFlipperPlugin));
          }
        });
    client.addPlugin(networkFlipperPlugin);
    client.start();

    // Fresco Plugin needs to ensure that ImagePipelineFactory is initialized
    // Hence we run if after all native modules have been initialized
    ReactContext reactContext = reactInstanceManager.getCurrentReactContext();
    if (reactContext == null) {
      reactInstanceManager.addReactInstanceEventListener(
          new ReactInstanceManager.ReactInstanceEventListener() {
            @Override
            public void onReactContextInitialized(ReactContext reactContext) {
              reactInstanceManager.removeReactInstanceEventListener(this);
              reactContext.runOnNativeModulesQueueThread(
                  new Runnable() {
                    @Override
                    public void run() {
                      client.addPlugin(new FrescoFlipperPlugin());
                    }
                  });
            }
          });
    } else {
      client.addPlugin(new FrescoFlipperPlugin());
    }
  }
}
```

Now you can initialize Flipper in your Application's `onCreate` method, which involves
initializing SoLoader (for loading the C++ part of Flipper) and starting a `FlipperClient`.

For this, we edit the `android/app/src/main/java/com/yourappname/MainApplication.java` file.

```java
package com.yourappname;

import ...
import com.facebook.react.ReactInstanceManager;

public class MainApplication extends Application implements ReactApplication {
  ...

  @Override
  public void onCreate() {
    ...
    initializeFlipper(this, getReactNativeHost().getReactInstanceManager());
  }

  /**
   * Loads Flipper in React Native templates. Call this in the onCreate method with something like
   * initializeFlipper(this, getReactNativeHost().getReactInstanceManager());
   *
   * @param context
   */
  private static void initializeFlipper(Context context, ReactInstanceManager reactInstanceManager) {
    if (BuildConfig.DEBUG) {
      try {
        Class<?> aClass = Class.forName("com.yourappname.ReactNativeFlipper");
        aClass
          .getMethod("initializeFlipper", Context.class, ReactInstanceManager.class)
          .invoke(null, context, reactInstanceManager);
      } catch (ClassNotFoundException e) {
        e.printStackTrace();
      } catch (NoSuchMethodException e) {
      e.printStackTrace();
      } catch (IllegalAccessException e) {
        e.printStackTrace();
      } catch (InvocationTargetException e) {
        e.printStackTrace();
      }
    }
  }
}
```

Finally, open the Flipper desktop app, and run `yarn android` in your terminal.

### iOS

We support both Swift and Objective-C for Flipper with CocoaPods as build and distribution mechanism. For CocoaPods 1.7+ following is the configuration.

#### CocoaPods

`ios/Podfile`

<!--DOCUSAURUS_CODE_TABS-->
<!--Objective-C-->

```ruby
platform :ios, '9.0'

def flipper_pods()
  flipperkit_version = '0.25'
  pod 'FlipperKit', '~>' + flipperkit_version, :configuration => 'Debug'
  pod 'FlipperKit/FlipperKitLayoutPlugin', '~>' + flipperkit_version, :configuration => 'Debug'
  pod 'FlipperKit/SKIOSNetworkPlugin', '~>' + flipperkit_version, :configuration => 'Debug'
  pod 'FlipperKit/FlipperKitUserDefaultsPlugin', '~>' + flipperkit_version, :configuration => 'Debug'
end

# Post Install processing for Flipper
def flipper_post_install(installer)
  installer.pods_project.targets.each do |target|
    if target.name == 'YogaKit'
      target.build_configurations.each do |config|
        config.build_settings['SWIFT_VERSION'] = '4.1'
      end
    end
  end
  file_name = Dir.glob("*.xcodeproj")[0]
  app_project = Xcodeproj::Project.open(file_name)
  app_project.native_targets.each do |target|
    target.build_configurations.each do |config|
      cflags = config.build_settings['OTHER_CFLAGS'] || '$(inherited) '
      unless cflags.include? '-DFB_SONARKIT_ENABLED=1'
        puts 'Adding -DFB_SONARKIT_ENABLED=1 in OTHER_CFLAGS...'
        cflags << '-DFB_SONARKIT_ENABLED=1'
      end
      config.build_settings['OTHER_CFLAGS'] = cflags
    end
    app_project.save
  end
  installer.pods_project.save
end

target 'your-app-name' do
  ...

  target 'your-app-nameTests' do
    inherit! :complete
    # Pods for testing
  end

  # For enabling Flipper.
  # Note that if you use_framework!, flipper will no work.
  # Disable these lines if you are doing use_framework!
  flipper_pods()
  post_install do |installer|
    flipper_post_install(installer)
  end
end
```

<!--END_DOCUSAURUS_CODE_TABS-->

You need to compile your project with the `FB_SONARKIT_ENABLED=1` compiler flag. The above `post_install` hook adds this compiler flag to your project settings.

<div class="warning">

On the first run of `pod install`, `FB_SONARKIT_ENABLED=1` may not be added in the "Build Settings" of your project, but in all the subsequent runs of `pod install`, the above `post_install` hook successfully adds the compiler flag. So before running your app, make sure that `FB_SONARKIT_ENABLED=1` is present in `OTHER_CFLAGS` and `OTHER_SWIFT_FLAGS` for Objective-C and Swift projects respectively.

</div>

Install the dependencies by running `cd ios && pod install`. You can now import and initialize Flipper in your
`ios/your-app-name/AppDelegate.m`.

The code below enables the following integrations:

- Layout Inspector
- Network
- Shared Preferences
- Crash Reporter

<!--DOCUSAURUS_CODE_TABS-->

<!--Objective-C-->

```objective-c
...
#ifdef DEBUG
  #import <FlipperKit/FlipperClient.h>
  #import <FlipperKitLayoutPlugin/FlipperKitLayoutPlugin.h>
  #import <FlipperKitLayoutPlugin/SKDescriptorMapper.h>
  #import <FlipperKitUserDefaultsPlugin/FKUserDefaultsPlugin.h>
  #import <FlipperKitNetworkPlugin/FlipperKitNetworkPlugin.h>
  #import <SKIOSNetworkPlugin/SKIOSNetworkAdapter.h>
  #import <FlipperKitNetworkPlugin/FlipperKitNetworkPlugin.h>
#endif

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
  [self initializeFlipper:application];
  ...
}

- (void) initializeFlipper:(UIApplication *)application {
  #ifdef DEBUG
    FlipperClient *client = [FlipperClient sharedClient];
    SKDescriptorMapper *layoutDescriptorMapper = [[SKDescriptorMapper alloc] initWithDefaults];
    [client addPlugin: [[FlipperKitLayoutPlugin alloc] initWithRootNode: application withDescriptorMapper: layoutDescriptorMapper]];
    [client addPlugin:[[FKUserDefaultsPlugin alloc] initWithSuiteName:nil]]; [client start];
    [client addPlugin: [[FlipperKitNetworkPlugin alloc] initWithNetworkAdapter:[SKIOSNetworkAdapter new]]];
    [client start];
  #endif
}

@end
```

<!--END_DOCUSAURUS_CODE_TABS-->

Lastly, open the Flipper desktop app, and run `yarn ios` in your terminal.

<div class="warning">

- If you do not use CocoaPods as a dependency management tool then currently there is no way to integrate FlipperKit other than manually including all the dependencies and building it.
- For Android, Flipper works with both emulators and physical devices connected through USB. However on iOS, we don't yet support physical devices.

</div>

## Having trouble?

See the [troubleshooting page](troubleshooting.html) for help with known problems.
