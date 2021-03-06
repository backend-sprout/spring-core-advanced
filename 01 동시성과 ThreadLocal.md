# 요구사항 

* 모든 PUBLIC 메서드의 호출과 응답 정보를 로그로 출력 
* 애플리케이션의 흐름을 변경하면 안됨
    * 로그를 남긴다고 해서 비즈니스 로직의 동작에 영향을 주면 안됨 메서드 호출에 걸린 시간
* 정상 흐름과 예외 흐름 구분
    * 예외 발생시 예외 정보가 남아야 함
* 메서드 호출의 깊이 표현 
* HTTP 요청을 구분
    * HTTP 요청 단위로 특정 ID를 남겨서 어떤 HTTP 요청에서 시작된 것인지 명확하게 구분이 가능해야 함
    * 트랜잭션 ID (DB 트랜잭션X), 여기서는 하나의 HTTP 요청이 시작해서 끝날 때 까지를 하나의 트랜잭션이라 함

## 준비물 

**LogTrace**
```kt
interface LogTrace {

    fun begin(message: String): TraceStatus

    fun end(traceStatus: TraceStatus)

    fun exception(traceStatus: TraceStatus, e: Exception)
}
```

# 동시성과 ThreadLocal    
  
**OrderController**
```kt
@RestController
class OrderControllerV3(
    private val orderService: OrderServiceV3,
    private val logTrace: LogTrace
) {
    ... 생략 
}
``` 

**FieldLogTrace**
```kt
@Component 
class FieldLogTrace : LogTrace {
    private var traceIdHolder: TraceId? = null
    private val log: Logger = LoggerFactory.getLogger(javaClass)

    override fun begin(message: String): TraceStatus {
        syncTraceId()
        val traceId = traceIdHolder!!
        val startTimeMs = System.currentTimeMillis()
        log.info("[{}] {}{}", traceId.id, addSpace(START_PREFIX, traceId.level), message)
        return TraceStatus(traceId, startTimeMs, message)
    }

    private fun syncTraceId() {
        if (traceIdHolder == null) {
            traceIdHolder = TraceId()
        } else {
            traceIdHolder = traceIdHolder!!.createNextId()
        }
    }

    override fun end(traceStatus: TraceStatus): Unit = complete(traceStatus, null)

    override fun exception(traceStatus: TraceStatus, e: Exception): Unit = complete(traceStatus, e)

    private fun complete(status: TraceStatus, e: java.lang.Exception?) {
        val stopTimeMs = System.currentTimeMillis()
        val resultTimeMs = stopTimeMs - status.startTimeMs
        val traceId = status.traceId
        if (e == null) {
            log.info(
                "[{}] {}{} time={}ms", traceId.id,
                addSpace(COMPLETE_PREFIX, traceId.level), status.message,
                resultTimeMs
            )
        } else {
            log.info(
                "[{}] {}{} time={}ms ex={}", traceId.id,
                addSpace(EX_PREFIX, traceId.level), status.message, resultTimeMs,
                e.toString()
            )
        }
        releaseTraceId()
    }

    private fun releaseTraceId() {
        if (traceIdHolder!!.isFirstLevel()) {
            traceIdHolder = null
        } else {
            traceIdHolder = traceIdHolder!!.createPreviousId()
        }
    }

    companion object {

        private const val START_PREFIX = "-->"
        private const val COMPLETE_PREFIX = "<--"
        private const val EX_PREFIX = "<X-"

        private fun addSpace(prefix: String, level: Int): String {
            val sb = StringBuilder()
            for (i in (0 until level)) {
                sb.append(if (i == level - 1) "|$prefix" else "|   ")
            }
            return sb.toString()
        }
    }
}
```

요구사항을 지키기 위해 `FieldLogTrace` 클래스를 작성했다.                    
하나의 Request 가 들어오면, traceIdHolder(TraceId)를 생성하고              
각각의 레이어를 넘나들면서 그 레벨을 올리고 줄이고를 하다가 요청이 끝나면 traceIdHolder를 null로 만든다.     

```console
[f8477cfc] OrderController.request()
[f8477cfc] |-->OrderService.orderItem()
[f8477cfc] |   |-->OrderRepository.save()
[f8477cfc] |   |<--OrderRepository.save() time=1004ms
[f8477cfc] |<--OrderService.orderItem() time=1006ms
[f8477cfc] OrderController.request() time=1007ms
```
  
실제로 잘 동작하고 겉보기에는 큰 문제가 없는 것 같지만,   
바로 동시성 문제가 발생하여, 시스템에 매우 큰 악영향을 끼칠 수 있다.                 
         
스프링의 기본 빈 생성 전략은 싱글톤이다.             
그렇기에 `FieldLogTrace`는 스프링 빈으로 등록되면서 싱글톤 패턴으로 생성될 것이다.              
하지만, 그 내부에 존재하는 `TraceId`는 싱글톤이 아니지만 싱글톤과 같이 동작한다.    

```console
[nio-8080-exec-3] [aaaaaaaa] OrderController.request()
[nio-8080-exec-3] [aaaaaaaa] |-->OrderService.orderItem()
[nio-8080-exec-3] [aaaaaaaa] |   |-->OrderRepository.save()
[nio-8080-exec-4] [aaaaaaaa] |   |   |-->OrderController.request()
[nio-8080-exec-4] [aaaaaaaa] |   |   |   |-->OrderService.orderItem()
[nio-8080-exec-4] [aaaaaaaa] |   |   |   |   |-->OrderRepository.save()
[nio-8080-exec-3] [aaaaaaaa] |   |<--OrderRepository.save() time=1005ms
[nio-8080-exec-3] [aaaaaaaa] |<--OrderService.orderItem() time=1005ms
[nio-8080-exec-3] [aaaaaaaa] OrderController.request() time=1005ms
[nio-8080-exec-4] [aaaaaaaa] |   |   |   |   |<--OrderRepository.save() time=1005ms
[nio-8080-exec-4] [aaaaaaaa] |   |   |   |<--OrderService.orderItem() time=1005ms
[nio-8080-exec-4] [aaaaaaaa] |   |   |<--OrderController.request() time=1005ms
```

Heap 영역에 저장되는 객체들은 각각의 쓰레드들에서 데이터를 공유한다.                 
동시에 여러 요청이 들어오면, 처리할 수 있을 만큼의 쓰레드들이 이를 담당하여 처리한다.               
즉, A쓰레드가 `FieldLogTrace`를 사용하다가 B쓰레드도 `FieldLogTrace`를 사용한다는 것이고       
아직, 사용되고 있는(널이 아닌) traceIdHolder 객체를 B쓰레드가 접근하여 시스템에 문제를 준다.    
  
## 해결   
우선, 모든 상황에서 완벽한 해결방법은 아니라고 말하고 싶다.        
현재 요구사항을 보면, 각 요청마다의 TrasactionId 가 일치해야한다고 한다.         
이는 곧 각각의 요청(쓰레드)에 대한 데이터를 관리하면 된다는 뜻이기도하다.        
 
```kt
class ThreadLocalLogTrace(private val threadLocal: ThreadLocal<TraceId> = ThreadLocal()) : LogTrace {

    private val log: Logger = LoggerFactory.getLogger(javaClass)

    override fun begin(message: String): TraceStatus {
        syncTraceId()
        val traceId = threadLocal.get()
        val startTimeMs = System.currentTimeMillis()
        log.info("[{}] {}{}", traceId.id, addSpace(START_PREFIX, traceId.level), message)
        return TraceStatus(traceId, startTimeMs, message)
    }

    private fun syncTraceId() {
        val thraceId = threadLocal.get()
        if (thraceId == null) {
            threadLocal.set(TraceId())
        } else {
            threadLocal.set(thraceId.createNextId())
        }
    }

    override fun end(traceStatus: TraceStatus): Unit = complete(traceStatus, null)

    override fun exception(traceStatus: TraceStatus, e: Exception): Unit = complete(traceStatus, e)

    private fun complete(status: TraceStatus, e: java.lang.Exception?) {
        val stopTimeMs = System.currentTimeMillis()
        val resultTimeMs = stopTimeMs - status.startTimeMs
        val traceId = status.traceId
        if (e == null) {
            log.info(
                "[{}] {}{} time={}ms", traceId.id,
                addSpace(COMPLETE_PREFIX, traceId.level), status.message,
                resultTimeMs
            )
        } else {
            log.info(
                "[{}] {}{} time={}ms ex={}", traceId.id,
                addSpace(EX_PREFIX, traceId.level), status.message, resultTimeMs,
                e.toString()
            )
        }
        releaseTraceId()
    }

    private fun releaseTraceId() {
        val traceId = threadLocal.get()
        if (traceId.isFirstLevel()) {
            threadLocal.remove()
        } else {
            threadLocal.set(traceId.createPreviousId())
        }
    }

    companion object {

        private const val START_PREFIX = "-->"
        private const val COMPLETE_PREFIX = "<--"
        private const val EX_PREFIX = "<X-"

        private fun addSpace(prefix: String, level: Int): String {
            val sb = StringBuilder()
            for (i in (0 until level)) {
                sb.append(if (i == level - 1) "|$prefix" else "|   ")
            }
            return sb.toString()
        }
    }
}
```
 
`java.lang` 에는 `ThreadLocal`라는 클래스가 존재한다.       
`ThreadLocal` 은 각각의 쓰레드마다 별도의 저장 공간을 제공해주는 역할을 한다.    
이를 활용하여, 각각의 쓰레드 저장공간이 비어있다면 thraceId 를 생성하고      
모든 작업이 끝나면 쓰레드의 저장공간을 비우는 작업을 진행하면 된다.     
그리고 사용된 쓰레드는 다시 쓰레드풀로 대기하므로 값을 비워주는 작업을 잊지말자  
      
앞서 말했듯이, 이는 각각의 쓰레드마다 다른 값을 사용하는 경우에만 적용되는 케이스다.               
스프링 프로젝트를 진행할때는, 싱글톤내의 객체 필드는 동시성 문제가 발생할 수 있음을 유념하자      
