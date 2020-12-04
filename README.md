### 背景

#### 项目问题
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
虽然官网的api中有部分相关可用之处,但是我实践的时候并未生效. xlsx-style插件的numFmt属性理论上是可以转换格式的,
但是我没有成功.

#### 解决方案
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

- [使用blob的形式下载](#)