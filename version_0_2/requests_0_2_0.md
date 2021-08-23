这篇文章对应的是requests v0.2.0版本：它包含requests文件夹、HISTORY文件、LICENSE、README和setup.py以及test_requests.py。
从README中我们可以看出作者的意图：

Most existing Python modules for dealing HTTP requests are insane. I have to look up *everything* that I want to do. Most of my worst Python experiences are a result of the various built-in HTTP libraries (yes, even worse than Logging). 

But this one's different. This one's going to be awesome. And simple.

Really simple. 

作者觉得当前Python的http模块非常难使用，所以想自己实现一个Python库来处理http请求，当然由于作者的Python经验还不是很好，所以库的实现是基于Python标准的http模块的。

requests库提供的API有Requests、Response、AuthObject


Requests：拥有GET、HEAD、PUT、POST、DELETE方法，它们的返回值都是一个Response对象

Response：拥有Request.status_code、Request.headers、Request.content属性


Request.status_code：（Integer）接收http请求的状态码

Request.headers：（Dictionary）接收http请求的头文件（headers）

Request.content：（Bytes）接收http请求的content