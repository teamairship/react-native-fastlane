# React Native Fastlane

The best approach to efficient React Native builds and distributions.

## Prerequisites

```bash
$ npx react-native init MyApp --template react-native-template-typescript
$ cd MyApp
$ touch .env .env.staging .env.production
```

## Dependencies

1. [Ruby 3.1.1](https://www.ruby-lang.org/)
2. [Bundler](https://bundler.io/)
3. [Homebrew](https://brew.sh/)
4. [ImageMagick](https://imagemagick.org/index.php) - `brew install imagemagick`
5. [react-native-config](https://github.com/luggit/react-native-config)
6. [Firebase App Distribution](https://rnfirebase.io/app-distribution/usage)

## New React Native App Setup

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
3. Create a `keystore.properties` in the root of your `android` directory. Add this to your `.gitignore` file.

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
  flavorDimensions "version"
  ...
}
```

```gradle
defaultConfig {
    ... other settings

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
        matchingFallbacks = ['debug', 'release']
    }
    release {
        ... other settings

        signingConfig signingConfigs.release
        minifyEnabled enableProguardInReleaseBuilds
        proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"
    }
}

// Add this whole block directly under buildTypes:
productFlavors {
        develop {
            applicationIdSuffix ".develop"
        }
        staging {
            applicationIdSuffix ".staging"
        }
        production {}
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

1. Run `mkdir -p android/app/src/debug/res/values`
2. Run `touch android/app/src/debug/res/values/strings.xml`
3. Add this to the newly created `strings.xml` file:

```xml
<resources>
    <string name="app_name">App Name (D)</string>
</resources>
```

### Obtaining Firebase JSON Key File

- Open the Google Play Console -> https://play.google.com/console/u/1/developers/
- Click **Setup** -> **API Access**
- Click button for “**Choose a project to link”**
- Click agree on “Terms of Service”
- Select  **Create a new Google Cloud project**  then click **Save** 
- In the **Google Cloud Project** section  click button for **View in Google Cloud Platform**
- In top left menu bar click **IAM & Admin**-> **Service Accounts**
- Click button “**Create Service Account”**
- For service account name put **prometheus\_mobile**
- This will populate the service account id
- For Service account description you can put **Fastlane Integration**
- Click button “**Create“** and then “**Continue”**
- You will see a drop down under role and your going to select **Service Accounts** -> **Service Account User**
- Click -> **Continue** -> **Done**
- On new screen you will see the service you just created
- Click the **three dots** under “**Actions”**
- Select -> Manage Keys
- Click -> **Add Key** -> **Create New Key**  and select **JSON** 
- This will create a JSON file that you can then send to **your_airship_email** 
- Once this is done go back in the Google Play Console and go to **Setup** -> **API Access** 
- You will now see the service you created below the **Credentials** section
- Click -> Manage Play Console permissions
- Under **Account Permissions**  select the following
	- View app information and download bulk reports
	- Create, edit and delete draft apps
	- Release apps to testing tracks
	- Manage testing tracks and edit tester lists
	- Manage store presence
- Then click -> Save Changes
- After this you are all set

### **Fastlane**

1. Run `bundle init`
2. Run `bundle add fastlane dotenv pry`
3. Run `mkdir fastlane && touch fastlane/Fastfile`
4. Add `import_from_git(url: "git@github.com:teamairship/react-native-fastlane.git", path: "Fastfile”)` to the top of your Fastfile.
5. Run `bundle exec fastlane liftoff`
6. Run `bundle exec fastlane install_plugins`

## Environment Variables

```properties
# The name of your product with proper capitalization. This is used to build the
# text under the icon of your application.
FASTLANE_PRODUCT_NAME=

# https://firebase.google.com/docs/app-distribution/ios/distribute-fastlane
FIREBASE_TOKEN=

# Comma delimited list of group names you've setup in App Distribution.
# https://firebase.google.com/docs/app-distribution/manage-testers#add-remove-testers-group
FIREBASE_GROUPS=

# These are the generated IDs from Firebase. Once you have setup your application
# in Firebase, follow the guide to retrieve the IDs needed. I use the staging app
# for these IDs.
# iOS: https://firebase.google.com/docs/app-distribution/ios/distribute-fastlane
# Android: https://firebase.google.com/docs/app-distribution/android/distribute-fastlane
FIREBASE_IOS_APP_ID=
FIREBASE_AND_APP_ID=

# The name of the Slack channel you want your build notifications to go to.
# Do not add the # prefix.
FASTLANE_SLACK_CHANNEL=

# The webhook URL of the Slack channel you want your build notifications to go to.
# https://you.slack.com/apps/manage/custom-integrations
FASTLANE_SLACK_WEBHOOK_URL=

# The package name of the app you want to distribute.
# Eg. com.equip
FASTLANE_ANDROID_PACKAGE_NAME=
# The location of the JSON key file for the Android app you want to distribute.
# Eg. /android/12346589000.json
FASTLANE_ANDROID_JSON_KEY_FILE=

# Email address that has access to the Apple Developer Portal
# for the app you want to distribute.
FASTLANE_APPLE_USERNAME=

# Base bundle identifier of your app.
# Eg. com.equip.AppName
FASTLANE_APPLE_BUNDLE_IDENTIFIER=

# The team ID of the Apple Developer account you want to distribute to.
# You can find this on the Membership screen once you've logged in. It's
# listed as Team ID. You can also find it in the URL.
FASTLANE_APPLE_DEV_TEAM_ID=

# The team ID of the App Store Connect team you want to distribute to.
FASTLANE_APPLE_ITC_TEAM_ID=

# To skip sign in on every run, generate an app specific password for the
# FASTLANE_APPLE_USERNAME email above and set it here.
# https://support.apple.com/en-us/HT204397
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
