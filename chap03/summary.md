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

### _index 메타 필드
_index 메타 필드는 해당 문서가 속한 인덱스의 이름을 담고 있다. 이를 이용해 검색된 문서의 인덱스명을 알 수 있으며, 해당 인덱스에 몇 개의 문서가 있는지 확인할 수 있다.

### _type 메타 필드
_type 메타 필드는 해당 문서가 속한 매핑의 타입 정보를 담고 있다. 이를 이용해 해당 인덱스 내부에서 타입별로 몇 개의 문서가 있는지 확인할 수 있다. 

### _id 메타 필드
_id 메타 필드는 문서를 식별하는 유일한 키 값이다. 한 인덱스에서 색인된 문서마다 서로 다른 키 값을 가진다. 

### _uid 메타필드
_uid 메타 필드는 특수한 목적의 식별키다. '#' 태그를 사용해 _type과 _id값을 조합해 사용한다. 하지만 내부적으로만 사용되기 때문에 검색 시 조회되는 값은 아니다.

### _source 메타 필드
_source 메타 필드는 문서의 원본 데이터를 제공한다. 내부에는 색인시 전달된 원본 JSON 문서의 본문이 저장돼 있다. 일반적으로 원본 JSON 문서를 검색 결과로 표시할 때 사용한다. 

_reindex API나 스크립트 사용해 해당 값을 계산할 때 해당 메타 필드를 활용할 수 있다. 예제를 하나 살펴보자. movie_search 인덱스에 있는 문서 중 movieCd 값이 "20173732"인 값만 조회해서 재색인한다고 가정했을 때 재색인되는 인덱스에서 prdtYear 값을 변경해 보겠다. 

먼저 재색인을 위해 reindex_movie 인덱스를 생성한다.
```
PUT /reindex_movie
```

재색인할 인덱스가 생성되면 reIndex API를 ㅣㅇ용해 재색인을 수행한다. prdtYear 값을 변경하기 위해 필드에 접근할 표기법이 필요한데, 이때 스크립트를 이용해 ctx._source.prdtYear 형태로 prdtYear 필드에 접근할 수 있다.

```
POST /_reindex
{
  "source": {
    "index": "movie_search",
    "query" : {
      "match" : {
        "movieCd": "20173732"
      }
    }
  },
  "dest": {
    "index": "reindex_movie"
  },
  "script": {
    "source": "ctx._source.prdtYear++"
  }
}
```
### _all 메타 필드
_all 메타 필드는 색인에 사용된 모든 필드의 정보를 가진 메타필드다. 모든 필드의 내용이 하나의 텍스트로 합쳐져서 제공된다. 특정 필드가 아닌 문서 전체 필드에서 특정 키워드를 검색한다면 _all 메타 필드를 사용하면 된다.

이를 이용하면 통합검색을 구현시 유리할 수 있으나, _all 메타 필드는 데이터 크기를 너무 많이 차지하는 문제가 있어 엘라스틱서치 6.0이상부턴는 폐기됐다. 그래서 필드 복사가 필요한 경우 copy_to 파라미터를 사용해야 하고, copy_to를 이용하면 _all과 동일한 효과를 낼 수 있다.

### _routing 메타 필드
_routing 메타 필드는 특정 문서를 특정 샤드에 저장하기 위해 사용자가 지정하는 메타 필드다. 기본적으로 색인을 하면 해당 문서는 다음 수식에 따라 문서 id를 이용해 문서가 색인될 샤드를 결정한다. 별도의 설정 없이 문서를 색인하면 문서는 샤드에 골고루 분산되어 저장된다. 

## 필드 데이터 타입
매핑 설정을 위해서는 엘라스틱서치에서 제공하는 데이터 타입으로 어떠한 종류가 있는지 정확하게 이해하는 것이 중요하다. 이를 바탕으로 데이터의 종류와 형태에 따라 데이터 타입을 선택적으로 사용해야 한다. 

```
- keyword, text와 같은 문자열 데이터 타입
- date, long, double, integer, boolean, ip 같은 일반적인 데이터 타입
- 객체 또는 중첩문과 같은 JSON 계층의 특성의 데이터 타입
- geo_point, geo_shape 같은 특수한 데이터 타입
```

### keyword 데이터 타입
Keyword 데이터 타입은 말 그대로 키워드 형태로 사용할 데이터에 적합한 데이터 타입이다. Keyword 타입을 사용할 경우 별도의 분석기를 거치지 않고 원문 그대로 색인하기 떄문에 특정 코드나 키워드등 정형화된 콘텐츠에 주로 사용된다. es의 일부 기능은 형태소 분석을 하지 않아야만 사용이 가능한데 이  경우에도 Keyword 데이터 타입이 사용된다. 

Keyword 데이터 타입은 아래의 항목에 많이 사용된다.
- 검색시 필터링 되는 항목
- 정렬이 필요한 항목
- 집계 해야 하는 항목

Keyword 데이터 타입의 주요 파라미터  

|이름|설명|
|---|---|
|boost|필드의 가중치로, 검색 결과  정렬에 영향을 준다. 기본값은 1.0으로서 1보다 크면 점수가 높게 오르고, 적으면 점수가 낮게 오른다. 이를 이용해 검색에 사용된 키워드와 문서간의 유사도 스코어 값을 계산할 때 필드의 가중치 값을 얼마나 더 줄것인지를 판단한다.
|doc_values|필드를 메모리에 로드해 캐시로 사용한다. 기본값은 true다.|
|index|해당 필드를 검색에 사용할지를 설정한다. 기본값은 true다.|
|null_value|기본적으로 엘라스틱서치는 데이터의 값이 없으면 필드를 생성하지 않는다. 데이터의 값이 없는 경우 null롤필드의 값을 대체할지를 설정한다.|
|store|필드 값을 필드와 별도로 _source에 저장하고 검색 가능하게 할지를 설정한다. 기본값은 false다.|

### Text 데이터 타입

Text 데이터 타입을 이용하면 색인 시 지정된 분석기가 칼럼의 데이터를 문자열 데이터로 인식하고 이를 분석한다. 만약 별도의 분석기를 정의하지 않았다면 기본적으로 Standard Analyzer를 사용한다. 영화의 제목이나 영화의 설명글과 같이 문장 형태의 데이터에 사용하기 적합한 데이터 타입이다. 

Text 데이터 타입은 전문 검색이 가능하다는 점이 가장 큰 특징이다. Text 타입으로 데이터를 색인하면 전체 텍스트가 토큰화되어 생성되며 특정 단어를 검색하는 것이 가능해진다. 
Text 데이터 타입을 사용할 경우 필드에 검색 뿐 아니라 정렬이나 집계 연산을 사용해야 할 때가 있다. 이러한 경우 Text 타입과 Keyword 타입을 동시에 갖도록 멀티 필드로 설정할 수 있다. 

*Text type의 주요 파라미터*
|이름|설명|
|---|---|
|analyzer|인덱스와 검색에 사용할 형태소 분석기를 선택한다. 기본값은 Standard Analyzer이다.|
|boost|필드의 가중치로, 검색 결과 정렬에 영향을 준다. 기본값은 1.0으로 1보다 크면 점수가 높게 오르고, 적으면 점수가 낮게 오른다.|
|fielddata|정렬, 집계, 스크립트 등에서 메모리에 저장된 필드 데이터를 사용할지를 설정한다. 기본값은 false이다.|
|index|해당 필드를 검색에 사용할지를 설정한다. 기본값은 true다.|
|norms|유사도 점수를 산정할 때 필드 길이를 고려할지를 결정한다. 기본값은 true다.|
|store|필드 값을 필드와 별도로 _source에 저장하고 검색 가능하게 할지를 설정한다. 기본값은 false다.|
|search_analyzer|검색에 사용할 형태소 분석기를 선택한다.|
|similarity|유사도 점수를 구하는 알고리즘을 선택한다. 기본값은 BM25다.|
|term_vector|Analyzed 필드에 텀벡터를 저장할지를 결정한다. 기본값은 no다.|

### Array 데이터 타입

데이터는 대부분 1차원 으로 표현되지만 2차원으로 존재하는 경우도 있다. 언어의 값으로 영어와 한국어 라는 두개의 데이터를 입력하고 싶을 경우 Array 데이터 타입을 사용해야 한다. Array 타입은 문자열이나 숫자처럼 일반적인 값을 지정할 수 도 있지만 객체 형태로도 정의할수 있따. 한가지 주의할 점은 Array타입에 저장되는 값은 모두 같은 타입으로만 구성해야 한다는 점이다.
- 문자열 배열 : ["one", "two"]
- 정수 배열 : [1, 2]
- 객체 배열 : [{"name" : "Marry", "age" : 12}, {"name" : "Marry", "age" : 12}]

### Numeric 데이터 타입
ES에서 숫자 데이터 타입은 여가지 종류가 제공된다. 숫자 데이터 타입이 여러개 제공되는 이유는 데이터 크기에 알맞은 타입을 제공하므로써 색인과 검색을 효율적으로 하기 위해서다 

*Numeric 데이터 타입*
|이름|설명|
|---|---|
|long|최솟값과 최댓값을 가지는 부호 있는 64비트 정수.|
|integer|최솟값과 최댓값을 가지는 부호 있는 32비트 정수.|
|short|최솟값과 최댓값을 가지는 부호 있는 16비트 정수.|
|byte|최솟값과 최댓값을 가지는 부호 있는 8비트 정수.|
|double|64비트 부동 소수점을 갖는 수|
|float|32비트 부동 소수점을 갖는 수|
|half_float|16비트 부동 소수점을 갖는 수|

### Date 데이터 타입
Date 타입은 JSON 포맷에서 문자열로 처리된다. 날짜는 다양하게 표현될 수 있기 때문에 올바르게 구문 분석될 수 있게 날짜 문자열 형식을 명시적으로 설정해야 한다. 만약 벼롣의 형식을 지정하지 않을 경우 기본 형식인 "yyyy-MM-ddTHH;mm:ssZ"로 지정된다. 
Date 타입은 다음과 같이 크게 세 가지 형태를 제공한다. 세 가지 중 어느 것을 사용해도 내부적으로 UTC의 밀리초 단위로 변환해 저장한다.
- 문자열이 포함된 날짜 형식: "2018-04-20","2018/04.20","2018-04-20 10:50:00", "2018/04/20 10:50:00"
- ISO_INSTANT 포맷의 날짜 형식 : " 2018-04-10T10:50:00Z"
- 밀리초: 113513535315

다음은 칼럼에 Date 데이터 타입을 사용하는 예다.
```
PUT movie_text/_mapping/_doc
{
    "properties": {
        "date": {
            "type": "date", "format": "yyyy-MM-dd HH:mm:ss"
        }
    }
}
```
### Ranage데이터 타입
Range 데이터 타입은 범위가 있는 데이터를 저장할 때 사용하는 데이터 타입이다. 만약 데이터의 범위가 10~20의 정수라면 10, 11, 12 ...20 까지의 숫자를 일일이 지정하는 것이 아니라 데이터의 시작과 끝만 정의하면 된다. 

|이름|설명|
|---|---|
|integer_range|최솟값과 최댓값을 갖는 부호 있는 32비트 정수 범위|
|float_range|부동 소수점 값을 갖는 32비트 실수 범위|
|long_range|최솟값과 최댓값을 갖는 부호 있는 64비트 정수의 범위|
|double_range|부동 소수점 값을 갖는 64비트 실수 범위|
|date_range|64비트 정수 형태의 밀리초로 표시되는 날짜값의 범위|
|ip_range|IPv4, IPv6 주소를 지원하는 IP값|

### Boolean 데이터 타입

Booelan 데이터 타입은 참과 거짓이라는 두 논리값을 가지는 데이텉 ㅏ입이다. 참과 거짓 값을 문자열로 표현하는 것도 가능하다.

|이름|설명|
|---|---|
|참|true, "true"|
|거짓|false, "false"|

### Geo-Point 데이터 타입
위도, 경도 등 위치 정보를 담은 데이터를 저장할 때 Geo-Point 데이터 타입을 사용할 수 있다. 위치 기반 쿼리를 이용해 반경 내 쿼리, 위치 기반 집계, 위치별 정렬 등을 사용할 수 있기 때문에 위치 기반 데이터를 색인하고 검색하는 데 유용하다. 

### IP 데이터 타입
IP주소와 같은 데이터를 저장하는 데 사용한다. IPv4나 IPv6를 모두 지저앟ㄹ 수 있다.

### Object 덷이터 타입
JSON포맷의 문서는 내부 객체를 계층적으로 포함할 수 있다. 문서의 필드는 단순히 값을 가질 수도 있지만 복잡한 형태의  또 다른 문서를 포함하는 것도 가능하다. 이처럼 값으로 문서를 가지는 필드의 데이터 타입을 Object 타입이라고 한다. Object 데이터 타입을 정으히ㅏㄹ 때는 다른 데이터 타입과 같이 특정 키워드를 이용하지 않는다. 단지 필드값으로 다른 문서의 구조를 입력하면 된다. 

### Nested 데이터 타입

Nested 데이터 타입은 Object 객체 배열을 독립적으로 색인하고 질의하는 형태의 데이터 타입이다. 앞서 살펴본 바와 같이 특정 필드 내에 Object형식으로 JSON 포맷을 표현할 수 있다. 그리고 필드에 객체가 배열 형태로도 저장될 수 있다. 

데이터가 배열 형태로 저장되면 한필드 내의 검색은 기본적으로 OR 조건으로 검색된다. 이러한 특성 탓에 저장되는 데이터의 구조가 조금만 복잡해지면 모호한 상황이 일어날 수 있다.(Array가 OR조건으로 검색될시 불필요하게 노출되는 문제)

위의 문제는 Nested 데이터 타입을 이용하여 검색할 때 일치하는 문서만 정확하게 출력할 수 있다.

## 엘라스틱서치 분석기

ES의 분석기는 검색엔진을 처음 접하는 사용자에게는 조금 이해하기 어려운 부분이다. 특정 단어를 검색했을 때 결과가 없거나 예기치 않은 결과가 나오는 경우가 종종 생기는데, 이 경우 실제 인덱스의 정보가 어떻게 저장돼 있는지 이해하지 못하고 분석기를 구성했을 확률이 높다. 

### 텍스트 분석 개요
ES는 루씬을 기반으로 구축된 텍스트 기반 검색엔진이다. 루씬은 내부적으로 다양한 분석기를 제공하는데, ES는 루씬이 제공하는 분석기를 그대로 활용한다. 텍스트 분석을 이해하려면 루씬이 제공하는 분석기가 어떻게 동작하는지를 먼저 이해하는 것이 중요하다. 

엘라스틱서치는 문서를 색인하기 전에 해당 문서의 필드 타입이 무엇인지 확인하고 텍스트 타입이면 분석기를 이용해 이를 분석한다. 텍스트가 분석되면 개별 텀으로 나뉘어 형태소 형태로 분석된다. 해당 형태소는 특정 원칙에 의해 필터링되어 단어가 삭제되거나 추가, 수정되고 최종적으로 역색인된다. 

**형태소 분석 수행**
```
POST _analyze
{
  "analyzer" : "standard",
  "text":"우리나라가 좋은나라, 대한민국 화이팅"
}
```
**결과**
```
{
  "tokens" : [
    {
      "token" : "우리나라가",
      "start_offset" : 0,
      "end_offset" : 5,
      "type" : "<HANGUL>",
      "position" : 0
    },
    {
      "token" : "좋은나라",
      "start_offset" : 6,
      "end_offset" : 10,
      "type" : "<HANGUL>",
      "position" : 1
    },
    {
      "token" : "대한민국",
      "start_offset" : 12,
      "end_offset" : 16,
      "type" : "<HANGUL>",
      "position" : 2
    },
    {
      "token" : "화이팅",
      "start_offset" : 17,
      "end_offset" : 20,
      "type" : "<HANGUL>",
      "position" : 3
    }
  ]
}

```

### 역색인 구조

루씬의 색인은 역색인이라는 특수한 방식으로 구조화돼 있다.

역색인 구조를 간단하게 정리하면 다음과 같다.
- 모든 문서가 가지는 단어의 고유 단어 목록
- 해당 단어가 어떤 문서에 속해 있는지에 대한 정보
- 전체 문서에 각 단어가 몇 개 들어있는지에 대한 정보
- 하나의 문서에 단어가 몇 번씩 출현했는지에 대한 빈도

**ex**
```
elasticsearch is cool
Elasticsearch is great
```

**토큰 정보**
|토큰|문서번호|텀의위치|텀의빈도|
|---|---|---|---|
|elasticsearch|문서1|1|1|
|Elasticsearch|문서2|1|1|
|is|문서1,2|2,2|2|
|cool|문서1|3|1|
|great|문서2|3|1|

이때 elasticsearch를 검색어로 지정할떄 대소문자 문제로 인해 다른 단어로 인식하는데, 가장 간단한 해결책은 색인전에 텍스트 전체를 소문자로 변환한다음 색인하는 것이다. 그렇게 되면 두 개의 문서가 elasticserach라는 토큰으로 나올 것이다.

**토큰 정보**
|토큰|문서번호|텀의위치|텀의빈도|
|---|---|---|---|
|elasticsearch|문서1,2|1,1|2|
|is|문서1,2|2,2|2|
|cool|문서1|3|1|
|great|문서2|3|1|

색인한다는 것은 역색인 파일을 만든다는 것이다. 그렇다고 원문 자체를 변경한다는 의미는 아니다. 따라서 색인 파일에 들어갈 토큰만 변경되어 저장되고 실제 문서의 내용은 변함없이 저장된다. 색인할 때 특정한 규칙과 흐름에 의해 텍스트를 변경하는 과정을 분석이라고 하고 해당 처리는 분석기라는 모듈을 조합해서 이뤄진다. 

### 분석기의 구조
```
1. 문장을 특정한 규칙에 의해 수정한다.
2. 수정한 문장을 개별 토큰으로 분리한다.
3. 개별 토큰을 특정한 규칙에 의해 변경한다.
```
이 세가지 동작이 분석기의 전부라 할 수 있따. 이 세가지 동작은 특성에 의해 각각 다음과 같은 용어로 불린다.
```
CHARACTER FILTER
문장을 분석하기 전에 입력 텍스트에 대해 특정한 단어를 변경하거나 HTML과 같은 태그를 제거하는 역할을 하는 필터.

TOKENNIZER FILTER
Tokenizer Filter는 분석기를 구성할 때 하나만 사용할 수 있으며 텍스트를 어떻게 나눌 것인지를 정의한다. 한글을 분해할 때는 한글 형태소 분석기의 Tokenizer를 사용하고, 영문 분석할 떄는 영문 분석기의 Tokenizer를 사용하는 등 상황에 맞게 적절한 Tokenizer를 사용한다.

TOKEN FILTER
토큰화된 단어를 하나씩 필터링해서 사용자가 원하는 토큰으로 반환한다. 
```

#### 분석기 사용 방법
엘라스틱서치는 루씬에 존재하는 기본 분석기를 별도의 정의 없이 사용할 수 있게 미리 정의해서 제공한다. 앞서 사용해본 "standard"라는 키워드는 루씬의 StandardAnalyzer를 의미한다. 이러한 분석기를 사용하기 위해 엘라스틱 서치에서는 _analyze API를 제공한다.


**분석기를 이용한 분석**
```
POST _analyze
{
  "analyzer": "standard",
  "text": "캐리비안의 해적"
}
```

**필드를 이용한 분석**
```
POST movie_analyzer/_analyze
{
  "field": "title",
  "text": "캐리비안의 해적"
}
```

**색인과 검색 시 분석기를 각각 설정**

인덱스를 생성할 때 색인용과 검색용 분석기를 각각 정의하고 적용하고자 하는 필드에 원하는 분석기를 지정하면 된다. 

```
PUT movie_analyzer 
{
  "settings" : {
    "index": {
      "number_of_shards": 5,
      "number_of_replicas": 1
    },
    "analysis": {
      "analyzer": {
        "movie_lower_test_analyzer": {
          "type":"custom",
          "tokenizer": "standard",
          "filter":[
            "lowercase"
          ]
        },
      "movie_stop_test_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter":[
            "lowercase",
            "english_stop"
          ]
        }
      }, 
      "filter": {
        "english_stop":{
          "type":"stop",
          "stopwords":"_english_"
        }
      }
    }
  },
  "mappings":{
    "_doc":{
      "properties": {
        "title": {
          "type":"text",
          "analyzer":"movie_stop_test_analyzer",
          "search_analyzer":"movie_lower_test_analyzer"
        }
      }
    }
  }
}
```

### 대표적인 분석기

**StandardAnalyzer**

인덱스를 생성할 떄 settings에 analyzer를 정의하게 된다. 하지만 아무런 정의를 하지 않고 필드의 데이터 타입을 Text 데이터 타입으로 사용한다면 기본적으로 Standard Analyzer를 사용한다. 이 분석기는 공백 혹은 특수 기호를 기준으로 토큰을 분리하고 모든 문자를 소문자로 변경하는 토큰 필터를 사용한다.

*Standard Analyzer 구성요소*
|Tokenizer|TokenFilter|
|---|---|
|StandardTokenizer|Standard TokenFilter, Lower Case Token Filter|


*Standard Analyzer 옵션*
|파라미터|설명|
|---|---|
|max_token_length|최대 토큰 길이를 초과하는 토큰이 보일 경우 해당 length 간격으로 분할한다. 기본값은 255자|
|stopwords|사전 정의된 불용어 사전을 사용한다. 기본값은 사용하지 않는다.|
|stopwords_path|불용어가 포함된 파일을 사용할 경우의 서버의 경로로 사용한다.|


**WhiteSpaceAnalyzer**

공백 문자열을 기준으로 토큰을 분리하는 간단한 분석기이다.
|Tokenizer|Token Filter|
|---|---|
|Whitespace Tokenizer|없음|

**Keyword 분석기**

전체 입력 문자열을 하나의 키워드처럼 분리한다. 

### 전처리 필터
엘라스틱서치에서 제공하는 분석기는 전처리필터(Character Filter)를 이용한 데이터 정제 후 토크나이저를 이용해 본격적인 토큰 분리 작업을 수행한다. 그런 다음, 생성된 토큰 리스트를 토큰 필터를 통해 재가공하는 3단계 방식으로 동작한다. 

**HTML strip char 필터**

문장에서 HTML을 제거하는 전처리 필터다.

|파라미터|설명|
|---|---|
|escaped_tags|특정 태그만 삭제한다. 기본값으로 HTML 태그를 전부 삭제한다.|

```
PUT movie_html_analyzer 
{
  "settings": {
    "analysis": {
      "analyzer": {
        "html_strip_analyzer": {
          "tokenizer":"keyword",
          "char_filter": [
            "html_strip_char_filter"
          ]
        }
      },
      "char_filter": {
        "html_strip_char_filter": {
          "type": "html_strip",
          "escaped_tags": [
            "b"
          ]
        }
      }
    }
  }
}
```

*테스트*
```
POST movie_html_analyzer/_analyze
{
  "analyzer": "html_strip_analyzer",
  "text": "<span>harry poter</span> and the <b>chamber</b> of Secrets"
}
```

### 토크나이저 필터

토크나이저 필터는 분석기를 구성하는 가장 핵심 구성요소다. 전처리 필터를 거쳐 토크나이저 필터로 문서가 넘어오면 해당 텍스트는 Tokenizer의 특성에 맞게 적절히 분해된다. 분석기에서 어떠한 토크나이저를 사용하느냐에 따라 분석기의 전체적인 성격이 결정된다. 

분석기를 구성하는 데 가장 중요한 구성요소인 만큼 엘라스틱서치에서도 다양한 특성의 토크나이저를 제공한다. 

**Standard 토크나이저**

엘라스틱서치에서 일반적으로 사용하는 토크나이저로 대부분의 기호를 만나면 토큰으로 나눈다. 
|파라미터|설명|
|max_token_length|최대 토큰 길이를 초과하는 경우 해당 간격으로 토큰을 분할한다. 기본값은 255|

```
POST movie_analyzer/_analyze
{
  "tokenizer": "standard",
  "text": "Harry Potter and the Chamber of Secrets"
}
```

**Whitespace 토크나이저**

공백을 만나면 토큰화한다.

```
POST movie_analyzer/_analyze
{
  "tokenizer": "whitespace",
  "text": "Harry Potter and the Chamber of Secrets"
}
```

**Ngram 토크나이저**

Ngram은 기본적으로 한 글자씩 토큰화한다. Ngram에 특정 문자를 지정할 수도 있으며, 이 경우 지정된 문자의 목록 중 하나를 만날 때마다 단어를 자른다. 그 밖에도 다양한 옵션을 조합해서 자동완성을 만들때 유용하게 활용할 수 있다.

*Ngram 토크나이저 옵션*
|파라미터|설명|
|---|---|
|min_gram|Ngram을 적용할 문자의 최소 길이를 나타낸다. 기본값은 1|
|max_gram|Ngram을 적용할 문자의 최대 길이를 나타낸다. 기본값은 2|
|token_chars|토큰에 포함할 문자열을 지정한다. 다음과 같은 다양한 옵션을 제공한다.</br>letter(문자)</br>digit(숫자)</br>whitespace(공백)</br>punctuation(구두점)</br>symbol(특수기호)|

Ngram 토크나이저를 테스트하기 위한 인덱스 생성
```
PUT movie_ngram_analyzer
{
  "settings": {
    "analysis": {
      "analyzer": {
        "ngram_analyzer": {
          "tokenizer": "ngram_tokenizer"
        }
      },
      "tokenizer": {
        "ngram_tokenizer": {
          "type": "ngram",
          "min_gram": 3,
          "max_gram": 3,
          "token_chars" : [
            "letter"
          ]
        }
      }
    }
  }
}
```

**Edge Ngram 토크나이저**
지정된 문자의 목록 중 하나를 만날 때마다 시작 부분을 고정시켜 단어를 자르는 방식으로 사용하는 토크나이저다. 해당 Tokenizer 역시 자동 완성을 구현할 때 유용하게 활용할 수 있다. 

*Edge Ngram 토크나이저 옵션*
|파라미터|설명|
|---|---|
|min_gram|Ngram을 적용할 문자의 최소 길이를 나타낸다. 기본값은 1|
|max_gram|Ngram을 적용할 문자의 최대 길이를 나타낸다. 기본값은 2|
|token_chars|토큰에 포함할 문자열을 지정한다. 다음과 같은 다양한 옵션을 제공한다.</br>letter(문자)</br>digit(숫자)</br>whitespace(공백)</br>punctuation(구두점)</br>symbol(특수기호)|

```
PUT movie_engram_analyzer
{
  "settings": {
    "analysis": {
      "analyzer": {
        "edge_ngram_analyzer": {
          "tokenizer": "edge_ngram_tokenizer"
        }
      },
      "tokenizer": {
        "edge_ngram_tokenizer": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 10,
          "token_chars": [
            "letter"
          ]
        }
      }
    }
  }
}
```

### 토큰 필터
토큰 필터는 토크나이저에서 분리된 토큰들을 변형하거나 추가, 삭제할 때 사용하는 필터다. 토크나이저에 의해 토큰이 모두 분리되면 분리된 토큰은 배열 형태로 토큰 필터로 전달된다. 토크나이저에 의해 토큰이 모두 분리돼야 비로소 동작하기 때문에 독립적으로는 사용할 수 없다.

**Ascii Folding 토큰필터**

아스키 코드엥 해당하는 127개의 알파벳 숫자, 기호에 해당하지 않는 경우 문자를 ASCII 요소로 변경한다. 

ex)
```
PUT movie_af_analyzer
{
  "settings": {
    "analysis": {
      "analyzer": {
        "asciifolding_analyzer": {
          "tokenizer": "standard",
          "filter": [
            "standard",
            "asciifolding"
          ]
        }
      }
    }
  }
}
```

**Lowercase 토큰 필터**

이 토큰 필터는 토큰을 구성하는 전체 문자열을 소문자로 변환한다. 
Lowercase 토큰 필터를 테스트하기 위해 다음과 같이 movie_lower_analyzer라는 인덱스를 하나 생성한다.
```
PUT movie_lower_analyzer
{
  "settings": {
    "analysis": {
      "analyzer": {
        "lowercase_analyzer": {
          "tokenizer": "standard",
          "filter": [
            "lowercase"
          ]
        }
      }
    }
  }
}
```

**Uppercase 토큰 필터**

이 토큰 필터는 토큰을 구성하는 전체 문자열을 대문자로 변환한다. 
Uppercase 토큰 필터를 테스트하기 위해 다음과 같이 movie_upper_analyzer라는 인덱스를 하나 생성한다.
```
PUT movie_upper_analyzer
{
  "settings": {
    "analysis": {
      "analyzer": {
        "uppercase_analyzer": {
          "tokenizer": "standard",
          "filter": [
            "uppercase"
          ]
        }
      }
    }
  }
}
```

**Stop토큰 필터**

불용어로 등록할 사전을 구축해서 사용하는 필터를 의미한다. 인덱스로 만들고 싶지 않거나 검색되지 않게 하고 싶은 단어를 등록해서 해당 단어에 대한 불용어 사전을 구축한다.

*Stop 토큰 필터 옵션*
|파라미터|설명|
|---|---|
|stopwords|불용어를 매핑에 직접 등록해서 사용한다.|
|stopwords_path|불용어 사전이 존재하는 경로를 지정한다. 해당 경로는 ES 서버가 있는 config폴더 안에 생성한다.|
|ignore_case|true로 지정할 경우 모든 단어를 소문자로 변경해서 저장한다. 기본값은 false|

```
PUT movie_stop_analyzer
{
  "settings": {
    "analysis": {
      "analyzer": {
        "stop_filter_analyzer": {
          "tokenizer": "standard",
          "filter": [
            "standard",
            "stop_filter"
          ]
        }
      },
      "filter": {
        "stop_filter": {
          "type": "stop",
          "stopwords": [
            "and",
            "is",
            "the"
          ]
        }
      }
    }
  }
}
```

**Stemmer 토큰 필터**

Stemming 알고리즘을 사용해 토큰을변형하는 필터다. 

*Stemmer 토큰 필터 옵션*
|파라미터|설명|
|---|---|
|name|english, light_english, minimal_english, possessive_english, porter2, lovins등 다른 나라의 언어도 사용가능하지만, 한글은 없다고 한다.|

```
PUT movie_stem_analyzer
{
  "settings": {
    "analysis": {
      "analyzer": {
        "stemmer_eng_analyzer": {
          "tokenizer": "standard",
          "filter": [
            "standard",
            "lowercase",
            "stemmer_eng_filter"
          ]
        }
      },
      "filter": {
        "stemmer_eng_filter": {
          "type": "stemmer",
          "name": "english"
        }
      }
    }
  }
}
```

**Synonym 토큰 필터**

동의어를 처리할 수 있는 필터다. 

*Synonym 토큰 필터 옵션*
|파라미터|설명|
|---|---|
|synonyms|동의어로 사용할 단어를 등록한다.|
|synonyms_path|파일로 관리할 경우 엘라스틱서치 서버의 config 폴더 아래에 생성한다.|

```
PUT movie_syno_analyzer
{
  "settings": {
    "analysis": {
      "analyzer": {
        "synonym_analyzer": {
          "tokenizer": "whitespace",
          "filter": [
            "synonym_filter"
          ]
        }
      },
      "filter": {
        "synonym_filter": {
          "type": "synonym",
          "synonyms": [
            "Harry => 해리"
          ]
        }
      }
    }
  }
}
```

**Trim 토큰 필터**

앞 뒤 공백을 제거하는 토큰 필터다.
```
PUT movie_trim_analyzer
{
  "settings": {
    "analysis": {
      "analyzer": {
         "trim_analyzer": {
           "tokenizer": "keyword",
           "filter": [
             "lowercase",
             "trim"
          ]
        }
      }
    }
  }
}
```

### 동의어 사전

토크나이저에 의해 토큰이 모두 분리되면 다양한 토큰 필터를 적용해 토큰을 가공할 수 있다. 토큰 필터를 이용하면 토큰을 변경하는 것은 물론이고 토큰을 추가하거나 삭제하는 것도 가능해진다. 엘라스틱 서치에서 제공하는 토큰 필터 중 Synonym 필터를 사용하면 동의어 처리가 가능해진다. 

동의어는 검색 기능을 풍부하게 할 수 있게 도와주는 도구 중 하나다. 원문에 특정 단어가 존재하지 않떠라도 색인 데이터를 토큰화해서 저장할 때 동의어나 유의어에 해당하는 단어를 함께 저장해서 검색이 가능해지게 하는 기술이다. 예를 들어, "ElasticSearch"라는 단어가 포함된 원문이 필터를 통해 인덱스에 저장된다면 "엘라스틱서치"라고 검색했을 때 검색되지 않을 것이다. 하지만 동의어 기능을 이용해 색인 할 때 "엘라스틱서치"도 함께 저장한다면 "Elasticsearch"로도 검색이 가능하고 "엘라스틱서치"로도 검색이 가능해질 것이다.

동의어를 추가하는 방식에는 크게 두 가지 방식이 있다.
- 동의어를 매핑 설정 정보에 미리 파라미터로 등록하는 방식
- 특정 파일을 별도로 생성해서 관리하는 방식

엘라스틱서치에서 가장 까다로운 부분 중 하나가 바로 동의어를 관리하는 것이다. 검색엔진에서 다루는 분야가 많아지면 많아질수록 동의어의 수도 늘어난다. 분야별로 파일도 늘어날 것이고 그 안의 동의어 변환 규칙도 많아질 것이다. 실무에서는 이러한 동의어를 모아둔 파일들을 통칭하여 "동의어 파일" 이라고 한다. 

**동의어 사전 만들기**

```
<엘라스틱서치 설치 디렉터리>/config/analysis/synonym.txt
```
이곳에는 다음의 두 가지 방법으로 데이터를 추가한다.
- 동의어 추가
- 동의어 치환


**동의어 추가**

동의어를 추가할 때 단어를 쉼표(,)로 분리해 등록하는 방법이다. 예를 들어, "Elasticsearch"와 "엘라스틱서치"를 동의어로 지정하고 싶다면 동의어 사전 파일에 "Elasticsearch,엘라스틱서치"라고 등록하면 된다.

여기서 주의할 부분은 동의어 처리 기준은 앞서 동작한 토큰 필터의 종류가 무엇이고 어떤 작업을 했느냐에 따라 달라질 수 있다. 예를 들어, "Elasticsearch"라는 토큰은 "elasticsearch"라는 토큰으로 변경될 것이다. 이 경우 동의어로 등록한 "Elasticsearch"와 일치하지 않기 때문에 다른 토큰으로 인식해서 동의어가 적용되지 않을 것이다. 

**동의어 치환하기**

특정 단어를 어떤 단어로 변경하고 싶다면 동의어 치환 기능을 이용하면 된다. 동의어 치환은 동의어 추가와 구분하기 위해 화살표(=>)로 표신한다.

*ES설치폴더/config/analyze/synonym.txt*
```
Elasticsearch, 앨라스틱서치
Harry=>해리
```

```
PUT movie_analyzer2
{
  "settings": {
    "index": {
      "analysis": {
        "analyzer": {
          "synonym_analyzer": {
            "tokenizer": "standard",
            "filter": [
              "lowercase",
              "synonym_filter"
            ]
          }
        },
        "filter": {
          "synonym_filter": {
            "type": "synonym",
            "ignore_case": "true",
            "synonyms_path": "analysis/synonym.txt"
          }
        }
      }
    }
  }
}
```
*테스트*
```
POST movie_analyzer2/_analyze
{
  "analyzer": "synonym_analyzer",
  "text": "Elasticsearch Harry Poter"
}
```

*인덱스 reload하기*
```
POST movie_analyzer2/_close
POST movie_analyzer2/_open
```
