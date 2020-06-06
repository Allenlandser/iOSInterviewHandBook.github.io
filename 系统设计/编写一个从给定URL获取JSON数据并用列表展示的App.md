# 编写一个从给定URL获取JSON数据并用列表展示的App

就这一道算法题而言，一般来说，获取了URL数据之后，面试官会要求用将获得的数据进行展示，并在每一个Cell中显示图片。虽然看上去十分简单，如果面试时间有两个小时，那么我们可以从容不迫地进行编写、调试。但是如果加上了时间限制，比如40分钟，那么难度可能会有所加大，这要求面试者能够有快速从零编写代码的能力。所以我建议在面试之前不要太轻视这道面试题，而是预先准备一个模板并将其熟练运用，以免在面试时忙中出错，阴沟里翻船。

这里我们来看看一道实际的题：

> 在40分钟时间内，从下面的开源API中获取出新闻的标题文字与图片的URL，并编写一个App用UITableView来展示
>
> API：https://openapi.newsbreak.com/serving?app=samsung_daily&token=20Ki5pvahIrV9CcNHkJuCewiIFVKPdGY

对于这道题来说，因为时间十分紧凑，我们可能并没有多余的时间来进行详细的系统设计，所以直接使用最基本的MVC模式就可以了。下面我会提供一下我的代码，仅做参考。

## 解析JSON数据

一般来说，写App的第一步就是解析JSON数据，这里提供一个比较有用的网站，能够根据传入的JSON数据来生成相应的能够Decode的数据模型。如果返回的JSON数据十分复杂的话，这能够大大地节约我们的时间。

[https://app.quicktype.io/](https://app.quicktype.io/)

完成之后，我们就拥有了我们的原始数据模型：

```swift
struct NewsResult: Codable {
    let status: String
    let code: Int
    let result: [Result]
}

// MARK: - Result
struct Result: Codable {
    let type: String
    let documents: [Document]
}

// MARK: - Document
struct Document: Codable {
    let summary: String?
    let epoch: Int
    let labelBgColor: LabelBgColor
    let id, titleText: String
    let imageSource: String
    let subTitleTextRight: String
    let linkURL: String
    let labelText: String
    
    enum CodingKeys: String, CodingKey {
        case summary, epoch, labelBgColor
        case id = "_id"
        case titleText, imageSource, subTitleTextRight
        case linkURL = "linkUrl"
        case labelText
    }
}

enum LabelBgColor: String, Codable {
    case e15B5B = "#E15B5B"
}
```

## 编写APIClient

有了原始的数据模型之后，下一步就是编写APIClient，这里我没有使用任何第三方的SDK，如果你对`Alamofire`，`AFNetworking`等比较熟悉，在与面试官确认之后，也可以使用。

我们先来写一个`Data`类的extension，来方便我们解析JSON：

```swift
extension Data {
    func decode<T: Decodable>() -> T? {
        let decoder: JSONDecoder = JSONDecoder()
        return try? decoder.decode(T.self, from: self)
    }
}
```

之后我们来定义一些其它的一些`Enum`和`Struct`来封装数据。

```swift
public struct URLString {
    static let news: String = "https://openapi.newsbreak.com/serving?app=samsung_daily&token=20Ki5pvahIrV9CcNHkJuCewiIFVKPdGY"
}

// 如果你使用Swift5，那么可以直接使用系统提供的Result
// 这个Enum用来封装API请求的结果
public enum ResponseResult<T> {
    case success(T)
    case error(Error)
}

public enum APIClientError: Error {
    case invalidURL
    case canNotDecodeData
}
```

最后我们添加我们的`APIClient`

```swift
public class APIClient {
    public static let shared: APIClient = APIClient()
    private let session: URLSession = URLSession.shared
    private let queue: DispatchQueue = DispatchQueue.init(label: "apiClient.queue", attributes: .concurrent)
    
    public func getData<T: Decodable>(from urlString: String, completion: @escaping (ResponseResult<T>) -> Void) {
        if let url = URL(string: urlString) {
            let task = self.session.dataTask(with: url) { (data, response, error) in
                if let resultData: T = data?.decode(), error == nil {
                    completion(ResponseResult.success(resultData))
                } else if let error = error {
                    completion(ResponseResult.error(error))
                } else {
                    completion(ResponseResult.error(APIClientError.canNotDecodeData))
                }
            }
            queue.async {
                task.resume()
            }
        } else {
            completion(ResponseResult.error(APIClientError.invalidURL))
        }
    }
}
```

在这里我简单地使用了单例模式，单纯为了节约时间，工程中可能会更加复杂一些；同时我也使用了泛型来编写`getData`的方法，来增加代码的通用性和可扩展性；最后我将`task.resume()`放到GCD的并发队列中执行来减轻`main thread`的压力。

## ViewController

有了APIclient之后，我们就可以进行下一步了，即完成我们的ViewController的编写。因为时间紧迫，我们可以让ViewController承担比较多的内容，比如获取数据和展示数据等工作。

因为需要显示的Cell不是特别复杂，所以我们先简单定义一下Cell的ViewModel

``` swift
struct NewsModel {
    let titleText: String
    let imageSource: String
    let subtitleText: String
    
    init(document: Document) {
        self.titleText = document.titleText
        self.imageSource = document.imageSource
        self.subtitleText = document.labelText
    }
}
```

在这之后我们来完成基础UI的部分：

``` swift
class ViewController: UIViewController {

    @IBOutlet var tableView: UITableView!
    private var news: [NewsModel] = []
    
    override func viewDidLoad() {
        super.viewDidLoad()
        configureTableView()
        getNews()
    }
    
    private func configureTableView() {
        tableView.delegate = self
        tableView.dataSource = self
        tableView.register(UINib(nibName: "newsTableViewCell", bundle: nil), forCellReuseIdentifier: "NewsTableViewCell")
        tableView.rowHeight = UITableViewAutomaticDimension
        tableView.estimatedRowHeight = 350
        tableView.separatorColor = .clear
    }
}
```

如何自定义一个UITableViewCell的方式有很多，大家选择一个自己最顺手的方式就可以了。因为每个公司的编码规范都不同，所以面试时不会强求。

接下来我们添加我们的`getNews`方法

``` swift
class ViewController: UIViewController {
  ...
  private func getNews() {
        APIClient.shared.getData(from: URLString.news) 
    			{ (response: ResponseResult<NewsResult>) in
            switch response {
                case let .success(newsResult):
                    if let documents = newsResult.result.first?.documents {
                        self.news = documents.map { document in
                            return NewsModel(document: document)
                        }
                        DispatchQueue.main.async {
                            self.tableView.reloadData()
                        }
                    }
                case let .error(error):
                    print(error) // 这里简化了Error Hanlding
            }
        }
    }
}
```

最后我们来实现一下`UITableView`的`delegate`方法：

``` swift
extension ViewController: UITableViewDelegate {
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        if let cell = tableView.dequeueReusableCell(withIdentifier: "NewsTableViewCell") as? NewsTableViewCell {
         let newsData = news[indexPath.row]
            cell.configure(newsData.titleText, subtitle: newsData.subtitleText, newsData.imageSource) {
            }
            return cell
        }
        return UITableViewCell()
    }
}

extension ViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return self.news.count
    }
}
```

这样我们就基本完成了`ViewController`的构建

## UITableViewCell

下一步我们进行UITableViewCell的编写。我这里采用了`.xib`做界面并使用了代码做进一步的逻辑控制。在界面方面我主要使用`UIStackView`进行布局，可以节省大量的时间。

`xib`的layout如下图所示：

具体的代码则如下所示

``` swift
class NewsTableViewCell: UITableViewCell {
    
    @IBOutlet var stackView: UIStackView!
    @IBOutlet var titleLabel: UILabel!
    @IBOutlet var subTitleLabel: UILabel!
    @IBOutlet var imageV: UIImageView!
    
    override func awakeFromNib() {
        super.awakeFromNib()
        setup()
    }
  
 	  override func prepareForReuse() {
        super.prepareForReuse()
        self.imageV.image = nil
    }
    
    private func setup() {
        self.contentView.autoresizingMask = .flexibleHeight
        self.imageV.layer.masksToBounds = true
        self.imageV.layer.cornerRadius = 5.0
    }
    
    func configure(_ title: String, subtitle: String, _ imageUrl: String, completion: @escaping () -> Void) {
        self.titleLabel.text = title
        self.subTitleLabel.text = subtitle
        self.imageV.downloadImage(from: imageUrl) { 
            completion()
        }
    }
}
```

这里我们可以看到我直接在`UIImageView`里调用了`downloadIamge`方法，我们会在下一节详解。

## Download Image

下载图片的第三方SDK有很多，我相信大家在工作中也使用了很多。如果可以使用的话，那么直接拿来用就可以了。这里我稍微展示一下一个简单的ImageDownloader要怎么写。

``` swift
import UIKit

let imageCache = NSCache<AnyObject, AnyObject>()

extension UIImageView {
    func downloadImage(from url: String, completion: (() -> Void)? = nil) {
        guard let url = URL(string: urlSting) else { return }
        image = nil
        
        if let imageFromCache = imageCache.object(forKey: urlSting as AnyObject) {
            image = imageFromCache as? UIImage
            return
        }
       APIClient.shared.getImageData(from: url) { [weak self] result in
            guard let self = self else { return }
            switch result {
            case .success(let data):
                guard let imageToCache = UIImage(data: data) else { return }
                imageCache.setObject(imageToCache, forKey: urlSting as AnyObject)
                self.image = UIImage(data: data)
            case .failure(_):
                break // 简化处理
            }
        }
    }
}
```

我们在APIClient中再添加相应的代码：

```swift
public class APIClient {
  ...
  public func getImageData(from urlString: String, completion: @escaping (ResponseResult<Data>) -> Void) {
        if let url = URL(string: urlString) {
            let task = self.session.dataTask(with: url) { (data, response, error) in
                if let data = data, error == nil {
                    completion(ResponseResult.success(data))
                } else if let error = error {
                    completion(ResponseResult.error(error))
                }
            }
            queue.async {
                task.resume()
            }
        } else {
            completion(ResponseResult.error(APIClientError.invalidURL))
        }
    }
}
```

这样我们就基本完成了一个简单的App的实现，当然，这里的Image Download的代码有很大可以优化的地方，我会在以后的文章中进行详细解读。

上面的这个例子只是最基本的例子，就像我所说的，使用第三方SDK或者别的设计模式，都能够对其进行优化。如果面试的时间长达两个小时，那么可能变化和需求会更多一些，例如增加UISearchBar，分页显示，详情页面跳转，添加单元测试等功能都会要求实现。个人建议经常练习这一个模式并熟练掌握，以免在面试时手忙脚乱。

最后来看一下Demo吧：

