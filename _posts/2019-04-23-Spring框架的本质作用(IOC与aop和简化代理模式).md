---
layout:     post
title:      Spring框架的小谈
subtitle:   Spring源码学习系列
date:       2019-04-23
author:     CodingAndLiving
header-img: img/com01.jpg
catalog: true
tags:
    - Spring框架
---
# 前言

>Spring框架，常说道的知识点是IOC，DI，AOP这几个，那为什么需要使用Spring呢？仅仅是控制反转（IOC），帮我们实例化一个对象？今日的学习，我有了不一样的想法。

# 为什么需要Spring
一开始，并没有特别的想法，现在倒是有了点，首先可能会说到是由于Spring支持IOC，所以需要它，但是，其实我们自己实例化一个对象，这个复杂吗？一点都不复杂，那问题回到，我们为什么需要Spring？
我现在的想法是，本质上还是IOC机制，但是，不仅仅是IOC机制，准确来说，是Spring帮我们管理起来，并且给我们返回对象，但是这个返回，就不简单了，里面可以做功夫。意思就是说Spring相当于一个bean工厂，它帮我们管理这些实例化对象，但是它返回给我们的时候，**可以是单纯的bean对象，也可以是代理对象**，相当于可以在替代我们管理的基础上，做了一层处理操作，然而，无论做什么处理都好，它的地基，就是ioc机制。



# Spring的AOP和代理模式
Spring框架的AOP并不是完全自己实现的，准确来说，应该是在jdk的动态代理和cglib的代理的基础上进行了封装。另外ASPECTJ也是一款优秀的AOP框架，虽然它的原理和Spring的原理，有所区别，*ASPECTJ是在编译期就实现的*。下面简单列举下Spring的AOP编码案例。（这里不包含AspectJ的编码案例）

Spring的AOP有两种织入方式，分别是Advice和Advisor，后者的织入能力更为强大些。我喜欢将后者称之为前者的动态升级版。

#### 基于Advice的编码案例


- 定义了一个接口

```

		
	public interface BaseService {
	       public void eat();//JoinCut 连接点
	       public void wc();//JoinCut 连接点
	}
```

- 接口的实现类

```
	
		
	public class Person implements BaseService {
	
		public void eat() {//切入点 PointCut  主要业务方法
			
	           System.out.println("吃泡面");
		}
	
		public void wc() {//切入点 PointCut   主要业务方法
			
			 System.out.println("上厕所");
		}
	
	}
	
```

- 定义一个Advice

```

		
	import org.springframework.aop.MethodBeforeAdvice;
	import org.springframework.aop.framework.ProxyFactoryBean;
	
	public class MyBeforeAdvice implements MethodBeforeAdvice {
	
		//切面：次要业务
		public void before(Method arg0, Object[] arg1, Object arg2) throws Throwable {
			System.out.println("-----洗手-----");
	        ProxyFactoryBean cc; 
		}
	
	}

```

- 编写xml配置文件

```

	 <!-- 注册被监控实现类 -->
       <bean id="person" class="com.ljh.serviceImpl.Person"></bean>
       
       <!-- 注册通知实现类 -->
       <bean id="before" class="com.ljh.advice.MyBeforeAdvice"></bean>
       
       <!-- 注册代理监控对象生产工厂 -->
       <bean id="personProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
           <property name="target" ref="person"></property>
           <property name="interceptorNames" >
               <array>
                  <value>before</value>
               </array>
           </property>
       </bean>
  
```

来总结下，首先呢advice的步骤里面大概编写了这么几个代码。

- 编写接口 和接口实现类，这一步骤相当于编写业务方法。
- 编写Advice，这一步骤相当于编写次要方法。
- 配置下xml。
对比下了jdk的动态代理，省略了一步骤就是ProxyBuilder那一步骤。其实Spring的AOP也可以理解为在动态代理的基础上做了封装。

#### 基于Advisor的编码案例

Advisor的编码实现，相当于在上面案例的基础上多了一个Advisor的实现类，代码如下：

- 配置文件

```

	  <!-- 注册被监控实现类 -->
       <bean id="person" class="com.ljh.serviceImpl.Person"></bean>
       <bean id="dog" class="com.ljh.serviceImpl.Gog"></bean>
       
       <!-- 注册通知实现类 -->
       <bean id="before" class="com.ljh.advice.MyBeforeAdvice"></bean>
       
       <!-- 注册类型过滤器 -->
       <bean id="classFilter" class="com.ljh.util.MyClassFilter"></bean>
       <!-- 注册方法匹配器 -->
       <bean id="methodMatcher" class="com.ljh.util.MyMethodMatcher"></bean>
       
       <!-- 注册切入点 -->
       <bean id="pointCut" class="com.ljh.util.MyPointCut" >
          <property name="classFilter" ref="classFilter"></property>
          <property name="metodMatcher" ref="methodMatcher"></property>
       </bean>
       
       <!-- 注册顾问 -->
       <bean id="myAdvisor" class="com.ljh.util.MyPointCutAdvisor">
           <property name="advice" ref="before"></property>
           <property name="pointcut" ref="pointCut"></property>
       </bean>
       
       <!-- 注册代理对象工厂 -->
       <!-- 
                             此时生成代理对象，只会负责person.eat方法监控
                             与Advice不同，不会对BaseService所有的方法进行监控                
        -->
       <bean id="personProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
              <property name="target" ref="person"></property>
              <property name="interceptorNames" value="myAdvisor"></property>
       </bean>
```

- Advisor的代码

```
	
		
	import org.aopalliance.aop.Advice;
	import org.springframework.aop.Pointcut;
	import org.springframework.aop.PointcutAdvisor;
	
	public class MyPointCutAdvisor implements PointcutAdvisor {
		//采用依赖注入 set
	    private Advice advice;//次要业务以及次要业务与主要业务执行顺序
	    private Pointcut pointcut;//当前拦截对象和对象调用主要业务方法 person对象.eat()
	    
	    
	    
		public void setAdvice(Advice advice) {
			this.advice = advice;
		}
	
		public void setPointcut(Pointcut pointcut) {
			this.pointcut = pointcut;
		}
	
		public Advice getAdvice() {
			// TODO Auto-generated method stub
			return this.advice;
		}
	
		public boolean isPerInstance() {
			// TODO Auto-generated method stub
			return false;
		}
	
		public Pointcut getPointcut() {
			// TODO Auto-generated method stub
			return this.pointcut;
		}
	
	}





	
	
	import org.springframework.aop.ClassFilter;
	import org.springframework.aop.MethodMatcher;
	import org.springframework.aop.Pointcut;
	
	public class MyPointCut implements Pointcut {
		
		
		/*
		 * InvocationHandler接口
		 *    invoke(){
		 *        if(obj.getClass ！= person.class){
		 *              return
		 *        }
		 *        
		 *        if(!methodObj.getName.equals("eat")){
		 *               return 
		 *        }
		 *        //织入方式:次要业务方法和 Peson.eat()执行顺序
		 *        //前置通知
		 *          wash（）；
		 *          Person.eat()
		 *    }
		 * 
		 * */
		//使用依赖注入
		private ClassFilter classFilter;
		private MethodMatcher metodMatcher;
	
		public void setClassFilter(ClassFilter classFilter) {
			this.classFilter = classFilter;
		}
	
		public void setMetodMatcher(MethodMatcher metodMatcher) {
			this.metodMatcher = metodMatcher;
		}
	
		public ClassFilter getClassFilter() {
			// TODO Auto-generated method stub
			return this.classFilter;
		}
	
		public MethodMatcher getMethodMatcher() {
			// TODO Auto-generated method stub
			return this.metodMatcher;
		}
	
	}




	
	
	public class MyMethodMatcher implements MethodMatcher {
	
		/*
		 *  被监控接口比如（BaseService），没有重载方法
		 *  每一个方法名称都是以唯一
		 *  此时可以采用 static检测方式，只根据方法名称判断
		 * 参数：method: 接口中某一个方法
		 *     targetClass: 接口中一个实现类
		 *     
		 *  业务：只想为Person类中eat方法提供织入   
		 */
		
		public boolean matches(Method method, Class<?> targetClass) {
			
			String methodName = method.getName();
			if("eat".equals(methodName)){
				return true;
			}
			return false;
		}
	
		public boolean isRuntime() {
			// TODO Auto-generated method stub
			return false;
		}
	
		public boolean matches(Method method, Class<?> targetClass, Object... args) {
			// TODO Auto-generated method stub
			return false;
		}
	
	}


	
	
		
	public class MyClassFilter implements ClassFilter {
	
		 /*
		  *  1.一个接口下会有多个实现类
		  *  2.判断当前实现类是不是我们织入方式关心的目标类
		  *  BaseService接口我们现在只想管理Person.
		  *  参数：就是当前被拦截类：可能Person，可能Gog
		  * */
		public boolean matches(Class<?> clazz) {
		    if(clazz == Person.class){
		    	return true;//告诉顾问，当前类是需要我们提供织入服务
		    }
		    //Gog
			return false;
		}
	
	}

```