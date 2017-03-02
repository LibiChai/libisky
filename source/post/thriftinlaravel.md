```toml
title = "Thrift在php框架laravel中的应用"
slug = "thrfit_in_laravel"
desc = "Thrift在lrarvel中的应用"
date = "2016-07-28 11:52:33"
update_date = "2016-07-28 11:52:33"
author = "libi"
thumb = ""
tags = ["thrift","laravel"]
```
##前言
最近在项目中遇到需要跨系统调用的需求，找了很多rpc框架对比,因为其他部分项目使用了golang语言，为了考虑日后系统间调用更加灵活，所以决定使用thrift框架。

##Thrif简介

Thrift是一个支持跨语言(支持但不限于php,golang,java,c++,Ruby,Node.js等)支持远程调用rpc的软件框架.

Thrift定义了一个简单的数据类型和服务接口标准，在使用时只需要实现定义好需要用的数据类型与服务（近似理解成调用函数）到.thrfit文件,使用thrfit指令生成对应版本的语言即刻,在下面会说如何编写.thrfit文件

##Thrift在osx下的安装
 需要注意的是，thrift并不需要安装到服务器，只需要安装到你自己的开发机上，使用指令生成对应对应开发语言的代码就可以了。
 我使用的系统是osx10.10，用的最简单的homebrew安装方式，只需要一行指令即刻

	brew install thrift
在等待片刻后,键入thrift命令看到thrift帮助指令即安装成功.

##.thrift文件定义
接下来我们需要根据项目需求来定义thrift文件,thrift支持的数据类型如下：

* 基本类型：
	* bool：布尔值，true 或 false，对应 Java 的 boolean
	* byte：8 位有符号整数，对应 Java 的 byte
	* i16：16 位有符号整数，对应 Java 的 short
	* i32：32 位有符号整数，对应 Java 的 int
	* i64：64 位有符号整数，对应 Java 的 long
	* double：64 位浮点数，对应 Java 的 double
	* string：utf-8编码的字符串，对应 Java 的 String
* 结构体类型：
	* struct：定义公共的对象，类似于 C 语言中的结构体定义，在 Java 中是一个 JavaBean
* 容器类型：
	* list：对应 Java 的 ArrayList
	* set：对应 Java 的 HashSet
	* map：对应 Java 的 HashMap

* 异常类型：
	* exception：对应 Java 的 Exception
* 服务类型：
	* service：对应服务的类

在知道大概的基础数据类型以后，我们来举个例子，假如有2个系统，分别提供用户信息服务和订单服务，订单服务在接受到订单请求时需要获取用户信息。

根据需求，我们可以定义一个服务类叫做 user ，它提供一个getinfo函数供调用，该函数需要传入一个整型的uid，返回一个map（定义struct更为准确，这里为了演示方便使用map）,写成user.thrift文件如下所示
	namespace php user //命名空间

	service user{
        	map<i32,string> getinfo(1:i32 uid)
	}

##代码生成
定义好服务（类）以后，使用thrift -r -gen 命令接口生成各个语言对应的代码版本

	thrift -r -gen php:server user.thrift 


命令执行成功以后会在改目录下生成一个gen-php的目录，直接在具体项目中引入即刻。

##laravel框架调用
thrift官网提供了各个语言版本的类库供调用,项目地址：https://github.com/apache/thrift/

php版本使用composer依赖库进行管理,laravel集成了composer包管理，所以只需要执行
	composer require apache/thrift
即可引入成功，在其他php项目中引入请参见compser说明

引入类库以后需要继续引入上一步生成的协议代码,这里我将gen-php文件夹里的user文件夹复制到laravel的跟目录的thrift目录下，并且在composer.json中添加
	"autoload"{
		"classmap":[
			"thrift"
		]
	}
执行下面代码即刻引入成功
	composer dump-auto

###laravel服务端添加代码
在laravel控制器中添加以下代码，并绑定路由

	use Thrift\Transport\TBufferedTransport;
	use Thrift\Protocol\TBinaryProtocol;
	use Thrift\Transport\THttpClient;
	use Thrift\Transport\TPhpStream;
	header('Content-Type', 'application/x-thrift');
        $handler = new user();//这里是定义在服务端提供服务的类
        $processor = new \thrift\opasProcessor($handler);


        $transport = new TBufferedTransport(new TPhpStream(TPhpStream::MODE_R | TPhpStream::MODE_W));
        $protocol = new TBinaryProtocol($transport, true, true);

        $transport->open();
        $processor->process($protocol, $protocol);
        $transport->close();

添加类服务定义,绑定\user\userIf接口
	
	class user implements \user\userIf{
		function getinfo($uid){
			$user = User::find($uid);
			return $user;
		}
	}

###laravel客户端添加代码
这里使用http协议调用远程方法，代码如下
	try {

            $socket = new THttpClient('www.xxx.com', 80, '/thrift/server');//服务端定义的url

            $transport = new TBufferedTransport($socket, 1024, 1024);
            $protocol = new TBinaryProtocol($transport);
            $client = new thrift\opasClient($protocol);

            $transport->open();


            $result = $client->getinfo(1);//调用远程方法
            $transport->close();
            return $result;

        } catch (TException $tx) {
            print 'TException: '.$tx->getMessage()."\n";
            return null;
        }

##其他
在上面的getinfo方法返回的是一个对象,直接复制代码运行会接收不到返回值，可以修改为返回一个array("key"=>"value")数组即可。

本文内容纯属个人理解，如有错误纰漏欢迎大家指正。

