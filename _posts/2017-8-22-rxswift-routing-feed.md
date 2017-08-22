---
layout: post 
title: 	"RxSwift Learning Notes: Feed & Routing" 
subtitle: "Keeping Feed States between views" 
date: 2017-08-17 
tags: [ios, rxswift, routing, mvvm] 
author: "Yeung Yiu Hung" 
comments: true
---

Favourite & Reblog
===

Before RxSwift, favourite and reblog in Mustor is done in following steps:

1. Start requesting API
2. Show Loading indictor on top of the button
3. Wait until server response, replace the item in feed
4. Post a notification to notify all the view / cells to update
5. Update View state (e.g. make favourite button selected, hide loading indicator)

The problem of this approach is when the user tap on favorite and reblog in short interval or server return error, the cell may not display correctly

Now I use MVVM and RxSwift, the steps become:

1. When user pressed favourite, it will trigger ```favoriteTrigger``` in View Model (RxSwift ```PublishSubject```)

```swift
// In view

self.favoriteButton.rx.tap.map{_ in}
    .asDriver(onErrorJustReturn: ())
    .drive(viewModel.favoriteTrigger)
    .disposed(by: disposeBag)
```

2. In View Model, when ```favoriteTrigger``` is trigger, toggle ```isFavourited```, and then call api to favourite the status

```swift
// In view model

let favoriteDisplay = isFavorited.asObservable()
    .sample(favoriteTrigger)
    .flatMap { value -> Observable<Bool> in
        var request: Observable<Bool>
        
        if !value == true {
            // Favourite API
            ...
            .catchErrorJustReturn(false) // When Error, isFavorited becomes false
        }else{
            // Unfavourite API
            ...
            .catchErrorJustReturn(true) // When Error, isFavorited becomes true
        }
        
        // Toggle Value first, then wait for API Response
        return Observable.concat(Observable<Bool>.just(!value), request)
    }

// Bind to isFavorited
favoriteDisplay
    .bind(to: isFavorited)
    .disposed(by: disposeBag)
```

3. Bind ```isFavorited``` to favourite button selected state

```swift
viewModel.isFavorited.asDriver()
        .drive(self.favoriteButton.rx.isSelected)
        .disposed(by: disposeBag)
```

4. Do the same to reblog button

After these implementation, the 2 button state will not mess up when user tap favorite and reblog withit a short interval. And the user can have instant response from ui (the cell is updated instantly when they tap the button) 

Routing
===

Before RxSwift, detail view will need delegate / notification center to notify the main feed to update if user pressed favourite or reblog button

After RxSwift + MVVM, the detail view with the same favorite and reblog button is much easy to implement:

1. When user select view model, bind the view model variable like ```isFavourite``` and the model to detail view model

```swift
// In Item View Model

func selected() {
    let detailViewModel = DetailViewModel()

    detailViewModel.isFavorited.asDriver()
        .drive(self.isFavorited)
        .disposed(by: disposeBag)

    detailViewModel.isReblogged.asDriver()
        .drive(self.isReblogged)
        .disposed(by: disposeBag)

    // Model 
    detailViewModel.contentStatus.asDriver()
        .drive(self.contentStatus)
        .disposed(by: disposeBag)
}

```

2. When favourited or reblogged in detail view model is update, the view model on the previous page will get update. The binding can go for multiple level, without ```NSNotificationCenter```