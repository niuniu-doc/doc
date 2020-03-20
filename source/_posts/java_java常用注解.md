---
title: java常用注解
date: 2020-03-20
categories:
  - Java
tags:
  - Java
---
`@Controller` mvc中、声明一个控制器
`@Component` 声明一个通用组件
`@Repository` 声明一个dao组件
`@Service` 声明一个service组件

`@Bean` 声明一个bean容器
`@Configuration` 声明配置类
`@ComponentScan` 包扫描

`@Async`声明一个异步方法
`@EnableAsync` 开启异步任务支持
`@Scheduled` 声明一个计划任务
       `fixdRate` 表明每隔固定时间间隔执行
      `cron`表明按照cron表达式在指定时间执行
`@EnableScheduling` 开启计划任务支持
`@EnableCaching` 开启注解式的缓存支持

`@RequestMapping` 配置url和方法之间的映射关系                                                    
`@Conditional` 条件注解
`@ResponseBody` 将返回值放在response返回体内、而不是返回一个页面
`@RequestBody`允许将request参数放在request体中、而不是放在连接地址后边
`@PathVariable`用来接收路径参数



组合注解
`@WiselyConfiguration` 代替 `@Configuration` + `@ComponentScan`
`@RestController` 代替 `@Controller` + `@ResponseBody`
`@SpringBootApplication` 组合了 
`@Configuration`+`@EnableAutoConfiguration`+`@ComponentScan`

`@ConditionalOnBean`当容器里有指定的bean的条件下
`@ConditionalOnClass` 当类路径下有指定的类的条件下
`@ConditionalOnExpression`基于SpEL做判断
`@ConditionalOnJava` 基于jvm版本做判断
`@ConditionalOnMissingBean` 当容器中无指定bean的条件下
`@ConditionalOnMissingClass` 类路径下无指定class的情况下
`@ConditionalOnNotWebApplication` 当前项目不是web项目的情况下
`@ConditionalOnWebApplication` 当前项目是web项目的情况下
`@ConditionalOnProperty` 指定的属性是否有指定值
`@ConditionalOnResource` 类路径是否有指定值
