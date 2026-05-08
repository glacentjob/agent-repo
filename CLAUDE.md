# CLAUDE.md / AGENTS.md — 프로젝트 아키텍처 규칙

이 문서는 Claude Code와 OpenAI Codex에 동일하게 적용한다.

- Claude Code용 파일명: `CLAUDE.md`
- Codex용 파일명: `AGENTS.md`
- 두 파일의 핵심 규칙은 항상 동일하게 유지한다.
- 한쪽 파일만 수정하는 PR은 허용하지 않는다.
- 하위 디렉터리에 별도 `AGENTS.md` 또는 `CLAUDE.md`를 둘 경우, 이 문서의 상위 규칙을 약화시킬 수 없다.

---

## 1. 환경

- Kotlin: 2.3.21
- AGP: 9.2.0
- Gradle: 9.4.1+
- JDK: 17
- iOS 배포 타겟: 17.0+
- macOS: M 시리즈
- 빌드 서버: Mac Mini M1
- CI: GitLab CI (`gitlab.aiconnecx.com`)

### 1.1 빌드 환경 고정 규칙

- Gradle Wrapper 버전은 저장소에 고정한다.
- CI JDK는 17로 고정한다.
- Xcode 버전은 CI 문서와 Runner 설정에 명시한다.
- Android SDK Build Tools, NDK, Xcode, Fastlane 버전 변경은 PR 설명에 명시한다.
- Kotlin / AGP / Gradle 중 하나라도 변경하면 `shared`, `androidApp`, `feature/*`, `iosApp` 빌드 검증을 모두 수행한다.

---

## 2. 관리 파일 운영 규칙

루트 디렉터리에 다음 두 파일을 둔다.

```text
CLAUDE.md
AGENTS.md
```

두 파일은 동일한 내용을 유지한다.

```bash
cp CLAUDE.md AGENTS.md
```

### 2.1 변경 규칙

- `CLAUDE.md`를 수정하면 같은 PR에서 `AGENTS.md`도 동일하게 수정한다.
- `AGENTS.md`를 수정하면 같은 PR에서 `CLAUDE.md`도 동일하게 수정한다.
- Agent 규칙 변경 PR에는 변경 사유를 반드시 적는다.
- KMP 경계, SKIE, DI, Gradle 모듈 구조, CI 구조 변경은 ADR 대상이다.

---

## 3. 프로젝트 모듈 구조

본 프로젝트는 AGP 9.x 기준으로 KMP shared 모듈과 Android application entry point 모듈을 반드시 분리한다.

```text
project/
├── shared/                         ← KMP 계약 레이어
│   ├── src/commonMain/kotlin/
│   │   ├── model/                  ← DTO, data class, sealed Error/Result
│   │   └── repository/             ← Repository interface
│   ├── src/commonTest/kotlin/
│   ├── src/androidMain/kotlin/
│   ├── src/iosMain/kotlin/
│   └── build.gradle.kts            ← org.jetbrains.kotlin.multiplatform + com.android.kotlin.multiplatform.library + SKIE
│
├── androidApp/                     ← Android 앱 진입점
│   ├── src/main/kotlin/            ← Application, MainActivity, navigation root
│   └── build.gradle.kts            ← com.android.application
│
├── feature/                        ← Android 전용 feature Gradle 모듈 집합
│   ├── login/
│   │   ├── src/main/kotlin/        ← LoginScreen, LoginViewModel, Android DI binding
│   │   ├── src/test/kotlin/
│   │   └── build.gradle.kts        ← com.android.library
│   └── home/
│       └── build.gradle.kts        ← com.android.library
│
├── iosApp/                         ← Xcode / SwiftUI 앱
│   ├── Adapter/                    ← shared import 허용 구역, KMP → Swift 매핑
│   ├── Repository/                 ← Swift-native Repository protocol/implementation
│   ├── ViewModel/                  ← @Observable ViewModel, KMP import 금지
│   └── View/                       ← SwiftUI View, KMP import 금지
│
├── fastlane/
├── .gitlab-ci.yml
├── build.gradle.kts
├── settings.gradle.kts
├── CLAUDE.md
└── AGENTS.md
```

### 3.1 `shared` 모듈

- `shared`는 KMP 계약 레이어 모듈이다.
- `shared`에는 `org.jetbrains.kotlin.multiplatform`을 적용한다.
- `shared`의 Android target은 `com.android.kotlin.multiplatform.library`를 사용한다.
- `shared`에 `com.android.application`을 적용하지 않는다.
- `shared`에 Android 앱 진입점, Activity, Application class를 두지 않는다.
- `shared`에는 Repository 구현체를 두지 않는다.

### 3.2 `androidApp` 모듈

- `androidApp`은 Android application entry point 모듈이다.
- `androidApp`에는 `com.android.application`을 적용한다.
- `androidApp`에 `org.jetbrains.kotlin.multiplatform`을 적용하지 않는다.
- `androidApp`은 `implementation(project(":shared"))`로 shared 모듈에 의존한다.
- Android navigation root, Application class, MainActivity는 `androidApp`에 둔다.

### 3.3 `feature/*` 모듈

`feature/*`는 Android 전용 Gradle library 모듈 집합이다.

- 각 feature 모듈에는 `com.android.library`를 적용한다.
- 각 feature 모듈에는 `org.jetbrains.kotlin.multiplatform`을 적용하지 않는다.
- 각 feature 모듈은 `implementation(project(":shared"))`로 shared 모듈에 의존한다.
- ViewModel, Screen, Android DI binding은 해당 feature 모듈에 둔다.
- `settings.gradle.kts`에 모든 feature 모듈을 명시한다.
- feature 모듈에서 iOS 코드를 생성하지 않는다.

예시:

```kotlin
include(":shared")
include(":androidApp")
include(":feature:login")
include(":feature:home")
```

### 3.4 `iosApp`

- `iosApp`은 SwiftUI 앱 진입점이다.
- SwiftUI View와 iOS ViewModel은 KMP 타입을 직접 import하지 않는다.
- `import shared`는 `iosApp/Adapter` 또는 명시적으로 허용된 mapping 파일에서만 허용한다.
- iOS ViewModel은 Swift-native Repository protocol에만 의존한다.

---

## 4. KMP 경계 규칙

`shared/commonMain`은 플랫폼 간 계약 계층으로 사용한다.

### 4.1 허용

`shared/commonMain`에는 다음만 허용한다.

- interface
- data class
- sealed class
- sealed interface
- enum class
- Error / Result 계층
- Request / Response DTO
- side-effect 없는 순수 타입 정의
- 플랫폼 API를 호출하지 않는 순수 mapper 또는 validator

### 4.2 금지

`shared/commonMain`에는 다음을 금지한다.

- Repository 구현체
- UseCase 구현체
- ViewModel 구현체
- 네트워크 구현
- DB 구현
- 파일 시스템 구현
- Keychain / SharedPreferences 구현
- Android/iOS 플랫폼 API 직접 참조
- DI framework 의존
- Logger 구현체
- UI framework 의존
- `Flow`, `StateFlow`, `SharedFlow`를 public interface에 노출

### 4.3 State/Event/Effect 정책

- UI State / Event / Effect는 기본적으로 Android와 iOS에서 각각 정의한다.
- 양 플랫폼 타입의 이름, 필드명, default value, 의미는 동일하게 유지한다.
- 공통 계약용 State/Event/Effect가 필요한 경우 `shared/commonMain`에 둘 수 있지만, SwiftUI View와 iOS ViewModel은 해당 KMP 타입을 직접 노출하지 않는다.

### 4.4 `expect/actual` 정책

- `expect/actual`은 기본 금지한다.
- 플랫폼 bridge가 불가피한 경우 ADR 승인 후 허용한다.
- `expect/actual` 사용 시 Android actual, iOS actual, 양 플랫폼 테스트를 같은 PR에 포함한다.
- 라이브러리 또는 빌드 도구가 생성하는 코드는 예외로 둘 수 있으나, 수동 작성은 ADR 대상이다.

---

## 5. Repository Interface 규칙

### 5.1 기본 원칙

- Repository interface는 `shared/commonMain/repository`에 둔다.
- Repository interface 추가/변경 시 Android 구현, iOS Swift protocol/implementation 또는 Adapter, 양 플랫폼 테스트를 같은 PR에 포함한다.
- KMP interface는 one-shot API 중심으로 설계한다.
- `suspend fun`은 SKIE 적용을 전제로 허용한다.
- SKIE 미적용 상태에서는 `suspend fun` 추가를 금지한다.
- `Flow`, `StateFlow`, `SharedFlow` 반환은 금지한다.
- Kotlin `Result<T>` 직접 노출은 금지한다.
- 실패는 sealed Error 또는 `XxxResult` sealed class로 표현한다.

### 5.2 KMP Repository 예시

```kotlin
interface LoginRepository {
    suspend fun login(request: LoginRequest): LoginResult
}

data class LoginRequest(
    val id: String,
    val password: String,
)

sealed class LoginResult {
    data class Success(val user: UserDto) : LoginResult()
    data class Failure(val error: LoginError) : LoginResult()
}

sealed class LoginError {
    data object InvalidCredential : LoginError()
    data object Network : LoginError()
    data class Unknown(val message: String?) : LoginError()
}
```

---

## 6. iOS Bridge 규칙: SKIE

본 프로젝트는 SKIE를 표준 iOS bridge로 사용한다.

### 6.1 SKIE 적용 목적

SKIE는 다음 목적에 한정해 사용한다.

- `suspend fun`을 Swift `async` 호출 형태로 안정화
- Kotlin sealed class / sealed interface를 Swift에서 exhaustive하게 분기할 수 있도록 지원
- Kotlin enum class를 SKIE Transparent Enums 기준으로 Swift exhaustive enum처럼 사용할 수 있도록 지원
- Agent가 iOS bridge 코드를 임의 방식으로 생성하지 않도록 표준화

### 6.2 SKIE 사용 금지 범위

- `Flow`, `StateFlow`, `SharedFlow`의 AsyncSequence 변환은 사용하지 않는다.
- SKIE를 Flow 공유 허가로 해석하지 않는다.
- SKIE default argument 생성 기능은 사용하지 않는다.
- SKIE annotation으로 FlowInterop을 개별 활성화하는 것도 금지한다.

### 6.3 SKIE 설정 예시

`shared/build.gradle.kts`에는 다음 정책이 드러나야 한다.

```kotlin
import co.touchlab.skie.configuration.DefaultArgumentInterop
import co.touchlab.skie.configuration.EnumInterop
import co.touchlab.skie.configuration.FlowInterop
import co.touchlab.skie.configuration.SealedInterop
import co.touchlab.skie.configuration.SuspendInterop

skie {
    features {
        coroutinesInterop.set(true)

        group {
            SuspendInterop.Enabled(true)
            SealedInterop.Enabled(true)
            EnumInterop.Enabled(true)

            FlowInterop.Enabled(false)
            DefaultArgumentInterop.Enabled(false)
        }
    }
}
```

### 6.4 전환 정책

- KMP-NativeCoroutines 전환은 ADR 승인 후에만 허용한다.
- JetBrains Swift Export 전환은 ADR 승인 후에만 허용한다.
- Swift Export가 `suspend`와 `sealed class`를 production 수준으로 공식 지원하면 SKIE 제거를 검토할 수 있다.
- SKIE 제거 또는 대체 시 iOS Adapter/Mapping 계층만 수정되도록 설계한다.

---

## 7. iOS Repository / Adapter 규칙

### 7.1 기본 원칙

- iOS ViewModel은 Kotlin Repository interface를 직접 사용하지 않는다.
- iOS ViewModel은 Swift-native Repository protocol에만 의존한다.
- iOS는 Kotlin Repository interface를 직접 구현하지 않는다.
- 일반 피처 구현은 Swift Repository protocol + Swift implementation을 기본으로 한다.
- KMP 타입을 소비해야 하는 파일만 `iosApp/Adapter` 또는 `iosApp/Mapping`에 둔다.
- Adapter 외부(ViewModel, View)에서 KMP 모듈을 import하지 않는다.

### 7.2 iOS 계층 구조

```text
shared/commonMain Repository interface
    ↓ 계약 기준
Android RepositoryImpl                  iOS Swift RepositoryProtocol
    ↓                                    ↓
Android ViewModel                       iOS Repository implementation / Adapter
                                         ↓ 순수 Swift 타입 반환
                                         iOS ViewModel (@Observable, KMP import 금지)
                                         ↓
                                         SwiftUI View (KMP import 금지)
```

### 7.3 Swift Repository Protocol 템플릿

```swift
protocol LoginRepositoryProtocol {
    func login(id: String, password: String) async throws -> LoginResultSwift
}

enum LoginResultSwift: Equatable {
    case success(UserSwift)
    case failure(LoginErrorSwift)
}

enum LoginErrorSwift: Equatable {
    case invalidCredential
    case network
    case unknown(String?)
}
```

### 7.4 Swift Repository 구현 템플릿

```swift
final class DefaultLoginRepository: LoginRepositoryProtocol {
    func login(id: String, password: String) async throws -> LoginResultSwift {
        // iOS native implementation
        // URLSession, Keychain, OSLog 등 iOS 네이티브 API 사용 가능
        fatalError("Implement platform-specific logic")
    }
}
```

### 7.5 KMP 타입 매핑 템플릿

KMP 타입을 실제로 소비해야 하는 경우에만 `iosApp/Adapter` 또는 `iosApp/Mapping` 파일에서 `import shared`를 허용한다.

```swift
import shared

extension LoginResult {
    func toSwift() -> LoginResultSwift {
        switch onEnum(of: self) {
        case .success(let value):
            return .success(UserSwift(from: value.user))
        case .failure(let value):
            return .failure(value.error.toSwift())
        }
    }
}

extension LoginError {
    func toSwift() -> LoginErrorSwift {
        switch onEnum(of: self) {
        case .invalidCredential:
            return .invalidCredential
        case .network:
            return .network
        case .unknown(let value):
            return .unknown(value.message)
        }
    }
}
```

### 7.6 금지 예시

다음 구조는 금지한다.

```swift
// 금지: ViewModel이 KMP Repository에 직접 의존
private let repo: LoginRepository

// 금지: ViewModel이 KMP 모듈 import
import shared

// 금지: Swift class가 Kotlin suspend interface를 직접 구현하는 방식에 의존
final class IosLoginRepository: LoginRepository { ... }
```

---

## 8. Android 아키텍처 규칙

### 8.1 Android ViewModel 템플릿

```kotlin
@HiltViewModel
class XxxViewModel @Inject constructor(
    private val repo: XxxRepository,   // KMP interface
) : ViewModel() {

    private val _state = MutableStateFlow(XxxState())
    val state = _state.asStateFlow()

    private val _effect = Channel<XxxEffect>()
    val effect = _effect.receiveAsFlow()

    fun onEvent(event: XxxEvent) = viewModelScope.launch {
        // state update + repository call + effect
    }
}
```

### 8.2 Android ViewModel 규칙

- Android DI는 Hilt를 표준으로 사용한다.
- ViewModel은 Repository interface에만 의존한다.
- ViewModel에서 Repository 구현체, Retrofit, Room, DataStore를 직접 참조하지 않는다.
- State 변경은 `_state.update { ... }` 사용을 우선한다.
- Effect는 navigation, toast, dialog 등 one-shot 이벤트에만 사용한다.
- KMP interface에 Flow를 노출하지 않는다.
- Flow는 Android presentation/data 계층 내부에서만 사용할 수 있다.

### 8.3 Android Repository 구현 규칙

- Android Repository 구현체는 Android 전용 모듈에 둔다.
- 구현체는 KMP Repository interface를 구현한다.
- 구현체는 Retrofit, Room, DataStore, Firebase 등 Android/JVM 의존성을 사용할 수 있다.
- Hilt binding module을 같은 feature 또는 data 모듈에 둔다.

예시:

```kotlin
class LoginRepositoryImpl @Inject constructor(
    private val api: LoginApi,
) : LoginRepository {
    override suspend fun login(request: LoginRequest): LoginResult {
        return try {
            val response = api.login(request)
            LoginResult.Success(response.user)
        } catch (e: IOException) {
            LoginResult.Failure(LoginError.Network)
        } catch (e: Throwable) {
            LoginResult.Failure(LoginError.Unknown(e.message))
        }
    }
}
```

---

## 9. iOS 아키텍처 규칙

### 9.1 iOS ViewModel 템플릿

```swift
@Observable
@MainActor
final class XxxViewModel {
    private(set) var state = XxxState()
    var effect: XxxEffectEnvelope? = nil

    private let repo: any XxxRepositoryProtocol

    init(repo: any XxxRepositoryProtocol) {
        self.repo = repo
    }

    func onEvent(_ event: XxxEvent) async {
        // state update + repository call + effect
    }
}

struct XxxEffectEnvelope: Identifiable {
    let id = UUID()
    let value: XxxEffect
}
```

### 9.2 iOS `@Observable` 규칙

- `@Observable`을 표준으로 사용한다.
- `@StateObject` 사용을 금지한다.
- `@Published` 사용을 금지한다.
- `ObservableObject` 사용을 금지한다.
- Observation은 `@Observable` 매크로만 사용한다.
- `ObservableObject`, `@Published`, `@StateObject` 패턴을 혼용하지 않는다.
- View가 ViewModel을 소유하면 `@State`를 사용한다.
- Child View에서 binding이 필요하면 `@Bindable`을 사용한다.
- Child View가 단순 참조만 필요하면 property로 전달한다.

### 9.3 iOS DI 규칙

- iOS는 생성자 주입을 기본 DI 전략으로 사용한다.
- ViewModel 생성자 파라미터는 concrete adapter가 아니라 Swift protocol을 받는다.
- DI 컨테이너 프레임워크(Swinject, Resolver 등)는 사용하지 않는다.
- SwiftUI App 진입점에서 실제 구현체를 생성하여 전달한다.
- `@Environment`로 ViewModel을 공유하지 않는다.
- `@Environment`는 필요한 경우 AppContainer 전달 용도로만 제한적으로 사용한다.
- Preview와 Test는 Mock Repository를 생성자 주입한다.

### 9.4 AppContainer 예시

```swift
@MainActor
final class AppContainer {
    let loginRepository: any LoginRepositoryProtocol

    init() {
        self.loginRepository = DefaultLoginRepository()
    }

    func makeLoginViewModel() -> LoginViewModel {
        LoginViewModel(repo: loginRepository)
    }
}
```

---

## 10. 양 플랫폼 State 동기화

### 10.1 기본 원칙

- State / Event / Effect 이름은 Android/iOS에서 동일하게 유지한다.
- State 필드명은 Android/iOS에서 동일하게 유지한다.
- default value는 Android/iOS에서 동일하게 유지한다.
- nullability 의미는 Android/iOS에서 동일하게 유지한다.
- 동일 Event 입력에 대해 Android/iOS State 전이가 동일해야 한다.
- State 필드 추가/삭제/이름 변경은 Android/iOS를 같은 PR에서 수정한다.

### 10.2 공통 필드명

다음 명칭을 우선 사용한다.

```text
isLoading
isRefreshing
isEmpty
error
errorMessage
data
items
selectedItem
```

### 10.3 Error 표현

- `error`: 도메인 에러 타입
- `errorMessage`: 사용자 표시 문자열

`errorMessage`만 두는 방식은 금지하지 않지만, 재시도 판단이나 로깅이 필요한 피처에서는 `error`와 `errorMessage`를 분리한다.

---

## 11. 로깅 / 에러 리포팅

KMP 공통 로거는 사용하지 않는다. 각 플랫폼이 네이티브 로거를 독립적으로 사용한다.

- Android: Timber + Firebase Crashlytics SDK
- iOS: OSLog + Firebase Crashlytics SDK

### 11.1 규칙

- `shared/commonMain`에는 logger interface 또는 logger implementation을 두지 않는다.
- Android Repository/ViewModel에서는 Android 로깅 정책을 따른다.
- iOS Repository/ViewModel에서는 iOS 로깅 정책을 따른다.
- Crashlytics custom key 이름은 양 플랫폼에서 가능한 한 동일하게 유지한다.

---

## 12. CI/CD

### 12.1 파이프라인 원칙

- `shared` 검증을 먼저 수행한다.
- `shared` 검증 실패 시 Android/iOS build job은 실행하지 않는다.
- `shared` 검증 통과 후 Android/iOS job은 같은 `build` stage에서 병렬 실행한다.
- Android deploy는 Android build 성공 후에만 실행한다.
- iOS deploy는 iOS build 성공 후에만 실행한다.

### 12.2 GitLab CI 구조

GitLab에서 stage는 순차 실행되고, 같은 stage의 job이 병렬 실행된다. 따라서 Android/iOS 병렬화를 위해 둘 다 `build` stage에 둔다.

```yaml
stages:
  - validate
  - build
  - deploy

shared:validate:
  stage: validate
  script:
    - ./gradlew :shared:build

android:build:
  stage: build
  needs: ["shared:validate"]
  script:
    - ./gradlew :androidApp:assembleRelease

ios:build:
  stage: build
  needs: ["shared:validate"]
  script:
    - xcodebuild test -scheme iosApp

android:deploy:
  stage: deploy
  needs: ["android:build"]
  script:
    - fastlane supply

ios:deploy:
  stage: deploy
  needs: ["ios:build"]
  script:
    - fastlane gym
    - fastlane pilot
```

### 12.3 필수 검증 명령

PR 완료 전 Agent는 가능한 범위에서 다음 명령을 실행한다.

```bash
./gradlew :shared:build
./gradlew :androidApp:assembleDebug
./gradlew :feature:login:testDebugUnitTest
xcodebuild test -scheme iosApp
```

실행하지 못한 명령이 있으면 완료 메시지에 이유를 명시한다.

### 12.4 Release pipeline

```bash
./gradlew clean :shared:build :androidApp:assembleRelease
fastlane supply
fastlane gym
fastlane pilot
```

### 12.5 CI 보안 기준

- Android signing key는 CI secret으로만 주입한다.
- iOS signing certificate와 provisioning profile은 CI secret 또는 Fastlane match로 관리한다.
- `.env`, keystore, provisioning profile, API key는 저장소에 커밋하지 않는다.
- Agent는 실제 credential을 코드에 삽입하지 않는다.

---

## 13. 피처 추가 체크리스트

새 피처를 생성하거나 Repository interface를 추가할 때 Agent는 반드시 아래 순서를 따른다.

### 13.1 shared/commonMain 계약 생성

- `XxxRepository` interface
- `XxxRequest`
- `XxxResponse` 또는 DTO
- `XxxResult`
- `XxxError`
- 필요한 계약용 model 타입

### 13.2 Android 구현 생성

- `XxxRepositoryImpl`
- Hilt binding module
- `XxxViewModel`
- `XxxState`
- `XxxEvent`
- `XxxEffect`
- `XxxScreen`
- ViewModel unit test
- Repository unit test 또는 fake/mock test

### 13.3 iOS 구현 생성

- Swift `XxxRepositoryProtocol`
- Swift `DefaultXxxRepository`
- 필요한 경우 KMP 타입 → Swift 타입 mapper
- Swift `XxxState`
- Swift `XxxEvent`
- Swift `XxxEffect`
- `@Observable @MainActor XxxViewModel`
- `XxxView`
- Mock Repository
- ViewModel unit test

### 13.4 State 동기화 확인

- Android/iOS State 필드명이 동일한지 확인한다.
- default value가 동일한지 확인한다.
- nullability 의미가 동일한지 확인한다.
- error / errorMessage 의미가 동일한지 확인한다.

### 13.5 빌드 검증

- `./gradlew :shared:build`
- `./gradlew :androidApp:assembleDebug`
- 변경된 feature module test
- `xcodebuild test -scheme iosApp`

### 13.6 완료 금지 조건

다음 상태에서는 완료 처리할 수 없다.

- Android만 구현된 상태
- iOS만 구현된 상태
- Repository interface 변경 후 한 플랫폼 구현 누락
- State/Event/Effect 한쪽 플랫폼 누락
- 테스트 없이 Repository interface 변경
- SKIE 설정 확인 없이 `suspend fun` 추가
- `CLAUDE.md`와 `AGENTS.md` 규칙 불일치
- ViewModel이 iOS에서 `import shared`를 사용하는 상태
- KMP public interface에 Flow 계열 타입이 노출된 상태

---

## 14. PR 리뷰 기준

### 14.1 Architecture

- `commonMain`에 구현 코드가 들어가지 않았는지 확인한다.
- `Flow`, `StateFlow`, `SharedFlow`가 KMP public interface에 노출되지 않았는지 확인한다.
- `expect/actual`이 ADR 없이 추가되지 않았는지 확인한다.
- Repository interface가 Android/iOS 양쪽에서 반영되었는지 확인한다.
- `CLAUDE.md`와 `AGENTS.md`가 동기화되어 있는지 확인한다.

### 14.2 Android

- ViewModel이 Hilt로 주입되는지 확인한다.
- ViewModel이 Repository interface에만 의존하는지 확인한다.
- StateFlow/Channel 사용이 템플릿과 일치하는지 확인한다.
- Repository 구현체나 플랫폼 API가 ViewModel에 직접 들어가지 않았는지 확인한다.
- feature 모듈에 KMP plugin이 적용되지 않았는지 확인한다.

### 14.3 iOS

- ViewModel이 `@Observable @MainActor final class`인지 확인한다.
- `@StateObject`, `@Published`, `ObservableObject`가 사용되지 않았는지 확인한다.
- 생성자 주입이 지켜졌는지 확인한다.
- ViewModel이 Swift protocol에 의존하는지 확인한다.
- SwiftUI View와 ViewModel이 KMP 타입을 직접 import하지 않는지 확인한다.

### 14.4 CI/Test

- shared build/test가 통과하는지 확인한다.
- Android build/test가 통과하는지 확인한다.
- iOS build/test가 통과하는지 확인한다.
- 실패한 명령이 있으면 PR 설명에 실패 사유와 미해결 항목을 명시한다.

---

## 15. ADR 필요 조건

다음 변경은 ADR 없이 진행할 수 없다.

- SKIE 제거 또는 KMP-NativeCoroutines로 전환
- Swift Export로 iOS bridge 전환
- `suspend fun` 금지/허용 정책 변경
- `expect/actual` 사용 허용 범위 확대
- KMP `commonMain`에 구현 계층 추가
- iOS DI 프레임워크 도입
- Android DI 프레임워크 변경
- Gradle 모듈 구조 변경
- `shared` 모듈에 Android application entry point 추가
- KMP 타입을 SwiftUI View 또는 iOS ViewModel에서 직접 소비하도록 정책 변경
- CI stage 구조 변경

ADR에는 다음을 포함한다.

```text
- 배경
- 결정 내용
- 대안
- 영향 범위
- 마이그레이션 계획
- 롤백 계획
- Android 영향
- iOS 영향
- CI 영향
```

---

## 16. 의사결정 이력

| 결정 | 선택 | 제거된 대안 | 사유 |
|------|------|-------------|------|
| KMP 전략 | Interface-only 계약 레이어 | Full KMP 로직 공유 | 경계 마찰 최소화, Agent 예측성 |
| KMP 로거 | 제거, 플랫폼별 독립 | Kermit commonMain | commonMain 구현 코드 최소화 |
| Android 아키텍처 | ViewModel + Flow UDF | Orbit MVI | Agent 훈련 데이터와 Android 표준성 |
| iOS 아키텍처 | MVVM + @Observable | TCA | Apple 표준, Agent 이해도 |
| 라우팅 | 플랫폼 네이티브 | Uber RIBs | 1인 개발 기준 오버헤드 감소 |
| iOS Bridge | SKIE 제한 사용 | KMP-NativeCoroutines | suspend/sealed/enum interop 표준화 |
| iOS KMP 소비 | Swift protocol + Adapter/Mapping 격리 | KMP 타입 직접 소비 | SKIE 교체 시 영향 범위 최소화 |
| Feature 구조 | Android 전용 `feature/*` Gradle 모듈 | shared feature KMP 모듈 | Android UI 구현과 KMP 계약 분리 |

---

## 17. Agent 최종 지시문

Agent는 이 저장소에서 아래 규칙을 반드시 따른다.

1. `commonMain`은 계약 계층으로만 사용한다.
2. Repository interface 추가 또는 변경 시 Android 구현, iOS 구현, 양 플랫폼 테스트를 같은 작업에 포함한다.
3. KMP public interface에 `Flow`, `StateFlow`, `SharedFlow`를 노출하지 않는다.
4. `suspend fun`은 SKIE 설정이 유지되는 경우에만 허용한다.
5. SKIE Flow interop과 default argument interop은 사용하지 않는다.
6. iOS SwiftUI View와 ViewModel은 KMP 타입을 직접 import하지 않는다.
7. iOS ViewModel은 `@Observable @MainActor final class`로 작성한다.
8. iOS ViewModel은 Swift Repository protocol에만 의존한다.
9. iOS DI는 생성자 주입과 AppContainer를 기본으로 한다.
10. Android DI는 Hilt를 기본으로 한다.
11. `shared`, `androidApp`, `feature/*` 모듈 역할을 혼동하지 않는다.
12. 완료 전 shared, Android, iOS 검증 명령을 실행하거나, 실행하지 못한 이유를 명시한다.
