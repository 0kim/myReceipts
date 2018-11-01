# 0. 시작

Python 코드로 간단하게 Google Spreadsheet에 데이터를 추가할 수 있었습니다. S3에 이메일이 저장되면, S3에 저장된 이메일 본문을 가져와 필요한 값(날짜, 금액, 가맹점이름 등)을 추출하고, 시트추가하는 코드를 작성하겠습니다. 그리고, AWS Lambda로 배포하면되겠죠? 메일을 수신할 때 마다 Lambda에 배포된 Python 코드가 수행되며 S3에 저장된 이메일을 읽어와 파싱한 다음 저장하게 됩니다. Lambda에 올릴 코드를 준비하고, 앞서 설정하였던 AWS의 SES(Simple Email Service) 및 S3 서비스 그리고, Google Spreadsheet를 연결해보겠습니다.

# 1. Python 코드 준비 

앞서 간단하게 테스트 했던 연동 코드를 바탕으로 수정하셔도 됩니다. 이해를 돕기 위해 미리 작성한 코드를 준비하도록 하겠습니다. 

프로젝트 디렉토리를 생성하고 이동합니다. 저는 디렉토리 이름을 myReceipts_lambda로 정했습니다.

```bash
$ mkdir myReceipts_lambda
$ cd myReceipts_lambda
```

다음으로 필요한 Python 라이브러리를 pip를 이용하여 다운로드 받습니다. 프로젝트 디렉토리에서 아래 명령을 차례대로 수행합니다.

```bash
$ pip3 install gspread -t .
$ pip3 install oauth2client -t .
$ pip3 install bccard_parser -t .
```

BC카드사가 발송하는 결제 이메일의 본문 내용 중 필요한 값(결제금액, 일시, 가맹점명 등)을 추출해야합니다. bccard_parser를 이용해서 HTML로 전달되는 BC카드의 결제 내용을 파싱할 수 있습니다. 필요에 따라 별도의 파서를 작성하시거나 변경하셔도 좋습니다. 

같은 폴더에 “myreceipt.py” 이름으로 Python 코드 파일을 생성하고 편집합니다. 
```bash 
$ vi myreceipt.py
```

“myreceipt.py” 코드는 아래와 같습니다. 아래 내용을 복사해서 붙여 넣으세요. 
```python
# coding=utf-8

import boto3
import json
import gspread
import os
import email.parser
from BccardEmailParser.BccardParser import *

from base64 import b64decode
from time import strftime as timestamp
from oauth2client.service_account import ServiceAccountCredentials


APPLICATION_NAME = 'myReceipts - Write to Google Spreadsheet'
SCOPES = ['https://spreadsheets.google.com/feeds']

# 1. 함수 실행시 환경 변수를 전달 받습니다.
spreadsheetsId = os.environ['spreadsheetsId']
worksheetName = os.environ['worksheetName']
S3BUCKET = os.environ['s3Bucket']
debug_flag = str(os.environ['debugmode']).lower() == 'true'
apiKey = os.environ['googleSheetApiKey']

apiKeyDict = json.loads(apiKey)
credentials = ServiceAccountCredentials.from_json_keyfile_dict(apiKeyDict, scopes=SCOPES)


def lambda_log( string ):
    if debug_flag:
        print(string)


def appendrow(spreadsheets_key, worksheet, receipt):
    gc = gspread.authorize(credentials)
    sps = gc.open_by_key(spreadsheets_key)
    wks = sps.worksheet(worksheet)
    rows = [str(receipt['date']), float(receipt['amount']), receipt['currency'],        # 1~4
            receipt['store_name'], "", receipt['card_name'], receipt['type']];  # 5~8
    print(rows)
    wks.append_row(rows)


def load_mail(s3_bucket, email_id):
    s3_client = boto3.client('s3')
    obj = s3_client.get_object(Bucket=s3_bucket, Key=email_id)
    return obj['Body'].read()


def extract_html_body(email_msg):
    p = None

    parser = email.parser.Parser()
    msgs = parser.parsestr(email_msg)

    for part in msgs.walk():
        if part.get_content_type() == 'text/html':
            p = part
            break

    if p is not None:
        lambda_log("message[type: " + p.get_content_type() +
                   ",transfer encoding: " + p.get('Content-Transfer-Encoding') + "]")
        email_content = p.get_payload(decode=p.get('Content-Transfer-Encoding'))
    else:
        raise Exception("Unsupported email message: " + email_msg)

    return email_content


def lambda_handler(event, context):
    # 2. 이메일 수신 이벤트에서 Email ID를 가져옵니다.
    message_id = event['Records'][0]['ses']['mail']['messageId']
    print('Processing with message id: ' + message_id)

    # 3. S3버킷에서 ID로 이메일을 가져오고, 본문 부분만 추출합니다.
    email_raw = load_mail(S3BUCKET, message_id)
    
    if type(email_raw)  is bytes:
        email_raw = email_raw.decode('utf-8')
    email_content = extract_html_body(email_raw)
    lambda_log(email_content)

    # 4. 메일 본문에서 결제 정보를 파싱합니다.
    bc_parser = BccardParser()
    
    if type(email_content) is bytes:
        email_content = email_content.decode('utf-8')
    bc_parser.parse(email_content, 'utf-8')
    receipt = bc_parser.get_result()
    lambda_log(receipt)

    # 5. 스프레드시트에 결제 정보를 입력합니다.
    appendrow(spreadsheetsId, worksheetName, receipt)
    return "OK"
```

Lambda에서 수행될 myreceipt.py 코드의 절차를 5단계로 구분하였습니다. 

1) 함수 실행시 환경 변수를 전달 받습니다
2) 이메일 수신 이벤트에서 Email ID를 가져옵니다. 
3) S3버킷에서 ID로 이메일을 가져오고, 본문 부분만 추출합니다.
4) 메일 본문에서 결제 정보를 파싱합니다. 
5) 스프레드시트에 결제 정보를 입력합니다. 

S3 저장된 이메일 본문을 프로그램으로 읽어온 후, 결제 정보를 파싱하여 마지막으로, 내가 사용하는 스프레드시트에 저장하는 간단한 절차입니다. S3 버킷이름 및 스프레드시트의 ID, Google Spreadsheet API 등의 인증 정보 등의 변경 될 수 있는 부분은 코드에서 분리하여, 환경변수로 입력 받도록 하였습니다. 총 5가지의 환경변수를 사용할 것 입니다. 환경변수는 AWS Lambda 콘솔 화면에서 코드의 변경 없이 필요한 값으로 언제든지 변경할 수 있습니다. 

* spreadsheetsId - Google Spreadsheet의 ID
* worksheetName - Google Spreadsheet의 worksheet 이름을 입력합니다.
* s3Bucket - Email이 저장 된 S3 버킷 이름을 입력합니다.
* googleSheetApiKey - Google Spreadsheet API 사용을 위한 json형태의 인증정보가 입력됩니다.
* debugmode - 코드 수정 중, 추가 정보를 추가하기 위한 용도입니다.


#2. Lambda로 배포 하기 

myreceipt.py 소스코드와 필요한 라이브러리가 모두 준비되었습니다. 아래와 목록과 같이 많은 파일과 디렉토리를 볼 수 있습니다. AWS Lambda에는 필요한 라이브러리를 함께 올려야 됩니다. 이 많은 파일과 폴더를 하나의 파일로 압축합시다. 

```bash
$ ls

BccardEmailParser
chardet
httplib2-0.11.3.dist-info
oauth2client-4.1.3.dist-info
requestssix.py
chardet-3.0.4.dist-info
idna
pyasn1
requests-2.20.0.dist-info
urllib3
bccard_parser-0.1.1.dist-info
gspread
idna-2.7.dist-info
pyasn1-0.4.4.dist-info
rsa
urllib3-1.24.dist-info
certifi
gspread-3.0.1.dist-info
myreceipt.py
pyasn1_modules
rsa-4.0.dist-info
certifi-2018.10.15.dist-info
httplib2
oauth2client
pyasn1_modules-0.2.2.dist-info
six-1.11.0.dist-info
```

현재 프로젝트 디렉터리에 있는 모든 파일 및 하위 디렉터리를 myReceipts.zip 이름으로 압축합니다. 그리고, 압축된 파일의 위치는 현재 프로젝트 상위 디렉터리에 저장하도록 합니다. 다른 위치에 압축파일이 저장되도록 할 수 있습니다. 다만, 소스코드가 포함된 위치에 저장하는 것은 피하는 것이 좋습니다. 코드 변경 후, 다시 압축 수행할 때 압축 파일에 이전 압축파일을 포함시키는 실수를 할 수 있거든요. 

```bash
$ zip ../myReceipts.zip -r \*
 adding: BccardEmailParser/ (stored 0%)
 adding: BccardEmailParser/__init__.py (stored 0%)
 adding: BccardEmailParser/__pycache__/ (stored 0%)
 adding: BccardEmailParser/__pycache__/BccardParser.cpython-36.pyc (deflated 49%)
 adding: BccardEmailParser/__pycache__/__init__.cpython-36.pyc (deflated 18%)
 adding: BccardEmailParser/BccardParser.py (deflated 73%)
 adding: __pycache__/ (stored 0%)
 adding: __pycache__/six.cpython-36.pyc (deflated 58%)
 adding: bccard_parser-0.1.1.dist-info/ (stored 0%)
 adding: bccard_parser-0.1.1.dist-info/RECORD (deflated 45%)
…

$ ls ../myReceipts.zip
myReceipts.zip
```

여기까지 코드 및 AWS Lambda로 배포할 코드 패키지가 준비가 완료되었습니다.

# 3. 이메일을 받으면, Lambda 함수가 실행되도록 하자 

이제 AWS Lambda를 시작해 봅시다. AWS Lambda 콘솔 페이지([https://console.aws.amazon.com/lambda/home](https://console.aws.amazon.com/lambda/home?region=us-east-1))로 이동합니다. 현재 접속한 리전(지역)을 다시 한 번 확인하세요. 앞서 SES를 설정한 리전과 동일한 리전에서 진행해야 합니다. 우리는 버지니아 북부에서 진행하고 있습니다.

AWS Lambda의 대시보드 화면에서, “함수 생성” 을 선택합니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-1.png" width="600" alt="4-1.">

“새로 작성”을 선택합니다. 이름을 “myReceipts” 입력하고, 런타임으로는 Python 3.6을 선택합니다. 

역할(IAM Role)은 ‘사용자 지정 역할 생성’을 선택합니다. Role 설정을 위한 새로운 창이 나타납니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-2.png" width="600" alt="4-2.">

새로운 역할 이름을 입력하고, ‘허용’을 선택합니다. 이름은 “myreceipt_lambda_execution”으로 하였습니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-3.png" width="600" alt="4-3.">

‘역할’ 항목이 ‘기존 역할 선택’으로 변경되어 있습니다. ‘기존 역할’에서 앞서 생성하였던 역할을 찾아 선택합니다. 

다음으로 '함수 생성'을 선택합니다.

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-4.png" width="600" alt="4-4.">

다음 그림과 같이 “myReceipts” Lambda 함수가 생성되었습니다. 우측 상단의 “테스트”를 클릭하여 helloworld 예제를 바로 실행할 수 있습니다. 이제 우리가 필요한 서비스를 연결하고 코드를 업로드하면 카드 결제 정보를 수집할 수 있겠죠? 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-5.png" width="600" alt="4-5.">

## Lambda 함수에 S3 권한 추가하기 

myReceipts 함수는 S3에서 이메일을 읽어와야합니다. Lambda 함수 생성하면서 추가했던 myreceipt_lambda_execution 역할(IAM Role)에 S3의 값을 읽을 수 있도록 적용하겠습니다. 

IAM Management Console 페이지(<https://console.aws.amazon.com/iam/home>)로 이동합니다. 좌측 메뉴 에서 “역할”을 선택합니다. 

검색에서 myreceipt를 입력 후 검색을 수행합니다. 앞서 만들었던, myreceipt_lambda_execution 역할을 찾을 수 있습니다. 선택합니다.

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-6.png" width="600" alt="4-6.">

oneClick으로 시작하는 정책(Policy)이 하나 있습니다. 왼쪽 화살표를 선택하여 속성을 확장합니다. “정책 편집” 버튼을 눌러 권한을 추가하겠습니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-7.png" width="600" alt="4-7.">

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-8.png" width="600" alt="4-8.">

"권한 추가”를 선택합니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-9.png" width="600" alt="4-9.">

서비스 항목에는 S3를 찾아 선택합니다. 

그리고, “작업” - “액세스 레벨”은 목록 및 읽기 를 체크합니다. 

다음으로 ‘리소스’ 항목에서 ‘특정’을 선택합니다. 그리고, object 항목에서 “ARN 추가”를 선택합니다. Bucket name에는 ‘myreceipt-bucket’을 입력합니다. Object name에는 “모두 선택”을 체크하고 저장합니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-10.png" width="600" alt="4-10.">

마지막으로, Review Policy 를 선택하고, Save changes를 선택해서 추가한 권한을 적용하도록 합니다.

다음으로 메일을 SES에서 수신했을 때, 우리가 만든 “myReceipts” Lambda 함수를 실행하도록 할 차례입니다. 이벤트를 통해 함수를 수행하는 것을 트리거한다고 합니다. 

이메일을 수신했을 때, myReceipt 함수가 실행되어 저장한 이메일을 S3에서 가져올 수 있도록 설정을 트리거를 설정해보겠습니다. 

여러 AWS 서비스를 이용하여 내가 작성한 Lambda 함수를 실행할 수 있도록 할 수 있습니다. SES의 내가 생성한 이메일 주소로 이메일을 수신했을 때, 우리가 만든 myReceipts Lambda 함수를 수행하게 할 것 입니다. 2가지 방식으로 트리거를 설정할 수 있습니다. 첫째는 SES에서 수신 이메일 주소로 Rule Set으로 Lambda 함수를 추가하는 방법입니다. 앞서 설정한 <receipt@myreceipt.net> 이메일 주소로 결제 이메일이 전달되면 S3에 해당 파일을 저장하게 했습니다. 그리고, 추가로 Lambda 함수를 실행하게 하여 S3에 저장된 이메일을 처리하게 하는 방법입니다. 두 번째는, 앞서 만들었던 Rule Set에 의해서 <receipt@myreceipt.net>로 수신한 새로운 이메일은 S3에 저장 됩니다. S3에 새로운 객체(파일)이 생성될 때, Lambda 함수를 실행하게 할 수 있습니다. 본에서는 첫 번째 방법으로 진행하도록 하겠습니다. SES에서 <receipt@myreceipt.net> 수신 이메일에 Lambda를 수행 동작을 추가하였습니다. 

## Rule Set에 Lambda 실행 동작 추가하기 

앞서 설정 했던 Rule Sets 페이지([https://console.aws.amazon.com/ses/home?region=us-east-1](https://console.aws.amazon.com/ses/home?region=us-east-1#rule-set:myreceipt))로 이동합니다. 

왼쪽 메뉴의 “Email Receiving” 항목 중, “Rule Sets”를 선택합니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-11.png" width="600" alt="4-11.">

그리고, Active Rule Set 페이지에서 “View Active Rule Set”으 선택합니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-12.png" width="600" alt="4-12.">

다음으로, 앞서 만들었었던 ‘myreceipt-invoke-lambda’ Rule을 선택하고, Actions에서 Edit를 선택합니다. 

Rule 편집화면에서 기존에 추가했던 S3 항목이 있으며, 아래에 “Add action”을 선택후, Lambda를 선택합니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-13.png" width="600" alt="4-13.">

두 번째 동작(Action)으로 Lambda가 추가됐습니다. Lambda function 항목에는 내가 작성했던 Lambda 함수 이름(myReceipts)을 입력하도록합니다. 만약 내가 작성한 Lambda함수가 보이지 않는다면, SES가 생성된 리전(us-east-1)과 동일한 리전에서 Lambda함수가 생성되었는지 확인합니다. 서로 다른 리전에 있는 경우, SES에서 Lambda를 트리거 할 수 없습니다. 

Invocation type은 “Event”로 선택합니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-14.png" width="600" alt="4-14.">

그리고, Save Rule을 선택하여 저장합니다. 아래와 같이 "Missing Permissions”라는 메시지가 나옵니다. SES가 Lambda 함수를 수행하기 위해서는 해당 함수에 대해 실행권한이 있어야 합니다. 권한이 없다는 것을 알려주며, “Add permissions”을 눌러 필요한 권한을 자동 추가합니다. 이런 안내와 부분적인 자동화 설정은 AWS 서비스를 처음으로 익히는데 매우 큰 도움이 됩니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-15.png" width="600" alt="4-15.">

자! Rule Set이 업데이트 되었습니다. 이메일을 수신했을 때, Lambda함수가 잘 수행되는지 체크해볼까요? 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-16.png" width="600" alt="4-16.">

AWS Lambda 함수 페이지로 이동합니다. 우리가 작성한 myReceipts 함수를 선택합니다. 모니터링 항목을 선택합니다. 

모니터링 항목에서는 해당 함수가 수행된 회수 및 시간, 에러 등을 그래프로 표시해줍니다. 내가 작성한 함수가 이메일 수신 할 때 수행되었는지 여기서 확인할 수 있습니다.

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-17.png" width="600" alt="4-17.">

테스트 메일을 보내보겠습니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-18.png" width="400" alt="4-18.">

몇 십초 뒤에 모니터링을 확인해보면 최근 1회 호출 된 것을 볼 수 있습니다. 수신한 이메일에 의해서 등록한 Lambda 함수가 정상 수행되고 있습니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-19.png" width="600" alt="4-19.">

# 4. 이메일을 분석하는 소스코드를 Lambda 함수에 적용하자 

이메일 수신 이벤트에 의해서 myReceipts Lambda 함수가 실행되는 것을 확인할 수 있습니다. 그러나, 함수가 실행되어도 “helloworld”를 출력하는 동작만 수행하고 있습니다. Lambda 함수에 파싱하고 필요한 명령을 추가하지 않았기 때문입니다. 다음으로 할 일은 helloworld를 출력하는 코드를 앞서 생성했던 myReceipts.zip 파일로 대체하는 것 입니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-20.png" width="600" alt="4-20.">

코드 입력 유형은 “.zip 파일 업로드”로 변경합니다. 그리고, 런타임과 핸들러 이름이 동일한지 확인합니다. 핸들러는 Lambda 함수가 실행 될 때 가장 처음 시작하는 함수를 의미합니다. 핸들러 이름의 구성은 “myreceipt.lambda_handler”로 되어있는데, 1) 파일이름과, 2) 해당 파일의 python 함수 이름을 의미합니다. 우리가 준비한 코드에서는 myreceipt.py 파일의 lambda_handler()함수가 수행되겠죠?

마지막으로 업로드를 누르고, myReceipts.zip 파일을 추가합니다. 1.7MB 크기네요. 저장 버튼을 누르면 파일이 업로드 됩니다. 업로드가 완료되면 압축했던 라이브러리와 파일의 내용을 편집할 수 있도록, 함수 코드 항목에서 나타납니다. PC에서 코드를 수정한 후, 압축 후 올리는 대신에, 함수 코드 화면에서 바로 편집하고 Lambda 함수를 테스트해 볼 수 있습니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-21.png" width="600" alt="4-21.">

아래로 내려오면 환경 변수 항목이 있습니다. 코드를 업로드 하면 환경 변수를 입력해보도록 하겠습니다. 입력할 환경변수는 5가지입니다.

왼쪽에는 키을 오른쪽에는 키에 해당하는 값을 입력합니다. googleSheetApiKey키는 Google Spreadsheet API에는 서비스 키 생성할 때, 다운로드 받았던 개인키의 본문을 여기에 붙여넣기 합니다.

|키 |값|
|---|---|
|debugmodefalsegoogleSheetApiKey | { "type": "service_account", "project_id": "prj-myreceipt", "private_key_id": “638996c51ec9 ...|
|s3Bucket | myreceipt-bucket|
|spreadsheetsId | 1WALlmF14--pOtrgvNQNX10OSMXSW1pJFWNuXWPILFWQ |
|worksheetName | 2018지출내역|

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-22.png" width="600" alt="4-22.">

마지막으로 Lambda 함수에 대해 설정을 할 수 있습니다. 메모리는 128MB로 설정하고, 제한 시간(timeout)은 1분으로 설정합니다. 우리가 작성한 코드는 많은 메모리를 요구하지 않습니다. 다만, Google Spreadsheet API를 호출하는 과정에서 십여초가가 소요될 수도 있습니다. 이로인해 Lambda 함수가 실패하지 않도록 충분한 제한 시간을 설정합니다. 그리고, 저장합니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-23.png" width="400" alt="4-23.">

이제 완료됐습니다. 바로 시작해보겠습니다. 이전에 전달 바든 결제 이메일로 테스트 해보겠습니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-24.png" width="600" alt="4-24.">

<receipt@myreceipts.net> 로 이전 결제 메일을 전달 합니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-25.png" width="600" alt="4-25.">

그리고, 몇 초에서 십초 뒤, 아래와 같이 결제 내역 정보가 Spreadsheet에 추가된 것을 확인할 수 있습니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/4/4-26.png" width="600" alt="4-26.">

여러분의 카드 결제 내역을 이메일로 받고 있다면, 이제 <receipt@myreceipts.net>와 같은 여러분의 이메일로 전달되도록 하세요. NAVER 및 Gmail 등의 이메일 서비스로 결제 메일을 이미 받고 있다면, 본인이 만든 이메일 주소로도 전달되도록 설정하여 결제 내역을 쉽게 수집할 수 있습니다. 

## 유의사항
* 절대! <receipt@myreceipts.net> 주소로 이메일을 보내지 않도록 합니다. 자신의 도메인의 이메일주소로 설정하세요.
