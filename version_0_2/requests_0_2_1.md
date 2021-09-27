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
以PUT方法为例，增加部分如下：

    if self.files:
        register_openers()
        datagen, headers = multipart_encode(self.files)
        req = _Request(self.url, data=datagen, headers=headers, method='PUT')
        
        if self.headers:
            req.headers.update(self.headers)

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

而register_openers方法会在全局默认的urllib2的opener对象中注册流式http处理服务，然后返回该创建的对象，其实现如下：

def register_openers():

    """Register the streaming http handlers in the global urllib2 default
    opener object.
    Returns the created OpenerDirector object."""

    handlers = [StreamingHTTPHandler, StreamingHTTPRedirectHandler]
    if hasattr(httplib, "HTTPS"):
        handlers.append(StreamingHTTPSHandler)

    opener = urllib2.build_opener(*handlers)

    urllib2.install_opener(opener)

    return opener

在此处有一个hasattr方法，通过该方法判断判断httplib对象是否有HTTPS属性，如果有就将StreamingHTTPSHandler类
加入到handlers中；然后调用urllib2中的build_opener方法，依据传入的自定义的handlers生成一个opener对象，并对opener对象初始化然后返回。
调用完register_openers方法后，接着调用了在packages/poster/encode.py中提供的multipart_encode方法，通过该方法对输入的files参数转化为
multipart/form-data的格式。该方法的使用说明如下：

def multipart_encode(params, boundary=None, cb=None):

    ``params`` should be a sequence of (name, value) pairs or MultipartParam
    objects, or a mapping of names to values.
    Values are either strings parameter values, or file-like objects to use as
    the parameter value.  The file-like objects must support .read() and either
    .fileno() or both .seek() and .tell().

    If ``boundary`` is set, then it as used as the MIME boundary.  Otherwise
    a randomly generated boundary will be used.  In either case, if the
    boundary string appears in the parameter values a ValueError will be
    raised.

    If ``cb`` is set, it should be a callback which will get called as blocks
    of data are encoded.  It will be called with (param, current, total),
    indicating the current parameter being encoded, the current amount encoded,
    and the total amount to encode.

    Returns a tuple of `datagen`, `headers`, where `datagen` is a
    generator that will yield blocks of data that make up the encoded
    parameters, and `headers` is a dictionary with the assoicated
    Content-Type and Content-Length headers.

    Examples:

    >>> datagen, headers = multipart_encode( [("key", "value1"), ("key", "value2")] )
    >>> s = "".join(datagen)
    >>> assert "value2" in s and "value1" in s

    >>> p = MultipartParam("key", "value2")
    >>> datagen, headers = multipart_encode( [("key", "value1"), p] )
    >>> s = "".join(datagen)
    >>> assert "value2" in s and "value1" in s

    >>> datagen, headers = multipart_encode( {"key": "value1"} )
    >>> s = "".join(datagen)
    >>> assert "value2" not in s and "value1" in s

其中params参数可以是字典、MultipartParam对象或者是一组（name，value）数据，其中value的类型要么是string，要么是和文件对象类似的对象（该对象
必须支持read方和fileno方法或者是seek和tell方法）。