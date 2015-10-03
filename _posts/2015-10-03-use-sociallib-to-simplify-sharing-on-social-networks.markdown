---

layout: post 
title: 	"My first library: SocialLib" 
subtitle: "So many delegates, so much pain" 
date: 2015-10-03 14:36:00 
tags: [ios, social network, library] 
author: "Yeung Yiu Hung" 
comments: true
---

So... I finally get a new project to work on (YAY!). It contains a news feed and product catalogue. One of the project requirement is to share posts on social networks (Facebook, Instagram, Weibo and WeChat). At first, I think this is not a big deal since I have implemented them before.

Then I start my development and see the problem of multiple social network sdk. For facebook, you can set delegate of share dialogue. But for Weibo, you set delegate in handleURL.

The original AppDelegate look like this:

{% highlight objective-c %}
//AppDelegate.h

#import <UIKit/UIKit.h>

//Will use in my Share View Controller
@protocol WeiboShareDelegate <NSObject>

- (void)didFinishSharingWeiboMessage;
- (void)didFailSharingWeiboMessageWithError:(NSError *)error;

@end

@interface AppDelegate : UIResponder <UIApplicationDelegate, WeiboSDKDelegate, WXApiDelegate>{
}

@property (strong, nonatomic) UIWindow *window;

//Storing Weibo Access Token
@property (nonatomic, strong) NSString* wbtoken;
@property (nonatomic, strong) NSString* wbCurrentUserID;

@property (nonatomic, assign) id<WeiboShareDelegate> weiboDelegate;
{% endhighlight %}

{% highlight objective-c %}
//AppDelegate.m
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Override point for customization after application launch.
    [WXApi registerApp:kWeChatAppKey withDescription:@"..."];
    [WeiboSDK enableDebugMode:YES];
    [WeiboSDK registerApp:kWeiboAppKey];
    
    /* You cannot see me :) */
    
    return return [[FBSDKApplicationDelegate sharedInstance] application:application didFinishLaunchingWithOptions:launchOptions];
}

- (void)applicationDidBecomeActive:(UIApplication *)application {
    // Restart any tasks that were paused (or not yet started) while the application was inactive. If the application was previously in the background, optionally refresh the user interface.
    [FBSDKAppEvents activateApp];
}

- (BOOL)application:(UIApplication *)application
            openURL:(NSURL *)url
  sourceApplication:(NSString *)sourceApplication
         annotation:(id)annotation {
    NSString *myString = [url absoluteString];
    
    if ([myString containString:kWeChatAppKey]) {
        return [WXApi handleOpenURL:url delegate:self];
    }else if([myString containString:kFBAppKey]){
        return [[FBSDKApplicationDelegate sharedInstance] application:application
                                                              openURL:url
                                                    sourceApplication:sourceApplication
                                                           annotation:annotation];
    }else{
        return [WeiboSDK handleOpenURL:url delegate:self];
    }
}

#pragma mark - Weibo Delegate

- (void)didReceiveWeiboRequest:(WBBaseRequest *)request
{
}


- (void)didReceiveWeiboResponse:(WBBaseResponse *)response
{
    if ([response isKindOfClass:WBSendMessageToWeiboResponse.class])
    {
        WBSendMessageToWeiboResponse* sendMessageToWeiboResponse = (WBSendMessageToWeiboResponse*)response;
        NSString* accessToken = [sendMessageToWeiboResponse.authResponse accessToken];
        if (accessToken)
        {
            self.wbtoken = accessToken;
        }
        NSString* userID = [sendMessageToWeiboResponse.authResponse userID];
        if (userID) {
            self.wbCurrentUserID = userID;
        }
        if(response.statusCode == 0){
            if (_weiboDelegate && [_weiboDelegate respondsToSelector:@selector(didFinishSharingWeiboMessage)]) {
                [_weiboDelegate didFinishSharingWeiboMessage];
            }
        }else{
            if (_weiboDelegate && [_weiboDelegate respondsToSelector:@selector(didFailSharingWeiboMessageWithError:)]) {
                [_weiboDelegate didFailSharingWeiboMessageWithError:nil];
            }
        }
    }
    else if ([response isKindOfClass:WBAuthorizeResponse.class])
    {
        self.wbtoken = [(WBAuthorizeResponse *)response accessToken];
        self.wbCurrentUserID = [(WBAuthorizeResponse *)response userID];
    }
    else if ([response isKindOfClass:WBPaymentResponse.class])
    {
        
    }
}
{% endhighlight %}

So far so good, next is my share view controller. The view controller is the problem started.

{% highlight objective-c %}
//
//  SocialMediaShareViewController.m
//

#import "SocialMediaShareViewController.h"

#import "SocialMediaTableViewCell.h"

NSInteger const kFacebookRow = 0;
NSInteger const kWeChatRow = 3;
NSInteger const kWeiboRow = 2;
NSInteger const kInstagramRow = 1;


@interface SocialMediaShareViewController ()

@end

@implementation SocialMediaShareViewController

/* You cannot see me :) */

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    /*
    shareInfo is the information to share, it contains 4 social media objects for each social media
    */
    switch (indexPath.row) {
        case kFacebookRow:{
            FBSDKShareLinkContent *content = [[FBSDKShareLinkContent alloc] init];
            content.contentDescription = shareInfo.facebookSharingInfo.sharingDescription;
            content.contentURL = [NSURL URLWithString:shareInfo.facebookSharingInfo.sharingURL];
            content.imageURL = [NSURL URLWithString:shareInfo.facebookSharingInfo.sharingImageURL];
            
            [FBSDKShareDialog showFromViewController:self
                                         withContent:content
                                            delegate:self];
            
        }
            break;
        case kInstagramRow:{
            NSURL *instagramURL = [NSURL URLWithString:@"instagram://app"];
            if ([[UIApplication sharedApplication] canOpenURL:instagramURL]) {
                AppDelegate *appDelegate = (AppDelegate *)[[UIApplication sharedApplication] delegate];
                [appDelegate showHUD];
                [SDWebImageManager.sharedManager downloadImageWithURL:[NSURL URLWithString:shareInfo.instagramSharingInfo.sharingImageURL]
                                                              options:SDWebImageDownloaderHighPriority progress:^(NSInteger receivedSize, NSInteger expectedSize) {
                                                                  
                                                              }
                                                            completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
                                                                [appDelegate hideHUD];
                                                                if (error) {
                                                                    if (_delegate && [_delegate respondsToSelector:@selector(socialMediaDidFailedSharing:)]) {
                                                                        [_delegate socialMediaDidFailedSharing:error];
                                                                    }
                                                                }else{
                                                                    // Create path.
                                                                    NSString* imagePath = [NSString stringWithFormat:@"%@/instagramShare.igo", [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) lastObject]];
                                                                    [[NSFileManager defaultManager] removeItemAtPath:imagePath error:nil];
                                                                    
                                                                    [UIImagePNGRepresentation(image) writeToFile:imagePath atomically:YES];
                                                                    NSLog(@"Image Size >>> %@", NSStringFromCGSize(image.size));
                                                                    
                                                                    self.dic=[UIDocumentInteractionController interactionControllerWithURL:[NSURL fileURLWithPath:imagePath]];
                                                                    self.dic.delegate = self;
                                                                    self.dic.UTI = @"com.instagram.exclusivegram";
                                                                    self.dic.annotation = @{@"InstagramCaption": shareInfo.instagramSharingInfo.sharingDescription};
                                                                    UIViewController *controller = [UIApplication sharedApplication].keyWindow.rootViewController;
                                                                    
                                                                    [self.dic presentOpenInMenuFromRect:controller.view.frame inView:controller.view animated:YES];
                                                                    
                                                                    if (_delegate && [_delegate respondsToSelector:@selector(socialMediaDidFinishSharing)]) {
                                                                        [_delegate socialMediaDidFinishSharing];
                                                                    }
                                                                }
                                                            }];
                
            }else{
                if (_delegate && [_delegate respondsToSelector:@selector(socialMediaDidFailedSharing:)]) {
                    NSError *error = [NSError errorWithDomain:[SocialMediaShareViewController errorDomain]
                                                         code:0
                                                     userInfo:@{NSLocalizedDescriptionKey : AMLocalizedString(@"no_instagram", nil)}];
                    [_delegate socialMediaDidFailedSharing:error];
                    [tableView deselectRowAtIndexPath:indexPath animated:NO];
                }
            }
        }
            break;
        case kWeChatRow:{
            if (![WXApi isWXAppInstalled]) {
                if (_delegate && [_delegate respondsToSelector:@selector(socialMediaDidFailedSharing:)]) {
                    NSError *error = [NSError errorWithDomain:[SocialMediaShareViewController errorDomain]
                                                         code:0
                                                     userInfo:@{NSLocalizedDescriptionKey : AMLocalizedString(@"no_wechat", nil)}];
                    [_delegate socialMediaDidFailedSharing:error];
                    [tableView deselectRowAtIndexPath:indexPath animated:NO];
                }
            }else{
                AppDelegate *appDelegate = (AppDelegate *)[[UIApplication sharedApplication] delegate];
                [appDelegate showHUD];
                [SDWebImageManager.sharedManager downloadImageWithURL:[NSURL URLWithString:shareInfo.instagramSharingInfo.sharingImageURL]
                                                              options:SDWebImageDownloaderHighPriority progress:^(NSInteger receivedSize, NSInteger expectedSize) {
                                                                  
                                                              }
                                                            completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
                                                                [appDelegate hideHUD];
                                                                if (error) {
                                                                    if (_delegate && [_delegate respondsToSelector:@selector(socialMediaDidFailedSharing:)]) {
                                                                        [_delegate socialMediaDidFailedSharing:error];
                                                                    }
                                                                }else{
                                                                    WXMediaMessage *message = [WXMediaMessage message];
                                                                    UIImage *aImage = image;
                                                                    UIImage *thumbImage = [aImage scaleToSize:CGSizeMake(200,200)];
                                                                    
                                                                    message.title = shareInfo.wechatSharingInfo.sharingDescription;
                                                                    [message setThumbImage:thumbImage];
                                                                    
                                                                    WXWebpageObject *ext = [WXWebpageObject object];
                                                                    ext.webpageUrl = shareInfo.wechatSharingInfo.sharingURL;
                                                                    message.mediaObject = ext;
                                                                    
                                                                    SendMessageToWXReq* req = [[SendMessageToWXReq alloc] init];
                                                                    req.bText = NO;
                                                                    req.message = message;
                                                                    req.scene = WXSceneTimeline;
                                                                    BOOL success = [WXApi sendReq:req];
                                                                    if(success){
                                                                        if (_delegate && [_delegate respondsToSelector:@selector(socialMediaDidFinishSharing)]) {
                                                                            [_tableView deselectRowAtIndexPath:[NSIndexPath indexPathForItem:kFacebookRow inSection:0] animated:NO];
                                                                            [_delegate socialMediaDidFinishSharing];
                                                                        }
                                                                    }else{
                                                                        if (_delegate && [_delegate respondsToSelector:@selector(socialMediaDidFailedSharing:)]) {
                                                                            NSError *error = [NSError errorWithDomain:[SocialMediaShareViewController errorDomain]
                                                                                                                 code:0
                                                                                                             userInfo:@{NSLocalizedDescriptionKey : AMLocalizedString(@"wechat_share_error", nil)}];
                                                                            [_delegate socialMediaDidFailedSharing:error];
                                                                        }
                                                                    }
                                                                }
                                                            }];
            }
        }
            break;
        case kWeiboRow:{
            AppDelegate *appDelegate = (AppDelegate *)[[UIApplication sharedApplication] delegate];
            [appDelegate showHUD];
            [SDWebImageManager.sharedManager downloadImageWithURL:[NSURL URLWithString:shareInfo.instagramSharingInfo.sharingImageURL]
                                                          options:SDWebImageDownloaderHighPriority
                                                         progress:^(NSInteger receivedSize, NSInteger expectedSize) {
                                                              
                                                          }
                                                        completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
                                                            [appDelegate hideHUD];
                                                            if (error) {
                                                                if (_delegate && [_delegate respondsToSelector:@selector(socialMediaDidFailedSharing:)]) {
                                                                    [_delegate socialMediaDidFailedSharing:error];
                                                                }
                                                            }else{
                                                                // Create path.
                                                                if ([WeiboSDK isWeiboAppInstalled]) {
                                                                    WBAuthorizeRequest *authRequest = [WBAuthorizeRequest request];
                                                                    authRequest.redirectURI = kRedirectURI;
                                                                    authRequest.scope = @"all";
                                                                    
                                                                    WBSendMessageToWeiboRequest *request = [WBSendMessageToWeiboRequest requestWithMessage:[self messageToShare:image] authInfo:authRequest access_token:appDelegate.wbtoken];
                                                                    
                                                                    //    request.shouldOpenWeiboAppInstallPageIfNotInstalled = NO;
                                                                    [WeiboSDK sendRequest:request];

                                                                }else{
                                                                    if (_delegate && [_delegate respondsToSelector:@selector(socialMediaDidCancelSharing)]) {
                                                                        pending = YES;
                                                                        [_delegate socialMediaDidCancelSharing];
                                                                    }
                                                                    WBAuthorizeRequest *authRequest = [WBAuthorizeRequest request];
                                                                    authRequest.redirectURI = kRedirectURI;
                                                                    authRequest.scope = @"all";
                                                                    
                                                                    WBSendMessageToWeiboRequest *request = [WBSendMessageToWeiboRequest requestWithMessage:[self messageToShare:image] authInfo:authRequest access_token:appDelegate.wbtoken];
                                                                    request.userInfo = @{@"ShareMessageFrom": @"SocialMediaShareViewController",
                                                                                         @"Other_Info_1": [NSNumber numberWithInt:123],
                                                                                         @"Other_Info_2": @[@"obj1", @"obj2"],
                                                                                         @"Other_Info_3": @{@"key1": @"obj1", @"key2": @"obj2"}};
                                                                    //    request.shouldOpenWeiboAppInstallPageIfNotInstalled = NO;
                                                                    [WeiboSDK sendRequest:request];
                                                                }
                                                            }
                                                        }];
            
        }
            break;
        default:
            break;
    }
}

#pragma mark -
#pragma WeiboShareDelegate

- (void)didFinishSharingWeiboMessage{
    if (_delegate && [_delegate respondsToSelector:@selector(socialMediaDidFinishSharing)]) {
        pending = NO;
        [_delegate socialMediaDidFinishSharing];
    }
}

- (void)didFailSharingWeiboMessageWithError:(NSError *)error{
    if (_delegate && [_delegate respondsToSelector:@selector(socialMediaDidFailedSharing:)]) {
        pending = NO;
        [_delegate socialMediaDidFailedSharing:error];
    }
}

#pragma mark -
#pragma Internal Method

- (WBMessageObject *)messageToShare:(UIImage *)weiboImage
{
    WBMessageObject *message = [WBMessageObject message];
    
    WBImageObject *image = [WBImageObject object];
    message.text = shareInfo.wechatSharingInfo.sharingDescription;
    image.imageData = UIImagePNGRepresentation(weiboImage);
    message.imageObject = image;
    return message;
}

#pragma mark - FBSDKSaringDelegate
- (void)sharer:(id<FBSDKSharing>)sharer didCompleteWithResults:(NSDictionary *)results{
    if (_delegate && [_delegate respondsToSelector:@selector(socialMediaDidFinishSharing)]) {
        [_tableView deselectRowAtIndexPath:[NSIndexPath indexPathForItem:kFacebookRow inSection:0] animated:NO];
        [_delegate socialMediaDidFinishSharing];
    }
}

- (void)sharer:(id<FBSDKSharing>)sharer didFailWithError:(NSError *)error{
    if (_delegate && [_delegate respondsToSelector:@selector(socialMediaDidFailedSharing:)]) {
        [_tableView deselectRowAtIndexPath:[NSIndexPath indexPathForItem:kFacebookRow inSection:0] animated:NO];
        [_delegate socialMediaDidFailedSharing:error];
    }
}

- (void)sharerDidCancel:(id<FBSDKSharing>)sharer{
    if (_delegate && [_delegate respondsToSelector:@selector(socialMediaDidCancelSharing)]) {
        [_tableView deselectRowAtIndexPath:[NSIndexPath indexPathForItem:kFacebookRow inSection:0] animated:NO];
        [_delegate socialMediaDidCancelSharing];
    }
}
{% endhighlight %}

Now you see how chaos it is to have multiple social network, each of them have different delegate, different object to allocate, and I have to implement a "proxy" delegate that may break, and I have to make a useless boolean to check wheather I can set Weibo delegate to my AppDelegate.

I start googling <s>becuase I am lazy</s> and I found an interesting library: [OpenShare](https://github.com/100apps/openshare), this library can share to weibo, wechat, renren and qq, with same object with one function call: 
{% highlight objective-c %}
[OpenShare shareToQQFriends:msg Success:^(OSMessage *message) {
    ULog(@"分享到QQ好友成功:%@",msg);
} Fail:^(OSMessage *message, NSError *error) {
    ULog(@"分享到QQ好友失败:%@\n%@",msg,error);
}];
{% endhighlight %}

It is exactly what I want, clean and simple, no delegates flying everywhere, YAY!

Then I realize there is not that simple, this library's aims is NOT to install any social network sdk so the app cannot share any post if the user does not install any social network client application. 

Why not do it myself?

I started a library project called [SocialLib](http://darkcl.github.io/SocialLib) which aims ONLY to share post on social media, no delegates, just one function call with a success block and a failure block. Also, it share any objects conforms with respective social media protocol.

The sharing looks like this now:
{% highlight objective-c %}
//SocialMediaShareViewController.m

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath{
    NSString *platform;
    switch (indexPath.row) {
        case kFacebookRow:{
            platform = kSocialLibPlatformFacebook;
        }
            break;
        case kInstagramRow:{
            platform = kSocialLibPlatformInstagram;
        }
            break;
        case kWeChatRow:{
            platform = kSocialLibPlatformWeixin;
        }
            break;
        case kWeiboRow:{
           platform = kSocialLibPlatformWeibo;
        }
            break;
        default:
            break;
    }
    
    [SocialLib shareModal:shareInfo
               toPlatform:platform
                  success:^(NSDictionary *message) {
                      if (_delegate && [_delegate respondsToSelector:@selector(socialMediaDidFinishSharing)]) {
                          [_delegate socialMediaDidFinishSharing];
                      }
                  }
                  failure:^(NSDictionary *message, NSError *error) {
                      if (_delegate && [_delegate respondsToSelector:@selector(socialMediaDidFailedSharing:)]) {
                          [_delegate socialMediaDidFailedSharing:error];
                      }
                  }];
}
{% endhighlight %}

And the AppDelegate become cleaner, too
{% highlight objective-c %}
//  AppDelegate.h

#import <UIKit/UIKit.h>

@interface AppDelegate : UIResponder <UIApplicationDelegate>{
    UIAlertView *updatePrompt;
    bool needUpdatePrompt;
}

@property (strong, nonatomic) UIWindow *window;

@end
{% endhighlight %}

{% highlight objective-c %}
// AppDelegate.m
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    // Override point for customization after application launch.
    
    /* skip! */
    
    return [SocialLib connectSocialPlatformWithApplication:application didFinishLaunchingWithOptions:launchOptions];
}

- (void)applicationDidBecomeActive:(UIApplication *)application
{
    [SocialLib applicationDidBecomeActie:application];
}

- (BOOL)application:(UIApplication *)application
            openURL:(NSURL *)url
  sourceApplication:(NSString *)sourceApplication
         annotation:(id)annotation {
    return [SocialLib handleOpenURL:application
                            openURL:url
                  sourceApplication:sourceApplication
                         annotation:annotation];
}
{% endhighlight %}

***[SocialLib](https://github.com/darkcl/SocialLib) is in early development, issue and pull request are welcome.**