---
layout: post
title:  "spring aop记录请求参数"
date:   2019-02-25
author: Dickie Yang 
tags: 
    - Java
    - springboot
    - aop
    - 日志 
---
## execution表达式
> execution(modifiers-pattern? ret-type-pattern declaring-type-pattern?name-pattern(param-pattern) 
throws-pattern?)  

> execution(public * com.xx.oo..*.*(..))

- execution()：表达式主体
- 第一个"*"符号：表示返回值的类型任意
- com.xx.oo:需要横切的业务类所在的包
- 包名后面"..":表示当前包及子包,一个点表示当前包
- 第二个"*":表示类名,*表示所有类
- .*(..):表示任何方法名，括号表示参数，两个点表示任何参数类型

## Aop记录请求参数    
```   
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.util.StringUtils;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import javax.servlet.ServletInputStream;
import javax.servlet.http.HttpServletRequest;
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.util.Map;
import java.util.Objects;

public class LogAop {

    private static final Logger log = LoggerFactory.getLogger(LogAop.class);

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Around("execution (public * com.xx.oo.web..*.*(..))")
    private Object around(ProceedingJoinPoint pjp) throws Throwable {
        StringBuilder sb = new StringBuilder();
        sb.append("Method Args:");
        ServletRequestAttributes sra = (ServletRequestAttributes) RequestContextHolder.currentRequestAttributes();
        if (Objects.nonNull(sra)){
            sb.append("[");
            HttpServletRequest request = sra.getRequest();
            Map<String, String[]> paramMap = request.getParameterMap();
            paramMap.forEach((k,v) -> sb.append(k).append("=").append(String.join(",",v)).append("&"));
            if (!paramMap.isEmpty()) sb.delete(sb.length() - 1,sb.length());
            try {
                ServletInputStream is = request.getInputStream();
                if (Objects.nonNull(is)){
                    BufferedReader br = new BufferedReader(new InputStreamReader(is));
                    StringBuilder body = new StringBuilder();
                    for (String line;(line = br.readLine()) != null;)
                        body.append(line);
                    br.close();
                    if (body.length() > 0) {
                        if (!paramMap.isEmpty()) sb.append("&");
                        sb.append("body=").append(body);
                    }
                }
            } catch (Exception ex){
				//由于在定义@RequestBody注解的方法中spring框架已经消费掉输入流，
                //在调用read()时会抛出异常,此条件下应取spring处理之后的参数对象。
                String args = writeObjAsString(pjp.getArgs());
                if (!StringUtils.isEmpty(args)){
                    if (!paramMap.isEmpty()) sb.append("&");
                    sb.append("body=").append(args);
                }
            }
            sb.append("]");
        }
        log.info(sb.toString());
        Object proceed = pjp.proceed();
        String resp = "\nResponse:[".concat(writeObjAsString(proceed)).concat("]");
        log.info(resp);
        return proceed;
    }

    private String writeObjAsString(Object obj){
        try {
            return MAPPER.writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            return "";
        }
    }
}
```