# flavorを使って development と  staging と　 productionの配布をテスターに配布できるようにする

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

  desc "development build apk and aab"
  lane :development do
    flutter_build(
      build_type: "release",
      flavor: "development",
      target: "lib/main_development.dart"
    )
  end

  desc "staging build apk and aab"
  lane :staging do
    flutter_build(
      build_type: "release",
      flavor: "staging",
      target: "lib/main_staging.dart"
    )
  end

  desc "production build apk and aab"
  lane :production do
    flutter_build(
      build_type: "release",
      flavor: "production",
      target: "lib/main_production.dart"
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

### productioをビルドしてみる
```sh
bundle exec fastlane production
```

![image](https://github.com/rensawamo/flavor-fastlane/assets/106803080/a8b205f2-8abd-4e0a-9167-18f58fb29b9b)


### ルートディレクトリ　に aabファイルと apkができていることを確認する
![image](https://github.com/rensawamo/flavor-fastlane/assets/106803080/56351c3a-aea5-40e9-9971-405fbbb95c5d)








