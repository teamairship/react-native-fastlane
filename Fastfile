fastlane_require "net/http"
fastlane_require "dotenv"

BUILD_SCHEMES = %i[develop staging]
AND_GRAD_PATH = "android/app/build.gradle"
IOS_DIRECTORY = "ios/#{ENV["FASTLANE_APPLE_PROJECT_NAME"]}"
IOS_PROJ_PATH = "#{IOS_DIRECTORY}.xcodeproj"
IOS_WORK_PATH = "#{IOS_DIRECTORY}.xcworkspace"

before_all do |lane, options|
  Dotenv.overload("../.env")

  ensure_env_vars(
    env_vars: [
      "FASTLANE_PRODUCT_NAME",
      "FASTLANE_ANDROID_PACKAGE_NAME",
      "FASTLANE_ANDROID_JSON_KEY_FILE",
      "FASTLANE_APPLE_ID",
      "FASTLANE_APPLE_TEAM_ID",
      "FASTLANE_APPLE_ITC_TEAM_ID",
      "FASTLANE_APPLE_PROJECT_NAME",
      "FASTLANE_APPLE_APP_ID",
      "FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD"
    ]
  )

  lane_context["IOS_SCHEME"]  = add_suffix_to(ENV["FASTLANE_APPLE_PROJECT_NAME"], lane)
  lane_context["IOS_APP_ID"]  = add_suffix_to(fetch_config(:app_identifier), lane)
  lane_context["AND_PACKAGE"] = add_suffix_to(fetch_config(:package_name), lane)
end

lane :liftoff do |options|
  populate_supporting_files
  generate_metadata
  generate_apple_identifiers if options[:identifiers] || options[:all]
  generate_apple_profiles if options[:profiles] || options[:all]
  create_app_in_portal if options[:portal] || options[:all]
  keystore if options[:keystore] || options[:all]
end

lane :keystore do |options|
  ensure_env_vars(
    env_vars: [
      "FASTLANE_ANDROID_KEYSTORE_KEYSTORE_NAME",
      "FASTLANE_ANDROID_KEYSTORE_ALIAS_NAME",
    ]
  )

  sh("sudo keytool -genkey -v -keystore #{ENV["FASTLANE_ANDROID_KEYSTORE_KEYSTORE_NAME"]} -alias #{ENV["FASTLANE_ANDROID_KEYSTORE_ALIAS_NAME"]} -keyalg RSA -keysize 2048 -validity 10000")
end

lane :update_version do |options|
  version = fetch_metadata("version.txt")

  increment_version_number(
    xcodeproj: IOS_PROJ_PATH,
    version_number: version
  )

  increment_version_name(
    gradle_file_path: AND_GRAD_PATH,
    version_name: version
  )
end

platform :ios do
  lane :build_application do |options|
    build_app(
      silent: true,
      include_symbols: true,
      include_bitcode: true,
      analyze_build_time: true,
      scheme: options[:scheme],
      workspace: IOS_WORK_PATH,
      output_directory: "ios/build",
      xcargs: "-allowProvisioningUpdates",
      configuration: options[:configuration]
    )
  end

  lane :prefix_name do |options|
    update_info_plist(
      xcodeproj: IOS_PROJ_PATH,
      app_identifier: lane_context["IOS_APP_ID"],
      plist_path: "/#{ENV["FASTLANE_APPLE_PROJECT_NAME"]}/Info.plist",
      display_name: "#{ENV["FASTLANE_PRODUCT_NAME"]} (#{options[:prefix]})"
    )
  end

  lane :fetch_profiles do |options|
    match(type: options[:type], readonly: is_ci, app_identifier: lane_context["IOS_APP_ID"])
  end

  desc "Development..."
  lane :develop do |options|
    increment_build(:ios)
    prefix_name(prefix: "D")
    prepare_icons(platform: :ios, lane: :develop)
    fetch_profiles(type: "adhoc")
    sh "cd .. && npx react-native run-ios --scheme #{options[:scheme] || lane_context["IOS_SCHEME"]} #{options[:device] ? "device" : ""}"
    discard_icons(platform: :ios)
  end

  desc "Submit a new staging build to internal testers..."
  lane :staging do
    ensure_env_vars(
      env_vars: [
        "FIREBASE_TOKEN",
        "FIREBASE_GROUPS",
        "FIREBASE_IOS_APP_ID"
      ]
    )

    increment_build(:ios)
    prefix_name(prefix: "S")
    prepare_icons(platform: :ios, lane: :staging)
    fetch_profiles(type: "adhoc")
    build_application(scheme: lane_context["IOS_SCHEME"], configuration: "Staging")

    # Upload to Firebase
    firebase_app_distribution(
      app: ENV["FIREBASE_IOS_APP_ID"],
      groups: ENV["FIREBASE_GROUPS"]
    )

    discard_icons(platform: :ios)
    notify_the_team(:ios, :staging)
  end

  desc "Once staging is approved, submit a production build to beta testers..."
  lane :beta do
    increment_build(:ios)
    prepare_icons(platform: :ios)
    fetch_profiles(type: "appstore")
    build_application(scheme: lane_context["IOS_SCHEME"], configuration: "Release")

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

    notify_the_team(:ios, :beta)
  end

  desc "Once beta is approved, promote beta build to production..."
  lane :release do
    upload_to_app_store(
      submission_information: "{\"export_compliance_uses_encryption\": false, \"add_id_info_uses_idfa\": false }"
    )

    notify_the_team(:ios, :release)
  end
end

platform :android do
  lane :create_keystore do |options|
    keystore
  end

  lane :build_application do |options|
    gradle(
      project_dir: "android",
      flavor: options[:flavor],
      task: "clean #{options[:task] || "bundle"}",
      build_type: options[:build_type] || "release"
    )
  end

  desc "Development..."
  lane :develop do
    increment_build(:android)
    prepare_icons(platform: :android, lane: :develop)
    sh "cd .. && adb reverse tcp:8081 tcp:8081 && npx react-native run-android --variant=#{options[:variant] || "developDebug"}"
    discard_icons(platform: :android)
  end

  desc "Submit a new staging build to internal testers..."
  lane :staging do
    ensure_env_vars(
      env_vars: [
        "FIREBASE_TOKEN",
        "FIREBASE_GROUPS",
        "FIREBASE_ANDROID_APP_ID"
      ]
    )

    increment_build(:android)
    prepare_icons(platform: :android, lane: :staging)
    build_application(flavor: "staging", build_type: "release", task: "assemble")

    firebase_app_distribution(
      android_artifact_type: "APK",
      groups: ENV["FIREBASE_GROUPS"],
      app: ENV["FIREBASE_ANDROID_APP_ID"],
      android_artifact_path: lane_context[SharedValues::GRADLE_APK_OUTPUT_PATH]
    )

    discard_icons(platform: :android)
    notify_the_team(:android, :staging)
  end

  desc "Submit a new production build to internal users..."
  lane :internal do
    increment_build(:android)
    prepare_icons(platform: :android)
    build_application(variant: "production", task: "assemble")

    firebase_app_distribution(
      android_artifact_type: "APK",
      groups: ENV["FIREBASE_GROUPS"],
      app: ENV["FIREBASE_ANDROID_APP_ID"],
      android_artifact_path: lane_context[SharedValues::GRADLE_APK_OUTPUT_PATH]
    )

    discard_icons(platform: :android)
    notify_the_team(:android, :internal)
  end

  desc "Once staging is approved, submit a production build to beta testers..."
  lane :beta do
    increment_build(:android)
    prepare_icons(platform: :android)
    build_application(flavor: "production")
    upload_to_play_store(track: "beta")
    notify_the_team(:android, :beta)
  end

  desc "Once beta is approved, promote beta build to production..."
  lane :release do
    upload_to_play_store(track: "beta", track_promote_to: "production")
    notify_the_team(:android, :production)
  end
end

lane :prepare_icons do |options|
  platform = options[:platform]

  if platform == :ios
    appicon(
      appicon_image_file: "app_icon.png",
      appicon_path: "#{IOS_DIRECTORY}/Images.xcassets",
      appicon_devices: [:ipad, :iphone, :ios_marketing]
    )
  elsif platform == :android
    android_appicon(
      generate_rounded: true,
      appicon_icon_types: [:launcher],
      appicon_image_file: "app_icon.png",
      appicon_path: "android/app/src/main/res/mipmap"
    )
  end

  if options[:lane]
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

def generate_metadata
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

def populate_supporting_files
  %w[Appfile Matchfile Pluginfile Precheckfile].each do |file|
    uri = "https://raw.githubusercontent.com/teamairship/react-native-fastlane/main/#{file}"
    uri = URI(uri)
    contents = Net::HTTP.get(uri)

    File.open(file, "w") { |f| f.write(contents) }
  end
end

def generate_apple_identifiers
  id = Spaceship::ConnectAPI::BundleId.create(
    name: ENV["FASTLANE_APPLE_PROJECT_NAME"],
    identifier: ENV["FASTLANE_APPLE_APP_ID"],
  )

  id.create_capability(Spaceship::ConnectAPI::BundleIdCapability::Type::PUSH_NOTIFICATIONS)

  BUILD_SCHEMES.each do |schema|
    id = Spaceship::ConnectAPI::BundleId.create(
      name: "#{ENV["FASTLANE_APPLE_PROJECT_NAME"].capitalize} #{schema.capitalize}",
      identifier: "#{ENV["FASTLANE_APPLE_APP_ID"]}.#{schema}",
    )

    id.create_capability(Spaceship::ConnectAPI::BundleIdCapability::Type::PUSH_NOTIFICATIONS)
  end
end

def generate_apple_profiles
  match(app_identifier: [
    ENV["FASTLANE_APPLE_APP_ID"],
    **BUILD_SCHEMES.map { |schema| "#{ENV["FASTLANE_APPLE_APP_ID"]}.#{schema}" }
  ])
end

def create_app_in_portal
  sku = prompt(
    text: "What is the SKU for this app?"
  )

  produce(
    app_name: ENV["FASTLANE_PRODUCT_NAME"],
    language: "English",
    app_version: fetch_metadata("version.txt") || "0.0.1",
    sku: sku
  )
end

def fetch_version(platform)
  if platform == :ios
    {
      name: get_version_number(xcodeproj: IOS_PROJ_PATH),
      code: get_build_number(xcodeproj: IOS_PROJ_PATH)
    }
  elsif platform == :android
    {
      name: android_get_version_name(gradle_file: AND_GRAD_PATH),
      code: android_get_version_code(gradle_file: AND_GRAD_PATH)
    }
  end
end

def increment_build(platform)
  if platform == :ios
    previous_build_number = latest_testflight_build_number
    current_build_number = previous_build_number + 1

    increment_build_number(
      xcodeproj: IOS_PROJ_PATH,
      build_number: current_build_number
    )
  elsif platform == :android
    latest_release = firebase_app_distribution_get_latest_release(
      app: ENV["FIREBASE_ANDROID_APP_ID"]
    )

    previous_build_number = latest_release[:buildVersion]&.to_i
    current_build_number = previous_build_number + 1

    increment_version_code(
      gradle_file_path: AND_GRAD_PATH,
      version_code: current_build_number
    )
  end
end

def notify_the_team(platform, lane)
  ensure_env_vars(
    env_vars: [
      "FASTLANE_SLACK_CHANNEL",
      "FASTLANE_SLACK_WEBHOOK_URL"
    ]
  )

  version = fetch_version(platform)
  message = platform == :ios \
    ? "Check TestFlight for new version..." \
    : "https://play.google.com/store/apps/details?id=#{lane_context["PACKAGE_NAME"]}"

  instructions = %i[staging internal].include?(lane) ? "Lookout for Firebase email..." : message

  slack(
    message: "#{instructions}\n\n",
    channel: "##{ENV["FASTLANE_SLACK_CHANNEL"]}",
    slack_url: ENV["FASTLANE_SLACK_WEBHOOK_URL"],
    use_webhook_configured_username_and_icon: false,
    pretext: "#{ENV["FASTLANE_PRODUCT_NAME"]} #{platform} Build: #{lane} #{version[:name]} (#{version[:code]})"
  )
end

def add_suffix_to(value, lane)
  value += ".#{lane}" if BUILD_SCHEMES.include?(lane)
  value
end

def fetch_config(value)
  CredentialsManager::AppfileConfig.try_fetch_value(value) || ""
end

def fetch_metadata(key)
  File.read("metadata/#{key}")
end

