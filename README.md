# 高性能异步WEB引擎使用说明
### High Performance Asynchronous (HPA) Web Engine v2.2
***********************************************************

#### HPA特性
- 包管理
    - 采用poetry，使用步骤
    1. pip install poetry
    2. poetry install
    3. poetry shell
        进入了virtualenv的环境，可以运行了:python main.py
        exit退出virtualenv环境
    4. 安装某个库的方式
        poetry add xxx库
        这个会自动安装并加入pyproject.toml文件

- 基础支持
    - 基础系统环境
        - Ubuntu 16.04
        - Python 3.8+
        - Tornado 6.1+
        - Postgresql 11.0
    - 基础依赖库
        - Postgresql使用库：[asyncpg](https://github.com/MagicStack/asyncpg)
        - Mysql：[aiomysql](https://github.com/aio-libs/aiomysql)
        - Redis使用库：[aioredis](https://github.com/aio-libs/aioredis)
        - MongoDB：[motor](https://github.com/mongodb/motor)
        - 图片库：pillow
        - Aliyun队列服务(mns)：重编码支持Python3的mns库
        - 微信SDK：建议使用[wechatpy](https://github.com/jxtech/wechatpy)，目前还是使用的自编码的sdk，可能有功能上的兼容问题
        - 算法加密库由pycrypto替换为pycryptodome,需要卸载掉pycrypto *（pycrypto 2012年停止维护，不直接支持Python 3）*
        
    - 其他
        - 删除了对memcache的支持，session，全局变量cache等统一使用redis，不再使用memcache
        - 为了保证base64编码url安全,不使用特殊的+/等字符进行编码 ,HPA中统一调用方式封装到utils中，统一调用
            ```python
            a = 'abc1234567890!efg中文@&+--我是实例句子_=#哈里路亚hijklmnopqrstwvwxyz'
            b = utils.Hbase64_encode(a)
            print (b)
            >>>> YWJjMTIzNDU2Nzg5MCFlZmfkuK3mlodAJistLeaIkeaYr-WunuS-i-WPpeWtkF89I-WTiOmHjOi3r-S6mmhpamtsbW5vcHFyc3R3dnd4eXo=
            c = utils.Hbase64_decode(b)
            print (c)
            >>>> abc1234567890!efg中文@&+--我是实例句子_=#哈里路亚hijklmnopqrstwvwxyz
            
            ```


- 开发环境
    ~~- VMware-workstation-full-14.1.2-8497320~~
    ~~- 为了和老版本兼容，内网地址为：192.168.22.200， 老版本为192.168.22.100~~
    
- pipenv install安装即可
    
- 使用要求
    - 所有输出使用loggin系统，日志文件存放在log目录下，按日期命名
    - 使用方式如下：

        ```python
            logging.debug('王承明debug message')
            logging.info('王承明info message')
            logging.warning('王承明warning message')
            logging.error('王承明error message')
            logging.critical('王承明critical message')
        ```
    - 缺省是系统所有信息输出到控制台，Error，Critical级别的输入出到日志文件
        
    >例如500的挂起错误，会输出到控制台和日志文件中    
    
- 异步新语法介绍    
    - 使用async+await语法后不用tornado.gen.Return，直接用return
    - 例如
    
        **老方式**

        ```python
        @tornado.gen.coroutine
        def GetUploadFileInfo(self, uuid):
            sqlstr = """SELECT - FROM "upload" WHERE uuid=$1;"""
            res = yield self.QuerySafe(sqlstr, uuid)
            if res:
                rv = res[0]
            else:
                rv = None
            raise tornado.gen.Return(rv)
        ```
        **新方式**
        
        ```python
        async def GetUploadFileInfo(self, uuid):
            sqlstr = """SELECT - FROM "upload" WHERE uuid=$1;"""
            res = await self.QuerySafe(sqlstr, uuid)
            if res:
                rv = res[0]
            else:
                rv = None
            return rv
        ```

    - 不加async前缀的函数无法调用加了async前缀的函数 *（同步函数无法调用异步函数）*
    - 使用await调用加了async前缀的函数为异步调用，异步函数要调用同步函数需要使用 **同步转异步**
    - 同步转异步调用方式
      
        **老方式**
        - class顶部需要写executor = ThreadPoolExecutor(64)
        - 同步函数需要加@run_on_executor的帽子才能通过yield调用
        
        **新方式**
        - 同步函数不需要增加任何多余帽子或者代码
        - 使用如下方式调用，sync_func为同步函数 args为参数列表展开传入，其中第一个参数为None就是让tornado自己创建executor
    
        ```python
        rv = await tornado.ioloop.IOLoop.current().run_in_executor(None, sync_func, args)
        ```

    - 关于异步http请求的变化，新的http异步请求代码如下
        > GET 请求(参数需要组合到url中)
        
        ```python
        http_client = tornado.httpclient.AsyncHTTPClient()
        try:
            resp = await http_client.fetch(url)
        except Exception as e:
            logging.error("Error: %s" % e)
            return ''
        return resp.body
        ```
        > POST 请求(body_args为body部分的参数内容)
        
        ```python
        http_client = tornado.httpclient.AsyncHTTPClient()
        try:
            resp = await http_client.fetch(url, method='POST', body_producer=body_args)
        except Exception as e:
            logging.error("Error: %s" % e)
            return ''
        return resp.body
        ```

- Postgresql操作的新语法介绍
    - 参数传递变化，主要变化是从%s变量占位符变成了$1、$2、$3 .... $n
      
        **老方法例子**

        ```python
        sqlstr = """SELECT - FROM "upload" WHERE uuid=%s;"""
        rv = yield self.QuerySafe(sqlstr, uuid)
        ```
        
        **新方法例子**
        
        ```python
        sqlstr = """SELECT - FROM "upload" WHERE uuid=$1;"""
        rv = await self.QuerySafe(sqlstr, uuid)
        ```
    - 参数和结果集中的对应类型变化
        - int型参数可以直接传入Python 的int型数字
        - json类型:
            - dict或者list不用json.dump后再做参数插入，直接当做参数传入即可
            - 查询出来的结果集中也直接是dict或者list，不用再做json.loads动作
        - 其他一些Postgresql和Python对应的类型见下面的链接
            
            >[Postgresql对应Python类型](https://magicstack.github.io/asyncpg/current/usage.html?highlight=json)
        
    - 插入指定时间格式的方法（Postgresql和Mysql都使用这种方式）
        - 老的库直接插入时间字符串类似这样2017-07-07 12:23:32
        - 新的库需要使用datetime型作为参数传入
        
            ```python
            need_time = utils.StrToDateTime('2017-07-07 12:32:23')
            ```
    
    
- MySql操作方式
    - 所有操作和Postgresql封装一致，同样的执行方式和结果获取
    - 和Postgresql操作不同的是
        - json.dumps(dict_or_list, ensure_ascii=False), 设置这个后，存入mysql的是正常的unicode编码中文，否则是ascii编码，显示会有问题
    - 时间字段类型选择
        - timestamp：时间戳，在navcat中显示为utc时间，程序还是能转换为本地时间
        - datetime*(建议采用)*：日期时间型，在navcat中心和程序中都为本地时间
    - 时间插入和Postgresql一样，采用Python的datetime类型

- MongoDB操作方式
    - 通过self.Mongo(集合名称)即可获取集合的操作句柄
    - 参考这个文档进行各种操作：[MongoDB-Motor操作文档](https://motor.readthedocs.io/en/latest/tutorial-asyncio.html)
    - 举例查询操作
    ```python
    coll = self.Mongo('jihe1')
    cursor = coll.find()
    rv = await cursor.to_list(length=100)
    print (rv)
    >>>
    [
        {'_id': ObjectId('5b715017dc26d546d6d21311'), 'aaa': 111.0}, 
        {'_id': ObjectId('5b728b9a5f627d195036178f'), 'a': 1, 'b': '中文abc'},
        {'_id': ObjectId('5b728ba05f627d1950361790'), 'a': 1, 'b': '中文abc'}
    ]
    ```
    - 返回值直接就是Python的list对象，可以直接在代码中操作

- 微服务使用方法
    > 根据具体使用情况，发现只有tick方式的service最好用，因此本引擎目前只支持tick方式

    - 微服务运行方式
        - 进入micro_service目录
        - python demo1_mq.py
    - 微服务程序编写方式
        - 参考demo1_mq.py
        - 复制一个demo1_mq.py根据需要编写代码

- 命令行工具使用方法
    > 有时候需要使用执行一次即退出的命令行工具模式的程序代码，HPA中有一个**cmd_tools.py**用于这种情况
    
    - 支持执行一次即退出
    - 只需编写Running()入口函数即可
    - 和HPA配置中的所有数据库，缓存的链接，可直接操作
    - 和主站一样的编写代码方式
    
    > 和微服务不同的是，命令行是执行一次程序即退出，释放所有资源，微服务为一个死循环，一直不停的执行OnTick()中的类容

- Html输入防注入过滤功能
    - 使用 [FilterHTML](https://github.com/dcollien/FilterHTML)
    - 常用的意见定义在utils.HtmlSpec.py文件中
    - 使用方式举例：
    
    ```python
        #content为html内容的str
        try:
            content = FilterHTML.filter_html(content, g_writelist)
        except Exception as e:
            print ('可能有非法字符')
    ```
    
    > 本功能可以过滤掉html内容str中的不合法的字符，保留合法的html
    
- 引擎中一些使用方法
    - 引用当前目录下的模块的方式
      
        **老的方式**

        ```python
        from utils import xxx
        ```
        
        **新的方式**
        
        ```python
        from .utils import xxx
        ```
        
    - 单进程用异步执行方式 *（service微服务）*
    
        ```python
        tornado.ioloop.IOLoop.current().run_sync(async_main)
        ```
    - 跨进程的全局变量使用
    
        ```python
        await GxAsyncRedis.Set() / Get()
        ```

- 其他的一些问题
    - **render是否异步？*- 是异步不过发现可以直接调用也可以await调用
    - **write（）后是否还需要调用finish（）？*- 仍然需要，不过可以直接使用finish(xxx)替代write
    - Tornado template中异步调用仍然**不可能**

_____________________________________________________________________


#### utils工具模块详解

- AESCrypt对称加密 *(AES:对称加密模块)* [参考文档](http://depado.markdownblog.com/2015-05-11-aes-cipher-with-python-3-x)
    - 主要功能：使用一个密钥对数据进行加密和解密工作
    - 每个不同的项目需要定义utils/define.py中的 **g_aes_prikey** 缺省密钥
    - 使用方式:
        ```python
            raw = '123123123sadadf中文abc！！。@*--'
            aec = utils.AESCrypt()
            ery_b64 = aec.encrypt(raw)
            print (ery_b64)
            >>> fw7tSg6ATbvvew6mMJ/Yqty7JdcgMQgsKCW5oPK1ltS8qHnoweBvQh8K4+aB5TqDxlXrmw7fiHmmwUUKl9IVHfdXiCNTXon0Uw2VTZphPio=
            raw_str = aec.decrypt(ery_b64)
            print (raw_str)
            >>> 123123123sadadf中文abc！！。@*--
        ```
- RSACrypt*(RSA： 非对称加密模块)*
- 本模块功能：
    - 数据加密功能：公钥加密，私钥解密。
    - 签名验证功能：私钥数字签名，公钥验证。
    - 产生公钥和私钥文本：支持私钥密码功能，各个函数密码作为可选参数传入
    - 用法介绍：
        - 产生密钥对，密钥密码可选填
            > 返回：private_key, public_key
    
            ```python
            rsa = utils.RSACrypt()
            pri,pub = rsa.generate_keys('123kkk')
            print (pri,pub)
            ```
        
        - 私钥数字签名，公钥验证功能
          
            - 使用私钥对数据签名，返回签名后的base64字符串，密钥密码可选填
            > sign(self, data, prikey='', password=None)
            RSA签名,参数prikey私钥，可以通过generate_keys生产私钥，签名时需要传入私钥
            return: 返回base64的签名字符串
            
                ```python
                s = rsa.sign(a, pri, '123kkk')
                print (s)
                ```
            
            - 使用公钥验证数据签名，返回True or False，密钥密码可选填
            > verify(self, data, signature, pubkey='', password=None)
            RSA验签参数pubkey，可以通过generate_keys生产公钥，前ing是需要传入公钥用于验证签名
            return：验证成功True,验证失败False
            
                ```python
                v = rsa.verify(a, s, pub, '123kkk')
                print (v)
                ```
        
        - 公钥加密，私钥解密功能
            - 使用公钥对数据加密，返回base64字符串， 密钥密码可选填
                > encrypt(self, data, pubkey='', password=None),加密函数，传入pubkey公钥进行加密
        返回密文的base64字符串
        
                ```python
                e = rsa.encrypt(a, pub, '123kkk')
                print (e)
                ```
            
            - 使用私钥对数据解密，密钥密码可选填
                > decrypt(self, encrypt_data, prikey='', password=None) 公钥加密，传入私钥进行解密,返回解密后的原文
                
                ```python
                d = rsa.decrypt(e, pri, '123kkk')
                print (d)
                ```
    - 应用实例，js使用公钥加密传输密码，Python用私钥解密密码
    
    - 前端js使用rsa的方式例子
    ```javascript
    <script src="https://cdn.jsdelivr.net/npm/node-forge@0.7.0/dist/forge.min.js"></script>
    
    var pubkey = '-----BEGIN PUBLIC KEY-----公钥文本省略...';
    var prikey = '-----BEGIN RSA PRIVATE KEY-----私钥文本省略...';

    //实例化对象
    var privateKey = forge.pki.privateKeyFromPem(prikey);
    var publicKey = forge.pki.publicKeyFromPem(pubkey);

    //公钥加密
    var t = publicKey.encrypt("abc123!@#", 'RSA-OAEP', {
        md: forge.md.sha1.create(),
        mgf1: {
            md: forge.md.sha1.create()
        }
    });
    //输出base64
    var y = forge.util.encode64(t);
    console.log('encrypted', y, '\n')

    //私钥解密
    t = privateKey.decrypt(forge.util.decode64(y), 'RSA-OAEP', {
        md: forge.md.sha1.create(),
        mgf1: {
            md: forge.md.sha1.create()
        }
    });
    console.log('decrypted', t, '\n')
    ```
    
    - 前端js加密。后端Python解密实例 *(目前rsa js端加密不支持中文)*
    ```
    <script src="https://cdn.jsdelivr.net/npm/node-forge@0.7.0/dist/forge.min.js"></script>
    
    var pubkey = '-----BEGIN PUBLIC KEY-----公钥文本省略...;
    var publicKey = forge.pki.publicKeyFromPem(pubkey);
    //公钥加密
    var t = publicKey.encrypt("abc123!@#", 'RSA-OAEP', {
        md: forge.md.sha1.create(),
        mgf1: {
            md: forge.md.sha1.create()
        }
    });
    //输出base64， entext返回给后端进行解密处理，传输过程中处于加密状态
    var entext = forge.util.encode64(t);
    ```
    
    entext返回给后端进行解密处理，传输过程中处于加密状态

    ```python
    prikey_str = """-----BEGIN RSA PRIVATE KEY-----私钥文本省略..."""
    rsa = utils.RSACrypt()
    detext = rsa.decrypt(entext, prikey_str)
    ```
    
    detext为Python解密后的字符串
    
- WordCheck输入字符串校验功能
   	- 常用的字符串检查类，检查各种字符串是否符合所需格式要求，使用正则工具
      	
       	- 目前有如下检测功能，可根据需要自行按照模块的编写模式添加
    - CheckEmail(text)
    
    > 检查email格式是否符合要求,返回True表示正常，False表示有非法字符
        
    -  CheckFileMD5(text)
        
    > 检查是否MD5格式字符串，32位16进制
        
    - CheckFilePath(text)
        
    > 检查文件路径
        
    - CheckFileUUID(text)
        
        > 检查文件uuid格式是否符合要求
        
    - CheckNubmer(text)
        
    > 检查字符串是否数字
        
    - CheckOverseaTelphone(text)
        
        > 检查海外手机号是否符合要求,返回True表示正常，False表示有非法字符
        
    - CheckSearchBar(text)
        
        > 检查搜索框类的输入是否有sql注入字符串，返回True表示正常，False表示有非法字符
    - CheckTelphone(text)
        
        > 检查国内手机号是否符合要求,返回True表示正常，False表示有非法字符
        
    - CheckUUID(text)
        
        > 检查uuid格式是否符合要求
        
    - CheckUsername(text)
        
    > 检查用户名是否符合要求, 返回True表示正常，False表示有非法字符
        
    - CleanUrl(text)
        
    > 清洗URl，如果出现http://这种，就去掉，没有就不管,只保留网址部分
    
   	- GetAddress(text)
   	    
   	> 获取地址，省市这种，如果没有返回其他地区
   	    
   	- GetDateStr(text)
   	    
   	> 获取日期部分字符串
   	    
   	- WashEmail(text)
   	    
   	> 清洗Email ，错误会返回None
   	    
   	- WashSQLParmas(parm)
   	    
   	    > 清洗sql参数，如果有不合格的参数，直接清洗为'',主要用于搜索框中输入的字符检查是否有sql注入等
   	
- switch放置switch case语法编写多逻辑功能
    - Python本身没有switch case语法，因此方便多逻辑编写，写了一个这个方法，具体使用方式参考下面代码
    
    ```python
     for case in utils.switch(condition_parms):
        if case('cond_1'):
            # do something in cond_1
            break
            
        if case('cond_2'):
            # do something in cond_2
            break
            
        else:
            # like default case
            # do someting in default
            break
    ```

- CreateGUID(prefix=None, guidlen=29, withpunctuation=False, islower=False)
    
> 产生一个HPA标准的UUID字符串
    
- CreateOPENAPI(prefix=None, guidlen=11, withpunctuation=False)
    
> 产生一个open user使用的app_id和app_secret
    
- DateTimeToFriendly(target_time: datetime.datetime) -> str
    > 显示传入时间的友好信息，例如：几分钟前，几小时前，几天前，几分钟后，几天后，几个月后
    :param target_time: 目标时间，会用它和当前时间比较厚格式化输出
    :return: 友好时间信息的str

- DateTimeToStr(dbdatetime)
    > 将datetime类型格式化为：YYYY-MM-DD HH:MM:SS
    并返回字符串

- DateTimeToTimeStamp(dbdatetime)
    
> datetime类型转换为时间戳
    
- DeepEncrypPassword(username, orgpassword, datetimestr)
    
> 通过加入用户名，时间戳作为参数进行的加密运算，避免库里直接复制密码就可以生效的漏洞
    
- FixedEmailAddress(email_str)
    
> 对email地址整形，中间有空格的，前后有空格的都去掉，让发送邮件有点鲁棒性
    
- FixedEmailList(email_str)
    
> 处理多个email组成的字符串，把分隔符号替换为标准的 ";"
    
- GenerateRandCode(length=6)
    
> 随机产生一个手机校验码字符串，6位全数字
    
- GenerateRandomIntNum(min, max)
    
> 产生一个整数范围内的随机数字
    
- GenerateRandomString(length, withpunctuation=True, withdigit=True)
    > 产生一个随机字符串
    参数：withpunctuation是否带标点符号,withdigit是否带数字

- GetDataSensitive(sensitive_data)
    > 手机和邮箱脱敏功能

    - 手机脱敏
        - 大陆：显示前 3 位 + **** + 后 4 位。如：137****9050
    - 邮箱脱敏
        - @前面的字符显示3位，3位后显示3个*，@后面完整显示如：con***@163.com
             如果少于三位，则全部显示，@前加***，例如mingming@163.com则显示为mi***@163.com

- GetDateTime(timestamp=None)
    > 获取标准时间格式字符串，没有timestamp时间戳参数就是当前时间
    格式为：YYYY-MM-DD HH:MM:SS

- GetFileMD5(fdfile)
    
> 获取文件的MD5码,:param fdfile: 文件open后的fd，外部自行close(),返回统一为大写
    
- GetFlowNumber(number)
    
> 生成流水号:201808150000001,:param number: int 业务ID,:return: str 流水号
    
- GetMD5(orgstr, islower=False)
    
> 获取一个字符串的MD5,返回统一为大写,可以通过islower
    
- GetRealStr(sorb_str)
    
> for python3,无论是str类型还是bytes（前缀b）类型，用这个函数都返回一个str类型，稳稳的幸福
    
- GetSecondByUnit(unit)
    > 根据单位得到这个单位的时间长度（秒单位）
    
    - unit参数:返回秒数
        - 'hour':3600
        - 'day':86400
        - 'month':2592000
        - 'year':31104000

- GetStrCoding(rawstr)
    
> 判断某个字符串的编码类型，如果有中文和英文比较精确，如果只有英文就无法判断
    
- PrintSqlStr(sqlstr, parm, ctype='pg', log_level='info')
    
> 打印代码中的数据库语句为可执行的样子，方便调试
    
- StrSplit(str, width)
    > 按长度分隔字符串，支持汉字
    
    - :param str: 字符串
    - :param width: 分隔的长度
    - :return: 分隔后的list

- StrToDateTime(datetimestr)
    
> 转换字符串为datetime类型，字符串支持类似2018/8/15这种格式
    
- TestExeTime(func)
    > 计算运行时间的帽子

    - 加到函数头上，可以打印出函数执行时间
    ```python
    @TestExeTime
    def func():
        pass
    ```
    
- TimeStampToDateTime(timestamp, convert_to_local=True)
    
> 时间戳转换成datetime,convert_to_local: 是否转化为本地时间
    
- isFloatEqual(f1, f2)
    
> 判断两个浮点数是否相等，精度为 0.000001,两个float的差小于这个误差的都认为相等
    
- Hbase64_encode(raw_str):
    
    > HPA统一封装的url安全的base64编码函数,返回str类型的base64编码
    
- Hbase64_decode(hbase64_str, need_byte=False):
    > HPA统一封装的url安全的base64解码函数,返回str类型的解码后字符串
    need_byte=显式指明需要使用byte类型的返回值
    
    - 如果need_byte为False的情况下，返回值如果是不能被转换为str的，会直接返回byte类型
    - 支持不转换的标准base64字符串的解密 *(不做+转换为-,/转换为_)*
    
- handler中获取参数使用：```get_argument_safe('name','default_value')```
    
> 做了sql注入等参数清洗动作 
    
- TransToPDF(htmstr, filename)
    > 转换html字符串为pdf文件，一般用于简历下载pdf
    
    ```python
    htmlstr = self.render_string('trans_to_pdf_test.html')
    await utils.TransToPDF(htmlstr, '1.pdf')
    ```
    
    - 注意事项：用于转换pdf的html参考trans_to_pdf_test.html文件，head部分必须添加：<meta http-equiv="Content-Type" content="text/html; charset=utf-8"/>才能输出pdf中文
    
    
#### session相关使用方法

- HPA中，支持两种类型的session
    1. 普通web使用的基于cookie的session
    2. 用于微信小程序的基于token的session
    
- 使用方法
    - 基于cookie的session
        1. basehandler继承xSessionMixin
        2. 通过self.session.set,get,del等进行操作，浏览器会生成对应的cookie，cookie名称通过修改g_cookie_session_id,和g_cookie_visit_session_id来指定
        3. 其中session_id一般用于登录后的用户信息存储，visit_session_id一般用于非登录用户的信息存储
        4. 用户注销后需要显式的调用cleanup接口清理session
        5. 缺省情况下，session保留时间为24小时，web会有机制保证和cookie的有效时间一致
        
    - 基于token的session
        1. basehandler继承xMinaSessionMixin
        2. 通过self.mina_session.set,get,del等进行操作，浏览器不会生成cookie，整个运作都是无状态模式
        3. 所有get/post调用必须带msid，这个msid通过self.mina_session.create_mina_session_id函数创建
        4. 用户注销后需要显式的调用cleanup接口清理session
        5. 缺省情况下，session保留时间为24小时


#### Aliyun队列相关使用方法

- QueueModel（队列模式）
    1. 先在阿里云上创建一个队列，如名字为xbmq-test
    2. 在本SDK中实例化这个队列（在文件末尾添加）
    ```python
    test_mq = QueueModel(queue_name='xbmq-test')
    ```
    3. 生产者处通过如下代码来发送消息
    ```python
    from services.aliyun_mns_sdk import test_mq
    yield test_mq.pulisher(json.dumps(jsondata))
    ```
    4. 消费者处通过
    ```python
    msgbody_rv = yield test_mq.consumers(num_of_msg=16)
    ```
    批量获取消息，最佳实践参考demo1_mq.py（微服务模式）
    
- TopicModel（订阅模式）
    1. 阿里云上创建一个订阅主题（topic）
    2. 在队列管理中为每个消费者创建一个“消费者队列”
    3. 然后在“订阅详情”中为每个消费者创建一个对应的“订阅”，推送方式使用“队列”，并填写第2步创建好的那个“消费者队列”
    4. 在本SDK中实例化这个订阅（文件末尾添加）
    ```python
    test_topic = TopicModel(topic_name='xbtopic-test') #topic生产者
    ```
    5. 生产者处通过如下代码发送消息
    ```python
    from services.aliyun_mns_sdk import test_topic
    yield test_topic.pulisher('aaaa1','s01')#第一个参数为消息体，第二个参数为tag（区分订阅者的）
    ```
    6. 消费者处通过正常的QueueModel自己“消费者队列”的方式获取消息体
    7. 这里没有做完的是，topic方式的消息还是原始的json格式，需要在从queue中取出来后使用的时候根据需要自行提取内容


#### Testhandler，所有的功能演示代码块
    ```python

class TestHandler(ctrl_base.BaseHandler, xMinaSessionMixin):
    async def get(self):
        logging.debug('王承明debug message')
        logging.info('王承明info message')
        logging.warning('王承明warning message')
        logging.error('王承明error message')
        logging.critical('王承明critical message')

        #查询
        # sql = """show time zone;"""
        # rv = await self.QuerySafe(sql)
        # print (rv)
        #
        # # sql = """SELECT * FROM "upload";"""
        # sql = """SET TIME ZONE PRC;"""
        # await self.ExecSafe(sql)
        # print ('ssssssssssssssssss')
        #
        # sql = """show time zone;"""
        # rv = await self.QuerySafe(sql)
        # print (rv)
    
        #--时间插入测试
        # timestr = '2017-07-07 12:32:23'
        # _intime = utils.StrToDateTime(timestr)
        #
        # sql ="""INSERT INTO "testtime"("t1", "t2", "t3")
        #         VALUES ($1, $2, now()); """
        # rv = await self.ExecSafe(sql,_intime,_intime)
        #
        #
        # WHERE IN 用法
        # co = ['南开大学','中国人民大学']
        ##要做一个类似这样的查询sql = """SELECT * FROM opt_school WHERE "name" IN ('南开大学','中国人民大学');"""
        #使用这种方式传入参数
        # sql = """SELECT * FROM opt_school WHERE "name" = ANY($1);"""
        # rv = await self.QuerySafe(sql, co)
        # for row in rv:
        #     print(row)
    
        # %xxx%模糊查询参数用法
        # co = '%%大学%%'
        # #要做一个类似这样的查询sql = """SELECT * FROM opt_school WHERE "name" LIKE '%技大学%';"""
        # #使用这种方式传入参数
        # sql = """SELECT * FROM opt_school WHERE "name" LIKE $1;"""
        # rv = await self.QuerySafe(sql, co)
        # for row in rv:
        #     print(row)
    
        #timestamptz类型转换本地时区
        # t2
        #from pytz import timezone
        # tzchina = timezone('Asia/Chongqing')
        # a = t.replace(tzinfo=timezone('UTC')).astimezone(tzchina)
        # print (t)
        # print (a)
        # r = utils.DateTimeToStr(row['dd'])
        # print ('-------------', r)
    
        #---------------------------------------------------------------------------------------------
    
        #插入
        # a = {
        #     'id': 55555,
        #     'name': 'jkhjkhh',
        #     'price': '中文123abc'
        # }
        # b= ['a',1,'中']
        # sql ="""INSERT INTO "upload"("uuid", "suffix", "uploadtime", "filepath", "fileurl", "thumbfilepath", "thumbfileurl",
        #         "thumbsize", "filesize", "filebyte", "status", "comment", "flag", "tjb")
        #         VALUES ($1, $2, now(), $3, $4, $5, $6, $7, $8, $9, 'nomoal', $10, $11, $12); """
        # rv = await self.ExecSafe(sql, 'uuid118','suffix111','filepath111','fileurl111',None, None, None,
        #                          '123', 887777,'cccccc','fffff', b)
        # ---------------------------------------------------------------------------------------------
    
        #事务
        # a = {
        #     'id': 55555,
        #     'name': 'jkhjkhh',
        #     'price': '中文123abc'
        # }
        # b= ['a',1,'中']
        # for i in range(3):
        #
        #     sql ="""INSERT INTO "upload"("uuid", "suffix", "uploadtime", "filepath", "fileurl", "thumbfilepath", "thumbfileurl",
        #             "thumbsize", "filesize", "filebyte", "status", "comment", "flag", "tjb")
        #             VALUES ($1, $2, now(), $3, $4, $5, $6, $7, $8, $9, 'nomoal', $10, $11, $12); """
        #     rv = self.TransExecSafe(sql, 'uuid97722228%d'%i,'suffix111','filepath111','fileurl111',None, None, None,
        #                              '123', 88777+i,'cccccc','fffff', b)
        #
        #
        # sql = """UPDATE "upload" SET flag='77' WHERE  uuid='uuid877222281';"""
        # self.TransExecSafe(sql)
        #
        # rv = await self.TransCommitSafe()
        # ---------------------------------------------------------------------------------------------
    
        #redis test
        # a = {
        #     'id': 55555,
        #     'name': 'jkhjkhh',
        #     'price': '中文123abc'
        # }
        # b= ['a',1,'中']
        # # await GxAsyncRedis.Set('kkk111','中文abc', 10)
        # await GxAsyncRedis.HSet('hhh1','k1',a)
        # await GxAsyncRedis.HSet('hhh1','k2',b, 60)
        # # await GxAsyncRedis.Set('kkk113',b)
        #
        #
        # print(1,await GxAsyncRedis.HGet('hhh1','k1'))
        # print(2,await GxAsyncRedis.HGet('hhh1','k2'))
        # print(2,await GxAsyncRedis.FlushAll())


        # print(2,await GxAsyncRedis.Del('hhh1'))
        # print(3,await GxAsyncRedis.Get('kkk112'))
        # print('done')
    
        # ---------------------------------------------------------------------------------------------
        #同步转异步测试
        # await tornado.ioloop.IOLoop.current().run_in_executor(None, self.sync_func_test)
        # self.sync_func_test()
    
        # ---------------------------------------------------------------------------------------------
        #数据库缓存测试
        # await GxAsyncRedis.FlushAll()
        # fileinfo = await self.GetUploadFileInfo(uuid='7C1597D07912878D736A88B729827174-JPG')
        # print (fileinfo)
        #
        #
        #
        # await tornado.gen.sleep(0.5)
    
        # ---------------------------------------------------------------------------------------------
        # msg = {'a':1,'b':2,'c':'中文abc'}
        # await test_mq.producer(json.dumps(msg))
    
        # ---------------------------------------------------------------------------------------------
        # MySql链接测试
        # sqlstr = """SELECT * FROM upload;"""
        # r = await self.QuerySafe_mysql(sqlstr)
        # for row in r:
        #     print ('-------------------------------------------------')
        #     print (row)
    
        # -----
        # mysql时间类型测试
    
        # timestr = '2017-07-07 12:32:23'
        # _intime = utils.StrToDateTime(timestr)
        # sql ="""INSERT INTO `testtime`(`t1`, `t2`, `t3`)
        #         VALUES (%s,%s,%s); """
        # rv = await self.ExecSafe_mysql(sql,_intime,_intime,_intime)
        #
        # sql = """SELECT * FROM `testtime`;"""
        # rv = await self.QuerySafe_mysql(sql)
        # # print (rv, type(rv))
        # for row in rv:
        #     t1 = row['t1']
        #     t2 = row['t2']
        #     t3 = row['t3']
        #     print (t1, utils.DateTimeToStr(t1))
        #     print (t2, utils.DateTimeToStr(t2))
        #     print (t3)


        # MySql 插入
        # a = {
        #     'id': 55555,
        #     'name': 'jkhjkhh',
        #     'price': '中文123abc'
        # }
        # b= ['a',1,'中']
        # sql ="""INSERT INTO `upload`(`uuid`, `suffix`, `uploadtime`, `filepath`, `fileurl`,`filesize`, `filebyte`, `status`, `comment`, `flag`)
        #         VALUES (%s, %s, now(), %s, %s, %s, %s, 'normal', %s, %s); """
        # rv = await self.ExecSafe_mysql(sql, 'uuidq002','suffix111','filepath111','fileurl111','中文abc', 887777,json.dumps(b,ensure_ascii=False),None)
        #---------------------------------------------------------------------------------------------
    
        # Mysql事务
        # for i in range(3):
        #     b= ['a', i,'中']
        #     sql ="""INSERT INTO `upload`(`uuid`, `suffix`, `uploadtime`, `filepath`, `fileurl`,`filesize`, `filebyte`, `status`, `comment`, `flag`)
        #             VALUES (%s, %s, now(), %s, %s, %s, %s, 'normal', %s, %s); """
        #     self.TransExecSafe_mysql(sql, 'uuid00%s'%(i+71),'suffix111','filepath111','fileurl111','123', 887777,json.dumps(b),None)
        #
        # sql = """UPDATE `upload` SET flag='88888888888889999999999999999999999888888888787' WHERE  uuid=%s;"""
        # self.TransExecSafe_mysql(sql, 'uuid109')
        #
        # rv = await self.TransCommitSafe_mysql()
        # print (rv)
    
        # ---------------------------------------------------------------------------------------------
        #mongodb 测试
        #查询数据
    
        from pymongo import InsertOne, DeleteMany, ReplaceOne, UpdateOne, DESCENDING
        # coll = self.Mongo('jihe1')
        # test_find_uuid = None
        # for i in range(100):
        #     uuid = utils.CreateGUID('KOK')
        #     if i == 77:
        #         test_find_uuid = uuid
        #     rv = await coll.insert_one({
        #         '_id': uuid,
        #         'j' : utils.GenerateRandomIntNum(1,1000)
        #     })
        #     print (rv)
        # result = await coll.bulk_write([
        #         DeleteMany({}),  # Remove all documents from the previous example.
        #         InsertOne({'j':1, 'k':10}),
        #         InsertOne({'j': 2.5, 'k':1}),
        #         InsertOne({'j': 3,'k':5}),
        #         UpdateOne({'j': 4.5,'k':2}, {'$set': {'foo': 'bar'}}, upsert=True),
        #         UpdateOne({'j': 6}, {'$inc': {'j': 1}}, upsert=True),
        #         ReplaceOne({'j': 1}, {'j': 2,'a':'aa1','b':'bb1','c':'cc1'})]
        # )
        # rv = await coll.create_index([("j",DESCENDING)])
        # print ('----->',rv)
    
        # print(result.bulk_api_result)
        # cursor = coll.find({'_id': {'$gt': 1}},{'j':1,'_id':0}).sort('j', DESCENDING)
        # rv = await cursor.to_list(length=100)
        # print (rv, type(rv))
    
        # cursor = await coll.find_one({'_id': test_find_uuid})
        # print (cursor)


        #---------------------------------------------------------------------------------------------
        # session测试
        # opt = self.get_argument_safe('t')
        # if opt == '0':
        #     msid = self.mina_session.create_mina_session_id()
        #     self.finish(msid)
        #     return
        # elif opt == '1':
        #     await self.mina_session.set('testkey1','aaaaaaaaaaaaaa中文',10)
        #     await self.mina_session.set('testkey2','bbbbbbbbbbbbbb中文',20)
        #     # await self.visitsession.set('testkey3','ccccccccc中文',60)
        # else:
        #     a = await self.mina_session.get('testkey1')
        #     b = await self.mina_session.get('testkey2')
        #     # c = await self.visitsession.get('testkey3')
        #     self.finish('%s---------%s'%(a,b))
        #     return
    
        # ---------------------------------------------------------------------------------------------
        # pdf转换测试
        # htmlstr = self.render_string('trans_to_pdf_test.html')
        # htmlstr = self.render_string('index.html')
        # await utils.TransToPDF(htmlstr, '1.pdf')
    
        # ---------------------------------------------------------------------------------------------
        # 邮件发送测试
        #第一种原始方式通过模版生成htmlstr，这种方式适用于不在handler中的方式，可以单独在类似service的进程中调用，不用引入tornado
        # with open('../templates/mail_tpl/test_mail_tpl.html', 'r') as fi:
        #     mail_tpl = tornado_tmpl.Template(fi.read())
        #     fi.close()
        # htmlstr = mail_tpl.generate(g_admin_domain=g_admin_domain,
        #                             short_url=short_url,
        #                              full_name=full_name,
        #                              year_month=year_month,
        #                              com_name=com_name)
    
        #第二种方式，使用tornado 的handler直接生产htmlstr，适合在tornado web的handler中调用
        # htmlstr = self.render_string('mail_tpl/test_mail_tpl.html',
        #                                 g_admin_domain='www.goodluck.com',
        #                                 short_url='ssss.com',
        #                                 full_name='好运连连',
        #                                 year_month='2020-12-12',
        #                                 com_name='明明的超级大托拉斯'
        #                                 )
        # #具体发送
        # subject = '收到的人超级好运，强大的测试邮件'
        # res = await G_SCEMS.SendEmail(['8537798@qq.com', 'tobyking80@163.com'], 'service@goodluck.com', '牛牛好运平台', subject, htmlstr)
        # print (res)
        # res = json.loads(res)
        # if res.get("message") and res.get("message") != "success":
        #     return 0
        # return 1
    
        # ---------------------------------------------------------------------------------------------
        # 短信发送测试
        # tplid = '20978'
        # datas = {"%randcode%": '123456', "%keeptime%": g_sms_keeptime}
        # rvjson = await G_SCSMS.SC_SendSMS('15881075900', tplid, json.dumps(datas))
        # print (rvjson)
        # if not rvjson or rvjson['res_code'] != 0:
        #     self.finish(ReturnCode(900, '获取验证码失败!请稍后重试!'))
        #     return


        # ---------------------------------------------------------------------------------------------
        # 百度dwz短网址接口调试
        sw = await GDWZSDK.GetShortUrl('http://www.vastsea.com/joinus', '1-year')
        print (sw)
    
        self.finish('done')
    
    ```
