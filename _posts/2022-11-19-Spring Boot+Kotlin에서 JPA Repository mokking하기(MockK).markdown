---
layout: post
title:  "Spring Boot + Kotlin에서 JPA Repository mokking하기(MockK)"
future: true
date:   2022-12-12 00:00:00 +0900
categories: Spring MockK Kotlin
---
Spring Boot + Kotlin 조합으로 단뒤 테스트 코드를 짜다 보면 repository를 어떻게 주입해줘야할지 고민이 된다. 이럴때 주로 가짜로 구현체를 만들어서 주입해주게 되는데 테스트할 클래스가 `JPARepository`를 바라보고 있는 경우에는 가짜 구현체에 구현해줄게 너무 많다. 그래서 여러가지 방법을 찾다가 moking 라이브러리를 사용하기로 하였는데 서치중 mockK를 어떻게 사용했는지 기록하고자 한다.

# MockK

MockK는 Kotlin에서 Mocking해주는 라이브러리이다. baeldung에서 이렇게 소개하고 있다.
>  kotlin은 모든 class와 method들이 기본으로 `final`이므로 `open` 키워드를 붙여주지 않으면 상속되지 않는데, 이러한 특성때문에 대부분의 mocking라이브러리에서 mocking에 문제가 생길 수 있다. 그렇다고 테스트를 위해 `open`키워드를 붙이는건 좋지 않는 방법이다.
mockK는 이에 대응하여 코틀린에서 유용하게 사용할 수 있는 mocking라이브러리이다. mocked class에 대한 프록시를 구축한다.

## 시작하기
- 공식 홈페이지에서 어떻게 설치하는지 확인할 수 있다. `build.gradle.kts`에 추가해주자.
    ```gradle
    testImplementation("io.mockk:mockk:${mockkVersion}")
    ```

## 예시
Memo라는 Entity가 있다고 가정하자. MemoRepository는 JPA 를 사용하고, MemoApplication은 MemoRepository와 SequenceGenerator(인터페이스)를 주입받아서 Memo를 만든다.
- Memo.kt
    ```kotlin
    @Entity
    @Table(name = "Memo")
    class Memo(
        @EmbeddedId
        @AttributeOverride(name = "value", column = Column(name = "id", nullable = false))
        val id: MemoId,

        @Embedded
        @AttributeOverride(name = "value", column = Column(name = "userId", nullable = false))
        val userId: UserId,

        @Embedded
        @AttributeOverride(name = "value", column = Column(name = "directoryId", nullable = true))
        var directoryId: DirectoryId? = null,

        param: MemoParam
    ) {
        @Column(length = 128, nullable = false)
        var title: String = param.title

        @Column(length = 2048, nullable = false)
        var content: String = param.content

        @Column(nullable = false)
        var status: CommonState = CommonState.ACTIVE


        fun update(param: MemoParam) {
            title = param.title
            content = param.content
        }

        fun updateDirectoryId(newDirectoryId: DirectoryId?) {
            directoryId = newDirectoryId
        }

        fun delete() {
            status = CommonState.DELETED
        }
    }
    ```
- MemoRepostiroy.kt
    ```kotlin
    @Repository
    interface MemoRepository: JpaRepository<Memo, MemoId>, JpaSpecificationExecutor<Memo>
     ```
- MemoApplication.kt
    ```kotlin
    @Service
    @Transactional
    class MemoApplication(
        private val sequenceGenerator: SequenceGenerator,
        private val repository: MemoRepository,
        private val directoryRepository: DirectoryRepository
    ) {
        fun createMemo(param: MemoParam, userId: String): Memo {
            val memoId = MemoId(sequenceGenerator.generate(Memo::class.java.simpleName))
            val memo = Memo(id = memoId, param = param, userId = UserId(value = userId.toLong()))
            repository.save(memo)
            return memo;
        }
    }
    ```
MemoApplication을 테스트 하기 위해서 필요한것은 sequenceGenerator와 repository, directoryRepository 3가지 이다. sequenceGenerator와 repository들의 메소드를 테스트하는것이 아닌, Memo가 create되는지 테스트 하고 싶으므로 `equenceGenerator.generate(Memo::class.java.simpleName)`와 `repository.save(memo)`는 동작한다고 가정하고 싶다. 이럴때 mockk 라이브러리를 사용하면 된다.
```kotlin
@ExtendWith(MockKExtension::class)
class MemoApplicationTest {
    private val memoRepository = mockk<MemoRepository>()
    private val sequenceGenerator = mockk<SequenceGenerator>()
    private val directoryRepository = mockk<DirectoryRepository>()
    private val memoApplication = MemoApplication(repository = memoRepository, sequenceGenerator = sequenceGenerator, directoryRepository = directoryRepository)

    @Test
    fun createMemoTest() {
        every {
            sequenceGenerator.generate(any())
        } returns 1

        every {
            memoRepository.save(any())
        } returnsArgument (0)

        val testParam = MemoParam(
            title = "테스트 메모1",
            content = "테스트 메모",
        )

        val result = memoApplication.createMemo(testParam, userId = "1")

        // memoRepository.save가 호출되었는지 체크한다.
        verify { memoRepository.save(any()) }

        assertThat(result.id.value).isEqualTo(1)
        assertThat(result.title).isEqualTo(testParam.title)
    }
}
```
- 테스트 클래스에서 간단하게 `mockk<T>()`로 mock 클래스를 만들 수 있다.
    ```kotlin
    private val memoRepository = mockk<MemoRepository>()
    ```
- `every` 키워드로 클래스의 메소드를 stub한다.
    - 예를 들어`repository.save`는 아래와 같이 인자와 결과값을 임의로 지정하면된다. 인자로  아무런 값이나(`any()`) 사용하고, 첫번째 인자를 return 하겠다는 뜻이다.
        ```kotlin
        every {
            memoRepository.save(any())
        } returnsArgument (0)
        ```
    
-  `verify` 키워로 mocking된 클래스의 메소드가 호출되었는지 확인할 수 있다.(행위 검증)
    ```kotlin
    verify { memoRepository.save(any()) }
    ```
    - `verify`는 정확히 몇 번 호출됐는지 체크하는 옵션이 있다.
        ```kotlin
        verify(exactly = 1) { memoRepository.save(any()) } // Success
        verify(exactly = 3) { memoRepository.save(any()) } // Error
        ```

### 에러 테스트하기
`every`키워드의 answer로 에러도 throw할 수 있다.
요건 좀 있다 작성하기 ~

## 참고
- https://notwoods.github.io/mockk-guidebook/docs/mocking/stubbing/





