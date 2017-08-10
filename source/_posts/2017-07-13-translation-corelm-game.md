---
title: 使用 Core ML 创建简单游戏
date: 2017-07-13 10:14:38
categories: 译文
---

> [原文链接](https://www.appcoda.com/coreml-game)

WWDC 2017 中，iOS 推出了很多惊艳的 API，Core ML 就是最受关注的一个（当然，ARKit 也是相当酷炫的！）。Core ML 可以使开发者在不了解神经网络或机器学习算法等相关知识的前提下，轻松地在应用中使用机器学习模型。今天，我们来展示下如何用 Core ML 轻松地构建出一款游戏。我们将创建一款简单的拾荒者游戏，玩家需要在房间中转来转去，去寻找指定的物品。在开始之前，我建议读者先参考下我们之前发布的关于**[ Core ML 介绍的教程](http://www.appcoda.com/coreml-introduction)**。

<!-- more -->

{% note success %} 
<span style="color: #5FB760;">**提示：**本教程需要用到 Xcode 9 beta 版，另外，你还需要一部安装了 iOS 11 beta 版的设备，用来运行和测试应用，本次的应用不会在模拟器中运行，当然，你也可以使用像 iPhone 5s 这样的老款设备，但运行速度会有点慢。Xcode 9 beta 兼容 Swift 3.2 和 4.0 版本，以下代码全部基于 Swift 4.0 的语法完成。</span> 
{% endnote %} &nbsp;

## 应用概述

今天我们将要创建一款简单的应用，当然，玩法也很简单。当玩家点击 Start 按钮后，屏幕上方将随机出现物品名称，玩家的任务就是去找到它们。当玩家发现了指定的物品，只需将设备对准它，iPhone 会通过机器学习算法来识别它，成功找到一个物品后，会自动显示下一个物品。每次搜寻成功后，都会增加点数。另外，当玩家无法找到指定物品时，也可以选择跳过它。

本次制作的应用，在物品识别方面，与之前介绍 Core ML 的教程有些许差异，主要的区别是，我们将采用摄像头采集实时图像，而非简单地选择一张图片。

{% asset_img images/image-1.png %} &nbsp;

## 创建工程

打开 Xcode 9 创建工程，选择 *Single View App* 模板，虽然我们创建的是一款游戏，但使用 *Single View App* 模板已经足够了。将工程命名为 *CoreMLScavenge*，当然，你也可以选择你喜欢的名称。另外，确保开发语言为 *Swift*。

{% asset_img images/image-2.png %} &nbsp;

成功创建工程后，取消选择工程描述文件中的 `Landscape Left` 和 `Landscape Right` 选项，我们希望这款游戏只在垂直模式下运行。

{% asset_img images/image-3.png %} &nbsp;

## 创建用户界面

是时候来点有意思的了！在导航面板中找到 `Main.storyboard` 文件，并在 View Controller 的上方和下方添加两个 `View`，将它们的宽度拉伸至与 View Controller 同宽，高度固定为 85px。整个的背景用来做图像采集窗口，所以我们要在之前添加的 `View` 组件上添加按钮和标签。将背景颜色设置为浅灰色，这样就能防止我们误将其他组件添加到背景上了。

{% asset_img images/image-4.png %} &nbsp;

在上方的视图中添加两个 `UILabel` 组件，一个居中放置，另一个靠左。居中的标签用来显示所要寻找的物品，所以，尽量将它拉宽点，这样就有足够的空间显示文字了。而左边的标签仅仅用来显示两位数的得分，所以可以适当地缩小它。

{% asset_img images/image-5.png %} &nbsp;

在下方的视图中添加两个 `UILabel` 和两个 `UIButton` 组件，一个标签显示当前记录，另一个显示历史最高记录，两个按钮中的一个显示 "Start"，点击后将开始游戏，另一个则显示为 "Skip"，当玩家找不到指定物品时，点击该按钮可以跳过。

{% asset_img images/image-6.png %} &nbsp;

此处，我不准备介绍 Auto Layout 的相关设置了，我建议你自己完成。可以访问**[这里](http://www.appcoda.com/introduction-auto-layout/)**来了解如何使用 Auto Layout。如果你实在无法完成 Auto Layout 设置，那就确保在你要运行的设备上进行界面设计工作。

## 创建视图

UI 的工作已经完毕了，下面我们要开始编码了。选择 `ViewController.swift` 文件，在文件的开始部分导入所需的框架。

{% codeblock lang:swift %}
import MobileCoreServices
import Vision
import AVKit
{% endcodeblock %} &nbsp;

接下来添加 8 个 outlet 并将它们与 UI 关联起来。

{% codeblock lang:swift %}
@IBOutlet var scoreLabel: UILabel!
@IBOutlet var highscoreLabel: UILabel!
@IBOutlet var timeLabel: UILabel!
@IBOutlet var objectLabel: UILabel!
@IBOutlet var startButton: UIButton!
@IBOutlet var skipButton: UIButton!
@IBOutlet var topView: UIView!
@IBOutlet var bottomView: UIView!
{% endcodeblock %} &nbsp;

现在，你的代码看起来应该是这样的：

{% codeblock lang:swift %}
import UIKit
import MobileCoreServices
import Vision
import AVKit

class ViewController: UIViewController {
    
    @IBOutlet var scoreLabel: UILabel!
    @IBOutlet var highscoreLabel: UILabel!
    @IBOutlet var timeLabel: UILabel!
    @IBOutlet var objectLabel: UILabel!
    @IBOutlet var startButton: UIButton!
    @IBOutlet var skipButton: UIButton!
    @IBOutlet var topView: UIView!
    @IBOutlet var bottomView: UIView!

    override func viewDidLoad() {
        super.viewDidLoad()
    }

    override func didReceiveMemoryWarning() {
        super.didReceiveMemoryWarning()
    }

}
{% endcodeblock %} &nbsp;

我们再来添加一些全局变量，把它们加在 outlet 的下方即可。

{% codeblock lang:swift %}
var cameraLayer: CALayer!
var gameTimer: Timer!
var timeRemaining = 60
var currentScore = 0
var highScore = 0
{% endcodeblock %} &nbsp;

让我们来一一解释下它们：
**第 1 行：**摄像头对应的层，后面会把它添加到 `View` 上并充满整个屏幕。
**第 2 行：**稍后，我们将创建一个游戏计时器，把它定义为一个全局变量，我们就可以在任何方法中使它失效（停止计时）。
**第 3 行：**这个变量用来保存剩余时间，它的初始值为 60，即游戏能持续 1 分钟。你可以任意修改它的值，让游戏时间更长或更短。
**第 4 行：**这个变量用来保存用户的记录，没找到一件物品后，记录的分数就会加 1。
**第 5 行：**这个变量用来保存用户的最高记录，每当应用启动后，我们会从 UserDefaults 中获取它，并把它赋值给当前变量。

将大量的代码都放入 `viewDidLoad` 方法中，会使代码结构相当凌乱，为此，我们会在 `viewDidLoad` 方法下方添加一个 `viewSetup` 方法，用来处理基本的 UI 设置。

将下面的内容添加到 `viewDidLoad` 方法中：

{% codeblock lang:swift %}
viewSetup()
{% endcodeblock %} &nbsp;

在 `viewDidLoad` 方法的下方添加如下代码：

{% codeblock lang:swift %}
func viewSetup() {
    let backgroundColor = UIColor(red: 255/255, green: 255/255, blue: 255/255, alpha: 0.8)
    topView.backgroundColor = backgroundColor
    bottomView.backgroundColor = backgroundColor
    scoreLabel.text = "0"
}
{% endcodeblock %} &nbsp;

上述代码的作用，是将上方和下方的视图背景设置为半透明，我们可以隐约它们后面的视频采集画面，另外，我们还设置了记录的分数为 0。

现在，你的代码看起来应该是这样的：

{% codeblock lang:swift %}
import UIKit
import MobileCoreServices
import Vision
import AVKit

class ViewController: UIViewController {
    
    @IBOutlet var scoreLabel: UILabel!
    @IBOutlet var highscoreLabel: UILabel!
    @IBOutlet var timeLabel: UILabel!
    @IBOutlet var objectLabel: UILabel!
    @IBOutlet var startButton: UIButton!
    @IBOutlet var skipButton: UIButton!
    @IBOutlet var topView: UIView!
    @IBOutlet var bottomView: UIView!
    
    var cameraLayer: CALayer!
    var gameTimer: Timer!
    var timeRemaining = 60
    var currentScore = 0
    var highScore = 0
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        viewSetup()
    }
    
    func viewSetup() {
        let backgroundColor = UIColor(red: 255/255, green: 255/255, blue: 255/255, alpha: 0.8)
        topView.backgroundColor = backgroundColor
        bottomView.backgroundColor = backgroundColor
        scoreLabel.text = "0"
    }

    override func didReceiveMemoryWarning() {
        super.didReceiveMemoryWarning()
    }

}
{% endcodeblock %} &nbsp;

## 创建摄像头

现在我们来创建实时视频采集窗口，它将占用整个背景视图。在我们添加代码前，需要向用户申请访问摄像头的权限，iOS 会完成大部分的工作，我们所要做的只是描述下使用摄像头的原因。

在导航面板中找到 `Info.plist` 文件并添加一行条目，其键为 *Privacy – Camera Usage Description*，值为描述文字。

{% asset_img images/image-7.png %} &nbsp;

下面来添加代码，我们把设置摄像头的准备工作放到一个叫 `cameraSetup` 的方法中。

把这段代码加在 `viewDidLoad` 方法中的 `viewSetup` 方法下面：

{% codeblock lang:swift %}
cameraSetup()
{% endcodeblock %} &nbsp;

然后，在 `viewDidLoad` 方法后面添加 `cameraSetup` 方法的定义：

{% codeblock lang:swift %}
func cameraSetup() {
    
}
{% endcodeblock %} &nbsp;

现在，我们来创建 `AVCaptureSession`，通过它，我们就可以进行实时视频扑捉了。把下面的方法添加到 `cameraSetup` 方法中：

{% codeblock lang:swift %}
let captureSession = AVCaptureSession()
captureSession.sessionPreset = AVCaptureSession.Preset.photo
let backCamera = AVCaptureDevice.default(.builtInWideAngleCamera, for: .video, position: .back)!
let input = try! AVCaptureDeviceInput(device: backCamera)
captureSession.addInput(input)
{% endcodeblock %} &nbsp;

这里究竟发生了什么？
**第 1 行：**我们创建了一个 `AVCaptureSession` 类型的常量。
**第 2 行：**为保证输出品质，我们使用预设值的选项，将其设置为照片格式，这样就能使其达到更高的分辨率。
**第 3 行：**我们采用后置摄像头创建了一个 `AVCaptureDevie`，这里，我们根本没理由使用前置摄像头。 
**第 4 行：**我们使用刚刚创建的 `AVCaptureDevice` 创建了一个 `AVCaptureDeviceInput` 对象。
**第 5 行：**我们将 backCamera 作为 captureSession 的输入。

还记得我们之前创建的 `cameraLayer` 参数吗？马上我们就要用到它了，把 `cameraLayer` 作为主视图的一个子层，并设置它的大小与主视图相同。

将下面的代码添加到我们刚刚设置 `captureSession` 的那部分代码后：

{% codeblock lang:swift %}
cameraLayer = AVCaptureVideoPreviewLayer(session: captureSession)
view.layer.addSublayer(cameraLayer)
cameraLayer.frame = view.bounds
 
view.bringSubview(toFront: topView)
view.bringSubview(toFront: bottomView)
{% endcodeblock %} &nbsp;

**第 1-3 行：**初始化一个 `AVCaptureVideoPreviewLayer` 类型的对象，并赋值给 `cameraLayer`，将 `captureSession` 作为构造方法的参数。然后，我们将 `cameraLayer` 作为子层添加到主视图的层中，并设置其大小为主视图的大小。
**第 5-6 行：**这里，我们将上方和下方的两个视图，从视图体系中上前移动，这样，`cameraLayer` 就不会遮盖住它们。

在我们刚刚添加的代码后面，添加如下代码，这样就完成了这个方法：

{% codeblock lang:swift %}
let videoOutput = AVCaptureVideoDataOutput()
videoOutput.setSampleBufferDelegate(self, queue: DispatchQueue(label: "buffer delegate"))
videoOutput.recommendedVideoSettings(forVideoCodecType: .jpeg, assetWriterOutputFileType: .mp4)
 
captureSession.addOutput(videoOutput)
captureSession.sessionPreset = .high
captureSession.startRunning()
{% endcodeblock %} &nbsp;

**第 1-3 行：**创建数据输出对象，并设置输出参数。此时，你会发现一些编译错误，不用担心，稍候我们就会解决它。
**第 5-7 行：**最后，我们将数据输出对象添加到 `captureSession` 中，并启动它。

`cameraSetup` 方法现在看起来应该是这个样子：

{% codeblock lang:swift %}
func cameraSetup() {
    let captureSession = AVCaptureSession()
    captureSession.sessionPreset = AVCaptureSession.Preset.photo
    let backCamera = AVCaptureDevice.default(.builtInWideAngleCamera, for: .video, position: .back)!
    let input = try! AVCaptureDeviceInput(device: backCamera)
    captureSession.addInput(input)
    
    cameraLayer = AVCaptureVideoPreviewLayer(session: captureSession)
    view.layer.addSublayer(cameraLayer)
    cameraLayer.frame = view.bounds
    
    view.bringSubview(toFront: topView)
    view.bringSubview(toFront: bottomView)
    
    let videoOutput = AVCaptureVideoDataOutput()
    videoOutput.setSampleBufferDelegate(self, queue: DispatchQueue(label: "buffer delegate"))
    videoOutput.recommendedVideoSettings(forVideoCodecType: .jpeg, assetWriterOutputFileType: .mp4)
    
    captureSession.addOutput(videoOutput)
    captureSession.sessionPreset = .high
    captureSession.startRunning()
}
{% endcodeblock %} &nbsp;

## 添加 Core ML 数据模型

完成了 UI 和摄像头的设置，是时候来编写使用 Core ML 进行物体识别的代码了。

在编码前，我们需要将 Core ML 数据模型添加到工程中，为了使用 Core ML，你需要一个预处理的数据模型，虽然可以为这个游戏建立自己的数据模型，但我们还是决定使用苹果开发者网站中所提供的。

访问**[苹果开发者网站中关于机器学习的页面](https://developer.apple.com/machine-learning/)**，滚动到页面下方，你会发现 4 种不同的预处理数据模型可供下载。

{% asset_img images/image-8.png %} &nbsp;

{% note warning %} 
<span style="color: #EEAC57;">**译者注：**在翻译该文时，苹果官网中所提供的数据模型已增加到 5 种。</span> 
{% endnote %} &nbsp;

在这个游戏中，我们使用 *Inception v3* 模型，下载完毕后，将其拖拽至工程的导航面板中，点击数据模型文件，看看它都显示了什么。

{% asset_img images/image-8.png %} &nbsp;

{% note default %} 
**提示：**保证数据模型的 Target Membership 处勾选了当前工程，否则应用将无法访问该文件。
{% endnote %} &nbsp;

你会发现，这个数据模型有一个神经网络分类器，它将尺寸为 `299x299` 的图片作为输入，并输出两部分内容：一个表示所识别品类的可能性的字典，和一个表示所识别品类的字符串信息。

## 识别物体

我们的短途旅行结束了，下面我们继续回到 `View Controller` 中，并添加两个神奇的方法。

首先添加 `predict` 方法，把如下代码添加到 `cameraSetup` 方法下方：

{% codeblock lang:swift %}
func predict(image: CGImage) {
    let model = try! VNCoreMLModel(for: Inceptionv3().model)
    let request = VNCoreMLRequest(model: model, completionHandler: results)
    let handler = VNSequenceRequestHandler()
    try! handler.perform([request], on: image)
}
{% endcodeblock %} &nbsp;

不要被它的代码行数骗了，它可是一个相当给力的方法：

**第 2 行：**创建一个 Inception v3 类型的常量。
**第 3 行：**创建一个 `VNCoreMLRequest` 类型的对象，这个对象会在请求完成时调用结果处理方法（稍候我们就会实现它，所以不要在意那些编译错误）。
**第 4-5 行：**创建一个 `VNCoreMLRequestHandler` 类型的对象并调用它，而 `predict` 方法的输入参数 `image` 也将作为 `perform` 方法的参数被传递进去。

下面，我们来实现 `results` 方法，通过它，我们来处理 `predict` 方法的处理结果，并使程序正常运行。

将下面的代码添加到 `predict` 方法下方：
 
{% codeblock lang:swift %}
func results(request: VNRequest, error: Error?) {
    guard let results = request.results as? [VNClassificationObservation] else {
        print("No result found")
        return
    }
        
    guard results.count != 0 else {
        print("No result found")
        return
    }
        
    let highestConfidenceResult = results.first!
    let identifier = highestConfidenceResult.identifier.contains(", ") ? String(describing: highestConfidenceResult.identifier.split(separator: ",").first!) : highestConfidenceResult.identifier
        
    if identifier == objectLabel.text! {
        currentScore += 1
        //nextObject()
    }
}
{% endcodeblock %} &nbsp;

这个方法会在物体识别完成后调用。首先使用两个 `guard` 语句保证结果的存在性，如果结果存在，则把可信度最高的结果转换成字符串，并赋值给 `identifier` 常量。然后，程序会比对结果字符串与目标物体标签中显示的内容，如果相同，就意味着玩家找到了正确的物体，分数也会随之增加。另外，你会发现一个被注释掉的方法 `nextObject()`，我们暂时还没实现它。

好了，我们已经完成了这个游戏的主要工作！是不是很简单？你的 `predict` 方法和 `results` 方法看起来应该是这样的：

{% codeblock lang:swift %}
func predict(image: CGImage) {
    let model = try! VNCoreMLModel(for: Inceptionv3().model)
    let request = VNCoreMLRequest(model: model, completionHandler: results)
    let handler = VNSequenceRequestHandler()
    try! handler.perform([request], on: image)
}

func results(request: VNRequest, error: Error?) {
    guard let results = request.results as? [VNClassificationObservation] else {
        print("No result found")
        return
    }
    
    guard results.count != 0 else {
        print("No result found")
        return
    }
    
    let highestConfidenceResult = results.first!
    let identifier = highestConfidenceResult.identifier.contains(", ")
        ? String(describing: highestConfidenceResult.identifier.split(separator: ",").first!)
        : highestConfidenceResult.identifier
    
    if identifier == objectLabel.text! {
        currentScore += 1
        //nextObject()
    }
}
{% endcodeblock %} &nbsp;

在完成这个游戏前，我们还有一件事要做，我们需要为 `ViewController` 添加扩展，来调用 `predict` 方法。将下面的代码添加到 `ViewController.swift` 文件的最底部：

{% codeblock lang:swift %}
extension ViewController: AVCaptureVideoDataOutputSampleBufferDelegate {
    func captureOutput(_ output: AVCaptureOutput, didOutput sampleBuffer: CMSampleBuffer, from connection: AVCaptureConnection) {
        guard let pixelBuffer = CMSampleBufferGetImageBuffer(sampleBuffer) else { fatalError("pixel buffer is nil") }
        let ciImage = CIImage(cvPixelBuffer: pixelBuffer)
        let context = CIContext(options: nil)
        
        guard let cgImage = context.createCGImage(ciImage, from: ciImage.extent) else { fatalError("cg image") }
        let uiImage = UIImage(cgImage: cgImage, scale: 1.0, orientation: .leftMirrored)
        
        DispatchQueue.main.sync {
            predict(image: uiImage.cgImage!)
        }
    }
}
{% endcodeblock %} &nbsp;

这段扩展代码使 `ViewController` 适配 `AVCaptureVideoDataOutputSampleBufferDelegate` 协议，从而处理捕捉到的视频缓冲，并根据视频内容创建图像对象。我们把图像作为参数，传递给 `predict` 方法。现在，你可以正常编译你的代码了。这就是我们如何通过不断采集视频，并传递给内置的机器学习数据模型，来达到物体识别功能的全过程。

## 为游戏预制物体数组

为了让游戏能够随机显示目标物体，我们准备了一个待选物体列表。我们创建了一个列表，来保存一批待选物体的名称，随后，我们会创建一个方法来随机选择物体，并告知玩家目标物体是什么。

这是一个有点长的数组，处于代码结构性的考虑，我把它放到了一个单独的 Swift 文件中，在导航面板中点击右键，并选择 `New File...`。

选择 Swift 文件类型并点击下一步按钮，将其命名为 "Objects"，将如下代码片段添加到刚刚创建的文件中：

{% codeblock lang:swift %}
struct Objects {
    let objectArray = ["computer keyboard", "mouse", "iPod", "printer", "digital clock", "digital watch", "backpack", "ping-pong ball", "envelope", "water bottle", "combination lock", "lampshade", "switch", "lighter", "pillow", "spider web", "sandal", "vacuum", "wall clock", "bath towel", "wallet", "poster", "chocolate"]
}
{% endcodeblock %} &nbsp;

我们创建了一个名为 `objectArray` 的数组对象，它包含了可随机选择的常用日用品，以供 Core ML 进行查找。当然，你可以任意增减这些选项。

在进行下一步之前，让我们先回到 `ViewController.swift` 文件。

## 保存记录

我们会使用 `UserDefaults` 保存用户的最高记录，这就意味着，我们需要一个 setter 把最高纪录保存到 `UserDefaults` 中，还需要一个 getter 方法去获得最高记录的值。

将下面两个方法添加到 `results` 方法的下方：

{% codeblock lang:swift %}
func getHighScore() {
    if let score = UserDefaults.standard.object(forKey: "highscore") {
        highscoreLabel.text = "\(score)"
        highScore = score as! Int
    }
    else {
        print("No highscore, setting to 0.")
        highscoreLabel.text = "0"
        highScore = 0
        setHighScore(score: 0)
    }
}
    
func setHighScore(score: Int) {
    UserDefaults.standard.set(score, forKey: "highscore")
}
{% endcodeblock %} &nbsp;

`getHighScore` 方法使用 `if let` 语句检查 `UserDefaults` 中是否存在被保存的最高记录，就把它赋值给 `highScore` 变量，如果不存在，则将 `highScore` 变量的值设为 0。`setHighScore` 只是把当前的记录设置为最高记录，当玩家刷新了最高记录后，我们将会调用它。

当应用启动时，我们需要调用 `getHighScore` 方法，这样，在打开应用后，玩家就能看到最高记录是多少。

将 `getHighScore` 方法添加到 `viewDidLoad` 方法中 `cameraSetup` 方法的后面。你的 `viewDidLoad` 方法看起来应该是这样的：

{% codeblock lang:swift %}
override func viewDidLoad() {
    super.viewDidLoad()
    
    viewSetup()
    cameraSetup()
    getHighScore()
}
{% endcodeblock %} &nbsp;

## 处理游戏的运行

现在，我们来让之前完成的那些方法协同工作，这样就能使游戏运行起来了。我们会通过在物品数组里随机挑选一件物品来启动游戏，还要处理游戏何时结束。

将下面的代码添加到 `setHighScore` 方法下面：

{% codeblock lang:swift %}
//1
func endGame() {
    //2
    startButton.isHidden = false
    skipButton.isHidden = true
    objectLabel.text = "Game Over"
    //3
    if currentScore > highScore {
        setHighScore(score: currentScore)
        highscoreLabel.text = "\(currentScore)"
    }
    //4
    currentScore = 0
    timeRemaining = 60
    
}

//5
func nextObject() {
    //6
    let allObjects = Objects().objectArray
    //7
    let randomObjectIndex = Int(arc4random_uniform(UInt32(allObjects.count)))
    //8
    guard allObjects[randomObjectIndex] != objectLabel.text else {
        nextObject()
        return
    }
    //9
    objectLabel.text = allObjects[randomObjectIndex]
    scoreLabel.text = "\(currentScore)"
}
{% endcodeblock %} &nbsp;

你会问，这些方法都做了些什么？

1. 当游戏时间结束时，我们会调用 `endGame` 方法。
2. 首先，我们显示 start 按钮，并隐藏 skip 按钮，因为游戏在非进行状态下是不需要 skip 按钮的，另外，我们还要设置 object 标签的文本为 "Game Over"。
3. 我们要检查下玩家的记录是不是超过了最高记录，如果超过了，就调用 `setHighScore` 方法。
4. 为下一局游戏重置所有变量的值。
5. 当玩家找到了正确的物品，或者点击了 `skip` 按钮，则调用 `nextObject` 方法，这个方法会从物品数组中随机挑选物品，并显示在 object 标签上，这样，玩家就知道该去寻找什么了。
6. 创建 `objectArray` 变量。
7. 生成一个随机数用于定位数组中的元素，该随机数的选取范围为 0 到 `objectArray` 数组长度之间的值。
8. 使用 `guard` 语句来保证本次所选的物品与上一次的物品不同，这样就能避免玩家连续找两件同样的物品了。
9. 将 object 标签的值设置为本次所选物品的值，另外还要将 score 标签的值设置成正确的值。

{% note default %} 
**提示：**现在，我们已经实现了 `nextObject` 方法，请记得将 `results` 方法中的的注释去掉，还有其他使用到 `nextObject` 方法的地方。
{% endnote %} &nbsp;

下面，我们来创建两个 action 方法，并把它们与 start 和 skip 按钮关联起来。在 `nextObject` 方法的下面添加如下代码：

{% codeblock lang:swift %}
@IBAction func startButtonTapped() {
    //1
    gameTimer = Timer.scheduledTimer(withTimeInterval: 1, repeats: true, block: { (gameTimer) in
        //2
        guard self.timeRemaining != 0 else {
            gameTimer.invalidate()
            self.endGame()
            return
        }
            
        self.timeRemaining -= 1
        self.timerLabel.text = "\(self.timeRemaining)"
    })
    //3  
    startButton.isHidden = true
    skipButton.isHidden = false
    nextObject()
        
}
 
//4  
@IBAction func skipButtonTapped() {
    nextObject()
}
{% endcodeblock %} &nbsp;

我们在这都做了些什么？

1. 首先我们添加了一个 action，当玩家点击 start 按钮时就会调用它。我们初始化了一个计时器，并赋值给 gameTimer 功能。我们使用代码块作为计时器的事件处理逻辑，当然也可以使用 selector，但这样会使代码看起来更整洁。
2. 我们使用 `guard` 语句，来确保游戏时间尚有剩余，如果游戏剩余时间为 0，则使 gameTimer 失效，并调用 `endGame` 方法，如果游戏时间还有剩余，则将 `timeRemaining` 属性的值减 1，并更新 `timerLabel` 显示的信息。
3. 隐藏 `start` 按钮并显示 `skip` 按钮，调用 `nextObject` 方法来显示第一个目标物体。
4. 我们为 `skip` 按钮添加了一个 `action`，它的功能仅仅是调用 `nextObject` 方法。

{% note default %} 
**提示：**请确保你已经 action 和 outlet 与 `Main.storyboard` 中的 UI 组件关联完毕。 
{% endnote %} &nbsp;

## 运行游戏

终于到了运行游戏的时间了，编译并启动应用。点击 start 按钮并对准目标物体，看看你能得多少分！在定位和识别物体时，你可能要等几秒钟，另外，如果你是在老设备上运行游戏，比如 iPhone 5s、iPhone 6，由于机能限制，程序的运行速度会很慢，我发现运行在 iPhone 7 上效果就会很不错，但运行在 iPhone 5s 上就会慢得要死……

{% asset_img images/image-10.png %} &nbsp;

*如你所料，它认不出任天堂 Switch 游戏机！*

{% note warning %} 
<span style="color: #EEAC57;">**译者注：**此处，作者是在搞笑，目标物体实际上是开关……</span> 
{% endnote %} &nbsp;

## 总结

愿你在创建游戏的过程中心情愉快，同样，也希望你对 Core ML 有了更多的了解。这个游戏并不是那么完美，有很多地方有待你去改进（比如在找寻到目标物体后播放音效），这仅仅是为了演示如何使用 Core ML 的一个小程序，你可以在它的基础上尽情发挥想象力。

你可以在**[GitHub](https://github.com/appcoda/CoreMLScavenge)**下载本教程的完整代码。

更多关于 Core ML 框架的细节，可以访问**[关于 Core ML 的官方文档](https://developer.apple.com/documentation/coreml)**。

如你喜欢这篇教程，欢迎留言给我。
