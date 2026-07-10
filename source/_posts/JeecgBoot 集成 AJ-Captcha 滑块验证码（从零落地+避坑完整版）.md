---
title: JeecgBoot 集成 AJ-Captcha 滑块验证码（从零落地+避坑完整版）
date: 2026-07-10 16:00:00
tags: 滑块验证码
categories: 
    - AJ-Captcha 滑块验证码
---

## 一、前言

在后台管理系统中，传统**字符图片验证码（Kaptcha）**存在辨识度低、用户体验差、容易被机器识别破解等问题。

目前企业级项目主流方案：**AJ\-Captcha 行为验证码**，支持：滑块拼图、文字点选、旋转校验，自带轨迹防刷、AES加密、Redis分布式缓存，完美适配 SpringBoot、**JeecgBoot** 框架。

本文从原理、原生手写对比、组件讲解、Jeecg 完整整合、常见报错避坑，一站式落地。

<!--more-->

## 二、两种验证码实现方案对比

### 1\. 原生手写滑块（不推荐生产）

早期纯手写 Java 滑块逻辑：自己绘图、裁剪滑块、计算坐标、维护缓存、判断误差。

**缺点：**

- 代码量大、重复造轮子

- 无防机器人轨迹校验，极易被脚本破解

- 无加密、容易抓包篡改坐标

- 单机缓存无法集群部署

### 2\. AJ\-Captcha 开源组件（企业推荐）

目前 Java 生态最成熟的行为验证码框架，JeecgBoot 项目首选替代原生验证码方案。

**优势：**

- 开箱即用，无需手动绘图、裁剪、坐标计算

- 自带鼠标轨迹、滑动速度校验，防爬虫脚本

- 支持 AES 坐标加密，防抓包篡改

- 原生支持 Redis 分布式缓存，适配集群

- 支持滑块、点选、旋转多种验证码类型

## 三、AJ\-Captcha 核心包说明

### 核心包结构

```Plain Text
com.anji.captcha
├─ model
│  ├─ common   # 统一返回结果 ResponseModel / ApiReturn
│  ├─ dto      # 前后端交互入参出参
│  │  ├─ CaptchaReq       # 前端提交校验参数
│  │  └─ GetCaptchaReq    # 获取验证码请求参数
│  ├─ enums    # 验证码类型枚举（滑块/点选/旋转）
│  └─ vo
│     └─ CaptchaVO        # 后端二次校验核心实体
├─ service     # 验证码生成、校验核心业务
└─ annotation  # 启动开启注解 @EnableCaptcha
```

## 四、JeecgBoot 完整集成步骤

### 步骤1：引入 Maven 依赖

在 `jeecg-boot-module-system` 模块 pom\.xml 新增依赖：

```Plain Text
<!-- AJ-Captcha 滑块行为验证码（企业稳定版） -->
<dependency>
    <groupId>com.anji-plus</groupId>
    <artifactId>spring-boot-starter-captcha</artifactId>
    <version>1.3.0</version>
</dependency>
```

Jeecg 自带 Redis 依赖，无需重复引入。

### 步骤2：配置 application\.yml

```Plain Text
# AJ-Captcha 滑块验证码配置
aj:
  captcha:
    cache-type: redis          # 统一使用redis，兼容单机/集群，不推荐local
    type: blockPuzzle          # blockPuzzle滑块拼图 / clickWord文字点选
    slip-offset: 5             # 允许滑动误差像素
    aes-status: true           # 开启AES坐标加密，防抓包篡改
    track-enable: true         # 开启鼠标轨迹校验，防机器脚本
    expire-seconds: 120        # 验证码过期时间
    water-mark: \u667a\u80fd\u7ba1\u7406\u7cfb\u7edf  # Unicode水印：智能管理系统
    jigsaw: classpath:captcha/ # 自定义拼图背景图路径
```

### 步骤3：启动类开启验证码注解

JeecgApplication 启动类添加 `@EnableCaptcha`

```Plain Text
@SpringBootApplication
@EnableCaptcha // 开启AJ验证码自动配置
public class JeecgApplication {
    public static void main(String[] args) {
        SpringApplication.run(JeecgApplication.class, args);
    }
}
```

### 步骤4：改造登录接口（核心）

AJ\-Captcha 标准流程：**前端滑动校验 → 获取加密凭证 → 登录接口二次校验凭证**，禁止直接校验坐标。

```Plain Text
@RestController
@RequestMapping("/sys")
public class LoginController {

    @Resource
    private CaptchaService captchaService;

    @PostMapping("/login")
    public Result<LoginResult> login(
            @RequestParam String username,
            @RequestParam String password,
            @RequestParam String captchaVerification
    ) {
        // 1. 滑块验证码二次校验
        CaptchaVO captchaVO = new CaptchaVO();
        captchaVO.setCaptchaType("blockPuzzle");
        captchaVO.setCaptchaVerification(captchaVerification);
        ResponseModel verifyRes = captchaService.verification(captchaVO);

        // 校验失败直接拦截登录
        if (!verifyRes.isSuccess()) {
            return Result.error("滑块验证失败，请重新滑动");
        }

        // 2. 原有Jeecg登录逻辑不变
        LoginResult loginResult = loginService.login(username, password);
        return Result.success(loginResult);
    }
}
```

## 五、内置默认接口（无需手动开发）

引入 Starter 后框架自动注册两个接口，前端直接调用：

- `POST /captcha/get`：获取滑块验证码图片、唯一ID

- `POST /captcha/check`：前端滑动校验，返回加密凭证 `captchaVerification`

## 六、自定义拓展

### 1\. 自定义滑块背景图

在 `resources` 新建 `captcha` 文件夹，放入多张 PNG 图片，框架自动随机加载。

### 2\. 关闭轨迹校验（简易项目）

```Plain Text
aj:
  captcha:
    track-enable: false
```

## 七、Jeecg 集成常见坑 \& 解决方案

### 1\. 验证码500 / Redis空指针异常

原因：未启动 Redis 或 `cache-type` 配置不对，集群环境**必须使用 redis**，不能用 local。

### 2\. 滑块一直验证失败

- 开启 `aes-status: true` 时，前端必须使用配套组件，不能手写简陋拖动

- 适当调大 `slip-offset` 误差至 8\~10px

- 未传递鼠标轨迹数据

### 3\. /captcha/get 接口404

启动类未添加 `@EnableCaptcha` 注解，导致自动配置失效。

### 4\. 登录验证码校验失败

错误做法：登录传 slideX 原始坐标；
正确做法：前端校验成功后传递 **captchaVerification 加密串** 做二次校验。

## 八、总结

- **选型优势**：AJ\-Captcha 替代老旧 Kaptcha，支持加密防刷、轨迹校验、Redis集群，安全性更强。

- **集成四步核心**：引依赖 → 配yml → 开注解 → 登录接口校验加密凭证。

- **核心规范**：登录只校验 `captchaVerification` 加密串，禁止校验原始坐标。

- **必避坑点**：必须Redis缓存、必须开启@EnableCaptcha、开启AES必须配套前端组件。

- **适用场景**：开箱即用，适配 JeecgBoot 企业级前后端分离项目。

> （注：部分内容可能由 AI 生成）
