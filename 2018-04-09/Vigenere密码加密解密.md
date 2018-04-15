---
title: Vigenere密码加密解密
date: 2018-04-14 22:11:24
tags: 密码学
mathjax: true
category: 密码学
author: Leno
thumbnail: 'https://i.loli.net/2018/04/15/5ad30ff4d44d1.png'
---

>今天换个口味，写点原来从没接触过的东西--密码学。前一阵信息安全课上留了一个作业，实现Vigenere加密解密，借着机会写篇博客。这次博客由于比较仓促，这次只写加密解密系统的实现，不涉及唯密文破解。

----


# 任务要求： #
 *      a.编程实现Vigenere加密/解密系统，并分析和评估该算法的安全性。
 *      b.编程实现唯密文破译系统，能够破译密钥为2到4个字符的Vigenere密文，并分析如何加快破译速度。
 *  时间要求： 布置任务后，在3周之内完成。
 *  提交结果：已设计并测试好的程序，包括源码、可执行程序、测试数据集、实验报告。

# 原理介绍： #
先普及下Caesar密码，作为单密码简单替换密码届的扛把子，他有着不可动摇的地位，它的原理很简单，对于需要加密的每个字符都进行相同大小的平移。先给出Caesar密码加密的字符对应表，如下：

![Caesar.png](https://i.loli.net/2018/04/15/5ad32ed02524c.png)

举个例子吧：明文为China，它对应的数字应为2 7 8 13 0.比如我们平移距离为3，那么加密之后的密文应该为FKLQD，简单到炸。

我们先不谈上述加密方法的缺点，这些我们放到唯密文解密中聊。有了以上的基础之后我们再聊Vigenere加密以及解密，它是使用一系列凯撒密码组成密码字母表的加密算法，属于多表密码的一种简单形式。同样先给出它的密码加密字符对应表（自己画太麻烦了，我就在百科上扒了一个图，溜。。）：

![222.png](https://i.loli.net/2018/04/15/5ad33112af0e9.png)

上图中的维吉尼亚表的第一列代表着密钥字母（这是有别于Caesar密码的地方），第一行代表着明文字母，行列分别使用当前需要加密字符和当前的密钥字符确定当前明文字符对应着的密文字符。

这里我们说一下，一般情况下，我们给出的密钥是短于我们的明文长度的。所以我们做Vigenere加密的时候第一步做的就是对照明文长度，补齐密钥字串。

说了这么多，举个例子说下吧：

* 例如我们的明文字串为：data security 

* 密钥：best

按照上述的规则，第b行，第d列，对应的字符为E，..... 加密之后的密文应为：EELTTIUNSMLR（不区分大小写）。

# 实现 #

### 第一步：使得密钥字符串长度与明文长度相同 ###


```java
public String dealKey(String str,String Key){   
        Key=Key.toUpperCase();// 将密钥转换成大写
        Key=Key.replaceAll("[^A-Z]", "");//去除所有非字母的字符  

		StringBuilder stringBuilder = new StringBuilder(Key);
        String newKey="";
        if(sstringBuilder.length()!=str.length()){ 
            //如果密钥长度与str不同，则需要生成密钥字符串
            if(stringBuilder.length()<str.length()){
                //如果密钥长度比str短，则以不断重复密钥的方式生成密钥字符串
                while(stringBuilder.length()<str.length()){
                    stringBuilder.append(Key);
                }
            }
            //此时，密钥字符串的长度大于或等于str长度
            //将密钥字符串截取为与str等长的字符串
            newKey=stringBuilder.substring(0, str.length());
        }
        return newKey;
    }

```

### 第二步：加密 ###

其实我们不用将上述的Vigenere密码表列出来，一是列出来费时费力费空间，二是一个简单的取余操作就能解决这个事情。

```java
private String PwTable = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";

public String Encryption(String P,String K){
        P = P.toUpperCase();// 将明文转换成大写
        P = P.replaceAll("[^A-Z]", "");//去除所有非字母的字符   
        K = dealKey(P,K);
        int len = K.length();
        StringBuilder stringBuilder = new StringBuilder();
        for(int i = 0;i < len;i++){
            int row=PwTable.indexOf(K.charAt(i));//行号
            int col=PwTable.indexOf(P.charAt(i));//列号
            int index = (row+col)%26;
            stringBuilder.append(PwTable.charAt(index));
        }
        return stringBuilder.toString();       
    }

```

### 第三步：解密 ###

这个过程其实是比较有趣的，我们取密钥所在行为解密的行号，密文所在列作为我们的列号，我们需要分两种情况考虑：

首先说一下第一种情况，将上述的密码表从主对角线分开，一是密文在我们的密码表的右上部，这种情况比较简单，其实就是一个加密过程的逆过程，我们此时的密文字符在密码表中的位置肯定是不比其对应的明文靠后的（理解这句话，这种情况其实也就明白了，再简单点讲就是PwTable.indexof(密文)>PwTable.indexof(对应的明文)），此时我们只要让列号减去行号，然后将结果做indexof操作就能得到相应的明文。

如果你理解了第一种情况，第二种情况也就好理解了，此时我们的密文字符在我们的密码表中的位置肯定是不比其对应的明文的位置靠前的，我们需要将列号加一圈字符表再去减行号。

理解之后下面的这些代码应该就不难理解了。

```java
public String Decryption(String C,String K){
        C=C.toUpperCase();// 将密文转换成大写
        C=C.replaceAll("[^A-Z]", "");//去除所有非字母的字符   
        K=dealKey(C,K);
        int len = K.length();
        StringBuilder stringBuilder=new StringBuilder();
        for(int i = 0;i<len;i++){
            int row = PwTable.indexOf(K.charAt(i));//行号
            int col = PwTable.indexOf(C.charAt(i));//列号
            int index;
            if(row>col){
                index=col+26-row;
            }else{
                index=col-row;
            }
            stringBuilder.append(PwTable.charAt(index));
        }
        return sb.toString();     
    }
```

以上是本篇博客的全部内容，希望对你有所帮助，感谢驻足~