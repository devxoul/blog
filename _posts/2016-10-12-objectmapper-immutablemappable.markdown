---
layout: post
title: "ObjectMapper에 ImmutableMappable 기능을 기여한 경험 공유"
description: ObjectMapper는 Swift에서 가장 인기있는 모델 프레임워크입니다. 2.1.0 릴리즈에 제가 개발한 ImmutableMappable 기능을 기여한 배경과 과정을 공유합니다.
---

## ObjectMapper

[ObjectMapper](https://github.com/Hearst-DD/ObjectMapper)는 Swift에서 가장 인기있는 모델 프레임워크입니다. JSON 데이터를 모델 클래스 또는 구조체에 매핑할 수 있는 직관적인 기능을 제공합니다. 값을 원하는 타입으로 변환해서 매핑할 수도 있고, 계층 구조도 지원합니다. 또한, [Alamofire](https://github.com/Alamofire/Alamofire)나 [Realm](https://realm.io)과 함께 사용할 수 있는 커뮤니티 기반의 확장 라이브러리도 존재합니다.

ObjectMapper를 사용하여 모델을 정의할 때에는 `Mappable` 프로토콜을 상속받아서 정의합니다. `Mappable` 프로토콜은 생성자와 속성 매핑 메서드를 정의하고 있습니다.

```swift
protocol Mappable {
  init?(map: Map)
  func mapping(map: Map)
}
```

- `init?(map: Map)` 생성자는 주어진 `Map` 객체에 담긴 JSON 정보가 올바른지를 검사하는데 사용됩니다. 특정 키가 없거나, 값이 잘못된 경우에 `nil`을 반환할 수 있도록 디자인되었습니다.
- `func mapping(map: Map)` 메서드에서는 속성과 JSON 키를 매핑합니다. `<-` 연산자를 사용하여 `property <- map["key"]`와 같이 속성과 JSON 키를 매핑할 수 있습니다.

`Mappable` 프로토콜을 적용한 모델의 예시는 다음과 같습니다.

```swift
struct User: Mappable {
  var id: Int!
  var email: String?
  var birthday: Date?

  init?(map: Map) {
  }
  
  mutating func mapping(map: Map) {
    self.id <- map["id"]
    self.email <- map["email"]
    self.birthday <- (map["birthday"], DateTransform())
  }
}
```

그리고 이렇게 정의된 모델을 다음과 같이 생성하여 사용할 수 있습니다.

```swift
let user: User? = User(JSONString: jsonString)
```

## 기존 방식의 한계

ObjectMapper는 기본적으로 모든 속성을 `func mapping(map:)` 메서드에서 매핑하도록 구현되어 있습니다. 이 방식에는 두 가지 문제가 존재합니다.

1. 모든 속성이 `Optional`이거나, 초깃값을 가지고 있어야 합니다.

    그렇지 않으면, 생성자에서 속성의 값이 초기화되지 않았다는 컴파일 에러가 발생하게 됩니다. 따라서 모든 속성을 `Optional`로 사용하거나, 혹은 값이 항상 있다고 가정하고 편의를 위해 `!`를 사용해야 합니다.

2. `let`으로 선언된 상수는 매핑할 수 없습니다.

    `<-` 연산자를 사용하면 값을 변경하기 위해 좌항을 `inout`으로 참조합니다. `let`은 값이 변경될 수 없으므로 `inout` 참조가 불가합니다. 따라서 모든 속성은 `var`로 정의되어야 합니다.

사실 이렇게만 보면 '이정도 한계쯤이야 뭐...' 하고 사용할 수 있겠습니다. 사실 딱히 불편하지도 않거든요. (저도 그랬고요) 하지만, Swift 3 버전이 릴리즈되면서 1번 문제가 큰 문제로 작용하기 시작했습니다. 바로 `ImplicitlyUnwrappedOptional` 타입이 제거[^1]된 것입니다. Swift 2 버전까지는 `Int!`가 `ImplicitlyUnwrappedOptional<Int>` 타입을 나타내는 것이었지만, Swift 3 버전부터는 `Optional<Int>`로 취급되고, 컴파일 타임에 강제로 옵셔널을 언래핑하도록 바뀌었습니다.

위의 변화는 단순해보이지만 문자열 포맷팅에 있어 치명적인 문제를 발생시킵니다. 아래와 같은 상황을 생각해봅시다.

```swift
let id: Int! = 123
let urlString = "myapp://user/\(id)"
```

Swift 2 버전에서는 `urlString` 변수가 `myapp://user/123`와 같이 의도한 대로 포맷팅됩니다. 하지만, Swift 3 버전에서는 `myapp://user/Optional<123>`로 포맷팅됩니다. (!!!) 심지어 컴파일러가 경고도 해주지 않습니다. 앞서 언급한 첫 번째 이유로 모델의 모든 속성은 옵셔널로 정의되기 때문에, 문자열을 포맷팅하는 코드에서 무수히 많은 버그가 생기게 됩니다.

또한, `id`와 같이 절대로 변하지 않아야 하는 값이 `var`로 선언되었기 때문에 항상 위험을 안고 가야 합니다.

## 대안 찾기

처음으로 생각한 대안은 `init?(map: Map)` 생성자에서 값을 직접 설정하는 방법입니다. 문서화되지 않은 `Map` 클래스의 `currentValue` 속성을 사용하는 방법입니다.

```swift
struct User {
  let id: Int
  var email: String?
  var birthday: Date?

  init?(map: Map) {
    if let id = map["id"].currentValue as? Int {
      self.id = id
    } else {
      return nil
    }
    self.email = map["email"].currentValue as? String
    self.birthday = DateTransform().transformFromJSON(map["birthday"].currentValue)
  }
  
  mutating func mapping(map: Map) {
    var id = self.id
    id <- map["id"]
    self.email <- map["email"]
    self.birthday <- (map["birthday"], DateTransform())
  }
}
```

하지만 이 방식은 달랑 3개의 속성을 매핑하는데만 해도 굉장히 많은 코딩양을 필요로 합니다. 또한 불필요한 `if-let`문과, 통일되지 않은 `Transform`의 사용으로 가독성에도 좋지 않습니다.

그 다음으로는 GitHub에 등록된 비슷한 이슈를 찾아봤습니다. `immutable` 이라는 키워드로 이슈를 검색하니 꽤나 많은 이슈들이 나왔습니다. 그 중 몇 개는 실제 구현을 해서 PR까지 보낸 경우도 있고, 꽤나 많은 논의가 오간 이슈도 있었습니다. 하지만 머지가 되지는 않았습니다.

## 직접 만들어보자!

그래서 직접 만들어보기로 했습니다. 다른 개발자가 작성한 PR이 머지되지 않은 이유를 살펴보니 기존에 `init?(map: Map)`으로 사용되던 `Mappable`의 코드를 모두 `init(map: Map) throws`로 바꾸는 큰 PR이어서 메인테이너가 쉽사리 결정을 내리지 못하는 것 같았습니다. 여기서 힌트를 얻어, 기존 코드에 영향을 주지 않는 방식으로 개발해보자는 전략을 세웠습니다. 핵심은 기존 코드를 하나도 건드리지 않고 추가만 하는 것입니다.

먼저, 실제로 모델을 정의하는 인터페이스를 상상해보았습니다.

```swift
struct User: ImmutableMappable {
  let name: String
  let createdAt: Date
  let updatedAt: Date?
  let posts: [Post]

  init(map: Map) throws {
    // 값이 없으면 에러를 던집니다.
    self.name = try map.value("name")

    // 값이 없거나 값 변환에 실패하면 에러를 던집니다.
    self.createdAt = try map.value("createdAt", using: DateTransform())

    // 값이 없거나 값 변환에 실패하면 `nil`을 반환합니다.
    self.updatedAt = try? map.value("updatedAt", using: DateTransform())
    
    // 값이 없으면 빈 배열을 기본값으로 사용합니다.
    self.posts = (try? map.value("posts")) ?? []
  }
}
```

생성자가 `throws`로 정의되었기 때문에, 모델 생성 시에도 `try`가 필요합니다.

```swift
let user = try User(JSONString: jsonString)
```

최상위 프로토콜인 `BaseMappable`을 상속받아 `ImmutableMappable`이라는 별도의 프로토콜을 만들었습니다. Swift에서는 `subscript` 문법이 제네릭을 지원하지 않기 때문에 `map["key"]` 대신에 별도의 메서드를 사용해야 했습니다. 이 메서드는 `Mapper` 클래스에 정의되어야 하는데, 기존 코드를 하나도 건드리지 않기 위해서 모든 확장 메서드를 **`ImmutableMappable.swift`** 파일에 모두 작성하였습니다. 이 전략을 사용했더니, 테스트 코드까지 포함한 총 파일 diff가 3개였습니다. (**`project.pbxproj`**, **`ImmutableMappable.swift`**, **`ImmutableMappableTests.swift`**)

그리고 나서 기존 구현을 건드리지 않는다는 내용을 충분히 강조하여 PR을 작성했습니다. ([ObjectMapper#592](https://github.com/Hearst-DD/ObjectMapper/pull/592)) 이 PR이 실제로 `master` 브랜치에 머지되기 전까지 아래와 같은 논의가 오갔습니다.

- 리뷰: 정말 별도의 기능을 만들어야 하나? 기존 `Mappable`을 고치는게 더 간결하지 않은지?
- 답변: 기존 코드가 변경되는 것을 우려하여 이전 PR들을 머지하지 않았다고 판단했다. 따라서 별도의 `ImmutableMappable`을 만든 것. 이를 베타 기능으로 출시하는 것이 어떤가? 그렇다면 이 기능을 필요로 하는 개발자들에게 기능을 제공할 수 있고, 동시에 정식 기능으로 출시되기 전까지 충분한 테스트를 거칠 수 있다.
- 리뷰: 아직 `master` 브랜치에 머지하기에는 조금 부족한 것 같다. (직접 한 말은 아니고, 내가 작성한 커밋에 몇 개의 커밋을 더해서 새로운 PR을 작성)
- 답변: 이렇게 하는 것보다 `immutable-mappable` 브랜치를 만들고 내가 작성한 PR을 우선 머지한 뒤, 이 브랜치에다가 부족한 기능들을 추가해 나가는 것이 어떤가? 그러면 추가 기능에 대한 PR들을 작고 간결하게 유지할 수 있을 것. 베타 릴리즈 하기에 충분하다고 판단되는 시점에 `master` 브랜치로 머지하면 될 것 같다.
- 리뷰: 이제 릴리즈 하기에 적당한 시점이 온 것 같다. 혹시 문서를 직접 작성해보겠는가?
- 답변: 좋다. 지금 바로 작성해서 `immutable-mappable` 브랜치에 PR을 보내겠다.

이후 `immutable-mappable` 브랜치는 `master` 브랜치에 머지됐고, `ImmutableMappable` 기능은 [2.1.0 릴리즈](https://github.com/Hearst-DD/ObjectMapper/releases/tag/2.1.0)에 포함되었습니다. 🎉

![objectmapper-2.1.0](/images/2016-10-12/objectmapper-2.1.0.png)

## 소감

타인의 프로젝트에 이렇게 큰 기능을 추가해본 적은 처음이었습니다. 특히나 개인적으로 굉장히 좋아하는 프로젝트였기 때문에 프로젝트 메인테이너와 논의를 시작할 때부터 PR이 머지되기까지의 모든 과정이 즐거웠습니다. 이 프로젝트를 관찰하면서 느낀 점은 메인테이너가 바쁘다는 것이었습니다. 메인테이너의 입장에서 우려되는 점들을 파악하고 이를 해결할 수 있는 방식을 대화를 통해 찾아가는 과정이 굉장히 의미있는 경험이었습니다.

[^1]: [[SE-0054] Abolish ImplicitlyUnwrappedOptional type](https://github.com/apple/swift-evolution/blob/master/proposals/0054-abolish-iuo.md)
