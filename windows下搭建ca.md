<ul id="tree" class="ztree"></ul>
<script type="text/javascript" src="http://wp.simman.cc/Public/file_dldq_org/jquery-1.10.2.min.js"></script>
<script type="text/javascript" src="http://wp.simman.cc/Public/file_dldq_org/jquery.ztree.all-3.5.min.js"></script>
<script type="text/javascript" src="http://wp.simman.cc/Public/file_dldq_org/jquery.ztree_toc.js"></script>
<SCRIPT type="text/javascript" > 
    <!--
    var winWidth = 0;
    var winHeight = 0;
    function findDimensions() //函数：获取尺寸
    {
    //获取窗口宽度
    if (window.innerWidth)
        winWidth = window.innerWidth;
        else if ((document.body) && (document.body.clientWidth))
        winWidth = document.body.clientWidth;
        //获取窗口高度
        if (window.innerHeight)
        winHeight = window.innerHeight;
        else if ((document.body) && (document.body.clientHeight))
        winHeight = document.body.clientHeight;
        //通过深入Document内部对body进行检测，获取窗口大小
        if (document.documentElement && document.documentElement.clientHeight && document.documentElement.clientWidth)
        {
            winHeight = document.documentElement.clientHeight;
            winWidth = document.documentElement.clientWidth;
        }
    }

    findDimensions();

    $(document).ready(function(){ 
        $('#tree').ztree_toc({ 
            is_auto_number:true, 
            documment_selector:'.markdown-body', 
            ztreeStyle: { 
                width:'260px', 
                height: winHeight,
                overflow: 'auto', 
                position: 'fixed', 
                'z-index': 2147483647, 
                border: '0px none', 
                left: '0px', 
                top: '0px' ,
            } 
        }); 
    }); 
    //--> 
</SCRIPT> 

<article class='markdown-body' style="width:85%; margin-left:25%;border: 1px solid #CACACA; padding: 30px; border-radius:3px;" >

#Windows下搭建CA

2014年9月17日

##前言

本文技术来源于《Java加密与解密的艺术》一书，对于本文看的不是很明白的可以查阅该书

##准备

在<http://slproweb.com/products/Win32OpenSSL.html>页面下下载最新的Windows版OpenSSL。这里选用`Win64 OpenSSL v1.0.1i Light`演示数字证书构建相关操作

###环境变量

设置系统变量`OpenSSL_Home`，并将其指向你的安装目录(`C:\OpenSSL`)。同时设置其执行目录(`%OpenSSL_Home%\bin;`)加入系统变量`Path`中。如下：

	＃OpenSSL系统变量配置
	
	OpenSSL_Home	C:\OpenSSL
	Path			%OpenSSL_Home%\bin

完成上述操作后，在命令行执行`OpenSSL`命令，如下：
	
	＃OpenSSL命令行操作
	
	C:\User\itan.feng>openssl
	OpenSSL>_
	
###工作目录

大家OpenSSL配置文件openssl.cfg(`%OpenSSL_Home%\bin\opensssl.cfg`)，找到配置`[CA_default]`，如下：
	
	＃OpenSSL初始配置
	
	####################################################################
	[ CA_default ]
	
	dir		        = ./demoCA         # Where everything is kept
	certs		    = $dir/certs       # Where the issued certs are kept
	crl_dir		    = $dir/crl         # Where the issued crl are kept
	database	    = $dir/index.txt   # database index file.
	#unique_subject	= no               # Set to 'no' to allow creation of
					                   # several ctificates with same subject.
	new_certs_dir	= $dir/newcerts    # default place for new certs.

	certificate	    = $dir/cacert.pem  # The CA certificate

注意变量`dir`，它指向的是CA的工作目录，本文将路径指向`C:/ca`作为工作目录，对变量dir做相应的修改。对于其他变量，我们无需修改。  
如果不方便修改OpenSSL默认配置文件，可以通过设置变量`“OPENSSL_CONF”`指定配置文件路径。如下：

	＃命令指定配置文件路径
	
	set	OPENSSL_CONF = C:\openssl.cfg
	
若该方法无法设置成功请自己到系统变量中进行设置

	＃系统变量自定配置文件路径
	
	OPENSSL_CONF = C:\openssl.cfg
	
建立CA工作目录后，我们需要构建一些子目录，用于存放证书、密钥等。完整命令代码清单如下：

	＃构建CA子目录
	
	echo  构建已发行证书存放目录  certs
	mkdir certs
	echo  构建新证书存放目录 newcerts
	mkdir newcerts
	echo  构建私钥存放目录  private
	mkdir private
	echo  构建证书吊销列表存放目录 crl
	mkdir crl
	
我们将在创建证书时用到上述目录，最终在certs目录中获得证书文件。  
接下来我们需要构建一些文件，完整命令代码清单如下：

	＃构建相关文件
	
	echo  构建索引文件 index.txt
	echo  0>index.txt
	echo  构建序列号文件 serial
	echo  01>serial
	
完成上述操作后，我们就可以进行证书的构建和签发工作了。

##构建根证书

###构建随机数文件

在构根证书之前，需要构建随机数文件（`.rand`），完整命令代码清单如下：

	＃构建随机数命令
	
	openssl rand -out private/.rand 1000
	
各参数的含义如下：  
rand     随机数命令  
-out     输出文件路径，这里将随机数文件输出到private目录下。  
这里的1000，只用来产生伪随机字节数。  
上述命令执行结果如下

	＃构建随机数文件
	
	Loading 'screen' into random state - done

注意如果在windows7中未能使用管理员权限执行上述命令，将无法创建随机数文件，如下
	
	＃构建随机数文件失败
	
	Loading 'screen' into random state - done
	unable to write 'random state'

### 生成 CA 证书的 RSA 密钥对

OpenSSL通常使用PEM（Privacy Enbanced Mail，隐私增强邮件）编码格式保存私钥。  
接下来，我们需要构建根证书密钥（`ca.key.pem`），完整命令代码清单如下：

	＃构建根证书私钥命令
	
	openssl genrsa -aes256 -out private/ca.key.pem 2048
	
各参数的含义如下：  

	genrsa		产生RSA密钥命令
	-aes256		使用AES算法（256位密钥）对产生的私钥加密。可选算法包括DES、DESede、IDEA和AES。
	-out		输出路径，这里指private/ca.key.pem。
这里的2048，指密钥长度位数，默认长度为512为。  
上述命令执行结果如下：

	＃构建根证书私钥
	
	Loading 'screen' into random state - done
	Generating RSA private key, 2048 bit long modulus
	.....................................+++
	...+++
	e is 65537 (0x10001)
	Enter pass phrase for private/ca.key.pem:
	Verifying - Enter pass phrase for private/ca.key.pem:

这时我们需要输入根证书密码`“www.aios.com”`  

###生成 CA 证书请求

完成密钥的构建操作后，我们需要生成根证书签发的申请文件（`ca.csr`），完整命令代码清单如下：

	＃生成根证书签发得申请命令
	
	openssl req -new -key private/ca.key.pem -out private/ca.csr
	
各参数的含义如下：
	
	req		产生证书签发申请命令		
	-new	产生新请求
	-key	密钥，这里为private/ca.key.pem文件
	-out	输出路径，这里为private/ca.csr文件
	
上述命令执行结果如下：

	#生成根证书签发申请
	
	Enter pass phrase for private/ca.key.pem:
	Loading 'screen' into random state - done
	You are about to be asked to enter information that will be incorporated into your certificate request.
	What you are about to enter is what is called a Distinguished Name or a DN.
	There are quite a few fields but you can leave some blank
	For some fields there will be a default value,
	If you enter '.', the field will be left blank.
	-----
	Country Name (2 letter code) [AU]:CN
	State or Province Name (full name) [Some-State]:BJ
	Locality Name (eg, city) []:BJ
	Organization Name (eg, company) [Internet Widgits Pty Ltd]:Tech.Settion
	Organizational Unit Name (eg, section) []:AIOS Inc.
	Common Name (e.g. server FQDN or YOUR name) []:Insight RootCA
	Email Address []:

	Please enter the following 'extra' attributes
	to be sent with your certificate request
	A challenge password []:
	An optional company name []:

这时我们需要输入根证书密码并填写证书相关信息字段 

###对 CA 证书请求进行签名

得到根证书签发申请文件后，我们可以将其发送给CA机构签发。当然，我们也可以自行签发根证书。签发根证书完整命令清单如下：

	＃签发根证书命令
	
	openssl x509 -req -days 10000 -sha1 -extensions v3_ca -signkey private/ca.key.pem -in private/ca.csr -out certs/ca.cer
	
各参数的含义如下：

	x509			签发X.509格式证书命令。
	-req			证书输入请求。
	-days			🈶效天数，这里为10000天
	-sha1			证书摘要算法，这里为SHA1算法
	-extensions		按OpenSSL配置文件v3_ca项添加扩展
	-signkey		自签名密钥，这里为private/ca.key.pem
	-in				输入文件，这里为private/ca.csr
	-out			输出文件，这里为certs/ca.csr
	
上述命令执行结果，如下：

	#签发根证书
	
	Loading 'screen' into random state - done
	Signature ok
	subject=/C=CN/ST=BJ/L=BJ/O=Tech.Settion/OU=AIOS Inc./CN=Insight RootCA
	Getting Private key
	Enter pass phrase for private/ca.key.pem:
	
这时我们需要输入根证书密码`“www.aios.com”`

###对 CA证书 进行转换

OpenSSL产生的数字证书不在Java语言环境中直接使用，需要将其转化为PKCS#12编码格式。完整命令代码清单如下：

	＃根证书转换命令
	
	openssl pkcs12 -export -cacerts -inkey private/ca.key.pem -in certs/ca.cer -out certs/ca.p12
	
各参数的含义如下：
	
	pkcs12		PKCS#12编码格式证书命令
	-export		导出证书
	-cacerts	仅导出CA证书
	-inkey		输入密钥，这里为private/ca.key.pem
	-in			输入文件，这里为certs/ca.cer
	-out		输出文件，这里为certs/ca.p12
	
上述命令执行结果如下：

	＃根证书转换
	
	Loading 'screen' into random state - done
	Enter pass phrase for private/ca.key.pem:
	Enter Export Password:
	Verifying - Enter Export Password:
	
这时我们需要输入根证书密码`www.aios.com`
	
###查看密钥库信息

个人信息交换文件（PKCS#12）可以作为密钥库或信任库使用，我们可以通过KeyTool查看该密钥库的详细信息。完整的代码清单如下：
	
	＃查看密钥库信息命令
	
	keytool -list -keystore certs/ca.p12 -storetype pkcs12 -v -storepass www.aios.com
	
🚩keytool命令JDK环境下才能使用，这里的参数-storetype值为“pkcs12”

##构建服务器证书

服务器证书的构建与根证书构建相似

###构建 服务器 私钥
	
构建服务器私钥，完整代码清单如下：

	＃构建服务器私钥命令
	
	openssl genrsa -aes256 -out private/server.key.pem 2048
	
各参数的含义如下：

	genrsa		产生RSA密钥命令
	-aes256		使用AES算法（256位密钥）对产生的私钥加密。可选算法包括DES、DESede、IDEA和AES
	-out		输出路径，这里指private/server.key.pem
	
这里的参数2048，指RSA密钥长度位数，默认长度为512位
上述命令执行结果，如下所示：

	＃产生服务器证书密钥
	
	Loading 'screen' into random state - done
	Generating RSA private key, 2048 bit long modulus
	.......................................+++
	....................................................+++
	e is 65537 (0x10001)
	Enter pass phrase for private/server.key.pem:
	Verifying - Enter pass phrase for private/server.key.pem:
	
这时我们需要输入服务器证书密码"`www.thtf.com`"

###生成 服务器证书 请求

完成服务器证书的密钥构建后，我们需要产生服务器证书的签发申请。完整代码清单如下：

	#生成服务器证书签发申请命令
	
	openssl req -new -key private/server.key.pem -out private/server.csr
	
各参数的含义如下：

	req			生成证书签发申请命令
	-new		新请求
	-key		密钥，这里为private/ca.key.pem
	-out		输出路径，这里为private/ca.csr文件

上述命令执行结果，如下

	＃生成服务器证书签发申请
	
	Enter pass phrase for private/server.key.pem:
	Loading 'screen' into random state - done
	You are about to be asked to enter information that will be incorporated into your certificate request.
	What you are about to enter is what is called a Distinguished Name or a DN.
	There are quite a few fields but you can leave some blank
	For some fields there will be a default value,
	If you enter '.', the field will be left blank.
	-----
	Country Name (2 letter code) [AU]:CN
	State or Province Name (full name) [Some-State]:BJ
	Locality Name (eg, city) []:BJ
	Organization Name (eg, company) [Internet Widgits Pty Ltd]:Tech.Settion
	Organizational Unit Name (eg, section) []:AIOS Inc.
	Common Name (e.g. server FQDN or YOUR name) []:Insight Server
	Email Address []:
	
	Please enter the following 'extra' attributes
	to be sent with your certificate request
	A challenge password []:
	An optional company name []:
	
这时我们需要输入服务器证书密码（“`www.thtf.com`"）并填写服务器证书相关信息字段

###签发 服务器证书

我们已经获得了根证书，可以使用根证书签发服务器证书。完整代码清单如下：

	#签发服务器证书命令
	
	openssl x509 -req -days 3650 -sha1 -extensions v3_req -CA certs/ca.cer -CAkey private/ca.key.pem -CAserial ca.srl -CAcreateserial -in private/server.csr -out certs/server.cer

各参数得含义如下：

	x509				签发X.509格式证书命令
	-req				证书输入请求
	-days				有效天数，这里为3650天
	-sha1				证书摘要算法，这里为SHA1算法
	-extensions			按OpenSSL配置文件v3_req项添加扩展
	-CA					CA证书，这里为certs/ca.cer
	-CAkey				CA证书密钥，这里为private/ca.key.pem
	-CAserial			CA证书序列号文件，这里为ca.srl
	-CAcreateserial		创建CA证书序列号
	-in					输入文件，这里为private/server.csr
	-out				输出文件，这里为certs/server.cer
	
上述命令执行结果，如下：

	＃签发服务器证书
	
	Loading 'screen' into random state - done
	Signature ok
	subject=/C=CN/ST=BJ/L=BJ/O=Tech.Settion/OU=AIOS Inc./CN=Insight Server
	Getting CA Private Key
	Enter pass phrase for private/ca.key.pem:
	
这时我们需要输入服务器证书密码"`www.thtf.com`"

###对 服务器证书 进行转换

这里我们同样要将OpenSSL产生得数字证书转化为PKCS#12编码格式。完整命令代码如下：

	#服务器证书转换命令
	
	openssl pkcs12 -export -clcerts -inkey private/server.key.pem -in certs/server.cer -out certs/server.p12
	
各参数得
	
	pkcs12		PKCS#12编码格式证书命令	
	-export		导出证书
	-clcerts	仅导出客户证书
	-inkey		输入密钥路径
	-in			输入文件路径
	-out		输出文件路径，这里为certs/server.p12
	
上述命令执行结果如下

	＃服务器转换证书
	
	Loading 'screen' into random state - done
	Enter pass phrase for private/server.key.pem:
	Enter Export Password:
	Verifying - Enter Export Password:

我们需要输入服务器证书密码"`www.thtf.com`"

##构建客户证书

客户证书的构建与服务器证书得构建基本一致

###构建 客户 私钥

构建客户私钥，完整代码如下：

	＃产生客户私钥命令
	
	openssl genrsa -aes256 -out private/800010000010007.key.pem 2048
	
各参数得含义如下：

	genrsa		产生RSA密钥命令
	-aes256		使用AES算法（256位密钥）对产生得私钥加密。可选算法包括DES、DESede、IDEA和AES。
	-out		输出路径，这里指private/800010000010007.key.pem 2048
	
这里得参数2048，指RSA密钥长度位数，默认长度为512位  
上述命令执行结果如下：
	
	＃产生客户密钥
	
	Loading 'screen' into random state - done
	Generating RSA private key, 2048 bit long modulus
	..........................+++
	....................+++
	e is 65537 (0x10001)
	Enter pass phrase for private/800010000010007.key.pem:
	Verifying - Enter pass phrase for private/800010000010007.key.pem:

这时我们需要输入客户证书得密码“`www.test.com`”

###生成 客户证书 请求

完成客户证书密钥构建后，我们需要产生客户证书签发申请。完整代码如下：

	＃生成客户证书签发申请命令
	
	openssl	req -new -key private/800010000010007.key.pem -out private/800010000010007.csr
	
各参数得含义如下：

	req			产生证书签发申请命令
	-new		新请求
	-key		密钥，这里为private/800010000010007.key.pem
	-out		输出路径，这里为private/800010000010007.csr
	
上述命令执行结果如下：
	
	＃生成客户证书签发申请
	
	Enter pass phrase for private/800010000010007.key.pem:
	Loading 'screen' into random state - done
	You are about to be asked to enter information that will be incorporated into your certificate request.
	What you are about to enter is what is called a Distinguished Name or a DN.
	There are quite a few fields but you can leave some blank
	For some fields there will be a default value,
	If you enter '.', the field will be left blank.
	-----
	Country Name (2 letter code) [AU]:CN
	State or Province Name (full name) [Some-State]:BJ
	Locality Name (eg, city) []:BJ
	Organization Name (eg, company) [Internet Widgits Pty Ltd]:Tech.Settion
	Organizational Unit Name (eg, section) []:Test Merchant
	Common Name (e.g. server FQDN or YOUR name) []:800010000010007
	Email Address []:

	Please enter the following 'extra' attributes
	to be sent with your certificate request
	A challenge password []:
	An optional company name []:

这时我们需要输入客户证书得密码“`www.test.com`”和客户相关信息

###签发 客户证书

我们已经获得了根证书，可以使用根证书签发客户证书（`800010000010007.cer`）。完整代码清单如下：
	
	＃签发客户证书命令
	
	openssl ca -days 3650 -in private/800010000010007.csr -out certs/800010000010007.cer -cert certs/ca.cer -keyfile private/ca.key.pem
	
各参数得含义如下：

	ca			签发证书命令
	-days		证书有效期，这里为3650天
	-in			输入文件，这里为private/800010000010007.csr
	-out		输出文件，这里为certs/800010000010007.cer
	-cert		证书文件，这里为certs/ca.cer
	-keyfile	根证书密钥文件，这里为private/ca.key.pem
	
上述命令执行结果，如下：

	＃签发客户证书
	
	Using configuration from C:\OpenSSL-Win64\bin\openssl.cfg
	Loading 'screen' into random state - done
	Enter pass phrase for private/ca.key.pem:
	Check that the request matches the signature
	Signature ok
	Certificate Details:
	        Serial Number: 1 (0x1)
	        Validity
	            Not Before: Sep 17 08:17:33 2014 GMT
	            Not After : Sep 14 08:17:33 2024 GMT
	        Subject:
	            countryName               = CN
	            stateOrProvinceName       = BJ
	            organizationName          = Tech.Settion
	            organizationalUnitName    = Test Merchant
	            commonName                = 800010000010007
	        X509v3 extensions:
	            X509v3 Basic Constraints:
	                CA:FALSE
	            Netscape Comment:
	                OpenSSL Generated Certificate
	            X509v3 Subject Key Identifier:
	                24:8B:78:66:97:35:0C:B2:A7:F3:EE:21:C2:94:AA:84:32:EC:3D:B8
	            X509v3 Authority Key Identifier:
	                DirName:/C=CN/ST=BJ/L=BJ/O=Tech.Settion/OU=AIOS Inc./CN=Insight RootCA
	                serial:FC:34:82:77:F4:3F:54:26

	Certificate is to be certified until Sep 14 08:17:33 2024 GMT (3650 days)
	Sign the certificate? [y/n]:y


	1 out of 1 certificate requests certified, commit? [y/n]y
	Write out database with 1 new entries
	Data Base Updated
	
这时我们需要输入客户证书密码“`www.test.com`”,并同意签发证书

###对 客户证书 进行转换

最后我们需要将获得的客户证书转化Java语言可以识别得PKCS#12编码格式。完整命令如下
	
	＃客户证书转换命令
	
	openssl pkcs12 -export -clcerts -inkey private/800010000010007.key.pem -in certs/800010000010007.cer -out certs/800010000010007.p12
	
各参数得含义如下

	pkcs12			PKCS#12编码格式证书命令
	-export			导出证书
	-clcerts		仅导出客户证书
	-inkey			输入密钥，这里为private/800010000010007.key.pem
	-in				输入文件，这里为certs/800010000010007.cer
	-out			输出文件，这里为certs/800010000010007.p12
	
上述命令执行结果如下：

	＃客户证书转换
	
	Loading 'screen' into random state - done
	Enter pass phrase for private/800010000010007.key.pem:
	Enter Export Password:
	Verifying - Enter Export Password:

这时我们需要输入客户证书密码“`www.test.com`”  
至此，我们完成了双向认证得所需全部证书。
</article>

<link href="http://wp.simman.cc/Public/file_dldq_org/github-bf51422f4bb36427d391e4b75a1daa083c2d840e.css" media="all" rel="stylesheet" type="text/css"/>
<link href="http://wp.simman.cc/Public/file_dldq_org/github2-d731afd4f624c99a4b19ad69f3083cd6d02b81d5.css" media="all" rel="stylesheet" type="text/css"/>
<link href="http://wp.simman.cc/Public/file_dldq_org/zTreeStyle.css" media="all" rel="stylesheet" type="text/css"/>