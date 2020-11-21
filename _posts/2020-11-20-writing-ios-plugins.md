---
layout: post
title: Writing ios plugins for Unity
subtitle: 
comments: false
---

# Part 1: Hello World

## Writing a library

Create a new **Static Library** project with just one file.

```objectivec
#import <Foundation/Foundation.h>
#import <UIKit/UIKit.h>

void _ConsoleLog(char* message) {
  NSString *nsMessage = [NSString stringWithUTF8String:message];
  NSLog(@"%@", nsMessage);
}
```

Build the library and copy it to your Unity project.

## Calling from C#

```C#
using System.Runtime.InteropServices;

[DllImport("__Internal")]
private static extern void _ConsoleLog(string message);

public void NativeConsoleLog(string message) {
  _ConsoleLog(message);
}
```
