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
增加部分如下（*号之间的内容）：

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