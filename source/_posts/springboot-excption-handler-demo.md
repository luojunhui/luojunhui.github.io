---
title: SpringBoot统一异常拦截 
date: 2019-05-21 17:22:34 
tags:
- Java
- spring
categories:
- Java
---

## 起因:
> 1. 公司某同事把我拉进了一个钉钉群,此群自动把生产环境的Spring项目异常信息通过钉钉机器人发出来.
> 2. 之后每天一看消息,全部都是"未知异常",还每天500+条,一口老血吐出来...
> 3. 长期反馈给同事/上级无果后,打算自己研究下怎么获取具体的异常信息,好让负责这个功能的同事去修改,这样下来总能减少异常报错数量吧...

## 寻求解决之道:
万事开头难,先搜下有没有解决方案吧,发现
[Error Handling for REST with Spring](https://www.baeldung.com/exception-handling-for-rest-with-spring "点击打开该文章")
提出了有4个方案
`@ExceptionHandler`,`HandlerExceptionResolver`,`@ControllerAdvice`,`ResponseStatusException`,结合当前公司项目情况,
选用`@ControllerAdvice`里的`@ExceptionHandler`作为全局异常拦截.


## Coding:
首先建立一个基础的Spring Boot web项目,作为一个合格的CV大师,一顿Ctrl+C
Ctrl+V后,项目已建好. 先写个`controller`,分别定义`get`和`post`接口.
```Java
@RestController
@RequestMapping("/api")
public class ApiController {
    @PostMapping(value = "div")
    public ResultVO div(@RequestBody MyModel obj) {
        DivModel div = new DivModel(obj.getA(), obj.getB(), obj.getA() / obj.getB());
        return new ResultVO("0000", null, div);
    }

    @GetMapping(value = "div1")
    public ResultVO div1(MyModel obj) {
        DivModel div = new DivModel(obj.getA(), obj.getB(), obj.getA() / obj.getB());
        return new ResultVO("0000", null, div);
    }
}
```
再写个拦截器
```Java
@ExceptionHandler({Exception.class})
public ResponseEntity<ResultVO> errorHandler(Exception e) {
    System.out.println("拦截到异常...");
    System.out.println("请求地址:" + request.getRequestURL());
    System.out.println("请求方式:" + request.getMethod());
    String param = request.getQueryString();
    if (!StringUtils.isEmpty(param)) {
        System.out.println("URL参数:" + param);
    }
    String fullStackTrace = ExceptionUtils.getFullStackTrace(e);
    System.out.println("堆栈信息:" + fullStackTrace);
    return new ResponseEntity<ResultVO>(new ResultVO("1111", "异常" + e.getMessage(), null), HttpStatus.INTERNAL_SERVER_ERROR);
}
```
然后运行项目试试两个接口(当被除数为0时会抛出异常).
get:`http://localhost:8080/api/div1?a=10&b=0`
返回
```json
{"code":"1111","msg":"异常/ by zero"}
```
再看控制台输出
```text
拦截到异常...
请求地址:http://localhost:8080/api/div1
请求方式:GET
URL参数:a=10&b=0
堆栈信息:java.lang.ArithmeticException: / by zero
	at cn.junhui.demo.controller.ApiController.div1(ApiController.java:25)
	...
```
好,get能获取到异常信息和参数,再来试试post请求.

返回的json同上,但控制台输出就不一样了,少了参数!!!
```text
拦截到异常...
请求地址:http://localhost:8080/api/div
请求方式:POST
堆栈信息:java.lang.ArithmeticException: / by zero
	at cn.junhui.demo.controller.ApiController.div(ApiController.java:19)
```
由于本公司的项目接口都使用post,所以获取参数就换成`request.getParameterMap();`
```Java
Map<String, String[]> param = request.getParameterMap();
param.forEach((key, value) -> System.out.printf("%s:%s\n", key,value[0]));
```
和get效果一样,但post依然没有拿到参数,在断点调试中可以看到`param`是空的`Map`,因为请求体已经读过一次了,所以在其他地方读取就没有了.

## 解决:
到google一下这个问题,发现[stackoverflow](https://stackoverflow.com/questions/5182417/reading-httprequest-content-from-spring-exception-handler/39921704#39921704 "点击打开")
上有解决方案. 接下来又是CV操作后再来看效果.
```text
拦截到异常...
请求地址:http://localhost:8080/api/div
请求方式:POST
请求参数:{
	"a":10,
	"b":0
}
堆栈信息:java.lang.ArithmeticException: / by zero
	at cn.junhui.demo.controller.ApiController.div(ApiController.java:19)
```
终于能看到参数了.本次任务完成!
最后附上[demo完整代码](https://github.com/luojunhui/springboot-excption-handler-demo.git "点击打开")

