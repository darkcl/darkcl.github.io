---
layout: post 
title: 	"RxSwift Learning Notes: Color Theme" 
subtitle: "Themeing your apps without Notification Center" 
date: 2017-08-17 
tags: [ios, rxswift, theming] 
author: "Yeung Yiu Hung" 
comments: true
---

Why am I writing this?
---

This is some learning notes from revamping Mustor. Hope to remind myself as RxSwift is quite large framework to learn.

RxSwift + Colors
---

For my new implementation of themeing, the core class is the ```ThemeingService``` :

```swift
final class ThemeingService {
    var isLightMode: Variable<Bool> = Variable(GlobalHelper.isLightMode()) 
    // Can be a color tone (light, yellow, blue)
    
    let disposeBag = DisposeBag()
    
    static let shared = ThemeingService()
    
    init() {
        isLightMode.asObservable().subscribe(onNext: { value in
            GlobalHelper.setLightMode(value) // Save to user default
        })
        .disposed(by: disposeBag)
    }
}
```

Everytime the value of ```isLightMode``` is change, it will trigger ```GlobalHelper``` to save current mode. Since ```isLightMode``` is ```Variable```, it can be observe, like the follow code: 

```swift
extension UIColor {
    public static func rx_colorKey(_ key: String) -> Driver<UIColor> {
        return ThemeingService.shared.isLightMode.asDriver().startWith(GlobalHelper.isLightMode())
                    .map({ value  in
                        return UIColor.init(forColorKey: key, isLightMode: value) // Dictonary mapping for different color keys
                    })
    }
}
```

In this extension of ```UIColor```, I create a function to return ```Driver<UIColor>```, everytime ```isLightMode``` is changed, the ```Driver<UIColor>``` can emit the color of the key. Then we can use this color in our component:

```swift
extension Reactive where Base: UIRefreshControl {

    // Implemtation of .rx extensions
    public var tintColor: UIBindingObserver<Base, UIColor> {
        return UIBindingObserver(UIElement: self.base, binding: { (view, color) in
            view.tintColor = color
        })
    }
}
```


Binding Single color to views
```swift
// In your views

UIColor.rx_colorKey("label")
    .drive(refreshControl.rx.tintColor)
    .disposed(by: disposeBag)

```

Binding Multiple colors to `NSAttributedString`

```swift
// In your view models
let navBarTitle: Driver<NSAttributedString> = 
    Observable.combineLatest(UIColor.rx_colorKey("label").asObservable(), UIColor.rx_colorKey("label_inactive").asObservable())
            .flatMap({ (color1, color2)
            // You can use this to construct your NSAttributedString
            // ...

            return Observable<NSAttributedString>.just(attrTitle)
        })
        .asDriver(onErrorJustReturn: NSAttributedString())

// In your views
navBarTitle.drive(titleButton.rx.attributedTitle(for: .normal))
    .disposed(by: disposeBag)

```


