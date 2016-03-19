---
layout: post
title: utf转GBK
---
目前github上的很多代码都是utf8编码，然而阅读源码时，SI软件中UTF8中文会乱码  
解决方法是：将UTF8转化GBK,重新打开后Source Insight软件中文不会乱码  
代码使用方法：直接运行convert_utf_2_asni函数，指定代码目录即可  

```
import os

def convertEncoding(from_encode,to_encode,old_filepath,target_file):
    f1=file(old_filepath)
    content2=[]
    content2.append("//flag: convert utf-8 to ASNI\n") #在文件首部标记转换
    while True:
        line=f1.readline()
        content2.append(line.decode(from_encode).encode(to_encode))
        if len(line) ==0:
            break
    f1.close()
    f2=file(target_file,'w')
    f2.writelines(content2)
    f2.close()

#把文件由GBK编码转换为UTF-8编码，也就是filepath的编码是GBK
def convertFromGBK2utf8(filepath):
    convertEncoding("GBK", "UTF-8", filepath, filepath+".bak")


#把文件由UTF-8编码转换为GBK编码，也就是filepath的编码是UTF-8
def convertFromUTF82gbk(filepath):
    convertEncoding("UTF-8", "GBK", filepath, filepath)
    print "convert success"

#打印rootDir目录下的树形结构
def tree_in_python(rootDir, level=1): 
    if level==1: print rootDir 
    for lists in os.listdir(rootDir): 
        path = os.path.join(rootDir, lists) 
        print '│  '*(level-1)+'│--'+lists 
        if os.path.isdir(path): 
            tree_in_python(path, level+1) 

#打印rootDir目录下的所有文件名
def list_file(rootDir): 
    list_dirs = os.walk(rootDir) 
    for root, dirs, files in list_dirs:   
        for f in files:   
            print os.path.join(root, f) 

#转换目录下所有的.c或者.h文件，方便sourceInsight阅读源代码
def convert_utf_2_asni(srcDir):
    list_dirs = os.walk(srcDir) 
    for root, dirs, files in list_dirs:     
        for f in files: 
            if f.endswith(".c") or f.endswith(".h"):
                filename = os.path.join(root,f)
                print filename
                convertFromUTF82gbk(filename)
    
            

# filepath="G:\\code\\opensource_code\\libevent_asni\\libevent-2.0.22-stable\\buffer.c"
# filepath="utf_test.tx"
# convertFromUTF82gbk(filepath)

#转换该目录下所有.c或者.h文件，从UTF-8到GBK,方便sourceInsight阅读源代码
convert_utf_2_asni("G:\code\opensource_code\libevent_asni\libevent-2.0.22-stable")
```

