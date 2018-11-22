接口
===============

保全 - /attestations
----------------------

客户在保全链(VREAXCHAIN)站上建好模板之后通过该接口传输模板渲染需要的数据。

payload
^^^^^^^^^^^^^^^

=================  ======================================= ================
参数名 				描述                                    是否可选
=================  ======================================= ================
unique_id          String字符串，不超过255位，保全唯一码          必选
template_id        String字符串，模板id                       必选
identities         Object对象，身份事项                        必选
factoids           数组对象，陈述集                           必选
completed          Boolean值，是否完成陈述集的上传            可选，默认为true
attachments        附件信息数组对象                           可选
=================  ======================================= ================

unique_id是保全唯一码，这个唯一码的作用是避免在网络超时或者其它异常的情况下接入方重复上传相同内容的保全数据。如果同样unique_id的保全内容多次上传，保全链(VREAXCHAIN)只进行1次保全，并返回相同的保全号。

**陈述** 是一个Object对象，包含unique_id,type和data三个字段，例如::

	{
		"unique_id": "9de7be94-a697-4398-945a-678d3f916b9f",
		"type": "hash",
		"data": {
			"userName": "张三",
			"idCard": "42012319800127691X"
		}
	}

陈述的unique_id的作用跟保全的unique_id类似，如果某次保全过程中同样unique_id的陈述内容多次上传到保全链(VREAXCHAIN)，保全链(VREAXCHAIN)只处理1次。

type是客户定义的陈述名称，data是陈述的字段值，如下图所示：

.. image:: images/use_template.png

模板中包含common和hash两个陈述。common中的attestation_no(保全号)、attestation_at(保全时间)、product_name(产品名称)、organization_name(组织名称)在模板渲染时由保全链(VREAXCHAIN)提供。hash这个陈述是客户自己定义的，所以需要客户通过API上传。

.. note:: 
	在添加陈述对象是要保证陈述对象跟编辑模板时的要求一致。以下几种情况会导致保全失败：
	
	- 上传了模板中没有的陈述对象，比如模板中没有type为product的对象却上传了。
	- 模板中有字段是必要的，但是完成陈述上传时并没有上传该字段，比如user.name需要上传且不能为空，
	  但是没有上传type为user的data或者data中没有name这个字段。
	- 上传的字段的格式不符，比如模板中要求user.money是int型，但是上传的user.money值是"$100"

模板中可能包含多个客户自定义的陈述，比如factoA和factoB，此时客户可以选择分两次上传，第一次上传factoA，并设置completed为false，第二次上传factoB，并设置completed为true。

.. note:: 一旦completed设置为true，则不再接受陈述上传。

假定payload如下所示::

	{
		"unique_id": "acafa00d-5579-4fe5-93c1-de89ec82006e",
		"template_id": "2hSWTZ4oqVEJKAmK2RiyT4",
		"identities": {
			"MO": "15857112383",
			"ID": "42012319800127691X"
		},
		"factoids": [
			{
				"unique_id": "9de7be94-a697-4398-945a-678d3f916b9f",
				"type": "hash",
				"data": {
					"userName": "李三",
					"idCard": "330124199501017791",
					"buyAmount": 0.3,
					"incomeStartTime": "2015-12-02",
					"incomeEndTime": "2016-01-01",
					"createTime": "2015-12-01 14:33:44",
					"payTime": "2015-12-01 14:33:59",
					"payAmount": 600
				}
			}
		],
		"completed": true
	}

经过保全后，在保全链(VREAXCHAIN)上可以通过保全号查看经过渲染后的内容，类似下图所示：

.. image:: images/render_template.png

附件
^^^^^^^^^^^^^^^

在上传陈述数据的时候可以同时上传跟该陈述相关的附件，在payload中 **attachments** 存放的是附件的校验码。

form表单形式上传单个附件::

	<form method='post' enctype='multipart/form-data'>
	  ...
	  <input type=file name="attachments[0][]">
	</form>

	payload = {
		"unique_id": "...",
		"template_id": "...",
		"identities": {...},
		"factoids": [
			{
				"unique_id": "...",
				"type": "...",
				"data": {...}
			}
		],
		"completed": true,
		"attachments": {
			"0": ["checkSum"]
		}
	}

form表单形式上传多个附件::

	<form method='post' enctype='multipart/form-data'>
	  ...
	  <input type=file name="attachments[0][]">
	  <input type=file name="attachments[0][]">
	  <input type=file name="attachments[1][]">
	</form>

	payload = {
		"unique_id": "...",
		"template_id": "...",
		"identities": {...},
		"factoids": [
			{
				"unique_id": "...",
				"type": "...",
				"data": {...}
			},
			{
				"unique_id": "...",
				"type": "...",
				"data": {...}
			}
		],
		"completed": true,
		"attachments": {
			"0": [
				"checkSum1", 
				{
					"checksum": "checkSum2",
					"sign": {
						"F98F99A554E944B6996882E8A68C60B2": ["甲方（签章）", "甲方法人（签章）"],
						"0A68783469E04CAC95ADEAE995A92E65": ["乙方（签章）"]
					}
				}
			],
			"1": ["checkSum3"]
		}
	}

attachments中的key对应的是factoids数组的下标，比如"0"对应的是factoid为factoids[0]。attachments中的value是一个数组，每个数组元素表示对应附件的附件信息。

附件信息有两种：校验码和电子签名信息，其中校验码是必须提供。当附件信息只有校验码时可以用字符串对象，当包含电子签名信息时需要使用object对象。

.. note:: 只有pdf附件才能进行电子签名。

校验码（checksum）是对文件进行SHA256产生的，以Java为例::

	String file = "/path/to/file";
	InputStream in = new FileInputStream(new File(file));

	// 使用SHA256对文件进行hash
	bytes[] digestBytes = DigestUtils.getDigest("SHA256").digest(StreamUtils.copyToByteArray(in));

	// 将bytes转换成16进制
	String checkSum = Hex.encodeHexString(digestBytes);

电子签名信息（sign）是一个object对象，key值是caId（客户调用申请ca证书接口时会返回caId），value值是签名关键字数组。比如“张三”和“李四”需要在“xxx合同.pdf”附件上进行电子签名，调用ca证书申请接口为“张三”申请得到的caId是"F98F99A554E944B6996882E8A68C60B2"，为“李四”申请得到的caId是"0A68783469E04CAC95ADEAE995A92E65"，其中“张三”需要在"甲方（签章）", "甲方法人（签章）"两个位置进行电子签名，”李四“只需要在"乙方（签章）"进行电子签名，那么sign对象可以表示为::

	"sign": {
		"F98F99A554E944B6996882E8A68C60B2": ["甲方（签章）", "甲方法人（签章）"],
		"0A68783469E04CAC95ADEAE995A92E65": ["乙方（签章）"]
	}

.. note:: 同一个用户可以在多处进行电子签名，但关键字要保证唯一，不能跟正文内容重复。


返回的data
^^^^^^^^^^^^^^

调用保全接口成功后会返回保全号

=================  ================================
字段名 				描述                            
=================  ================================
no                 String字符串，保全号                                         
=================  ================================

例如::

	{
		"request_id": "2XiTgZ2oVrBgGqKQ1ruCKh",
		"data": {
			"no": "rBgGqKQ1ruCKhXiTgZ2oVr",
		}
	}




