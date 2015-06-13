---
layout: post
title: "CocoaSeeds로 iOS 7 프로젝트에서 Swift 라이브러리 사용하기"
description: iOS 7 프로젝트에서 Swift로 작성된 라이브러리 의존성 관리를 도와주는 CocoaSeeds를 개발하게 된 배경과 사용법에 대해 공유합니다.
---

## Cocoa 의존성 관리 도구와 Swift

Python에는 pip이 있고 Java에는 Maven과 Gradle이 있듯, Cocoa 진영에도 강력한 의존성 관리 도구인 [CocoaPods][cocoapods]이 있다[^1]. CocoaPods은 외부에 있는 라이브러리의 소스코드를 다운받아서 가지고 있다가, 프로젝트를 빌드할 때 static library(.a)를 만들어서 Linking하는 방식으로 작동한다. 하지만 Swift로 작성된 라이브러리는 static library를 만들 수 없었기 때문에[^2] 애플에서는 dynamic framework라는 새로운 대안을 제시했다. CocoaPods은 발빠르게 dynamic framework를 지원하는 버전을 내놓았지만, **dynamic framework는 iOS 8 버전을 최소로 요구**하기 때문에 문제가 발생하기 시작했다. **Swift로 작성된 라이브러리를 iOS 7 프로젝트에서는 사용할 수 없다는 것**이다.
> ld: warning: embedded dylibs/frameworks only run on iOS 8 or later

iOS 7을 지원하는 프로젝트에서 Swift로 작성된 라이브러리를 사용하고 싶을 때에는 기존 의존성 관리 도구를 사용할 수 없고, 직접 소스코드를 다운받거나 Git Submodule을 이용하여 소스코드를 받은 뒤 프로젝트에 직접 임베드시켜야 한다. 하지만 이 방법에는 의존성 관리가 어렵다는 단점이 있다.


## 해결책 - CocoaSeeds

이를 해결하기 위해 [CocoaSeeds][cocoaseeds]를 만들었다. CocoaSeeds는 Seedfile에 작성된 의존성을 기반으로 소스코드를 다운받고, 프로젝트에 자동으로 임베드시켜주는 기능을 한다. CocoaSeeds는 static library나 dynamic framework를 전혀 사용하지 않기 때문에 iOS 7 프로젝트에서도 Swift로 작성된 라이브러리를 문제없이 사용할 수 있게 해준다. 또한, CocoaPods이나 Carthage와 같은 다른 의존성 관리 도구와도 함께 사용될 수 있다.

CocoaSeeds는 Ruby로 작성되었으며, RubyGem을 통해 설치할 수 있다.

```bash
$ [sudo] gem install cocoaseeds
```

CocoaSeeds를 사용하기 위해서는 **Seedfile**에 의존성을 기재해야 한다. Seedfile은 **.xcodeproj**와 동일한 디렉토리에 만들어주면 된다.

Seedfile은 아래와 같이 생겼다. 라이브러리의 소스코드 위치와 버전, 그리고 프로젝트에 사용할 파일 패턴 정보를 가지고 있다. `files` 정보를 입력하지 않으면, 기본적으로 Objective-C나 Swift와 관련된 모든 소스코드를 가져온다. 만약 소스코드에 예제 프로젝트와 같이 실제 라이브러리 외의 코드가 포함되어 있다면 문제가 생길 여지가 있기 때문에 가능하면 파일 패턴을 입력해주도록 한다. 현재는 GitHub에 저장된 소스코드만 사용할 수 있다.

```ruby
# 모든 타겟에 사용될 Seeds
github "Alamofire/Alamofire", "1.2.1", :files => "Source/*.{swift,h}"
github "devxoul/JLToast", "1.2.2", :files => "JLToast/*.{swift,h}"
github "devxoul/SwipeBack", "1.0.4"  # files 기본값: */**.{h,m,mm,swift}
github "Masonry/SnapKit", "0.10.0", :files => "Source/*.{swift,h}"

# MyAppTests 타겟에만 포함될 Seeds
target :MyAppTests do
  github "Quick/Quick", "v0.3.1", :files => "Quick/**.{swift,h}"
  github "Quick/Nimble", "v0.4.2", :files => "Nimble/**.{swift,h}"
end
```

Seedfile을 작성한 뒤에는 `seed` 명령어를 통해 라이브러리를 설치할 수 있다.

```bash
$ seed install
```

![seed-install](/images/2015-06-14/seed-install.png)

위와 같이 라이브러리를 설치하는 과정이 끝난 후, Xcode를 확인하면 아래와 같이 설치한 라이브러리들이 임베드되어있는 것을 확인할 수 있다.

![seed-project-navigator](/images/2015-06-14/seed-project-navigator.png)

프로젝트 네비게이터에 Seeds라는 그룹으로 소스코드가 임베드되었다.

![seed-compile-sources](/images/2015-06-14/seed-compile-sources.png)

설정한 타겟에 필요한 소스코드만 Compile Sources에 추가된 것을 볼 수 있다. 프로젝트를 빌드하면 성공적으로 빌드된다.


## 기술적 한계

### 불필요한 diff

CocoaSeeds는 소스코드를 직접 프로젝트에 임베드시키기 때문에, Xcode 프로젝트 파일인 .pbxproj의 diff 변화가 불가피하다. 각 파일의 파일명에 기반한 UUID를 생성하도록 하는 방식으로 설치될 때마다 똑같은 파일이 다른 UUID를 가지는 경우는 없게 함[^3]으로써 문제를 최소화시켰지만, 여전히 불필요한 diff가 발생하는 점은 기술적인 한계이다.


### 네임스페이스

위와 마찬가지로 소스코드를 직접 임베드하기 때문에 프레임워크에 할당되는 네임스페이스가 존재하지 않는다. 따라서 네임스페이스의 충돌에 각별히 신경을 써야 한다. [Alamofire][alamofire]와 같이 네임스페이스를 기반으로 한 독특한 인터페이스를 가진 라이브러리의 맛을 100% 느낄 수 없다는 것도 나와 같이 '아름다운 코드'에 관심이 있는 개발자들에게는 아쉬운 부분일 것이다.

> `Alamofire.request()`는  `request()`로 사용해야 한다.


## 마치며

CocoaSeeds는 iOS 7 점유율이 낮아지면서 자연스럽게 도태될 프로젝트임에 틀림없다. 프로젝트 자체만으로 볼 때에는 안타까운 일이겠지만, 개인적으로는 예전부터 배우고싶던 Ruby를 공부하면서 만들어본 첫 프로젝트이기 때문에 의미가 깊다. 이 글을 작성하는 도중에 한 칠레 개발자가 이슈를 등록해주었는데, 처음으로 나 이외의 CocoaSeeds 사용자를 보게 되는 것이라 너무 기뻤다.[^4]



[^1]: 새롭게 뜨고 있는 [Carthage][carthage]도 있지만, 글의 간결함을 위하여 CocoaPods을 의존성 관리 도구의 대명사처럼 사용했다.
[^2]: [http://openradar.appspot.com/radar?id=5536341827780608](http://openradar.appspot.com/radar?id=5536341827780608)
[^3]: [CocoaSeeds #3](https://github.com/devxoul/CocoaSeeds/pull/3)
[^4]: 사실 내가 일하고 있는 [StyleShare](https://stylesha.re)에서도 사용한다.


[cocoapods]: https://cocoapods.org
[carthage]: https://github.com/Carthage/Carthage
[cocoaseeds]: https://github.com/devxoul/CocoaSeeds
[alamofire]: https://github.com/Alamofire/Alamofire
