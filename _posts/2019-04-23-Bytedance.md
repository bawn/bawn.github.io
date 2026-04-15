---
layout: post
title: "A Brief Analysis of ByteDance Products"
date: 2019-04-23
comments: true
categories: [iOS]
tags: [ByteDance]
keywords: [ByteDance]
publish: true
## description: 字节跳动
---


This article provides a simple analysis of several representative products under ByteDance using three approaches: Reveal, AppSight, and IPA package inspection. Below is some basic information:

Product	Version	Main Language	Third-party Libraries	Interface Builder
Toutiao	7.1.7	Objective-C	AppSight	Storyboard
Pipixia	1.7.2	Objective-C	AppSight	/
Douyin	5.8.0	Objective-C	AppSight	Storyboard (Launch Screen)
Lark (Feishu)	2.6.3	Swift	/	Storyboard (Launch Screen)


⸻

Toutiao

Partial Page Structure
	•	Home: UICollectionView + UITableView (unexpected, right?)
	•	Article Detail Page: WKWebView (content) + Native (comments), wrapped inside a UIScrollView

For more details, refer to: iOS News App Content Page Technical Exploration.

Third-party Libraries

From the imported libraries, the following are commonly used in news apps:
	•	GDataXML-HTML
	•	libwebp
	•	YYKit
	•	TTTAttributedLabel

These libraries also appear in NetEase News.

For data transmission, Toutiao uses protobuf instead of traditional JSON. GIF rendering is handled by FLAnimatedImage.

Additionally, absolute layout is used in modules like:
	•	Home
	•	Xigua Video
	•	Short Video
	•	Comment section

This is likely for performance optimization.

React Native Usage

Toutiao includes React Native (v0.55.4). In the IPA package, I found:

assets/react-native-feedcell/components/

with folders like:
	•	interest-tags
	•	weather

Initially, I thought these features were not implemented using React Native after inspecting via Reveal. However, it was eventually confirmed on the “Follow Interesting People” page, which contains deeply nested RCTView structures.

Flutter Integration

Interestingly, the IPA package also contains:
	•	flutter_assets
	•	Frameworks/Flutter.framework

This confirms that Toutiao is using Flutter, likely for short video-related pages.

Other Findings
	•	WCDB.framework (Tencent database framework)
	•	yw_1222.jpg (~1KB): likely an Alibaba security-related image
	•	vconsole.js (debugging tool)
	•	Lottie resources (pause_to_play_list)
	•	Night mode images (*_night in Assets.car)

For night mode, Toutiao uses an inheritance-based solution with base classes like:
	•	SSThemedView
	•	SSThemedLabel
	•	SSThemedButton

This raises a question: why not use something like DKNightVersion?

⸻

Pipixia

Compared to Toutiao, Pipixia (released in 2018) appears more conservative.
	•	Although Swift is introduced, there’s no visible usage in UI
	•	No clear signs of cross-platform frameworks
	•	Possibly inherited from the legacy Neihan Duanzi project

Notable Library
	•	AsyncDisplayKit (Texture)

However, it is only used in the “Image” tab on the home page.

Another interesting detail:
	•	A persistent MPVolumeView is attached to the keyWindow, though its exact purpose is unclear.

⸻

Douyin

Partial Page Structure
	•	Home: UIPageViewController + UITableView
	•	Channel Detail Page: UIScrollView + UICollectionView

(Implementation details are covered in another blog post.)

Observations

Reveal and IPA analysis didn’t reveal much interesting information.

However, something unusual:
	•	AppSight data shows outdated updates:
	•	China version: last updated in 2017
	•	Overseas version: 2017.12

My speculation:
At Douyin’s scale, it may have completely moved away from third-party libraries, relying entirely on in-house solutions since around 2017.

⸻

Lark (Feishu)

Lark is one of ByteDance’s newer products.
	•	First released about 8 months prior (version 1.9.1)
	•	Uses Swift
	•	Minimum iOS version: 10.0

⸻

App Architecture

keyWindow.rootViewController
    → UINavigationController
        → UITabBarController

Advantages
	•	All pages share a single navigation stack
	•	Easier management

(Requires that the tab bar is not persistent.)

⸻

WebView Optimization

1. WebView Pool

Two hidden WebViews are always kept in keyWindow.

Workflow:
	1.	When needed, a WebView is removed from keyWindow
	2.	Added to the target ViewController
	3.	A new WebView is created and added back to keyWindow

This significantly improves loading performance.

⸻

2. Offline Package

From sandbox analysis:

DocsSDK/ResourceService/docs_channel/eesz

Contains:
	•	.md5 file → integrity verification
	•	current_revision → version control
	•	resource/bear → local JS/CSS files
	•	template/bear → HTML templates

This reminds me of the Bear note-taking app.

Workflow
	1.	Check version
	2.	Download/update offline resources
	3.	Load HTML templates locally

let url = URL(fileURLWithPath: "")
let request = URLRequest(url: url)
webView.load(request)

All JS/CSS are loaded locally → faster rendering.

⸻

3. Request Interception

However, HTML templates still reference remote resources:

<script src="//s3.pstatp.com/..."></script>

This seems contradictory.

My conclusion:

👉 Lark intercepts WebView requests
👉 When JS/CSS/images are requested, it serves local cached files instead

This makes the resource/bear folder meaningful and ensures high performance.

⸻

Final Thoughts

From this analysis, ByteDance shows a few clear engineering strategies:
	•	Heavy performance optimization (absolute layout, WebView pooling)
	•	Hybrid tech stack (Objective-C + React Native + Flutter)
	•	Gradual evolution toward in-house solutions
	•	Strong emphasis on WebView optimization for complex apps like Lark

