![image](https://github.com/rensawamo/flavor-fastlane/assets/106803080/59dd5ebd-b030-4d48-9b05-392e9e14679b)# flavorを使って development と  staging と　 productionの配布をテスターに配布できるようにする

## fvnを使用
```sh
fvm list
fvm use 上記ver
```

## pubspec.yaml の dev_dependencies に以下を追記しプロジェクトに追加
```sh
flutter_flavorizr:
```

## pubspec.yamlの下に以下を追加
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


## 各フレーバーのビルド設定を行う
![image](https://github.com/rensawamo/flavor-fastlane/assets/106803080/92f8523a-9e82-4560-af20-9000545e1140)

このライブラリの使用により
xcodeでもフレーバの設定を自動追加できる
![image](https://github.com/rensawamo/flavor-fastlane/assets/106803080/9a4cc80c-798d-4a36-8b1a-3407c1b3548e)



