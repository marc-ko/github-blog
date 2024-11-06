---
title: "Flutter Error: 'UnmodifiableUint8ListView' is restricted and can't be extended or implemented" 
date: "2024-10-06"
---


# Flutter Error: 'UnmodifiableUint8ListView' is restricted and can't be extended or implemented

### Problem Reproduction
After upgrading Flutter to the ^3.24 version, you may encounter the following error message:
Flutter Error: ‘UnmodifiableUint8ListView‘ is restricted and can‘t be extended or implemented
---


### Definition of the problem
In the Upgraded version unmodifiable_typed_data.dart file, the UnmodifiableUint8ListView class is removed from the dart:typed_data library. 
This class is restricted and can't be extended or implemented. 

Since there is still bunch of libraries or old libraries is still using this library, you may fucked up the build process.
---

### Solution 
To solve this issue, you can use the following steps:
**If your error related to win32 package, you can use the following steps:**
```yaml
dependency_overrides:
     win32: ^5.5.4  
```

or ```flutter pub upgrade win32``` to upgrade the win32 package to the latest version.

But my problem is related to the tflite_flutter package
you may 
```bash
     flutter pub upgrade tflite_flutter
```

since **tflite_flutter_helper** is using and dependant on 0.9.1 version of the **tflite_flutter package**, you need to refactor the whole helper package or just revert back to the old version of flutter to get back old dart:typed_data library.
```bash
flutter downgrade 3.22.0
```

> After that you can run the ```flutter clean``` and ```flutter pub get``` to get rid of the error.

