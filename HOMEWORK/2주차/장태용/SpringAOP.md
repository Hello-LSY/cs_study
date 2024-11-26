## AOP란

AOP란 Aspect Oriented Programming의 약자로 관점 지향 프로그래밍이란 뜻이다.

비즈니스 로직과 공통 로직을 분리하여 개발하는 방법론이다.

## AspectJ 와 Spring AOP

AspectJ는 java에서 AOP를 지원하는 라이브러리이다.

config를 통해 컴파일 타임 위빙, 런타임 위빙 등 다양한 방법으로 AOP를 적용시킬 수 있다.

Spring AOP는 AspectJ의 어노테이션을 그대로 사용하지만, `@EnableAspectJAutoProxy` 를 사용하면 자체적인 Spring AOP를 적용할 수 있다. 스프링은 Proxy를 사용하여 AOP를 적용하는데 대표적인 두 가지 proxy 방식이 있다. 후에 설명할 CGLib Proxy, JDK-Dynamic Proxy이다.

```java
@EnableAspectJAutoProxy // @Aspect 어노테이션 쓰는 객체를 Spring AOP로 적용시킨다.
public class WebConfig implements WebMvcConfigurer {
	//... 기타 구현
}
```

Spring Boot에서는 Auto Config로 `@EnableAspectJAutoProxy` 가 적용되어 있다.

## CGLib Proxy, JDK-Dynamic Proxy

스프링 프레임워크는 언제 두 가지의 Proxy를 사용할까?

`DefaultAopProxyFactory.class` 를 살펴보자.

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
        if (NativeDetector.inNativeImage() || !config.isOptimize() && !config.isProxyTargetClass() && !this.hasNoUserSuppliedProxyInterfaces(config)) {
            return new JdkDynamicAopProxy(config);
        } else {
            Class<?> targetClass = config.getTargetClass();
            if (targetClass == null) {
                throw new AopConfigException("TargetSource cannot determine target class: Either an interface or a target is required for proxy creation.");
            } else {
                return (AopProxy)(!targetClass.isInterface() && !Proxy.isProxyClass(targetClass) && !ClassUtils.isLambdaClass(targetClass) ? new ObjenesisCglibAopProxy(config) : new JdkDynamicAopProxy(config));
            }
        }
    }
```

### JDK-Dynamic Proxy

```java
Object proxy = Proxy.newProxyInstance(ClassLoader, Class<?>[],InvocationHandler)
```

인터페이스를 기준으로 Proxy 객체를 만들고 인터페이스 메소드의 특정 구간에 실행 될 어드바이스를 실행 시킨다.

만약 다른 해당 빈 객체 자체를 의존하고 있지 않고, 빈 객체의 Interface를 의존하고 있다면 JDK Dynamic 프록시로 빈 객체를 찾을 수 있다.

프록시로 구현한 빈 객체가 기존의 빈 객체를 포장하여 2개의 빈을 모두 등록하는 것이 아니라 실제 빈을 프록시가 적용된 빈으로 바꿔치기 때문이다.

문제점은 구현한 클래스를 의존하면, 구현 클래스를 찾을 수 없다는 문제점이 생긴다.

### CGlib Proxy

Code Generation Library Proxy의 약자이며, Enhancer 객체를 통해 proxy를 만들 수 있다.

```java
Enhancer enhancer = new Enhancer();
		enhancer.setSuperclass(MemberService.class);
		enhancer.setCallback(MethodInterceptor);
Object proxy = enhancer.create();
```

CGLib Proxy는 TargetClass를 상속받아 Proxy를 만드는 클래스 기반 프록시이다.

CGLib 프록시를 사용하게 되면 구현 클래스를 의존하더라도 찾을 수 있다.

대신 CGLib 프록시는 클래스를 상속 받으므로 생성자가 2번 호출되며 final 클래스나, final 메소드가 있다면 프록시로 생성이 안된다는 단점이 있다.

최근에는 이러한 단점을 보완하여 Spring Boot는 CGLib 프록시를 채택하고 있다고 한다.
