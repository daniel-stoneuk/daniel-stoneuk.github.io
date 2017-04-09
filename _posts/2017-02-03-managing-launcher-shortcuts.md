---
published: true
layout: post
date: '2017-02-03 21:11 +0000'
title: Managing Launcher Shortcuts
---
## Android Nougat

I recently found myself needing to complete a German writing task, respond to a few emails and revise for a geography test. Obviously I decided to procratinate by adding a small feature to my Android App - [Monitor for EnergyHive & Engage](https://play.google.com/store/apps/details?id=com.danielstone.energyhive). 

Launcher shortcuts seemed pretty simple at first. All I had to do was create the shortcut info with the intent, icon and label and then add it using `ShortcutManager`.

It worked great until I hit the limit at which instead of silently handling the exception, the app decides to crash.

Okay, so I need to keep the number of shortcuts below 5 (inclusive). This code should work, right?

```java
List<ShortcutInfo> oldShortcuts = shortcutManager.getDynamicShortcuts();
List<ShortcutInfo> newShortcuts = new ArrayList<>();
for (int i = 0; i < (oldShortcuts.size() < 5 ? oldShortcuts.size() : 5); i++) {
    newShortcuts.add(oldShortcuts.get(i));
}
newShortcuts.add(shortcut);
shortcutManager.setDynamicShortcuts(newShortcuts);
```

Nope. For some reason the icons are lost in the process. 

![Image]({{site.baseurl}}/assets/device-2017-02-03-210527.png){:.screenshot}

I found the best way of managing them for now is like so: 

```java
ArrayList<ShortcutInfo> dynamicShortcuts = new ArrayList<>();
dynamicShortcuts.addAll(shortcutManager.getDynamicShortcuts());

boolean alreadyAdded = false;
for (ShortcutInfo s :
        dynamicShortcuts) {
    if (s.getId().equals(shortcut.getId())) {
        shortcutManager.removeDynamicShortcuts(Arrays.asList(shortcut.getId()));
        alreadyAdded = true;
        break;
    }
}

if (!alreadyAdded && dynamicShortcuts.size() > 3) {
    shortcutManager.removeDynamicShortcuts(Arrays.asList(dynamicShortcuts.get(0).getId()));
}

shortcutManager.addDynamicShortcuts(Arrays.asList(shortcut));
```

Let me know if you have a solution...
