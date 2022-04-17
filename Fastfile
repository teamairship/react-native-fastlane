fastlane_require 'dotenv'
fastlane_require 'net/http'

before_all do |lane, options|
  Dotenv.overload("../.env")
end

lane :liftoff do
  %w[Appfile Matchfile Pluginfile].each do |file|
    uri = "https://raw.githubusercontent.com/teamairship/react-native-fastlane/main/#{file}"
    uri = URI(uri)
    contents = Net::HTTP.get(uri)

    File.open(file, "w") { |f| f.write(contents) }
  end
end

lane :update_version do |options|
  version = fetch_metadata("version.txt")

  increment_version_number(
    xcodeproj: fetch_xcodeproj,
    version_number: version
  )

  increment_version_name(
    gradle_file_path: fetch_gradle,
    version_name: version
  )
end

platform :ios do
  before_all do
    lane_context["IOS_TARGET"] = ENV["FASTLANE_APPLE_PROJECT_NAME"]
    lane_context["IOS_TARGET"] += ".staging" if lane == "staging"
    lane_context["IOS_SCHEME"] = ENV["FASTLANE_APPLE_PROJECT_NAME"]
    lane_context["IOS_SCHEME"] += ".staging" if lane == "staging"
    lane_context["IOS_APP_ID"] = CredentialsManager::AppfileConfig.try_fetch_value(:app_identifier)
    lane_context["IOS_APP_ID"] += ".staging" if lane == "staging"
  end

  lane :metadata do
    Dir.mkdir("metadata") unless Dir.exist?("metadata")
    FileUtils.mkdir_p("metadata/default") unless Dir.exist?("metadata/default")
    FileUtils.mkdir_p("metadata/review_information") unless Dir.exist?("metadata/review_information")

    [
      "copyright.txt",
      "primary_category.txt",
      "secondary_category.txt",
      "primary_first_sub_category.txt",
      "primary_second_sub_category.txt",
      "secondary_first_sub_category.txt",
      "secondary_second_sub_category.txt",
    ].each { |file| FileUtils.touch("metadata/#{file}") }

    [
      "name.txt",
      "subtitle.txt",
      "privacy_url.txt",
      "apple_tv_privacy_policy.txt",
      "description.txt",
      "keywords.txt",
      "release_notes.txt",
      "support_url.txt",
      "marketing_url.txt",
      "promotional_text.txt"
    ].each { |file| FileUtils.touch("metadata/default/#{file}") }

    [
      "first_name.txt",
      "last_name.txt",
      "email_address.txt",
      "phone_number.txt",
      "demo_user.txt",
      "demo_password.txt",
      "notes.txt"
    ].each { |file| FileUtils.touch("metadata/review_information/#{file}") }
  end

  lane :build do |options|
    build_app(
      silent: true,
      include_symbols: true,
      include_bitcode: true,
      analyze_build_time: true,
      scheme: options[:scheme],
      output_directory: "ios/build",
      xcargs: "-allowProvisioningUpdates",
      workspace: fetch_xcworkspace
    )
  end

  desc "Step 1: Submit a new staging build to internal testers..."
  lane :staging do
    bump_build_number(:ios)

    # Add badge to app icon
    prepare_icons(platform: :ios, lane: :staging)

    # Import or create certificates
    match(
      type: "adhoc",
      readonly: ENV["FASTLANE_CI"] == "true",
      app_identifier: lane_context["IOS_APP_ID"]
    )

    # Build staging app
    build(scheme: lane_context["IOS_SCHEME"])

    # Upload to Firebase
    firebase_app_distribution(
      app: ENV["FIREBASE_IOS_APP_ID"],
      groups: ENV["FIREBASE_GROUPS"]
    )

    # Remove badge from app icon
    discard_icons(platform: :ios)

    # Notify team in Slack
    notify_the_team(:ios, :staging)
  end

  desc "Step 2: Once staging is approved, submit a production build to beta testers..."
  lane :beta do
    bump_build_number(:ios)

    # Generate app icons
    prepare_icons(platform: :ios)

    # Import or create certificates
    match(
      type: "appstore",
      readonly: ENV["FASTLANE_CI"] == "true",
      app_identifier: lane_context["IOS_APP_ID"]
    )

    # Build production ready app
    build(scheme: ENV["FASTLANE_APPLE_PROJECT_NAME"])

    # Upload to TestFlight
    upload_to_testflight(
      changelog: fetch_metadata("default/release_notes.txt"),
      skip_waiting_for_build_processing: true,
      reject_build_waiting_for_review: true,
      submit_beta_review: true,
      beta_app_review_info: {
        contact_email: fetch_metadata("review_information/email_address.txt"),
        contact_phone: fetch_metadata("review_information/phone_number.txt"),
        contact_first_name: fetch_metadata("review_information/first_name.txt"),
        contact_last_name: fetch_metadata("review_information/last_name.txt"),
        demo_account_name: fetch_metadata("review_information/demo_user.txt"),
        demo_account_password: fetch_metadata("review_information/demo_password.txt"),
        notes: fetch_metadata("review_information/notes.txt")
      },
      localized_app_info: {
        default: {
          feedback_email: fetch_metadata("review_information/email_address.txt"),
          marketing_url: fetch_metadata("default/marketing_url.txt"),
          privacy_policy_url: fetch_metadata("default/privacy_url.txt"),
          description: fetch_metadata("default/description.txt")
        }
      }
    )

    # Notify team in Slack
    notify_the_team(:ios, :beta)
  end

  desc "Step 3: Once beta is approved, promote beta build to production..."
  lane :release do
    upload_to_app_store(
      submission_information: "{\"export_compliance_uses_encryption\": false, \"add_id_info_uses_idfa\": false }"
    )

    notify_the_team(:ios, :release)
  end
end

platform :android do
  before_all do |lane, options|
    lane_context["PACKAGE_NAME"] = CredentialsManager::AppfileConfig.try_fetch_value(:package_name)
    lane_context["PACKAGE_NAME"] += ".staging" if lane == "staging"
  end

  lane :build do |options|
    gradle(
      task: "clean #{options[:task] || "bundle"}",
      build_type: "Release",
      project_dir: "android",
      flavor: options[:flavor]
    )
  end

  desc "Submit a new Staging Build to Firebase"
  lane :staging do
    bump_build_number(:android)

    # Add badge to app icon
    prepare_icons(platform: :android, lane: :staging)

    # Build staging app
    build(flavor: "Staging", task: "assemble")

    # Upload to Firebase
    firebase_app_distribution(
      app: ENV["FIREBASE_ANDROID_APP_ID"],
      groups: ENV["FIREBASE_GROUPS"],
      android_artifact_type: "APK",
      android_artifact_path: lane_context[SharedValues::GRADLE_APK_OUTPUT_PATH]
    )

    # Remove badge from app icon
    discard_icons(platform: :android)

    # Notify team in Slack
    notify_the_team(:android, :staging)
  end

  desc "Submit a new Beta Build to Google Play"
  lane :beta do
    bump_build_number(:android)

    # Generate app icons
    prepare_icons(platform: :android)

    # Build production ready app
    build(flavor: "Production")

    # Upload to Google Play
    upload_to_play_store(track: "beta")

    # Notify team in Slack
    notify_the_team(:android, :beta)
  end

  desc "Promote Beta Track to Production Release"
  lane :release do
    # Promote Beta Track to Production Release
    upload_to_play_store(track: "beta", track_promote_to: "production")

    # Notify team in Slack
    notify_the_team(:android, :production)
  end
end

lane :prepare_icons do |options|
  platform = options[:platform]

  if platform == :ios
    appicon(
      appicon_devices: [:ipad, :iphone, :ios_marketing],
      appicon_image_file: "app_icon.png",
      appicon_path: "#{fetch_ios_prefix}/Images.xcassets"
    )
  elsif platform == :android
    android_appicon(
      generate_rounded: true,
      appicon_icon_types: [:launcher],
      appicon_image_file: "app_icon.png",
      appicon_path: "android/app/src/main/res/mipmap"
    )
  end

  if options[:lane] == :staging
    version = fetch_version(platform)
    shield = "#{version[:name]}-#{version[:code]}-green"

    if platform == :ios
      add_badge(shield: shield, no_badge: true)
    elsif platform == :android
      add_badge(no_badge: true, shield: shield, alpha_channel: true, glob: "/android/**/*/ic_launcher*.png")
    end
  end
end

lane :discard_icons do |options|
  platform = options[:platform]
  location = platform == :ios \
    ? "../ios/**/*.appiconset/*.png" \
    : "../android/**/*/ic_launcher*.png"

  reset_git_repo(
    force: true,
    files: Dir.glob(location).map {|f| File.expand_path(f)}
  )
end

def fetch_version(platform)
  if platform == :ios
    {
      name: get_version_number(xcodeproj: fetch_xcodeproj, target: lane_context["IOS_TARGET"]),
      code: get_build_number(xcodeproj: fetch_xcodeproj)
    }
  elsif platform == :android
    {
      name: android_get_version_name(gradle_file: fetch_gradle),
      code: android_get_version_code(gradle_file: fetch_gradle)
    }
  end
end

def fetch_xcodeproj
  "#{fetch_ios_prefix}.xcodeproj"
end

def fetch_xcworkspace
  "#{fetch_ios_prefix}.xcworkspace"
end

def fetch_ios_prefix
  "ios/#{ENV["FASTLANE_APPLE_PROJECT_NAME"]}"
end

def fetch_gradle
  "android/app/build.gradle"
end

def fetch_metadata(key)
  File.read("metadata/#{key}")
end

def bump_build_number(platform)
  if platform == :ios
    previous_build_number = latest_testflight_build_number
    current_build_number = previous_build_number + 1

    increment_build_number(
      xcodeproj: fetch_xcodeproj,
      build_number: current_build_number
    )
  elsif platform == :android
    latest_release = firebase_app_distribution_get_latest_release(
      app: ENV["FIREBASE_ANDROID_APP_ID"]
    )

    previous_build_number = latest_release[:buildVersion]&.to_i
    current_build_number = previous_build_number + 1

    increment_version_code(
      gradle_file_path: fetch_gradle,
      version_code: current_build_number
    )
  end
end

def notify_the_team(platform, lane)
  version = fetch_version(platform)
  message = platform == :ios \
    ? "Check TestFlight for new version..." \
    : "https://play.google.com/store/apps/details?id=#{lane_context["PACKAGE_NAME"]}"

  instructions = lane == :staging ? "Lookout for Firebase email..." : message

  send_slack_message(
    "#{ENV["FASTLANE_PRODUCT_NAME"]} #{platform} Build: #{lane} #{version[:name]} (#{version[:code]})\n---\n#{instructions}\n---"
  )
end

def send_slack_message(message)
  slack(
    message: message,
    channel: "##{ENV["FASTLANE_SLACK_CHANNEL"]}",
    slack_url: ENV["FASTLANE_SLACK_WEBHOOK_URL"]
  )
end
