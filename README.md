# 图像去雾网络 FFA-Net

本项目使用FFA-Net实现了对带雾图像的去雾处理, 在进行模型推理的过程中, 尽量使用```./test_img/```中提供的图像.

## 1.原理简介

本项目所用模型来自论文[FFA-Net: Feature Fusion Attention Network for Single Image Dehazing](https://arxiv.org/abs/1911.07559)(AAAI 2020)

源码参考: [FFA-Net](https://github.com/zhilin007/FFA-Net)

### 1.1 去雾任务问题描述

图像去雾的是从被破坏的输入中恢复出干净的图像, 大气散射模型为雾霾效应提供了一个简单的近似:
$$I(z) = J(z)t(z) + A\left( {1 - t(z)} \right)$$

其中, $I(z)$为观测到的带雾图像, $A$为全球大气光系数, $t(z)$为介质透射函数, $J(z)$为原始的无雾图像. 此外, $t(z)=e^{-βd(z)}$, 其中$β$和$d(z)$分别为大气散射参数和场景深度. 大气散射模型表明图像去雾是一个不知道$A$和$t(z)$的欠确定问题, 因此上式可以表示为:
$$J(z)=(I(z)-A)/t(z) +A $$
因此, 可以通过对$t(z)$和$A$的估计, 就能从带雾图像$I(z)$中, 恢复原始图像$J(z)$.

<div align=center>
    <img style="width:80%;
    height:80%;
    border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://user-images.githubusercontent.com/71765492/172847823-113a5995-f61f-4245-9872-fbc915e60037.png">
</div>
<p align="center">图片出自论文</p>

### 1.2 算法模型

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://github.com/Liuzy0908/Images/blob/main/FFA-Net%20model.png?raw=true">
</center>


<p align="center">模型结构</p>

模型的总体结构如上图所示。模型的输入是带雾图片。经过初步卷积后，进入模型的主要结构，由多个Group大模块构成。每个Group大型模块包含多个Block小模块。这里的大模块个数及每个大模块数包含的小模块个数都是可调参数，训练的模型的Group数为3，Block数为20。经过了Group大型模块后，将每个Group模块的输出用concat层相叠加，就得到了模型关键的中间输出。该中间输出经过该模型设计的特殊模块:通道注意力块 CA 和像素注意力块 PA 再进行处理后，经过两层简单的卷积层，就得到了该图像的修正图, 最终修正图与输入图相加就得到了输出的去雾图像 —— [FFA-Net图像去雾模型](https://aistudio.baidu.com/aistudio/projectdetail/3887034).

## 2. 项目文件结构

``` text
│  data_utils.py      # 数据处理工具程序
│  main.py	      # 训练主程序
│  metrics.py	      # 计算评价指标
│  option.py	      # 参数设置
│  README.md          # 项目说明
│  run.sh             # 自动化训练
│  test.py            # 模型推理程序
│  test.sh            # 自动化推理
│  server.py          # Web服务端
│
├─models              # 模型文件夹
│    FFA.py           # FFA-Net模型
│
├─pred_FFA_ots        # 推理结果文件夹
├─templates           # html文件夹
│    error.html       # 报错页面
│    home.html        # 主页面
│    show.html        # 推理结果展示页面
│
└─test_imgs           # 测试图片
```



## 3. 系统架构

项目使用 Flask 来搭建 Web 应用，连通用户前端，后端服务，和算法推理。

`server.py` 采用 `Flask` 在服务器端启动过一个 Web 服务。这个 FLASK 服务器前接用户来自浏览器的请求，后接用于推理图片结果的 Dehze对象 。整体架构如下所示：

```text
┌───────┐           ┌───────┐        ┌─────────┐
│       │           │       │        │         │
│       ├───────────►       ├────────►         │
│ USER  │           │ FLASK │        │  Dehaze │
│       ◄───────────┤       ◄────────┤         │
│       │           │       │        │         │
└───────┘           └───────┘        └─────────┘
```

### 3.1. 前端

前端实现代码在```templates/home.html```中，它提供了按钮```<上传图片>```用于用户拍照/上传图片：

```html
<div class="upload">
    <input type="file" id="file" name="photo" multiple @change="upload">
</div>
```

当用户拍照/上传图片后，通过```提交form表单```将图片通过 POST 请求发送给后端。

```html
<form method="POST" action="{{ base_dir+"/dehaze"}}" enctype="multipart/form-data">
    <div class="dehze">
        <input type="submit" value="提交图片" class="button-new" style="margin-top:15px;"/>
    </div>
</form>
```

后端接收用户上传图片，传入调用模型，产生推理结果(即上色后的图像)，将其更新到前端页面上。

```python
return render_template('show.html', base64_str=base64_str, base_dir=base_dir)
```

### 3.2. 后端

在 server.py 中，注册了路由```'/'```:

```python
@app.route("/", methods=["GET"])
#...
```

当接受 GET 请求时，返回主页面（home.html）

### 3.3. 推理

输入待识别的图片，模型进行推理并输出识别结果:

```python
@app.route("/dehaze", methods=["POST"])
def up_photo():
    if request.files["photo"].filename == '':
        return render_template("error.html"), 500

	'''推理过程'''
    haze = request.files["photo"]
    haze = Image.open(haze).convert('RGB')
    haze1 = tfs.Compose([
        tfs.ToTensor(),
        tfs.Normalize(mean=[0.64, 0.6, 0.58], std=[0.14, 0.15, 0.152])
    ])(haze)[None, ::]

    with flow.no_grad():
        pred = net(haze1.cuda())

	'''将推理后的结果转为图片展示给用户'''
    ts = flow.squeeze(pred.clamp(0, 1).cpu())
    img_nparr = np.transpose(np.array(ts), (1, 2, 0))
    img_dehazed_Image = Image.fromarray(np.uint8(img_nparr*255))

    output_buffer = BytesIO()
    img_dehazed_Image.save(output_buffer, format="png")
    byte_data = output_buffer.getvalue()
    base64_str = base64.b64encode(byte_data).decode()

    return render_template('show.html', base64_str=base64_str, base_dir=base_dir)
```

## 4. 使用方法

本项目支持训练与推理功能. 具体操作步骤如下:

1. 在项目详情页, 点击```Fork```按钮, 弹出 Fork 项目基本信息，点击【提交】即成功 Fork 本项目. 此时项目成为```我的项目```.

2. fork成功后点击```运行```按钮, 待项目运行后点击```进入```按钮

3. 成功后会进入网页版vscode, 使用快捷键``` ctrl + ` ```打开终端命令行:

>  在终端中输入```sh run.sh -s```即可执行Web端的推理功能<br/>
>  在终端中输入```sh run.sh -t```即可执行服务器端的推理功能<br/>
>  在终端中输入```sh run.sh -m```即可执行服务器端的训练功能<br/>

4. 在利用训练完毕的模型进行Web端推理时, 需要在```新建网页```中打开测试端口, 测试端口的获取方式如下图所示:

<div align=center>
    <img style="width:100%;
    height:100%;
    border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://user-images.githubusercontent.com/71765492/175498678-14be17b1-132d-4bc3-a4a6-142f7e2e5fc6.png">    
</div>


* 新建浏览器窗口, 键入上述网址

* 点击```上传图片```按钮, 上传带雾图片(使用```./test_img/```中提供的测试图像会有更好的效果)

* 点击```提交图片```按钮, 进行模型推理. 待推理完成后, 页面自动刷新, 展示推理后的效果图片

<div align=center>
    <img style="width:30%;
    height:30%;
    border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://user-images.githubusercontent.com/71765492/175318192-621d809e-fac8-4ab2-a0e0-60beae7c7da5.png">
    <img style="width:30%;
    height:30%;
    border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://user-images.githubusercontent.com/71765492/175318348-0b1c8e21-8550-4b54-aab6-114709b10bb1.png">
</div>


## 5. 写在最后

在训练过程中发现FFA-Net程序对于训练集以外的带雾图像(尤其是随手拍摄的真实雾天图像)推理结果并不优秀, 可能的原因有:

1. 模型的泛化性能力不足
2. 训练时间, 数据集以及所用设备不足
3. 其它原因导致的过拟合

本项目的前端代码及描述参考了: [黑白老图片自动上色](https://www.oneflow.cloud/drill/#/project/public/code?id=62e5061c42743444ea4af2691cdfb5ed)
