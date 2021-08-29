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

它首先声明了一个私有的元组类变量_METHODS，通过它来记录该类有哪些方法，
然后在对类进行了初始化，**初始化时并没有直接把类的属性作为参数进行初始化，
而是选择在实现GET、POST等方法时根据方法的实现方式去初始化方法所需要的属性，
这样可以让Request类的实现更加灵活。**
接着重写了__repr__方法，

	def __repr__(self):
		try:
			repr = '<Request [%s]>' % (self.method)
		except:
			repr = '<Request object>'
		return repr

该方法提供的功能是：当直接打印该类的实例化对象时，系统会输出对象的自我描述信息。
然后在__setattr__中对method进行了限制，如果输入的method的值不在_METHODS中，就会抛出InvalidMethod的异常。

	def __setattr__(self, name, value):
		if (name == 'method') and (value):
			if not value in self._METHODS:
				raise InvalidMethod() 
		
		object.__setattr__(self, name, value)
此处对method的检查实现的比较有意思，它是和Request类的初始化实现相照应的。
前面我们说过Request类初始化时并没有把类的属性作为参数进行初始化 而是在实现具体方法时再去对类的属性进行赋值；
而__setattr__的执行逻辑是：**每次对类的属性进行初始化时都会自动执行方法**，
这样才能在初始化时将self.method设置为None。

接下来看send方法，它是Request类重要逻辑的实现部分，我们还是以GET方法为例，其实现如下：

	def send(self, anyway=False):
		"""Sends the request. Returns True of successfull, false if not.
		    If there was an HTTPError during transmission,
		    self.response.status_code will contain the HTTPError code.

		    Once a request is successfully sent, `sent` will equal True.
		
		    :param anyway: If True, request will be sent, even if it has
		    already been sent.
		"""
		self._checks()

		success = False
		
		if self.method in ('GET', 'HEAD', 'DELETE'):
			if (not self.sent) or anyway:

				# url encode GET params if it's a dict
				if isinstance(self.params, dict):
					params = urllib.urlencode(self.params)
				else:

					params = self.params

				req = _Request(("%s?%s" % (self.url, params)), method=self.method)

				if self.headers:
					req.headers = self.headers

				opener = self._get_opener()

				try:
					resp = opener(req)
					self.response.status_code = resp.code
					self.response.headers = resp.info().dict
					if self.method.lower() == 'get':
						self.response.content = resp.read()

					success = True
				except urllib2.HTTPError, why:
					self.response.status_code = why.code

首先调用了_checks方法，对url是否初始化进行了检查(并没有对URL的格式和内容进行检查)，
然后对输入的params参数调用urllib.urlencode方法将键值对的参数格式化使用&划分，
使用效果如下：params = urllib.urlencode({"a":1,"b":2})则params=“a=1&b=2”。
接着声明了_Request的对象，该类的实现如下： 

class _Request(urllib2.Request):

	"""Hidden wrapper around the urllib2.Request object. Allows for manual
	setting of HTTP methods.
	"""

	def __init__(self, url,
					data=None, headers={}, origin_req_host=None,
					unverifiable=False, method=None):
		urllib2.Request.__init__( self, url, data, headers, origin_req_host,
								  unverifiable)
	   	self.method = method

	def get_method(self):
		if self.method:
			return self.method

		return urllib2.Request.get_method(self)

它继承自urllib2的Request，在初始化时直接调用了父类的初始化方法。
然后调用了_get_opener方法，并返回一个urllib2的opener对象，接下来就是调用Python标准的http库发送请求：
resp = opener(req)，其等价于resp = urllib2.urlopen(urllib2.Request())，最后组装Response对象就结束了。

到此requests模块的V0.2.0版本的解析就结束了，由于版本比较旧有些地方可能没有感受到独特之处，期待V0.2.1版本。