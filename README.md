# 프로젝트 '오디' 핵심기능 구현일지 : 실시간 친구 도착 예정정보 공유 기능

---

언제, 어디서, 무엇을 하든 지각하지 않게 도와주는 서비스 `프로젝트 오디`에서는
3차 스프린트 목표로 **실시간 친구 도착 여부를 알 수 있는 기능**을 핵심 기능으로 설정했습니다.

여기서 `실시간 친구의 도착 여부를 알 수 있는 기능`이란 약속 30분 전부터 친구에게 물어보지 않아도 시간 내에 도착할 수 있을지,
어느정도 걸릴지 등 실시간으로 친구의 위치를 기반으로 도착 예정정보를 제공하는 기능을 일컫습니다.

![img.png](img.png)

즉, 우리는 소비자에게 두 가지 기능을 제공해주고자 했습니다.

> 1) 시간 정보 : 친구가 약속장소까지 몇 분정도 남았는가?  
> 2) 지각 정보 : 친구가 지각할 예정인가?

이를 통해 친구들 사이에서 \`너 오디야?\`라고 묻는 상황을 최소화하고 만약 지각하는 친구가 있더라도 이를 유쾌하게 풀어낼 수 있는 서비스를 기획했습니다.

이번 글에서는 서비스 정책 기획부터 구현, 리팩터링까지 현재의 코드로 변화하기 까지
오디팀에서는 어떤 생각들을 해왔는지, 그 과정을 정리해보고자 합니다.

---

## **1\. 서비스 정책 및 구현 방식 정하기**

우선 제한 사항을 살펴보겠습니다.

> - API 사용량 : 대중교통 소요시간 API 제공량은 일 1000건  
> - 위치 공유 허용 여부 : 기기 사용자가 위치정보 공유에 동의해야 한다  
> - 화면 동기화 : 약속 참여자들이 동시에 같은 도착 예정 정보를 보아야 한다

이 3가지 제한 사항을 고려하여 서비스 정책과 큰 구현 흐름이 정해졌습니다.

---

### **1) 폴링 활용**

- 약속 정보 화면 동기화를 위해 **약속 30분 전부터 10초 간격으로 도착 정보를 요청하는 폴링**이 시작됩니다.

![img_1.png](imgs/img_1.png)

즉, 약속이 2시라면 **약속 30분 전인 오후 1시 30분 부터 약속 참여원들은 본인들의 위치 정보를 10초 간격으로 보내줍니다.**

![img_2.png](imgs/img_2.png)

서버는 보내준 클라이언트의 위치 정보를 바탕으로 **클라이언트의 상태를 갱신**합니다.

이렇듯 **일정한 주기를 가지고 클라이언트가 서버에 요청을 보내 응답값을 갱신하는 방식**을 폴링이라고 하는데요. 
팀에서 폴링을 선택한 이유는 다름 아닌 HTTP 프로토콜 내에서 가장 간단한 방식으로 실시간성을 표현할 수 있는 방식이기 때문입니다.
물론, 매번 커넥션을 연결해야 하고, 로직상 클라이언트가 호출 예약을 해야 하는 단점도 존재했으나 짧은 개발 주기 특성상 1차적인 구현을 목표로 *KISS원칙을 기반으로 선택하게 된 구현방식입니다.

*KISS원칙(Keep it simple stupid) : 쉽게 말해 간단하게 구현하라는 프로그래밍 격언입니다.

---

### **2) 도착 예정 시간 측정**

그럼 도착 예정 시간은 어떻게 측정이 될까요? 마찬가지로 도식화를 통해 보겠ㅅ습니다.

![img_3.png](imgs/img_3.png)
- API 호출 건수 절약을 위해 **10분 간격으로 현재 위치로부터 약속 장소까지의 대중교통 소요시간을 갱신**합니다.


- 10분이 지났다면 : API를 호출하여 현재위치로부터 대중교통 소요시간을 계산하여 반환합니다.

- 10분이 지나지 않았다면: 남은 시간에서 최근 호출 시간까지의 간격을 카운트 다운한 시간을 반환합니다.

ex) 2분전에 갱신된 소요시간이 10분이라면 8분이 남았다고 반환

---

### **3) 도착 기준**

![img_4.png](imgs/img_4.png)

- 약속 장소까지의 위경도 **직선거리가 300m 반경 안으로 들어오면 도착**으로 취급합니다.

---

### **4) 도착예정정보(ETA) 상태 정의**

다음으로 용어에 대한 통일이 필요했습니다. 약속 이전에 지각 위기인 상태, 약속 이후에 실제로 지각한 상태 등 서로가 생각하는 상황이 다른데도
같은 용어를 사용하는 상황이 생기다보니 도착예정정보인 ETA에 대한 용어를 약속 전후를 기반으로 구분하기로 하였습니다.

**4-1) 약속 시간 전**

- 지각 위기 : 약속 시간 내에 도착할 수 없는 상태
- 도착 예정 : 약속 시간 내에 도착가능한 상태
- 도착 : 약속 장소 반경 300m 이내로 들어온 상태
- 행방불명 : 위치 정보를 허용하지 않은 상태

**4-2) 약속 시간 후**

- 지각 : 약속 장소에 도착하지 않은 상태
- 도착 : 약속 장소 반경 300m 이내로 들어온 상태
- 행방불명 : 위치 정보를 허용하지 않은 상태

---

## **2\. 로직 구현**

이렇게 도메인에 대한 용어를 통일한 이후, 이제 본격적인 로직 구현에 들어갔습니다.
먼저 구현 과정을 이해하기 전에 우리 팀의 도메인 객체들을 설명하면 다음과 같습니다.

![img_5.png](imgs/img_5.png)

- Meeting : 약속
- Mate : 약속에 참여하는 모임원
- ETA : 약속 모임원의 도착 예정 정보
- Member : 회원 정보

---

## **2-1) API 확정하기**

다음으로 안드로이드 측과 협의하여 API를 확정하였는데요.

약속 참여원인 mate에 대한 도착정보 목록 요청이기 떄문에 **POST /v1/meetings/{meetingId}/mates/etas**로 api를 정했습니다.

안드로이드에서는 다음 정보를 담아 요청을 주게 됩니다.

**RequestBody**

| **이름** | **타입** | **설명** | **필수** |
| --- | --- | --- | --- |
| isMissing | Boolean | 위치추적 불가 여부 | O |
| currentLatitude | String | 현재 위도 | X |
| currentLongitude | String | 현재 경도 | X |

위경도 좌표는 디바이스에서 사용자가 위치정보 접근 허용권한을 꺼놓았을 수도 있으므로 null값을 허용했습니다.

**ResponseBody**

| **이름** |   | **타입** | **설명** |  | **필수** |
| --- | --- | --- | --- | --- | --- |
| ownerNickname |   | String | 기기 사용자 닉네임 |   |   |
| mateEtas |   | List<MateEtaResponse> | 참여자 도착 정보 리스트 |   | O |
|   | nickname | String | 참여자 닉네임 |   | O |
|   | status | String | 참여자 ETA 상태 | 지각 위기: LATE\_WARNING   도착 예정: ARRIVAL\_SOON   도착: ARRIVED   지각: LATE   추적 불가: MISSING | O |
|   | durationMinutes | Long | 도착지까지 남은 시간 |   | O |

**ex1) 약속 시간 전**

**: LATE\_WARNING(지각 위기) : 약속 시간까지 도착 못할 예정**

**: ARRIVAL\_SOON(도착 예정) : 약속 시간까지는 도착 가능함**

**: ARRIVED(도착)**

**: MISSING (행방불명) : 위치정보를 추적하지 못함**

```
{
  "ownerNickname" : "카키공주",
	"mateEtas": [
		{
			"nickname": "콜리",
			"status": "LATE_WARNING",
			"durationMinutes": 83
		},
		{
			"nickname": "올리브",
			"status": "ARRIVAL_SOON",
			"durationMinutes": 10
		},
		{
			"nickname": "해음",
			"status": "ARRIVED",
			"durationMinutes": 0
		},
		{
			"nickname": "카키공주",
			"status": "MISSING",
			"durationMinutes": -1
		}
	]
}
```

**ex2) 약속 시간 후 : LATE - ARRIVED 로 도착 여부를 판정하게 된다**

```
{
  "ownerNickname" : "카키공주",
	"mateEtas": [
		{
			"nickname": "콜리",
			"status": "LATE"
			"durationMinutes": 30
		},
		{
			"nickname": "올리브",
			"status": "ARRIVED"
			"durationMinutes": 0
		},
	]
}
```

이렇게 API에 대한 전반적인 내용을 확정하고 페어인 카키와 함께 본격적인 로직 구현에 들어갔습니다.

---

## **2-2) 로직 구현하기**

페어와 함께 코드 작성에 들어가기 전에 도착 예정 정보를 판단하는 알고리즘의 전반적인 흐름을 화이트보드에 쭉 정리해보았습니다.

![img_30.png](imgs/img_30.png)

이를 순서도로 다시 도식화하면 다음과 같습니다.

![img_28.png](imgs/img_28.png)

도식화에 나타나있는 주요 판단 로직은 순서대로 다음과 같습니다
- 도착을 했는가? > ARRIVED
- 위치를 허용했는가? > 안했으면 MISSING
- 약속 시간이 지났는가? > 지났다면 LATE or ARRIVED / 안 지났다면 ARRIVAL_SOON or LATE_WARNING
- 약속 시간 내에 도착이 가능한가? > 가능하면 ARRIVAL_SOON / 지각 예정이면 LATE_WARNING

로직이 복잡한 만큼 bottom-up 방식으로 구현해보기로 했습니다.


---

### **1) 하위 모듈 만들기 : DistanceCalculator**

먼저 두 위 경도 좌표 간에 직선 거리를 계산하는 DistanceCalculator를 만들었습니다.
이 하위 모듈은 도착의 기준인 약속 지점 반경 300m 이내를 판단하는데 사용될 것입니다.

![img_27.png](imgs/img_27.png)

공식은 하버사인 공식을 사용했습니다.

```
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class DistanceCalculator {

    public static double calculate(Coordinates origin, Coordinates target) {
        ...// 하버사인 공식을 활용하여 직선 거리 계산
    }
```
---

### **2) 도착 예정정보 판단 로직 작성하기**

다음으로 EtaStatus를 상황에 따라 반환하는 로직을 작성해주었습니다.

![img_26.png](imgs/img_26.png)

```
public enum EtaStatus {
    ARRIVED,
    ARRIVAL_SOON,
    LATE_WARNING,
    LATE,
    MISSING,
    ;

    public static EtaStatus from(Eta mateEta, LocalDateTime meetingTime, LocalDateTime now, boolean isMissing) {
        // 행방불명이면 missing 반환
        if (isMissing) {
            return MISSING;
        }
        
        // 도착예정정보가 도착상태라면 arrived
        if (mateEta.isArrived()) {
            return ARRIVED;
        }
        
        // 약속 시간에 늦지 않을 예정이고 아직 약속 시간이 지나지 않았다면 arrival soon
        if (!mateEta.willBeLate(meetingTime) && (now.isBefore(meetingTime))) {
            return ARRIVAL_SOON;
        }
        
        if (mateEta.willBeLate(meetingTime)) {
            //약속 시간에 늦을 예정이고 아직 약속 시간 이전이라면 late_warning
            if (now.isBefore(meetingTime)) {
                return LATE_WARNING;
            }
            
            //이미 약속 시간에 늦었다면 late
            return LATE;
        }

        throw new OdyServerErrorException("참여자의 ETA 상태를 판단할 수 없습니다");
    }
}
```

---

### **3) 서비스 정책에 따른 조건식을 private method로 만들어주기**

**\-- 도착 정보를 판단하는 로직 : 위경도 직선거리가 300m이내 + 약속시간 전이라면 => 도착**

![img_25.png](imgs/img_25.png)
```
     private boolean determineArrived(MateEtaRequest mateEtaRequest, Meeting meeting, LocalDateTime now) {
        LocalDateTime meetingTime = meeting.getMeetingTime().withSecond(0).withNano(0);
        double distance = DistanceCalculator.calculate(
                Double.valueOf(mateEtaRequest.currentLatitude()),
                Double.valueOf(mateEtaRequest.currentLongitude()),
                Double.valueOf(meeting.getTarget().getLatitude()),
                Double.valueOf(meeting.getTarget().getLongitude())
        );
        return distance <= ARRIVED_THRESHOLD_METER && (now.isBefore(meetingTime) || now.isEqual(meetingTime));
    }
```

**\- API를 호출해야 하는지 판단하는 로직 : 최초호출이거나, api를 호출한지 10분이 지났다면 호출해야 합니다.**

![img_24.png](imgs/img_24.png)
```
    private boolean isOdysayCallTime(Eta mateEta) {
        return !mateEta.isModified() || mateEta.differenceMinutesFromLastUpdated() >= ODSAY_CALL_CYCLE_MINUTES;
    }
```

---

### **4) EtaService 구현하기**

위에서 구현한 내용을 합쳐 service 단에서 로직을 모두 드러나게 나열하는 식으로 코드를 작성했습니다.

당시 처음으로 구현한 코드의 모습은 다음과 같습니다. 흐름이 잘 드러나지 않고 메서드의 길이가 매우 긴 모습을 볼 수 있습니다.

```
    @Transactional
    public MateEtaResponses findAllMateEtas(MateEtaRequest mateEtaRequest, Long meetingId, Member member) {
        //먼저 위치를 보내준 약속 참여원을 가져온다
        Mate requestMate = findByMeetingIdAndMemberId(meetingId, member.getId());
        Meeting meeting = requestMate.getMeeting(); //약속
        LocalDateTime meetingTime = meeting.getMeetingTime().withSecond(0).withNano(0); //약속시간
        Eta mateEta = findByMateId(requestMate.getId()); //참여원의 도착예정 정보
        LocalDateTime now = LocalDateTime.now().withSecond(0).withNano(0); //현재 시간

        //도착했다면 ETA를 도착상태로 업데이트한다
        if (determineArrived(mateEtaRequest, meeting, now)) {
            mateEta.updateArrived();
        }
        
        // 도착하지 않고 api를 호출해야 한다면(호출한지 10분이 지났다면) 호출하여 시간을 업데이트한다
        if (!mateEta.isArrived() && isOdysayCallTime(mateEta)) {
            RouteTime routeTime = routeService.calculateRouteTime(requestMate.getOrigin(), meeting.getTarget());
            mateEta.updateRemainingMinutes(routeTime.getMinutes());
        }
        
        //약속참여원들의 ETA를 매핑하여 전달한다
        List<MateEtaResponse> mateEtaResponses = etaRepository.findAllByMeetingId(meetingId).stream()
                .map(eta -> MateEtaResponse.of(eta, mateEtaRequest.isMissing(), meetingTime, now))
                .toList();
        return new MateEtaResponses(requestMate.getNicknameValue(), mateEtaResponses);
    }
}
```

분명 동작은 했습니다. 그런데 진짜 동작만 했고 여러 부분에서 몇가지 불편한 부분들이 있었습니다.

---

## **2-3) 정확히 무엇이 불편했을까?**

### **첫째, 중복 코드가 많았습니다.**

- 시간 선후관계 판단에서 나노초를 제거하는 코드 : withNano(0)
- 약속 시간이 지났는지 판단하는 코드 : (now.isBefore(meetingTime) || now.isEqual(meetingTime)); 

### **둘째, 메서드 하나에 너무 많은 책임이 들어있었습니다**

- 메서드 1개가 10줄이 넘었다.
- Eta를 매핑할 때(Eta.from), 파라미터가 4개에 달했다. 
- findAllMateEtas라는 메서드 명은 단지 쿼리형 메서드의 느낌을 준다. 실제로는 eta의 상태가 변경된다. 

### **셋째, 시키지 않고 물어보는 코드가 많았습니다다.**

```java
    @Transactional
    public MateEtaResponses findAllMateEtas(MateEtaRequest mateEtaRequest, Long meetingId, Member member) {
        //먼저 위치를 보내준 약속 참여원을 가져온다
        Mate requestMate = findByMeetingIdAndMemberId(meetingId, member.getId());
        Meeting meeting = requestMate.getMeeting(); //약속
        LocalDateTime meetingTime = meeting.getMeetingTime().withSecond(0).withNano(0); //약속시간
        Eta mateEta = findByMateId(requestMate.getId()); //참여원의 도착예정 정보
        LocalDateTime now = LocalDateTime.now().withSecond(0).withNano(0); //현재 시간
        .....
```

- **묻지말고 시켜라! 원칙에 어긋나는 코드**가 많았습니다.
- 절차지향적으로 필요한 데이터를 메서드 초반에 나열하고, 이를 활용하는 식으로 로직을 짜는 모습을 자주 발견할 수 있었습니다.

---

# **3. 리팩터링 하기**

코드를 다시 검토해보니 결과적으로 마감기한에 쫓겨 OOP 스럽지 못한 코드를 짜내었다는 생각이 들었습니다. 
그래서 페어인 카키에게 리팩터링을 제안했고 너무나 고맙게도 11시 이후에 새벽까지 24시간 카페에 가서 몇 시간 더 리팩터링을 함께 할 수 있었습니다.

## **리팩터링1. 중복 코드 > 객체에게 책임 부여하기**

중복되는 로직을 객체가 처리할 수 있도록 크로스 커팅해주었습니다.

### **\- 중복코드1 : 시간 선후관계 판단에서 나노초를 제거하는 코드 : withNano(0)**

시가느이 선후관계 판단에 있어서 나노초의 영향을 제거하고 싶었는데요. 예를 들어 약속 상에서 10시 1나노초와 10시를 동일 시간으로 판단하고 싶었습니다.
그러나, 이러한 정책이 도입됨으로써 코드 곳곳에 일관성을 위해 나노초를 제거하는 코드인 withNano(0)이 들어가기 시작했습니다.

첫번째 리팩터링으로 이러한 중복코드를 크로스커팅하기 위한 유틸 객체인 TimeUtil 객체를 만들어 나노초 trim 작업을 담당하게 하였습니다.

```
public class TimeUtil {

    public static LocalDateTime trimSecondsAndNanos(LocalDateTime time) {
        return time.withSecond(ROUND_DIGITS)
                .withNano(ROUND_DIGITS);
    }
}
```

---

### **\- 중복 코드2 : 약속 시간이 지났는지 판단하는 코드 : (now.isBefore(meetingTime) || now.isEqual(meetingTime));**

친구의 도착예정정보는 약속이 끝났는지를 기반으로 나뉘게 됩니다. 예를 들어 약속이 끝나지 않았다면 지각 위기인 친구는 약속 시간이 지나는 순간 지각으로 처리됩니다.
이를 위해 약속이 끝났는지를 판단하는 로직이 있어야 했는데요. 기존에는 EtaStatus를 매핑하는 과정에서 코드가 바깥으로 나열되어 있었습니다.

그러나, 약속 시간에 대한 가장 많은 정볼르 가진 meeting에게 약속이 끝났는지 물어보게 하여 조금은 더 객체 지향스러운 코드로 리팩터링해보았습니다.

```
public class Meeting extends BaseEntity { 
    ...

    //약속에게 물어본다 : 너 끝난 약속이니?
    public boolean isEnd() {
        return TimeUtil.nowWithTrim().isAfter(getMeetingTime());
    }

    public LocalDateTime getMeetingTime() {
        return TimeUtil.trimSecondsAndNanos(LocalDateTime.of(date, time));
    }
}
```

또한, 도착 예정정보 서비스는 서버 쪽에서 시간을 카운트다운하여 반환하고 있었는데요. 카운트 다운을 한다는 것은 api로부터 받은 정확한 도착 예정 소요시간을 기준으로 호출이 된 시점과의 간격만큼을 빼어 계산이 됩니다.
예를 들어 2분전에 호출한 api가 10분이 걸린다고 답을 주었다면 카운트다운한 소요시간은 2분이 흘렀기 때문에 8분이 남았다고 반환해주어야 합니다.


이를 판단하기 위해서 가장 최근 api를 호출한 시간, 남은 소요시간 등의 정보를 알고 있는 **ETA 객체에게 카운트 다운한 값을 물어보도록 리팩터링**하였습니다.

```
public class Eta {
   .... 중략 ...

    public long countDownMinutes() {
        LocalDateTime now = TimeUtil.nowWithTrim();
        long minutesDifference = Duration.between(updatedAt, now).toMinutes();
        return Math.max(remainingMinutes - minutesDifference, 0);
    }

    public boolean isArrivalSoon(Meeting meeting) {
        LocalDateTime now = TimeUtil.nowWithTrim();
        LocalDateTime eta = now.plusMinutes(countDownMinutes());
        return (eta.isBefore(meeting.getMeetingTime()) || eta.isEqual(meeting.getMeetingTime())) && !isArrived;
    }
}
```

그 결과 EtaStatus에서 매핑하는 로직이 정말 깔끔하게 빠진 모습을 카키와 체감하여 리팩터링을 진행할 수 있었습니다.
가장 많은 정보를 가진 객체들에게 메시지를 던져 리팩터링한 코드의 모습은 다음과 같습니다.

```
public enum EtaStatus {

    MISSING((eta, meeting) -> eta.isMissing()),
    ARRIVED((eta, meeting) -> eta.isArrived()),
    ARRIVAL_SOON((eta, meeting) -> eta.isArrivalSoon(meeting) && !meeting.isEnd()),
    LATE_WARNING((eta, meeting) -> !eta.isArrivalSoon(meeting) && !meeting.isEnd()),
    LATE((eta, meeting) -> !eta.isArrivalSoon(meeting) && meeting.isEnd()),
    ;

    private final BiPredicate<Eta, Meeting> condition;

    EtaStatus(BiPredicate<Eta, Meeting> condition) {
        this.condition = condition;
    }

    public static EtaStatus of(Eta mateEta, Meeting meeting) {
        return Arrays.stream(values())
                .filter(status -> status.condition.test(mateEta, meeting))
                .findFirst()
                .orElseThrow(() -> new OdyServerErrorException("참여자의 ETA 상태를 판단할 수 없습니다"));
    }
}
```

---

## **리팩터링2. 두꺼운 메서드 > 메서드 분리**

다음으로 메서드의 길이가 너무 길었다는 문제점인데요.
**상태 매핑의 흐름을 담은 순서도를 보면 크게 2가지로 흐름이 나뉜다는 점을 알 수 있었습니다..**

![img_23.png](imgs/img_23.png)

**\- 1. 위치를 보내준 디바이스 사용자의 ETA 상태 업데이트**

**\- 2. 모임 참여원들의 ETA 조회**

두 메서드를 private method로 분리하여 쿼리용 메서드와 명령형 메서드를 분리해주었습니다.

```
    @Transactional
    public MateEtaResponsesV2 findAllMateEtas(MateEtaRequest mateEtaRequest, Mate mate) {
        .....중략...
        
        //1. 나의 ETA 상태 업데이트
        updateMateEta(mateEtaRequest, mateEta, meeting);
        
        //2. 내가 속한 모임원들의 ETA 조회
        return etaRepository.findAllByMeetingId(meeting.getId()).stream()
                .map(eta -> MateEtaResponseV2.of(eta, meeting))
                .collect(Collectors.collectingAndThen(
                        Collectors.toList(),
                        mateEtas -> new MateEtaResponsesV2(mate.getId(), mateEtas)
                ));
    }
```

앞서 요청을 보낸 약속 참여원의 ETA 상태를 업데이트 하는 코드부터 타 모임원들의 ETA를 조회하는 로직까지 모두 하나의 메서드에 담겨있었던 기존의 코드와는 다르게,
이제는 상태를 업데이트하는 하나의 흐름과 모임원들의 ETA를 조회한다는 두 가지 흐름이 메서드 내에서 조금은 수월하게 읽히게 되었습니다.

이렇게 리팩터링을 마친 상태로 프로젝트의 핵심 기능인 실시간 친구 도착 예정정보 공유 기능을 완성할 수 있었습니다.

---

## **4\. 느낀 점**

친구들의 실시간 위치 정보를 확인하는 기능은 프로젝트 오디의 정체성을 나타내는 핵심기능인 만큼, 기능의 정책 설계부터 구현까지 백엔드 팀원 모두가 많은 신경을 쓴 핵심기능입니다.
토론을 통해 팀원들과 직접 서비스 정책을 세우고 설계부터 리팩터링까지의 사이클을 경험하며 도메인에 대한 이해와, 기획의 명확도를 올리는 것이 매우 중요한 일임을 체감했습니다.

예를 들어 어느 정도까지 추상화하여 도착정보를 보여줄 것인가?, 도착 정보는 시간이어야 하는가 거리여야 하는가?, 언제부터 정보를 보여줄 것인가? 도착의 기준은 몇 m가 되어야 하는가? 등의 물음은
코드 구현도 중요하지만 우리가 해결하고자 하는 문제를 명확히 규정하고 사용자들이 겪는 문제의 근원적 수요를 파악했어야 했습니다.

또한 돌아가는 코드를 구현한 이후에 리팩터링을 하며 조금은 악취가 덜 나는 코드로 리팩터링하는 과정은 OOP를 의식하는 과정이 단순히 유지보수를 넘어 가독성이 좋은 코드를 만들어준다는 사실을 느꼈습니다.
조금 더 명확해진 서비스 로직에서 비즈니스의 흐름을 더 명확히 읽게되었던 순간이 일례라 할 수 있을 것 같습니다.

직접 하나의 기능을 맡아 책임감있게 기능을 안정화하는 과정에서 많은 걸 배웠습니다.
폴링, 더 나은 객체지향을 위한 개발자의 태도까지 사람은 항상 겸손해야 하며 점진적으로 더 나은 무언가를 향해 정진해야함을 다시금 되새겼던 기회가 아니었나 싶습니다.

