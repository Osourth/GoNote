## gin使用总结

### 一.下载安装

要安装Gin包首先要安装Go并设置Go工作区

#### 1.下载并安装

```go
$ go get -u github.com/gin-gonic/gin
```

#### 2.在代码中导入

```go
import "github.com/gin-gonic/gin"
```

### 二.基本配置

#### 1.代码实列

```go
package main

import (
	"SDT/controller"
	"SDT/global"
	"SDT/model"
	"github.com/gin-gonic/gin"		//导入gin的包
	"log"
	"net/http"
)

func main()  {
	//设置为调试模式/非调试模式
	gin.SetMode(gin.DebugMode)
    //gin.SetMode(gin.ReleaseMode)
    
    //默认启动方式，包含Logger,Recovery中间件
	router := gin.Default()
	router.Delims("{[{", "}]}")
	//设置静态网页路径
	router.Static("/static", "./static")
    /*
    LoadHTMLGlob和LoadHTMLFiles两个方法来对我们的模板进行加载
    其中LoadHTMLGlob方法可以将一个目录下所有的模板进行加载，
    而LoadHTMLFiles只会加载一个文件，他的参数为可变长参数，需要我们一个一个的手动将模板文件填写。
    */
	//engine.LoadHTMLGlob("view/**/*")
	router.LoadHTMLFiles("./index.html",)
	//router.LoadHTMLFiles("./login.html",)
	//路由
	webRouter := router.Group("/sdt")
	{
		webRouter.GET("", controller.IndexHtml)
		webRouter.POST("/login", controller.Login)
		//想定的加载路由
		scenRouter := webRouter.Group("/scen")
		{
			scenRouter.POST("/addScen",controller.AddScen)
			scenRouter.GET("/getScens",controller.GetScens)
			scenRouter.GET("/delScenById",controller.DelScenById)
			scenRouter.GET("/downloadXML",controller.DownloadXML)
			scenRouter.POST("/updateScen",controller.UpdateScen)
			scenRouter.POST("/GetScenarioList",controller.ScenarioList)
			scenRouter.POST("/GetScenarioDefs",controller.GetScenarioDefs)
		}
		//实体的加载路由
		entityRouter := webRouter.Group("/entity")
		{
			entityRouter.GET("/getEntitys",controller.GetEntitys)
			entityRouter.POST("/editXML",controller.EditXML)
			entityRouter.POST("/updateXML",controller.UpdateXML)
		}
	}
	//监听端口
	router.Run(":8848")
}
```



### 三.http参数接收

#### 1.get请求

前端发送请求：

```js
 $.get('/sdt/scen/test', {"id":12,"message":"今天"},function (result) {})
```

后端接收参数：

```go
func Test(c *gin.Context) {
	
	id := c.Query("id")
	fmt.Println("id: ",id)
    message := c.Query("message")
    fmt.Println("message: ",message)
    
    c.JSON(200, gin.H{
        "status":  "posted",
        "message": "已经收到参数",
        
     })
    return
}
```

#### 2.post

前端发送请求：

```js
let data ={
    "requestId":"100",
    "body":{
        "preScenarioId": 0,
        "pageSize": 2
    }
}
$.post('/sdt/scen/test',data,function (result) {
    console.log(result)
},"json")
```

后端接收参数：

```go
func Test(c *gin.Context) {
	//此处可以接收
	requestId := c.PostForm("requestId")
	fmt.Println("requestId: ",requestId)
	//此处获取不到
    body := c.PostForm("body")
    fmt.Println("body: ",body)
    
    c.JSON(200, gin.H{
        "status":  "posted",
        "message": "已经收到参数",
        
     })
    return
}
```

#### 3.使用bind接收参数并封装到结构体中

前端发送请求：

```js
let data ={
    "requestId":"101",
    "preScenarioId": 0,
    "pageSize": 2，
    "body":{
    	 "preScenarioId": 0,
   		 "pageSize": 2，
	}
   
}

$.post('/sdt/scen/addScen',data,function (result) {
    console.log(result)
},"json")
```

后端接收参数：

```go
type Body struct {
	PreScenarioId int `form:"preScenarioId"`
	PageSize int `form:"pageSize"`
}
type Message struct {
	RequestId string `form:"requestId"`
	PreScenarioId int `form:"preScenarioId"`
	PageSize int `form:"pageSize"`
    Body Body `form:"body"`
}
func AddScen(c *gin.Context) {
	var message Message
    //post,get请求均可
	c.Bind(&message)
	fmt.Println(message.RequestId)
	fmt.Println(message.PreScenarioId)
    //注意此处接收不到，bind 仅支持没有 form 的嵌套结构体
	fmt.Println(message.Body)
    c.JSON(200, gin.H{
        "status":  "posted",
        "message": "已经收到参数",
        
     })
    return
}

```

#### 4.使用BindJson接收参数并封装到结构体内

前端发送请求：

```js
let data ={
    "requestId":"101",
    "body":{
        "preScenarioId": 0,
        "pageSize": 2
    }
}
//此处使用BindJson,需要将数据json格式化,且仅可以使用post
$.post('/sdt/scen/addScen',JSON.stringify(data),function (result) {
    console.log(result)js
},"json")
```

后端接收参数：

```go
type Message struct {
    RequestId string `form:"requestId"`
    Body Body `form:"body"`
}
func AddScen(c *gin.Context) {

    var message Message
    c.BindJSON(&message)
    fmt.Println(message.RequestId)
    fmt.Println(message.Body)
}
```

#### 5.文件上传

##### a.单文件上传

HTML部分：

```HTML
<input type="file" name="photo" id="photo" value="" placeholder="免冠照片">
<input type="button" οnclick="postData();" value="下一步" name="" style="width:100px;height:30px;">

```

Js部分：

```js
function postData(){
    var formData = new FormData();
    formData.append("photo",$("#photo")[0].files[0]);
    formData.append("service",'App.Passion.UploadFile');
    formData.append("token",token);
    $.ajax({
        url:'后台地址', /*接口域名地址*/
        type:'post',
        data: formData,
        contentType: false,
        processData: false,
        success:function(res){
            console.log(res.data);
            if(res.data["code"]=="succ"){
                alert('成功');
            }else if(res.data["code"]=="err"){
                alert('失败');
            }else{
                console.log(res);
            }
        }
    })
}

```

后端部分：

```go
//单文件上传
func UploadFile(c *gin.Context) {
	// 单文件
	file, _ := c.FormFile("file")
    //获取文件名称
	log.Println(file.Filename)

	// 上传文件到指定的路径
	err := c.SaveUploadedFile(file,"G:/files/"+file.Filename)
	if err != nil {
		fmt.Println("err : ",err)
	}
	c.String(http.StatusOK, fmt.Sprintf("'%s' uploaded!", file.Filename))
}
```

##### b.多文件上传

前端部分：

​	如上

后端部分：

```go
//多文件上传
func UploadFiles(c *gin.Context) {
	// 多文件
	form, _ := c.MultipartForm()
	files := form.File["upload[]"]

	for _, file := range files {
		log.Println(file.Filename)

		// 上传文件到指定的路径
		err := c.SaveUploadedFile(file,"G:/files/"+file.Filename)
		if err != nil {
			fmt.Println("err : ",err)
		}
	}
	c.String(http.StatusOK, fmt.Sprintf("%d files uploaded!", len(files)))
}

```

#### 6.文件下载

##### a.点击按钮实现文件下载（前端将数据组织成文件）

前端部分：

```js
'click #download': function (e, value, row, index) {
    //下载想定文件
    $.get('/sdt/scen/downloadXML', {"id":row.id},
          function (result) {
        // console.log(result.data);
        var  a = document.createElement('a');
        //设置下载的文件名
        a.download = "scen.xml";
        a.style.display = 'none';
        //此处获取后台传递来的数据，并组织成文件下载
        var  blob = new Blob([result.data]);
        a.href = URL.createObjectURL(blob);
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
    })
}
```

后端部分：

```go
func DownloadXML(c *gin.Context) {
	scenId := c.Query("id")
	id,_ := strconv.Atoi(scenId)
	fmt.Println("id : ",id)
	scen := model.GetScen(id)
	troop := model.GetTroop(id)
	//此处返回的是string
	xmlScen := CreateXML(scen,troop)
	fmt.Println("文件下载")
	c.JSON(http.StatusOK, gin.H{
		"status":   200,
		"data":    xmlScen,
	})
	fmt.Println("kdflsj")o
	return
}
```

##### b.点击按钮实现文件下载

前端部分：

```js
//1.通过创建a标签形式，注意此处后台方法需要为Get接口
function(){
    var link = document.createElement('a');
    //给下载的文件起名
    link.setAttribute("download", "");
    link.href = "http://localhost:8848/sdt/scen/fileDownload";
    link.click();
}
//2。针对后端post请求。利用原生的XMLHttpRequest方法实现

function request () {
    const req = new XMLHttpRequest();
    req.open('POST', '<接口地址>', true);
    req.responseType = 'blob';
    req.setRequestHeader('Content-Type', 'application/json');
    req.onload = function() {
      const data = req.response;
      const a = document.createElement('a');
      const blob = new Blob([data]);
      const blobUrl = window.URL.createObjectURL(blob);
      download(blobUrl) ;
    };
    req.send('<请求参数：json字符串>');

  }; 

function download(blobUrl) {
  const a = document.createElement('a');
  a.style.display = 'none';
  a.download = '<文件名>';
  a.href = blobUrl;
  a.click();
  document.body.removeChild(a)；
}
request();

//方法3.针对post接口，利用原生的fetch方法实现
function request() {
  fetch('<接口地址>', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: '<请求参数：json字符串>',
  })
    .then(res => res.blob())
    .then(data => {
      let blobUrl = window.URL.createObjectURL(data);
      download(blobUrl);
    });
}
function download(blobUrl) {
  const a = document.createElement('a');
  a.style.display = 'none';
  a.download = '<文件名>';
  a.href = blobUrl;
  a.click();
  document.body.removeChild(a);
}
request();


```

后端部分：

```go
func FileDownload(c *gin.Context){

    c.Writer.Header().Add("Content-Disposition", fmt.Sprintf("attachment; filename=%s", "test"))
    //fmt.Sprintf("attachment; filename=%s", filename)对下载的文件重命名
    c.Writer.Header().Add("Content-Type", "application/octet-stream")
    c.File("./index.html")
   

}

```

