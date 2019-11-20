### 0.web push 是怎么工作的？
Service Worker 理解为一个介于客户端和服务器之间的一个代理服务器。在 Service Worker 中我们可以做很多事情，比如拦截客户端的请求、向客户端发送消息、向服务器发起请求等等，其中最重要的作用之一就是离线资源缓存。

后台同步应该算是Service Worker相关功能（API）中比较易于理解与使用的一个。
流程如下：
![](https://user-gold-cdn.xitu.io/2018/5/13/1635905056b125a7?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
首先，你需要在Service Worker中监听sync事件；
然后，在浏览器中发起后台同步sync（图中第一步）；
之后，会触发Service Worker的sync事件，在该监听的回调中进行操作，例如向后端发起请求（图中第二步）
最后，可以在Service Worker中对服务端返回的数据进行处理。
由于Service Worker在用户关闭该网站后仍可以运行，因此该流程名为“后台同步”实在是非常贴切。

### 1.发送Push过程
给用户浏览器或者客户端发送一个Push，这个过程是这样的：

![](https://pic3.zhimg.com/80/v2-65be84459285fef5ad0703c578692c1e_hd.jpg)

在浏览器端，注册一个Service Worker之后会返回一个注册的对象，调这个对象的pushManager.subscribe的方法让浏览器弹一个框，询问用户是否允许接受消息通知：

![](https://pic4.zhimg.com/80/v2-64b0886708c84ec8f15d639c58345407_hd.jpg)
![](https://pic4.zhimg.com/80/v2-64b0886708c84ec8f15d639c58345407_hd.jpg)

如果点击允许的话，浏览器就会向FCM请求生成一个subscription（订阅）的标志信息，然后把这个subscription发给服务端存起来，用来发Push给当前用户。服务端使用这个subscription的信息调web push提供的API向FCM发送消息，FCM再下发给对应的浏览器。然后浏览器会触发Service Worker的push事件，让Service Worker调showNotification显示这个push的内容。操作系统就会显示这个Push：

（1）浏览器发起询问，生成subscription
在注册完service worker后，调用subscribe询问用户是否允许接收通知，如下代码所示：
```
function subscribeUser(swRegistration) {
    const applicationServerPublicKey = "BBlY_5OeDkp2zl_Hx9jFxymKyK4kQKZdzoCoe0L5RqpiV2eK0t4zx-d3JPHlISZ0P1nQdSZsxuA5SRlDB0MZWLw";
    const applicationServerKey = urlB64ToUint8Array(applicationServerPublicKey);
    swRegistration.pushManager.subscribe({
        userVisibleOnly: true,
        applicationServerKey: applicationServerKey
    })
    // 用户同意
    .then(function(subscription) {
        console.log('User is subscribed:', JSON.stringify(subscription));
        jQuery.post("/add-subscription.php", {subscription: JSON.stringify(subscription)}, function(result) {
            console.log(result);
        });
    })
    // 用户不同意或者生成失败
    .catch(function(err) {
        console.log('Failed to subscribe the user: ', err);
    });
}

navigator.serviceWorker.register("sw-4.js").then(function(reg){
    console.log("Yes, it did register service worker.");
    if (window.PushManager) {
        reg.pushManager.getSubscription().then(subscription => {
            // 如果用户没有订阅
            if (!subscription) {
                subscribeUser(reg);
            } else {
                console.log("You have subscribed our notification");
            }
        });
    }
}).catch(function(err) {
    console.log("No it didn't. This happened: ", err)
});
```
subscribe传两个参数，
一个是userVisibleOnly，这个表示消息是否必须要可见，如果设置为不可见，Chrome将会报错：
第二个参数applicationServerKey是服务端的公钥，这个可以用web push的Node包生成，先安装一个：
公密钥只要能配套就好，公钥在浏览器端使用，用来生成subscription，密钥在服务端使用，用来发Push。

如果用户同意浏览器就会向FCM服务请求生成subscription，然后执行Promise链里的then，返回该subscription，在这个then里面把这个subscription发给服务端存起来。反之，如果用户不同意，或者用户无法连到FCM的服务器将会抛异常：

```
{
    "endpoint":"https://fcm.googleapis.com/fcm/send/ci3-kIulf9A:APA91bEaQfDU8zuLSKpjzLfQ8121pNf3Rq7pjomSu4Vg-nMwLGfJSvkOUsJNCyYCOTZgmHDTu9I1xvI-dMVLZm1EgmEH0vDA7QFLjPKShG86W2zwX0IbtBPHEDLO0WgQ8OIhZ6yTnu-S",
    "expirationTime":null,
    "keys":{
        "p256dh":"BAdAo6ldzRT5oCN8stqYRemoihPGOEJjrUDL6y8zhdA_swao_q-HlY_69mzIVobWX2MH02TzmtRWj_VeWUFMnXQ=","auth":"SS1PBnGwfMXjpJEfnoUIeQ=="
    }
}

```


### 2.FCM
https://firebase.google.com/docs/cloud-messaging/?hl=zh-cn
Firebase 云信息传递 (FCM) 是一种跨平台消息传递解决方案，可供您免费、可靠地传递消息。

使用 FCM，您可以通知客户端应用有新的电子邮件或其他数据有待同步。您可以发送通知消息进行用户再互动并留住他们。在即时通讯等使用情形中，一条消息可将最多 4KB 的有效负载传送至客户端应用

它最大的优点是同一套Push机制可以在IOS/Android/Web三端使用。


### 3.发送推送
FCM提供的web push的库，它支持多种语言，包括Node.js/PHP等版本。用Node.js可以这样发Push：
```
const webpush = require('web-push');
// 从数据库取出用户的subsciption
const pushSubscription = {"endpoint":"https://fcm.googleapis.com/fcm/send/ci3-kIulf9A:APA91bEaQfDU8zuLSKpjzLfQ8121pNf3Rq7pjomSu4Vg-nMwLGfJSvkOUsJNCyYCOTZgmHDTu9I1xvI-dMVLZm1EgmEH0vDA7QFLjPKShG86W2zwX0IbtBPHEDLO0WgQ8OIhZ6yTnu-S","expirationTime":null,"keys":{"p256dh":"BAdAo6ldzRT5oCN8stqYRemoihPGOEJjrUDL6y8zhdA_swao_q-HlY_69mzIVobWX2MH02TzmtRWj_VeWUFMnXQ=","auth":"SS1PBnGwfMXjpJEfnoUIeQ=="}};

// push的数据
const payload = {
    title: '一篇新的文章',
    body: '点开看看吧',
    icon: '/html/app-manifest/logo_512.png',
    data: {url: "https://www.rrfed.com"}
  //badge: '/html/app-manifest/logo_512.png'
};

webpush.sendNotification(pushSubscription, JSON.stringify(payload));
```


### 4.接收推送消息
运行在后台的Service Worker接收，监听push事件：
registration.showNotification()方法来显示消息提醒，该方法接收两个参数：title与option。title用来设置该提醒的主标题，option中则包含了一些其他设置。
```
body：提醒的内容
icon：提醒的图标
actions：提醒可以包含一些自定义操作
tag：相当于是ID，通过该ID标识可以操作特定的notification
renotify：是否允许重复提醒，默认为false。当不允许重复提醒时，同一个tag的notification只会显示一次
```
以下是消息提醒的代码示例
```
this.addEventListener('push', function(event) {
    console.log('[Service Worker] Push Received.');
    console.log(`[Service Worker] Push had this data: "${event.data.text()}"`);
    const title = event.data.text()
    const options = {
        body: event.data.text(),
        icon: '/img/avatar.21a57010.jpg',
    }
    // 弹消息框
    event.waitUntil(self.registration.showNotification(title, notificationData));
});
```
此外还有notificationclick事件，然后可以调用clients.openWindow打开一个新的页面

再比如，很多时候我们希望能引导用户进行交互。例如推荐一本新书，让用户点击阅读或购买。在上一部分我们设置的提醒框中，包含了“去看看”和“联系我”两个按钮选项，

![](https://user-gold-cdn.xitu.io/2018/5/1/1631a6c6007ffec9?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
![](https://user-gold-cdn.xitu.io/2018/5/1/1631a6f12cc17f0c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
```
actions: [{
    action: 'show-book',
    title: '去看看'
    }, {
    action: 'contact-me',
    title: '联系我'
}]
```
```
// sw.js
self.addEventListener('notificationclick', function (e) {
    var action = e.action;
    console.log(`action tag: ${e.notification.tag}`, `action: ${action}`);

    switch (action) {
        case 'show-book':
            console.log('show-book');
            break;
        case 'contact-me':
            console.log('contact-me');
            break;
        default:
            console.log(`未处理的action: ${e.action}`);
            action = 'default';
            break;
    }
    e.notification.close();
});

```
e.action获取的值，就是我们在showNotification()中定义的actions里的action。因此，通过e.action就可以知道用户点击了哪一个操作选项。注意，当用户点击提醒本身时，也会触发notificationclick，但是不包含任何action值，所以在代码中将其置于default默认操作中。

注意，由于不同浏览器中，对于option属性的支持情况并不相同。部分属性在一些浏览器中并不支持。

#### 4.1 Service Worker与client通信

到目前为止，我们已经可以顺利得给用户展示提醒，并且在用户操作提醒后准确捕获到用户的操作。然而，还缺最重要的一步——针对不同的操作，触发不同的交互。例如，

点击提醒本身会弹出书籍简介；
点击“看一看”会给用户展示本书的详情；
点击“联系我”会向应用管理者发邮件等等。

这里有个很重要的地方：我们在Service Worker中捕获用户操作，但是需要在client（这里的client是指前端页面的脚本环境）中触发相应操作（调用页面方法/进行页面跳转…）。因此，这就需要让Service Worker与client进行通信。通信包括下面两个部分：

1.在Service Worker中使用Worker的postMessage()方法来通知client：
```
// sw.js
self.addEventListener('notificationclick', function (e) {
    …… // 略去上一节内容

    e.waitUntil(
        // 获取所有clients
        self.clients.matchAll().then(function (clients) {
            if (!clients || clients.length === 0) {
                return;
            }
            clients.forEach(function (client) {
                // 使用postMessage进行通信
                client.postMessage(action);
            });
        })
    );
});

```
2.在client中监听message事件，判断data，进行不同的操作
```
// index.js
navigator.serviceWorker.addEventListener('message', function (e) {
    var action = e.data;
    console.log(`receive post-message from sw, action is '${e.data}'`);
    switch (action) {
        case 'show-book':
            location.href = 'https://book.douban.com/subject/20515024/';
            break;
        case 'contact-me':
            location.href = 'mailto:someone@sample.com';
            break;
        default:
            document.querySelector('.panel').classList.add('show');
            break;
    }
});

```

当用户点击提醒后，我们在notificationclick监听中，将action通过postMessage()通信给client；然后在client中监听message事件，基于action（e.data）来进行不同的操作（跳转到图书详情页/发送邮件/显示简介面板）。


我们还可以将这个功能再丰富一些。由于用户在关闭该网站时仍然可以收到提醒，因此加入一些更强大功能：
当用户切换到其他Tab时，点击提醒会立刻回到网站的tab；
当用户未打开该网站时，点击提醒可以直接打开网站。

### 兼容性
![](https://static.dingtalk.com/media/lALPDgQ9rPJf7yLNAsLNBUM_1347_706.png_620x10000q90g.jpg?auth_bizType=IM&auth_bizEntity=%7B%22cid%22%3A%224248001%3A418282984%22%2C%22msgId%22%3A%221852736438642%22%7D&bizType=im&open_id=418282984)




### 参考
https://developers.google.com/web/ilt/pwa/introduction-to-push-notifications
https://zhuanlan.zhihu.com/p/29932160
https://juejin.im/post/5ae7f7fd518825670960fe96
https://juejin.im/post/5af80c336fb9a07aab29f19c
https://github.com/web-push-libs/pywebpush
https://developers.google.com/web/fundamentals/push-notifications/subscribing-a-user#permissions_and_subscribe
https://github.com/web-push-libs/pywebpush
https://segmentfault.com/a/1190000013061924
https://developers.google.com/web/ilt/pwa/introduction-to-push-notifications
https://liaobu.de/h5-notification-and-web-push-introduction/
https://firebase.google.com/docs/projects/learn-more?hl=zh-cn
https://firebase.google.com/docs/web/setup?hl=zh-cn
https://juejin.im/post/5b06a7b3f265da0dd8567513
