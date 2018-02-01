# 0. 시작

Python 코드로 Spreadsheet에 데이터를 저장하는 방법을 설명합니다. Google에서는 [Google Sheets API]( https://developers.google.com/sheets/guides/concepts?hl=ko)를 제공합니다.  Sheets API를 이용하여 시트에 저장된 데이터를 읽어오거나, 기록할 수 있습니다. 시트를 생성하고 Python 코드로 생성한 시트에 레코드를 저장하는 과정까지 알아보도록 하겠습니다. 여기서 Python 버전은 3.6 버전을 기준으로 설명합니다. 참고로 AWS Lambda에서는 Python 버전 3.6과 2.7을 지원합니다. 두 버전 중 하나를 선택하여 진행하시기 바랍니다. 

> 유의사항: API의 최신 버전은 v4입니다. 다만, 처음 개발시에 레거시 API인 v3 기반의 Python 라이브러리를 이용하게 되었습니다. 이점 양지하시기 바랍니다. 추후에 신규 버전으로 업데이트 하도록 할 예정입니다. 

# 1. Spreadsheet 준비 

Google 계정을 갖고 있어야 하겠죠? Google Spreadsheet로 이동합니다. 앞선 챕터에서 설명해 드린 템플릿을 이용하셔도 되고, 새로운 Spreadsheet를 생성하여 테스트하셔도 됩니다. 

여기서는 템플릿을 이용하도록 하겠습니다. 공유한 템플릿은 읽기 전용입니다. 본인의 Google Spreadsheet로 복사하여 이용하시기 바랍니다. 

1) “2018 My Expenses”  템플릿으로 이동합니다.
   * 2018 My Expenses 시트 템플릿 - http://bit.ly/myexpense2018

2) 메뉴의 ‘파일’ - ‘사본 만들기…’ 를 선택 후, 본인의 계정으로(본인의 Google 드라이브로) 시트를 복제합니다. 
<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/3/3-1.png" width="600" alt="3-1.">


3) 이름도 바꾸고, 폴더를 선택하고 저장합니다. 다른 사람과 공유할 필요가 없으므로, ‘같은 사람과 공유’는 체크하지 않고 ‘확인’을 선택합니다. 
<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/3/3-2.png" width="300" alt="3-2.">


4) Google Spreadsheets가 복제되고 새로운 창으로 나타납니다. 복사 되면, 아래의 세가지 정보를 별도로 기록하시기 바랍니다. 
   - 본인 Google 계정 이메일 주소
   - Spreadsheets 아이디
   - 데이터를 추가할 시트 이름 (예, 2018지출내역)

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/3/3-3.png" width="600" alt="3-3.">



# 2. API 사용을 위한 인증 설정

Google 서비스와 관련된 API 및 인증 등은 Google Cloud Platform에서 통합하여 관리됩니다. Sheets API 사용을 위해 새로운 프로젝트를 생성하고 API 이용을 위한 키를 발급 받을 수 있습니다. 

1) [Google Cloud Platform](https://console.cloud.google.com/home/dashboard)으로 이동합니다. 처음 이용하신다면, 서비스 약관을 확인하고 약관에 동의하여 신청하시기 바랍니다. 
<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/3/3-4.png" width="500" alt="3-4.">


2) 새로운 프로젝트를 생성합니다. 상단 메뉴의 프로젝트 이름을 선택 후, 팝업 화면에서 ‘프로젝트 생성’을 선택합니다. 
<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/3/3-5.png" width="500" alt="3-5.">



3) 이름은 myReceipts로 하겠습니다. 아래의 프로젝트 ID는 프로젝트 이름을 바탕으로 자동 생성됩니다. 만들기를 선택합니다. 프로젝트가 생성되고 사용가능하기 까지 준비 시간이 필요합니다. 
<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/3/3-6.png" width="500" alt="3-6.">

4) 프로젝트가 준비되면, 프로젝트 목록에서 생성한 프로젝트를 선택하여 이동하시기 바랍니다. 

5) ‘IAM 및 관리자’ - ‘서비스 계정’을 선택합니다. 
<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/3/3-7.png" width="500" alt="3-7.">

'서비스 계정 만들기'를 선택합니다.

서비스 계정 만들기 화면에서 서비스 계정의 이름을 입력하고, ‘새 비공개 키 제공’을 체크, 키 유형은 ‘JSON’으로 선택 후, ‘만들기’를 수행합니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/3/3-8.png" width="500" alt="3-8.">

서비스 계정이 성공적으로 생성되었습니다. 생성 과정에서 서비스 계정의 비공개 키는 JSON 파일로 저장됩니다. 해당 파일은 분실하거나 노출 되지 않도록 주의하여 관리하시기 바랍니다. 키 값이 노출 되거나 분실 되었다면, 서비스 계정을 삭제 후, 다시 발급 받을 수 있습니다. 

‘서비스 계정 ID’는 이메일 주소 형식으로 되어있습니다. 서비스 계정 ID는 시트의 쓰기 권한 설정을 위해 이용합니다. 
서비스 계정 ID를 복사합니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/3/3-9.png" width="500" alt="3-9.">


앞서 생성한 Spreadsheets로 이동합니다. 우측 상단의 ‘공유’ 버튼을 누르고, 생성한 ‘서비스 계정 ID’에게 해당 시트를 공유합니다. 권한은 ‘수정 가능’ 권한을 주셔야 쓰기 가능합니다. ‘전송’을 누릅니다. 

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/3/3-10.png" width="500" alt="3-10.">

본인의 Gmail 계정으로 서비스 계정의 이메일 주소를 찾을 수 없다는 반송 메일이 전달 될 것 입니다. 해당 이메일은 무시하셔도 됩니다. 

# 3. Python 코드 

사용하는 라이브러리가 두개 있습니다. 인증을 위한 OAuth2Client와 Sheets API의 래퍼 라이브러리인 gspread입니다. 

작업 디렉토리에서 라이브러리를 설치합니다. 작성한 코드를 AWS Lambda로 배치하기 위해서는 사용하는 라이브러리도 함께 포함해야합니다. 그러므로, 여기서도 작업 디렉토리에 사용할 라이브러리를 설치하도록 합니다. pip install 명령어에 -t 옵션으로 라이브러리가 저장 될 위치를 선택할 수 있습니다. 
```bash
$ pip3 install gspread -t . 
$ pip3 install oauth2client -t . 
```

아래의 main.py 파이썬 코드를 준비합니다. 

```python
# main.py
import gspread
from oauth2client.service_account import ServiceAccountCredentials

APPLICATION_NAME = 'Google Spreadsheet API sample'
SCOPES = ['https://spreadsheets.google.com/feeds']

worksheetName = '2018지출내역'
spreadsheetsId = '1WALlmF14--pOtrgvNQNX10OSMXSW1pJFWNuXWPILFWQ'
keyfile = 'cred.json'

new_row = ['hello', 123, 'new test', u'utf8//한글']

credentials = ServiceAccountCredentials.from_json_keyfile_name(keyfile, scopes=SCOPES)

gc = gspread.authorize(credentials)
spreadsheet = gc.open_by_key(spreadsheetsId)
worksheet = spreadsheet.worksheet(worksheetName)

worksheet.append_row(new_row)

print('Done')
```

'1. Spreadsheets 준비'에서 별도로 기록해 두었던 워크시트의 이름과 Spreadsheets 아이디를 코드의 worksheetName과 spreadsheetsId의 값에 각각 치환합니다. 

다음으로, keyfile의 값에 서비스 계정시 생성되었던 비공개 키 JSON 파일의 경로/이름을 입력합니다. 해당 코드에서는 main.py와 동일한 경로에 cred.json이름으로 가정하였습니다. 

명령어를 실행합니다. 아래와 같이 새로운 행이 추가 된 것을 볼 수 있습니다. 
```bash
$ python3 main.py
```

<img src="https://raw.githubusercontent.com/0kim/myReceipts/master/images/3/3-11.png" width="500" alt="3-11.">

> 유의사항: 새로운 워크시트 생성 시, 기본으로 1000행의 비어있는 행이 생성되어있습니다. append_row()함수는 현재 생성되어 있는 행, 다음에 새로운 행을 추가하고 값을 넣게 됩니다. 따라서, 신규 워크시트에 append_row() 실행시, 1000번째 행 뒤에 행 하나를 추가하고 값을 입력하게 됩니다. 사용하지 않는 비어 있는 행을 삭제하시기 바랍니다. 상단에서 부터 순차적으로 값이 추가 되게 할 수 있습니다. 



