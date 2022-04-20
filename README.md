# React Native Fastlane

The best approach to efficient react native builds and distributions.

## Prerequisites

```bash
$ npx react-native init MyApp --template react-native-template-typescript
$ cd MyApp
$ touch .env .env.staging .env.production
```

## Dependencies

1. [Bundler](https://bundler.io/)
2. [Homebrew](https://brew.sh/)
3. ImageMagick - `brew install imagemagick`
4. [react-native-config](https://github.com/luggit/react-native-config)

## Setup

For newly initialized React Native projects, follow the steps below to standardize the project for the lanes you will use. If you're adding onto an existing project, you can skip the steps you've already done. Pay attention to the naming conventions as they are very important.

### **iOS**

#### **Step 1** - Create Build Configuration

1. Open the Xcode Workspace (\*.xcworkspace)
2. Click on the project, then the “+” icon under the “Configurations” section and select “Duplicate Release Build Configuration” and name it “Staging”
3. In your terminal, navigate to the `ios` directory and run `pod install`

#### **Step 2** - Create Staging Scheme

1. In the main menu, select Product > Scheme > Manage Schemes
2. Click on the main project scheme (same as project name) and click on the icon at the bottom and select “Duplicate”
3. The new scheme name will be highlight, and call it “_project_name_.staging”
4. The edit scheme screen should remain open
5. Change the “Build Configuration” on the “Run”, “Profile” and “Archive” to the new “Staging” configuration
6. Back on the “Manage Schemes” window, make sure “Shared” is checked for the _project_name_.staging scheme

#### **Step 3** - Create Develop Scheme

1. In the main menu, select Product > Scheme > Manage Schemes
2. Click on the main project scheme (same as project name) and click on the icon at the bottom and select “Duplicate”
3. The new scheme name will be highlight, and call it “_project_name_.develop
4. The edit scheme screen should remain open
5. Change the “Build Configuration” on the “Run”, “Profile” and “Archive” to the “Debug" configuration
6. Back on the “Manage Schemes” window, make sure “Shared” is checked for the _project_name_.develop scheme

#### **Step 4** - Configure Environment Variables

- Main Scheme
  - In the main menu, select Product > Scheme > Manage Schemes.
  - Click on the main project scheme and click the Edit button.
  - Expand the Build menu and click on Pre-actions.
  - Click the + sign and select New Run Script Action.
  - In the field, copy and paste `echo ".env.production" > /tmp/envfile`
- Staging Scheme
  - In the main menu, select Product > Scheme > Manage Schemes.
  - Click on the \*.staging project scheme and click the Edit button.
  - Expand the Build menu and click on Pre-actions.
  - Click the + sign and select New Run Script Action.
  - In the field, copy and paste `echo ".env.staging” > /tmp/envfile`
- Develop Scheme
  - In the main menu, select Product > Scheme > Manage Schemes.
  - Click on the \*.develop project scheme and click the Edit button.
  - Expand the Build menu and click on Pre-actions.
  - Click the + sign and select New Run Script Action.
  - In the field, copy and paste `echo ".env” > /tmp/envfile`

### **Android**

#### **Step 1** - Setup Keystore

1. Generate a new keystore by following steps [here](https://developer.android.com/studio/publish/app-signing#generate-key)
2. Place keystore file in the `android/app` directory.
3. Create a `keystore.properties` in the root of your `android` directory.

```properties
keyAlias=
keyPassword=
storeFile=keystore_name.keystore
storePassword=
```

#### **Step 2** - Gradle Updates

Open `android/app/build.gradle` and make the following updates.

```gradle
// Under the `apply` dependencies on top of file.

project.ext.envConfigFiles = [
    debug: ".env",
    staging: ".env.staging",
    release: ".env.production"
]
apply from: project(':react-native-config').projectDir.getPath() + "/dotenv.gradle"
```

```gradle
def keystorePropertiesFile= rootProject.file("keystore.properties")
def keystoreProperties = new Properties()
keystoreProperties.load(new FileInputStream(keystorePropertiesFile))

android {
  ...
}
```

```gradle
defaultConfig {
    configurations.all {
        resolutionStrategy { force 'androidx.core:core:1.6.0' }
    }
    versionCode 1
    versionName "0.0.1"
    minSdkVersion rootProject.ext.minSdkVersion
    targetSdkVersion rootProject.ext.targetSdkVersion
    // Replace with your own package name.
    resValue "string", "build_config_package", "com.your.package"
}
```

```gradle
signingConfigs {
    debug {
        ...
    }
    release {
        storeFile file(keystoreProperties['storeFile'])
        storePassword keystoreProperties['storePassword']
        keyAlias keystoreProperties['keyAlias']
        keyPassword keystoreProperties['keyPassword']
    }

}
```

```gradle
buildTypes {
    debug {
        signingConfig signingConfigs.debug
        applicationIdSuffix ".develop"
        matchingFallbacks = ['debug', 'release']
    }
    staging {
        signingConfig signingConfigs.release
        minifyEnabled enableProguardInReleaseBuilds
        proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"
        applicationIdSuffix ".staging"
        matchingFallbacks = ['release']
    }
    release {
        signingConfig signingConfigs.release
        minifyEnabled enableProguardInReleaseBuilds
        proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"
    }
}
```

#### **Step 3** - Update Build Names

1. Run `mkdir -p android/app/src/staging/res/values`
2. Run `touch android/app/src/staging/res/values/strings.xml`
3. Add this to the newly created `strings.xml` file:

```xml
<resources>
    <string name="app_name">App Name (S)</string>
</resources>
```

3. Run `mkdir -p android/app/src/debug/res/values`
4. Run `touch android/app/src/debug/res/values/strings.xml`
5. Add this to the newly created `strings.xml` file:

```xml
<resources>
    <string name="app_name">App Name (D)</string>
</resources>
```

### **Fastlane**

1. Run `bundle add fastlane dotenv pry`
2. Run `mkdir fastlane && touch fastlane/Fastfile`
3. Add `import_from_git(url: "git@github.com:teamairship/react-native-fastlane.git", path: "Fastfile”)` to the top of your Fastfile.
4. Run `bundle exec fastlane liftoff`
5. Run `bundle exec fastlane install_plugins`

## Environment Variables

```properties
FASTLANE_CI=
FASTLANE_PRODUCT_NAME=

FIREBASE_TOKEN=
FIREBASE_GROUPS=
FIREBASE_IOS_APP_ID=
FIREBASE_ANDROID_APP_ID=

FASTLANE_SLACK_CHANNEL=
FASTLANE_SLACK_WEBHOOK_URL=

FASTLANE_ANDROID_PACKAGE_NAME=
FASTLANE_ANDROID_JSON_KEY_FILE=

FASTLANE_APPLE_ID=
FASTLANE_APPLE_TEAM_ID=
FASTLANE_APPLE_ITC_TEAM_ID=
FASTLANE_APPLE_PROJECT_NAME=
FASTLANE_APPLE_APP_ID=
FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD=
```

## Resources

[Fastlane Docs](https://docs.fastlane.tools/)

- [iOS Setup](https://docs.fastlane.tools/getting-started/ios/setup/)
- [Android Setup](https://docs.fastlane.tools/getting-started/android/setup/)

[Firebase](https://firebase.google.com/)

- [App Distribution](https://rnfirebase.io/app-distribution/usage)

[Slack](https://slack.com/)

- [Incoming Webhooks](https://slack.com/help/articles/115005265063-Incoming-webhooks-for-Slack)

[Apple Developer Portal](https://developer.apple.com/)

- [Fastlane Authentication - Method 3](https://docs.fastlane.tools/getting-started/ios/authentication/)
- [App Specific Password](https://support.apple.com/en-us/HT204397)
