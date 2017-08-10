---
title: Core ML 初体验：构建简单的图像识别应用
date: 2017-07-12 08:45:22
categories: 译文
---

> [原文链接](https://www.appcoda.com/coreml-introduction)

在 WWDC 2017 上，苹果公司发布了很多令开发者激动不已的框架和 API。在这些框架中，Core ML 无疑是最受关注的框架之一。利用 Core ML，你可将机器学习模型整合到应用中，但并不需要了解神经网络和机器学习的相关知识。另外，你可以使用一些预处理的数据模型，只要将其转换成 Core ML 的模型即可。为了演示，我们将使用苹果开发者网站所提供的 Core ML 模型。闲话少叙，让我们开始学习 Core ML 吧。

<!-- more -->

{% note success %} 
<span style="color: #5FB760;">**提示：**为了完成教程，你需要使用 Xcode 9 beta 版，另外还需要运行 iOS 11 beta 版的设备，来测试教程中所介绍的特性。Xcode 9 beta 兼容 Swift 3.2 和 4.0 版本，以下代码全部基于 Swift 4.0 的语法完成。</span> 
{% endnote %} &nbsp;

## Core ML 简介

> Core ML 可以使你将种类繁多的机器学习模型整合到应用中，除了支持30多种深度学习的模型类型外，还支持如 tree ensemble、SVM 和 generalized linear model 等标准模型。由于 Core ML 构建于 Matel 和 Accelerate 这种底层技术之上，因此，它可以充分利用 CPU、GPU 所提供的超高性能。你可以在设备上运行机器学习模型，而不用将分析设备和数据分离。
> 
> **苹果官方文档** - [Core ML](https://developer.apple.com/machine-learning/)

Core ML 是一款全新的框架，整合于 iOS 11 中，首次亮相于今年的 WWDC 上。有了它，你可以将机器学习模型整合进自己的应用中，多说一句，究竟什么是机器学习呢？简言之，机器学习是一种应用，这种应用可以不通过编程的方法，来给予计算机学习的能力。而模型则是机器学习算法与被处理数据结合后的产物。 

{% asset_img images/trained-model.png %} &nbsp;

作为一名应用开发者，我们最关心的事，是如何把这些模型加入应用中，来做一些真正有趣的事。幸运的是，通过 Core ML，可以轻松集成多种机器学习模型，这使得开发者们可以实现很多特性，如图像识别、自然语言处理、文本预测等。

现在，你可能会问，将这种人工智能整合到应用中会不会很困难？重点来了，Core ML 相当易于使用，在这篇教程中，你会看到我们仅用 10 行代码，就将 Core ML 整合到了应用中。

是不是很酷？让我们开始吧！

## Demo 预览

应用很简单，用户可以通过拍照或在相册中选择照片，然后，机器学习算法会尝试预测图中的物体是什么，预测结果有可能并不那么完美，但重点是，你可以学会如何在应用中使用 Core ML。

{% asset_img images/coreml-app-demo.png %} &nbsp;

## Getting Started

首先，打开 Xcode 9 并创建工程，选择 Single View App 模板，开发语言选择 Swift。

{% asset_img images/xcode9-new-proj.png %} &nbsp;


## 创建用户界面 

{% note success %} 
<span style="color: #5FB760;">**编者提示：**如果你不想从零开始构建 UI，可以**[下载初始工程模板](https://github.com/appcoda/CoreMLDemo/raw/master/CoreMLDemoStarter.zip)**，并直接跳转到 Core ML 部分。</span> 
{% endnote %} &nbsp;

首先，切换至 `Main.storyboard` 文件，为视图添加一些 UI 组件，选中 ViewController 后，点击 Xcode 菜单栏中的 `Editor -> Embed In -> Navigation Controller`，完成该操作后，你会发现导航栏出现在了视图的顶部，双击导航栏，将标题修改为 Core ML（或者你认为合适的内容）。

{% asset_img images/pic3.png %} &nbsp;

接着，拖拽两个 BarButtonItem：分别放置于导航栏的两侧。选择左侧的 BarButtonItem，在 `Attributes Inspector` 中，修改 `System Item` 属性为"Camera"。然后选择右侧的 BarButtonItem，修改其名称为"Library"，这两个按钮可以让用户拍照或从相册中选择照片。

最后，你还需要添加 `UILabel` 和 `UIImageView` 这两个组件。将 `UIImageView` 置于视图的中间，并修改其大小为 `299x299`。将 `UILabel` 置于视图地步，将其拉伸并顶满左右边框。至此，UI 部分已经完成了。

我并没有提及如何配置视图组件的 Auto Layout 设置，我建议你来尝试完成，避免出现位置有歧义的视图组件。如果你对 Auto Layout 并不熟悉，从而无法完成这些设置，请在你要运行的设备上构建 Storyboard。

{% asset_img images/coreml-storyboard.png %} &nbsp;

## 实现 Camera 和照片库功能

UI 部分已经设计完成，让我们转到实现部分。在这部分中，我们会实现 Camera 和 Library 按钮的功能。选择 `ViewController.swift 文件`，使其符合 `UINavigationControllerDelegate` 协议，该协议是使用 `UIImagePickerController` 类所必须的。

{% codeblock lang:swift %}
class ViewController: UIViewController, UINavigationControllerDelegate
{% endcodeblock %} &nbsp;

然后，为 label 和 image view 添加两个 outlet，将 `UIImageView` 类型的变量命名为 imageView，将 `UILabel` 类型的变量命名为 classifier。现在，你的代码看起来应该像下面这样：

{% codeblock lang:swift %}
import UIKit

class ViewController: UIViewController, UINavigationControllerDelegate {
    
    @IBOutlet weak var imageView: UIImageView!
    @IBOutlet weak var classifer: UILabel!
    
    override func viewDidLoad() {
        super.viewDidLoad()
    }

    override func didReceiveMemoryWarning() {
        super.didReceiveMemoryWarning()
    }

}
{% endcodeblock %} &nbsp;

接下来，为两个 BarButtonItem 创建对应的 action，来响应其点击事件。将下面的方法添加到 `ViewController` 类中：

{% codeblock lang:swift %}
@IBAction func camera(_ sender: Any) {
    if !UIImagePickerController.isSourceTypeAvailable(.camera) {
        return
    }
    
    let cameraPicker = UIImagePickerController()
    cameraPicker.delegate = self
    cameraPicker.sourceType = .camera
    cameraPicker.allowsEditing = false
    
    present(cameraPicker, animated: true)
}

@IBAction func openLibrary(_ sender: Any) {
    let picker = UIImagePickerController()
    picker.delegate  = self
    picker.sourceType = .photoLibrary
    picker.allowsEditing = false
    
    present(picker, animated: true)
}
{% endcodeblock %} &nbsp;

我们在这两个 action 中所做的，就是创建了一个 `UIImagePickerController` 类型的常量，禁止用户对照片进行编辑（无论是拍照还是从相册中选取的），然后设置该常量的 delegate 为 `ViewController` 对象，最后，我们把 `UIImagePickerController` 对象展现给用户。

我们会看到一些编译错误，因为我们尚未实现 `UIImagePickerControllerDelegate` 类的方法，接下来，我们通过扩展的方式，使 `ViewController` 对象适配代理协议。 

{% codeblock lang:swift %}
extension ViewController: UIImagePickerControllerDelegate {
    
    func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
        dismiss(animated: true, completion: nil)
    }
    
}
{% endcodeblock %} &nbsp;

当用户取消选择图片时，通过上面的代码进行处理。另外，这段代码也是 `ViewController.swift` 文件的一部分，现在，你的代码看起来应该像这样：

{% codeblock lang:swift %}
import UIKit

class ViewController: UIViewController, UINavigationControllerDelegate {
    
    @IBOutlet weak var imageView: UIImageView!
    @IBOutlet weak var classifer: UILabel!
    
    override func viewDidLoad() {
        super.viewDidLoad()
    }

    override func didReceiveMemoryWarning() {
        super.didReceiveMemoryWarning()
    }
    
    @IBAction func camera(_ sender: Any) {
        if !UIImagePickerController.isSourceTypeAvailable(.camera) {
            return
        }
        
        let cameraPicker = UIImagePickerController()
        cameraPicker.delegate = self
        cameraPicker.sourceType = .camera
        cameraPicker.allowsEditing = false
        
        present(cameraPicker, animated: true)
    }
    
    @IBAction func openLibrary(_ sender: Any) {
        let picker = UIImagePickerController()
        picker.delegate  = self
        picker.sourceType = .photoLibrary
        picker.allowsEditing = false
        
        present(picker, animated: true)
    }

}

extension ViewController: UIImagePickerControllerDelegate {
    
    func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
        dismiss(animated: true, completion: nil)
    }
    
}
{% endcodeblock %} &nbsp;

回过头来，切换到 `Main.storyboard` 文件，保证你设置的 outlet 和 action 均已绑定完毕。

为了访问摄像头和照片库，还有最后一件我们必须要做的事。找到工程中的 `Info.plist` 文件，添加两个条目：*Privacy – Camera Usage Description* 和 *Privacy – Photo Library Usage Description*。从 iOS 10 开始，若要使用摄像头和照片库，你必须描述其使用原因。

{% asset_img images/coreml-plist-privacy.png %} &nbsp;

好了，就这样！马上，你就要进入到本教程的核心部分！再重复一遍，如果你不想从零开始构建 UI，可以**[下载出事工程模板](https://github.com/appcoda/CoreMLDemo/raw/master/CoreMLDemoStarter.zip)**。

## 整合 Core ML 数据模型

现在，我们来换换档，把 Core ML 数据模型整合进应用中。之前提到过，我们需要使用预处理过的数据模型，与 Core ML 协同工作。当然，你可以构建自己的模型，但在这个 demo 中，我们使用苹果开发者网站中提供的预处理模型。

访问苹果开发者网站中的**[机器学习](https://developer.apple.com/machine-learning/)**页面，滚动到页面的下方，会发现4种预处理的数据模型。

{% note warning %} 
<span style="color: #EEAC57;">**译者注：**在翻译该文时，苹果官网中所提供的数据模型已增加到 5 种。</span> 
{% endnote %} &nbsp;

{% asset_img images/coreml-pretrained-model.png %} &nbsp;

在本教程中，我们选择 *Inception v3* 模型，当然，你也可以去尝试其他模型。下载完成后，将它添加到工程中，看看它都显示了什么？

{% asset_img images/coreml-model-desc.png %} &nbsp;

{% note default %} 
**提示：**保证数据模型的 Target Membership 处勾选了当前工程，否则应用将无法访问该文件。
{% endnote %} &nbsp;

从上图中可以发现，当前模型为神经网络分类，另外，你还要注意的信息是 Model Evaluation Parameters，它说明了模型接受的输入参数的要求，以及输出内容的信息。当前模型接收 299x299 的图片作为输入，并返回可能性最高的分类描述，以及每种分类的可能性。

另外要注意的是 Model Class，它（Inceptionv3）是由机器学习模型所生成的，可以直接在代码中使用。如果你点击 `Inceptionv3` 类右侧的下一步箭头，就可以看到这个类的源代码。

{% asset_img images/inceptionv3-class.png %} &nbsp;

下面，我们来把数据模型加入到代码中。切换到 `ViewController.swift` 文件，在文件的最开始部分，导入 Core ML 框架。
 
{% codeblock lang:swift %}
import CoreML
{% endcodeblock %} &nbsp;

然后，定义一个 `Inceptionv3` 类型的变量，并在 `viewWillAppear()` 方法中进行初始化：

{% codeblock lang:swift %}
var model: Inceptionv3!

override func viewWillAppear(_ animated: Bool) {
    model = Inceptionv3()
}
{% endcodeblock %} &nbsp;

我知道你在想什么。

"好吧，Sai，你为什么不早点初始化它？"

"在 `viewWillAppear` 方法中定义它的重点是什么？"

好吧，朋友，重点就是，当你的应用尝试识别图像中的物体时，它会快很多！

如果我们回过头来看 `Inceptionv3.mlmodel`，你会发现，输入仅仅是一张尺寸为 `299x299` 的图片，那我们如何把图片转换成这个尺寸呢？别介，这就是下面我们要做的。

## 转换图像

把 `ViewController.swift` 文件的扩展部分的代码更新成如下所示，我们会实现 `imagePickerController(_:didFinishPickingMediaWithInfo)` 来处理所选的图片。

{% codeblock lang:swift %}
extension ViewController: UIImagePickerControllerDelegate {
    
    func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
        dismiss(animated: true, completion: nil)
    }
    
    func imagePickerController(_ picker: UIImagePickerController,
                               didFinishPickingMediaWithInfo info: [String : Any]) {
        
        picker.dismiss(animated: true)
        
        classifer.text = "Analyzing Image..."
        
        guard let image = info["UIImagePickerControllerOriginalImage"] as? UIImage else { return }
        
        UIGraphicsBeginImageContextWithOptions(CGSize(width: 299, height: 299), true, 2.0)
        image.draw(in: CGRect(x: 0, y: 0, width: 299, height: 299))
        
        let newImage = UIGraphicsGetImageFromCurrentImageContext()!
        UIGraphicsEndImageContext()
        
        let attrs = [kCVPixelBufferCGImageCompatibilityKey: kCFBooleanTrue,
                     kCVPixelBufferCGBitmapContextCompatibilityKey: kCFBooleanTrue] as CFDictionary
        
        var pixelBuffer: CVPixelBuffer?
        let status = CVPixelBufferCreate(kCFAllocatorDefault,
                                         Int(newImage.size.width),
                                         Int(newImage.size.height),
                                         kCVPixelFormatType_32ARGB,
                                         attrs,
                                         &pixelBuffer)
        
        guard status == kCVReturnSuccess else { return }
        
        CVPixelBufferLockBaseAddress(pixelBuffer!, CVPixelBufferLockFlags(rawValue: 0))
        let pixelData = CVPixelBufferGetBaseAddress(pixelBuffer!)
        
        let rgbColorSpace = CGColorSpaceCreateDeviceRGB()
        let context = CGContext(data: pixelData,
                                width: Int(newImage.size.width),
                                height: Int(newImage.size.height),
                                bitsPerComponent: 8,
                                bytesPerRow: CVPixelBufferGetBytesPerRow(pixelBuffer!),
                                space: rgbColorSpace,
                                bitmapInfo: CGImageAlphaInfo.noneSkipFirst.rawValue)
        
        context?.translateBy(x: 0, y: newImage.size.height)
        context?.scaleBy(x: 1.0, y: -1.0)
        
        UIGraphicsPushContext(context!)
        newImage.draw(in: CGRect(x: 0, y: 0, width: newImage.size.width, height: newImage.size.height))
        UIGraphicsPopContext()
        CVPixelBufferUnlockBaseAddress(pixelBuffer!, CVPixelBufferLockFlags(rawValue: 0))
        
        imageView.image = newImage
        
    }
    
}
{% endcodeblock %} &nbsp;

以上代码的作用：

1. **10-14 行**：在方法中的最初几行，我们从 `info` 字典中获取所选择的图片（使用 `UIImagePickerControllerOriginalImage` 键），另外，一旦我们选择完毕，就关闭 `UIImagePickerController`。
2. **16-20 行**：由于数据模型仅接受尺寸为 `299x299` 的图片，所以我们把所选图片转换成方形，接着，我们把这个方形图片赋值给另一个常量 `newImage`。
3. **22-33 行**：现在，我们把 `newImage` 转换成 `CVPixelBuffer`。你可能不太熟悉 `CVPixelBuffer`，其实，它就是内存中一块保存像素的图像缓存区。你可以**[在这](https://developer.apple.com/documentation/corevideo/cvpixelbuffer-q2e)**了解更多关于它的信息。
4. **47-48 行**：我们使用所有呈现图片的像素，并将其转化为依赖设备的 RGB 色彩空间。接着，使用所有数据创建 `CGContext`，当我们打算渲染（或更改）一些底层参数时，可以很轻松的调用它，也就是接下来两行代码所做的——转换和缩放图片。
5. **50-55 行**：最后，我们生成图像，移除栈顶的上下文内容，并将 `imageView.image` 的值设置为 `newImage`。

即使你不太了解上面的代码也没关系，这些都是高级的 `Core Image` 代码，而这并不在本教程所涵盖的范围内。你只需要知道，我们将图像转换成了数据模型可以接受的格式。我建议你修改其中的数字并观察最终的生成结果，来更好地理解上述代码。

## 使用 Core ML

让我们将焦点移回到 Core ML。我们使用 Inceptionv3 模型来实现物体识别，来吧，只需要几行代码而已。将下面的代码片段粘贴到 `imageView.image = newImage` 的下一行。

{% codeblock lang:swift %}
guard let prediction = try? model.prediction(image: pixelBuffer!) else { return }
classifer.text = "I think this is a \(prediction.classLabel)."
{% endcodeblock %} &nbsp;

搞定了！使用 `Inceptionv3` 类中的 `prediction(image:)` 方法来预测图像中的物体。我们将 `pixelBuffer`（重置大小的图像）作为参数传递给方法，一旦预测完成，预测结果会以字符串的形式返回，我们只要更新 `classifier` 的 `text` 属性，即可显示识别结果。

是时候来测试一下我们的应用了！编译代码并在模拟器中运行它，拍照或从照片库中选择一张照片，应用就会告诉你，它识别出了什么。

{% asset_img images/coreml-successful-case.jpg %} &nbsp;

在测试过程中，有时你可能会发现识别结果并不正确。这并非程序的问题，而是由于预处理数据模型造成的。

{% asset_img images/coreml-failed-case.jpg %} &nbsp;

## 总结

我希望你已经了解了如何将 Core ML 整合进应用中，这只是一片介绍性的教程，如果你有兴趣将预处理的 Caffe、Keras 或 SciKit 模型转换成 Core ML 模型，请继续关注 Core ML 系列教程的下一篇，我会告诉你如何转换它们。

你可以在 **[GitHub](https://github.com/appcoda/CoreMLDemo)** 上获取本教程的全部代码。

更多关于 Core ML 框架的细节，可通过访问**[苹果官方网站](https://developer.apple.com/documentation/coreml)**进行查询。你也可以参考 WWDC 2017 中关于 Core ML 的相关部分：

* **[Introducing Core ML](https://developer.apple.com/videos/play/wwdc2017/703/)**
* **[Core ML in Depth](https://developer.apple.com/videos/play/wwdc2017/710/)**

你对 Core ML 有何感想？请留言给我！

> [原文链接](https://www.appcoda.com/coreml-introduction)
