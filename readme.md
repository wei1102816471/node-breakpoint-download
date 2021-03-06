
## 背景

在做nw应用的下载更新时候，需求是可以暂停下载。

最开始考虑的是用request下载，会返回一个流，流有暂停、恢复的功能。暂停不长的时间可以恢复正常下载，但暂停较长时间后恢复下载，会立马结束。不满足需求。

网上搜了下node断点下载的资料，大都是断点续传的。搜到些其他语言比如java做断点下载的文章，总结了下，自己用node做了实现。

## 思路

首先往资源发起一个请求，设置headers Range 为 `bytes=0-1`，可以获得资源响应的header，判断其中`accept-ranges`的值是否为`bytes`，有则支持断点下载。同时还能通过`content-range`获得资源的整体大小。

判断本地文件是否存在，存在则获取大小判断是否下载完成。不存在则本地文件大小视为0。

创建写入流，追加写入
```
fileWrite = fs.createWriteStream(destinationPath, {
        flags: "a"
      });
```
根据本地大小，发起文件下载请求，设置headers的`Range`
```
downStream = request(
        {
          method: "GET",
          url,
          forever: true,
          headers: {
            "Cache-Control": "no-cache",
            Range: `bytes=${localSize}-${totalSize - 1}`
          }
        },
        function() {}
      );
```

返回的流，可以用来中断下载。需要重新下载时，重新走一遍上面流程。根据大小判断下载完成。若已知md5，再校验下载文件的md5和已知的md5是否一致。

## 代码示例
```
const path = require('path')
const createDownloadTask = require("../index");
const tmpdir = path.join(__dirname, "down");
createDownloadTask({
    url:
      "https://folger-1251685788.cos.ap-guangzhou.myqcloud.com/blog/breakpoint/downloadtest.txt",//50M
    md5: "c8c3085051e21d57e13d6544b7bbb832",
    filename: "test1.txt",
    tmpdir
  }).then(function(task) {
    task.emit("start");
    let st;
    task
      .on("totalSize", function({ totalSize }) {
        // console.log("get total size", totalSize);
      })
      .on("progress", function({ percent }) {
        // console.log("on progress", percent);
      })
      .on("end", function({ filepath }) {
        console.log("on end", filepath);
        clearInterval(st);
      })
      .on("error", function(err) {
        console.log("on error", err);
        clearInterval(st);
      })
      .on("abort", function() {
        console.log("on abort");
      });

    st = setInterval(function() {
      task.emit("stop");
      task.emit("start");
    }, 2000);
  });
```
