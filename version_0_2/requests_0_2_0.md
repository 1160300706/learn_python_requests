这篇文章对应的是requests v0.2.0版本：它包含requests文件夹、HISTORY文件、LICENSE、
README和setup.py以及test_requests.py。 从README中我们可以看出作者的想法：

Most existing Python modules for dealing HTTP requests are insane.
I have to look up *everything* that I want to do. 
Most of my worst Python experiences are a result of the various built-in
HTTP libraries (yes, even worse than Logging). 

But this one's different. This one's going to be awesome. And simple.

Really simple. 

作者觉得当前Python的http模块非常难使用，所以想自己实现一个Python库来处理http请求，
当然由于作者的Python经验还不是很好，所以库的实现是基于Python标准的http模块的。

requests库提供的API有Requests、Response、AuthObject


Requests：拥有GET、HEAD、PUT、POST、DELETE方法，它们的返回值都是一个Response对象

Response：拥有Request.status_code、Request.headers、Request.content属性

Request.status_code：（Integer）接收http请求的状态码

Request.headers：（Dictionary）接收http请求的头文件（headers）

Request.content：（Bytes）接收http请求的content、

HTTP Authentication Registry：提供add_autoauth方法对给定的URL进行认证。

其中的setup.py文件是Python模块进行发布，在此处不在详细说明，可参考以下链接：
https://blog.csdn.net/lynn_kong/article/details/17540207

了解了requests模块提供的功能之后，我们再来看该模块的具体实现core.py文件。

我们首先看它提供的GET、PUT等方法的实现，以GET为例，

def get(url, params={}, headers={}, auth=None):

	"""Sends a GET request. Returns :class:`Response` object.
	:param url: URL for the new :class:`Request` object.
	:param params: (optional) Dictionary of GET Parameters to send with the :class:`Request`.
	:param headers: (optional) Dictionary of HTTP Headers to sent with the :class:`Request`.
	:param auth: (optional) AuthObject to enable Basic HTTP Auth.
	"""
	
	r = Request()
	
	r.method = 'GET'
	r.url = url
	r.params = params
	r.headers = headers
	r.auth = _detect_auth(url, auth)
	
	r.send()
	
	return r.response

它会先实例化一个Request对象，然后设置request对象的method、url、params等属性，
然后会调用request对象的send方法，最后GET方法会返回request对象的response属性，
接下来我们看Request类的实现。

class Request(object):


	"""The :class:`Request` object. It carries out all functionality of
	Requests. Recommended interface is with the Requests functions.
	
	"""
	
	_METHODS = ('GET', 'HEAD', 'PUT', 'POST', 'DELETE')
	
	def __init__(self):
		self.url = None
		self.headers = dict()
		self.method = None
		self.params = {}
		self.data = {}
		self.response = Response()
		self.auth = None
		self.sent = False

它首先声明了一个私有的元组类变量_METHODS，通过它来记录该类可以实现哪些方法，
然后对类进行了初始化，**初始化时并没有直接将该类的属性直接进行初始化，
而是选择在实现GET、POST等方法的时候根据方法的不同去初始化方法需要的属性，
这种做法让Request类的实现更加灵活。**

