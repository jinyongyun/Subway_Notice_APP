# Subway_Notice_APP
지하철 도착 정보 앱 만들기 🚊

## 비동기 처리 알아보기

그렇다 우리가 자바에서 많이 사용했던 동기와 비동기! 

오늘은 서버 통신이 있다 보니 그녀석에 대해 정리해보려고 한다.

우리가 토요일 아침에 집안일을 한다고 해보자.

**청소기를 돌리고 → 부모님께 전화를 하고 → 빨래를 한다**

하나의 일을 끝난뒤, 순서대로 진행한다면 우리는 이걸 동기라고 한다.

하지만 만약 우리가

**로봇청소기를 돌리면서**

**부모님께 전화를 하면서**

**빨래를 한다면**

이처럼 병렬적으로 일이 진행되는 것이 바로 비동기 방식이다.

 비동기 처리가 가장 많이 행해지는 때가 서버와 네트워크 통신이 있을 때다!

우리가 GET 메서드 방식으로 서버에 어떤 정보를 요청했을 때

데이터와 .success 등의 상태값이 날아오기 전까지 우리는 그저 멍하니 있을 순 없다.

뭔가 애니메이션도 표시해야하고, 다양한 동작을 앱은 데이터를 기다리는 동안 수행하고 있어야 한다.

이를 위해서는 기다리면서 ***동시에***  어떤 일을 진행해야 하니 비동기처리가 필요하다!

iOS에서 비동기 처리를 구현하기 위한 방법은 아래와 같이 5가지가 있다.

- Notification Center
- Delegate Pattern
- Closure
- RxSwift
- Combine

이번 앱에서는 도착 정보 API를 request 할 때, 비동기 처리를 위해 closure를 사용할 예정이다.

이번에는 지하철 역을 검색하고 해당 지하철 역의 실시간 정보를 확인할 수 있는

지하철 도착 정보 앱을 만들어보겠다.

일단 이번 앱을 만들면서 사용하게 될 외부 라이브러리는 SnapKit과 Alamofire가 있다.

SnapKit에서 눈치챌 수 있듯이, 이번에도 storyboard 없이 오직 코드만으로 UI를 구성해보고자 한다.

굳이 pod 파일을 사용하지 않고 package manager를 이용해서 라이브러리를 설치할 예정이다.

Xcode 프로젝트 생성 → 외부 라이브러리 설치 → main.storyboard 설치

이 과정은 이제 익숙해졌으리라 생각한다.

<img width="274" alt="스크린샷 2024-01-22 오후 6 39 56" src="https://github.com/jinyongyun/Subway_Notice_APP/assets/102133961/4ac8b0fc-cf34-4e09-8e68-2689843c401e">


File > Add Package Dependency로 almofire와 snapKit을 설치한다.

main.storyboard를 삭제하고. info.plist와 설정에 남아있는 main 관련 정보를 지워준다.
<img width="887" alt="스크린샷 2024-01-22 오후 6 42 26" src="https://github.com/jinyongyun/Subway_Notice_APP/assets/102133961/e8627b50-4be4-435f-abab-2535ea22422d">

이제 SceneDelegate로 가서 사용하지 않는 메서드를 지우고, 

window를 초기화하는 걸 사용해서, 새롭게 viewController를 베이스로 하는 Scene을 만들어준다.

```swift
//  SceneDelegate.swift
//  SubwayNotice
//
//  Created by jinyong yun on 1/22/24.
//

import UIKit

class SceneDelegate: UIResponder, UIWindowSceneDelegate {

    var window: UIWindow?

    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options connectionOptions: UIScene.ConnectionOptions) {
        guard let windowScene = (scene as? UIWindowScene) else { return }
        
        window = UIWindow(windowScene: windowScene)
        window?.backgroundColor = .systemBackground
        window?.rootViewController = ViewController()
        window?.makeKeyAndVisible() //이거 빠뜨리지 말라고 했지!! 이거 해야 적용됨
    }
}
```

## 지하철 검색 화면 구현하기

지하철 검색화면의 흐름을 그려보자.

먼저 유저가 지하철 도착 정보 화면에 검색창을 클릭하면, 검색창에 입력된 값을 자동완성 하기 위한 

UITableView가 표시된다. (어디여 앱에서 경험해봤다!)

유저가 검색창에 키워드를 입력하게 되면, 검색창에 입력된 값에 맞는 지하철 역 이름에 맞는 자동 완성 결과를 서버에 요청한다.

그렇게 서버에서 받은 지하철 검색 결과를 UITableView의 cell에 표시하게 된다.

검색창을 닫으면(Cancel) 검색 결과가 초기화 되어야 한다.

이를 구현하기 위해서 뭐가 필요할까?

먼저 UISearchController가 필요하다. 어디여 앱에서는 viewController에 searchBar와 tableView를 연결시키고, 델리게이트로 구현했는데

이번에는 **UISearchController(+UITableView)**라는 컨트롤러를 사용한다!

UISearchController는 UIKit의 UI 컴포넌트 중 하나로 UINavigationItem에 추가해서 사용할 예정이다.

(UISearchController안에 이미 UISearchBar 있다. UISearchController에는 cancel 버튼이라던가 다양한 기능이 포함되어있어 어디여에서 했던 방식보다 더 빠르고 편리하다.)

 물론 UITableView는 포함되어 있지 않아서 ishidden을 이용해서 처리해줘야 한다. 🤦🏻

이제 본격적으로 코드를 작성해보자.

기존에 viewController를 StationSearchViewController로 rename 해줬다.

SceneDelegate에서도 

```swift
window?.rootViewController = UINavigationController(rootViewController: StationSearchViewController())
```

루트 뷰 컨트롤러 설정해줬던 부분을 다음과 같이 바꿔주자. 네비게이션 컨트롤러를 끼워준 것이다.

```swift
//  StationSearchViewController.swift
//  SubwayNotice
//
//  Created by jinyong yun on 1/22/24.
//

import SnapKit
import UIKit

class StationSearchViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()
        
        navigationController?.navigationBar.prefersLargeTitles = true
        navigationItem.title = "지하철 도착 정보 🚇"

        //검색 기능을 위해 반드시 필요한 UISearchController 추가
        let searchController = UISearchController()
        searchController.searchBar.placeholder = "지하철 역을 입력해주세요" //서치바의 placeholder 설정
        searchController.obscuresBackgroundDuringPresentation = false // TODO
        
        navigationItem.searchController = searchController
    }

}
```

자 일단 여기까지 작성했다. 화면 맨 위에 큰 제목을 부여하기 위해 네비게이션 바에 prefersLargeTitles를 true로 설정했다.

제일 중요한 UISearchController()의 경우 navigationItem에 이미 searchController라는 프로퍼티가 있어서 여기에 넣어주면 된다!

![simulator_screenshot_7EEF57EF-7183-4964-BFC5-AD4CC6261782](https://github.com/jinyongyun/Subway_Notice_APP/assets/102133961/a9c459d2-e45a-48ab-b1a9-b1a87bef9163)


여기까지 따라왔다면, 다음과 같이 화면이 나타날 것이다!

obscuresBackgroundDuringPresentation → 이게 뭡니까?

독특한 녀석이 끼어있는데, 이녀석은 searchBar를 클릭했을 때, false면 밑에 화면이 백그라운드 색상으로 보이고, true면 약간 불투명한 회색으로 보인다.

<img width="485" alt="스크린샷 2024-01-22 오후 7 28 26" src="https://github.com/jinyongyun/Subway_Notice_APP/assets/102133961/75e6a56f-d8cf-4cb4-ab4d-b65d8939bd44">


obscuresBackgroundDuringPresentation

= false

<img width="478" alt="스크린샷 2024-01-22 오후 7 30 08" src="https://github.com/jinyongyun/Subway_Notice_APP/assets/102133961/d0176fcb-d478-4f21-ae3d-a91ac32b1d08">


obscuresBackgroundDuringPresentation

= true

우리는 searchBar 밑에 테이블 뷰를 작성해야 해서, false를 줬다.

```swift
//  StationSearchViewController.swift
//  SubwayNotice
//
//  Created by jinyong yun on 1/22/24.
//

import SnapKit
import UIKit

class StationSearchViewController: UIViewController {
    
    private lazy var tableView: UITableView = {
        let tableView = UITableView()
        tableView.dataSource = self
        return tableView
    }()
    

    override func viewDidLoad() {
        super.viewDidLoad()
        
        navigationController?.navigationBar.prefersLargeTitles = true
        navigationItem.title = "지하철 도착 정보 🚇"

        //검색 기능을 위해 반드시 필요한 UISearchController 추가
        let searchController = UISearchController()
        searchController.searchBar.placeholder = "지하철 역을 입력해주세요" //서치바의 placeholder 설정
        searchController.obscuresBackgroundDuringPresentation = false // TODO
        
        navigationItem.searchController = searchController
    }

    
}

extension StationSearchViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return 10 //임의 설정
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = UITableViewCell()
        cell.textLabel?.text = "\(indexPath.item)"
        return cell
    }
    
    
}
```

다음으로는 lazy var로 tableView를 선언했다.

lazy로 설정한 이유는 공식문서에서 그렇게 하라고 추천하기도 했고, 

굳이 메모리에 미리 올려놓을 필요가 없기 때문이다. 

테이블 뷰를 클로저 형식으로 정의했는데, 안에 dataSource도 같이 지정해줬다.

dataSource는 따로 extension으로 빼줘서, 안에 numberOfRowsInSection과 cellForRowAt을 구현해줬다. numberOfRowInSection에서 임의로 10개의 셀을 보여주도록 설정했고, cellForRowAt에서 이번에는 굳이 커스텀 셀을 만들지 않기 때문에, 기본 셀을 리턴해줬다.

이제 레이아웃을 설정해보자.

viewDidLoad가 너무 뚱뚱해질까봐 따로 메서드로 빼줬다.

```swift
override func viewDidLoad() {
        super.viewDidLoad()
        setNavigationItems()
        setTableViewLayout()
    }
    
    private func setNavigationItems() {
        navigationController?.navigationBar.prefersLargeTitles = true
        navigationItem.title = "지하철 도착 정보 🚇"

        //검색 기능을 위해 반드시 필요한 UISearchController 추가
        let searchController = UISearchController()
        searchController.searchBar.placeholder = "지하철 역을 입력해주세요" //서치바의 placeholder 설정
        searchController.obscuresBackgroundDuringPresentation = false // TODO
        
        navigationItem.searchController = searchController
    }

    private func setTableViewLayout() {
        view.addSubview(tableView)
        tableView.snp.makeConstraints { $0.edges.equalToSuperview() } //전체 UIViewController에 딱 맞게!
    }
```

![simulator_screenshot_90487969-17FC-42D2-AC61-6D7964AB62AA](https://github.com/jinyongyun/Subway_Notice_APP/assets/102133961/d5305d17-0924-47b9-b5f9-fd6e1a5fe24e)


잘 나왔지만, searchBar에 커서가 있을 때만 테이블 뷰가 나타나야 한다.

지금은 늘 표시된다.

커서의 유무에 따라 테이블 뷰를 표시시켜보자.

UISearchBarDelegate로 구현할 수 있고, 

searchController.searchBar.delegate = **self를 당연히 searchController를 설정하는 부분, 즉 setNavigationItems 메서드에 추가해야한다.**

```swift
extension StationSearchViewController: UISearchBarDelegate {
    func searchBarTextDidBeginEditing(_ searchBar: UISearchBar) {
        tableView.isHidden = false
    }
    
    func searchBarTextDidEndEditing(_ searchBar: UISearchBar) {
        tableView.isHidden = true
    }
}
```

UISearchBarDelegate에서 구현해야 하는 함수는 searchBarTextDidBeginEditing과 

searchBarTextDidEndEditing이다.

굉장히 직관적인 이름처럼 text 입력이 시작될 때와 끝날 때 실행되는 메서드이다.

tableView.isHidden = **true 를 기본값으로 주는 걸 잊지 말자!**

아마 실행시켜보면 서치바가 안보일텐데 당황하지 말고 아래로 화면을 약간 내려보면 나온다.

왜 이렇게 되냐면, 우리가 numberOfRowsInSection에서 보여질 셀의 개수를 10개로 설정해서 그렇다. 

서치바와 테이블 뷰의 상관관계가 어느정도 있어서 이런 기이한 현상이 발생한다.

**private** **var** numberOfCell: Int = 0

이렇게 변수를 설정해주고 

numberOfRowsInSection에서 이걸 리턴해주면 잘 보인다.  0으로 계속 초기화를 시켜주지만, 서버에서 자동검색 결과를 받아오기 전까지만 이렇게 설정해주는 것이다.

```swift
extension StationSearchViewController: UISearchBarDelegate {
    func searchBarTextDidBeginEditing(_ searchBar: UISearchBar) {
        numberOfCell = 10
        tableView.isHidden = false
    }
    
    func searchBarTextDidEndEditing(_ searchBar: UISearchBar) {
        numberOfCell = 0
        tableView.isHidden = true
    }
}
```

여기도 임시적으로 보여질 때 셀 개수를 10개로 하고, 작성 끝나면 0개로 다시 만들어주도록 해줬다.



https://github.com/jinyongyun/Subway_Notice_APP/assets/102133961/2a701e0b-db94-448e-94ff-5c7e2abc7ba6



## 도착 정보 화면 구현하기

우리가 구현해야 하는 도착 정보 화면의 플로우를 살펴보자.

searchBar 밑에 구현된 테이블 뷰 셀을 클릭하면, 해당 지하철 역에 지하철 도착 정보가 CollectionView 형태(테이블 뷰랑 비슷하게)로 나타나는 것을 원한다.

이 컬렉션 뷰를 밑으로 내리면 새로운 정보가 업데이트 된다.

가장 먼저 구현해야 하는 기능은 테이블 뷰 셀을 선택했을 때 도착 정보 화면이 나타나게 하는 것이다.

일단 상세 도착 정보 화면이 나타날 뷰 컨트롤러를 먼저 만들었다.

StationDetailViewController로 이름지어줬다.

그리고 StationSearchViewController에서 셀을 눌렀을 때 해당 화면으로 이동하도록 설정해줘야 한다.

UITableViewDelegate가 필요하니, extension으로 빼주고 tableView 클로저 내부에서 delegate를 매핑해줬다.

```swift
extension StationSearchViewController: UITableViewDelegate {
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        let vc = StationDetailViewController()
        navigationController?.pushViewController(vc, animated: true)
    }
    
}
```

그리고 UITableViewDelegate에서는 didSelectRowAt를 작성해줬다.

클릭했을 때 화면전환이 되도록 pushViewController 메서드를 이용했다.

아 그리고 서치바 클릭 시 ***테이블 뷰가 나타나지 않는 문제***가 있었는데

단지 리로드를 안해줘서 그렇다.

searchBarTextDidBeginEditing에서 numberOfCell=10으로 설정해준 밑에 

tableView.reloadData()

해주면 다시 나온다!



https://github.com/jinyongyun/Subway_Notice_APP/assets/102133961/1b4398cf-7c71-40c2-ab79-be683d38c9c4



이제 StationDetailViewController로 돌아가서

아직 셀에 서버로부터 정보를 받아와 반영하지 못했기 때문에

navigationItem.title을 임시적으로 설정해줬다.

collectionView를 먼저 정의해서, 클로저 형태로 UICollectionViewFlowLayout도 만들어주고

대충 크기도 잡아주고(양쪽 16정도 띄고, margin도 설정), 스크롤 방향도 설정했다.

셀을 임시적으로 등록해주고 dataSource도 설정했다.

그러니 dataSource를 extension으로 빼서 구현해줘야 한다.

numberOfItemsInSection은 일단 3을 리턴하도록 해줬고, 

cellForItemAt에서는 구현할 커스텀 셀을 먼저 입력해줬다. 그리고 미리 알아볼 수 있게 .gray 색상을 바탕에 주었다. 

```swift
//
//  StationDetailViewController.swift
//  SubwayNotice
//
//  Created by jinyong yun on 1/22/24.
//

import SnapKit
import UIKit

final class StationDetailViewController: UIViewController {
    
    
    private lazy var collectionView: UICollectionView = {
        let layout = UICollectionViewFlowLayout()
        layout.estimatedItemSize = CGSize(width: view.frame.width-32.0, height: 100.0)
        layout.sectionInset = UIEdgeInsets(top: 16.0, left: 16.0, bottom: 16.0, right: 16.0)
        layout.scrollDirection = .vertical
        
        let collectionView = UICollectionView(frame: .zero, collectionViewLayout: layout)
        collectionView.backgroundColor = .systemBackground
        collectionView.register(UICollectionViewCell.self, forCellWithReuseIdentifier: "StationDetailCollectionViewCell")
       
        collectionView.dataSource = self
        
        return collectionView
    }()
    
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        navigationItem.title = "왕십리" //임시 설정
        
        view.addSubview(collectionView)
        collectionView.snp.makeConstraints { $0.edges.equalToSuperview() }
    }
    
}

extension StationDetailViewController: UICollectionViewDataSource {
    
    func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
        return 3
    }
    
    func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
        let cell = collectionView.dequeueReusableCell(withReuseIdentifier: "StationDetailCollectionViewCell", for: indexPath)
        
        cell.backgroundColor = .gray
        return cell
    }
    
}
```

![simulator_screenshot_B528C2E0-65C1-4B32-9658-568B627E7155](https://github.com/jinyongyun/Subway_Notice_APP/assets/102133961/9f823476-db14-485d-8f67-887b2ae68c4d)


시뮬레이션을 실행시켜보면

엄청 투박한 화면이 나온다….

이 역시 우리가 바라는 UI와 다르기 때문에 

커스텀 셀을 생성해주고, 몇 가지 레이아웃을 추가해줘야겠다.

위에서 미리 정의한 대로 StationDetailCollectionViewCell 파일을 새로 만들어준다.

```swift
//  StationDetailCollectionViewCell.swift
//  SubwayNotice
//
//  Created by jinyong yun on 1/23/24.
//

import SnapKit
import UIKit

class StationDetailCollectionViewCell: UICollectionViewCell {
    func setup() {
        backgroundColor = .gray
        
    }
    
    
}
```

그럼 기존에 StationDetailViewController에서 임시적으로 일반 UICollectionViewCell을 리턴해줬던 부분을 고칠 수 있다.

```swift
collectionView.register(StationDetailCollectionViewCell.self, forCellWithReuseIdentifier: "StationDetailCollectionViewCell")

...

func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
        let cell = collectionView.dequeueReusableCell(withReuseIdentifier: "StationDetailCollectionViewCell", for: indexPath) as? StationDetailCollectionViewCell
        
        cell?.setup()
        
        return cell ?? UICollectionViewCell()
    }
```

해당 메서드들을 수정한다.

커스텀 셀과 CollectionView를 연결했으니, 이제 커스텀 셀을 꾸며보도록 하자

```swift
//
//  StationDetailCollectionViewCell.swift
//  SubwayNotice
//
//  Created by jinyong yun on 1/23/24.
//

import SnapKit
import UIKit

class StationDetailCollectionViewCell: UICollectionViewCell {
    
    private lazy var lineLabel: UILabel = {
        let label = UILabel()
        label.font = .systemFont(ofSize: 15.0, weight: .bold)

        return label
    }()
    
    private lazy var remainTimeLabel: UILabel = {
        let label = UILabel()
        label.font = .systemFont(ofSize: 15.0, weight: .medium)
        
        return label
    }()
    
    
    func setup() {
        layer.cornerRadius = 12.0
        layer.shadowColor = UIColor.black.cgColor
        layer.shadowOpacity = 0.2
        layer.shadowRadius = 10

        backgroundColor = .systemBackground

        [lineLabel, remainTimeLabel].forEach { addSubview($0) }
        
        lineLabel.snp.makeConstraints {
            $0.leading.equalToSuperview().inset(16.0)
            $0.top.equalToSuperview().inset(16.0)
        }
        
        remainTimeLabel.snp.makeConstraints {
            $0.leading.equalTo(lineLabel)
            $0.top.equalTo(lineLabel.snp.bottom).offset(16.0)
            $0.bottom.equalToSuperview().inset(16.0)
        }
        
        lineLabel.text = "한양대 방면"
        remainTimeLabel.text = "뚝섬 도착"
        
    }
    
    
}
```

![simulator_screenshot_11A6605A-C610-47F9-A78C-2B5D9E9158E3](https://github.com/jinyongyun/Subway_Notice_APP/assets/102133961/6e1a3ebb-f513-4620-b072-5b3cddba5f4a)


어느 방면인지 표시해주는 lineLabel과 몇 분 남았는지 표시해주는 remainTimeLabel을 정의해줬다.

cellForItemAt에서 호출해줬던 setup()에서 addSubview 해주고 각 라벨의 레이아웃을 지정해줬다.

임시적으로 라벨 값을 넣어주어 시뮬레이션에서 알아볼 수 있도록 했다.

setup 부분에서 레이어를 만져줘서 

거슬리던 UI를 깔끔하게 바꿔줬다.

일단 컬렉션 뷰 UI까지 구현했다. 

이어서 서버에서 데이터를 받아와서 화면에 뿌려주고, 새로 업데이트가 발생했을 때 화면을 아래로 내려 리로드하는 동작을 구현해보려 한다.

여기서 이용해 볼 것은 UIRefreshControl이다. [머니뭐니] 프로젝트에서 사용했던 기억이 난다.

UIRefreshControl은 단독적으로 사용되기 보다는 UICollectionView와 같이 어딘가에 대입되어서 사용되는 경우가 빈번하다.

구현이 복잡하지는 않다. 액션 메서드만 구현해주면 작동하는 아주아주 편리한 녀석이다.

```swift
final class StationDetailViewController: UIViewController {
    private lazy var refreshControl: UIRefreshControl = {
       let refreshControl = UIRefreshControl()
        refreshControl.addTarget(self, action: #selector(fetchData), for: .valueChanged)
        return refreshControl
    }()
    
    @objc func fetchData(){
        print("Refresh data!")
        refreshControl.endRefreshing()
    }
    
    private lazy var collectionView: UICollectionView = {
         ...
        collectionView.refreshControl = refreshControl
         ...
    }()

...
```

이런식으로 refreshControl 하나 만들어 준 다음에

버튼처럼 refreshControl에 만들어 준 fetchData() selector를 addTarget 해주고

이걸 collectionView에 이미 있는 프로퍼티인 refreshControl에 넣어주면 끝!



https://github.com/jinyongyun/Subway_Notice_APP/assets/102133961/3591afe7-31fb-43ca-99d0-e8d2886d1d9b



## 지하철 도착 정보를 가져오는 네트워크 통신 구현하기

사용할 API는 모두 서울 열린 광장의 공공 API 사용한다.

지하철 검색에는 지하철 역명을 바탕으로 검색할 수 있게 해주는 API

[서울시 지하철역 정보 검색 (역명)](https://data.seoul.go.kr/dataList/OA-121/S/1/datasetView.do)

[http://openAPI.seoul.go.kr:8088/(인증키)/xml/SearchInfoBySubwayNameService/1/5/동대문역사문화공원/](http://openapi.seoul.go.kr:8088/sample/xml/SearchInfoBySubwayNameService/1/5/%EB%8F%99%EB%8C%80%EB%AC%B8%EC%97%AD%EC%82%AC%EB%AC%B8%ED%99%94%EA%B3%B5%EC%9B%90/)

위에는 샘플 url인데 역 이름을 가장 마지막에 두어 request 한 것을 알 수 있다.

인증키가 원래 필요한데, 우리는 샘플 앱이기 때문에 그냥 sample을 넣어 진행하면 된다!

```swift
<?xml version="1.0" encoding="UTF-8"?>
<SearchInfoBySubwayNameService>
<list_total_count>3</list_total_count>
<RESULT>
<CODE>INFO-000</CODE>
<MESSAGE>정상 처리되었습니다</MESSAGE>
</RESULT>
<row>
<STATION_CD>0319</STATION_CD>
<STATION_NM>종로3가</STATION_NM>
<LINE_NUM>03호선</LINE_NUM>
<FR_CODE>329</FR_CODE>
</row>
<row>
<STATION_CD>2535</STATION_CD>
<STATION_NM>종로3가</STATION_NM>
<LINE_NUM>05호선</LINE_NUM>
<FR_CODE>534</FR_CODE>
</row>
<row>
<STATION_CD>0153</STATION_CD>
<STATION_NM>종로3가</STATION_NM>
<LINE_NUM>01호선</LINE_NUM>
<FR_CODE>130</FR_CODE>
</row>
</SearchInfoBySubwayNameService>
```

예제를 보면 다음과 같은 형식으로 response가 날라오는데

우리가 여기서 필요한 건, row 영역의 value이다.

이번 네트워크 통신에 필요한 건 Alamofire + Codable 이다.

URLSession은 저번에 해봤기에 이번에는 Alamofire를 선택했다.

우선 네트워크 통신을 위해 info.plist 부터 수정해준다.

아래의 App Transport Security Settings를 추가해주고

하위 요소로 Allow Arbitrary Loads 값을 YES로 바꿔준다!
<img width="589" alt="스크린샷 2024-01-23 오전 10 34 04" src="https://github.com/jinyongyun/Subway_Notice_APP/assets/102133961/22b26308-3b87-499e-a948-4d52cdf08cb2">


그리고 우리가 필요한 response 값을 Codable로 정의해주도록 하자.

개인적으로 복잡한 모델을 정의할 땐 가장 하위부터 만드는 것이 조금 편한 것 같다.

```swift
//
//  StationResponseModel.swift
//  SubwayNotice
//
//  Created by jinyong yun on 1/23/24.
//

import Foundation

struct StationResponseModel: Decodable {
    
    var stations: [Station] { searchInfo.row }
    
    private let searchInfo: SearchInfoBySubwayNameServiceModel
    
    enum CodingKeys: String, CodingKey {
        case searchInfo = "SearchInfoBySubwayNameService"
    }
    
    struct SearchInfoBySubwayNameServiceModel: Decodable {
        var row: [Station] = []
        
    }
    
}

struct Station: Decodable {
    let stationName: String
    let lineNumber: String
    
    enum CodingKeys: String, CodingKey {
        case stationName = "STATION_NM"
        case lineNumber = "LINE_NUM"
    }
    
}
```

코드를 천천히 살펴보면 이해할 수 있다! 대략적인 구조는 SearchInfoBySubwayNameServiceModel이 있고 이 안에 Station 배열이 있는 구조이다. 여기서 SearchInfoBySubwayNameServiceModel 타입인 searchInfo를 private으로 숨기고 stations란 Station 배열에 바로 접근할 수 있도록 변수를 만들어줘서 사용자가 

StationResponseModel().searchInfo.row

원래 이렇게 접근해야 하는 걸

StationResponseModel().stations

이렇게 쉽게 접근하도록 해줬다. 우리가 원하는 걸 보여주고, 원하지 않는 걸 숨기는 게 캡슐화 즉 private의 장점이 아니겠는가!

모델을 완성했으니 이제 Alamofire를 사용해 실질적으로 request하는 부분을 구현해보도록 하자.

```swift
class StationSearchViewController: UIViewController {
    
...
    

    override func viewDidLoad() {
        super.viewDidLoad()
        setNavigationItems()
        setTableViewLayout()
        requestStationName()
    }
    
  ...

    private func requestStationName() {
        let urlString = "http://openapi.seoul.go.kr:8088/sample/json/SearchInfoBySubwayNameService/1/5/서울역"        
        AF.request(urlString.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? "")
            .responseDecodable(of: StationResponseModel.self) { response in
                guard case .success(let data) = response.result else {return}
                
                print(data.stations)
            }
            .resume()
        
    }
}
```

위와 같이 requestStationName 메서드에서 일단 서울을 검색해보도록 url을 구성했다.

그 다음 AF.request를 사용해서 url을 넘겨 request를 날리도록 해준다.

이때 주의해야 할 점이 우리가 작성한 url에서 마지막에 한글로 “서울”이라고 보내면

이게 깨져서 특수문자로 변환되어 서버에 전달된다.

서버에서 다시 조합된다 하더라도 서울이란 단어가 제대로 완성되지 않을 확률이 높기 때문에

**urlString.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? ""**

구문을 사용해서 String을 한 번 안전하게 감싸줘야 한다.

영어 아닌 값으로 request 할 때는 꼭! **addingPercentEncoding 해주기**

꼭 responseDecodable를 붙여주어, 받은 데이터를 우리가 만든 모델로 디코드 시켜줘야 읽을 수 있다.

guard case문을 사용해 성공했을 때만 해당 데이터를 읽어서 stations(Station 배열)를 출력하도록 해줬다.

**그리고 꼭!! 잊지말고 resume() 붙이기!!!**

<img width="1252" alt="스크린샷 2024-01-23 오전 11 11 41" src="https://github.com/jinyongyun/Subway_Notice_APP/assets/102133961/57b6ac6e-d257-42a3-b186-3f9d5aec325a">


시뮬레이션을 실행시켜보면 다음과 같이 제대로 데이터가 도착한 것을 알 수 있다.

다음 API를 구현해보자.

다음에 구현할 API는 바로 지하철역을 검색한 뒤, 셀을 클릭했을 때 나타나는 **서울시 지하철 실시간 도착정보**이다. 

[서울시 지하철 실시간 도착정보](https://data.seoul.go.kr/dataList/OA-12764/F/1/datasetView.do)

response 하는 값이 굉장히 많지만 당연히 이 중에서 골라 데이터를 뽑아내 줄 것이다.

```swift
//
//  StationArrivalDataResponseModel.swift
//  SubwayNotice
//
//  Created by jinyong yun on 1/23/24.
//

import Foundation

struct StationArrivalDataResponseModel: Decodable {
    
    var realtimeArrivalList: [RealTimeArrival] = []
    
    struct RealTimeArrival: Decodable {
        let line: String // ~행
        let remainTime: String // 도착까지 남은 시간
        let currentStation: String // 현재 위치
        
        enum CodingKeys: String, CodingKey {
            case line = "trainLineNm"
            case remainTime = "arvlMsg2"
            case currentStation = "arvlMsg3"
            
        }
    }
}
```

모델은 아까보다 더욱 간단하다.

모델을 완성했으면 StationDetailViewController에서 request하는 메서드를 만들어보자.

기존에 UIRefreshControl을 위해 작성했던 fetchData() selector 메서드에 request 하는 내용을 작성하려고 한다. 아까와 똑같은 구문이다.

```swift
@objc func fetchData(){
       
        //refreshControl.endRefreshing()
        
        let urlString = "http://swopenapi.seoul.go.kr/api/subway/sample/json/realtimeStationArrival/0/5/왕십리"
        
        AF.request(urlString.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? "")
            .responseDecodable(of: StationArrivalDataResponseModel.self) { resposne in
                guard case .success(let data) = resposne.result else {return}
                
                print(data.realtimeArrivalList)
            }
            .resume()
        
    }
```

화면에 바로 나타날 수 있게 viewDidLoad에서도 fetchData를 불러주고

시뮬레이터를 실행시켜보면
<img width="1250" alt="스크린샷 2024-01-23 오전 11 27 03" src="https://github.com/jinyongyun/Subway_Notice_APP/assets/102133961/544803a8-d08c-46ea-8994-aab1c118858f">



두 종류의 데이터가 모두 잘 도착하는 것을 알 수 있다.

여기서 조금 문제랄까 헷갈렸던 부분이 있었는데

두 번째 API인 **서울시 지하철 실시간 도착정보**에서 특이한 점이 url에 “서울역”이라고 request를 보내면 실패하고, “서울” 이렇게 보내면 request가 성공한다.

그래서 유저가 서울을 치던, 서울역을 치던 모두 성공하게 하기 위해

역 이름을 변수로 빼서 조정하는 단계가 필요할 것 같다.

```swift
let stationName = "서울역"
let urlString = "http://swopenapi.seoul.go.kr/api/subway/sample/json/realtimeStationArrival/0/5/\(stationName.replacingOccurrences(of: "역", with: ""))" 
```

이렇게 설정해줬는데 

stationName.replacingOccurrences(of: "역", with: "")

를 사용해서, 만약 역 이름에 “역” 단어가 있다면 이를 빈 문자열로 대체하도록 해줬다.

```swift
@objc private func fetchData(){
       
        //refreshControl.endRefreshing()
        
        let stationName = "서울역"
        let urlString = "http://swopenapi.seoul.go.kr/api/subway/sample/json/realtimeStationArrival/0/5/\(stationName.replacingOccurrences(of: "역", with: ""))"
        
        AF.request(urlString.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? "")
            .responseDecodable(of: StationArrivalDataResponseModel.self) { resposne in
                guard case .success(let data) = resposne.result else {return}
                
                print(data.realtimeArrivalList)
            }
            .resume()
        
    }
```

그럼 여기서 우리가 은근슬쩍 주석처리했던   //refreshControl.endRefreshing()

이녀석의 위치는 대체 어디일까?

보통 앱을 사용할 때 refreshing이 끝나야 하는 타이밍은 서버의 요청이 완료되고, 뷰를 새로 그려야 하는 시점이다.

이걸 코드로 살펴보면 만약 print(data.realtimeArrivalList) 뒤에 endRefreshing이 오게 되면

실패했을 때는 refreshControl이 절대 멈추지 않는다(?!)

하지만 실패와 성공에 상관없이 일단 reponse가 오면 refreshing이 완료되는 것을 원하기 때문에 

guard case 문, 즉 분기 전에 endRefreshing을 설정해줬다.

```swift
@objc private func fetchData(){
        
        let stationName = "서울역"
        let urlString = "http://swopenapi.seoul.go.kr/api/subway/sample/json/realtimeStationArrival/0/5/\(stationName.replacingOccurrences(of: "역", with: ""))"
        
        AF.request(urlString.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? "")
            .responseDecodable(of: StationArrivalDataResponseModel.self) { [weak self] resposne in
                
                self?.refreshControl.endRefreshing()
                
                guard case .success(let data) = resposne.result else {return}
                
                print(data.realtimeArrivalList)
            }
            .resume()
        
    }
```

## 지하철 도착 정보 데이터 화면에 표시하기

자동 완성 기능을 구현하기 위해 

서치바 입력 시 나타나는 키보드의 글씨 입력을 델리게이트 메서드로 인식하고 

바로바로 서버에 request를 요청하는 타이밍을 잘 조절해 줘야 한다.

또 키보드가 한 번 눌려졌을 때 서버에 리퀘스트를 하고, 서버에서 받아온 response 값을 바로바로 테이블 뷰에 표시를 해줘야 한다.

```swift
override func viewDidLoad() {
        super.viewDidLoad()
        setNavigationItems()
        setTableViewLayout()
        requestStationName()
    }
```

우리는 지금까지 StationSearchViewController의 viewDidLoad()에서 무조건적으로 

 requestStationName()를 호출하도록 했다.

이번에는 UISearchBarDelegate 메서드를 사용해 글자를 받자마자 즉 유저가 키보드에 입력할 때

API 통신을 진행하도록 구현해보겠다.

유저가 SearchBar에 한글자씩 입력했을 때 불려지는 메서드가 바로

UISearchBarDelegate의 textDidChange 메서드이다.

```swift
private func requestStationName(from stationName: String) {
        let urlString = "http://openapi.seoul.go.kr:8088/sample/json/SearchInfoBySubwayNameService/1/5/\(stationName)"
        
        AF.request(urlString.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? "")
            .responseDecodable(of: StationResponseModel.self) { response in
                guard case .success(let data) = response.result else {return}
                
                print(data.stations)
            }
            .resume()
        
    }

extension StationSearchViewController: UISearchBarDelegate {
 ...
    
    func searchBar(_ searchBar: UISearchBar, textDidChange searchText: String) {
        requestStationName(from: searchText)
    }
}
```

기존에 만들었던 requestStationName 메서드가 다음과 같이 searchText를 인자로 받을 수 있도록 하고, textDidChange 메서드 내에서 searchBar의 텍스트가 변할 때마다 서버에 해당 텍스트로 request를 날릴 수 있도록 해줬다.

우리가 서버 통신을 해주긴 했는데, 그럼 어디서 비동기 처리가 된 것일까? 우리는 분명 앞에서 클로저를 사용해서 비동기 처리를 한다고 했다.

```swift
 AF.request(urlString.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? "")
            .responseDecodable(of: StationResponseModel.self) { response in
                guard case .success(let data) = response.result else {return}
                
                print(data.stations)
            }
            .resume()
```

바로 request 처리를 해줬던 위의 부분이다. 

이 부분에서 기다리지 않고 가져온 정보들을 바로바로 프린트 하는 것 자체가 벌써 비동기 처리가 구현되어 있다는 것이다!

생각해보면 서버에서 결과값이 오고 파싱이 끝나는 타이밍까지 앱이 꺼져있거나, holding 되어 있지 않기 때문에 병렬적으로 처리되는 무언가가 있다는 것이다.

그리고 우리는 그동안 numberOfCell로 셀의 개수를 (조잡하게) 핸들링 하고 있었다.

이 프로퍼티를 삭제하고 제대로 된 stations라는 Station의 배열 프로퍼티를 만들어준다.

```swift
extension StationSearchViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return stations.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = UITableViewCell(style: .subtitle, reuseIdentifier: "Cell")
        let station = stations[indexPath.row]
        cell.textLabel?.text = station.stationName
        cell.detailTextLabel?.text = station.lineNumber
        
        return cell
    }
    
    
}
```

이제 기존 numberOfCell 자리를 stations 가 채워주도록 수정해주면 된다.

numberOfRowsInSection의 리턴값을 stations 배열의 count로 해주고

cellForRowAt에서도 제대로 된 값을 넘겨줄 수 있다.

```swift
private func requestStationName(from stationName: String) {
        let urlString = "http://openapi.seoul.go.kr:8088/sample/json/SearchInfoBySubwayNameService/1/5/\(stationName)"
        
        AF.request(urlString.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? "")
            .responseDecodable(of: StationResponseModel.self) { [weak self] response in
                guard let self = self,
                        case .success(let data) = response.result else {return}
                
                self.stations = data.stations
                self.tableView.reloadData()
            }
            .resume()
        
    }
```

requestStationName 메서드 안에서도 받아온 stations 모델을 StationSearchViewController의 stations에 대입해주고, 클로저 내부에서 self를 사용했기 때문에 강한 참조를 방지하기 위해 [weak self]와 self 옵셔널 바인딩을 해준다.

![simulator_screenshot_889CFC94-DE86-4C3A-8D79-4D0324D20F13](https://github.com/jinyongyun/Subway_Notice_APP/assets/102133961/057747b5-9043-4d3e-a59b-0ce2442a68fe)


시뮬레이션을 실행시켜주면

다음과 같이 잘 나오는 것을 알 수 있는데 

여기서 하나 문제점이 보인다.

cancel을 눌러줬을 때 입력값이 초기화가 되어야 하는데, 다시 입력하려고 들어가보면 테이블 뷰에 이전의 값이 그대로 남아있다는 문제이다.

```swift
func searchBarTextDidEndEditing(_ searchBar: UISearchBar) {
        tableView.isHidden = true
        stations = []
    }
```

우리가 초반에 작성했던 searchBarTextDidEndEditing에 다음과 같이 stations가 빈 배열이 되도록 초기화 해주면 해결된다!

지하철 역 자동완성은 완성했으니, 지하철 도착 정보 데이터를 화면에 표시해보도록 하자.

가장 먼저 해줘야 할 부분이 검색 결과가 표시되고, 해당 셀을 선택했을 때

정보를 지하철 도착 정보 화면 즉 StationDetailViewController에 넘겨주는 부분이다.

```swift
final class StationDetailViewController: UIViewController {
    
    private let station: Station
...

init(station: Station) {
        self.station = station
        super.init(nibName: nil, bundle: nil)
    }
```

station을 담을 프로퍼티와 init 메서드를 만들어준다.

StationSearchViewController로 돌아가서 didSelectRowAt에서 indexPath에 맞는 station 값을 넘겨준다.

```swift
extension StationSearchViewController: UITableViewDelegate {
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        let station = stations[indexPath.row]
        let vc = StationDetailViewController(station: station)
        navigationController?.pushViewController(vc, animated: true)
    }
    
}
```

StationDetailViewController에서 기존에 왕십리로 선언해줬던 navigationItem.title을 stationName으로 바꿔준다.

```swift
override func viewDidLoad() {
        super.viewDidLoad()
        
        navigationItem.title = station.stationName
        
        view.addSubview(collectionView)
        collectionView.snp.makeConstraints { $0.edges.equalToSuperview() }
        
        fetchData()
    }
```

![simulator_screenshot_DF56E533-C443-43CD-8AB1-B13A398786B2](https://github.com/jinyongyun/Subway_Notice_APP/assets/102133961/b2072496-a088-4e33-93df-22a607fd4672)


다음과 같이 시뮬레이션에 안국을 검색하면, 네비게이션 아이템의 타이틀이 잘 나타나는 것을 알 수 있다.

StationDetailViewController의 fetchData 메서드에서도 제대로 된 역 이름을 요청할 수 있도록 해보자.

```swift
final class StationDetailViewController: UIViewController {
    
    ...
    
    private var realtimeArrivalList: [StationArrivalDataResponseModel.RealTimeArrival] = []

    ...
    @objc private func fetchData(){
        
        let stationName = station.stationName
        let urlString = "http://swopenapi.seoul.go.kr/api/subway/sample/json/realtimeStationArrival/0/5/\(stationName.replacingOccurrences(of: "역", with: ""))"
        
        AF.request(urlString.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? "")
            .responseDecodable(of: StationArrivalDataResponseModel.self) { [weak self] resposne in
                
                self?.refreshControl.endRefreshing()
                
                guard case .success(let data) = resposne.result else {return}
                
                self?.realtimeArrivalList = data.realtimeArrivalList
                self?.collectionView.reloadData()
            }
            .resume()
        
    }

...

extension StationDetailViewController: UICollectionViewDataSource {
    
    func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
        return realtimeArrivalList.count
    }
    
    func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
        let cell = collectionView.dequeueReusableCell(withReuseIdentifier: "StationDetailCollectionViewCell", for: indexPath) as? StationDetailCollectionViewCell
        
        let realTimeArrival = realtimeArrivalList[indexPath.row]
        cell?.setup(with: realTimeArrival)
        
        return cell ?? UICollectionViewCell()
    }
    
}

```

```swift
func setup(with realTimeArrival: StationArrivalDataResponseModel.RealTimeArrival) {
        layer.cornerRadius = 12.0
        layer.shadowColor = UIColor.black.cgColor
        layer.shadowOpacity = 0.2
        layer.shadowRadius = 10
        
        backgroundColor = .systemBackground
        
        [lineLabel, remainTimeLabel].forEach { addSubview($0) }
        
        lineLabel.snp.makeConstraints {
            $0.leading.equalToSuperview().inset(16.0)
            $0.top.equalToSuperview().inset(16.0)
        }
        
        remainTimeLabel.snp.makeConstraints {
            $0.leading.equalTo(lineLabel)
            $0.top.equalTo(lineLabel.snp.bottom).offset(16.0)
            $0.bottom.equalToSuperview().inset(16.0)
        }
        
        lineLabel.text = realTimeArrival.line
        remainTimeLabel.text = realTimeArrival.remainTime
        
    }
```

 private var realtimeArrivalList: [StationArrivalDataResponseModel.RealTimeArrival] = []
를 만들어주고, fetchData()에서                 

self?.realtimeArrivalList = data.realtimeArrivalList 를 해줬다. 즉 서버 데이터의 realtimeArrivalList가 우리 컨트롤러의 프로퍼티에 들어가고

해당 리스트의 데이터를 화면에 나타내기 위해 DataSource 또한 수정해줬다.

## 실제 구동 화면



https://github.com/jinyongyun/Subway_Notice_APP/assets/102133961/e2089cc6-62b0-48eb-9d2a-066b17813975

