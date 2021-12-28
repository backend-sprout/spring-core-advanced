# JDK 다이나믹 프록시 
    
JDK 다이나믹 프록시는 자바에서 제공해주는 동적 프록시 기술이다.          
인터페이스 기반의 클래스를 프록시로 만들어주는 역할을 한다.         
대신, JDK 다이나믹 프록시를 알기 전에 리플랙션이라는 개념부터 알고가자.   

## 리플랙션 
     
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

