BinSpec是什么
=============
BinSpec是一个基于WTFPL协议(Do What The Fuck You Want To Public License)的开源项目，这个项目的目标是构建一门简单灵活的**二进制数据结构描述语言**。  

BinSpec的典型应用场景有：网络应用协议描述、二进制序列化格式描述。  

BinSpec是一门描述性语言，它只能描述数据结构，并且它的语法被设计成尽量与实现无关，所以它只是一套语言规范，它“正则化”了二进制数据结构的描述方式。  

正则化让描述文档变得可以解析，我们利用这一特性编写了一个文档解析工具(bs)，它可以通过解析BinSpec语法知道文档描述了些什么内容，但是它不能理解这些内容，bs的唯一作用只有把基于BinSpec编写的文档转换成模板语言的代码（目前只支持PHP）。  

bs转换输出的代码中包含了一组bs\_开头的类和函数，其中有一个bs\_get\_doc函数它的返回值就是按文档内容生成的树状结构的对象。bs通过“混合”机制让模板可以调用到bs\_get\_doc函数，从而可以获得文档中的描述信息。  

利用文档中的描述信息模板就可以自由发挥做任何事情，最典型的应用就是生成客户端和服务端通讯层的封包解包代码。  


BinSpec起步
===========
首先你需要准备一个有gcc和make的环境，然后下载BinSpec的项目代码并在本地解压，然后进入BinSpec项目跟目录运行make，完成后你就可以在bin目录找到BinSpec的命令行程序bs。  

你可以试着运行它，像这样：  

    ./bin/bs

你会看到屏幕上输出一些帮助信息，接着你可以试着用它转换项目自带的test.bs文件，像这样：

    ./bin/bs -c test.bs

你会看到屏幕输出一段PHP代码，这段代码就是test.bs转换后的结果。再接着你可以试着用它混合项目自带的test.bs和test.php，像这样：

    ./bin/bs -c test.bs test.php

你会发现屏幕上的输出和前一步几乎一样，只是最后多了一行代码，打开test.php你会发现多出来的那行代码就是test.php的内容。是的，混合是一个纯粹的拼接过程。

接着你可以试着把混合后的结果传递给PHP命令行程序运行，像这样(这里假设你的系统安装了PHP命令行，并且PHP命令所在目录存在于PATH环境变量中)：

    ./bin/bs -c test.bs test.php | php

现在你可以看到屏幕上打印了一个有复杂树状结构的PHP变量，这就是temp.php执行的结果，试着理解混合后的PHP代码可以帮助理解如何制作模板，你可以用下面的方式保存混合后的PHP代码：

    ./bin/bs -c test.bs test.php > output.php

混合后的代码会被保存到output.php中。


BinSpec语法
===========
BinSpec代码以文档为单位，一个.bs文件被解析成一个文档，一个文档中包含多个"包"，BinSpec的包用**pkg**关键字声明，例1演示了如何声明一个包。  

**例1**

    pkg player
    {
    }

包本身没身没有数据结构描述作用，包的主要作用是区隔，就好像系统中的文件夹。BinSepc的包可以嵌套，这一点也跟文件夹一样，例2演示了如何声明嵌套的包。 

**例2**

    pkg game
    {
      pkg player
      {
      }

      pkg admin
      {
      }
    }

在BinSpec中对一个数据结构的描述称为“定义”，定义只能存在于包中，就像文件之于文件夹，例3演示了如何定义一个数据结构。  

**例3**

    pkg player
    {
      def login
      {
      }
    }

包和定义都有一个可选的结构称为“编号”，编号通常是为了区别数据包的类型，你可以根据需要来决定是否使用编号，例4演示了如何声明一个有编号的包，里面有一个有编号的定义。  

**例4**

    pkg player = 1
    {
      def login = 1
      {
      }
    }

编号的一个典型应用场景是：为每个数据包都预留两个字节的头部信息，一个字节存储包编号，一个字节存储定义编号，通过这两个字节可以唯一确定数据包的类型，以便调用对应的解包函数对数据包进行解包。  

一个定义中可以包含多个数据项，这些数据项称为“字段”，每个字段都有各自的名称和类型，名称和类型之间用冒号分隔，字段和字段之间用逗号分隔，例5演示了如何定一个有两个字符串字段的登录接口。  

**例5**

    pkg player = 1
    {
      def login = 1
      {
        username : string,
        password : string
      }
    }

定义也可以不包含任何字段，没有字段的定义称为“空定义”。  

BinSpec支持一些基本的数据类型，包括：有符号整型、无符号整型、字符串、列表、枚举。  

有符号整型按占用字节数不同可细分为：  

1. int8, byte   -   
2. int16, short -   
3. int32, int   -  
4. int64, long  -  

无符号整型按占用字节数不同可细分为：  

1. uint8, ubyte   -   
2. uint16, ushort -   
3. uint32, uint   -  
4. uint64, ulong  -  

BinSpec中的字符串以固定长度的头部开始，头信息存放的是字符串的**字节长度**，读取字符串时先读取长度信息，接着顺序读取指定长度的字符串内容，空字符串长度为0。  

字符串按头部占字节数可细分为：

1. string8, string -   
2. string16        -  
3. string32        -  
4. string64        -  

BinSpec中的列表和字符串类似以固定长度的头部开始，不同之处在于列表的头部存放的时列表的**元素个数**，读取列表时先读取个数信息N，接着按元素的结构定义解析N次，空列表长度为0。  

1. list8, list -   
2. list16      -  
3. list32      -  
4. list64      -  

列表类型的字段后面必须紧跟列表元素的结构定义，例6演示了如何定义一个列表。 

**例6**

    pkg friend = 3
    {
      def get_friend_list_result = 1
      {
        id : int,
        friends : list {
          id : int,
          name : string
        }
      }
    }

BinSpec中的枚举使用enum关键字声明，枚举类型跟定义一样必须在包当中声明，在字段中指定枚举类型时使用enum<T>语法，例7演示了如何声明一个枚举类型。  

**例7**

    pkg item = 4
    {
      enum colors
      {
        RED = 1, GREEN = 2, BULE = 3
      }

      def get_item_by_color = 1
      {
        color : enum<colors>
      }
    }

枚举有一个可选的语法结构叫做继承，枚举根据继承的类型不同占用的字节数也不同，枚举默认是继承byte类型，枚举只能继承整型类型，例8演示了枚举如何显式继承。  

**例8**

    pkg item = 4
    {
      enum colors : int
      {
        RED = 1, GREEN = 2, BULE = 3
      }
    
      def get_item_by_color = 1
      {
        color : enum<colors>
      }
    }

上面我们介绍了BinSpec的语法和基本数据类型，下面我们介绍BinSpec的高级语法。  

在实际项目中经常会遇到一种情况，例如游戏中有三个接口，一个接口用于获取玩家背包中的物品列表，一个接口用于获取NPC出售的物品列表，一个接口用于获取仓库中的物品列表，这三个接口有细微区别但是又有公共的部分，它们都返回物品列表，在基础物品信息上他们是一致的，但是商店的接口还会包含物品的价格。这种情况下，我们可以在三个接口中重复定义物品信息的字段，但是更有效的做法是使用BinSpec提供的**自定义类型**语法，自定义类型语法允许我们使用基本类型构建复杂类型，并在不同的地方引用。例9演示了如何用自定义类型解决上述问题。

**例9**

    pkg item = 4
    {
      type item_info
      {
        id : int,
        name : string
      }

      def get_player_item_result = 1
      {
        items : list {
          item : type<item_info>
        }
      }

      def get_npc_item_result = 2
      {
        items : list {
          item : type<item_info>,
          price : int
        }
      }

      def get_warehouse_item_result = 3
      {
        items : list {
          item : type<item_info>
        }
      }
    }


BinSpec模版编写
===============

**未完待续**
