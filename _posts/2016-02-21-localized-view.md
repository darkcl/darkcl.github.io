---
layout: post 
title: 	"Localized your view within your xib" 
subtitle: "Application specific localization can be a nightmare." 
date: 2016-02-21 
tags: [ios, localization, IBInspectable] 
author: "Yeung Yiu Hung" 
comments: true
---

As I start working on next phase of my project, I notice something common with my ViewControllers, a list of IBOutlet that only use once. They only use once for setting up static strings. Because this app has application specific localization (select language within the app, not using system's one), localizing the xib is not an option. So my view controller have a lot of outlet that only use once.

One day, I read this [article](http://nshipster.com/ibinspectable-ibdesignable/) on NSHipster, and I think, what if I can set all the keys in my xib with IBInspectable, and using Objective-C Category to set up my string without create a list of IBOutlet that only call once?

So I created [LocalizedView](https://github.com/darkcl/LocalizedView), a list of uiview categories that aims to set up static localized string within the xib, the app specific localization is done by [AMLocalizedString](http://aggressive-mediocrity.blogspot.hk/2010/03/custom-localization-system-for-your.html), which is used in multiple projects that I have worked on.

The logic of this library is simple: I created a UIView category which contains an IBInspectable propery of localized key and a UIViewController category which loop through subviews and set the static string. You can see how simple to set the string from the follow video.

<iframe width="420" height="315" src="https://www.youtube.com/embed/fjumi94jlWo" frameborder="0" allowfullscreen></iframe>