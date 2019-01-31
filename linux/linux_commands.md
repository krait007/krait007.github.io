# Linux常用命令



### Linux系统分类

#### 著名的linux系统基本上分两大类

1.RedHat系列：Redhat、Centos、Fedora等 
2.Debian系列：Debian、Ubuntu、Kali等

##### RedHat 系列 

1 常见的安装包格式 rpm包,安装rpm包的命令是“rpm -参数”

2 包管理工具 yum

##### Debian系列 

1 常见的安装包格式 deb包,安装deb包的命令是“dpkg -参数”

2 包管理工具 apt-get

---------------------
##### Suse linux

​    软件包管理系统：YaST (RPM), 第三方APT (RPM) 软件库（repository）



### 查询及帮助命令

man查看命令帮助，命令的词典，更复杂的还有 info，但不常用。 

```bash
man ls
ls --help
```

help查看 Linux 内置命令的帮助。

```bash
help cd
```



### 文件和目录操作命令

ls全拼 list，功能是列出目录的内容及其内容属性信息。

```bash
# a 显示所有文件, .开头的文件； l列表模式  h 符合人类阅读习惯  t按时间先后排序
ls  -alht
#按文件大->小
ls -Sl
#按文件小->大
ls -Slr
```

cd 从当前工作目录切换到指定的工作目录。 

```bash
cd ./dir
cd /fullpath
# 接入用户home目录
cd ~
#回到上次的目录
cd - 
```

cp 复制文件或目录。 

```bash
# -a 保持原来的属性      -r 递归复制    -p:保持属性
cp file_from  file_to
cp -r dir_from dir_to
```

find 用于查找目录及目录下的文件。

```bash
find . -type d -print     #只列出当前目录所有的子目录
find . ! -type d -print   #只列出当前目录的非子目录（文件）
find . -type f -print     #只列出当前目录所有的文件
find . -type l -print     #只列出当前目录的所有符号链接
find . -type c -print     #只列出当前目录的所有字符设备
find . -type b -print     #只列出当前目录的所有块设备
find . -type s -print     #只列出当前目录的所有套接字
find . -type p -print     #只列出当前目录的所有Fifo

#基于路径或文件名：
find . -name ap* -o -name may*    　　　　　　　　 #查找以ap或may开头的文件
find . -name "*.txt" -print 　　　　 　　　　　　　 #在当前目录中中查.txt文件并显示
find . -name "[A-Z]*" -print 　　　 　　　　　　　  #在当前目录中查以大写字母开头的文件并显示
find /etc -name "host*" -print 　　 　　　　　　　　#在/etc目录中查以host开头的文件并显示
find . -name "[a-z][a-z][0–9][0–9].txt" -print 　#查以两个小写字母和两个数字开头的txt文件
find . ！-name "*.txt" -print 　　　 　　　　　　　　#列出所有不以.txt结尾的文件名
find /home/user -path "*ttt*" -print 　　　　 　　 #-path将文件路径和文件名当做一个整体来匹配，
                                                 #所以文件路径或者文件名中含有ttt的都会被匹配到
                                                 
find . -name "*.txt" -print 　　　　 　　　　　　　　#匹配当前目录下所有.txt文件，并输出其路径
find . -iname "example*" -print 　  　　　　　　　　#匹配当前目录下以example开头的文件名，
                                                 #不区分大小写，并输出其路径
find . \( -name "*.txt" -o -name "*.pdf" \) -print 　#匹配当前目录下所有.txt文件和.pdf文件，
                                                     #并输出其路径
find . -type f -name "*.swp" -delete 　　　　　　 　#删除当前目录下所有的.swp文件
find . -type f -name "*.c" -exec cat {} \;>all_c_files.txt  #找到所有C文件并拼接起来写入单个
                         #文件all_c_files.txt。find命令的输出只是一个单数据流，所以不用>>进行追加

find /mnt -name tom.txt -ftype vfat 　　  #在/mnt下查找名称为tom.txt且文件系统类型为vfat的文件
find /mnt -name tom.txt ! -ftype vfat 　　#在/mnt下查找名称为tom.txt且文件系统类型不为vfat的文件

#基于用户文件权限：
find . -perm 755 -print 　　　　　　　　　　　　　#查当前目录下下权限为755的文件
find / -group cat 　　　　　　　　　　　　　　　　 #查找在系统中属于cat组的目录和文件
find / -nouser 　　　　　　　　　　　　　　　　　  #查找在系统中属于作废用户的文件
find / -user fred 　　　　　　　　　　　　　　　　  #查找在系统中属于fred这个用户的文件
find . -type f -user ubuntu -print 　　　　　　　    #列出当前目录下属于用户ubuntu的文件
find . -type f -name "*.php" | -perm 644 -print    #列出当前目录下所有权限为644的php文件
find /home -uid +501 　　　　　　　　　　　　　  #列出/home目录内用户的识别码大于501的文件或目录
find /home -gid 501 　　　　　　　　　　　　　　  #列出/home内组id为501的文件或目录
find /home -nouser 　　　　　　　　　　　　　　   #列出/home内不属于本地用户的文件或目录
find /home -nogroup 　　　　　　　　　　　　　    #列出/home内不属于本地组的文件或目录

#基于目录或文件大小：
find . -name "*" -type f -size 0c | xargs -n 1 rm -f    #批量删除空文件（大小等于0的文件）的方法
find . -type f -empty 　　　　　　　　      #查找大小为0的文件或空目录
find . -type f -size +1000000c -print    #查长度大于1Mb的文件
find . -type f -size 100c -print 　　　　  #查长度为100c的文件
find . -type f -size +10 -print 　　　　   #查长度超过期作废10块的文件（1块=512字节）
find . -type f -size +2k 　　　　　　       #搜索当前目录下大于2KB的文件
find . -type f -size -2k 　　　　　　　　   #搜索当前目录下小于2KB的文件
#除了千字节KB（k）之外，还有块（b 512字节），字节（c），字（w），兆字节（M），吉字节（G）

#基于操作时间：
find / -type f -amin -10 　　　　　　　　 #查找在系统中最后10分钟访问的文件
find / -type f -atime -2 　　　　　　　　 #查找在系统中最后48小时访问的文件
find / -type f -empty 　　　　　　　　　　 #查找在系统中为空的文件或者文件夹
find / -type f -mmin -5 　　　　　　　　  #查找在系统中最后5分钟里修改过的文件
find / -type f -mtime -1 　　　　　　　　 #查找在系统中最后24小时里修改过的文件
find /home -type f -mtime -2 　　　　　　 #在/home下查最近两天内改动过的文件
find /home -type f -atime -1 　　　　　　 #在/home下查1天之内被存取过的文件
find /home -type f -mmin +60 　　　　　   #在/home下查60分钟前改动过的文件
find /home -type f -amin +30 　　　　　　 #在/home下查最近30分钟前被存取过的文件
find . -type f -atime -7 -print 　　　　　#列出当前目录在最近7天内被访问过的所有文件
find ．-type f -atime 7 -print 　　　　　　#列出当前目录恰好在第七天前被访问过的所有文件
find . -type f -atime +7 -print 　　　　　 #列出当前目录访问时间超过七天的所有文件
find . -type f -newer file.txt -print 　　#找出当前目录比file.txt修改时间更长的所有文件，
  
#find与-exec参数：
find . -perm -007 -exec ls -l {} \; 　　　　　　　　　#查所有用户都可读写执行的文件同-perm 777

########## 下面的命令不要轻易尝试######################
find . -type f -user root -exec chown ubuntu {} \;   #将当前目录下所有root的文件改为属于
                          #ubuntu，此处{}会替换成每一个匹配的文件名，{}表示匹配，与-exec结合使用

#找到所有C文件并拼接起来写入单个文件all_c_files.txt。find命令的输出只是一个单数据流，所以不用>>进行追加
find . -type f -name "*.c" -exec cat {} \;>all_c_files.txt 　

#找到10天前访问的.txt文件并复制到/data目录中
find . -type f -atime +10 -name "*.txt" -exec cp {} /data \;  

#找到当前目录下以log02开头的文件并将其移动到/data/game目录下
find ./log02* -exec mv {} /data/game \; 　　　   　
find . -type f -mtime +5 -exec -ok rm {} \ 　  #在当前目录中查找更改时间在5日以前的文件并删除它们

#-exec结合printf输出信息(find 的-exec参数可以接其他任何命令，具体看需求是怎样的)
find . -type f -name "*.txt" -exec printf "Text file:%s\n" {} \;   


#find 与 xargs 组合:(rmdir rm -rf 不要轻易尝试)
find . -type d -empty | xargs rmdir        #删除当前目录下所有空文件夹
#递归查找当前目录及子目录下所有空文件并删除，rm 的 -r参数表示递归，-f表示强制删除
find . -type f -empty | xargs rm -rf　　　 

find . -name "*.txt" | xargs rm -rf                #查找当前目录下所有.txt文件并删除
#匹配并删除所有的.txt文件，xargs -0将\0作为输入定界符
find . -type f -name "*.txt" -print0 | xargs -0 rm -f 　　　　
find /data -type f -name "*.c" -print0 | xargs -0 wc -l 　　#统计/data目录下所有C文件的行数

#通过第一个管道find查找当前目录下所有以test开头的log文件的内容，并将内容输送到第二个管道xargs grep，过滤并查找出所有含有”[AAAA]”的行，然后再送给wc命令进行统计行数
find ./ -name "test*.log" | xargs grep "\[AAAA\]" | wc –l 　 
#同上例，将查找到的行重定向到re.txt文件
find ./ -name "test*.log" | xargs grep "\[AAAA\]" > re.txt 　  
#同上例，当需要匹配多个字段时：例如此时匹配[AAAA]和 ”SVR”:1 两个字段
find ./ -name "test*.log" | xargs grep "\[AAAA\]" | grep "\"SVR\":1" >re.log 　　　　　　　　　　　　　　　　　　
#同上例，多匹配一个[Y-m-d H:i:s]格式的时间字段
#(xargs命令应该紧跟在管道操作符后面，以标准输入作为主要的源数据流。)
find ./ -name "test*.log" | xargs grep "\[AAAA\]" | grep "\"SVR\":1" | grep "\[2016-12-27 10:01:59\]" > re.log   

#其他：
find . -iregex ".*\(\.py\|\.sh\)$" 　　　　　　　　　　　　　　#-iregex忽略正则表达式的大小写 此处为忽略后缀的大小写
find . -maxdepth 2 -type f -print 　　　　　　　　　　　　　 #遍历的最大深度距离此目录最多为2层子目录，列出所有普通文件
find . -mindepth 2 -type f -print 　　　　　　　　　　　　　  #遍历的深度距离当前目录至少两个子目录，列出所有文件
find /data \( -name ".git" -prune \) -o \( -type f -print \)      #在/data目录下搜索所有文件，搜索时跳过.git子目录。 \( -name ".git" -prune \)这里用于排除.git目录
find /etc -name "passwd*" -exec grep "cnscn" {} \ 　　　　  #看是否存在cnscn用户
find . -type f -name april* fprint file  #在当前目录下查找以april开始的文件，并把结果输出到file中
find . -links +2 　　　　　　　　　　　　　　　　　　　　　　  #查硬连接数大于2的文件或目录
```

mkdir创建目录。 

```bash
cd /tmp
mkdir test1
#-p 自动创建不存在的父目录 如test2
mkdir -p test2/test3/test4/test5
#创建多个目录
mkdir -v tmp{1..9}
#创建目录树
mkdir -vp project/{lib/,bin/,doc/{info,prod},logs/{info,prod},service/deploy/{info,prod}}

```

mv 移动或重命名文件

```bash
mv file1 file2
mv file* dir
mv file1 file2 dir
```

pwd全拼 print working directory，其功能是显示当前工作目录的绝对路径。 

```bash

```

rename用于重命名文件

```bash

```


rm其功能是删除一个或多个文件或目录。 

```bash
rm file1
rm -f fire
rm -rf dir
```

rmdir功能是删除空目录

```

```

touch创建新的空文件，改变已有文件的时间戳属性。 

```bash
ls -al filename
touch filename
ls -al filename
```

tree功能是以树形结构显示目录下的内容

```bash
tree
tree -d
```

basename显示文件名或目录名

```bash
basename /usr/bin/ls
basename /tmp/test.txt
```

dirname显示文件或目录路径

```bash
dirname /usr/bin/ls
```

lsattr 查看文件扩展属性。 

```
lsattr -a 
lsattr /usr/bin/ls
```

chattr 改变文件的扩展属性

```bash
#用chattr命令防止系统中某个关键文件被修改：
chattr +i /tmp/test.txt
rm -rf /tmp/test.txt # Operation not permitted

#让文件只能往里面追加数据，但不能删除
chattr +a /tmp/test.txt
lsattr /tmp/test.txt
```

file 显示文件的类型

```bash
file /usr/bin/ls
```

md5sum 计算和校验文件的 MD5 值。

```bash
md5sum /usr/bin/ls
```



### 查看文件及内容处理命令

cat功能是用于连接多个文件并且打印到屏幕输出或重定向到指定文件中

```bash
cat file
```


tac  是 cat 的反向拼写，因此命令的功能为反向显示文件内容

```bash
tac file
```

more分页显示文件内容

```bash
more file
cat file|more
```

less分页显示文件内容，more 命令的相反用法。

```
less file
```

head显示文件内容的头部
tail显示文件内容的尾部

```
head file

tail file
tail -f file
```

cut将文件的每一行按指定分隔符分割并输出。

```bash
echo \
"2104,3,399900
1600,3,329900
2400,3,369000
1416,2,232000
3000,4,539900
1985,4,299900
1534,3,314900
1427,3,198999
1380,3,212000
1494,3,242500
1940,4,239999
2000,3,347000
1890,3,329999
4478,5,699900" > house.txt

#按分隔符分割，获取第二列
cut -d ,  -f 2  house.txt

#按byte分割
cat house.txt|cut -b 1-4

```

split分割文件为不同的小片段

```bash
dd if=/dev/zero bs=100k count=1 of=date.file
split -b 10k date.file
#date.file xaa xab xac xad xae xaf xag xah xai xaj

split -b 10k date.file -d -a 3
#date.file x000 x001 x002 x003 x004 x005 x006 x007 x008 x009

#分割名前加前缀
split -b 10k date.file -d -a 3 split_file

#合并
cat x*>>y*

```

paste按行合并文件内容。 

```bash
echo \
"ID897
ID666
ID982" >pas1
echo \
"P.Jones
S.Round
L.Clip">pas2

#基本paste命令将pas1和pas2两文件粘贴成两列：
paste pas1 pas2

ID897   P.Jones
ID666   S.Round
ID982   L.Clip
#通过交换文件名即可指定哪一列先粘：
paste pas2 pas1

P.Jones ID897
S.Round ID666
L.Clip ID982

#要创建不同于空格或tab键的域分隔符，使用-d选项。下面的例子用冒号做域分隔符。
paste -d: pas2 pas1
```

sort对文件的文本内容排序

```

```

uniq去除重复行

```

```

wc统计文件的行数、单词数或字节数

```
# -c 统计字节数   -l 统计行数  -m 统计字符数，不能和-c同时使用
# -L 打印最长行的长度
# -w 统计字数
wc -l house.txt
wc -c house.txt 
```

iconv转换文件的编码格式

```bash
echo 'utf8中文' >utf8.txt
#utf8编码转成gbk国标
iconv -f utf8 -t gbk -o gbk.txt utf8.txt

#显示支持的编码
iconv -l
```

dos2unix将 DOS 格式文件转换成 UNIX 格式。 

```

```

diff全拼 difference，比较文件的差异，常用于文本文件

```bash
diff pas1  pas2
```

rev反向输出文件内容。

```bash
rev pas1
798DI
666DI
289DI
```


grep/egrep过滤字符串

```bash
grep -i id pas1 #不区分大小写地搜索。默认情况区分大小写，
grep -l ID pas1 #只列出匹配的文件名，
grep -L ID pas1 #列出不匹配的文件名，
grep -w ID pas1 #只匹配整个单词，而不是字符串的一部分（如匹配‘magic’，而不是‘magical’），
grep -C number ID pas1            #匹配的上下文分别显示[number]行，
grep ID pas1 | grep 66  #显示既匹配 pattern1 又匹配 pattern2 的行。
grep -r utf *
grep -E 'ID|66' pas1
egrep 'ID|66' pas1

#ip
grep -oP "([0-9]{1,3}\.){3}[0-9]{1,3}" file.txt
#email
grep -oP "[a-zA-Z0-9_-]+@[a-zA-Z0-9_-]+(\.[a-zA-Z0-9_-]+)+" file.txt
#手机号
grep -E "\<1[3|4|5|8][0-9]{9}\>"  file.txt
```

```
grep的规则表达式:
\     反义字符：如"\"\""表示匹配""
[ - ] 匹配一个范围，[0-9a-zA-Z]匹配所有数字和字母
* 所有字符，长度可为0
+ 前面的字符出现了一次或者多次
^  #匹配行的开始 如：'^grep'匹配所有以grep开头的行。    
$  #匹配行的结束 如：'grep$'匹配所有以grep结尾的行。    
.  #匹配一个非换行符的字符 如：'gr.p'匹配gr后接一个任意字符，然后是p。    
*  #匹配零个或多个先前字符 如：'*grep'匹配所有一个或多个空格后紧跟grep的行。    
.*   #一起用代表任意字符。   
[]   #匹配一个指定范围内的字符，如'[Gg]rep'匹配Grep和grep。    
[^]  #匹配一个不在指定范围内的字符，如：'[^A-FH-Z]rep'匹配不包含A-R和T-Z的一个字母开头，紧跟rep的行。    
\(..\)  #标记匹配字符，如'\(love\)'，love被标记为1。    
\<      #到匹配正则表达式的行开始，如:'\<grep'匹配包含以grep开头的单词的行。    
\>      #到匹配正则表达式的行结束，如'grep\>'匹配包含以grep结尾的单词的行。    
x\{m\}  #重复字符x，m次，如：'0\{5\}'匹配包含5个o的行。    
x\{m,\}  #重复字符x,至少m次，如：'o\{5,\}'匹配至少有5个o的行。    
x\{m,n\}  #重复字符x，至少m次，不多于n次，如：'o\{5,10\}'匹配5--10个o的行。   
\w    #匹配文字和数字字符，也就是[A-Za-z0-9]，如：'G\w*p'匹配以G后跟零个或多个文字或数字字符，然后是p。   
\W    #\w的反置形式，匹配一个或多个非单词字符，如点号句号等。   
\b    #单词锁定符，如: '\bgrep\b'只匹配grep。  
```



join按两个文件的相同字段合并

```

```

tr替换或删除字符。 

```

```

vi/vim命令行文本编辑器

```

```



### 文件压缩及解压缩命令

tar打包压缩

```bash
#打包
tar cf pkg.tar *.txt 
tar czf pkg.tar.gz *.txt   #带压缩

#解包
tar xvf pkg.tar.gz
```

gunzip解压文件

```

```

gzip 压缩工具    zip压缩工具

```

```



### 信息显示命令

uname显示操作系统相关信息的命令

```bash
uname
uname -a
uname -r
uname -v
```

hostname显示或者设置当前系统的主机名

```bash
hostname             #显示主机名
hostname newname     #修改主机名，临时，重启后丢失
vi /etc/hostname     #编辑次文件永久修改文件名， 修改主机名后，关注下/etc/hosts是否有同步修改的项
```

dmesg显示开机信息，用于诊断系统故障

```
dmesg
dmesg|more
```

uptime显示系统运行时间及负载

```bash
uptime 
#14:33:07 up  5:16,  2 users,  load average: 0.00, 0.01, 0.05
```


stat显示文件或文件系统的状态

```bash
stat /dev/sda
stat -f /dev/sda
```


du计算磁盘空间使用情况

```bash
du -h 
du -h -d 1  #max-depth=1
```


df报告文件系统磁盘空间的使用情况

```
df -h
df -hT
```

top实时显示系统资源使用情况

```

```

free查看系统内存

```

```

date显示与设置系统时间

```

```


cal查看日历等时间信息

```

```



### 搜索文件命令

which查找二进制命令，按环境变量 PATH 路径查找

```
which ls
```


find从磁盘遍历查找文件或目录

```bash
#见上面
```

whereis查找二进制命令，按环境变量 PATH 路径查找

```bash
whereis ls
```


locate从数据库 (/var/lib/mlocate/mlocate.db) 查找命令，使用 updatedb 更新库。

```
locate passwd -n 5    #查找和passwd相关的，显示5条
updatedb
```



### 用户管理命令



groupadd添加用户组

```
groupadd testg
```

useradd添加用户。 

```bash
useradd -m tester6 -g testg -G testg,root    #-g 主组 -G组列表 -m创建home目录  -d 指定home目录
```

usermod修改系统已经存在的用户属性

```bash
 usermod -a - G www tester5    # 将tester5添加到www用户组
```

userdel删除用户 groupdel 删除组

```
userdel tester5
groupdel testg
```

passwd修改用户密码

```bash
passwd    #修改当前用户密码
passwd tester5   #修改指定用户密码
echo "newpasswd" |passwd --stdin tester5
```


chage修改用户密码有效期限

```bash
chage -M 5 sherry     #5天后修改密码
```

- 

- id查看用户的 uid,gid 及归属的用户组。 

- su切换用户身份。 

- ```bash
  su - user
  su user
  ```

- visudo编辑 / etc/sudoers 文件的专属命令。 
  sudo以另外一个用户身份（默认 root 用户）执行事先在 sudoers 文件允许的命令。



### 基础网络操作命令

telnet使用 TELNET 协议远程登录

```bash
#本身协议不安全，很少采用telnet远程连接了，现常见用于网络连通性测试
telnet serverip  port
```

ssh使用 SSH 加密协议远程登录

```bash
ssh user@host_address
ssh -p 2222 user@host_address
#公钥登录
ssh-keygen
ssh-copy-id user@host_address

```

scp用于不同主机之间复制文件

```bash
#本地上传远端服务器
scp -r ./localfiles root@remote_server:/home/serverdir/

#远端服务器copy到本地
scp -r  root@remote_server:/home/serverdir/  ./localfiles
```

wget命令行下载文件

```bash
#1、下载单个文件：
wget http://www.baidu.com

#2、将下载的文件存放到指定的文件夹下，同时重命名下载的文件，利用-O：
wget -O /home/index http://www.baidu.com

#3、下载多个文件：首先，创建一个file.txt文件，写入两个url（换行），如http://www.baidu.com;然后，
wget -i file.txt

#4、下载时，不显示详细信息，即在后台下载, 命令执行后会，下载的详细信息不会显示在终端，会在当前目录下生成一个web-log记录下载的详细信息。
wget -b http://www.baidu.com

#5、下载时，不显示详细信息，同时将下载信息保存到执行的文件中（同4）：
wget -o dw.txt http://www.baidu.com

#6、断点续传：
wget -c http://www.baidu.com

#7、限制下载的的速度：
wget --limit-rate=100k -O zfj.html http://www.baidu.com

#8、测试是否能正常访问：
wget --spider http://www.baidu.com

#9、设置下载重试的次数：
wget --tries=3 http://www.baidu.com

#10、下载一个完整的网站，即当前页面所依赖的所有文件：
wget --mirror -p --convert-links -P./test http://localhost
# -p:下载所有用于显示给定网址所必须的文件
# --mirror:打开镜像选项
# --convert-links：下载以后，转换链接用于本地显示
# -P LOCAL_DIR：保存所有的文件或目录到指定的目录下

#11、下载的过程中拒绝下载指定类型的文件:
wget --reject=png --mirror -p --convert-links -P./test http://localhost

#12、多文件下载中拒绝下载超过设置大小的文件：注意：此选项只能在下载多个文件时有用，当你下载一个文件时没用
wget -Q5m -i file.txt

#13、从指定网站中下载所有指定类型的文件：
wget -r -A .png http://www.baidu.com

#14、wget下载时，某些资源必须使用 
wget --no-check-certificate http://www.baidu.com

#15、使用wget实现FTP下载：
wget --file-user=USERNAME --file-password=PASSWORD url
```

ping测试主机之间网络的连通性

```bash
ping localhost
#ping大包
ping -s 65500 localhost
ping -c 3 localhost
```

route显示和设置 linux 系统的路由表

```bash
route add -host 192.168.1.2 dev eth0
route add -host 10.20.30.148 gw 10.20.30.40
route add -net 10.20.30.40 netmask 255.255.255.248 eth0   #添加10.20.30.40的网络
route add -net 10.20.30.48 netmask 255.255.255.248 gw 10.20.30.41 #添加10.20.30.48的网络
route add -net 192.168.1.0/24 eth1

#添加默认路由
route add default gw 192.168.1.1  

#删除
route del -host 192.168.1.2 dev eth0:0
route del -host 10.20.30.148 gw 10.20.30.40
route del -net 10.20.30.40 netmask 255.255.255.248 eth0
route del -net 10.20.30.48 netmask 255.255.255.248 gw 10.20.30.41
route del -net 192.168.1.0/24 eth1
route del default gw 192.168.1.1

```


ifconfig查看、配置、启用或禁用网络接口的命令

```bash
#查看网络
ifconfig
#启用或关闭网卡
ifconfig eth0 up
ifconfig eth0 down

#为网卡配置和删除IPv6地址
ifconfig eth0 add 33ffe:3240:800:1005::2/64
ifconfig eth0 del 33ffe:3240:800:1005::2/64

#配置IP地址
ifconfig eth0 192.168.0.56 
ifconfig eth0 192.168.0.56 netmask 255.255.255.0 
ifconfig eth0 192.168.0.56 netmask 255.255.255.0 broadcast 192.168.0.255

#启用和关闭ARP协议
ifconfig eth0 arp
ifconfig eth0 -arp

#设置最大传输单元
ifconfig eth0 mtu 1500
```

ifup启动网卡

```
ifup eth0
```

ifdown关闭网卡

```
ifdown eth0
```


netstat查看网络状态

```bash
#显示所有tcp 监听端口
netstat -altp
#显示所有tcp 监听端口，直接显示port而不是服务名
netstat -altpn
#显示每个协议的统计信息
netstat -s
netstat -st 
#持续输出 netstat 信息
netstat -t -c 2
#显示核心路由信息
netstat -rn
#找出程序运行的端口
netstat -apn | grep ssh
#找出运行在指定端口的进程
netstat -an | grep ':22'
#显示网络接口列表
netstat -i
netstat -ie
```

ss查看网络状态

```bash
#显示TCP连接
ss -t -a
#查看进程使用的socket
ss -pl
ss -pl|grep 22
#显示所有UDP Sockets
ss -u –a
#显示所有状态为established的SMTP连接
ss -o state 'established'
ss -o state established '( dport = :smtp or sport = :smtp )'
#列举出处于 FIN-WAIT-1状态的源端口为 80或者 443，目标网络为 193.233.7/24所有 tcp套接字
ss -o state FIN-WAIT-1 dst 192.168.25.100/24
#匹配远程地址和端口号
ss dst 192.168.25.100
ss dst 192.168.25.100:50460
#匹配本地地址和端口号
ss src 192.168.25.140
```



### 深入网络操作命令

nmap网络扫描命令

```

```

lsof列举系统中已经被打开的文件

```bash
lsof
lsof|grep sshd
```

mail发送和接收邮件

```

```

mutt邮件管理命令

```

```

nslookup交互式查询互联网 DNS 服务器的命令

```
nslookup
nslookup www.baidu.com
```


dig查找 DNS 解析过程

```bash
dig www.163.com
#只查询DNS记录，即A记录
dig 163.com A +noall +answer
#查找163.com的权威DNS
dig 163.com NS +noall +answer
```

host查询 DNS 的命令

```
host www.163.com
host -a www.163.com
```

traceroute追踪数据传输路由状况

```bash
traceroute www.baidu.com 
#每个网关发送4个数据包
traceroute -q 4 www.baidu.com
#跳数设置 
traceroute -m 10 www.baidu.com 
#探测包使用的基本UDP端口设置6888
traceroute -p 6888 www.baidu.com
#把对外发探测包的等待响应时间设置为3秒
traceroute -w 3 www.baidu.com
```


tcpdump命令行的抓包工具

```bash
#默认监视第一个网络接口上所有流过的数据包
tcpdump
tcpdump -i eth0
#打印所有进入或离开指定地址的的数据包.
tcpdump host 192.168.1.1
#截获主机210.27.48.1 和主机210.27.48.2 或210.27.48.3的通信
tcpdump host 210.27.48.1 and \ (210.27.48.2 or 210.27.48.3 \) 
#打印ace与任何其他主机之间通信的IP 数据包, 但不包括与helios之间的数据包.
tcpdump ip host ace and not helios
#如果想要获取主机210.27.48.1除了和主机210.27.48.2之外所有主机通信的ip包，使用命令：
tcpdump ip host 210.27.48.1 and ! 210.27.48.2
#截获主机hostname发送的所有数据
tcpdump -i eth0 src host hostname
#监视所有送到主机hostname的数据包
tcpdump -i eth0 dst host hostname

#监视指定主机和端口的数据包
tcpdump tcp port 23 and host 210.27.48.1
tcpdump udp port 123 
```



### 有关磁盘与文件系统的命令

mount挂载文件系统

```bash
mount /dev/sda2 /data
mount -t ext3 /dev/sda2 /data
```

umount卸载文件系统

```bash
umount /data
umount /dev/sda2
#查看被占用的进程
fuser -v /data
#install fuser: yum install psmisc
#终止所有在正访问指定的文件系统的进程：慎用
fuser -km /data
```

fsck检查并修复 Linux 文件系统

```bash
#用来检查和维护不一致的文件系统。若系统掉电或磁盘发生问题，可利用fsck命令对文件系统进行检查
#/dev/hda2硬盘的分区有问题
fsck -y /dev/hda2
#自检全部的硬盘
fsck
```

dd转换或复制文件

```bash
#将本地的/dev/hdb整盘备份到/dev/hdd
dd if=/dev/hdb of=/dev/hdd

#将/dev/hdb全盘数据备份到指定路径的image文件
dd if=/dev/hdb of=/root/image

#将备份文件恢复到指定盘
dd if=/root/image of=/dev/hdb

#备份/dev/hdb全盘数据，并利用gzip工具进行压缩，保存到指定路径
dd if=/dev/hdb | gzip > /root/image.gz

#将压缩的备份文件恢复到指定盘
gzip -dc /root/image.gz | dd of=/dev/hdb

#备份与恢复MBR
#备份磁盘开始的512个字节大小的MBR信息到指定文件：
dd if=/dev/hda of=/root/image count=1 bs=512
#count=1指仅拷贝一个块；bs=512指块大小为512个字节。
#恢复：
dd if=/root/image of=/dev/had

#备份软盘
dd if=/dev/fd0 of=disk.img count=1 bs=1440k (即块大小为1.44M)

#拷贝内存内容到硬盘
dd if=/dev/mem of=/root/mem.bin bs=1024 (指定块大小为1k)  

#拷贝光盘内容到指定文件夹，并保存为cd.iso文件
dd if=/dev/cdrom(hdc) of=/root/cd.iso

#增加swap分区文件大小
#第一步：创建一个大小为256M的文件：
dd if=/dev/zero of=/swapfile bs=1024 count=262144
#第二步：把这个文件变成swap文件：
mkswap /swapfile
#第三步：启用这个swap文件：
swapon /swapfile
#第四步：编辑/etc/fstab文件，使在每次开机时自动加载swap文件：
/swapfile    swap    swap    default   0 0

#销毁磁盘数据
dd if=/dev/urandom of=/dev/hda1
#注意：利用随机的数据填充硬盘，在某些必要的场合可以用来销毁数据。

#测试硬盘的读写速度
dd if=/dev/zero bs=1024 count=1000000 of=/root/1Gb.file
dd if=/root/1Gb.file bs=64k | dd of=/dev/null
#通过以上两个命令输出的命令执行时间，可以计算出硬盘的读、写速度。

#确定硬盘的最佳块大小：
dd if=/dev/zero bs=1024 count=1000000 of=/root/1Gb.file
dd if=/dev/zero bs=2048 count=500000  of=/root/1Gb.file
dd if=/dev/zero bs=4096 count=250000  of=/root/1Gb.file
dd if=/dev/zero bs=8192 count=125000  of=/root/1Gb.file
#通过比较以上命令输出中所显示的命令执行时间，即可确定系统最佳的块大小。

#修复硬盘：
dd if=/dev/sda of=/dev/sda 
dd if=/dev/hda of=/dev/hda
#当硬盘较长时间(一年以上)放置不使用后，磁盘上会产生magnetic flux point，当磁头读到这些区域时会遇到困难，并可能导致I/O错误。当这种情况影响到硬盘的第一个扇区时，可能导致硬盘报废。上边的命令有可能使这些数 据起死回生。并且这个过程是安全、高效的。

#利用netcat远程备份
dd if=/dev/hda bs=16065b | netcat < targethost-IP > 1234
#在源主机上执行此命令备份/dev/hda
netcat -l -p 1234 | dd of=/dev/hdc bs=16065b
#在目的主机上执行此命令来接收数据并写入/dev/hdc
netcat -l -p 1234 | bzip2 > partition.img
netcat -l -p 1234 | gzip > partition.img
#以上两条指令是目的主机指令的变化分别采用bzip2、gzip对数据进行压缩，并将备份文件保存在当前目录。

#将一个大视频文件的第i个字节的值改成0x41（大写字母A的ASCII值）
echo A | dd of=bigfile seek=$i bs=1 count=1 conv=notrunc

```

dumpe2fs导出 ext2/ext3/ext4 文件系统信息

```bash
#查看某个磁盘的所有信息


```

dump ext2/3/4 文件系统备份工具

```
dumpe2fs /dev/sda1
dumpe2fs /dev/sda1 | grep -i "inode size"
```

fdisk磁盘分区命令，适用于 2TB 以下磁盘分区

```

```

parted磁盘分区命令，没有磁盘大小限制，常用于 2TB 以下磁盘分区

```

```

mkfs格式化创建 Linux 文件系统

```bash
mfks -t ext3 /dev/sda1
mkfs.ext3    /dev/sda1 
```

partprobe更新内核的硬盘分区表信息

```bash
#partprobe: 通知系统分区表的变化
partprobe /dev/sdb
```


e2fsck检查 ext2/ext3/ext4 类型文件系统

```bash
#执行 e2fsck 或 fsck 前请先 umount partition，否则有机会令档案系统毁损
e2fsck -a  /dev/sda1
```

mkswap创建 Linux 交换分区

```bash
#创建交换分区，指定页大小2048
mkswap -p 2048 /dev/sdb1
```

swapon启用交换分区

swapoff关闭交换分区

```bash
swapoff /dev/sda2
```


sync将内存缓冲区内的数据写入磁盘

```bash
#迫使缓冲块数据立即写盘并更新超级块
sync
```

resize2fs调整 ext2/ext3/ext4 文件系统大小

```bash
#lvextend或其他分区扩容后，生效需要执行resize2fs
resize2fs /dev/mapper/centos-root
```



### 系统权限及用户授权相关命令

chmod改变文件或目录权限

```bash
#给file的属主增加执行权限
chmod u+x file                　　　 
#给file的属主分配读、写、执行(7)的权限，给file的所在组分配读、执行(5)的权限，给其他用户分配执行(1)的权限
chmod 751 file                　　　   
chmod u=rwx,g=rx,o=x file      
#为所有用户分配读权限
chmod =r file 
chmod 444 file              　　　　 
chmod a-wx,a+r   file   　　 　   

#递归地给directory目录下所有文件和子目录的属主分配读的权限
chmod -R u+r directory
#设置用ID，给属主分配读、写和执行权限，给组和其他用户分配读、执行的权限。
chmod 4755 
```

chown改变文件或目录的属主和属组

```bash
chown -R username test
chown -R username:groupname test
```

chgrp更改文件用户组

```bash
chgrp 0 log1
chgrp  -Rv dba dir1
#将log2的所属组改为和log1一样
chgrp  -v --reference=log1 log2
```

umask显示或设置权限掩码

```bash
#设置登录系统之后创建文件或目录的默认权限的
#查询
umask
#临时修改
umask 023
#永久修改
#/etc/profile文件或是修改/etc/bashrc添加行: umask 023
```



### 查看系统用户登陆信息的命令

whoami显示当前有效的用户名称，相当于执行 id -un 命令

```bash
whoami
```


who显示目前登录系统的用户信息

```

```

w显示已经登陆系统的用户列表，并显示用户正在执行的指令

```
w
```

last显示登入系统的用户

```
last
last -n 10
last -10
#将IP 地址转换为主机地址
last -10 -d
```

lastlog显示系统中所有用户最近一次登录信息

```bash
lastlog
```

users显示当前登录系统的所有用户的用户列表

```bash
users
```

finger查找并显示用户信息

```bash

```





### 内置命令及其它

echo打印变量,或直接输出指定的字符串

```bash
echo "hello $HOSTNAME"
echo hello $HOSTNAME
#转义
echo "hello \"$HOSTNAME\""

\a 发出警告声；
\b 删除前一个字符；
\c 最后不加上换行符号；
\f 换行但光标仍旧停留在原来的位置；
\n 换行且光标移至行首；
\r 光标移至行首，但不换行；
\t 插入tab；
\v 与\f相同；
\\ 插入\字符；
\nnn 插入nnn（八进制）所代表的ASCII字符；

#原样输出字符串，不进行转义或取变量(用单引号)
echo 'hello $HOSTNAME'

#显示命令执行结果
echo `date`
```

 

printf将结果格式化输出到标准输出

```bash
#类似c语言
printf 'the integer is:%d\nthe string is: %s\n' 3 "test"
```

rpm管理 rpm 包的命令

```bash
#rpm
－ivh：安装显示安装进度--install--verbose--hash
－Uvh：升级软件包--Update；
－qpl： 列出RPM软件包内的文件信息[Query Package list]；
－qpi：列出RPM软件包的描述信息[Query Package install package(s)]；
－qf：查找指定文件属于哪个RPM软件包[Query File]；
－Va：校验所有的 RPM软件包，查找丢失的文件[View Lost]；
－e：删除包
rpm -q samba    #查询程序是否安装
rpm -ivh /media/cdrom/RedHat/RPMS/samba-3.0.10-1.4E.i386.rpm #按路径安装并显示进度
rpm -ivh --relocate /=/opt/gaim gaim-1.3.0-1.fc4.i386.rpm    #指定安装目录
rpm -ivh --test gaim-1.3.0-1.fc4.i386.rpm　　　 #用来检查依赖关系；并不是真正的安装；
rpm -Uvh --oldpackage gaim-1.3.0-1.fc4.i386.rpm #新版本降级为旧版本
rpm -qa | grep httpd　　　　　 #[搜索指定rpm包是否安装]--all搜索*httpd*
rpm -ql httpd　　　　　　　　　#[搜索rpm包]--list所有文件安装目录
rpm -qpi Linux-1.4-6.i368.rpm　#[查看rpm包]--query--package--install package信息
rpm -qpf Linux-1.4-6.i368.rpm　#[查看rpm包]--file
rpm -qpR file.rpm　　　　　　　#[查看包]依赖关系
rpm2cpio file.rpm |cpio -div    #[抽出文件]
rpm -ivh file.rpm 　   #[安装新的rpm]--install--verbose--hash
rpm -ivh http://mirrors.kernel.org/fedora/core/4/i386/os/Fedora/RPMS/gaim-1.3.0-1.fc4.i386.rpm
rpm -Uvh file.rpm    #[升级一个rpm]--upgrade
rpm -e file.rpm      #[删除一个rpm包]--erase
```

yum自动化简单化地管理 rpm 包的命令

```bash
#自动搜索最快镜像插件:
yum install yum-fastestmirror
#安装yum图形窗口插件:
yum install yumex
#查看可能批量安装的列表:
yum grouplist

#安装
yum install            #全部安装
yum install package1   #安装指定的安装包package1
yum groupinsall group1 #安装程序组group1

#更新和升级
yum update             #全部更新
yum update package1    #更新指定程序包package1
yum check-update       #检查可更新的程序
yum upgrade package1   #升级指定程序包package1
yum groupupdate group1 #升级程序组group1

#查找和显示
yum info package1      #显示安装包信息package1
yum list               #显示所有已经安装和可以安装的程序包
yum list package1      #显示指定程序包安装情况package1
yum groupinfo group1   #显示程序组group1信息yum search string 根据关键字string查找安装包

#删除程序
yum remove package1     #删除程序包package1
yum groupremove group1  #删除程序组group1
yum deplist package1    #查看程序package1依赖情况

#清除缓存
yum clean packages      #清除缓存目录下的软件包
yum clean headers       #清除缓存目录下的 headers
yum clean oldheaders    #清除缓存目录下旧的 headers
yum clean               #清除缓存目录下的软件包及旧的headers
yum clean all   

#程序组安装
yum grouplist
yum groupinstall "GNOME Desktop Environment"

```



watch周期性的执行给定的命令，并将命令的输出以全屏方式显示

```bash
#每5秒执行一次
watch -n 5 iostat
#显示差异
watch -d 'ls -l'
```

alias设置系统别名

```bash
alias cddev='cd /home/develop/'

#永久生效  添加到 .bashrc

```

unalias取消系统别名

```
unalias cddev
```

date查看或设置系统时间

```bash
date +"%Y-%m-%d"
date -d "1 day ago" +"%Y-%m-%d"
date -d "2 second" +"%Y-%m-%d %H:%M.%S"
date -d "2009-12-12" +"%Y/%m/%d %H:%M.%S"

date -s                        #设置当前时间，只有root权限才能设置，其他只能查看
date -s 20120523               #设置成20120523，这样会把具体时间设置成空00:00:00
date -s 01:01:01               #设置具体时间，不会对日期做更改
date -s "01:01:01 2012-05-23"  #这样可以设置全部时间
date -s "01:01:01 20120523"    #这样可以设置全部时间
date -s "2012-05-23 01:01:01"  #这样可以设置全部时间
date -s "20120523 01:01:01"    #这样可以设置全部时间

```

clear清除屏幕，简称清屏

```bash
clear
printf "\033c"
```

history查看命令执行的历史纪录

```bash
history
-c：清空当前历史命令；
-a：将历史命令缓冲区中命令写入历史命令文件中；
-r：将历史命令文件中的命令读入当前历史命令缓冲区；
-w：将当前历史命令缓冲区命令写入历史命令文件中;
-d<offset>：删除历史记录中第offset个命令
```

eject弹出光驱

```

```

time计算命令执行时间

```
time ls 
```

nc功能强大的网络工具

```bash
#连到192.168.x.x的TCP80端口.
nc -nvv 192.168.x.x 80
#该连接将在 10 秒后中断。
nc -w 10 localhost 2389

nc -v 127.0.0.1 22

#监听本地主机:
nc -l 80

#端口扫描
nc -z -v -n 192.168.1.1 21-25

#文件传输
#server
nc -l 20000 < file.txt
#client
nc -n 192.168.1.1 20000 > file.txt

#目录传输
#发送一个文件很简单，但是如果我们想要发送多个文件，或者整个目录，一样很简单，只需要使用压缩工具tar，压缩后发送压缩包。如果你想要通过网络传输一个目录从A到B。
#Server
tar -cvf – dir_name | nc -l 20000
#Client
nc -n 192.168.1.1 20000 | tar -xvf -

#压缩
tar -cvf – dir_name| bzip2 -z | nc -l 20000
nc -n 192.168.1.1 20000 | bzip2 -d |tar -xvf -


#加密你通过网络发送的数据
#Server
nc localhost 20000 | mcrypt –flush –bare -F -q -d -m ecb > file.txt
#使用mcrypt工具加密数据。
#Client
mcrypt –flush –bare -F -q -m ecb < file.txt | nc -l 20000

#克隆一个设备
#Server
dd if=/dev/sda | nc -l 20000
#Client
nc -n 192.168.1.1 20000 | dd of=/dev/sda

#打开一个shell
#Server
nc -l 20000 -e /bin/bash
#Client
nc localhost 20000

#反向shell
#Server
nc -l 20000
#在客户端，简单地告诉netcat在连接完成后，执行shell。
#Client
nc localhost 20000 -e /bin/bash

```

xargs将标准输入转换成命令行参数

```bash
#之所以能用到这个命令，关键是由于很多命令不支持|管道来传递参数，而日常工作中有有这个必要，所以就有了xargs命令，例如：
#这个命令是错误的
find /sbin -perm +700 |ls -l
#这样才是正确的
find /sbin -perm +700 |xargs ls -l  

-0         #当sdtin含有特殊字元时候，将其当成一般字符，想/'空格等
-a file    #从文件中读入作为sdtin
-e flag    #注意有的时候可能会是-E，flag必须是一个以空格分隔的标志，当xargs分析到含有flag这个标志的时候就停止。
-n num  #后面加次数，表示命令在执行的时候一次用的argument的个数，默认是用所有的。
-p      #操作具有可交互性，每次执行comand都交互式提示用户选择，当每次执行一个argument的时候询问一次用户
-t      #表示先打印命令，然后再执行。
-i      #或者是-I，这得看linux支持了，将xargs的每项名称，一般是一行一行赋值给{}，可以用{}代替。
ls *.txt |xargs -t -i mv {} {}.bak
-r  no-run-if-empty  #如果没有要处理的参数传递给xargsxargs 默认是带 空参数运行一次，如果你希望无参数时，停止 xargs，直接退出，使用 -r 选项即可，其可以防止xargs 后面命令带空参数运行报错。

```


exec调用并执行指令的命令

```bash
find . -name "*.txt"  -exec echo {} \;
find . -name "*.txt" |xargs  echo
find . -name "*.txt" |xargs -n 1 echo
find . -regextype posix-egrep -regex "./[0-9]{1,5}.txt" -exec ./test.sh {} \;
find . -regextype posix-egrep -regex "./[0-9]{1,5}.txt" -exec ./test.sh {} +
find . -regextype posix-egrep -regex "./[0-9]{1,5}.txt" |xargs ./test.sh
```

export设置或者显示环境变量

```bash
#修改profile文件
```


unset删除变量或函数

```

```

type用于判断另外一个命令是否是内置命令

```bash
#type命令的基本使用方式就是直接跟上命令名字。
type -a  ps  #可以显示所有可能的类型，比如有些命令如pwd是shell内建命令，也可以是外部命令。
type -p  ps  #只返回外部命令的信息，相当于which命令。
type -f  ps  #只返回shell函数的信息。
type -t  ps  #只返回指定类型的信息。
```

bc命令行科学计算器

```bash
#10进制转2进制
echo "obase=2;ibase=10;100" | bc
#10进制转16进制
echo "obase=16;ibase=10;100" | bc
#16进制转2进制：
echo "ibase=16;obase=2;F1" | bc
```



### 系统管理与性能监视命令

chkconfig管理 Linux 系统开机启动项

```bash
chkconfig --list             #列出所有的系统服务
chkconfig --add httpd        #增加httpd服务
chkconfig --del httpd        #删除httpd服务
chkconfig --level httpd 2345 on  #设置httpd在运行级别为2、3、4、5的情况下都是on（开启）的状态
chkconfig --list          #列出系统所有的服务启动情况
chkconfig --list mysqld   #列出mysqld服务设置情况
chkconfig --level 35 mysqld on    #设定mysqld在等级3和5为开机运行服务，--level 35表示操作只在等级3和5执行，on表示启动，off表示关闭
chkconfig mysqld on        #设定mysqld在各等级为on，“各等级”包括2、3、4、5等级

```

vmstat虚拟内存统计

```bash
vmstat 1      #1表示每秒采集一次
vmstat 2 1    #2表示2秒采集一次，1表示只采集一次

r 表示运行队列(就是说多少个进程真的分配到CPU)
b 表示阻塞的进程
swpd 虚拟内存已使用的大小
free   空闲的物理内存的大小
buff 设备和设备之间的缓冲 
cache  cpu和内存之间的缓冲
si  每秒从磁盘读入虚拟内存的大小
so  每秒虚拟内存写入磁盘的大小，如果这个值大于0，同上。
bi  块设备每秒接收的块数量，这里的块设备是指系统上所有的磁盘和其他块设备，默认块大小是1024byte
bo  块设备每秒发送的块数量，例如我们读取文件，bo就要大于0。bi和bo一般都要接近0，
in  每秒CPU的中断次数，包括时间中断
cs  每秒上下文切换次数
us 用户CPU时间
sy 系统CPU时间，如果太高，表示系统调用时间长，例如是IO操作频繁。
id  空闲 CPU时间，一般来说，id + us + sy = 100,一般我认为id是空闲CPU使用率，us是用户CPU使用率，sy是系统CPU使用率。
wa 等待IO CPU时间。
```


mpstat显示各个可用 CPU 的状态统计

```bash
mpstat
#每个cpu核心的详细当前运行状况信息
mpstat  -P ALL 2

%user      在internal时间段里，用户态的CPU时间(%)，不包含nice值为负进程  (usr/total)*100
%nice      在internal时间段里，nice值为负进程的CPU时间(%)   (nice/total)*100
%sys       在internal时间段里，内核时间(%)       (system/total)*100
%iowait    在internal时间段里，硬盘IO等待时间(%) (iowait/total)*100
%irq       在internal时间段里，硬中断时间(%)     (irq/total)*100
%soft      在internal时间段里，软中断时间(%)     (softirq/total)*100
%idle      在internal时间段里，CPU除去等待磁盘IO操作外的因为任何原因而空闲的时间闲置时间(%) (idle/total)*100
```

iostat统计系统 IO

```bash
iostat
-c：只显示系统CPU统计信息，即单独输出avg-cpu结果，不包括device结果
-d：单独输出Device结果，不包括cpu结果
-k/-m：输出结果以kB/mB为单位，而不是以扇区数为单位
-x:输出更详细的io设备统计信息
interval/count：每次输出间隔时间，count表示输出次数，不带count表示循环输出

Device: 以sdX形式显示的设备名称
tps: 每秒进程下发的IO读、写请求数量
Blk_read/s: 每秒读扇区数量(一扇区为512bytes)
Blk_wrtn/s: 每秒写扇区数量
Blk_read: 取样时间间隔内读扇区总数量
Blk_wrtn: 取样时间间隔内写扇区总数量

iostat -x -k -d 1 2    #每隔1S输出磁盘IO的详细详细，总共采样2次。
# 重点关注参数
1、iowait% 表示CPU等待IO时间占整个CPU周期的百分比，如果iowait值超过50%，或者明显大于%system、%user以及%idle，表示IO可能存在问题。
2、avgqu-sz 表示磁盘IO队列长度，即IO等待个数。
3、await 表示每次IO请求等待时间，包括等待时间和处理时间
4、svctm 表示每次IO请求处理的时间
5、%util 表示磁盘忙碌情况，一般该值超过80%表示该磁盘可能处于繁忙状态。
```

sar全面地获取系统的 CPU、运行队列、磁盘 I/O、分页（交换区）、内存、 CPU 中断和网络等性能数据

```bash
sar命令的选项很多，下面只列出常用选项： 
-A：所有报告的总和 
-u：输出整体CPU使用情况的统计信息 
-v：输出inode、文件和其他内核表的统计信息 
-d：输出每一个块设备的活动信息 
-r：输出内存和交换空间的统计信息 
-b：显示I/O和传送速率的统计信息 
-a：文件读写情况 
-c：输出进程统计信息，每秒创建的进程数 
-R：输出内存页面的统计信息 
-y：终端设备活动情况 
-w：输出系统交换活动信息

#整体CPU使用统计(-u) 默认
#采样时间为3s，采样次数为2次，整体CPU的使用情况
sar 3 2
sar -u 3 2
#各个CPU使用统计(-P)
sar -P ALL 3  2     #选项指示对每个内核输出统计信息
sar -P 0 3  2       #选项指示对0内核输出统计信息

#内存使用情况统计(-r)
sar -r 1 2 

#整体I/O情况(-b)
sar -b 3 2 

#各个I/O设备情况(-d)
#使用-d选项可以显示各个磁盘的统计信息，再增加-p选项可以以sdX的形式显示设备名称： 
sar -d -p 3 2 

#网络统计(-n)
sar -n DEV 1 1 
```


ipcs用于报告 Linux 中进程间通信设施的状态，显示的信息包括消息列表、共享内存和信号量的信息

```bash
ipcs -m　　#查看系统共享内存信息
ipcs -q　　#查看系统消息队列信息
ipcs -s　　#查看系统信号量信息
ipcs [-a]　#系统默认输出信息，显示系统内所有的IPC信息

ipcs -c　　#查看IPC的创建者和所有者
ipcs -l　　#查看IPC资源的限制信息
ipcs -p　　#查看IPC资源的创建者和使用的进程ID
ipcs -t　　#查看最新调用IPC资源的详细时间
ipcs -u　　#查看IPC资源状态汇总信息
ipcs -l --human


```

ipcrm用来删除一个或更多的消息队列、信号量集或者共享内存标识

```bash
ipcrm -M shmkey  #移除用shmkey创建的共享内存段
ipcrm -m shmid   #移除用shmid标识的共享内存段
ipcrm -S semkey  #移除用semkey创建的信号量
ipcrm -s semid   #移除用semid标识的信号量
ipcrm -Q msgkey  #移除用msgkey创建的消息队列
ipcrm -q msgid   #移除用msgid标识的消息队列
```

strace用于诊断、调试 Linux 用户空间跟踪器。我们用它来监控用户空间进程和内核的交互，比如**系统调用、信号传递、进程状态变更**等

```bash
#表示跟踪2899进程的所有系统调用，并统计系统调用的时间开销，以及调用起始时间(以可视化的时分秒格式显示)，最后将记录结果存入out.txt文件
strace -o out.txt -T -tt -e trace=all -p 2899
#踪ls -l命令执行过程()
strace ls -l

#统计分析进程所有的系统调用
strace -c ps
#可看到程序调用哪些系统函数，调用次数、所耗时间及出错次数等信息，有助于分析程序运行速度瓶颈

% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 46.92    0.008625          22       392           read
 20.35    0.003741           9       398         7 open
 13.04    0.002397           6       392           close
  8.02    0.001475           8       195         8 stat
  4.57    0.000840          12        70           mmap
  2.59    0.000476          11        45           mprotect
  1.26    0.000231         116         2           getdents
  1.06    0.000195           7        28           fstat
  0.69    0.000126           5        24           rt_sigaction
  0.28    0.000052           9         6           munmap
  0.20    0.000037           9         4           write
  0.20    0.000036          12         3           readlink
  0.16    0.000030          10         3         2 access
  0.13    0.000024           8         3           brk
  0.13    0.000023          12         2         2 statfs
  0.07    0.000013           7         2           ioctl
  0.07    0.000013          13         1           execve
  0.05    0.000010           5         2           lseek
  0.04    0.000008           8         1           openat
  0.03    0.000006           6         1           rt_sigprocmask
  0.03    0.000006           6         1           getrlimit
  0.03    0.000006           6         1           geteuid
  0.03    0.000006           6         1           set_tid_address
  0.03    0.000006           6         1           set_robust_list
  0.00    0.000000           0         1           uname
  0.00    0.000000           0         1           arch_prctl
------ ----------- ----------- --------- --------- ----------------
100.00    0.018382                  1580        19 total

#借助strace找出access调用错误原因
strace -e trace=access ps

access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
access("/etc/system-fips", F_OK)        = -1 ENOENT (No such file or directory)
access("/etc/selinux/config", F_OK)     = 0
  PID TTY          TIME CMD
26977 pts/0    00:00:00 bash
27354 pts/0    00:00:00 strace
27356 pts/0    00:00:00 ps
+++ exited with 0 +++

#跟踪进程执行时的系统调用
strace -p 16168 -s 51200 -o debug.log
-p 需要调试的进程ID
-r 打印出相对时间关于,每一个系统调用
-t 在输出中的每一行前加上时间信息
-s 指定输出的字符串的最大长度.默认为32
-o 将输出写入文件
```

ltrace命令会跟踪进程的库函数调用, 它会显现出哪个**库函数被调用**

```bash
ltrace ps
ltrace psql -c  "select * from pg_class limit 1"
#输出调用时间开销
ltrace -T ps
#显示系统调用
ltrace -S ps
```



### 关机 / 重启 / 注销和查看系统信息的命令

shutdown关机

```bash
#关闭系统且关闭电源
shutdown -h now 

#利用shutdown重启电脑
shutdown -r now    #关机后重启
shutdown -h +10 "10 minute after shutdown"  #10分钟之后关闭系统和电源
shutdown -r +10    #10分钟后重启
shutdown -r 10:00  #10点钟重启
shutdown -h +10    #10分钟后关机
shutdown -h 10:00  #10点钟关机
```

halt关机

```bash
halt   #调用 shutdown -h
```

poweroff关闭电源

```bash
poweroff
```

logout退出当前登录的 Shell

```bash
logout
```


exit退出当前登录的 Shell

```bash
exit
```

Ctrl+d退出当前登录的 Shell 的快捷键

```

```



centos下这些命令都是通过systemctl实现

```bash
ls -alh|grep systemctl

lrwxrwxrwx.  1 root root       16 Jan 11 11:53 halt -> ../bin/systemctl
lrwxrwxrwx.  1 root root       16 Jan 11 11:53 poweroff -> ../bin/systemctl
lrwxrwxrwx.  1 root root       16 Jan 11 11:53 reboot -> ../bin/systemctl
lrwxrwxrwx.  1 root root       16 Jan 11 11:53 runlevel -> ../bin/systemctl
lrwxrwxrwx.  1 root root       16 Jan 11 11:53 shutdown -> ../bin/systemctl
lrwxrwxrwx.  1 root root       16 Jan 11 11:53 telinit -> ../bin/systemctl

systemctl poweroff    #关闭电源
```



### 进程管理相关命令

bg  fg  jobs 命令

```bash
#运行长时间任务
vi demo_job

#ctrl-z挂起这个程序，然后可以看到系统的提示:
[1]+  Stopped                 vi demo_job

#用jobs命令查看任务
jobs

#此时进程处于停止状态, 我们可以让它在后台继续执行
bg 1
[1]+ vi demo_job &

#把它调回到前台运行
fg 1

# &最经常被用到,这个用在一个命令的最后，可以把这个命令放到后台执行
vi demo_job &

# ctrl + z  可以将一个正在前台执行的命令放到后台，并且暂停
# jobs  查看当前有多少在后台运行的命令
# fg 将后台中的命令调至前台继续运行
# 如果后台有多个命令，可以用fg %jobnumber将选中的命令调出，%jobnumber是通过jobs命令查到的后台正在执行的命令的序号（不是pid）
```

kill killall pkill终止进程

```bash
#kill命令用来终止指定的进程
-l  信号，若果不加信号的编号参数，则使用“-l”参数会列出全部的信号名称
-a  当处理当前进程时，不限制命令名和进程号的对应关系
-p  指定kill 命令只打印相关进程的进程号，而不发送任何信号
-s  指定发送信号
-u  指定用户

#列出所有信号名称
kill -l

#只有第9种信号(SIGKILL)才可以无条件终止进程，其他信号进程都有权利忽略。    
#下面是常用的信号：
HUP    1    终端断线
INT     2    中断（同 Ctrl + C）
QUIT    3    退出（同 Ctrl + \）
TERM   15    终止
KILL    9    强制终止
CONT   18    继续（与STOP相反， fg/bg命令）
STOP    19    暂停（同 Ctrl + Z）

#得到指定信号的数值
kill -l KILL
kill -l SIGKILL

#先用ps查找进程，然后用kill杀掉
kill pid
kill -9 pid     #彻底杀死进程


#按照进程名来杀死进程
killall vi

#按照进程名来杀死进程
pkill -9 -t tty1     
pkill -9 -t pts/0    #杀终端  who可以查看在线终端

#一些组合命令
pgrep vi | xargs kill -s 9
kill -s 9 `ps -aux | grep vi | awk '{print $2}'`
```


crontab定时任务命令

```bash
#编辑自动任务文件taskfile，提交给crontab执行
crontab taskfile
#列出crontab 任务
crontab -l
#编辑任务
crontab -e     #vi编辑，保存退出后自动提交任务
 
#任务格式
 # Example of job definition:
 # .---------------- minute (0 - 59)
 # |  .------------- hour (0 - 23)
 # |  |  .---------- day of month (1 - 31)
 # |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
 # |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
 # |  |  |  |  |
 # *  *  *  *  * user-name  command to be executed

#每一小时重启smb服务
* */1 * * * /etc/init.d/smb restart
#默认执行时不带环境变量，如果任务需要环境变量自己加载
0 * * * * . /home/username/.bash_profile;/bin/sh /opt/task_run.sh
```

ps 显示进程的快照

```bash
#-a 代表 all。x参数会显示没有控制终端的进程
ps -ax
ps -ax|less

#根据用户过滤进程
ps -u `whoami`

# 通过cpu和内存使用来过滤进程
ps -aux|less

#默认的结果集是未排好序的。可以通过 --sort命令来排序
ps -aux --sort -pcpu | less
ps -aux --sort -pmem | less
ps -aux --sort -pcpu,+pmem | head -n 10

#通过进程名和PID过滤
ps -C getty
ps -f -C getty

#树形显示进程
ps -axjf

# 显示安全信息 -e 显示所有进程信息，-o 参数控制输出。Pid,User 和 Args参数显示PID，运行应用的用户和该应用。
ps -eo pid,user,args

# 使用PS实时监控进程状态
watch -n 1 -d 'ps -aux --sort -pmem,-pcpu'

#查看进程的niceness
ps -l

```

pstree树形显示进程

```bash
pstree
pstree -p   #显示进程PID
pstree 16162 -p
```

nice/renice调整程序运行的优先级

```

```

nohup忽略挂起信号运行指定的命令

```
nohup command &
nohup command > myout.file 2>&1 &
```

pgrep查找匹配条件的进程

```bash
pgrep -l sshd
pgrep -l -o sshd   #-o 当匹配多个进程时，显示进程号最小的那个
pgrep -l -n sshd   #-n 当匹配多个进程时，显示进程号最大的那个
```

runlevel查看系统当前运行级别

```bash
runlevel

运行级别0：系统停机状态，系统默认运行级别不能设为0，否则不能正常启动 
运行级别1：单用户工作状态，root权限，用于系统维护，禁止远程登陆 
运行级别2：多用户状态(没有NFS) 
运行级别3：完全的多用户状态(有NFS)，登陆后进入控制台命令行模式 
运行级别4：系统未使用，保留 
运行级别5：X11控制台，登陆后进入图形GUI模式 
运行级别6：系统正常关闭并重启，默认运行级别不能设为6，否则不能正常启动


```

init切换运行级别

```bash
#进入其它运行级别用：(sudo) 
init N 
init 0    #为关机，
init 6    #为重启系统
```



service启动、停止、重新启动和关闭系统服务，还可以显示所有系统服务的当前状态

```bash
service --status-all
service <service> start
service <service> stop
service <service> restart
chkconfig --list

#centos新版本采用systemctl
```



### systemd为系统的启动和管理提供一套完整的解决方案

```bash
systemctl --version
#系统管理
#-----------------------------
# 重启系统
systemctl reboot
# 关闭系统，切断电源
systemctl poweroff
# CPU停止工作
systemctl halt
# 暂停系统
systemctl suspend
# 让系统进入冬眠状态
systemctl hibernate
# 让系统进入交互式休眠状态
systemctl hybrid-sleep
# 启动进入救援状态（单用户状态）
systemctl rescue

#Unit
#Systemd 可以管理所有系统资源。不同的资源统称为 Unit（单位）,一共分成12种.
Service unit：系统服务
Target unit：多个 Unit 构成的一个组
Device Unit：硬件设备
Mount Unit：文件系统的挂载点
Automount Unit：自动挂载点
Path Unit：文件或路径
Scope Unit：不是由 Systemd 启动的外部进程
Slice Unit：进程组
Snapshot Unit：Systemd 快照，可以切回某个快照
Socket Unit：进程间通信的 socket
Swap Unit：swap 文件
Timer Unit：定时器

# 列出正在运行的 Unit
systemctl list-units
# 列出所有Unit，包括没有找到配置文件的或者启动失败的
systemctl list-units --all
# 列出所有没有运行的 Unit
systemctl list-units --all --state=inactive
# 列出所有加载失败的 Unit
systemctl list-units --failed
# 列出所有正在运行的、类型为 service 的 Unit
systemctl list-units --type=service

# 显示系统状态
systemctl status
# 显示单个 Unit 的状态
$ sysystemctl status bluetooth.service
# 显示远程主机的某个 Unit 的状态
systemctl -H root@rhel7.example.com status httpd.service

# 显示某个 Unit 是否正在运行
systemctl is-active application.service
# 显示某个 Unit 是否处于启动失败状态
systemctl is-failed application.service
# 显示某个 Unit 服务是否建立了启动链接
systemctl is-enabled application.service
# 立即启动一个服务
systemctl start apache.service
# 立即停止一个服务
systemctl stop apache.service
# 重启一个服务
systemctl restart apache.service
# 杀死一个服务的所有子进程
systemctl kill apache.service
# 重新加载一个服务的配置文件
systemctl reload apache.service
# 重载所有修改过的配置文件
systemctl daemon-reload
# 显示某个 Unit 的所有底层参数
systemctl show httpd.service
# 显示某个 Unit 的指定属性的值
systemctl show -p CPUShares httpd.service
# 设置某个 Unit 的指定属性
systemctl set-property httpd.service CPUShares=500
```





### linux系统日志

```bash
#日志通常存放在 /var/log
#核心启动日志： /var/log/dmesg
#系统报错或重启服务等日志： /var/log/messages
#邮件系统日志： /var/log/maillog
#cron(定制任务日志)日志：/var/log/cron 
#验证系统用户登录： /var/log/secure
#所有的登入和登出: /var/log/wtmp
```

