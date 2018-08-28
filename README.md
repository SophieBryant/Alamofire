# Alamofire

SwiftyBeaver 可以为 Xcode 控制台提供强大的日志记录功能，包括颜色（Xcode 9 以后都没有颜色输出了，但表情可以。）， 可自定义格式等等。

## 一、功能介绍

* 链式请求/响应方法
* URL / JSON / plist参数编码
* 上传文件/数据/流/多表单数据
* 使用请求或者断点下载来下载文件
* 使用URL凭据进行身份认证
* HTTP响应验证
* 包含进度的上传和下载闭包
* cURL命令的输出
* 动态适配和重试请求
* TLS证书和Public Key Pinning
* 网络可达性
* 全面的单元和集成测试覆盖率

### 组件库

为了让Alamofire专注于核心网络的实现，Alamofire生态系统还有另外两个库：

* [AlamofireImage](https://link.jianshu.com/?t=https%3A%2F%2Fgithub.com%2FAlamofire%2FAlamofireImage)：一个图片库，包括图像响应序列化器、UIImage和UIImageView的扩展、自定义图像滤镜、内存中自动清除和基于优先级的图像下载系统。
* [AlamofireNetworkActivityIndicator](https://link.jianshu.com/?t=https%3A%2F%2Fgithub.com%2FAlamofire%2FAlamofireNetworkActivityIndicator):控制iOS应用的网络活动指示器。包含可配置的延迟计时器来帮助减少闪光，并且支持不受Alamofire管理的URLSession实例。

## 二、安装

### CocoaPods

在 `Podfile` 编辑：
```
source 'https://github.com/CocoaPods/Specs.git'
platform :ios, '10.0'
use_frameworks!

target '<Your Target Name>' do
    pod 'Alamofire', '~> 4.7'
end
```

然后运行 `pod install` 即可。

### Carthage

在 `Cartfile` 中输入:

```
Github“Alamofire / Alamofire”〜> 4.7
```

运行 `carthage update` 更新以构建框架并将构建的 Alamofire.framework 拖入 Xcode 项目。


## 三、如何使用

### 1.发请求

导入完成后，我们就可以使用 Alamofire 请求网络了，我们可以发送最基本的 GET 请求：

```Swift
Alamofire.request("http://baidu.com/")
```

当然，我们还可以为请求添加参数，并且处理响应信息，调用方式也很简单：

```Swift
Alamofire.request(.GET, "https://httpbin.org/get", parameters: ["foo": "bar"])
    .responseJSON { response in

        print(response.request)  // 请求对象
        print(response.response) // 响应对象
        print(response.data)     // 服务端返回的数据

        if let JSON = response.result.value {
            print("JSON: \(JSON)")
        }

}
```

Alamofire 提供了多种返回数据的序列化方法，比如刚才我们用到的 `responseJSON`， 会返回服务端返回的 JSON 数据，这样就不用我们自己再去解析了。

下面是 Alamofire 目前支持的数据序列化方法：

response()
responseData()
responseString(encoding: NSStringEncoding)
responseJSON(options: NSJSONReadingOptions)
responsePropertyList(options: NSPropertyListReadOptions)
支持普通数据，字符串， JSON 和 plist 形式的返回。

### 上传文件

Alamofire 提供了 upload 方法用于上传本地文件到服务器：

```Swift
let fileURL = NSBundle.mainBundle().URLForResource("Default", withExtension: "png")
Alamofire.upload(.POST, "https://httpbin.org/post", file: fileURL)
```

当然，我们还可以获取上传时候的进度：

```Swift
Alamofire.upload(.POST, "https://httpbin.org/post", file: fileURL)
         .progress { bytesWritten, totalBytesWritten, totalBytesExpectedToWrite in
             print(totalBytesWritten)

             // This closure is NOT called on the main queue for performance
             // reasons. To update your ui, dispatch to the main queue.
             dispatch_async(dispatch_get_main_queue()) {
                 print("Total bytes written on main queue: \(totalBytesWritten)")
             }
         }
         .responseJSON { response in
             debugPrint(response)
         }
```

还支持 MultipartFormData 形式的表单数据上传：

```Swift
Alamofire.upload(
    .POST,
    "https://httpbin.org/post",
    multipartFormData: { multipartFormData in
        multipartFormData.appendBodyPart(fileURL: unicornImageURL, name: "unicorn")
        multipartFormData.appendBodyPart(fileURL: rainbowImageURL, name: "rainbow")
    },
    encodingCompletion: { encodingResult in
        switch encodingResult {
        case .Success(let upload, _, _):
            upload.responseJSON { response in
                debugPrint(response)
            }
        case .Failure(let encodingError):
            print(encodingError)
        }
    }
)
```

### 下载文件

* Alamofire 同样也提供了文件下载的方法：

```Swift
Alamofire.download(.GET, "https://httpbin.org/stream/100") { temporaryURL, response in
    let fileManager = NSFileManager.defaultManager()
    let directoryURL = fileManager.URLsForDirectory(.DocumentDirectory, inDomains: .UserDomainMask)[0]
    let pathComponent = response.suggestedFilename

    return directoryURL.URLByAppendingPathComponent(pathComponent!)
}
```

还可以设置默认的下载存储位置：

```Swift
let destination = Alamofire.Request.suggestedDownloadDestination(directory: .DocumentDirectory, domain: .UserDomainMask)
Alamofire.download(.GET, "https://httpbin.org/stream/100", destination: destination)
```

也提供了 progress 方法检测下载的进度：

```Swift
Alamofire.download(.GET, "https://httpbin.org/stream/100", destination: destination)
         .progress { bytesRead, totalBytesRead, totalBytesExpectedToRead in
             print(totalBytesRead)

             dispatch_async(dispatch_get_main_queue()) {
                 print("Total bytes read on main queue: \(totalBytesRead)")
             }
         }
         .response { _, _, _, error in
             if let error = error {
                 print("Failed with error: \(error)")
             } else {
                 print("Downloaded file successfully")
             }
         }
```

恢复下载——如果一个 DownloadRequest 被取消或中断，底层的 URL 会话会生成一个恢复数据。恢复数据可以被重新利用并在中断的位置继续下载。恢复数据可以通过下载响应访问，然后在重新开始请求的时候被利用。

重要：在iOS 10 - 10.2, macOS 10.12 - 10.12.2, tvOS 10 - 10.1, watchOS 3 - 3.1.1中，resumeData 会被后台 URL 会话配置破坏。因为在 resumeData 的生成逻辑有一个底层的 bug，不能恢复下载。具体情况可以到 [Stack Overflow](https://link.jianshu.com/?t=http%3A%2F%2Fstackoverflow.com%2Fquestions%2F39346231%2Fresume-nsurlsession-on-ios10%2F39347461%2339347461) 看看。

```Swift
class ImageRequestor {
    private var resumeData: Data?
    private var image: UIImage?

    func fetchImage(completion: (UIImage?) -> Void) {
        guard image == nil else { completion(image) ; return }

        let destination: DownloadRequest.DownloadFileDestination = { _, _ in
            let documentsURL = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
            let fileURL = documentsURL.appendPathComponent("pig.png")

            return (fileURL, [.removePreviousFile, .createIntermediateDirectories])
        }

        let request: DownloadRequest

        if let resumeData = resumeData {
            request = Alamofire.download(resumingWith: resumeData)
        } else {
            request = Alamofire.download("https://httpbin.org/image/png")
        }

        request.responseData { response in
            switch response.result {
            case .success(let data):
                self.image = UIImage(data: data)
            case .failure:
                self.resumeData = response.resumeData
            }
        }
    }
}
```

### HTTP 验证

Alamofire 还很提供了一个非常方便的 authenticate 方法提供了 HTTP 验证：

```Swift
let user = "user"
let password = "password"

Alamofire.request(.GET, "https://httpbin.org/basic-auth/\(user)/\(password)")
         .authenticate(user: user, password: password)
         .responseJSON { response in
             debugPrint(response)
         }
```

### HTTP 响应状态信息识别

Alamofire 还提供了 HTTP 响应状态的判断识别，通过 validate 方法，对于在我们期望之外的 HTTP 响应状态信息，Alamofire 会提供报错信息：

```Swift
Alamofire.request(.GET, "https://httpbin.org/get", parameters: ["foo": "bar"])
         .validate(statusCode: 200..<300)
         .validate(contentType: ["application/json"])
         .response { response in
             print(response)
         }
```

validate 方法还提供自动识别机制，我们调用 validate 方法时不传入任何参数，则会自动认为 200...299 的状态吗为正常：

```Swift
Alamofire.request(.GET, "https://httpbin.org/get", parameters: ["foo": "bar"])
         .validate()
         .responseJSON { response in
             switch response.result {
             case .Success:
                 print("Validation Successful")
             case .Failure(let error):
                 print(error)
             }
         }
```

### 调试状态

我们通过使用 debugPrint 函数，可以打印出请求的详细信息，这样对我们调试非常的方便：

```Swift
let request = Alamofire.request(.GET, "https://httpbin.org/get", parameters: ["foo": "bar"])

debugPrint(request)
```

这样就会产生如下输出：

```
$ curl -i \
    -H "User-Agent: Alamofire" \
    -H "Accept-Encoding: Accept-Encoding: gzip;q=1.0,compress;q=0.5" \
    -H "Accept-Language: en;q=1.0,fr;q=0.9,de;q=0.8,zh-Hans;q=0.7,zh-Hant;q=0.6,ja;q=0.5" \
    "https://httpbin.org/get?foo=bar"
```


