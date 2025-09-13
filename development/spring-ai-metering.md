---
layout: default
title: Spring AI에서 미터링 구현하기
---

Spring AI에서 매우 간단한 Metring 구현기를 작성해본다. 개인 프로젝트에 내부적으로 spring-ai-autogen이라는 라이브러리를 직접 구축해서 사용하고 있다.

> 참고로 아무도 안보겠지, 해서 대충 적어둔다.

```kt
@Component
@Retention(AnnotationRetention.RUNTIME)
@Target(AnnotationTarget.CLASS)
annotation class Agent(
    ...
    val chatModel: String = "openAiChatModel",
    val fallbackChatModel: String = "",
    val tokenMeter: String = "",
    ...
)
```

이때 tokenMeter를 구현하는데, 인터페이스 TokenMeter를 상속받아 선언된 tokenMeter를 가져온다. 만약 없다면 default는 object인 NoopTokenMeter를 기본적으로 사용한다.

```kt
private var tokenMeter: TokenMeter = NoopTokenMeter
...
this.tokenMeterName = annotation.tokenMeter.ifBlank { null }
...
tokenMeterName?.apply { tokenMeter = applicationContext.getBean(this, TokenMeter::class.java) }
```


```kt
object NoopTokenMeter : TokenMeter {
    override fun record(conversationId: String, tokens: Long) {
        return
    }

    override fun isFull(conversationId: String, criteria: Long): Boolean {
        return false
    }
}
```

여기서 주의할 점은 TokenMeter의 count를 올리기 위해 `chatResponse().metadata` 의 metadata의 totalToken을 그대로 사용하면 advisor 전파에 실패하므로, TokenMeter도 advisor를 사용하도록 한다.

따라서 AutoConfiguration을 하나 만들고, factory로 사용하도록 한다.

```kt
@Bean
    @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
    fun tokenMeteringAdvisor(tokenMeter: TokenMeter): BaseTokenMeteringAdvisor = TokenMeteringAdvisor(tokenMeter)
```

```kt
interface BaseTokenMeteringAdvisor : BaseChatMemoryAdvisor {
    val tokenMeter: TokenMeter

    override fun before(chatClientRequest: ChatClientRequest, advisorChain: AdvisorChain): ChatClientRequest {
        return chatClientRequest
    }

    override fun after(chatClientResponse: ChatClientResponse, advisorChain: AdvisorChain): ChatClientResponse {
        val metadata = chatClientResponse.chatResponse?.metadata ?: return chatClientResponse
        val tokens = metadata.usage.totalTokens.toLong()

        val conversationId = getConversationId(chatClientResponse.context, ChatMemory.DEFAULT_CONVERSATION_ID)

        metering(conversationId = conversationId, tokens = tokens)

        return chatClientResponse
    }

    override fun getOrder(): Int = Ordered.HIGHEST_PRECEDENCE

    private fun metering(conversationId: String, tokens: Long) {
        tokenMeter.record(conversationId, tokens)
    }
}

class TokenMeteringAdvisor(override val tokenMeter: TokenMeter) : BaseTokenMeteringAdvisor
```

이제 실제 애플리케이션에서는 아래처럼 적용해서 쓰면 된다.

```kt
@Agent(
    ...
    chatModel = "gpt41Model",
    fallbackChatModel = "gpt41MiniModel",
    tokenMeter = "userTokenMeter",
    advisors = ["chatMemoryAdvisor"]
)
class OpenAIChatAgent : AgentService()
```

이 밖에도 ChatMemoryAdvisor를 MultiLayedChatMemory 형태로 L1 캐시 -> L2 캐시 -> 퍼시스턴트 레이어 형태로 구현한 썰도 있지만 다음 기회에 여유가 되면 적어보자.