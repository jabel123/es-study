# 데이터 모델링

ES에서는 색인할 때 문서의 데이터 유형에 따라 필드에 적절한 데이터 타입을 지정해야 한다. 이러한 과정을 매핑이라고 하며, 매핑은 색인될 문서의 데이터 모델링이라고도 할 수 있다. 사전에 매핑을 설정하면 지정된 데이터 타입으로 색인되지만 매핑을 설정해 두지 않으면 엘라스틱서치가 자동으로 필드를 생성하고 필드 타입까지 결정한다. 필드 데이터 타입이 자동으로 지정될 경우 예기치 않은 문제를 일으킬 수 있기 때문에 이 매핑과정은 아주 중요하다.

## 매핑 API 이해하기
매핑은 색인시 데이터가 어디에 어떻게 저장될지를 결정하는 설정이다. 데이터베이스의 스키마에 대응하는 개념이라고도 할 수 있는데 인덱스에 추가되는 각 데이터 타입을 구체적으로 정의하는 일이다. 문서에 존재하는 필드의 속성을 정의할 때 각 필드 속성에는 데이터 타입과 메타데이터가 포함된다. 이를 통해 색인 과정에서 문서가 어떻게 역색인으로 변환되는지를 상세하게 정의할 수 있다. 
 
### 매핑 인덱스 만들기

인덱스 매핑을 다음과 같이 만들어보는 예제를 작성

|매핑명|필드명|필드 타입|
|---|---|---|
|인덱스키|movieCd|keyword|
|영화제목_국문|movieNm|text|
|영화제목_영문|movieNmEn|text|
|제작연도|prdtYear|integer|
|개봉연도|openDt|integer|
|영화유형|typeNm|keyword|
|제작상태|prdtStatNm|keyword|
|제작국가(전체)|nationAlt|keyword|
|장르(전체)|genreAlt|keyword|
|대표 제작국가|repNationNm|keyword|
|대표 장르|repGenreNm|keyword|
|영화감독명|directors.peopleNm|object -> keyword|
|제작사코드|directors.companyCd|object -> keyword|
|제작사명|directors.companyNm|object -> keyword|

다음과 같이 인덱스를 생성한다.
```
PUT movie_search
{
    "settings": {
        "number_of_shards": 5,
        "number_of_replicas": 1
    },
    "mappings": {
        "_doc": {
            "properties": {
                "movieCd": {
                    "type": "keyword"
                },
                "movieNm": {
                    "type": "text",
                    "analyzer": "standard"
                },
                "movieNmEn": {
                    "type": "text",
                    "analyzer": "standard"
                },
                "prdtYear": {
                    "type": "integer"
                },
                "openDt": {
                    "type": "integer"
                },
                "typeNm": {
                    "type": "keyword"
                },
                "prdtStatNm": {
                    "type": "keyword"
                },
                "nationAlt": {
                    "type": "keyword"
                },
                "genreAlt": {
                    "type": "keyword"
                },
                "repNationNm": {
                    "type": "keyword"
                },
                "repGenreNm": {
                    "type": "keyword"
                },
                "companies": {
                    "properties": {
                        "companyCd": {
                            "type": "keyword"
                        },
                        "companyNm": {
                            "type": "keyword"
                        }
                    }
                },
                "directors": {
                    "properties": {
                        "peopleNm": {
                            "type": "keyword"
                        }
                    }
                }
            }
        }
    }
}
```

생성후 결과값
```
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "index": "movie_search"
}
```

### 매핑정보 확인
```
GET movie_search/_mapping
```

### 매핑 파라미터
매핑 파라미터는 색인할 필드의 데이터를 어떻게 저장할지에 대한 다양한 옵션을 제공한다. 이러한 옵션은 필드에 매핑 정보를 설정할 때 유용하게 사용할 수 있다.

**analyzer**  
해당 필드의 데이터를 형태소 분석하겠다는 의미의 파라미터이다. 색인과 검색시 지정한 분석기로 형태소 분석을 수행한다. text 데이터 타입의 필드는 analyzer 매핑 파라미터를 기본적으로 사용해야 한다. 별도의 분석기를 지정하지 않으면 Standard Analyzer로 형태소 분석을 수행한다.

**normalizer**  
normalizer 매핑 파라미터는 term query에 분석기를 사용하기 위해 사용된다. 예를 들어 keyword 데이터 타입의 경우 원문을 기준으로 문서가 색인되기 때문에, cafe, Cafe, CAFE는 서로 다른 문서로 인식한다. 하지만 해당 유형을 normalizer를 통해 분석기에 asciifolding과 같은 필터를 사용하면 같은 데이터로 인식하게 되어있다.

**boost**  
필드에 가중치를 부여한다. 가중치에 따라 유사도 점수(_score)가 달라지기 때문에 boost설정시 검색 결과의 노출 순서에 영향을 준다. 만약 색인 시점에 boost 설정을 하게된다면 재색인하지 않는 이상 가중치 변경을 할 수 없기 때문에 주의해서 사용해야 한다. 가급적이면 검색 시점에만 사용하는 것을 권장한다. (ES 7.0부터는 boost기능이 제거됐는데,, 이유는 루씬에서 해당 기능이 제거되었기 때문)

**coerce**  
색인시 자동 변환을 허용할지 여부를 결정하는 파라미터. 예를들면 "10"과 같은 숫자 형태의 문자열이 integer타입의 필드에 들어온다면 자동 형변환을 수행하지만, coerce설정을 미사용으로 변경한다면 색인에 실패할 것.

**copy_to**
매핑 파라미터를 추가한 필드의 값을 지정한 필드로 복사한다. 예컨대 keyword타입의 필드에 copy_to매핑 파라미터를 사용해 다른 필드로 값을 복사하면 복사된 필드에서는 text타입을 지정해 형태소 분석을 할 수 도 있다.

**fielddata**  
엘라스틱서치가 힙 공간에 생성하는 메모리 캐시다. 과거에는fielddata를 많이 사용했지만 반복적인 메모리 부족 현상과 잦은 GC로 현재는 거의 사용하지 않는다. 

**doc_values**  
ES에서 사용하는 기본 캐시다. text타입을 제외한 모든 타입에서 기본적으로 doc_values 캐시를 사용한다. doc_values는 루씬을 기반으로 하는 캐시 방식이다. 과거에는 캐시를 모두 메모리에 올려 사용했으나 현재는 doc_values를 이용함으로써 힙 사용에 대한 부담을 없애고 운영체제의 파일 시스템 캐시를 통해 디스크에 있는 데이터에 빠르게 접근할 수 있다. 이로 인해 GC 의 비용이 들지 않으면서도 메모리 연산과 비슷한 성능을 보여준다.

필드를 정렬, 집계할 필요가 없고 스크립트에서 필드 값에 액세스할 필요가 없다면 디스크 공간을 절약하기 위해 doc_values를 비활성화할수도 있다. 한번 비활성화된 필드는 인덱스를 재색인하지 않는 한 변경이 불가능하다.

**dynamic**  
매핑에 필드를 추가할때 동적으로 생성할지, 생성하지 않을지를 결정한다.
|인자|설명|
|---|---|
|true|새로 추가되는 필드를 매핑에 추가한다.|
|false|새로 추가되는 필드를 무시한다. 해당 필드는 색인되지 않아 검색할 수 없지만 _source에는 표시된다.|
|strict|새로운 필드가 감지되면 예외가 발생하고 문서 자체가 색인되지 않는다. 새로 유입되는 필드는 사용자가 매핑에 명시적으로 추가해야 한다.|

**enabled**  
검색 결과에는 포함하지만 색인은 하고 싶지 않은 경우도 있다. 메타 성격의 데이터가 그런데, 일반적인 게시판이라면 제목과 요약글만 색인하고 날짜와 사용자 ID는 색인하지 않는 경우다.  enabled를 false로 설정하면 _source에는 검색이 되지만 색인은 하지 않는다.

**format**  
ES는 날짜/시간을 문자열로 표시한다. 이때 날짜/시간을 문자열로 변경할 때 미리 구성된 포맷을 사용할 수 있다.  
*날짜/시간 데이터 타입의 날짜 형식*
|포맷|날짜 형식|비고|
|---|---|---|
|basic_date|yyyyMMdd|년도/월/일|
|basic_date_time|yyyyMMdd'T'HHmmss.SSSZ|년도/월/일/T/시/분/초/밀리초/Z|
|basic_time|HHmmss.SSS|시/분/초/밀리초/Z|
|date/strict_date|yyyy-MM-dd|년도/월/일|
|date_hour_minute_second/strict_date_hour_minute_second|yyyy-MM-dd'T'HH;mm:ss.|년도/월/일/T/시/분/초|

**ignore_above**  
필드에 저장되는 문자열이 지정된 크기를 넘어서면 빈 값으로 색인한다. 지정된 크기만큼만 색인하는게 아니라 빈값으로 색인하므로 주의 필요

**ignore_malformed**  
매핑시 잘못된 데이터타입을 색인할 경우 해당 필드만 무시하고 문서 색인이 가능하도록 함

**index**  
필드값을 색인할지를 결정한다. 기본값은 true이며, false로 변경하면 해당 필드를 색인하지 않는다.

**fields**
다중 필드를 설정할 수 있는 옵션이다. 필드 안에 또 다른 필드의 정보를 추가할 수 있어 같은 string값을 각각 다른 분석기로 처리하도록 설정할 수 있다.

**norms**  
문서의 _score값 계산에 필요한 정규화 인수를 사용할지 여부를 설정한다. 기본값은 true다. _score계산이 필요없거나 단순 필터링 용도로 사용하는 필드는 비활성화해서 디스크 공간을 절약할 수 있다.

**null_value**  
엘라스틱서치는 색인시 문서에 필드가 없거나 필드의 값이 null이면 색인시 필드를 생성하지 않는다. 이 경우 null_value를 설정하면 문서의 값이 null이더라도 필드를 생성하고 그에 해당하는 값으로 저장한다. 

**position_increment_gap**  
배열 형태의 데이터를 색인할 때 검색의 정확도를 높이기 위해 제공하는 옵션이다. 필드 데이터 중 단어와 사이의 간격을 허용할지를 설정한다. 검색 시 단어와 단어 사이의 간격을 기준으로 일치하는 문서를 찾는 데 필요하다. 

**properties**  
오브젝트 타입이나 중첩 타입의 스키마를 정의할 때 사용되는 옵션으로 필드의 타입을 매핑한다. 오브젝트 필드 및 중첩 필드에는 properties라는 서브 필드가 있다. 

**serach_analyzer**  
일반적으로 색인과 검색 시 같은 분석기를 사용한다. 만약 다른 분석기를 사용하고 싶은 경우 search_analyzer를 설정해서 검색 시 사용할 분석기를 별도로 지정할 수 있다.

**store**  
필드 값을 저장해 검색 결과에 값을 포함하기 위한 매핑 파라미터다. 기본적으로 엘라스틱서칠에서는 _source에 색인된 문서가 저장된다. 하지만 store 매핑 파라미터를 사용하면 해당 필드를 자체적으로 저장할 수 있다. 예를 들어 10개의 필드가 존재하고 해당 필드에 데이터를 매핑한 상태라면 _source를 로드해서 해당 필드를 찾는 것보다 사용할 각 필드만 로드해서 사용하는 편이 효율적이다. 하지만 해당 매핑 파라미터를 사용하면 디스크를 더 많이 사용한다. 

**term_vector**  
루씬에서 분석된 용어의 정보를 포함할지 여부를 결정하는 매핑 파라미터다. 설정 가능한 인자는 다음과 같다.  
|인자|설명|
|---|---|
|no|텀벡터를 지정하지 않는다.|
|yes|필드와 용어만 저장한다.|
|with_positions|용어, 용어의 시작과 끝 위치를 저장한다.|
|with_offsets|용어, 문자 오프셋을 저장한다.|
|with_positions_offsets|용어, 용어의 시작과 끝 위치, 문자 오프셋을 모두 저장한다.|

## 메타 필드
메타 필드는 엘라스틱서치에서 생성한 문서에서 제공하는 특별한 필드다. 이것은 메타데이터를 저장하는 특수 목적의 필드로서 이를 이용하면 검색 시 문서를 다양한 형태로 제어하는 것이 가능해진다. 