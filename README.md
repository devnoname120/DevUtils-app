<h1 align="center">Dev<a href="https://github.com/DevUtilsApp/DevUtils-app#">Utils</a>.app</h1>
<h3 align="center">Developer Utilities for macOS</h3>

<p align="center">
  <img src="https://devutils.app/assets/screenshots/dark/jsonformatter.png">
  <br/>
  <a href="https://devutils.app/">🚀 &nbsp; Download</a> | <a href="https://devutils.app/demo">🎬 &nbsp; Demo & Screenshots</a> | <a href="https://github.com/DevUtilsApp/DevUtils-app/tree/master/TINOBHNYWE">📝 &nbsp; View source</a> | <a href="https://twitter.com/devutils_app">📣 &nbsp; Follow on Twitter</a>
  <br/>
  <br/>
  <a href="https://apps.apple.com/us/app/id1533756032"><img src="https://developer.apple.com/app-store/marketing/guidelines/images/badge-download-on-the-mac-app-store.svg" /></a> <a href="https://go.setapp.com/stp307?refAppID=445"><img src="https://developer.setapp.com/setapp-button/setapp.b6aff864.svg" /></a>
  
</p>


Format/Validate JSON, encode/decode Base64, convert timestamps, debug JWT… with just one click! Native macOS app and works offline.

# A friendly note ❤️
DevUtils is a **paid "source-available"** app. That means I'm selling the pre-built version of the app. It takes time and effort to develop, distribute, maintain, and provide supports. If you like the app, please consider [buying here](https://devutils.app) (one-time purchase). That would help me a lot! Thank you very much! 🙏

# Source code

This source code is a delayed version of DevUtils.

Development environment:
- Swift 5.1+
- Xcode 11.1+
- Swift Package Manager
- Carthage

# Build Instructions

- Clone the repository and open a terminal in it.
- Use Carthage to fetch the dependencies, delete the iOS dependency because it breaks the build, and finally build with xcode:
  ```shell
  arch -arch x86_64 /bin/sh
  cd DevUtilsApp

  carthage update --no-build
  rm Carthage/Checkouts/Highlightr/Highlightr.xcodeproj/xcshareddata/xcschemes/Highlightr-iOS.xcscheme  

  carthage build --platform macOS
  xcodebuild -scheme DevUtils CODE_SIGN_IDENTITY="" DEVELOPMENT_TEAM="" build
  ```

 - Congrats, you can now open the folder where you will find the built `DevUtils.app`:
   ```shell
   open ~/Library/Developer/Xcode/DerivedData/DevUtils-*/Build/Products/Debug/
   ``` 
 
# Contributing

Please see [CONTRIBUTING.md](https://github.com/DevUtilsApp/DevUtils-app/blob/master/CONTRIBUTING.md)

# Contact
- **FAQs**: https://devutils.app/faqs
- **Email**: admin@devutils.app
- **Twitter**: https://twitter.com/devutils_app
