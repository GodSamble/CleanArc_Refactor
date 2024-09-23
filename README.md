# CleanArc_Refactor
## MVC -> MVVM + Clean Architecture(일부 적용완료) (for.테스트/유지보수 용이)


<img width="820" alt="스크린샷 2024-09-19 오전 9 15 32" src="https://github.com/user-attachments/assets/0d3ee628-7283-499b-9a2e-5d4d42650c81">

<img width="823" alt="스크린샷 2024-09-19 오전 9 15 59" src="https://github.com/user-attachments/assets/ab970657-2e27-4853-bb6a-cc7acb8e95d0">

## Domain
> Entity
- 서비스에 쓰이는 데이터를 정의합니다. 뷰 구성을 디퍼블 데이터소스를 이용했기에, Hashable을 채택함을 감안 부탁드립니다. 
``` swift

struct Thumbnail: Hashable {
    let images: [String]
}

struct TitleInfo: Hashable {
    let brandName: String
    let itemName: String
    let itemPrice: Int
    let itemSalePrice: Int?
    let saleRate: Int?
    let avgStarRating: Double
    let avgStarCounting: Int!
    let expectedSavePoint: Int
    let expectedSavePointRate: Int
    
    init(_ data: ItemDetailResDto){
        brandName = data.item.brand.name
        itemName = data.item.name
        itemPrice = data.item.price
        itemSalePrice = data.item.salePrice
        saleRate = data.item.saleRate
        avgStarRating = data.avgStarRating
        avgStarCounting = data.avgStarRatingCount
        expectedSavePoint = data.expectedSavePoint
        expectedSavePointRate = data.expectedSavePointRate
    }
}
```
> UseCase
- Entity가 사용되는 시나리오를 정의합니다. 
``` swift

protocol ItemUseCaseProtocol {
    func excute() -> Observable<TitleInfo>
}

final class ItemUseCase : ItemUseCaseProtocol {
    
    private let itemRepository: ItemRepositoryInterface
    init(itemRepository: ItemRepositoryInterface) {
        self.itemRepository = itemRepository
    }
    
    func excute() -> RxSwift.Observable<TitleInfo> {
        return self.itemRepository.data()
    }
}
```
> Repository Interface
- 클린 아키텍처 다이어그램의 더 안쪽에 위치한 Domain 영역이 Data 영역의 Repository에 접근하기 위해 `의존성 역전`를 통해 레퍼지토리에 접근하기 위한 방법을 제공합니다.
``` swift


protocol ItemRepositoryInterface {
    func data() -> Observable<TitleInfo>
}
```
## Data
> Repository
- 데이터를 직접적으로 획득합니다. 데이터를 획득하는 방법을 확장성있도록 하기 위해 사용됩니다.
``` swift
final class ItemRepository: ItemRepositoryInterface {
    private let service: ItemApiManager //MARK: 서비스 끌어옴
    init(service: ItemApiManager) {
        self.service = service
    }
    
    func data() -> RxSwift.Observable<TitleInfo> {
        self.service.getItemDetail(id: 100) 
        
    func data() -> Observable<HomeGetDTO> {
             self.service.getService(from: Config.baseURL + "api/home", isUseHeader: true)
         }
}
```
## Presentation
> View
- 기존 MVC 패턴 시 UIViewController의 `loadView` 생명주기에 UIView를 바꿔주어 ViewController의 레이아웃 코드를 최소화하고자 했습니다.
``` swift
    // MARK: - Life Cycle
    override func loadView() {
        super.loadView()
        historyView = HistoryView(frame: self.view.frame)
        self.view = historyView
    }
```

> ViewModel
- SwiftUI와 UIKit에 모두 비동기프로그래밍을 적용하고자 `Combine`을 학습 중입니다.
- UIKit의 ViewModel은 `Input/Output Modeling`을 통해 View로부터 전달된 이벤트는 Input, View로 전달할 데이터는 Output을 통해 Binding합니다.
``` swift

class ItemDetailViewModel: CommonViewModelType {
    
    let itemUseCase: ItemUseCaseProtocol
    init(itemUseCase: ItemUseCaseProtocol){
        self.itemUseCase = itemUseCase
    }

    func transform(input: Input) -> Output {
        let data = input.viewDidLoad
            .flatMap{ ItemService.shared.getItemDetail(id: self.itemId) }
            .do(onNext: { data in
                self.itemDetail = data
            })
        
        let dataSource = data.map { dto -> [DetailInfoSectionItem] in
            var result = [DetailInfoSectionItem]()
            SectionLayoutKind.allCases.forEach { section in
                switch section {
                case .ItemThumbnail:
                    let thumbnail = Thumbnail(images: dto.item.images)
                    result.append(.Thumbnail(thumbnail)) // MARK: 기존 방식
                case .ItemTitleInfo:
                    let data = self.itemUseCase.excute()
                                    .subscribe(onNext: { titleInfo in
                                        result.append(.TitleInfo(titleInfo)) // MARK: UseCase를 만들어서 주입한 것.
                                    })
                }
            }
>
```
