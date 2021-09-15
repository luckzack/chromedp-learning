# Golang爬虫终极杀器——Chromedp让你成为二维码登陆终结者（教程）

> [Github源码 - chromedp](https://github.com/chromedp/chromedp)

# 1 Chromedp是什么
`chromedp`是一个`更快`、`更简单`的Golang库用于调用支持`Chrome DevTools协议`的浏览器，同时不需要额外的依赖（例如`Selenium`和`PhantomJS`）
> Chrome和Golang都与Google有着相当密切的关系，而Chrome DevTools其实就是Chrome浏览器按下F12之后的控制终端

# 2 为什么不使用Selenium

对于`Golang`开发来说，使用`chromedp`更为便捷，因为它仅仅需要`Chrome`浏览器而并不需要依赖`ChromeDriver`，省去了依赖问题，有助于自动化的构建和多平台架构的迁移

# 3 文章解决了什么需求
1. 如何使用`chromedp`进行二维码登陆
2. 如何将二维码展示在无图形化的终端上（`makiuchi-d/gozxing`解码 `skip2/ go-qrcode`编码）
3. 如何保存Cookies实现短时间免登陆

> 网站会更新，文章不保证更新，请务必学会举一反三

# 4 如何使用chromedp进行二维码登陆

## 4.1 安装chromedp
1. 下载并安装Chrome浏览器
2. 创建Golang项目，开启`Go Module`(在项目目录下使用终端输入`go mod init`)
3. 在项目目录下使用终端输入：`go get -u github.com/chromedp/chromedp`（如果有依赖问题请删除`-u`）

## 4.2 尝试打开网站（以金山文档`https://account.wps.cn/`为例）
1. 重新设置chromedp使用`"有头"`的方式打开，以便于我们进行`debug`
```golang
func main(){
    // chromdp依赖context上限传递参数
	ctx, _ := chromedp.NewExecAllocator(
		context.Background(),

		// 以默认配置的数组为基础，覆写headless参数
		// 当然也可以根据自己的需要进行修改，这个flag是浏览器的设置
		append(
			chromedp.DefaultExecAllocatorOptions[:],
			chromedp.Flag("headless", false),
		)...,
	)
}
```

2. 创建chromedp上下文对象
```golang
func main(){
    // chromdp依赖context上限传递参数
    ...
    
    // 创建新的chromedp上下文对象，超时时间的设置不分先后
    // 注意第二个返回的参数是cancel()，只是我省略了
	ctx, _ = context.WithTimeout(ctx, 30*time.Second)
	ctx, _ = chromedp.NewContext(
		ctx,
		// 设置日志方法
		chromedp.WithLogf(log.Printf),
	)

	// 通常可以使用 defer cancel() 去取消
	// 但是在Windows环境下，我们希望程序能顺带关闭掉浏览器
	// 如果不希望浏览器关闭，使用cancel()方法即可
	// defer cancel()
	defer chromedp.Cancel(ctx)
}
```

3. 执行自定义的任务
```golang
func main(){
    // chromdp依赖context上限传递参数
    ...
    
    // 创建新的chromedp上下文对象，超时时间的设置不分先后
    // 注意第二个返回的参数是cancel()，只是我省略了
    ...
    
    // 执行我们自定义的任务 - myTasks函数在第4步
	if err := chromedp.Run(ctx, myTasks()); err != nil {
		log.Fatal(err)
		return
	}
}
```

4. 至此程序的初始化过程已经完成，接下来就是任务——打开登陆页面
```golang
// 自定义任务
func myTasks() chromedp.Tasks {
	return chromedp.Tasks{
		// 1. 打开金山文档的登陆界面
		chromedp.Navigate(loginURL),
	}
}
```

5. 运行一下程序，可以看到Chrome被打开，同时访问了我们指定的网站

![输入图片说明](https://images.gitee.com/uploads/images/2020/1108/173058_2d4a5939_2051718.png "1.png")

## 4.3 获取二维码（点击过程）

1. 需要点击`微信登陆`按钮，先找到按钮的`选择器`，右键按钮并在菜单中点击`检查`，然后可以看到按钮的元素

![输入图片说明](https://images.gitee.com/uploads/images/2020/1108/173107_6f43766e_2051718.png "2.png")

2. 右键元素打开菜单找到`copy`下的`copy selector`，即获取到`选择器`

![输入图片说明](https://images.gitee.com/uploads/images/2020/1108/173113_573eaba9_2051718.png "3.png")

3. 我们尝试点击微信登陆按钮，发现还需要点击一下确认，重复上述步骤获取`确认按钮`的选择器

![输入图片说明](https://images.gitee.com/uploads/images/2020/1108/173119_48c19f98_2051718.png "4.png")

4. 用代码执行上述点击步骤
```golang
// 自定义任务
func myTasks() chromedp.Tasks {
	return chromedp.Tasks{
		// 1. 打开金山文档的登陆界面
		chromedp.Navigate(loginURL),

		// 2. 点击微信登陆按钮
		// #wechat > span:nth-child(2)
		chromedp.Click(`#wechat > span:nth-child(2)`),

		// 3. 点击确认按钮
		// #dialog > div.dialog-wrapper > div > div.dialog-footer > div.dialog-footer-ok
		chromedp.Click(`#dialog > div.dialog-wrapper > div > div.dialog-footer > div.dialog-footer-ok`),
	}
}
```

5. 运行程序即可直达二维码展示界面

![输入图片说明](https://images.gitee.com/uploads/images/2020/1108/173125_4952b06a_2051718.png "5.png")

6. 用同样的方式，获取`二维码图片`的`选择器`

![输入图片说明](https://images.gitee.com/uploads/images/2020/1108/173131_6a34f0bc_2051718.png "6.png")

7. 用代码实现获取二维码，有两点需要注意，第一是二维码有加载过程，第二是二维码是元素渲染，我们需要用截图的方式获取（也可以用`js`来获取对应的`href`并下载，但是为了照顾小白，选择最简单的）
```golang
func myTasks() chromedp.Tasks {
	return chromedp.Tasks{
		// 1. 打开金山文档的登陆界面
		...

		// 2. 点击微信登陆按钮
		...

		// 3. 点击确认按钮
		...

		// 4. 获取二维码
		// #wximport
		getCode(),
	}
}

func getCode() chromedp.ActionFunc {
	return func(ctx context.Context) (err error) {
		// 1. 用于存储图片的字节切片
		var code []byte

		// 2. 截图
		// 注意这里需要注明直接使用ID选择器来获取元素（chromedp.ByID）
		if err = chromedp.Screenshot(`#wximport`, &code, chromedp.ByID).Do(ctx); err != nil {
			return
		}

		// 3. 保存文件
		if err = ioutil.WriteFile("code.png", code, 0755); err != nil {
			return
		}
		return
	}
}
```

8. 执行程序即可发现目录下已经存储了二维码图片文件，我们可以通过扫描此二维码进行登陆，与浏览器上扫描为同一种效果

![输入图片说明](https://images.gitee.com/uploads/images/2020/1108/173138_469d0d34_2051718.png "7.png")

## 5. 如何将二维码展示在无图形化的终端上（与chromedp无关，属于额外内容）

1. 在上述步骤中，我们已经获取了二维码，接下来我们需要在终端显示二维码，首先是解码，这里使用[`gozxing`](github.com/makiuchi-d/gozxing)库
```golang
func printQRCode(code []byte) (err error) {
	// 1. 因为我们的字节流是图像，所以我们需要先解码字节流
	img, _, err := image.Decode(bytes.NewReader(code))
	if err != nil {
		return
	}

	// 2. 然后使用gozxing库解码图片获取二进制位图
	bmp, err := gozxing.NewBinaryBitmapFromImage(img)
	if err != nil {
		return
	}

	// 3. 用二进制位图解码获取gozxing的二维码对象
	res, err := qrcode.NewQRCodeReader().Decode(bmp, nil)
	if err != nil {
		return
	}
	return
}
```

2. 然后重新编码来输出二维码到终端，这里使用[`go-qrcode`](github.com/skip2/go-qrcode)库
```golang
// 请注意import的库发生了重名
import (
	"github.com/makiuchi-d/gozxing"
	"github.com/makiuchi-d/gozxing/qrcode"
	goQrcode "github.com/skip2/go-qrcode"
)


func printQRCode(code []byte) (err error) {
	// 1. 因为我们的字节流是图像，所以我们需要先解码字节流
	...

	// 2. 然后使用gozxing库解码图片获取二进制位图
	...

	// 3. 用二进制位图解码获取gozxing的二维码对象
	...

	// 4. 用结果来获取go-qrcode对象（注意这里我用了库的别名）
	qr, err := goQrcode.New(res.String(), goQrcode.High)
	if err != nil {
		return
	}

	// 5. 输出到标准输出流
	fmt.Println(qr.ToSmallString(false))

	return
}
```

3. 修改我们第二步的过程
```golang
func getCode() chromedp.ActionFunc {
	return func(ctx context.Context) (err error) {
		// 1. 用于存储图片的字节切片
		...

		// 2. 截图
		// 注意这里需要注明直接使用ID选择器来获取元素（chromedp.ByID）
		...

		// 3. 把二维码输出到标准输出流
		if err = printQRCode(code); err != nil {
			return err
		}
		return
	}
}
```

3. 运行程序即可查看效果

![输入图片说明](https://images.gitee.com/uploads/images/2020/1108/173146_e4040b9f_2051718.png "8.png")

# 6. 如何保存Cookies实现短时间免登陆

1. 在上述过程中，我们可以通过二维码扫描登陆，网站会在登陆之后进行跳转，跳转后我们需要保存`cookies`来维持我们的登录状态，代码实现如下
```golang
// 保存Cookies
func saveCookies() chromedp.ActionFunc {
	return func(ctx context.Context) (err error) {
		// 等待二维码登陆
		if err = chromedp.WaitVisible(`#app`, chromedp.ByID).Do(ctx); err != nil {
			return
		}

		// cookies的获取对应是在devTools的network面板中
		// 1. 获取cookies
		cookies, err := network.GetAllCookies().Do(ctx)
		if err != nil {
			return
		}

		// 2. 序列化
		cookiesData, err := network.GetAllCookiesReturns{Cookies: cookies}.MarshalJSON()
		if err != nil {
			return
		}

		// 3. 存储到临时文件
		if err = ioutil.WriteFile("cookies.tmp", cookiesData, 0755); err != nil {
			return
		}
		return
	}
}
```

2. 获取到Cookies之后，我们需要在程序运行时将Cookies从临时文件中加载到浏览器中
```golang
// 加载Cookies
func loadCookies() chromedp.ActionFunc {
	return func(ctx context.Context) (err error) {
		// 如果cookies临时文件不存在则直接跳过
		if _, _err := os.Stat("cookies.tmp"); os.IsNotExist(_err) {
			return
		}

		// 如果存在则读取cookies的数据
		cookiesData, err := ioutil.ReadFile("cookies.tmp")
		if err != nil {
			return
		}

		// 反序列化
		cookiesParams := network.SetCookiesParams{}
		if err = cookiesParams.UnmarshalJSON(cookiesData); err != nil {
			return
		}

		// 设置cookies
		return network.SetCookies(cookiesParams.Cookies).Do(ctx)
	}
}
```

3. 通过上述两步我们已经可以保持登陆状态，然后我们需要检查一下是否成功，这里调用浏览器执行`js`脚本获取当前页面的网址，判断是否已经个人中心页面，如果为真，则停止操作
```golang
// 检查是否登陆
func checkLoginStatus() chromedp.ActionFunc {
	return func(ctx context.Context) (err error) {
		var url string
		if err = chromedp.Evaluate(`window.location.href`, &url).Do(ctx); err != nil {
			return
		}
		if strings.Contains(url, "https://account.wps.cn/usercenter/apps") {
			log.Println("已经使用cookies登陆")
			chromedp.Stop()
		}
		return
	}
}
```

4. 最终重新设置我们的浏览器任务即可
```golang
// 自定义任务
func myTasks() chromedp.Tasks {
	return chromedp.Tasks{
		// 0. 加载cookies <-- 变动
		loadCookies(),

		// 1. 打开金山文档的登陆界面
		...

		// 判断一下是否已经登陆  <-- 变动
		checkLoginStatus(),

		// 2. 点击微信登陆按钮
		// #wechat > span:nth-child(2)
		...

		// 3. 点击确认按钮
		// #dialog > div.dialog-wrapper > div > div.dialog-footer > div.dialog-footer-ok
		...

		// 4. 获取二维码
		// #wximport
		...

		// 5. 若二维码登录后，浏览器会自动跳转到用户信息页面  <-- 变动
		saveCookies(),
	}
}
```

5. 我们使用`已经登陆的cookies`运行程序可以发现我们成功跳过登陆过程

![输入图片说明](https://images.gitee.com/uploads/images/2020/1108/173153_ec73bbe4_2051718.png "9.png")
