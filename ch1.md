# 0. 시작 

본 챕터의 목표는 수신용 이메일 주소 만들기 입니다. AWS에서 도메인을 등록하고 수신 이메일 주소를 생성하는(이메인 수신 규칙을 생성하는) 과정을 순서대로 설명하겠습니다. 

제일 먼저 할 일은 ~~돈을 쓰는~~ 도메인을 등록하는 것 입니다.  다음으로, Simple Email Service(SES)를 이용하여, 등록한 도메인으로 이메일 주소(수신용)를 생성할 것 입니다. 도메인을 등록해서, 수신 이메일 주소를 설정하는 과정까지 진행해봅시다. 

AWS Console에서는 도메인을 등록, 네임서버 설정, 수신 이메일 생성 등을 몇 번의 타이핑과 클릭으로 쉽게 할 수 있습니다. 

# 1. 도메인 등록 

<a href="https://console.aws.amazon.com/route53" target="_blank">Route 53 Console</a>로 이동합니다. 아쉽게도 Route 53 페이지는 한글화되지 않았네요. 친절하게 ‘Register Domain’ 항목이 처음화면에 있습니다. 본인이 사용하고 싶은 도메인 이름을 입력하고 최상위 도메인(.com, .net 등)을 선택합니다.  Check를 선택합니다. 

저는 결제내역(영수증)을 수집할 것 이기 때문에, myreceipt.net 이름으로 도메인을 결정하였습니다. 

![1-1. Route 53 Console](https://github.com/0kim/myReceipts/blob/master/images/1/1-1.png?raw=true)


만들고 싶은 도메인 네임이 이미 등록되어있어 선택할 수 없다면, 다른 이름을 입력하고 Check로 확인해보세요. 최상위 도메인으로 .kim도 있으니, 본인의 성이 김씨이면 .kim 도메인도 고려해보세요. 참고로, 보이는 가격에는 VAT가 포함되어있지 않습니다. 결제시에 10% 가산됩니다. 

'Add to cart' 하여 장바구니에 넣고, 페이지 하단에 'Continue’로 다음 화면으로 넘어 갑니다 
![1-2. Domain search](https://github.com/0kim/myReceipts/blob/master/images/1/1-2.png?raw=true)


도메인 등록을 위한 개인정보를 입력합니다. 1년 단위 등록이 가능하며, 우측의 Shopping cart 항목에서 등록 년 수를 변경할 수 있습니다. 

![1-3. Contact detail](https://github.com/0kim/myReceipts/blob/master/images/1/1-3.png?raw=true)

입력한 내용과 ‘이용 약관(Terms and Conditions)’을 잘 읽어보세요. AWS Domain Name Registration Agreement 항목에 체크하세요. 본인의 계정 이메일 주소로 도메인 확인 메일이 발송되며, 해당 이메일을 확인하고 확인하세요. 


마지막으로, 결제를 완료하면 환불 및 취소는 불가하니 이용하고자 하는 도메인 이름이 맞는지 다시 한번 확인하시고 진행하시기 바랍니다. 그리고, 등록된 도메인은 매년 자동 갱신됩니다. 

Complete Purchase로 구매를 진행합니다. 카드 해외 결제 승인이 되며, 결제가 진행됩니다. 
![1-4. Verify & Purchase](https://github.com/0kim/myReceipts/blob/master/images/1/1-4.png?raw=true)

여기까지 도메인 등록을 마쳤습니다. 도메인을 등록하면, 실제 등록까지 시일이 소요됩니다. 저의 경우에 15분 정도 소요되었고, 길면 3일까지 걸릴 수 있습니다. ‘Pending requests’ 에서 등록 상태를 확인할 수 있습니다. 실제 등록이 완료되면, ‘Registered domains’ 항목에서 확인할 수 있습니다. 등록이 완료될 때 까지 기다리고 다음 절차를 진행하시기 바랍니다. 
![1-5. Thank you for registering](https://github.com/0kim/myReceipts/blob/master/images/1/1-5.png?raw=true)


# 2. 호스팅 영역(Hosted Zone) 호

다시, Route 53 Console의 처음 화면으로 이동합니다. 아래 그림 처럼 등록이 완료되었네요. DNS Management 항목의 Hosted zones 숫자가 1 증가했습니다. Hosted zone은 도메인 관리를 위한 도메인 네임 서버라고 보시면 됩니다. 

1개의 Hosted Zone에 대해 $0.5 씩 과금되며, 1백만 쿼리 당 $ 0.4가 과금됩니다. 상세한 과금정책은 여기(<a href="https://aws.amazon.com/ko/route53/pricing/" target="_blank">https://aws.amazon.com/ko/route53/pricing/</a>)를 확인하시기 바랍니다.

Hosted zones(호스팅 영역)을 선택하여 설정 내용을 확인합니다. 
![1-6. Hosted zone](https://github.com/0kim/myReceipts/blob/master/images/1/1-6.png?raw=true)


myreceipt.net 도메인이 잘 등록되어있네요. 
클릭해서 세부 내용을 확인하고 도메인 네임 서버의 규칙을 추가/변경할 수 있습니다. 예를들어 웹서버의 IP와 도메인 네임을 연결하거나, 메일서비스와 도메인을 연결할 때, 여기에서 설정을 변경할 수 있습니다. 

그러나, 우리가 여기서 별도로 설정할 것은 없습니다. 왜냐하면, 수신 이메일 주소를 만들 때, 필요한 정보를 자동으로 입력할 수 있기 때문입니다. 참 돈쓰니 편리하네요. 

![1-7. Hosted zone and domain list](https://github.com/0kim/myReceipts/blob/master/images/1/1-7.png?raw=true)


# 3. 수신용 이메일 주소 등록 하기 

이제 Simple Email Service(SES)를 이용하여 수신용 이메일 주소를 등록해 봅시다. 이메일 주소라고 했지만, 생각했던 Gmail, 네이버와 같은 이메일 서비스를 생각하셨다면 놀라셨을 것 입니다. 흔히 생각하는 Gmail/네이버 메일 서비스와는 큰차이가 있습니다. 이런 메일 서비스는 이메일 서버+클라이언트의 조합이죠. 이에 반해서 SES는 이름 그대로 Simple한 낮은 수준의 이메일 서비스입니다. 대량 이메일 발송이나 특정 서비스의 메시지 수신 용도 등의 메시징 용도로 사용하기 좋습니다. SES에 대해 자세한 내용은 여기를 참고하세요. (<a href="https://aws.amazon.com/ko/ses/details/" target="_blank">https://aws.amazon.com/ko/ses/details/</a>)

SES Console 화면으로 이동합니다. SES는 한국 리전에 없으며, N. Virginia, Oregon 및 Ireland. 현재는 세 곳에서만 사용할 수 있습니다. 우리는 N. Virginia(us-east-1)을 사용하기로 합니다. 

* SES Console - <a href="https://console.aws.amazon.com/ses/home?region=us-east-1" target="_blank">https://console.aws.amazon.com/ses/home?region=us-east-1</a> 

Email Receiving(이메일 수신) 항목의 Rule Sets(규칙 세트)를 선택합니다. 

![1-8. Simple Email Service]https://github.com/0kim/myReceipts/blob/master/images/1/1-8.png?raw=true)


이전에 생성된 이메일 수신 규칙이 없으므로, 아래와 같은 화면이 보입니다. 
'Create a Receipt Rule’을 눌러 새로운 이메일 수신 규칙을 만들어 봅시다. 

![1-9. Rule Sets](https://github.com/0kim/myReceipts/blob/master/images/1/1-9.png?raw=true)


저는 receipt@myreceipt.net를 수신 이메일 주소로 하기로 했습니다. 앞서서 등록한 도메인이 포함된 이메일 주소를 하나 선택하여, Recipient항목에 넣고, ‘Add Recipient’를 선택하시기 바랍니다. 

![1-10. Recipients](https://github.com/0kim/myReceipts/blob/master/images/1/1-10.png?raw=true)

Recipient항목에 등록되었습니다. 해당 도메인의 소유 확인을 위해,  'Verify domain'을 선택합니다. 

![1-11. Added Recipient](https://github.com/0kim/myReceipts/blob/master/images/1/1-11.png?raw=true)

뭔가 복잡한 내용이 나타납니다. 도메인 확인을 위한 정보와 이메일을 수신을 위한 이메일 서버 정보를 도메인 네임 서버 규칙에 입력하라는 내용입니다. 만약, AWS 이외에 다른 도메인 네임 서버를 이용하신다면, 아래 정보를 네임서버에 직접 입력하셔야 합니다. 

우리는 Route 53을 사용하니, 'Use Route 53’만 선택합니다. 두 번째 화면이 나타나며, ‘Email Receiving Record’를 반드시 체크합니다. ‘Create Records Sets’을 눌러 등록을 완료합니다. 

Next Step을 선택 후, ‘Step 2: Actions’ 항목으로 넘어갑니다. 
    
![1-12. Verify Domain](https://github.com/0kim/myReceipts/blob/master/images/1/1-12.png?raw=true)
![1-13. Create RecordSet](https://github.com/0kim/myReceipts/blob/master/images/1/1-13.png?raw=true)


두 번째, Actions 항목입니다. 이메일이 앞서 설정한 수신자(Recipient)에게 수신 될 때, 수행해야할 작업들을 기술할 수 있습니다. 이메일이 전달되면 읽어봐야겠지요? S3에 저장하도록 합니다. 

1) 'Add action' 항목에서 S3 선택하고 추가합니다. 
2) S3 bucket에서 ‘Create S3 bucket’을 선택 후, 새로운 버킷을 생성합니다. 이름은 myreceipt-bucket으로 하겠습니다. 
3) Next Step을 선택하여 다음 화면으로 넘어 갑니다. 

![1-14. Actions](https://github.com/0kim/myReceipts/blob/master/images/1/1-14.png?raw=true)

1) Rule name 항목에는 생성할 규칙이름을 입력합니다. 저는 ‘receipt-invoke-lambda’로 입력하였습니다. 
2) Rule set은 default-rule-set(Active)를 선택합니다. 
3) 그리고 하단의 ‘Next Step’을 선택합니다. 
4) Review 페이지가 나타나며, 수신 규칙 생성이 완료됩니다. 

![1-15. Rule Detail](https://github.com/0kim/myReceipts/blob/master/images/1/1-15.png?raw=true)

이메일 수신 규칙이 잘 만들어졌다면, 아래와 같은 화면을 볼 수 있습니다. 아래 규칙을 말로 풀어보면, ‘receipt@myreciept.net’ 이메일 주소로 메일이 전달되면, S3 ‘myreceipt-bucket’에 메일을 쓰는 작업을 수행하라라는 의미입니다. 


![1-16. Detail of Rule ](https://github.com/0kim/myReceipts/blob/master/images/1/1-16.png?raw=true)


그러면, 정상 동작했는지 테스트해볼까요? 생성한 수신자로 이메일을 보내봅니다. receipt@myreceipt.net으로 메일을 보냈습니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/1/1-17.png" width="300" alt="1-17. send an email">


‘myreceipt-bucket’ S3 버킷을 확인합니다. 보낸 메시지가 저장되었다면, 아래와 같은 화면을 볼 수 있습니다. 
메시지를 다운로드 받아 내가 보낸 메일이 맞는지 확인해 봅시다. 

![1-18. Mail received in S3](https://github.com/0kim/myReceipts/blob/master/images/1/1-18.png?raw=true)
<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/1/1-19.png" width="400" alt="1-19. received mail text">

여기까지 도메인 등록에서 부터 수신 이메일 주소를 생성하고 동작확인하는 과정을 모두 거쳤습니다. 아래의 3가지에서 모두 성공하였다면 이번 챕터는 Pass입니다. 

1. 도메인 등록 ‘myreceipt.net’
2. 수신 이메일 주소 생성 ‘receipt@myreceipt.net’
3. 수신 이메일 주소 동작 확인 (이메일 S3에 저장)