---
layout: post
title: "서울시 지하철 실시간 위치정보 API 분석"
category: dev
tags: dev-android
comments: true
---

최근 수도권 전철의 실시간 위치정보를 사용해야 할 일이 생겨 서울시 데이터광장에서 API를 사용하여 안드로이드 앱을 제작해 보았다. 

## API 키 발급
서울시 수도권전철 실시간 위치정보 API는 일반적인 Open API와 달리 전용 API키를 발급받아야 한다.
   
![01.png](https://misakachan.moe/assets/img/dev/android/2020-04-02-seoul-subway-api/01.png)
**나의화면** -> **인증키 신청** -> **실시간 지하철 인증키 신청**에서 신청시 1~2일 내로 승인이 난다. 전용 키를 사용하지 않으면 오류가 발생한다.

## API 명세
- baseURL: ``http://swopenAPI.seoul.go.kr/api/subway/(인증키)/xml/realtimePosition/0/5/1호선``

- 파라미터

|변수명|타입|설명|값|
|:-----|:--|:--|:--|
|KEY|String|인증키|인증키|
|TYPE|String|파일타입|xml, json|
|SERVICE|String|서비스명(고정)|realtimePosition|
|START_INDEX|Integer|요청 시작위치|0~|
|END_INDEX|Integer|요청 끝위치|~1000|
|subwayNm|String|지하철 노선명|~~선|

- 지원하는 노선 목록
  - 1호선
  - 2호선
  - 3호선
  - 4호선
  - 5호선
  - 6호선
  - 7호선
  - 8호선
  - 9호선
  - 수인선
  - 분당선
  - 신분당선
  - 경의중앙선
  - 경춘선
  - 공항철도
  - 우이신설선

- 공통 리턴값
  
|출력명|설명|
|:----|:---|
|list_total_count|총 데이터 건수 (정상조회 시 출력됨)|
|RESULT.CODE|요청결과 코드|
|RESULT.MESSAGE|요청결과 메시지|

- 요청 성공시 리턴값(추가)

|출력명|설명|
|:----|:---|
|subwayId|지하철 노선 ID|
|subwayNm|지하철 노선명|
|statnId|지하철역 ID|
|statnNm|**지하철역명**|
|trainNo|**열차번호**|
|lastRecptnDt|최종갱신일자|
|recptnDt|최종갱신시간|
|updnLine|상,하행 구분(0:상행)/2호선 내,외선 구분(0:외선)|
|statnTid|종착역 ID|
|statnTnm|종착역명|
|trainSttus|열차상태(0:진입, 1:도착, 이외:출발)|
|directAt|급행여부(0:아님)|
|lstcarAt|막차여부(0:아님)|

- 요청결과 메시지

|메시지|설명|
|:----|:---|
|INFO-000|정상|
|INFO-100|인증키 무효|
|INFO-200|데이터 없음|
|ERROR-300|필수 파라미터 누락|
|ERROR-301|파라미터 TYPE 누락|
|ERROR-310|파라미터 SERVICE 확인|
|ERROR-331|파라미터 START_INDEX 확인|
|ERROR-332|파라미터 END_INDEX 확인|
|ERROR-333|START_INDEX 또는 END_INDEX가 정수가 아님|
|ERROR-334|END_INDEX보다 START_INDEX가 더 큼|
|ERROR-335|Sample 데이터는 5건 이상 조회 불가능|
|ERROR-336|요청은 한번에 1000건까지만 가능|
|ERROR-500|서버 오류|
|ERROR-600|DB 에러|
|ERROR-601|SQL 오류|

## 기타
지하철 역 ID의 경우, 노선번호와 역번호기 결합되어 10자리를 이룬다. 상위 4자리가 노선번호이며, 하위 6자리는 여유공간으로 역번호에 알파벳이나 보조번호 등이 붙는 경우 0이 아닌 다른 값으로 채워진다. 

예시:
- 1호선 청량리(역번호 124) : 1001000***124***
- 1호선 안양(역번호 P147) : 10010***80147***
- 1호선 서동탄(역번호 P157-1) 1001***801571***

|노선명|노선번호|
|------|------|
|1호선|1001|
|2호선|1002|
|3호선|1003|
|4호선|1004|
|5호선|1005|
|6호선|1006|
|7호선|1007|
|8호선|1008|
|9호선|1009|
|우이신설선|1092|
|경춘선|1067|
|경의중앙선|1063|
|분당수인선|1075|
|신분당선|1077|
|공항철도|1065|

## API 안드로이드에서 사용하기
안드로이드에서 HTTP 통신을 처리하기 위해서 ``Retrofit``이나 ``Volley``같은 것들을 사용할 수 있다. 나의 경우 단순히 데이터를 받아오는 것만이 아니라 특정 데이터만 뽑아오도록 만들고자 했기에 중간 서버를 경유하도록 제작하게 되었다.

### Firebase Fuctions
Firebase는 Functions라는 기능을 제공한다. 여기서는 HTTP 요청을 처리하기 위해 Functions를 구성하였다. 

```javascript
const functions = require("firebase-functions");
const cors = require("cors")({ origin: true });
const axios = require("axios");

const admin = require("firebase-admin");
admin.initializeApp();

exports.trainListByLocation = functions.https.onRequest((req, res) => {
  cors(req, res, async () => {
    if (req.method !== "GET") {
      return res.status(401).json({
        message: "Not allowed"
      });
    }

    const { subwayNm, updnLine, statnId } = req.query;

    const encodedSubNm = encodeURI(subwayNm);
    const updnLineSelect = updnLine;
    const station = statnId;

    let response = await axios
      .get(
        "http://swopenapi.seoul.go.kr/api/subway/(key)/json/realtimePosition/0/200/" +
          encodedSubNm
      )
      .catch(err => {
        return res
          .status(500)
          .json({ messag: "could not request", reason: err });
      });

    let { realtimePositionList } = response.data;
    if (realtimePositionList === undefined && realtimePositionList === null)
      return res.status(500).json({ message: "error" });

    let result = realtimePositionList
      .filter(msg => {
        const { updnLine, statnId } = msg;

        return updnLine === updnLineSelect && statnId === station;
      })
      .map(x => {
        const {
          statnId,
          statnNm,
          trainNo,
          recptnDt,
          updnLine,
          statnTid,
          statnTnm,
          trainSttus,
          directAt,
          lstcarAt
        } = x;
        return {
          trainNo,
          updnLine,
          trainSttus,
          isDirectCar: directAt,
          isLastCar: lstcarAt,
          stationInfo: {
            stationId: statnId,
            stationName: statnNm
          },
          destinationInfo: {
            stationId: statnTid,
            stationName: statnTnm
          },
          receiptTime: recptnDt
        };
      });

    if (result.length === 0)
      return res.status(404).json({ message: "information not found" });
    return res.status(200).json(result);
  });
});
```
>Credit: https://github.com/iwin2471

내가 사용한 functions code다. 리퀘스트를 가공하여 json으로 리턴받는 간단한 function이다.
주의사항은 Firebase Functions에서 Google 외의 사이트로 요청을 보내고 싶을 경우, Blaze 요금제로 업그레이드하여야만 한다는 것이다. 이거 몰라서 좀 헤멨던 기억이 있다.

## Android에서 Retrofit으로 데이터 수신
Retrofit은 Android에서 HTTP 통신을 위한 라이브러리 중 하나다. 여기서는 Retrofit2를 사용할 것인데, Retrofit에 대한 설명은 다른 포스팅에서 하도록 하고, 오늘은 데이터 수신 과정만 보도록 하겠다.
>*TrainInfo.kt*
```Kotlin
data class TrainInfo(
    @SerializedName("trainNo")
    var trainNo: String = "",

    @SerializedName("updnLine")
    var updnLine: String = "",

    @SerializedName("trainSttus")
    var trainSttus: String? = "",

    @SerializedName("isDirectCar")
    var isDirectCar: String? = "",

    @SerializedName("isLastCar")
    var isLastCar: String? = "",

    @SerializedName("stationInfo")
    var stationInfo: Station? = Station(),

    @SerializedName("destinationInfo")
    var destinationInfo: Station? = Station(),

    @SerializedName("receiptTime")
    var receiptTime: String? = ""
) : Serializable
```
>*Station.kt*
```Kotlin
data class Station (
    @SerializedName("stationName")
    var name: String = "",
    var id:String = "",
    @SerializedName("stationId")
    var strId:String = ""
)
```

Retrofit을 위해 DTO 클래스를 생성하고, Interface를 만든다.

```Kotlin
interface ITrainInfo {
    @GET("/trainListByLocation")
    fun trainListByLocation(
        @Query("subwayNm")
        subwayNm:String,
        @Query("updnLine")
        updnLine:String,
        @Query("statnId")
        stationId:String
    ) : Call<List<TrainInfo>>
}
```
Firebase Function을 사용하기 때문에, baseUrl은 펑션 이름이 되었다.  

```Kotlin
val client: Retrofit =
            Retrofit.Builder().baseUrl("baseURL goes here")
                .addConverterFactory(GsonConverterFactory.create())
                .build()
val service = client.create(ITrainInfo::class.java)
val call = service.trainListByLocation(subwayNm, updnLine, stationId)
call.enqueue(object : Callback<List<TrainInfo>> {
    override fun onFailure(call: Call<List<TrainInfo>>, t: Throwable) {
        Log.d("Error", "Error while get data")
    }

    override fun onResponse(call: Call<List<TrainInfo>>, response: Response<List<TrainInfo>>) {
        if(response.isSuccessful) {
            response.body()?.let {
                for(info in it)
                    Log.d("Data", "${info.trainNo}\n${info.receiptTime}\n${info.stationInfo?.name}\n${info.destinationInfo?.name}")
            }
        }
    }
})
```
```subwayNm```, ```updnLine```, ```stationId```를 파라미터로 넘겨 요청을 받아오면 DTO 오브젝트 형태로 결과값이 돌아온다. onResponse에서 이후의 데이터 처리를 진행하면 된다.

