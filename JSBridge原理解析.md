## JSBridge原理解析——以WebviewJavascriptBridge实现方式为例

## 一、什么是 JSBridge？

JSBridge 是一种 webview 侧和 native 侧进行通信的手段，webview 可以通过 jsb 调用 native 的能力，native 也可以通过 jsb 在 webview 上执行一些逻辑。

## 二、JSB 的实现方式

在比较流行的 JSBridge 中，主要是通过拦截 URL 请求来达到 native 端和 webview 端相互通信的效果的。

这里我们以比较火的 WebviewJavascriptBridge 为例，来解析一下它的实现方式。

源码地址：https://github.com/marcuswestin/WebViewJavascriptBridge

### 2-1、在 native 端和 webview 端注册 Bridge

注册的时候，需要在 webview 侧和 native 侧分别注册 bridge，其实就是用一个对象把所有函数储存起来。

```
function registerHandler(handlerName, handler) {
    messageHandlers[handlerName] = handler;
}
- (void)registerHandler:(NSString *)handlerName handler:(WVJBHandler)handler {
    _base.messageHandlers[handlerName] = [handler copy];
}
```

### 2-2、在 webview 里面注入初始化代码

```
function setupWebViewJavascriptBridge(callback) {
       if (window.WebViewJavascriptBridge) { return callback(WebViewJavascriptBridge); }
       if (window.WVJBCallbacks) { return window.WVJBCallbacks.push(callback); }
       window.WVJBCallbacks = [callback];
       var WVJBIframe = document.createElement('iframe');
       WVJBIframe.style.display = 'none';
       WVJBIframe.src = 'https://__bridge_loaded__';
       document.documentElement.appendChild(WVJBIframe);
       setTimeout(function() { document.documentElement.removeChild(WVJBIframe) }, 0)
}
```

这段代码主要做了以下几件事：

（1）创建一个名为 WVJBCallbacks 的数组，将传入的 callback 参数放到数组内

（2）创建一个 iframe，设置不可见，设置 src 为`https://__bridge_loaded__`

（3）设置定时器移除这个 iframe

### 2-3、在 native 端监听 URL 请求

iOS 中有两种 webview，一种是 UIWebview，另一种是 WKWebview，这里以 WKWebview 为例：

```
- (void)webView:(WKWebView *)webView decidePolicyForNavigationResponse:(WKNavigationResponse *)navigationResponse decisionHandler:(void (^)(WKNavigationResponsePolicy))decisionHandler {
    if (webView != _webView) { return; }

    __strong typeof(_webViewDelegate) strongDelegate = _webViewDelegate;
    if (strongDelegate && [strongDelegate respondsToSelector:@selector(webView:decidePolicyForNavigationResponse:decisionHandler:)]) {
        [strongDelegate webView:webView decidePolicyForNavigationResponse:navigationResponse decisionHandler:decisionHandler];
    }
    else {
        decisionHandler(WKNavigationResponsePolicyAllow);
    }
}
```

这段代码主要做了以下几件事：

（1）拦截了所有的 URL 请求并拿到 url

（2）首先判断`isWebViewJavascriptBridgeURL`，判断这个 url 是不是 webview 的 iframe 触发的，具体可以通过 host 去判断。

（3）继续判断，如果是`isBridgeLoadedURL`，那么会执行`injectJavascriptFile`方法，会向 webview 中再次注入一些逻辑，其中最重要的逻辑就是，在 window 对象上挂载一些全局变量和`WebViewJavascriptBridge`属性，具体值如下：

```
window.WebViewJavascriptBridge = {
        registerHandler: registerHandler,
        callHandler: callHandler,
        disableJavscriptAlertBoxSafetyTimeout: disableJavscriptAlertBoxSafetyTimeout,
        _fetchQueue: _fetchQueue,
        _handleMessageFromObjC: _handleMessageFromObjC
};

var sendMessageQueue = [];
var messageHandlers = {};

var responseCallbacks = {};
var uniqueId = 1;
```

（4）继续判断，如果是 isQueueMessageURL，那么这就是个处理消息的回调，需要执行一些消息处理的方法（第四步会详细讲）

### 2-4、webview 调用 native 能力

当 native 和 webview 都注册好了 Bridge 之后，双方就可以互相调用了，这里先介绍 webview 调用 native 能力的过程。

#### 2-4-1、webview 侧 callHandler

当 webview 调用 native 时，会调用 callHandler 方法，这个方法具体逻辑如下：

```
bridge.callHandler('ObjC Echo', {'key':'value'}, function responseCallback(responseData) {
       console.log("JS received response:", responseData)
})

function callHandler(handlerName, data, responseCallback) {
        if (arguments.length == 2 && typeof data == 'function') {
                responseCallback = data;
                data = null;
         }
        _doSend({ handlerName:handlerName, data:data }, responseCallback);
}

function _doSend(message, responseCallback) {
        if (responseCallback) {
               var callbackId = 'cb_'+(uniqueId++)+'_'+new Date().getTime();
               responseCallbacks[callbackId] = responseCallback;
               message['callbackId'] = callbackId;
         }
         sendMessageQueue.push(message);
         messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
}
```

实际上就是先生成一个 message，然后 push 到 sendMessageQueue 里，然后更改 iframe 的 src。

#### 2-4-2、native 侧 flushMessageQueue

然后，当 native 端检测到 iframe src 的变化时，会走到 isQueueMessageURL 的判断逻辑，然后执行 WKFlushMessageQueue 函数，获取到 JS 侧的 sendMessageQueue 中的所有 message。

```
- (void)WKFlushMessageQueue {
    [_webView evaluateJavaScript:[_base webViewJavascriptFetchQueyCommand] completionHandler:^(NSString* result, NSError* error) {
        if (error != nil) {
            NSLog(@"WebViewJavascriptBridge: WARNING: Error when trying to fetch data from WKWebView: %@", error);
        }
        [_base flushMessageQueue:result];
    }];
}

- (void)flushMessageQueue:(NSString *)messageQueueString{
    if (messageQueueString == nil || messageQueueString.length == 0) {
        NSLog(@"WebViewJavascriptBridge: WARNING: ObjC got nil while fetching the message queue JSON from webview. This can happen if the WebViewJavascriptBridge JS is not currently present in the webview, e.g if the webview just loaded a new page.");
        return;
    }

    id messages = [self _deserializeMessageJSON:messageQueueString];
    for (WVJBMessage* message in messages) {
        if (![message isKindOfClass:[WVJBMessage class]]) {
            NSLog(@"WebViewJavascriptBridge: WARNING: Invalid %@ received: %@", [message class], message);
            continue;
        }
        [self _log:@"RCVD" json:message];

        NSString* responseId = message[@"responseId"];
        if (responseId) {
            WVJBResponseCallback responseCallback = _responseCallbacks[responseId];
            responseCallback(message[@"responseData"]);
            [self.responseCallbacks removeObjectForKey:responseId];
        } else {
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

            WVJBHandler handler = self.messageHandlers[message[@"handlerName"]];

            if (!handler) {
                NSLog(@"WVJBNoHandlerException, No handler for message from JS: %@", message);
                continue;
            }

            handler(message[@"data"], responseCallback);
        }
    }
}
```

当一个 message 结构存在 responseId 的时候说明这个 message 是执行 bridge 后传回的。取不到 responseId 说明是第一次调用 bridge 传过来的，这个时候会生成一个返回给调用方的 message，其 reponseId 是传过来的 message 的 callbackId，当 native 执行 responseCallback 时，会触发_dispatchMessage 方法执行 webview 环境的的 js 逻辑，将生成的包含 responseId 的 message 返回给 webview。

#### 2-4-3、webview 侧 handleMessageFromObjC

```
function _handleMessageFromObjC(messageJSON) {
    _dispatchMessageFromObjC(messageJSON);
}

function _dispatchMessageFromObjC(messageJSON) {
       if (dispatchMessagesWithTimeoutSafety) {
             setTimeout(_doDispatchMessageFromObjC);
 } else {
             _doDispatchMessageFromObjC();
 }

 function _doDispatchMessageFromObjC() {
        var message = JSON.parse(messageJSON);
        var messageHandler;
        var responseCallback;
        if (message.responseId) {
                responseCallback = responseCallbacks[message.responseId];
                if (!responseCallback) {
                          return;
                }
                responseCallback(message.responseData);
                delete responseCallbacks[message.responseId];
         } else {
               if (message.callbackId) {
                       var callbackResponseId = message.callbackId;
                       responseCallback = function(responseData) {
                             _doSend({ handlerName:message.handlerName, responseId:callbackResponseId, responseData:responseData });
                        };
                }

                var handler = messageHandlers[message.handlerName];
                if (!handler) {
                        console.log("WebViewJavascriptBridge: WARNING: no handler for message from ObjC:", message);
                } else {
                        handler(message.data, responseCallback);
                }
          }
     }
}
```

如果从 native 获取到的 message 中有 responseId，说明这个 message 是 JS 调 Native 之后回调接收的 message，所以从一开始 sendData 中添加的 responseCallbacks 中根据 responseId（一开始存的时候是用的 callbackId，两个值是相同的）取出这个回调函数并执行，这样就完成了一次 JS 调用 Native 的流程。

#### 2-4-4、过程总结

过程如下图

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

1、native 端注册 jsb

2、webview 侧创建 iframe，设置 src 为`__bridge_load__`

3、native 端捕获请求，注入 jsb 初始化代码，在 window 上挂载相关对象和方法

4、webview 侧调用`callHandler`方法，并在`responseCallback`上添加`callbackId: responseCallback`，并修改 iframe 的 src，触发捕获

5、native 收到 message，生成一个`responseCallback`，并执行 native 侧注册好的方法

6、native 执行完毕后，通过 webview 执行`_handleMessageFromObjC`方法，取出 callback 函数，并执行

### 2-5、native 调用 webview 能力

native 调用 webview 注册的 jsb 的逻辑是相似的，不过就不是通过触发 iframe 的 src 触发执行的了，因为 Native 可以自己主动调用 JS 侧的方法。其具体过程如下图：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

1、native 侧调用`callHandler`方法，并在`responseCallback`上添加`callbackId: responseCallback`

2、native 侧主动调用`_handleMessageFromObjC`方法，在 webview 中执行对应的逻辑

3、webview 侧执行结束后，生成带有`responseId`的 message，添加到`sendMessageQueue`中，并修改 iframe 的 src 为`__wvjb_queue_message__`

4、native 端拦截到 url 变化，调用 webview 的逻辑获取到 message，拿到`responseId`，并执行对应的 callback 函数

