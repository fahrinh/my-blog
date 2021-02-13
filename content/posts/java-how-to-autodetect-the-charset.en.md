---
title: "Java: How To Autodetect The Charset Encoding of A Text File and Remove Byte Order Mark (BOM)"
date: 2017-11-08T04:42:36+07:00
categories: ["Java"]
tags: ["BOM", "Byte Order Mark", "Charset Encoding"]
draft: false
toc: true
aliases: [/post/2017-11-08-java-how-to-autodetect-the-charset/]
---

## TLDR;
Required dependencies (`pom.xml`) :

```xml
<dependency>
    <groupId>com.ibm.icu</groupId>
    <artifactId>icu4j</artifactId>
    <version>60.1</version>
</dependency>
<dependency>
      <groupId>commons-io</groupId>
      <artifactId>commons-io</artifactId>
      <version>2.6</version>
</dependency>
```

Autodetect the charset encoding of a text file or input stream then 'remove' (skip) Byte Order Mark (BOM) while reading based on detected charset  :

```java
File inputFile = new File("/Users/fahri/Downloads/UNKNOWN_TEXT.txt");

BOMInputStream bomInputStream = new BOMInputStream(new BufferedInputStream(new FileInputStream(inputFile)),
        ByteOrderMark.UTF_8, ByteOrderMark.UTF_16BE, ByteOrderMark.UTF_16LE, ByteOrderMark.UTF_32BE, ByteOrderMark.UTF_32LE);

System.out.println("HAS BOM : " + bomInputStream.hasBOM());

CharsetDetector detector = new CharsetDetector();
detector.setText(bomInputStream);

CharsetMatch charsetMatch = detector.detect();
System.out.println("CHARSET MATCH : " + charsetMatch.getName());

BufferedReader br = new BufferedReader(new InputStreamReader(bomInputStream, charsetMatch.getName()));
for (String line = br.readLine(); line != null; line = br.readLine()) {
    System.out.println(line);
}
br.close();
```

<!--more-->

## Problem

You got a text file and you have no idea why your application could not process (parse) that file. 

This is a sample of that file:  [UNKNOWN_TEXT.txt](https://github.com/fahrinh/fahrinh.github.io/files/1451863/UNKNOWN_TEXT.txt)

## Analyze The Content

### `cat`
If we inspect the content using `cat` :

![ss-cat-unknown](https://user-images.githubusercontent.com/55460/32521234-57d76492-c445-11e7-855d-57c95197307e.png)

The content looks normal. _But, wait_.  
What is that weird characters at the beginning and end ? (`��` and `%`)

<!--more-->

### `more`

Display the content using another tool, `more` :
![ss-more-unknown](https://user-images.githubusercontent.com/55460/32521773-6b818124-c447-11e7-965e-eea4cfe6ac17.png)

It looks weirder. `<FF><FE>` at the beginning and `^@` between the characters.

### `file`

What `file` command says ?  

```shell
$ file UNKNOWN_TEXT.txt
UNKNOWN_TEXT.txt: Little-endian UTF-16 Unicode text  
```
The text file is a Little-Endian UTF-16 text and our viewer (iTerm2 on OSX, in this case) only supports UTF-8. That is why it looks not normal and your application can not process (parse) further.

### `hexdump`

If we investigate in more detail using `hexdump` :

```shell
$ hexdump UNKNOWN_TEXT.txt
0000000 ff fe 31 00 7c 00 43 00 65 00 6c 00 6c 00 32 00
0000010 31 00 7c 00 31 00 32 00 33 00 34 00 7c 00 32 00
0000020 30 00 31 00 37 00 2f 00 31 00 30 00 2f 00 32 00
0000030 30 00 0a 00 32 00 7c 00 43 00 65 00 6c 00 6c 00
0000040 32 00 32 00 7c 00 35 00 36 00 37 00 38 00 7c 00
0000050 32 00 30 00 31 00 37 00 2f 00 31 00 30 00 2f 00
0000060 32 00 30 00
0000064
```

We can conclude that `<FF><FE>` and `^@` represents value `ff fe` and `00` in hex, respectively.

## Endianness
UTF-16 represents a character using two bytes (2x8 byte=16, make sense, huh?). There are two ways to determine the order of the bytes [^1] : 

- Big Endian : you store the most significant byte in the smallest address
- Little Endian : you store the least significant byte in the smallest address

For the concrete example, these are characters `Z` in various encoding :
```
Character                    : Z
ASCII (in hex)               : 5A
UTF16 Little Endian (in hex) : 5A 00
UTF16 Big Endian (in hex)    : 00 5A
```

## Byte Order Mark (BOM)

You already know the meaning of `00`. Then, what exactly is `ff fe` in the first bytes ?


[^1]: https://www.cs.umd.edu/class/sum2003/cmsc311/Notes/Data/endian.html