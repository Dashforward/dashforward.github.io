---
layout: post
title: Writing ios plugins for Unity
subtitle: 
comments: false
---

# Part 1: Hello World

## Writing a library

Create a new **Static Library** project with just one file.

<img src="{{site.baseurl}}/assets/img/2020-11-20-writing-ios-plugins/writing-ios-plugins-001.png">

```objectivec
#import <Foundation/Foundation.h>
#import <UIKit/UIKit.h>

void _ConsoleLog(char* message) {
  NSString *nsMessage = [NSString stringWithUTF8String:message];
  NSLog(@"%@", nsMessage);
}
```

## Copying the library

Build the library and copy it to your Unity project.

To automate that go the build settings in xcode, select your library as a target add a new Run Script Build Phase

<img src="{{site.baseurl}}/assets/img/2020-11-20-writing-ios-plugins/writing-ios-plugins-002.png">

Here we use a path relative to our project.

```shell
cp -f -R ${BUILT_PRODUCTS_DIR}/${APPLICATION_NAME} ${PROJECT_DIR}/../../Assets/Plugins
```

<img src="{{site.baseurl}}/assets/img/2020-11-20-writing-ios-plugins/writing-ios-plugins-003.png">

## Calling from C#

```csharp
using System.Runtime.InteropServices;

[DllImport("__Internal")]
private static extern void _ConsoleLog(string message);

public void NativeConsoleLog(string message) {
  _ConsoleLog(message);
}
```
