---
title: Load ipa file into iOS using Flutter build ipa (WITHOUT Apple Dev Account)
date: 2024-12-10
---

# It is possible to build a ipa file and load it in iPhone *EVEN* without a Paid Apple Developer Program Account
## Prerequesites 
* Install AltServer 
* Have flutter and Using Mac
### First, we build the ios app first 

```bash
flutter build ios --release --no-codesign
```

### Find Your Payload File (Runner.app)
> It should be located in the build folder if you didn't set another path for it
While you are in root of flutter project, excute below
``` bash
cd build/ios/archive/Runner.xcarchive/Product/Applications/
```
### Packing the application 
```bash
mkdir Payload/
mv Runnner.app/ Payload/
```

### Archive it as an ipa file
```bash
zip -qq -r -9 filename.ipa Payload
```

* Then you have a fresh ipa file of your flutter app

### Then, move(mv) to somewhere you like e.g ~/Desktop or ~/Downloads
### Go to the menu bar, press the AltServer icon (Ensure you launched Altserver in the background)
### HOLD the Option(For MacOS), and press Sideload ipa..
### Choose the fresh ipa file you just created and volia!
* https://altstore.io/
> If you still have confusion, just go look at the altstore.io docs
