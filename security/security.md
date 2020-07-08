### iOS 如何防止抓包

---

### 1、抓包原理

为了防止被抓包那么就要了解抓包的原理。

其实原理很是简单：一般抓包都是通过代理服务来冒充你的服务器，客户端真正交互的是这个假冒的代理服务，这个假冒的服务再和我们真正的服务交互，这个代理就是一个`中间者` ，我们所有的数据都会通过这个`中间者`，所以我们的数据就会被抓取。HTTPS 也同样会被这个`中间者`伪造的证书来获取我们加密的数据。

### 2、防止抓包

为了数据的更安全，那么我们如何来防止被抓包。

第一种思路是：如果我们能判断是否有代理，有代理那么就存在风险。

第二种思路：针对HTTPS 请求。我们判断证书的合法性。

第一种方式的实现：

 >一、发起请求之前判断是否存在代理，存在代理就直接返回，请求失败。
  >
  >```objective-c
  >CFDictionaryRef dicRef = CFNetworkCopySystemProxySettings();
  >    CFStringRef proxyStr = CFDictionaryGetValue(dicRef, kCFNetworkProxiesHTTPProxy);
  >    NSString *proxy = (__bridge NSString *)(proxyStr);
  >```
  >
  >二、我们可以在请求配置中清空代理，让请求不走代理
  >
  >我们通过hook到`sessionWithConfiguration:` 方法。然后清空代理
  >
  >````objective-c
  >+ (void)load{
  >    Method method1 = class_getClassMethod([NSURLSession class],@selector(sessionWithConfiguration:));
  >    Method method2 = class_getClassMethod([NSURLSession class],@selector(px_sessionWithConfiguration:));
  >    method_exchangeImplementations(method1, method2);
  >
  >    Method method3 = class_getClassMethod([NSURLSession class],@selector(sessionWithConfiguration:delegate:delegateQueue:));
  >    Method method4 = class_getClassMethod([NSURLSession class],@selector(px_sessionWithConfiguration:delegate:delegateQueue:));
  >    method_exchangeImplementations(method3, method4);
  >}
  >
  >+ (NSURLSession*)px_sessionWithConfiguration:(NSURLSessionConfiguration*)configuration delegate:(nullable id)delegate delegateQueue:(nullable NSOperationQueue*)queue
  >{
  >        if(configuration) configuration.connectionProxyDictionary = @{};
  >
  >    return [self px_sessionWithConfiguration:configuration delegate:delegate delegateQueue:queue];
  >}
  >
  >+ (NSURLSession*)px_sessionWithConfiguration:(NSURLSessionConfiguration*)configuration
  >{
  > 
  >        if(configuration) configuration.connectionProxyDictionary = @{};
  >
  >    return [self px_sessionWithConfiguration:configuration];
  >}
  >````
  >
  >
  >
  >

​     第二种思路的实现：

>主要是针对HTTPS 请求，对证书的一个验证。
>
>通过 SecTrustRef 获取服务端证书的内容
>
>```objective-c
>static NSArray * AFCertificateTrustChainForServerTrust(SecTrustRef serverTrust) {
>    CFIndex certificateCount = SecTrustGetCertificateCount(serverTrust);
>    NSMutableArray *trustChain = [NSMutableArray arrayWithCapacity:(NSUInteger)certificateCount];
>
>    for (CFIndex i = 0; i < certificateCount; i++) {
>        SecCertificateRef certificate = SecTrustGetCertificateAtIndex(serverTrust, i);
>        [trustChain addObject:(__bridge_transfer NSData *)SecCertificateCopyData(certificate)];
>    }
>
>    return [NSArray arrayWithArray:trustChain];
>}
>
>```
>
>然后读取本地证书的内容进行对比
>
>````objective-c
>  NSString *cerPath = [[NSBundle mainBundle] pathForResource:@"certificate" ofType:@"cer"];//证书的路径
>    NSData *certData = [NSData dataWithContentsOfFile:cerPath];
>SSet<NSData*> * set = [[NSSet alloc]initWithObjects:certData  , nil];  // 本地证书内容
>// 服务端证书内容
>NSArray *serverCertificates = AFCertificateTrustChainForServerTrust(serverTrust);
>            
>            for (NSData *trustChainCertificate in [serverCertificates reverseObjectEnumerator]) {
>                if ([set containsObject:trustChainCertificate]) {
>                    // 证书验证通过
>                }
>            }
>````
>
>
>
>