---
layout: post
title:  "Java实现Ip与Int的相互转换"
date:   2019-03-10
author: Dickie Yang 
tags: 
    - Java
    - 常识
---

## IP转Int代码

```
    public void ip2int(){
        String ip = "1.255.255.25";
        String[] split = ip.split("\\.");
        int result = 0;
        for (int i = 0; i < split.length; i++) {
            result = result | (Integer.parseInt(split[i]) << (8*i));
        }
        System.out.println(result);
    }
```

## Int转IP代码

```
	public void int2ip(int ip){
        String[] res = new String[4];
        for (int i = 0; i < 4; i++) {
            res[i] = String.valueOf((ip & (255 << (i * 8))) >>> (i * 8));
        }
        System.out.println(String.join(".",res));
    }
```

> 此处不用关心int类型首位为符号位的情况，只是需要用到首位的空间，对是否是符号位无所谓。