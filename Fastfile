fastlane_require "net/http"
fastlane_require "dotenv"

BUILD_SCHEMES  = %i[develop staging]
AND_GRAD_PATH  = "android/app/build.gradle"
AND_PACKAGE_ID = CredentialsManager::AppfileConfig.try_fetch_value(:package_name) || ""
IOS_IDENTIFIER = CredentialsManager::AppfileConfig.try_fetch_value(:app_identifier) || ""

Dir.glob("../ios/*.xcworkspace") do |f|
  IOS_PROJ_NAME = File.basename(f, File.extname(f))
  IOS_DIRECTORY = "ios/#{IOS_PROJ_NAME}"
  IOS_PROJ_PATH = "#{IOS_DIRECTORY}.xcodeproj"
  IOS_WORK_PATH = "#{IOS_DIRECTORY}.xcworkspace"
end

before_all do |lane, options|
  Dotenv.overload("../.env")

  ensure_env_vars(
    env_vars: [
      "FASTLANE_PRODUCT_NAME",
      "FASTLANE_APPLE_USERNAME",
      "FASTLANE_ANDROID_PACKAGE_NAME",
      "FASTLANE_APPLE_BUNDLE_IDENTIFIER",
      "FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD"
    ]
  )

  lane_context["IOS_SCHEME"]  = add_suffix_to(IOS_PROJ_NAME, lane)
  lane_context["IOS_APP_ID"]  = add_suffix_to(IOS_IDENTIFIER, lane)
  lane_context["AND_PACKAGE"] = add_suffix_to(AND_PACKAGE_ID, lane)
end

lane :liftoff do |options|
  if ENV["FASTLANE_APPLE_DEV_TEAM_ID"].empty? && ENV["FASTLANE_APPLE_ITC_TEAM_ID"].empty?
    UI.message("You haven't set your Apple team IDs, fetching teams now... ⏳")

    client = fetch_teams(true, true)
    dev_team_id = client.portal_client.instance_variable_get(:@current_team_id)
    itc_team_id = client.tunes_client.instance_variable_get(:@current_team_id)

    UI.important("
      If you'd like to skip this prompt in the future\n
      Set the FASTLANE_APPLE_DEV_TEAM_ID environment variable to #{dev_team_id}
      Set the FASTLANE_APPLE_ITC_TEAM_ID environment variable to #{itc_team_id}
    ")
  elsif ENV["FASTLANE_APPLE_DEV_TEAM_ID"].empty?
    UI.message("You haven't set your Apple Developer Team ID, fetching teams now... ⏳")

    client = fetch_teams(true, false)
    the_id = client.portal_client.instance_variable_get(:@current_team_id)

    UI.important("
      If you'd like to skip this prompt in the future\n
      Set the FASTLANE_APPLE_DEV_TEAM_ID environment variable to #{the_id}
    ")
  elsif ENV["FASTLANE_APPLE_ITC_TEAM_ID"].empty?
    UI.message("You haven't set your App Store Connect Team ID, fetching teams now... ⏳")

    client = fetch_teams(false, true)
    the_id = client.tunes_client.instance_variable_get(:@current_team_id)

    UI.important("
      If you'd like to skip this prompt in the future\n
      Set the FASTLANE_APPLE_ITC_TEAM_ID environment variable to #{the_id}
    ")
  end

  UI.message("🚀 Let's GOOOOOOOOOOOO! 🚀")
  populate_supporting_files
  generate_metadata

  if UI.confirm("Do you need me to generate Apple bundle identifiers? (yN)")
    generate_apple_identifiers
  end

  if UI.confirm("Do you need me to generate Apple Provisioning Profiles? (yN)")
    generate_apple_profiles
  end

  if UI.confirm("Do you need me to create an app in App Store Connect? (yN)")
    create_app_in_portal
  end

  if UI.confirm("Do you need me to generate an Android keystore for production? (yN)")
    generate_keystore
  end

  UI.success("🎉 All done! 🎉")
end

lane :generate_push_certificate do
  get_push_certificate(
    force: true,
    app_identifier: IOS_IDENTIFIER,
    output_path: 'ios/build'
  )
end

lane :generate_keystore do |options|
  ensure_env_vars(
    env_vars: [
      "FASTLANE_ANDROID_KEYSTORE_ALIAS_NAME",
      "FASTLANE_ANDROID_KEYSTORE_KEYSTORE_NAME"
    ]
  )

  sh("sudo keytool -genkey -v -keystore #{ENV["FASTLANE_ANDROID_KEYSTORE_KEYSTORE_NAME"]} -alias #{ENV["FASTLANE_ANDROID_KEYSTORE_ALIAS_NAME"]} -keyalg RSA -keysize 2048 -validity 10000")
end

lane :update_version do |options|
  version = fetch_metadata("version.txt")
  increment_version_number(xcodeproj: IOS_PROJ_PATH, version_number: version)
  increment_version_name(gradle_file_path: AND_GRAD_PATH, version_name: version)
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

  lane :suffix_name do |options|
    update_info_plist(
      xcodeproj: IOS_PROJ_PATH,
      app_identifier: options[:suffix] ? lane_context["IOS_APP_ID"] : IOS_IDENTIFIER,
      plist_path: "/#{IOS_PROJ_NAME}/Info.plist",
      display_name: "#{ENV["FASTLANE_PRODUCT_NAME"]}#{options[:suffix] ? " (#{options[:suffix]})" : ''}"
    )
  end

  lane :fetch_profiles do |options|
    match(
      type: options[:type],
      readonly: is_ci,
      generate_apple_certs: options[:type] == "appstore",
      app_identifier: lane_context["IOS_APP_ID"]
    )
  end

  desc "Local development using iOS Simulator..."
  lane :develop do |options|
    increment_build(:ios)
    suffix_name(suffix: "D")
    prepare_icons(platform: :ios, lane: :develop)
    fetch_profiles(type: "development")
    sh "cd .. && npx react-native run-ios --scheme #{options[:scheme] || lane_context["IOS_SCHEME"]} #{options[:device] ? "--device" : ""}"
    suffix_name(suffix: nil)
    discard_icons(platform: :ios)
  end

  desc "Submit a new staging build to Firebase testers..."
  lane :staging do |options|
    ensure_env_vars(
      env_vars: [
        "FIREBASE_TOKEN",
        "FIREBASE_GROUPS",
        "FIREBASE_IOS_APP_ID"
      ]
    )

    increment_build(:ios)
    suffix_name(suffix: "S")
    prepare_icons(platform: :ios, lane: :staging)
    fetch_profiles(type: "adhoc")

    if options[:local]
      sh "cd .. && npx react-native run-ios --scheme #{options[:scheme] || lane_context["IOS_SCHEME"]} #{options[:device] ? "--device" : ""}"
    else
      build_application(scheme: lane_context["IOS_SCHEME"], configuration: "Staging")

      firebase_app_distribution(
        app: ENV["FIREBASE_IOS_APP_ID"],
        groups: ENV["FIREBASE_GROUPS"]
      )

      notify_the_team(:ios, :staging)
    end

    suffix_name(suffix: nil)
    discard_icons(platform: :ios)
  end

  desc "Once staging is approved, submit a production build to TestFlight testers..."
  lane :beta do
    increment_build(:ios)
    suffix_name(suffix: nil)
    prepare_icons(platform: :ios)
    fetch_profiles(type: "appstore")
    build_application(scheme: lane_context["IOS_SCHEME"], configuration: "Release")

    upload_to_testflight(
      app_version: fetch_metadata("version.txt"),
      skip_waiting_for_build_processing: true,
      reject_build_waiting_for_review: true,
      submit_beta_review: true,
      beta_app_feedback_email: fetch_metadata("review_information/email_address.txt"),
      beta_app_description: fetch_metadata("default/release_notes.txt"),
      demo_account_required: true,
      changelog: fetch_metadata("default/release_notes.txt"),
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
        "default": {
          feedback_email: fetch_metadata("review_information/email_address.txt"),
          marketing_url: fetch_metadata("default/marketing_url.txt"),
          privacy_policy_url: fetch_metadata("default/privacy_url.txt"),
          description: fetch_metadata("default/description.txt")
        }
      }
    )

    notify_the_team(:ios, :beta)
  end

  desc "Once beta is approved, promote beta build to App Store..."
  lane :release do
    upload_to_app_store(
      submission_information: "{\"export_compliance_uses_encryption\": false, \"add_id_info_uses_idfa\": false }",
      app_version: fetch_metadata("version.txt"),
      skip_screenshots: true,
      include_in_app_purchases: false
    )

    notify_the_team(:ios, :release)
  end
end

platform :android do
  lane :build_application do |options|
    gradle(
      project_dir: "android",
      flavor: options[:flavor],
      task: "clean #{options[:task] || "bundle"}",
      build_type: options[:build_type] || "release"
    )
  end

  desc "Development..."
  lane :develop do |options|
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
        "FIREBASE_AND_APP_ID"
      ]
    )

    increment_build(:android)
    prepare_icons(platform: :android, lane: :staging)
    build_application(flavor: "staging", build_type: "release", task: "assemble")

    firebase_app_distribution(
      android_artifact_type: "APK",
      groups: ENV["FIREBASE_GROUPS"],
      app: ENV["FIREBASE_AND_APP_ID"],
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
      app: ENV["FIREBASE_AND_APP_ID"],
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
  platform = options[:platform].to_sym

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
  location = platform == :android \
    ? "../android/**/*/ic_launcher*.png"\
    : "../ios/**/*.appiconset/*.png"

  reset_git_repo(force: true, files: Dir.glob(location).map {|f| File.expand_path(f)})
end

def fetch_teams(portal, tunes)
  client = Spaceship::ConnectAPI.login(
    use_portal: portal,
    use_tunes: tunes,
    portal_team_id: nil,
    tunes_team_id: nil,
    team_name: nil,
    skip_select_team: false
  )
end

def populate_supporting_files
  UI.message("Fetching files from react-native-fastlane... 🚀")

  %w[Appfile Matchfile Pluginfile Precheckfile Deliverfile].each do |file|
    uri = "https://raw.githubusercontent.com/teamairship/react-native-fastlane/main/#{file}"
    uri = URI(uri)
    contents = Net::HTTP.get(uri)

    File.open(file, "w") { |f| f.write(contents) }
  end
end

def generate_metadata
  UI.message("Creating metadata files for easy app creation... ✍🏼")

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

def generate_apple_identifiers
  id = Spaceship::ConnectAPI::BundleId.create(
    name: IOS_PROJ_NAME,
    identifier: IOS_IDENTIFIER,
  )

  UI.success("Created main bundle identifier... #{IOS_IDENTIFIER} ✅")

  id.create_capability(Spaceship::ConnectAPI::BundleIdCapability::Type::PUSH_NOTIFICATIONS)

  UI.success("Added push notifications as a capability... ✅")

  BUILD_SCHEMES.each do |schema|
    id = Spaceship::ConnectAPI::BundleId.create(
      identifier: "#{IOS_IDENTIFIER}.#{schema}",
      name: "#{IOS_PROJ_NAME.capitalize} #{schema.capitalize}"
    )

    UI.success("Created #{scheme} bundle identifier... #{IOS_IDENTIFIER}.#{schema} ✅")

    id.create_capability(Spaceship::ConnectAPI::BundleIdCapability::Type::PUSH_NOTIFICATIONS)
  end

  generate_push_certificate

  UI.success("Apple identifiers created successfully... 🏁")
end

def generate_apple_profiles
  match(
    app_identifier: [IOS_IDENTIFIER] + BUILD_SCHEMES.map {|schema| "#{IOS_IDENTIFIER}.#{schema}"}
  )

  UI.success("Apple provisioning profiles created successfully... 🏁")
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
    prev_build_number = latest_testflight_build_number
    curr_build_number = prev_build_number + 1

    increment_build_number(
      xcodeproj: IOS_PROJ_PATH,
      build_number: curr_build_number
    )
  elsif platform == :android
    latest_release = firebase_app_distribution_get_latest_release(
      app: ENV["FIREBASE_AND_APP_ID"]
    )

    prev_build_number = latest_release ? latest_release[:buildVersion]&.to_i : 0
    curr_build_number = prev_build_number + 1

    increment_version_code(
      gradle_file_path: AND_GRAD_PATH,
      version_code: curr_build_number
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

def fetch_metadata(key)
  File.read("metadata/#{key}")
end

