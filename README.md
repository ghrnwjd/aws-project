## SW 산학연계프로젝트 정리

2024.02.19~2024.02.23까지 이루어진 한국외국어대학교 SW산학연계프로젝트에서 실습한 내용입니다.

### 서비스 구조도
![Pasted image 20240229165350](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FUSzzL%2FbtsFm85ti1P%2FtR6mAJKSIwMvobSaYBvcqK%2Fimg.png)

웹 프론트는 HTML와 CSS, 백은 SpringBoot를 통하여 구현하였음.

#### 사전작업
**SageMaker를 통한 모델 배포**
![Pasted image 20240229165535.png](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FAW0Gt%2FbtsFlOT2zTc%2FWb7g39Fn95QdGodyzRj8ck%2Fimg.png)

모델은 입력으로 receiver_id, sender_id, amount, timestamp를 입력으로 받고 transaction_category를 예측한다.

transacton_category는  `Uncategorized, Entertainment, Education, Shopping, Personal Care, Health and Fitness, FOod and Dining, Gifts and Donations, Investments, Bills and Utilities, Auto and Transport, Travel, Fees and Charges, Business Services, Personal Services, Taxes, Gambling, Home, Pension and Inurances` 로 총 19개의 클래스로 구성되어 있다.

웹페이지에서 엔드포인트를 호출하기 위하여 AWS Lambda와 AWS API Gateway를 사용한다.


**AWS Lambda Code**
코드를 작성하기에 앞서 환경변수로 ENDPOINT_NAME에 SageMaker의 호출주소를 입력한다.
또한 result를 받을 때 `JSONDecodeError: Extra data` 에러가 발생하여, JSON 형식에 맞출 수 있도록 \[] 배열로 한번 더 감싸며 해결하였다.
```python
import os
import io
import boto3
import json
import csv

ENDPOINT_NAME = os.environ['ENDPOINT_NAME']
runtime = boto3.client('runtime.sagemaker')

def lambda_handler(event, context):
	data = json.loads(json.dumps(event))
	payload = data['data']
	response = runtime.invoke_endpoint(EndpointName=ENDPOINT_NAME,
										ContentType='text/csv',
										Body=payload)
	result = json.dumps([response['Body'].read().decode()])
	return result
```

또한 람다에서 sagemaker에 접근하기 위한 정책을 설정하였다.
```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "",
			"Effect": "Allow",
			"Action": "sagemaker:InvokeEndpoint",
			"Resource": "*"
		}
	]
}
```
**API Gateway**
![Pasted image 20240229170733.png](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Fxwgbv%2FbtsFkJFrypX%2FqzgOWcKh4r3cA5kN1YBidK%2Fimg.png)
모델의 요청은 `POST` 방식으로 전달할 것이기에 리소스에 POST를 추가하였고, 이후 CORS 에러가 발생하여 옵션에서 `Access-Control-Allow-Origin: ‘*’`으로 설정하였다.
![Pasted image 20240229170827.png](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbNRPhh%2FbtsFpPRLwrt%2FCEN5qn520Gxmvrwm41pEak%2Fimg.png)

AWS lambda, API Gateway의 구성은 다음과 같다.
![Pasted image 20240229170302.png](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FWFixa%2FbtsFlUs5cTn%2FSkiBXVPzx2p4PdFlworPT1%2Fimg.png)

**EC2 인스턴스 생성**
웹사이트를 띄우기 위해 EC2 인스턴스를 생성한다.
![Pasted image 20240229171309.png](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FBZbsp%2FbtsFspZcEpv%2FCgunwbf2VJCJpK1vZ9bUkk%2Fimg.png)

HTTP(80), HTTPS(443), 앱(8080)을 올리기 위해 해당 포트에 대해 인바운드 규칙을 설정하고, 모델이 호출된 로그를 S3 버킷에 저장하기 위하여 S3에도 접근할 수 있는 권한 `EC2Project`라는 이름으로 부여하였다. 정책은 `S3FullAccess`만 포함된다.


**프로젝트 시연 영상 주소**
https://www.youtube.com/watch?v=kj1LvqkwzYQ
