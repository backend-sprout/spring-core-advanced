# 리플랙션 
동적 프록시 기법에 대해서 알기 전에 리플랙션이라는 개념부터 알고가자.        
리플렉션은 클래스나 메서드의 메타정보를 동적으로 획득하고, 코드도 동적으로 호출할 수 있다.        
리플랙션을 통해 메서드 정보를 조회하면 메서드 시그니처에대한 적은 의존으로도 메서드를 호출할 수 있다.  

```kt
@Test
fun reflection1() {
    //클래스 정보
    val classHello = Class.forName("hello.proxy.jdkdynamic.ReflectionTest\$Hello")
    val target = Hello()
    
    //callA 메서드 정보
    val methodCallA: Method = classHello.getMethod("callA")
    val result1: Any = methodCallA.invoke(target)
    log.info("result1={}", result1)
    
    //callB 메서드 정보
    val methodCallB: Method = classHello.getMethod("callB")
    val result2: Any = methodCallB.invoke(target)
    log.info("result2={}", result2)
}
```  
* Class.forName(FQCN) : 클래스 메타 정보를 획득한다.         
* Class인스턴스.getMethod("메서드이름") : 특정 클래스의 해당 메서드 메타정보를 획득한다.       
* method인스턴스.invoke(객체) : 메서드 메타정보로 실제 인스턴스의 메서드를 호출한다(인자는 파라미터값이 아닌 해당 객체)   
    
일반적인 인스턴스의 메서드를 호출하는 것과 큰 차이는 없어보이지만       
여기서 중요한 핵심은 클래스나 메서드 정보를 동적으로 변경할 수 있다는 점이다.  

```kt
@Test
fun reflection2() {
    val classHello = Class.forName("hello.proxy.jdkdynamic.ReflectionTest\$Hello")
    val target = Hello()
    val methodCallA: Method = classHello.getMethod("callA")
    dynamicCall(methodCallA, target)
    val methodCallB: Method = classHello.getMethod("callB")
    dynamicCall(methodCallB, target)
}

private fun dynamicCall(method: Method, target: Any) {
    log.info("start")
    val result: Any = method.invoke(target)
    log.info("result={}", result)
}
```
메서드 시그니처가 다르더라도(여기서는 이름) 동일한 구조로 메서드를 호출할 수 있게되었다.    
  
**주의**    
리플렉션을 사용하면 클래스와 메서드의 메타정보를 사용해서      
애플리케이션을 동적으로 유연하게 만들 수 있다.        
하지만 리플렉션 기술은 런타임에 동작하기 때문에, 컴파일 시점에 오류를 잡을 수 없다.     
     
가장 좋은 오류는 개발자가 즉시 확인할 수 있는 컴파일 오류이고,        
가장 무서운 오류는 사용자가 직접 실행할 때 발생하는 런타임 오류다.     
따라서 리플렉션은 일반적으로 사용하면 안된다.    
   
지금까지 프로그래밍 언어가 발달하면서 타입 정보를 기반으로      
컴파일 시점에 오류를 잡아준 덕분에 개발자가 편하게 살았는데,      
리플렉션은 그것에 역행하는 방식이다.    
  
리플렉션은 프레임워크 개발이나      
또는 매우 일반적인 공통 처리가 필요할 때 부분적으로 주의해서 사용해야 한다.   

# JDK 다이나믹 프록시  

JDK 다이나믹 프록시는 자바에서 제공해주는 동적 프록시 기술이다.          
인터페이스 기반의 클래스를 프록시로 만들어주는 역할을 한다.         

## 준비물 

```kt
fun interface AInterface {
    fun call(): String
}
```
```kt
@Slf4j
class AImpl : AInterface {
    override fun call(): String {
        log.info("A 호출")
        return "a"
    }
}
```
인터페이스와 이를 구현한 구현체를 준비한다.   

## 실습 
JDK 다이나믹 프록시를 적용할때, 실제 프록시에 적용할 코드는 `InvocationHandler`에 정의한다.     

```java
package java.lang.reflect;

public interface InvocationHandler {
     public Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
}    
```   
  
**제공되는 파라미터는 다음과 같다.**    
   
* **Object proxy** : 프록시 자신
* **Method method** : 호출한 메서드
* **Object[] args** : 메서드를 호출할 때 전달한 인수

```kt
@Slf4j
class TimeInvocationHandler(private val target: Any) : InvocationHandler {
    override fun invoke(proxy: Any, method: Method, args: Array<Any?>?): Any {
        log.info("TimeProxy 실행")
        val startTime = System.currentTimeMillis()
        
        val result: Any = method.invoke(target, args)
        
        val endTime = System.currentTimeMillis()
        val resultTime = endTime - startTime
        log.info("TimeProxy 종료 resultTime={}", resultTime)
        return result
    }
}
```
* `InvocationHandler`를 구현한 구체 클래스를 통해 프록시에 적용할 코드를 작성할 수 있다.          
* `method.invoke(target, args)`를 호출하는 구문이 실제 타겟 객체를 호출하는 구문이다.            

```kt
@Test
fun dynamicA() {
    val target: AInterface = AImpl()
    val handler = TimeInvocationHandler(target)
    val proxy: AInterface = Proxy.newProxyInstance(
        AInterface::class.java.getClassLoader(), 
        arrayOf<Class<*>>(AInterface::class.java), 
        handler
    ) as AInterface
    proxy.call()
    log.info("targetClass={}", target.getClass())
    log.info("proxyClass={}", proxy.getClass())
}
```
* Proxy.newProxyInstance()를 사용하면 해당 메서드를 통해 프록시 객체를 만들 수 있다.  
    * 클래스 로더를 지정한다.     
    * 클래스에서 사용하는 인터페이스를 기술한다.(다중 기술 가능)    
    * InvocationHandler 를 구현한 핸들러를 입력한다.      
    * 반환형이 Object여서 형변환 구문을 넣어준다.    

```console
TimeInvocationHandler - TimeProxy 실행
AImpl - A 호출
TimeInvocationHandler - TimeProxy 종료 resultTime=0
JdkDynamicProxyTest - targetClass=class hello.proxy.jdkdynamic.code.AImpl
JdkDynamicProxyTest - proxyClass=class com.sun.proxy.$Proxy1
```   
실제 출력 결과를 보면 프록시가 정상 수행된 것을 확인할 수 있다.   

## 실행 순서   
  
![image](https://user-images.githubusercontent.com/50267433/147561169-4f957640-743a-4d43-8118-8c63322066fc.png)

1. 클라이언트는 JDK 동적 프록시의 call() 을 실행한다.      
2. JDK 동적 프록시는 `InvocationHandler.invoke()`를 호출한다.     
    `TimeInvocationHandler`가 구현체로 있으로 `TimeInvocationHandler.invoke()`가 호출된다.      
3. `TimeInvocationHandler`가 내부 로직을 수행하고,     
    `method.invoke(target, args)`를 호출해서 target 인 실제 객체 `AImpl`의 메서드를 호출한다.   
4. AImpl 인스턴스의 call() 이 실행된다.        
5. AImpl 인스턴스의 call() 의 실행이 끝나면 TimeInvocationHandler 로 응답이 돌아온다.     
    시간 로그를 출력하고 결과를 반환한다   
  

