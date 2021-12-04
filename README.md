# C2concealer

C2concealer is a command line tool that generates randomized C2 malleable profiles for use in Cobalt Strike. 

## Installation 

```bash
chmod u+x install.sh
./install.sh
```

## Building Docker image

```docker build -t C2concealer .```

## Running with Docker

```docker container run -it -v <cobalt_strike_location>:/usr/share/cobaltstrike/ C2concealer --hostname google.com --variant 3```

## Example Usage

```bash
Usage:
	$ C2concealer --hostname google.com --variant 3

Flags:
	
	(optional)
	--hostname 
		The hostname used in HTTP client and server side settings. Default is None.
	--variant 
		An integer defining the number of HTTP client/server variants to generate. 
		Recommend between 1 and 5. Max 10.
```

## Example Console Output

```bash
root@kali:~# C2concealer --variant 1 --hostname google.com
[i] Searching for the c2lint tool on your system (part of Cobalt Strike). Might take 10-20 seconds.
[i] Found c2lint in the /opt/cobaltstrike/c2lint directory.

Choose an SSL option:
1. Self-signed SSL cert (just input a few details)
2. LetsEncrypt SSL cert (requies a temporary A record for the relevant domain to be pointed to this machine)
3. Existing keystore
4. No SSL

[?] Option [1/2/3/4]:
```

> Tip: Always use an SSL certificate. Preferably a cert from LetsEncrypt or similar.


> Tip: HTTP Variants allow you to select different IOCs for http traffic on different beacons. Recommend a value of at least 1. 

## How it works

We poured over the Cobalt Strike documentation and defined ranges of values that would make sense for each profile attribute. Sometimes that data is as simple as a random integer within some range and other times we need to pick a random value from a python dictionary. Either way, we started tool creation with defining the data that would make a valid profile. 

Then we divided each malleable profile section (or block) into a separate .py file, which contains the logic to draw random appropriate values for each attribute and then output a formatted string for that profile block. We concatenate all profile blocks together, run a few quick consistency checks and then run the profile through the Cobalt Strike linter (c2lint). The output is a profile that *should* work for your engagements. We always recommend testing the profile (including process injection and spawning) prior to running a campaign.

If you're looking into the code, we recommend starting with these two files: /C2concealer/__main__.py and /C2concealer/profile.py. After reviewing the comments, check out individuals profile block generators in the folder: /C2concealer/components.

## Customizing the tool

This is crucial. This is an open sourced version of a tool we've been using privately for about a year. Our private repo has several additional IOCs and a completely different data set. While running the tool provides an excellent start for building a Cobalt Strike malleable profile, we recommend digging into the following areas to customize the data that is randomly populating the tool:

/C2concealer/data/
- dns.py (customize the dns subdomains)
- file_type_prepend.py (customize how http-get-server repsonses look ... aka c2 control instructions)
- params.py (two dictionaries containing common parameter names and a generic wordlist)
- post_ex.py (spawn_to process list...definitely change this one)
- reg_headers.py (typical http headers like user-agent and server)
- smb.py (smb pipenames for use when comms go over smb)
- stage.py (data for changing IOCs related to the stager)
- transform.py (payload data transformations...no need to change this)
- urls.py (filetypes and url path components used for building URIs all across the tool...definitely change this)

In addition, you can customize various attributes all throughout the profile generation process. As an example, in the file: "/C2concealer/components/stageblock.py", you can change the range from which PE image size value is drawn from (near lines 73-74). Please look through all the different files in the components directory. 

If you've made it this far, then we know you'll get a lot of use out of this tool. The way we recommend viewing this tool is that we've built the skeleton code to automatically generate these profiles, now it's up to you to think through what values make sense for each attribute for your campaigns and update the data sources.

## Shoutouts

Big shoutout to Raphael Mudge for constantly improving on the malleable profile feature set and the documentation to learn about it. Also, huge thanks to @killswitch-GUI for his script that automates LetsEncrypt cert generation for CS team servers. Finally, two blog posts that made life so much easier: @bluescreenofjeff's post (https://bluescreenofjeff.com/2017-01-24-how-to-write-malleable-c2-profiles-for-cobalt-strike/) and Joe Vest's post (https://posts.specterops.io/a-deep-dive-into-cobalt-strike-malleable-c2-6660e33b0e0b).

## Version Changelog

Version 1.0
- Public version of FortyNorth Security's internal tool.
- Added support for CS 4.0 (specifically multiple HTTP variants)
- Updated README.md



# 使用案例

使用方法：

~~~
安装命令：
chmod u+x install.sh
./install.sh
使用命令:
C2concealer --variant 1 --hostname aaa.test.com
最后需要更改cs的配置文件中https-certificate的配置
~~~

centos环境默认不能直接使用，可以用kali，当然centos也是可以通过其他方法

~~~
// 看下install.sh内容，发现安装了两个程序
[root@racknerd-22446c C2concealer]# cat install.sh
if [ "$(id -u)" != "0" ]; then
  echo '[Error]: You must run this setup script with root privileges.'
  echo
  exit 1
fi
apt-get -y install python3-pip
apt-get -y install default-jre
pip3 install -e .

// 可以用yum替代,因为java环境是通过yum 安装的,所以这里不需要安装jre了
[root@racknerd-22446c C2concealer]# yum install python3-pip
......
[root@racknerd-22446c C2concealer]# pip3 install -e .
WARNING: Running pip install with root privileges is generally not a good idea. Try `pip3 install --user` instead.
Obtaining file:///root/tools/C2concealer
Installing collected packages: C2concealer
  Running setup.py develop for C2concealer
Successfully installed C2concealer
~~~
我这里使用的是kali

第一步：先将服务器生成 CDN_https.store 下载复制到kali内
第二步：kali用户切换的到root
第三步：执行代码, 将aaa.test.com替换成CDN上线域名

```
C2concealer --variant 1 --hostname aaa.test.com
```

![](\README\image-20211203165632498.png)

- 因为我们使用的CDN给的证书，所以选择3
- 在(absolute path)中输入CDN_https.store绝对路径
- 当出现 **Enter the keystore's password** 后,输入store的密码`CDN_HTTPS123`并回车确认
(注: 密码不会显示，所以输错也不会知道)
- 成功后会生成一个随机名的profile，将这个profile，上传到cobaltstrike目录下。
- 最后进行验证这个profile能否使用

![](\README\image-20211203173013151.png)

首次验证报了两行错误
```
第13行是：dns_beacon，还没有用到DNS上线直接删除
第149行是编译时间：compile_time，不请求有啥用直接删除
```
再次测试发现检查结果中没有**will use built-in SSL cert**，但在后续的测试过程中确定能够正常上线，说明这个配置文件是可以用的
![](\README\image-20211203172142568.png)
