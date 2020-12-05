### 背景

#### 项目问题
------------------
前端在开发后台系统的时候,经常会遇到处理文件的相关问题.比如点击下载文件.
本人在项目遇到的问题也与此有关.具体的需求是前端提供一个上传excel的组件,
在组件里面同时提供一个下载模板的功能方便用户填写.
涉及到了excel的文件处理, 前端还是使用了js-xlsx的插件通过一些api来方便操作
下载模板的核心代码片段如下:
```
      const wopts = {
        bookType: "xlsx",
        bookSST: false,
        type: "binary",
        showGridLines: false
      };
      const wb = { SheetNames: ["Sheet1"], Sheets: {}, Props: {} };
      let data = this.exportList; 
      wb.Sheets["Sheet1"] = XLSX.utils.json_to_sheet(data);
      wb.Sheets["Sheet1"]["!cols"] = [
        { wch: 20 },
        { wch: 20 },
        { wch: 20 },
        { wch: 20 },
        { wch: 20 },
        { wch: 20 },
        { wch: 20 }
      ];
      //创建二进制对象写入转换好的字节流
      let tmpDown = new Blob([this.s2ab(XLSX.write(wb, wopts))], {
        type: "application/octet-stream"
      });
      // 保存文件
      FileSaver.saveAs(tmpDown, this.tem_file_name);
---------------------------------------------------------------------
exportList 是前端自己写的一个数组 用于写入excel的表头数据
    exportList: [
        {
            渠道名称: "",
            分机号: "",
            分机密码: "",
            坐席id: "",
            状态: ""
        }
    ]
```
用上述代码的方式完成了纯前端驱动的代码模板下载
当然其中还是用到了一个插件 FileSaver来保存文件
详细的作用可以自行查阅 [FileSaver](https://github.com/eligrey/FileSaver.js)
利用filesaver把流文件存在本地,完成下载.这一套流程下来看似完美.
但是细心的测试发现excel在输入的时候,单元格格式默认为常规,
常规格式下的单元格如果输入的数字过大会被转换为科学输入法,输入为年月日的时间有时候也会被转换成与输入不相同的值
这样在读取excel数据的时候就会不准确导致最后传给后端的参数也不对.

首先想到用xlsx的有关插件或者api来修改单元格的文本格式,但是使用了网上的很多办法都没有成功.
虽然官网的api中有部分相关可用之处,但是我实践的时候并未生效. 也存在对单个单元格设置格式的,但是
对于当下这个需求

#### 解决方案
-------
在修改插件不成功的情况,想到了直接把本地存储的方式改为把模板上传oss服务器上.在上传之前control+a全局修改了单元格
的格式为文本类型, 对应的参数也就改为了字符串string也就不存在之前的大数字和时间格式的问题了,这样用户输入什么,拿到
的输出参数就是什么. 通过一个附件上传调用公司的oss服务器上传,得到响应的值之后,手动抄下了文件的地址,在项目中自己做了
一个映射.
```
baseOssUrl是公司的oss地址
export const EXCEL_ADRESS = {
  relation: `${baseOSSUrl}uploads/images/20201204/1c3cdabe2fc3d8f4c54c8fbea2a81e7b.xlsx`,
  groupSend: `${baseOSSUrl}uploads/images/20201204/b38b60cd4933f02fe6a9802ee63697c5.xlsx`,
  singleSend: `${baseOSSUrl}uploads/images/20201204/c6d2a1dc235f0d8a3f734b3d08782fd8.xlsx`,
  createRobot: `${baseOSSUrl}uploads/images/20201204/e5f351c852d850ee9e6b93be460ab875.xlsx`,
  addFriend:`${baseOSSUrl}uploads/images/20201204/c3324cd802bf04b90f7ed773d7b6e950.xlsx`
}
export const FILENAME_MAP = {
  relation: '批量关联模板.xlsx',
  groupSend: '群发任务模板.xlsx',
  singleSend: '私发任务模板.xlsx',
  createRobot: '机器人模板.xlsx',
  addFriend: '添加好友任务模板.xlsx'
}
```

接下来我们只要用前端下载文件的方案来处理这些对应url就行了.
本来问题到这里应该就处理的差不多了.但是让我非常头痛的是下完文件之后发现默认的文件名
读取的oss服务器上面的加上处理时间的乱码文件名.
为了解决这个问题也查阅了很多相关资料.也就引出了最终我们要探讨的这个问题,有关前端文件下载
的一些方案.

#### 前端关于文件下载的一些方案

- ##### 利用a标签的download属性进行下载
果 a 标签的 href 属性的值是一个指向浏览器可以打开的 MIME 中的一种时，浏览器会加载该 URI 指向的文件的并展示出来；如果 URI 指向的文件并不能被浏览器展现时，则会被下载到本地。

而在 HTML5 中，a 标签新增来一个 download 属性，如果一个 a 标签在使用时添加了 download 属性的话，在点击时，浏览器会将 href 指向的文件下载到本地。如果 download 属性设置了值的话，该属性的值会作为下载到本地文件的名字。

但是，如果 a 标签的 href 是指向的一个接口，通过接口下载文件的话，download 属性即使设置了值，也不能更改下载到本地的文件的名字；同样，下载 OSS 上的文件，也不能通过设置 download 属性来改变下载到本地文件的名字。所以，如果使用 a 标签下载文件并且想修改下载到本地的文件名时，需要服务端配合修改 HTTP 的协议头 Content-Disposition。
利用a标签可以完成下载功能
&lt;a href='url' download='filename' /> 

不过在大多数需求中不希望页面内有实质性的a标签展示出来
所以通常会采用下面这种方式
```
export const downloadByHyperlink = (url, filename = "") => {
  const link = document.createElement("a");
  document.body.append(link);
  link.href = url;
  link.download = filename;
  link.target = "_self";
  link.click();
  document.body.removeChild(link);
};
```
原理也很简单 就是动态创建一个a标签让他执行一个click事件之后移除
- ##### 利用iframe下载
使用 iframe 下载文件与使用 虚拟 a 标签下载具有一样的局限：只能下载浏览器不能渲染的文件。其本质也是借助浏览器会下载不能渲染的文件的特性。

下载代码与使用 虚拟 a 标签下载 差不多：
```
let f = document.createElement('iframe')
# document.body.append(f)
f.src = 'URL/to/file'
# document.body.remove(f)
```
- ##### 利用winodw.open下载
同没有添加 download 属性的 a 标签一样，可以通过 window.open 方法下载部分文件，这些可以下载的文件是不能被浏览器展现出来的文件；对于可以被浏览器解析并展现的文件，windown.open 方法只会在新打开的窗口渲染文件内容，并不会下载到本地。

除了以上的问题外，使用 window.open 还会出现以下几个问题：
 - window.open 方法还会先打开一个空白的页面，然后在新打开的页面中实现下载，体验不是很好；
 - 新打开的页面不会自己关闭，需要开发者自己手动关闭新打开的页面，这里就会出现一个问题：如果关闭新窗口的代码执行的太早，下载的请求链接没有传输完成时，则该下载会被中断。而且开发者没有办法知道下载请求链接是否完成，所以要么不关闭新打开的窗口，由用户关闭；或者设置一个比较大的定时器，由定时器来关闭新打开的空白页面。
 -对于异步获取的下载 url，通过 window.open 打开新页面时会被浏览器拦截，即该页面不会被打开，会被浏览器折叠在地址栏的最右边，需要用户手动信任后才能下载；

 所以不建议通过 window.open 方法下载文件。

**注：**对于使用 window.open 打开异步获取的 url 被浏览器拦截的问题，可以通过先创建新的空白页面，然后设置 url 的方式打开：
```
let w = window.open()
let url = (async function () {
	return await getUrlAsync()
})()
w.location = url

```
##### 另外:不要通过 window.open 方法打开不安全的下载页面，因为新打开的页面可以通过 window.opener 获取你的页面引用

- ##### 使用 blob + ObjectURL + a 标签的方式下载
该方法的原理是:
- 通过 Ajax 请求将要下载的文件以 blob 的格式下载到本地；
- 通过 window.URL.createObjectURL(blob) 创建一个标识文件对象的 Object URL；
- 通过 使用虚拟 a 标签下载 下载到本地；

而使用这种方法的优势和劣势也是很明显的
##### 优势
- a标签的download如果是oss的链接下载下来其实是不支持改名字的,这种方法就可以
,因为下载的已经是我们自己创建的文件对象了,对a标签来说href对应的并不是一个接口
##### 劣势
- 需要下载 blob 格式的文件，所以需要服务器支持 responseType: blob；
- 需要先将文件下载到本地之后再使用 window.URL.createObjectURL(blob) 创建 Object URL，所以如果文件比较大，ajax 请求需要很久才能下载完成，下载期间没有任何反应，所以体验不好；
- 并不支持跨域,所以只能下载同源资源,或者解决ajax的跨域问题;
##### 代码如下
```
const getBlob = url => {
  return new Promise(resolve => {
    const xhr = new XMLHttpRequest();
    xhr.open("GET", url, true);
    // xhr.setRequestHeader( 'Access-Control-Allow-Origin', '*')
    xhr.responseType = "blob";
    xhr.onload = () => {
      if (xhr.status === 200) {
        resolve(xhr.response);
      }
    };

    xhr.send();
  });
};
const saveAs = (blob, filename) => {
  if (window.navigator.msSaveOrOpenBlob) {
    navigator.msSaveBlob(blob, filename);
  } else {
    const link = document.createElement("a");
    const body = document.querySelector("body");

    link.href = window.URL.createObjectURL(blob); // 创建对象url
    link.download = filename;

    // fix Firefox
    link.style.display = "none";
    body.appendChild(link);

    link.click();
    body.removeChild(link);

    window.URL.revokeObjectURL(link.href); // 通过调用 URL.createObjectURL() 创建的 URL 对象
  }
};

export const downloadByBlod = (url, filename = "") => {
  getBlob(url).then(blob => {
    saveAs(blob, filename);
  });
};

```

#### 结语
实际上,上面这几种方式都没有办法解决目前所遇到的oss文件下载无法更名的问题.
我们知道服务器上的静态文件可以通过 a 标签 + download 属性的方式实现下载，并且可以修改下载到本地的文件名字；而 OSS 上的文件，或者通过请求接口下载的文件，不能通过设置 download 属性来修改下载到本地的文件的名字，这个时候可以请服务端配合，在下载接口中返回如下 HTTP 协议头：
```
'Content-Disposition: attachment; filename="downloaded.pdf"'
```
浏览器在请求响应时，如发现该 HTTP 协议头，会将 filename 的值设置为下载文件的名字，这样就可以避免使用 blob 方式下载时的“假死”问题，也修改了下载文件的名字。

可以在设计接口时，留一个设置文件名的参数，这样就可以在调用下载接口时，将想要设置的文件名以参数的形式传递到服务端；服务端接口在响应时，在响应中带上 HTTP 协议头，通知浏览器修改下载文件的名字。


```
// 告诉浏览器这是下载文件
response.setHeader("content-disposition", "attachment;filename="+ filename);
response.setHeader("content-type", "image/jpeg");

当在火狐浏览器中，以上代码不能正常显示文件名。

// 设置文件名的编码方式，使得文件的名字能够正常安全的显示。
filename = URLEncoder.encode(filename, "UTF-8");

// 告诉浏览器这是下载文件
response.setHeader("content-disposition", "attachment;filename*=UTF-8''"+ filename);
response.setHeader("content-type", "image/jpeg");

```