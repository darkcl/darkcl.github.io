---
layout: post 
title: 	"RxSwift Learning Notes: Wrapping third-party library delegates" 
subtitle: "Remove ugly extensions" 
date: 2017-09-07 
tags: [ios, rxswift, delegate] 
author: "Yeung Yiu Hung" 
comments: true
---

In Mustor, the link taps events are handle in ```TTTAttributedLabelDelegate```, 

```objc
#pragma mark - TTTAttributedLabelDelegate
- (void)attributedLabel:(TTTAttributedLabel *)label
   didSelectLinkWithURL:(NSURL *)url{
    NSLog(@"Tap: %@", url.absoluteString);
    
    // Link Tap Event Handling
}
```

Using RxSwift ```DelegateProxyType``` we can wrap the link tap event as an observerable:

```swift

// RxTTTAttributeLabel.swift

import UIKit

import TTTAttributedLabel

import RxSwift
import RxCocoa

fileprivate class RxTTTAttributedLabelDelegateProxy: DelegateProxy, TTTAttributedLabelDelegate, DelegateProxyType {
    static func currentDelegateFor(_ object: AnyObject) -> AnyObject? {
        let label: TTTAttributedLabel = object as! TTTAttributedLabel
        return label.delegate
    }
    
    static func setCurrentDelegate(_ delegate: AnyObject?, toObject object: AnyObject) {
        let label: TTTAttributedLabel = object as! TTTAttributedLabel
        label.delegate = delegate as? TTTAttributedLabelDelegate
    }

}

extension Reactive where Base: TTTAttributedLabel {
    
    public var delegate: DelegateProxy {
        return RxTTTAttributedLabelDelegateProxy.proxyForObject(base)
    }
    
    public var selectedLink: Observable<URL> {
        let selector = #selector(
            ((TTTAttributedLabelDelegate.attributedLabel(_:didSelectLinkWith:))!
                as (TTTAttributedLabelDelegate) -> (TTTAttributedLabel, URL) -> Void)) // Hanling for Objective-C Delegate which have same name but different param type
        return delegate.methodInvoked(selector)
            .map { params in
                return params[1] as! URL
        }
    }
    
}

// In View

self.postContentLabel.rx.selectedLink
    .bind(to: viewModel.linkTrigger)
    .disposed(by: disposeBag)

// In ViewModel

self.linkTrigger
    .observeOn(MainScheduler.instance)
    .subscribe(onNext: {[weak self] url in
        guard let weakSelf = self else{
            return
        }
        // Handle the link
        
    })
    .disposed(by: disposeBag)

```