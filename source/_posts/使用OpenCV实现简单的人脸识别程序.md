---
title: 使用 OpenCV 实现简单的人脸识别程序
date: 2019-09-10 15:34:13
tags:
    - OpenCV
---

本问就 Mac 系统安装 OpenCV 以及实现一个简单的人脸识别程序进行记录。

### 安装 OpenCV

实际上，OpenCV 的安装方式比较多，这里为了避免一些第三方安装的问题，我们采用源代码方式安装。

安全前请确保本机已经安装了 CMake 和 Xcode。

我们去[ OpenCV 的网站](https://opencv.org/releases/) 下载源代码，选择 Release -> SourceCode，可以选择最新的 4.11 版本。

这里以 4.1.1 版本为例，下载后我们解压到 `opencv-4.1.1`，然后进入到该目录，新建一个 release 目录用于存放我们构建好的内容，并进入到该目录：

```
mkdir release
cd release/
```

然后我们依次执行以下命令安装：

```
cmake -G “Unix Makefiles” .. 
make
make install
```

全部命令执行成功后，实际上就安装完成了，我们可以从最后的输出中看到，相关内容已经被安装到了 `/usr/local/include`、`/usr/local/lib` 等文件夹下。

### 使用 Xcode 编写人脸识别程序

我们可以使用 Xcode 建立一个命令行程序，这里我们还需要处理两个问题：

* OpenCV 的引入
* 摄像头权限的获取

#### OpenCV 的引入

对于第一点，我们在 **Build Setting** 的 **Search Paths** 中增加 Header 和 Library 的路径：

![路径](/img/cv1.jpg)

然后我们需要在 **Build Phases** 的 **Link Binary With Libraries** 中增加动态链接库。

我们可以点击左下角加号，选择 `Add Others` 然后进入 `/usr/local/lib` 把 OpenCV 相关的库均包含进来即可。

>实际上我们可以部分引入，但是由于我们是初步上手，全部引入也可以。

#### 摄像头权限的获取

这里如果我们直接运行我们的程序，在 macOS 最新的系统中是无法运行通过的，这里涉及到摄像头权限问题。

一般来说，对于 macOS，我们需要在运行程序的目录下声明 `info.plist`, 这样程序在运行的时候系统会自动有申请权限的弹窗，对于我们测试场景下而言，我们可以这样做：

* 进入我们 Product 存放的目录（注意不是项目代码目录，可以在 Products 条目右单击 `Show in Finder`）
* 复制一个 info.plist（这里我们可以随便找一个本地安装的应用程序的 info.plist，一般右单击显示包内容即可看到）
* 在 info.plist 中增加 `NSCameraUsageDescription`，value 即提示语，可以写比如 `摄像头权限的获取`。

#### 书写并运行程序

做完上述准备工作后，我们可以写我们的人脸识别程序了，这里给出一个成功运行的代码示例（参考了网上的一些例子）：

```
#include <iostream>
#include <opencv2/opencv.hpp>

using namespace std;
using namespace cv;

void capture();

// 是否退出摄像头抓取线程
static bool g_quit = false;

int main(int argc, char** argv) {
    capture();
    return 0;
}

void capture()
{
    // 打开摄像头
    cv::VideoCapture cap(0);
    
    // 如果打开失败，返回错误
    if (!cap.isOpened())
    {
        cout<<"Open Capture Failed"<<endl;
        return;
    }
    
    // 人脸识别分类器
    cv::CascadeClassifier faceCascadeClassifier("/usr/local/share/opencv4/haarcascades/haarcascade_frontalface_alt2.xml");
    
    // 读取 Frame ，直到退出系统
    while (!g_quit)
    {
        cv::Mat frame;
        if (!cap.read(frame))
        {
            // 读取失败，返回错误
            break;
        }
        
        // 进行人脸识别
        std::vector<cv::Rect> faces;
        faceCascadeClassifier.detectMultiScale(frame, faces);
        // 将人脸识别结果绘制到图片上
        for (const auto& face : faces)
        {
            cout<<"Find Face"<<endl;
            cv::rectangle(frame,
                          cv::Point(face.x, face.y),
                          cv::Point(face.x + face.width, face.y + face.height),
                          CV_RGB(255, 0, 0),
                          2);
        }
        imshow("Display Image", frame);
        waitKey(100);
    }
}
```

这里值得注意的是，我们这里使用的人脸识别分类器是 OpenCV 安装后自带的，你本机的目录可能并不是这一个（这个路径实际上安装好 OpenCV 之后会打印在控制台）。

正常情况下，以上程序可以直接编译执行。