-----
=====
/*
1.采用新浪SAE的Fetchurl 、KVDB、cron服务，速度快，准确。
中央气象台提供一个API可以查询最近六天的天气，返回的格式为json，例如http://m.weather.com.cn/data/101200101.html 101200101为该城市武汉的城市代码，所以只要我们知道城市代码，就可以抓取该城市天气了。笔者准备了经过了筛选的全国所有城市的数据，这些数据经过本人 一一检验，保证真实有效，mysql数据库文件下载链接http://www.kuaipan.cn/file/id_33799150646592444.html ，excel文件下载链接http://www.kuaipan.cn/file/id_33799150646592446.html
2.初开始我想在每天凌晨自动把城市数据装进数据库，然后在早上九点开始抓取，每一分钟运行一次脚本，每个脚本抓取20条数据，然后每抓取成功一次数据，从数据库删除这个城市记录，后来测试一下，数据库读写太频繁，豆豆消耗太多，效率不太高。
3.经过分析，决定采用KVDB，读写速度极快，装入两千六百条数据只用了5秒左右。
代码如下：
装入城市ID的代码，命名为KV.php：
*/
 
<?php
$mysql = new SaeMysql();
$sql = "SELECT * FROM `city` ";
$data = $mysql->getData( $sql ); //从数据库取出所有城市信息
$num=count($data);
$kv = new SaeKV();
$ret = $kv->init(); //初始化KVDB服务
for($i=0;$i<$num;$i++)
{
$cityid=$data[$i]['cityid'];
$cityname=$data[$i]['cityname'];
$ret = $kv->set($cityid, $cityname);
}
?>
/*
抓取数据代码，命名为curl.php
*/
$kv = new SaeKV();
$mysql = new SaeMysql();
$mysql->setCharset('UTF8');
$ret = $kv->init();
$result=$kv->pkrget('',20);//从KVDB取出20条数据
$f = new SaeFetchurl();
foreach($result as $key=>$value)
{
$url="http://m.weather.com.cn/data/".$key.".html";
if($content = $f->fetch($url))
{
$data=json_decode($content,true);
$data=$data['weatherinfo'];
$cityid=$key;
$tl=array();
$th=array();
$wind=array();
$fl=array();
$fx=array();
$weather=array();
for($i=1;$i<7;$i++)
{
$tmp=0;
$temparray=explode("℃",$data['temp'.$i]);
$th[$i]=$temparray[0];
$tltemp=explode("~",$temparray[1]);
$tl[$i]=$tltemp[1];
if($tl[$i]>$th[$i])
{
$tmp=$th[$i];
$th[$i]=$tl[$i];
$tl[$i]=$tmp;
}
if(isset($data['wind'.$i]))
{
$wind[$i]=$data['wind'.$i];}
else{
$wind[$i]="无数据";
}
if(isset($data['fl'.$i]))
{
$fl[$i]=$data['fl'.$i];}
else{
$fl[$i]="无数据";
}
if(isset($data['fx'.$i]))
{
$fx[$i]=$data['fx'.$i];}
else{
$fx[$i]="无数据";
}
if(isset($data['weather'.$i])){
$weather[$i]=$data['weather'.$i];}
else{
$weather[$i]="无数据";
}
}
$cityname=$data['city'];
$time=time();
$time=date("Y-m-d" ,$time);
$sql="insert into weather(cityid,cityname,time,tl1,tl2,tl3,tl4,tl5,tl6,th1,th2,th3,th4,th5,th6,wind1,wind2,wind3,wind4,wind5,wind6,fl1,fl2,fl3,fl4,fl5,fl6,fx1,fx2,fx3,fx4,fx5,fx6,weather1,weather2,weather3,weather4,weather5,weather6) values( '" . $mysql->escape( $cityid ) . "' ,'" . $mysql->escape( $cityname ) . "' ,'" . $mysql->escape( $time) . "' ,
'" . $mysql->escape( $tl[1] ) . "' ,'" . $mysql->escape( $tl[2]) . "' ,'" . $mysql->escape( $tl[3]) . "' ,'" . $mysql->escape( $tl[4] ) . "' ,'" . $mysql->escape( $tl[5] ) . "' ,'" . $mysql->escape( $tl[6]) . "' ,
'" . $mysql->escape( $th[1] ) . "' ,'" . $mysql->escape( $th[2] ) . "' ,'" . $mysql->escape( $th[3] ) . "' ,'" . $mysql->escape( $th[4] ) . "' ,'" . $mysql->escape( $th[5] ) . "' ,'" . $mysql->escape( $th[6] ) . "' ,
'" . $mysql->escape( $wind[1] ) . "' ,'" . $mysql->escape( $wind[2] ) . "' ,'" . $mysql->escape( $wind[3] ) . "' ,'" . $mysql->escape( $wind[4] ) . "' ,'" . $mysql->escape( $wind[5] ) . "' ,'" . $mysql->escape( $wind[6] ) . "' ,
'" . $mysql->escape( $fl[1] ) . "' ,'" . $mysql->escape( $fl[2] ) . "' ,'" . $mysql->escape( $fl[3] ) . "' ,'" . $mysql->escape( $fl[4] ) . "' ,'" . $mysql->escape( $fl[5] ) . "' ,'" . $mysql->escape( $fl[6] ) . "' ,
'" . $mysql->escape( $fx[1] ) . "' ,'" . $mysql->escape( $fx[2] ) . "' ,'" . $mysql->escape( $fx[3] ) . "' ,'" . $mysql->escape( $fx[4] ) . "' ,'" . $mysql->escape( $fx[5] ) . "' ,'" . $mysql->escape( $fx[6] ) . "' ,
'" . $mysql->escape( $weather[1] ) . "' ,'" . $mysql->escape( $weather[2] ) . "' ,'" . $mysql->escape( $weather[3] ) . "' ,'" . $mysql->escape( $weather[4] ) . "' ,'" . $mysql->escape( $weather[5] ) . "' ,'" . $mysql->escape( $weather[6] ) . "'
)";
$mysql->runSql($sql);
if( $mysql->errno() != 0 )
{
die( "Error:" . $mysql->errmsg() );
}else{
$kv->delete($key);//读取成功从KVDB删除这条记录,否则保留，下次继续抓取
}
}
}
/*
然后编写定时执行命令就行
*/
 
cron:
- description: weather
url: curl.php
schedule: every 1 min
- description: kv
url: kv.php
schedule: every day of month 10:00
//本文链接：http://lvxinwei.sinaapp.com/832.html
