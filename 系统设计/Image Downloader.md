# Image Downloader

这是一道Google Onsite的题目，要求十分简单：

	> Write an Image Downloader.

面试中面对越是简单的题目我们就越要小心面对，因为缺少足够的约束条件，我们需要与面试官仔细讨论需要解决的问题，确认需求，做出假设，选定技术之后才进入到写代码阶段。在做出技术选定时，我们也要陈述充分的理由来支持我们的选择。

我们先来看看一个`Image Downloader`在实际使用时可能会遇到的一些问题：

* 一个比较常见的例子，我们在UITableViewCell中下载图片时，当我们快速地滑动UITableView或者碰到速度较慢的网络时，可能某一个UITableViewCell还没有完成下载图片就进入了`reuse cycle`，那么这个Cell之前发出的API call返回的内容就不会被用到了。所以我们需要我们的`Image Downloader`拥有能够取消`data request`的功能。这样不但能够减少本地资源的消耗，也能够减轻服务器的负担。
* 另外一个问题，当我们的`Image Downloader`在遇到一个之前已经下载过图片的URL时，我们是不是需要再重新下载一次呢？答案当然是不需要的，如果我们在之前的下载任务中已经成功地下载了该图片，我们的`Image Downloader`可以直接返回之前已经下载了的图片。
* 按照常规的设计，我们的Image Downloader肯定是要在后台线程中运行我们的下载任务，那么`Image Downloader`最多可以让多少个task同时运行呢？我们是不是要提供一些简单地API来让工程师可以根据机器的性能进行设置呢？
* 最后一个问题，我们要将`Image Downloader`设计成什么呢？是一个`Helper Class`？一个`UIImageView`的`Extension`？个人觉得将`Image Downloader`设计成一个`Helper Class`并在`UIImageView`或者`UIImage`的`extension`中提供一些`helper function`可能会更好。这样不但可以提高我们`Image Donwoader`的泛用性，也可以为需要的类提供Syntax Suger。

根据上面的一些问题和假设，我们可以进入写代码的阶段了。

## ImageCache

首先，我们需要一个`Image Cache`来返回已经成功地下载了的图片，避免造成资源的浪费。

``` swift
protocol ImageCacheManager {
    func imageFor(url: String) -> UIImage?
    func insertImage(for url: String, image: UIImage)
    func removeImage(for url: String)
}
```

为什么要先定义一个`protocol`？这里我是出于测试和扩展的考虑，有了这个`protocol`并使用依赖注入，我们在测试的时候可以通过简单的Mock来为测试环境提供简单的`image cache`；同时，考虑到我们可能想向`Image Downloader`中传入用不同方式实现的`ImageCacheManager`来为其提供不同的功能。所以，这里我选择了先定义`ImageCacheManger Protocol`。

``` swift
final class ImageCache: ImageCacheManager {
    public static let shared: ImageCache = ImageCache() // 在这里使用了一个简单的Singleton并使用了默认的cacheLimit
    
    private lazy var imageCache: NSCache<AnyObject, AnyObject> = {
       let cache = NSCache<AnyObject, AnyObject>()
        cache.countLimit = self.cacheLimit
        return cache
    }()
    private let lock = NSLock() // 用NSLock来实现读写时的线程安全，也可以使用Serial Queue来解决
    private let cacheLimit: Int
    
    init(cacheLimit: Int = 30) {
        self.cacheLimit = cacheLimit
    }
    
    func imageFor(url: String) -> UIImage? {
        lock.lock()
        defer { lock.unlock() }
        if let image = imageCache.object(forKey: url as AnyObject) as? UIImage {
            return image
        } else {
            return nil
        }
    }
    
    func insertImage(for url: String, image: UIImage) {
        lock.lock()
        defer { lock.unlock() }
        imageCache.setObject(image as AnyObject, forKey: url as AnyObject)
    }
    
    func removeImage(for url: String) {
        lock.lock()
        defer { lock.unlock() }
        imageCache.removeObject(forKey: url as AnyObject)
    }
}
```



## ImageDownloadTask

根据之前我们的假设与需求，我们需要下载任务能够执行取消这一功能。而在Swift中，只有`Operation`支持取消的功能，所以这里我们新建一个类，并让其继承自`Operation`

``` swift
class ImageDownloadTask: Operation  {
    public let url: String
    private let completion: (UIImage?) -> Void // 这个completion closure用于下载成功之后的回调
    
    init(url: String, completion: @escaping (UIImage?) -> Void) {
        self.url = url
        self.completion = completion
    }
    
    override func main() {
        if isCancelled {
            return
        }
        
        guard let url = URL(string: url) else { return }
        let task = URLSession.shared.dataTask(with: url, completionHandler: { (data, response, error) in
            if let data = data, let downloadedImage = UIImage(data: data) {
                if !self.isCancelled {
                    self.completion(downloadedImage)
                }
            } else {
              self.completion(nil)
            }
        })
        
        if isCancelled { return }
        task.resume()
    }
}
```

有了`ImageDownloadTask`之后，我们就可以继续写`ImageDownloader`了:

``` swift
class ImageDownloader {
    public static let shared: ImageDownloader = ImageDownloader()
    private let operationQueue: OperationQueue
    private var downloadTasks: [String: ImageDownloadTask] = [:] // 我们使用了一个简单的Dict来记录正在执行和还未执行的下载任务
    private let lock = NSLock() // 同样的，我们也使用NSLock来保证对dict的线程安全的操作
    
    init(maxConcurrentTask: Int = 5) {
        self.operationQueue = OperationQueue()
        self.operationQueue.maxConcurrentOperationCount = maxConcurrentTask
    }
    
    func downloadImage(from url: String, completion: @escaping (UIImage?) -> Void) {
        lock.lock()
        if let downloadTask = downloadTasks[url] {
            downloadTask.cancel()
            downloadTasks[url] = nil
        } else {
            let downloadTask = ImageDownloadTask(url: url) { (image) in
                self.downloadTasks[url] = nil
                completion(image)
            }
            downloadTasks[url] = downloadTask
            operationQueue.addOperation(downloadTask)
        }
        lock.unlock()
    }
}
```

上面的代码逻辑并不复杂，下面我们来把`ImageDownloader`和`ImageCache`结合起来为`UIImageView`添加`extension`

## UIImageView Extension

``` swift
extension UIImageView {
    public func downloadImage(from url: String, imageCache: ImageCacheManager = ImageCache.shared, Image completion: (() -> Void)? = nil) {
        if let downloadedImage = imageCache.imageFor(url: url) {
            self.image = downloadedImage
        } else {
            ImageDownloader.shared.downloadImage(from: url) { [weak self] image in
                guard let image = image else { return }
	              imageCache.insertImage(for: url, image: image)
                DispatchQueue.main.async {
	                  guard let strongSelf = self else { return }
                    strongSelf.image = image
                    completion?()
                }
            }
        }
    }
}
```

这样我们就完成了一个简单的`ImageDownloader`,当然实现有些简陋，如果能在白板上写成这样我觉得基本可以了。我觉得也有一些可以改进的地方，比如说现在的Cache实现的有些简单，只能在内存中实现，我们应该提供可以永久保存下载图片的选项；

最后说一下，现在也有很多成熟的第三方开源`Image Download`的SDK，比如经典的`SDWebImage`，这里强烈建议阅读它的源代码，并进行这道题的练习。