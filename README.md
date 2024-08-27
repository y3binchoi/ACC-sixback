# ACC-sixback
ACC 해커톤 1팀 식스백

# 프로젝트 소개

> 주제: 이메일 대량 발송 시스템 설계
> 

https://github.com/aws-cloud-clubs/ACC-sixback

- 
    
    안녕하세요 1팀 식스백 팀장 지민혁 입니다. 발표 시작하겠습니다.
    
    저희는 이메일 대량 발송 시스템 설계라는 주제로 프로젝트를 진행했습니다.
    

## 요구 및 제약 사항

> 대량의 이메일을 적정 지연 시간 안에 발송하기 위해 확장성을 고려한 아키텍처 설계가 필요하다.
> 
- 이메일의 크기는 평균 `100KB`로 가정한다.
- 이메일은 비즈니스의 성격에 따라 발송 패턴이 각기 다르다.
    - 1,000개의 메일을 5분 이내에 발송한다. **← 대회 최소 요구 사항**
    - 30,000개의 메일을 5분 이내에 발송한다.
    - 1,000,000개의 메일을 5분 이내에 발송한다.

> 사용자의 데이터와 서비스 안정성을 위해 고가용성 설계가 필요하다.
> 
- 이메일 발송 기능은 주요 비즈니스 로직과 관련이 적다. 따라서 서버리스 아키텍처로 인프라 오버헤드와 비용을 절감하고 유연하게 대량 트래픽에 대처한다.

- 
    
    저희의 요구 및 제약 사항은 다음과 같습니다.
    
    이메일의 크기는 평균 100kb로 가정하고, 
    
    해당 시스템의 기본 요구사항인 1000개의 메일을 5분 이내에 발송한다.
    
    30,000개의 메일을 5분 이내에 발송한다.
    
    1,000,000개의 메일을 5분 이내에 발송한다.
    
    와 같이 비즈니스 성격에 따라 발송 패턴이 다름을 고려하여 설계하였습니다.
    
    서버리스 기반의 시스템은 aws의 완전 관리형 서비스이기에 고가용성을 보장한다고 판단하였습니다.
    

# 최종 아키텍처

![image](https://github.com/user-attachments/assets/943db283-0363-4795-bfe4-c43c51867daa)


- 
    
    저희의 최종 아키텍처는 다음과 같습니다.
    
    이 아키텍처를 순서에 따라 설명드리도록 하겠습니다.
    

# 서비스 플로우

### STEP 0 : 송신자가 이메일 템플릿을 (이메일 발송 전에) DynamoDB에 저장한다.

```json
{
	"template_key" : 3, 
	"template_value" : "
	<!DOCTYPE html>
	<html>
	<head>
	<title>{{mailTitle}}</title>
	</head>
	<body>
    <h1>Welcome, {{recipientName}}!</h1>
    <p>Thank you for joining us. We're excited to have you.</p>
    <p>Please find the details below:</p>
    <p>{{mailContent}}</p>
    <p>Best regards,<br>{{senderName}}</p>
    <div><img src={{image_0}} alt="Image 1" style="width: 300px; height: 300px;" />
    <img src={{image_1}} alt="Image 2" style="width: 300px; height: 300px;" /></div>
	</canvas>
	</body>
	</html>
	"
}
```

- 
    
    먼저 송신자는 다음과 같이 사전에 DynamoDB에 이메일 템플릿을 저장한다는 가정하에 진행하였습니다.
    

### STEP 1: 송신자는 발송할 메일 템플릿을 선택한 뒤, 내용을 완성하고 이메일 수신자 목록(.csv)을 업로드한다.

![image](https://github.com/user-attachments/assets/3c4c8bfd-014d-44a2-898a-0b7c8a332c71)


![image](https://github.com/user-attachments/assets/df50432e-83bb-459b-a8a6-9788c9aa9cd8)

- 송신자는 프론트엔드 상에서 메일 템플릿을 선택한 뒤, 내용을 완성하고, 이메일 수신자 목록을 csv파일로 업로드 합니다.
    

### [/api/url]

![image](https://github.com/user-attachments/assets/6d49bb70-2256-4b45-b589-aa0b58411b70)


- `email-urls-lambda`: Presigned URL 생성 요청 후 반환
- Client: Presigned URL을 통해 이미지 및 이메일 수신자 목록(.csv) 파일 버킷 업로드
- 이런 방식을 사용한 이유는 API Gateway를 통해 이미지를 직접 삽입하면 10MB라는 제한에 의해 고품질의 이미지를 저장할 수 없습니다.
- 또한 SQS에는 이미지를 저장할 수 없고 페이로드가 256kb기 때문에
- 
- 
    
    먼저 email-url 람다를 통해 S3에 Presigned URL 생성 요청 후 반환합니다.
    
    이후 클라이언트는 해당 URL을 통해 이미지 및 이메일 수신자 목록 즉 csv 파일을 버킷에 업로드합니다.
    

### STEP 2: 입력한 내용에 대해 ‘즉시발송’ OR ‘예약발송’

### 즉시발송: [/api/general]

![image](https://github.com/user-attachments/assets/7c60eaf8-a344-4885-8672-457bff1f8bb3)

- Client: json을 전송해 `generalinput-lambda` 실행

```json
{
  "Records": [
    {
      "body": {
        "TemplateID": "template_001",
        "Version": "1",
        "recipientEmail": "luckyrkd@naver.com/emails/9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08.csv",
        "senderEmail": "gmh8257@gmail.com",
        "mailTitle": "Welcome!",
        "recipientName": "SixBackMember!",
        "mailContent": "Thank you for joining us.",
        "senderName": "SixBack",
        "image_0": "https://dev-email-s3-attachment.s3.ap-northeast-2.amazonaws.com/SnedEmailTest001/37a12c3801e61148ca5dc72f0b5049f3.jpg",
        "image_1": "https://dev-email-s3-attachment.s3.ap-northeast-2.amazonaws.com/SnedEmailTest001/5c3519fc9803776ed2bd917a87503954.jpg"
      }
    }
  ]
}
```

- `generalinput-lambda`: recipientEmail의 s3 링크로 접근해 수신자 이메일이 저장돼있는 csv파일을 불러온 후, 수신자 목록에서 수신자를 한명씩 분리하여 dev-email-partition-sqs로 전달
- `partition-sqs`: dev-email-send-lambda-primary-1로 수신자별 이메일 정보를 전달
- `email-send-lambda`: 전달 받은 수신자별 이메일을 기반으로 DynamoDB에 저장된 이메일 템플릿과 합쳐 SES에 이메일 전송을 명령
- `SES`: 이메일 전송
- 
    
    이후 recipientEmail의 s3 링크에 접근해 수신자 이메일이 저장되어있는 csv파일을 불러온 뒤, 수신자 목록에서 수신자를 한 명 씩 분리하여 partition SQS로 전달합니다.
    
    partition SQS는 emailSend람다로 수신자 별 이메일 정보 전달을 합니다. 
    
    이후 emailSend람다는 전달 받은 수신자별 이메일을 기반으로 DynamoDB에 저장된 이메일 템플릿과 합쳐 SES에 이메일 전송을 명령합니다.
    
    이후 SES는 이메일을 전송합니다.
    

### 예약발송: [/api/reservation]

![image](https://github.com/user-attachments/assets/d68eb489-e04b-4c02-b53c-d995323c6230)


- Client: json을 전송해 `reservationinput-lambda` 실행

```json
{
  "Records": [
    {
      "body": {
        "TemplateID": "template_001",
        "Version": "1",
        "recipientEmail": "luckyrkd@naver.com/emails/9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08.csv",
        "senderEmail": "gmh8257@gmail.com",
        "mailTitle": "Welcome!",
        "recipientName": "SixBackMember!",
        "mailContent": "Thank you for joining us.",
        "senderName": "SixBack",
        "sendingSchedule": "2024-08-01T23:00:00",
        "image_0": "https://dev-email-s3-attachment.s3.ap-northeast-2.amazonaws.com/SnedEmailTest001/37a12c3801e61148ca5dc72f0b5049f3.jpg",
        "image_1": "https://dev-email-s3-attachment.s3.ap-northeast-2.amazonaws.com/SnedEmailTest001/5c3519fc9803776ed2bd917a87503954.jpg"
      }
    }
  ]
}

```

- `reservationinput-lambda`: sendingSchedule의 전송 예약 시각을 바탕으로 eventbridge에 create schedule을 진행, payload로 sendingSchedule을 제외한 나머지 json 부분을 전달
- eventbridge: 전송 예약 시각에 `eventtriggered-lambda`로  payload 전달
- `reservated-email-sqs`: `dev-email-generalinput-lambda-primary-1`로 전송 예약 시각을 제외한 json 전송
- **나머지 부분은 즉시 발송과 동일**
- 
    
    클라이언트는 json을 전송해 레저베이션 인풋 람다를 실행합니다.
    
    레저베이션 인풋 람다는 센딩스케쥴의 정보를 바탕으로 eventBridge에 규칙 생성을 진행하여 페이로드로 예약 정보를 제외한 나머지 json을 전달합니다.
    
    eventBridge는 정해진 예약 시간에 이벤트 트리거 람다로 페이로드를 전달합니다.
    

![reservationinput-lambda-primary-1에서 create schedule을 하여 eventbridge schedule에 표시되는 모습](https://github.com/user-attachments/assets/1767890d-e747-47eb-9f82-92eff76acd3d)


reservationinput-lambda-primary-1에서 create schedule을 하여 eventbridge schedule에 표시되는 모습

# **아키텍처 검증**

## 1,000개의 메일을 5분 이내에 발송하는 경우

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/fa4f43b4-b3fb-4cab-b91a-4f18e9afdc65/7b3f78c8-d481-4bff-9be1-7d8748fee8af/Untitled.png)

- 공식문서에 의하면, 기본적으로 람다 함수의 각 실행 환경은 초당 10개가 최대이다. 따라서 실행 시간이 100ms 이하인 요청이 3,000개 들어오면 300개 인스턴스로 동시성 처리를 할 수 있다.
    - 실제 테스트 시 이메일 발송 1건을 발송 시 500ms 이하의 시간이 책정되어 1초의 1개의 람다가 2개의 메일 발송이 가능합니다. 람다의 동시성은 1000개로 1초당 2000개의 메일 처리가 가능할 것으로 예상합니다.
    - 따라서 5분 내의 메일의 처리는 기본 1000개 이상이 가능할 것으로 보입니다.

## 30,000개의 메일을 5분 이내에 발송하는 경우

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/fa4f43b4-b3fb-4cab-b91a-4f18e9afdc65/6e52fe67-4321-4c7b-92d9-09ecafd402cb/Untitled.png)

- ~50,000명까지 월 20.02 USD vs 월 21만원(기존 이메일 마케팅 서비스)
    
    ![스크린샷 2024-08-03 오전 7.21.38.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/fa4f43b4-b3fb-4cab-b91a-4f18e9afdc65/06a55e0b-3a2e-4377-9fbb-ee2445b0a586/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2024-08-03_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_7.21.38.png)
    
    ![스크린샷 2024-08-03 오전 7.35.13.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/fa4f43b4-b3fb-4cab-b91a-4f18e9afdc65/fe76d058-aa84-4943-9531-2df636316d41/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2024-08-03_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_7.35.13.png)
    
    위 서비스는 [스티비](https://stibee.com/)라는 이메일 마케팅 서비스이다. 
    

## 1,000,000개의 메일을 5분 이내에 발송하는 경우

<aside>
🔢 단순하게 계산하면 인스턴스 최대 1000개, 인스턴스 환경 당 함수 10개/초 실행

→ 1분에 60,000번 실행 가능 

⇒ 5분에 300,000번 가능 

</aside>

- 따라서 1,000,000개의 이메일을 5분 이내에 발송할 수 없습니다. 하지만 이 정도 규모의 구독자가 있는 서비스라면 인프라를 이미 갖추고 있고 개발·운영 인력이 충분할 것으로 판단됩니다. 따라서 이런 기업이라면 자체적으로 이메일 발송 기능을 개발하는 것이 낫다고 생각합니다.

## 기술 선택 이유

### Lambda vs Fargate

|  | Lambda ✅ | Fargate |
| --- | --- | --- |
| 설명 | AWS의 서버리스 컴퓨팅 서비스로, 코드 실행에 필요한 인프라를 자동으로 관리 | 컨테이너 기반의 서버리스 컴퓨팅 서비스로, ECS에서 컨테이너를 실행 |
| 장점 | 자동 확장, 관리가 필요 없는 인프라, 비용 효율적 | 컨테이너 기반의 유연성, 다양한 언어 및 프레임워크 지원 |
| 단점 | 시작 지연(Lambda cold start), 상태 저장이 어려움 | 초기 설정 복잡, 비용이 더 높을 수 있음 |
- 클라우드 기반, 인프라 오버헤드 절감, 애플리케이션을 작은 구성 요소로 분해하여 배포하는 점에서 유사
- 이메일은 24/7 동안 지속적으로 트래픽이 발생하지 않고 전송 시간에 덜 민감하기 때문에 인프라 관리 지점과 비용을 줄이기 위해 완전 관리형 서비스인 Lambda를 선택하였습니다.
- **해커톤이라는 특수한 상황 고려**

<aside>
💡 **Step Function을 사용하지 않은 이유**

Step Function은 내부의 상태 전환에 따라 비용이 청구되는데 Lambda가 실행되는 모든 부분이 상태 전환이라 `(요청 수)*(Lambda 개수)` 만큼이 비용이 청구됩니다. 여기서 Lambda 실행에 대한 요금은 별개이므로 Step Function을 사용하는 만큼의 효용이 있어야 합니다. 하지만 현재 아키텍처는 이메일 발송 실패에 대한 로깅 및 재전송을 SES 이후의 SNS가 비동기적으로 처리하는 형태이므로 Step Function을 사용하지 않기로 결정했습니다. 

</aside>

### DynamoDB vs RDS

|  | DynamoDB ✅ | RDS |
| --- | --- | --- |
| 설명 | AWS의 완전 관리형 NoSQL 데이터베이스 서비스 | AWS의 관계형 데이터베이스 서비스 |
| 장점 | 자동 확장, 고성능, 관리 필요 없음 | SQL 지원, 트랜잭션 처리 |
| 단점 | 복잡한 쿼리 지원 부족 | 스케일링에 제한이 있을 수 있음 |
| 캐시 | DAX | ElastiCache |
- 이 시스템에서 DB는 이메일 템플릿 저장 용도로 쓰입니다. 저장되는 이메일 템플릿의 크기는 100KB 이하입니다. 읽기:쓰기 비율이 99:1이며 24/7 동안 지속적으로 트래픽이 발생하지 않기 때문에 인스턴스 실행 시간 단위로 비용을 산정하는 RDS보다는 실제 사용한 데이터 읽기 및 쓰기에 대해서만 비용을 지불하는 DynamoDB를 선택하였습니다.
