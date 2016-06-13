---
layout: post
title: URL_shortener简单实现
categories: Python
---
`URL Shortening`是一项将长网址转换为短网址的技术，方便用户沟通交流，尤其是在限制字符长度的场景下。  

比如 "http://en.m.wikipedia.org/wiki/URL_shortening" 可以缩短为"https://goo.gl/vvwncI".  

更多资料请参考`Wiki`：[URL Shortener](https://en.wikipedia.org/wiki/URL_shortening "URL Shortener")  

### 思路：
1. 用`uuid1`为每个网址生成一个唯一id  
2. 将id和网址用redis存储
3. 用"abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789" 64个字符来编码网址id，再加上固定短网址前缀即可  
4. 解码时，需要用短网址来获取id，再从`redis`中取得长网址，返回即可   

### Python实现
```python
#encoding:utf-8
import uuid
import redis

r = redis.Redis(host='localhost',port=6379,db=0)

ALPHABET = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
BASE = len(ALPHABET)
#短地址前缀
STATIC_URL="http://yoururl/"

#生成唯一id
def generate_id():
    return uuid.uuid1().int
#编码
def encode(num):
    ret_str=''
    while num:
        index = num % BASE
        ret_str += ALPHABET[index]
        num /= BASE
    return ret_str[::-1]

#解码
def decode(str):
    num=0
    length = len(str)
    index=0
    while index < length:
        num = num*BASE + ALPHABET.index(str[index])
        index += 1
    return num

#获取短url
def url_short(url):
    url_id = generate_id()
    encode_url = encode(url_id)
    r.set(url_id,url)
    return STATIC_URL+encode_url

#返回长url
def url_long(short_url):
    short_url = short_url[-22:]
    url_id = decode(short_url)
    long_url = r.get(url_id)
    return long_url

#example
if __name__ == "__main__":
    url ="http://stackoverflow.com/questions/742013/how-to-code-a-url-shortener"
    url_short_str = url_short(url)
    url_long_str = url_long(url_short_str)
    print url_short_str
    print url_long_str
```

### Reference
1. [URL_shortening](https://en.wikipedia.org/wiki/URL_shortening "URL_shortening")
2. [Building Your Own URL Shortener](https://www.sitepoint.com/building-your-own-url-shortener/ "Building Your Own URL Shortener")
3. [How to code a URL shortener?](http://stackoverflow.com/questions/742013/how-to-code-a-url-shortener "How to code a URL shortener?")
4. [站长之家---在线短链生成器](http://tool.chinaz.com/tools/dwz.aspx)