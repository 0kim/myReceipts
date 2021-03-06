# 0. 시작

myReceipts 서비스의 두 번째 챕터를 시작합니다. 이전 챕터에서는 AWS에서 1) 도메인을 가입/등록하고, 2) 수신 이메일 주소를 만드는 절차를 살펴보았습니다. 카드사에서 보내는 결제 이메일을 수신하는 단계까지 준비가 되었습니다. 

이번 챕터에서는 카드사에 보내준 이메일에 대해서 살펴보고, 이메일에서 추출한 데이터를 저장할 Spreadsheet를 살펴보기로 하겠습니다. 


# 1. BC카드사 카드 결제 이메일

결제 정보를 어떻게 받아올지는 본인이 편리한 방식을 찾아보기를 바랍니다. 이메일을 수신하기 위해 도메인 등록하는 일은 작지만한 비용이 드는 일이기 때문입니다. 결제 정보 수집 대상을 이메일로 국한할 필요는 없습니다. 단문 메시지라든가 수기 입력하는 방법을 이용하셔도 됩니다. 제가 사용하는 BC카드에서 이메일로 결제 정보를 안정적으로 발송해주고 있었고 사용하는 스마트폰이 아이폰이기에 기기 내에서 단문 메시지를 다루기 어렵다는 결론에 이르러서, 이메일로 부터 수집하는 서비스를 만들게 되었습니다. 본 글에서는 BC카드사의 결제시 발송하는 이메일 기준으로 설명하는점 양해 바랍니다. 

저는 총 4가지 종류의 결제 이메일을 수신합니다. 해외/국내 결제에 따른 메일에 대해 각각 승인/취소 유형으로 총 4가지 종류의 메일을 확인할 수 있습니다. 국내/해외 거래 승인 만 처리하도록 하겠습니다. 승인 취소 메일은 처리하지 않도록 합니다. 

* 국내 거래 승인
* 국내 거래 승인 취소
* 해외 거래 승인 
* 해외 거래 승인 취소 

	
| 국내거래승인 메일  |  해외거래승인 메일 |
|---|---|
| ![2-1. 국내거래승인 메일](https://github.com/0kim/myReceipts/blob/master/images/2/2-1.png?raw=true)  | ![2-2. 해외거래승인 메일](https://github.com/0kim/myReceipts/blob/master/images/2/2-2.png?raw=true)  |


위의 두 개의 이메일로 부터 추출할 정보의 공통 항목을 선택합니다. 아래 6가지의 항목을 설정하였습니다. 그리고 6개의 항목은 이메일의 항목과 각각 매칭됩니다. 

1. **날짜** - 사용일시 
2. **금액** - 사용금액
3. **통화**
    1. 국내거래승인의 경우, KRW
    2. 해외거래승인의 경우, 사용금액의 해당 국가 통화
4. **장소** - 가맹점명
5. **지출방법** - 카드명
6. **국가**
    1. 국내거래승인의 경우, 국내
    2. 해외거래승인의 경우, 해외

위의 정보대로 데이터 저장소에 스키마를 생성하고, 저장소에 이메일을 파싱해서 입력만하면 프로젝트가 끝나겠죠? 

# 2. Google Spreadsheet의 선택

Google Spreadsheet를 데이터 저장소로 결정했습니다. 향 후, 데이터 분석과 관리 관점에서는 DBMS 사용이 당연한 선택일 수 있습니다만, 본 서비스를 만들 때 상정한 몇 가지 요구사항으로 인해서 Google Spreadsheet를 쓰기로 했습니다. 그 요구사항들은 다음과 같습니다. 

1) 결제 정보를 수집해서, 현재까지의 지출 금액을 빠르게 파악하는 것이 중요하다. 
2) 그러므로, 동작하는 서비스를 빠르게 개발할 수 있어야 한다. 
3) 데스크탑 및 모바일에서 결제 내역 현황을 편리하게 확인할 수 있어야 한다. 
4) 자동 입력된 데이터를 편리하게 수정/추가/삭제 할 수 있어야 한다. 

위의 4가지를 종합했을 때, Google Spreadsheet가 최적의 선택으로 판단했습니다. 데이터를 조작하고 가시화하는데 편리하며, (Google 드라이브를 이용하여) 모바일에서도 원하는 정보를 빠르고 편리하게 접근할 수 있는 장점 때문입니다. 


# 3. 스프레드 시트의 스키마 작성

그러면, 스프레드 시트에서 칼럼을 구성해보도록 하겠습니다. 자동 수집한 정보만으로 부족함이 있어, 앞서 6가지 칼럼에 더해서 ‘세부내용’과 ‘취소여부’를 추가하였습니다. 그림은 아래와 같습니다.

![2-3. 스프레드시트 샘플 메일](https://github.com/0kim/myReceipts/blob/master/images/2/2-3.png?raw=true)

여러분께서 직접 만드셔도 되지만, 위의 스프레드시트 템플릿을 공유 드립니다. 2018년 기준이며, 아래 링크에서 시트를 확인하고 본인의 계정으로 복사하여 사용하시기 바랍니다. 템플릿 안에는 월간통계와 한 해의 지출 금액을 한눈에 파악할 수 있는 대시보드가 포함되어있습니다. 대시보드와 월간 통계는 본인의 구미에 맞게 변경해서 사용하세요. 

* 2018 My Expenses 시트 템플릿 - http://bit.ly/myexpense2018

# 4. 정리 

자 여기까지 ‘결제내역’ 이메일과 이메일 포함된 정보로 부터 Google Spreadsheet에서 스프레드 시트를 만들어보았습니다. 다음에 할 일은 결제내역 이메일에서 결제 정보를 추출하고 스프레드시트에 넣는 일이 남아있습니다. 그런데, AWS에서 어떻게 Google Spreadsheet로 데이터를 넣을까요? 다음 챕터에서는 Google API를 이용하여 Google Spreadsheet에 데이터를 추가하는 방법을 알아보겠습니다. 
