# Transform this into a `node_require` generic function:
def node_require(script)
  # Resolve script with node to allow for hoisting
  require Pod::Executable.execute_command('node', ['-p',
    "require.resolve(
      '#{script}',
      {paths: [process.argv[1]]},
    )", __dir__]).strip
end

# Use it to require both react-native's and this package's scripts:
node_require('react-native/scripts/react_native_pods.rb')
node_require('react-native-permissions/scripts/setup.rb')

platform :ios, '15.5' # min_ios_version_supported
prepare_react_native_project!

 # Setup permissions for react-native-permissions
  setup_permissions([
    'AppTrackingTransparency',
    'Notifications', 
    'PhotoLibrary',
    'Camera',
    'LocationWhenInUse',
    'LocationAlways',
    'Microphone',
    'SpeechRecognition'
  ])


linkage = ENV['USE_FRAMEWORKS']
if linkage != nil
  Pod::UI.puts "Configuring Pod with #{linkage}ally linked Frameworks".green
  use_frameworks! :linkage => linkage.to_sym
end

#To forcefully send data to test env in moengage , FCM etc, uncomment this line
#project 'app6thstreet',
#  'Debug' => :debug,
#  'Release' => :release

target 'app6thstreet' do
  config = use_native_modules!

   if ENV['RNV_SAMPLE_VIDEO_CACHING']
    $RNVideoUseVideoCaching = true
  end

  use_react_native!(
    :path => config[:reactNativePath],

    :hermes_enabled => true, # Set to false if not using Hermes
    :fabric_enabled => true,  # Enable New Architecture

    # An absolute path to your application root.
    :app_path => "#{Pod::Config.instance.installation_root}/.."
  )

  pod 'MoEngage-iOS-SDK'
  pod 'MoEngage-iOS-SDK/Inbox'
  pod 'MoEngagePluginInbox'
  # pod 'ReactNativeMoEngage', :path => '../node_modules/react-native-moengage', :modular_headers => true
  pod 'FirebaseCore', :modular_headers => true
  pod 'Firebase/Crashlytics', :modular_headers => true
  pod 'Firebase/RemoteConfig', :modular_headers => true
  pod 'FirebaseABTesting', :modular_headers => true
  pod 'FirebaseInstallations', :modular_headers => true
  pod 'FirebaseSessions', :modular_headers => true
  pod 'FirebaseCoreExtension', :modular_headers => true
  pod 'GoogleDataTransport', :modular_headers => true
  pod 'nanopb', :modular_headers => true
  pod 'GoogleUtilities', :modular_headers => true
  pod 'RecaptchaEnterprise', :modular_headers => true
  pod 'RecaptchaInterop', :modular_headers => true

  pod 'react-native-config', :path => '../node_modules/react-native-config'
  # pod 'RNShare', :path => '../node_modules/react-native-share', :modular_headers => true



  post_install do |installer|
    # https://github.com/facebook/react-native/blob/main/packages/react-native/scripts/react_native_pods.rb#L197-L202
    begin
      react_native_post_install(
        installer,
        config[:reactNativePath],
        :mac_catalyst_enabled => false,
        # :ccache_enabled => true
      )
    rescue => e
      if e.message.include?("invalid byte sequence in UTF-8")
        puts "⚠️  Warning: Encountered binary plist in Pods, manually setting RCTNewArchEnabled on main Info.plist files..."
        # Manually set RCTNewArchEnabled on main app Info.plist files
        require 'xcodeproj'
        main_info_plists = [
          'app6thstreet/Info.plist',
          'MoEngageNotificationService/Info.plist',
          'PushTemplateExtension/Info.plist'
        ]
        main_info_plists.each do |plist_path|
          full_path = File.join(installer.sandbox.project_path.dirname, '..', plist_path)
          if File.exist?(full_path)
            begin
              plist = Xcodeproj::Plist.read_from_path(full_path)
              plist['RCTNewArchEnabled'] = true
              Xcodeproj::Plist.write_to_path(plist, full_path)
              puts "✅ Set RCTNewArchEnabled on #{plist_path}"
            rescue => plist_error
              puts "⚠️  Could not update #{plist_path}: #{plist_error.message}"
            end
          end
        end
      else
        raise e
      end
    end
    
    # Fix privacy manifest conflicts - remove manual PrivacyInfo.xcprivacy references
    # to avoid conflicts with CocoaPods automatic privacy manifest aggregation
    puts "🔧 Checking for privacy manifest conflicts..."
    
    # Find the main project file
    project_path = File.join(installer.sandbox.project_path.dirname, '../app6thstreet.xcodeproj/project.pbxproj')
    if File.exist?(project_path)
      content = File.read(project_path)
      
      # Check if manual PrivacyInfo.xcprivacy references exist
      has_build_file = content.match(/^\s*[A-F0-9]+ \/\* PrivacyInfo\.xcprivacy in Resources \*\/ = \{isa = PBXBuildFile/)
      has_file_ref = content.match(/^\s*[A-F0-9]+ \/\* PrivacyInfo\.xcprivacy \*\/ = \{isa = PBXFileReference/)
      has_group_ref = content.match(/^\s*[A-F0-9]+ \/\* PrivacyInfo\.xcprivacy \*\/,/)
      has_resource_ref = content.match(/^\s*[A-F0-9]+ \/\* PrivacyInfo\.xcprivacy in Resources \*\/,/)
      
      if has_build_file || has_file_ref || has_group_ref || has_resource_ref
        puts "⚠️  Found manual PrivacyInfo.xcprivacy references that conflict with CocoaPods aggregation"
        
        # Remove PBXBuildFile entries for PrivacyInfo.xcprivacy
        content.gsub!(/^\s*[A-F0-9a-f]+ \/\* PrivacyInfo\.xcprivacy in Resources \*\/ = \{isa = PBXBuildFile;[^}]+\};\n?/, '')
        content.gsub!(/^\s*[A-F0-9a-f]+ \/\* PrivacyInfo\.xcprivacy \*\/,\n?/, '')
        content.gsub!(/^\s*[A-F0-9a-f]+ \/\* PrivacyInfo\.xcprivacy in Resources \*\/,\n?/, '')
        
        File.write(project_path, content)
        puts "✅ Successfully removed manual PrivacyInfo.xcprivacy references from Xcode project"
        puts "ℹ️  CocoaPods will handle privacy manifest aggregation automatically"
      else
        puts "✅ No privacy manifest conflicts detected - project is clean"
      end
    else
      puts "⚠️  Could not find project.pbxproj file at: #{project_path}"
    end
    
    # Fix Folly compilation error for TestFlight builds
    # This fixes the "No matching function for call to object of type 'const conditional_t'" error
    puts "🔧 Applying Folly compilation fixes..."
    installer.pods_project.targets.each do |target|
      target.build_configurations.each do |config|
        config.build_settings['EXCLUDED_ARCHS[sdk=iphonesimulator*]'] = 'x86_64' 
        # Apply fixes specifically to RCT-Folly
        if target.name == 'RCT-Folly'
          config.build_settings['OTHER_CPLUSPLUSFLAGS'] ||= ['$(OTHER_CFLAGS)']
          existing_flags = config.build_settings['OTHER_CPLUSPLUSFLAGS'] || []
          # Convert to array if it's a string
          if existing_flags.is_a?(String)
            existing_flags = existing_flags.split(' ')
          end
          # Add Folly-specific flags if not already present
          folly_flags = [
            '-DFOLLY_NO_CONFIG',
            '-DFOLLY_MOBILE=1',
            '-DFOLLY_USE_LIBCPP=1',
            '-DFOLLY_CFG_NO_COROUTINES=1',
            '-DFOLLY_HAVE_CLOCK_GETTIME=1',
            '-Wno-error'
          ]
          folly_flags.each do |flag|
            existing_flags << flag unless existing_flags.include?(flag)
          end
          config.build_settings['OTHER_CPLUSPLUSFLAGS'] = existing_flags
          config.build_settings['CLANG_CXX_LANGUAGE_STANDARD'] = 'c++20'
          config.build_settings['CLANG_WARN_QUOTED_INCLUDE_IN_FRAMEWORK_HEADER'] = 'NO'
        end
        
        # Ensure consistent C++ standard for all pods except extension targets
        skip_targets = ['PushTemplateExtension', 'MoEngageNotificationService']
        if config.build_settings['CLANG_CXX_LANGUAGE_STANDARD'].nil? && !skip_targets.include?(target.name)
          config.build_settings['CLANG_CXX_LANGUAGE_STANDARD'] = 'c++20'
        end
      end
    end
    puts "✅ Applied Folly compilation fixes"
  end
end

target "PushTemplateExtension" do
  pod 'MoEngageRichNotification'
end

target "MoEngageNotificationService" do
  pod 'MoEngageRichNotification'
end
