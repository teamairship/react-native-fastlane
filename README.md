# React Native Fastlane

## Installation

[Fastlane Docs](https://docs.fastlane.tools/)

## Setup

### Fastlane

[iOS Setup](https://docs.fastlane.tools/getting-started/ios/setup/)

[Android Setup](https://docs.fastlane.tools/getting-started/android/setup/)

### Firebase

[App Distribution](https://rnfirebase.io/app-distribution/usage)

### Slack

[Incoming Webhooks](https://slack.com/help/articles/115005265063-Incoming-webhooks-for-Slack)

### Apple

[Fastlane Authentication - Method 3](https://docs.fastlane.tools/getting-started/ios/authentication/)

[App Specific Password](https://support.apple.com/en-us/HT204397)

## Required Environment Variables

```
FASTLANE_CI=
FASTLANE_PRODUCT_NAME=

# Use Staging app's information for all FIREBASE_* environment variables
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
