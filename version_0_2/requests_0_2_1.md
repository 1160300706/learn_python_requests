这篇文章对应的是v0.2.1版本，废话不多说直接开始。

从目录结构上看，v0.2.1版本新增加了一个packages目录，其余的目录结构保持不变。


我们先从测试模块开始，对比上个版本的测试，这次新增加了对POST_FILES的测试：

	def test_POSTBIN_GET_POST_FILES(self):

		bin = requests.post('http://www.postbin.org/')
		self.assertEqual(bin.status_code, 200)

		post = requests.post(bin.url, data={'some': 'data'})
		self.assertEqual(post.status_code, 201)

		post2 = requests.post(bin.url, files={'some': StringIO('data')})
		self.assertEqual(post2.status_code, 201)

测试了POST方法不同参数情况下的执行情况。

接下来看看core的实现，在Request类中增加了一个属性files，修改了__repr__方法的实现，
没有使用try-except而是打印出Request对象的方法名。

	def __repr__(self):
		return '<Request [%s]>' % (self.method)

send中GET、HEAD、DELETE等类型的方法没有变更，而在PUT和POST方法中增加了对files属性适配。
以PUT方法为例，增加部分如下（*号之间的内容）：

		elif self.method == 'PUT':
			if (not self.sent) or anyway:

				********if self.files:
					register_openers()
					datagen, headers = multipart_encode(self.files)
					req = _Request(self.url, data=datagen, headers=headers, method='PUT')

					if self.headers:
						req.headers.update(self.headers)********

				else:

					req = _Request(self.url, method='PUT')

					if self.headers:
						req.headers = self.headers

					req.data = self.data

				try:
					opener = self._get_opener()
					resp =  opener(req)

					self.response.status_code = resp.code
					self.response.headers = resp.info().dict
					self.response.content = resp.read()
					self.response.url = resp.url

					success = True

				except urllib2.HTTPError as why:
					self.response.status_code = why.code

新增部分调用了register_openers方法，该方法位于/requests/pacakges/poster的streaminghttp模块中，
该模块的功能说明如下：

Streaming HTTP uploads module.

This module extends the standard httplib and urllib2 objects so that
iterable objects can be used in the body of HTTP requests.

In most cases all one should have to do is call :func:`register_openers()`
to register the new streaming http handlers which will take priority over
the default handlers, and then you can use iterable objects in the body
of HTTP requests.

**N.B.** You must specify a Content-Length header if using an iterable object
since there is no way to determine in advance the total size that will be
yielded, and there is no way to reset an interator.

Example usage:

from StringIO import StringIO

import urllib2, poster.streaminghttp

opener = poster.streaminghttp.register_openers()

s = "Test file data"

f = StringIO(s)

req = urllib2.Request("http://localhost:5000", f,
                       {'Content-Length': str(len(s))})

简单来说就是扩充了标准的httplib和urllib2这两个模块，以便于可迭代对象能用于http请求中，
大多数时候，我们只需要调用register_openers方法，去注册新的流式http handlers，就可以了；
此外必须指定Content-Length header，因为无法确定产生的总大小同时也无法重置迭代器。

而register_openers方法会在全局默认的urllib2的opener对象中注册流式http处理服务，然后返回该创建的对象。
在此处