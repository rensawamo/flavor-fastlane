# flavor(flutter_flavorizr)とfastlaneをつかった　iOS と　anoroidの　自動配布
### fvnを使用
```sh
fvm list
fvm use 上記ver
```

### pubspec.yaml の dev_dependencies に以下を追記しプロジェクトに追加
```sh
flutter_flavorizr:
```

### pubspec.yamlの下に以下を追加
最近は dart defineに乗り換えているそちらを推奨
```sh
flavorizr:
  app:
    android:
      flavorDimensions: "flavor-type"

  flavors:
    development:
      app:
        name: "Development"
      android:
        applicationId: "com.YOURTEAMNAME.fastlane.flavor.dev"
      ios:
        bundleId: "com.YOURTEAMNAME.fastlaneFlavor.dev"

    staging:
      app:
        name: "Staging"
      android:
        applicationId: "com.YOURTEAMNAME.fastlane.flavor.staging"
      ios:
        bundleId: "com.YOURTEAMNAME.fastlaneFlavor.staging"

    production:
      app:
        name: "Production"
      android:
        applicationId: "com.YOURTEAMNAME.fastlane.flavor.prod"
      ios:
        bundleId: "com.YOURTEAMNAME.fastlaneFlavor.prod"
```

以下コマンドを実行
```sh
 flutter pub run flutter_flavorizr
```


### 各フレーバーのビルド設定を行う
![image](https://github.com/rensawamo/flavor-fastlane/assets/106803080/92f8523a-9e82-4560-af20-9000545e1140)

このライブラリの使用により
xcodeでもフレーバの設定を自動追加できる
![image](https://github.com/rensawamo/flavor-fastlane/assets/106803080/9a4cc80c-798d-4a36-8b1a-3407c1b3548e)

![image](https://github.com/rensawamo/flavor-fastlane/assets/106803080/eb300f33-cc11-46a1-8a03-ccea889d7e30)

# Android 
# release設定
## アップロードkeyの準備

### プロジェクトファイルの andoroid/app で以下のコマンドを実行し証明書を作成する
alias_nameは覚えやすい名前にする

```sh
keytool -genkey -v -keystore release.jks -alias alias_name -keyalg RSA -keysize 2048 -validity 10000
```

### github actionの環境変数で使うためエンコードしておく
```sh
 openssl base64 -in release.jks  -out release.pem 
```

### android/keystore.propertiesを作成（local用）
```sh
storePassword=パスワード
keyPassword=パスワード
keyAlias=alias_name
storeFile=release.jks
```

### android/app/build.gradeの編集
```sh
// android 設定の上
def keystorePropertiesFile = rootProject.file("keystore.properties")
android {
....
// defalutConfigの下
signingConfigs {
        release {
            if (System.getenv("GITHUB_ACTIONS")) { // gitaction用
                storeFile file("release.jks")
                storePassword System.getenv()["STORE_PASSWORD"]
                keyAlias System.getenv()["KEY_ALIAS"]
                keyPassword System.getenv()["KEY_PASSWORD"]
            } else if (keystorePropertiesFile.exists()) { // local用
                def keystoreProperties = new Properties()
                keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
                keyAlias keystoreProperties['keyAlias']
                keyPassword keystoreProperties['keyPassword']
                storeFile file(keystoreProperties['storeFile'])
                storePassword keystoreProperties['storePassword']
            }
        }
    }

....
// この中に releaseの追加
buildTypes {
        release {
            signingConfig signingConfigs.release

```


## fastlane設定
### rubyのバージョンを確認する
```sh
 ruby --version
```


### プロジェクトでのrubyのバージョンを固定する
```sh
rbenv local 3.3.0(上記ver)
```


### Flutter プロジェクト直下で以下を実行し bundler をインストール
```sh
gem install bundler
```


### Gemfile を作成
```sh
bundle init
```


### 作成された Gemfile の一番下に以下を追加して fastlane を明記
```sh
gem 'fastlane'
```


### 以下コマンドで fastlane のインストール
```sh
bundle config --local path vendor/bundle
bundle install
bundle package
bundle install --local
```


### android　ディレクトリに移動して以下のコマンドを実行し、fastlaneディレクトリが作成されていることを確認する
```sh
cd android
bundle exec fastlane init
```
![image](https://github.com/rensawamo/flavor-fastlane/assets/106803080/27edfae3-fce8-4c1e-af24-29f0142944e7)


### Fastlaneファイルを以下に変更
```sh
default_platform(:android)

platform :android do

#　内部テスト
  desc "development build apk and aab"
  lane :development do
    flutter_build(
      build_type: "release",
      flavor: "development",
      target: "lib/main_development.dart"
    )
  end
  desc "upload_to_play_store"
      lane :upload_inside do
        upload_to_play_inside(
          track: 'internal',
          release_status: 'draft',
          package_name: "com.YOURNAME.fastlane.flavor.dev",
          track: "internal",
          aab: "../build/app/outputs/bundle/productionRelease/app-development-release.aab"
        )
    end

#  オープンテスト
  desc "staging build apk and aab"
  lane :staging do
    flutter_build(
      build_type: "release",
      flavor: "staging",
      target: "lib/main_staging.dart"
    )
  end
  desc "upload_to_upload_open_test"
  lane :upload_open_test do
    upload_to_play_store(
      track: 'open',
      release_status: 'completed',
      package_name: "com.YOURNAME.fastlane.flavor.staging",
      track: "internal",
      aab: "../build/app/outputs/bundle/stagingRelease/app-staging-release.aab"
  )
  end

#  本番
  desc "production build apk and aab"
  lane :production do
    flutter_build(
      build_type: "release",
      flavor: "production",
      target: "lib/main_production.dart"
    )
  end
  desc "Upload to Play Store"
  lane :upload_production_to_play_store do
    upload_to_play_store(
      track: 'production',
      release_status: 'completed',
      package_name: "com.YOURNAME.fastlane.flavor.prod",
      aab: "../build/app/outputs/bundle/productionRelease/app-production-release.aab"
    )
  end



# Build the above with -- to specify flavor　
  desc "common build"
  private_lane :flutter_build do |options|
    target = options[:target]
    build_type = ["--", options[:build_type].downcase].join
    flavor = options[:flavor].downcase

    gradle(task: "clean")
    sh("fvm", "flutter", "build", "apk", build_type, "--flavor", flavor, "--target", target)
    sh("fvm", "flutter", "build", "appbundle", build_type, "--flavor", flavor, "--target", target)
  end
end
```

### production をビルドしてみる
```sh
bundle exec fastlane production
```



### ルートディレクトリ　に aabファイルと apkができていることを確認する
![image](https://github.com/rensawamo/flavor-fastlane/assets/106803080/56351c3a-aea5-40e9-9971-405fbbb95c5d)


# fastlane アップロード
## jsonの作成
https://docs.fastlane.tools/actions/supply/
以下より jsonファイルをダウンロードする


以下のコマンドで成功と出たら上記のjsonでgoogle storeとのコネクトが可能になるのでプロジェクトに埋め込んでいく
```sh
fastlane run validate_play_store_json_key json_key:/path/to/your/downloaded/file.json
```

![image](https://github.com/rensawamo/flavor-fastlane/assets/106803080/7858610b-889c-4779-8966-bcb383eac4e4)


### 環境変数の設定
.envを作成
```sh
PLAY_STORE_CREDENTIALS_JSON_PATH="path/to/your/play-store-credentials.json"
PACKAGE_NAME="my.package.name"
```

### Appfileを書き換える
```sh
json_key_file(ENV["PLAY_STORE_CREDENTIALS_JSON_PATH"])
package_name(ENV["PACKAGE_NAME"])
```


### 以下を実行し　プロジェクトと GooglePlayStoreを紐付ける
```sh
fastlane supply init
```


エラーの場合は
supply/lib/supply/client.rbを変更
```sh
def latest_version(track)
     - latest_version = tracks.select { |t| t.track == Supply::Tracks::DEFAULT }.map(&:releases).flatten.reject { |r| r.name.nil? }.max_by(&:name)
     + latest_version = tracks.select { |t| t.track == Supply::Tracks::DEFAULT }.map(&:releases).flatten.reject { |r| (r&.name).nil? }.max_by(&:name)

```
### aab ファイルの作成
```sh
bundle exec fastlane development
```

### 一度手動でGoogle Developerにアップロード(内部テストへ)
Google play console の内部テストに出来上がった build/app/outputs/bundle/productionRelease/app-production-release.aabをドラッグドロップ



<img width="884" alt="image" src="https://github.com/rensawamo/flavor-fastlane/assets/106803080/9cc776ec-6f30-4f97-9688-9e0d8e56b770">


### ver 変更して自動デプロイ
pubspec.yamlのアプリバージョンを上げる
```sh
version: 1.0.0+2
```

### 内部テストへアップロード
```sh
bundle exec fastlane upload_to_inside
```


# iOS
matchを使用。
githubに本来は情報を格納（このリポジトリにはしていないプライベートにする）

## iOSのディレクトリに移動
```sh
cd ios
bundle exec fastlane init
```

### Falslaneのファイルを以下に修正
```sh
 
 default_platform(:ios)
platform :ios do

# developmentを省略

  default_platform(:ios)
platform :ios do

  desc "テストフライトへ"
  lane :staging_upto_appstore do
    api_key = app_store_connect_api_key(
      key_id: ENV["KEY_ID"],
      issuer_id: ENV["ISSUER_ID"],
      key_filepath: ENV["KEY_FILEPATH"],
      duration: 1200,
      in_house: false
    )
    match(
      api_key: api_key,
      type: "appstore",
      app_identifier: ["com.YOURTEAMNAME.fastlaneFlavor.staging"],
      force_for_new_devices: true
    )
    # match を使用している場合でも証明書の更新は行わずに、リポジトリからの取得のみを行う
    match(type: "appstore", readonly: true)
    gym(
      scheme: "staging",
      export_method: "app-store",
      export_options: {
        provisioningProfiles: {
          "com.YOURTEAMNAME.flavorFastlane.staging" => "match AppStore com.YOURTEAMNAME.fastlaneFlavor.staging"
        }
      }
    )
    # ベータ版をappstoreへアップロード
    upload_to_testflight(
      api_key: api_key,
      skip_waiting_for_build_processing: true, # ビルド処理の完了を待たない
      skip_submission: true # ビルドのレビュー提出をスキップ
    )
  end

  desc "本番環境へ"
  lane :prod_upto_appstore do
    api_key = app_store_connect_api_key(
      key_id: ENV["KEY_ID"],
      issuer_id: ENV["ISSUER_ID"],
      key_filepath: ENV["KEY_FILEPATH"],
      duration: 1200,
      in_house: false
    )
    match(
      api_key: api_key,
      type: "appstore",
      app_identifier: ["com.YOURTEAMNAME.fastlaneFlavor.prod"],
      force_for_new_devices: true
    )
    # match を使用している場合でも証明書の更新は行わずに、リポジトリからの取得のみを行う
    match(type: "appstore", readonly: true)

    gym(
      scheme: "production",
      export_method: "app-store",
      export_options: {
        provisioningProfiles: {
          "com.YOURTEAMNAME.flavorFastlane.prod" => "match AppStore com.YOURTEAMNAME.fastlaneFlavor.prod"
        }
      }
    )
    # 本番製品をappstoreへアップロード
    upload_to_app_store(
      precheck_include_in_app_purchases: false,
      api_key: api_key,
      skip_metadata: true,
      skip_screenshots: true,
      skip_binary_upload: false
    )
  end
end


```

### App Store Connect API Key 作成
apple store connect　の　以下ページの
Issuer ID、キーID、ダウンロードp8を準備
![image](https://github.com/rensawamo/flavor-fastlane/assets/106803080/33d4875d-eadd-4563-898d-ba3acb803c97)

この時、権限を以下に設定



![image](https://github.com/rensawamo/flavor-fastlane/assets/106803080/e64d5f4b-c8ab-43ac-a75d-0d1c28b3ace6)


.envに情報を入れ込む
```sh
# Fastfile
 KEY_ID = "<App Store Connect：キーID>"
 ISSUER_ID = "<App Store Connect: Issuer ID>"
 KEY_FILEPATH = "<p8ファイルの格納Path>"
```

以下コマンドを実行し、gitを選択しリポジトリに証明書などを埋め込む。

```sh
fastlane match init
```
![image](https://github.com/rensawamo/flavor-fastlane/assets/106803080/72a1f1b1-b4cd-43c6-8d96-448053aff15c)



これにより、チームメンバーは以下のコマンドより証明書を取得可能
```sh
fastlane match appstore(development)
```

[Apple Developer Program](https://developer.apple.com/programs/)へ移動し、Identifiersを
com.YOURTEAMNAME.fastlaneFlavor.staging(flavor)（３種類）作成してapple connect へ移動し、アプリの登録をおこなう


以下でflavorを使ったデプロイが可能になる
例えば本番環境

```sh
fastlane ios prod_upto_appstore
```
appstoreにアップロード
<img width="543" alt="image" src="https://github.com/rensawamo/flavor-fastlane/assets/106803080/73989371-e0a8-4df5-bf15-8edb40d885e2">

![image](https://github.com/rensawamo/flavor-fastlane/assets/106803080/1c8724a6-955d-41e0-a4f2-4d5db280767c)
（再度コマンド実行したらバージョンが上がる）
<img width="856" alt="image" src="https://github.com/rensawamo/flavor-fastlane/assets/106803080/ddb52223-2e85-4049-9c13-ad0cc42a5705">


以下のように、

developmentでは 内部テストへ


stagingではテストフライト、


productionでは本番へというように環境を変えた配信を行える


上記flavorでアップロード先を適宜分けることが可能




