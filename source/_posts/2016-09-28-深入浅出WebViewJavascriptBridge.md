---
layout: post_layout
title: 深入浅出WebViewJavascriptBridge
date: 2016-09-28 00:00:00
location: 北京
pulished: true
abstract: 本文是笔者对WebViewJavascriptBridge的源码分析记录。
---

## 1 前言

[WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge)是iOS/OSX平台上支撑Obj-C和UIWebViews/WebViews JavaScript互发消息的库。目前主流App几乎都是某种程度的Hybrid App，该库因而得到广泛应用。

## 2 基础知识

在学习该库之前我们必须了解一些基础知识。主要包含前端和Native两大部分。

### 2.1 前端部分——HTML

Keypoint:

-  `<script>` 标签包裹的是JavaScript代码
- window、iframe
- setTimeout(0)

### 2.2 前端部分——JavaScript

Keypoint：

- JavaScript函数、对象

资料: 

- [JavaScript教程 w3school](http://www.w3school.com.cn/js/)
- [JavaScript高级程序设计(第3版)](http://product.china-pub.com/199113)
- [JavaScript权威指南(第6版)](http://product.china-pub.com/199271)

### 2.3 Native部分-关于UIWebView

```objc
__TVOS_PROHIBITED @protocol UIWebViewDelegate <NSObject>

@optional
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType;
- (void)webViewDidStartLoad:(UIWebView *)webView;
- (void)webViewDidFinishLoad:(UIWebView *)webView;
- (void)webView:(UIWebView *)webView didFailLoadWithError:(NSError *)error;

@end
```

UIWebView的代理UIWebViewDelegate，会在UIWebView各个事件节点收到回调消息。其中最重要的是`- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType`

当UIWebView加载URL page或者iframe设置src的时候，UIWebViewDelegate都会执行该回调。


## 3 WebViewJavascriptBridge设计

在分析WebViewJavascriptBridge源码之前，我们先聊一下WebViewJavascriptBridge设计。


### 3.1 整体框架

![图片](/assets/img/postImage/WebViewJavascriptBridge/1.png)

WebViewJavascriptBridge整体框架如上图所示。

包含4部分:

- 前端业务逻辑
- 前端js bridge基础设施
- Native js bridge基础设施
- Native业务逻辑

### 3.2 js bridge基础设施

总的来说js bridge基础设施主要由3部分组成：

1.  **消息流**: FE和Native之间的消息传递过程;
2. **消息体（message）**:message即Native和前端消息流中的消息体，主要有4个部分：函数名、参数、回调ID、响应ID;
3. **消息队列（FE message queue）**:前端消息队列用来暂存前端到Native的消息体。


#### 3.2.1 消息流

消息流如下图所示。

![图片](/assets/img/postImage/WebViewJavascriptBridge/2.png)

从图中我们可以看出消息流有两个参与者，即**调用方**和**被调方**。`调用方发起请求，收到对方的回调消息。被调方收到请求，执行请求，发送回调消息。`Native和FE都可能是调用方和被调用方，所以Native和FE都至少包含两部分功能：

- send（发送自己的调用请求到对端）
- receive（收到了来自对端的调用请求）

#### 3.2.2 消息体

消息体有四个成员：

1. 函数名
2. 参数
3. callbackID
4. responseID

其中函数名和参数都很好理解。这里我们主要说一下callbackID和responseID。

调用方在发起调用的同时设置`回调块`，该`回调块`在被调方执行完任务后再执行。具体的实现手段是，调用方在拼接消息体的时候，把`回调块`管理起来，并设置一个唯一的ID, 放到消息体的callbackID上面。 此时被调方收到的消息包含callbackID，在执行完成对应函数后，会生成一个应答消息，告知对方自己已经执行完成，这个应答消息也是一个消息体，该消息体的responseID设置为其所应答消息的callbackID，表示对该消息的应答。这时，调用方收到应答消息，检查responseID，匹配后找到之前对应的`回调块`并执行。

样例如图所示：
![图片](/assets/img/postImage/WebViewJavascriptBridge/3.jpg)


## 4 WebViewJavascriptBridge实现

### 4.1 消息流和消息队列实现

**消息流（前端到Native）**

前端到Native的消息流由隐藏的iframe发起。每次调用js bridge函数时设置iframe的src，然后，Native的`UIWebViewDelegate`收到`- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType`回调，在回调上加一些额外的逻辑区分，Native就知道前端发起了js bridge函数调用。

**消息流（Native到前端）**

Native到前端的消息流比较简单。它是由UIWebView本身完成。UIWebView的`- (nullable NSString *)stringByEvaluatingJavaScriptFromString:(NSString *)script`可以直接执行js命令。Native要调用前端的方法时，可以把方法转化为js命令直接调用。

**消息队列（FE message queue）**

前端消息队列用来暂存前端到Native的消息体。
相关的点如下：

- 前端设置iframe的src之前会先把消息存到消息队列；
- Native收到回调后，调用相关js命令从前端获取消息队列，得到消息队列后，按照消息队列的每条消息执行相应操作——函数调用。


### 4.2 前端js bridge源码

#### 4.2.1 send

参考前端到Native消息流。send通过iframe设置src和messageQueue缓存消息体实现。

```javascript
// 前端调用Native
function callHandler(handlerName, data, responseCallback) {
	if (arguments.length == 2 && typeof data == 'function') {
		responseCallback = data;
		data = null;
	}
	
	var message = { handlerName:handlerName, data:data };
	if (responseCallback) { // 回调管理
		var callbackId = 'cb_'+(uniqueId++)+'_'+new Date().getTime();
		responseCallbacks[callbackId] = responseCallback; 
		message['callbackId'] = callbackId;
	}
	sendMessageQueue.push(message);
	messagingIframe.src = 'wvjbscheme://__WVJB_QUEUE_MESSAGE__';
}

// 获取并清空message queue，暴露给OC
function _fetchQueue() {
	var messageQueueString = JSON.stringify(sendMessageQueue);
	sendMessageQueue = [];
	return messageQueueString;
}

```

#### 4.2.2 receive

参考Native到前端的消息流。receive通过registerHandler注册js bridge函数，通过_handleMessageFromObjC方法执行messageHandlers里面的函数体。

```javascript
// 前端注册js bridge方法供OC调用，比OC直接调用js普通方法好在对回调的支持上面。
var messageHandlers = {};
function registerHandler(handlerName, handler) {
	messageHandlers[handlerName] = handler; 
}
```

```javascript
// OC调用处理，暴露给OC
function _handleMessageFromObjC(messageJSON) {
	var message = JSON.parse(messageJSON);
	var messageHandler;
	var responseCallback;

	if (message.responseId) { // 回调管理（responseId匹配查找）=> message有responseId表示是一个回调调用
		responseCallback = responseCallbacks[message.responseId]; // 这个responseId必须要与当时消息寄送时所填写的responseId一致
		if (!responseCallback) {
			return;
		}
		responseCallback(message.responseData);
		delete responseCallbacks[message.responseId];
	} else {
		if (message.callbackId) { 
			var callbackResponseId = message.callbackId;
			responseCallback = function(responseData) {
			        var message = { handlerName:message.handlerName, responseId:callbackResponseId, responseData:responseData };
	                        sendMessageQueue.push(message);
	                        messagingIframe.src = 'wvjbscheme://__WVJB_QUEUE_MESSAGE__';
			};
		}
		
		// messageHandlers在这里
		var handler = messageHandlers[message.handlerName];
		if (!handler) {
			console.log("WebViewJavascriptBridge: WARNING: no handler for message from ObjC:", message);
		} else {
			handler(message.data, responseCallback);
		}
	}
}
```

### 4.3 Native js bridge源码

#### 4.3.1 send

参考Native到前端的消息流。Native的send是先拼接出js命令，再直接执行`stringByEvaluatingJavaScriptFromString`。

```objc
- (void)sendData:(id)data responseCallback:(WVJBResponseCallback)responseCallback handlerName:(NSString*)handlerName {
    NSMutableDictionary* message = [NSMutableDictionary dictionary];
    
    if (data) {
        message[@"data"] = data;
    }
    
    if (responseCallback) {
        NSString* callbackId = [NSString stringWithFormat:@"objc_cb_%ld", ++_uniqueId];
        self.responseCallbacks[callbackId] = [responseCallback copy];
        message[@"callbackId"] = callbackId;
    }
    
    if (handlerName) {
        message[@"handlerName"] = handlerName;
    }
    [self _queueMessage:message];
}
```

```objc
//命令拼接
- (void)_dispatchMessage:(WVJBMessage*)message {
    NSString *messageJSON = [self _serializeMessage:message pretty:NO];
    [self _log:@"SEND" json:messageJSON];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\\" withString:@"\\\\"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\"" withString:@"\\\""];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\'" withString:@"\\\'"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\n" withString:@"\\n"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\r" withString:@"\\r"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\f" withString:@"\\f"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\u2028" withString:@"\\u2028"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\u2029" withString:@"\\u2029"];
    
    NSString* javascriptCommand = [NSString stringWithFormat:@"WebViewJavascriptBridge._handleMessageFromObjC('%@');", messageJSON];
    if ([[NSThread currentThread] isMainThread]) {
        [self _evaluateJavascript:javascriptCommand];

    } else {
        dispatch_sync(dispatch_get_main_queue(), ^{
            [self _evaluateJavascript:javascriptCommand];
        });
    }
}
```

```objc
//命令执行
- (NSString*) _evaluateJavascript:(NSString*)javascriptCommand
{
    return [_webView stringByEvaluatingJavaScriptFromString:javascriptCommand];
}
```

#### 4.3.2 receive

参考FE到Native消息流。

```objc
//Native js bridge方法管理（给js用的）
- (void)registerHandler:(NSString *)handlerName handler:(WVJBHandler)handler {
    _base.messageHandlers[handlerName] = [handler copy];
}
```

```objc
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType {
    
    // core code {
    NSURL *url = [request URL];
    __strong WVJB_WEBVIEW_DELEGATE_TYPE* strongDelegate = _webViewDelegate;
    if ([_base isCorrectProcotocolScheme:url]) {
        if ([_base isQueueMessageURL:url]) {
            // 获取JS的messageQueue
            NSString *messageQueueString = [self _evaluateJavascript:[_base webViewJavascriptFetchQueyCommand]];
            [_base flushMessageQueue:messageQueueString];
        }
        return NO;
    }
    // core code }
  
}
```

```objc
-(NSString *)webViewJavascriptFetchQueyCommand {
    return @"WebViewJavascriptBridge._fetchQueue();";
}
```

```objc
- (void)flushMessageQueue:(NSString *)messageQueueString{
    if (messageQueueString == nil || messageQueueString.length == 0) {
        NSLog(@"WebViewJavascriptBridge: WARNING: ObjC got nil while fetching the message queue JSON from webview. This can happen if the WebViewJavascriptBridge JS is not currently present in the webview, e.g if the webview just loaded a new page.");
        return;
    }

    // 拿到消息，按照消息handler
    id messages = [self _deserializeMessageJSON:messageQueueString];
    for (WVJBMessage* message in messages) {
        if (![message isKindOfClass:[WVJBMessage class]]) {
            NSLog(@"WebViewJavascriptBridge: WARNING: Invalid %@ received: %@", [message class], message);
            continue;
        }
        [self _log:@"RCVD" json:message];
        
        NSString* responseId = message[@"responseId"];
        if (responseId) { // 如果是应答消息则执行并结束
            WVJBResponseCallback responseCallback = _responseCallbacks[responseId];
            responseCallback(message[@"responseData"]);
            [self.responseCallbacks removeObjectForKey:responseId];
        } else { // 如果是普通调用消息，则根据是否需要对其应答做相应处理
            WVJBResponseCallback responseCallback = NULL;
            NSString* callbackId = message[@"callbackId"];
            if (callbackId) {
                responseCallback = ^(id responseData) {
                    if (responseData == nil) {
                        responseData = [NSNull null];
                    }
                    
                    WVJBMessage* msg = @{ @"responseId":callbackId, @"responseData":responseData };
                    [self _queueMessage:msg];
                };
            } else {
                responseCallback = ^(id ignoreResponseData) {
                    // Do nothing
                };
            }
            
            // 在messageHandlers里面查找handler
            WVJBHandler handler = self.messageHandlers[message[@"handlerName"]];
            
            if (!handler) {
                NSLog(@"WVJBNoHandlerException, No handler for message from JS: %@", message);
                continue;
            }
            // 执行handler
            handler(message[@"data"], responseCallback);
        }
    }
}
```

## 5总结

本文从JS bridge的基础知识讲到WebViewJavascriptBridge的源码实现。涉及的点有消息流，消息体，消息队列等。其中比较有意思的是回调实现原理。算是对自己阅读代码的一个记录。
