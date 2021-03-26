# 데이터 검색

엘라스틱서치는 인덱스에 저장된 문서를 검색할 수 있도록 다양한 검색 기능을 제공한다. 문서는 색인시 설정한 분석기에 의해 분석 과정을 거쳐 토큰으로 분리되는데, 이러한 분석기는 색인 시점에 사용할수도 있지만 검색 시점에 사용하는 것도 가능하다. 특정 문장이 검색어로 요청되면 분석기를 통해 분석된 토큰의 일치 여부를 판단해서 그 결과에 점수(Score)를 매긴다. 그리고 이를 기반으로 순서를 적용해 결과를 사용자에게 최종적으로 출력하게 된다. 

검색의 대상이 되는 필드는 분석이 되는 Text타입의 유형이 도리 수도 있고 분석이 되지 않는 Keyword 타입의 유형이 될 수도 있다. 엘라스틱서치에서는 다양한 검색 조건을 충족시키기 위해 Query DSL이라는 특수한 쿼리 문법을 제공한다. 이를 이용하면 엘라스틱서치가 제공하는 다양한 검색 기능을 활용할 수 있게 된다.

--- 
## 검색 API

문장은 색인시점에 텀으로 분해된다. 검색 시에는 이텀을 일치시켜야 검색이 가능해진다. 그렇다면 색인 시점에 생성한 텀을 어떻게 검색 시점에 찾을 수 있을까, 라는 의문이 생길 것이다. 

엘라스틱서치는 색인 시점에 Analyzer를 통해 분석된 텀을 Term, 출현빈도, 문서번호와 같이 역색인 구조로 만들어 내부적으로 젖아한다. 검색 시점에는 Keyword 타입과 같은 분석이 불가능한 데이터와 Text타입과 같은 분석이 가능한 데이터를 구분해서 분석이 가능할 경우 분석기를 이용해 분석을 수행한다. 이를 통해 검색 시점에도 텀을 얻을 수 있으며, 해당 텀으로 역색인 구조를 이용해 문서를 찾고 이를 통해 스코어를 계산해서 결과로 제공한다. 이러한 동작 방식을 이해하고 이에 맞춰 목적에 맞게 검색 쿼리를 사용해야 한다.

```
curl -XGET 'http://localhost:9200/_snapshot/javacafe/_all?pretty'
```

*스냅숏 복구하기*
```
curl -XPOST 'http://localhost:9200/_snapshot/javacafe/movie-search/_restore'
```

### 검색 질의 표현 방식

엘라스틱서치에서 제공하는 검색 API는 기본적으로 질의(Query)를 기반으로 동작한다. 검색 질의에는 검색하고자 하는 각종 조건들을 명시할 수 있으며 동일한 조건을 다음과 같은 두 가지 방식으로 표현할 수 있다.

- API 검색
- Request Body 검색 

**URI 검색**

URI를 이용하는 방식은 HTTPGET 요청을 활용하는 방식으로 파라미터를 'Key=Value' 형태로 전달하는 방식이다. URL에 검색할 칼럼과 검색어를 직접 지정하면 검색이 수행된다. 파라미터로 표현할 수 있는 표현의 한계로 복잡한 질의를 작성하는 것은 불가능하다.

```
GET movie_search/_search?q=prdtYear:2018
```

**Request Body 검색**

Request Body 방식은 HTTP 요청 시 Body에 검색할 칼럼과 검색어를 JSON 형태로 표현해서 전달하는 방식이다. JSON 형태의 표현을 좀 더 효율적으로 하기 위해 엘라스틱서치에서는 Query DSL 이라는 특별한 문법을 지원한다. 이를 이용하면 URI 방식에 비해 다양한 조합의 검색 쿼리를 작성할 수 있다. 엘라스틱서치가 제공하는 검색 API를 모두 활용하기 위해서는 Request Body형식을 이용해야 한다.

```
POST movie_search/_search
{
    "query": {
        "term" : {"prdtYear": "2018"}
    }
}
```

### URI 검색

URI검색은 Request Body검색에 비해 단순하고 사용하기 편하지만 복잡한 질의문을 입력하기 힘들다는 치명적인 단점이 있다. 또한 URI 검색을 이용할 경우에는 엘라스틱서치에서 제공하는 모든 검색 옵션을 사용할 수 없다. 파라미터를 전달하는 표현법으로는 표현에 상당한 제약이 있기 떄문이다. 

```
POST movie_search/_search?q=movieNmEn:Family
```

*URI 검색에서 자주 사용되는 파라미터*
|파라미터|기본값|설명|
|---|---|---|
|q|-|검색을 수행할 쿼리 문자열 조건을 지정한다|
|df|-|쿼리에서 검색을 수행할 필드가 지정되지 않았을 경우 기본값으로 검색할 필드를 지정한다.|
|analyzer|검색 대상 필드에 설정된 형태소 분석기|쿼리 문자열을 형태소 분석할 때 사용할 형태소 분석기를 지정한다.|
|analyze_wildcard|false|접두어/와일드카드(*) 검색 활성화 여부를 지정한다.|
|default_operator|OR|두 개 이상의 검색 조건이 쿼리 문자열에 포함된 경우 검색 조건 연산자를 설정한다.|
|_source|true|검색 결과에 문서 본문 포함 여부를 지정한다.|
|sort|-|검색 결과의 정렬 기준 필드를 지정한다.|
|from|-|검색을 시작한 문서의 위치를 설정한다.|
|size|-|반환할 검색 결과 개수를 설정한다.|

q옵션에는 기본적으로 '[필드명]:검색어' 형태로 입력할 수 있으며, 예제와 같이 여러 개의 필드를 검색할 떄는 공백을 입력한 후 추가적인 필드명과 검색어를 입력한다. URI검색에 q 옵션에 사용되는 검색 문자열은 사실 Reuqest Body 검색에서 제공하는 Query String Search 옵션과 동일하게 동작한다. 상세한 방법은 추후 Query DSL절에서 본다.

```
POST movie_search/_search?q=movieNmEn:* AND prdtYear:2017&analyze_wildcard=true&from=0&size=5&sort=_score:desc,movieCd:asc&_source_includes=movieCd,movieNm,movieNmEn,typeNm
```
하지만,, 위의 쿼리는 너무 복잡하여 가독성이 떨어진다.

### Request Body 검색

엘라스틱서치에서 제공하는 두 번쨰 검색 방식은 Request Body 검색이다. 이 방식에서는 HTTP 요청시 본문에 JSON  형태로 검색 조건을 기록해서 검색을 요청한다. 앞서 살펴본 간단한 URI 검색과 동일한 예제를 Request Body 검색으로 변경해보자.

```
POST movie_search/_search
{
    "query": {
        "query_string": {
            "default_field": "movieNmEn",
            "query": "Family"
        }
    }
}
```

```
POST movie_search/_search 
{
    "query": {
        "query_string": {
            "default_field": "movieNmEn",
            "query": "movieNmEn:* OR prdtYear:2017"
        }
    },
    "from": 0,
    "size": 5,
    "sort": [
        {
            "_score": {
                "order": "desc"
            },
            "movieCd": {
                "order": "asc"
            }
        }
    ],
    "_source": [
        "movieCd",
        "movieNm",
        "movieNmEn",
        "typeNm"
    ]
}
```
JSON 구조를 파악하고 살펴보면 상대적으로 가독성이 좋아진 것을 느낄 수 있다. 즉, Query DSL을 사용하면 복잡한 검색 옵션도 깔끔한 JSON 구조로 표현하는 것이 가능하다.

## Query DSL 이해하기

엘라스틱서치로 검색 질의를 요청할 떄는 Request Body 검색과 URI 검색 모두 _search API를 이용해 검색을 질의한다. 하지만 Query DSL을 이용하면 여러 개의 질의를 조합하거나 질의 결과에 대해 다시 검색을 수행하는 등 기존의 URI 검색보다 강력한 검색이 가능해진다. 

엘라스틱서치에서는 정밀한 검색을 위해 JSON 구조를 기반으로 한 Query DSL을 제공한다. Query DSL은 엘라스틱서치의 가장 강력한 기능 중 하나로서 Request Body 검색을 이용할 떄 사용하는JSON 구조를 지원한다.

### Query DSL 쿼리의 구조 

```
{
    "size":  ---- 1)
    "from":  ---- 2)
    "timeout":  ---- 3)

    "_source": {}  ---- 4)
    "query": {}  ---- 5)
    "aggs": {}  ---- 6)
    "sort": {}  ---- 7)
}
```
1) 리턴받은 결과의 개수를 지정한다. 기본값은 10이다.
2) 몇 번쨰 문서부터 가져올지 지정한다. 기본값은 0이며, 페이지별로 구성하려면 다음 번 문서는 10번째부터 가지고 오면 된다.
3) 검색을 요청해서 결과를 받는 데 까지 걸리는 시간을 나타낸다. timeout 시간을 너무 짧게 잡으면 전체 샤드에서  timeout을 넘기지 않은 문서만 결과로 출력되기 때문에 상황에 따라 결과의 일부만 나올 수 있다. 기본값은 무한대다.
4) 검색 시 필요한 필드만 출력하고 싶을 때 사용한다.
5) 검색 조건문이 들어가야 하는 공간이다. 
6) 통계 및 집계 데이터를 사용할 때 사용하는 공간이다.
7) 문서 결과를 어떻게 출력할지에 대한 조건을 사용하는 공간이다. 


엘라스틱 서치로 쿼리가 요청되면 해당 쿼리를 파싱해서 문법에 맞는 요청인지 검사한다. 파싱에 성공하면 해당 쿼리를 기반으로 검색을 수행하고, 결과를 JSON 형식으로 제공한다. 응답 JSON 구조는 아래와 같다.

```
{
    "took":  --- 1)

    "timed_out":  --- 2)
    "_shards": {
        "total":  --- 3)
        "successful":  --- 4)
        "failed":  --- 5)
    },

    "hits":{
        "total":  --- 6)
        "max_score":  --- 7)
        "hits": []  --- 8)
    }
}
```
1) 쿼리가 실행한 시간을 나타낸다.
2) 쿼리 시간이 초과할 경우를 나타낸다.
3) 쿼리를 요청한 전체 샤드의 개수를 나타낸다.
4) 검색 요청에 성공적으로 응답한 샤드의 개수를 나타낸다.
5) 검색 요청에 실패한 샤드의 개수를 나타낸다.
6) 검색어에 매칭된 문서의 전체 개수를 나타낸다.
7) 일치하는 문서의 스코어 값 중 가장 높은 값을 출력한다.
8) 각 문서 정보와 스코어 값을 보여준다.
   

### Query DSL 쿼리와 필터

Query DSL을 이용해 검색 질의를 작성할 떄 조금만 조건이 복잡해지더라도 여러 개의 작은 질의를 조합해서 사용해야 한다. 작성되는 작은 질의들을 두 가지 형태로 나눠서 생각해볼 수 있다. 실제 분석기에 의한 전문 분석이 필요한 경우와 단순히 'Yes/No'로 판단할 수 있는 조건 검색의 경우다. 엘라스틱서치에서는 전자를 쿼리(Queries) 컨텍스트라고 하고, 후자를 필터(Filter) 컨텍스트라는 요어로 구분하고 있다.

*쿼리 컨텍스트와 필터 컨텍스트의 차이점*
||쿼리 컨텍스트|필터 컨텍스트|
|---|---|---|
|용도|전문 검색시 사용 조건 검색시 사용(예:Yes/No)
|특징|분석기에 의해 분석이 수행됨.</br>연관성 관련 Score를 계산</br>루씬 레벨에서 분석 과정을 거처ㅕ야 하므로 상대적으로 느림|Yes/No로 단순 판별 가능</br>연관성 관련 계산을 하지 않음</br>엘라스틱서치 레벨에서 처리가 가능하므로 상대적으로 빠름|
|사용 예|"Harry Potter" 같은 문장 분석|"create_year"필드의 값이 2018년인지 여부</br> "status"필드에 'use'라는 코드 포함 여부|

**쿼리 컨텍스트**
- 문서가 쿼리와 얼마나 유사한지를 스코어로 계산한다.
- 질의가 요청될 때마다 엘라스틱서치에서 내부의 루씬을 이용해 계산을 수행한다.(이때 결과가 캐싱되지 않는다.)
- 일반적으로 전문 검색에 많이 사용한다.
- 캐싱되지 않고 디스크 연산을 수행하기 때문에 상대적으로 느리다. 


```
POST movie_search/_search
{
    "query": {
        "match": {
            "movieNm": "기묘한 가족"
        }
    }
}
```

**필터 컨텍스트**
- 쿼리의 조건과 문서가 일치하는지(Yes/No)를 구분한다.
- 별도로 스코어를 계산하지 않고 단순 매칭 여부를 검사한다.
- 자주 사용되는 필터의 결과는 엘라스틱서치가 내부적으로 캐싱한다.
- 기본적으로 메모리 연산을 수행하기 때문에 상대적으로 빠르다. 

```
POST movie_search/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match_all": {}
        }
      ],
      "filter": {
        "term": {
          "repGenreNm": "다큐멘터리"
        }
      }
    }
  }
}
```


**_source필드 필터링**

```
POST movie_search/_search
{
    "_source": [
        "movieNm"
    ],
    "query": {
        "term": {
            "repNationNm": "한국"
        }
    }
}
```

**범위 검색**

*범위 연산자*
|문법|연산자|설명|
|---|---|---|
|lt|<|피연산자보다 작음|
|lt|>|피연산자보다 작음|
|lt|<=|피연산자보다 작음|
|lt|>=|피연산자보다 작음|

```
POST movie_search/_search
{
  "query": {
    "range": {
      "prdtYear": {
        "gte": "2016",
        "lte": "2017"
      }
    }
  }
}
```

**operator 설정**

Query DSL에서는 operator 파라미터를 통해 연산자를 명시적으로 지정하는 것이 가능하다. 이를 이용해 명시적으로 "and"나 "or"연산자를 명시적으로 지정할 수 있다. operator 파라미터가 생략된 경우에는 기본적으로 텀과 텀에 OR 연산을 적용한다. 

```
POST movie_search/_search
{
    "query": {
        "match": {
            "movieNm": {
                "query": "자전차왕 엄복동"
                "operator": "and"
            }
        }
    }
}
```

**minimum_sholud_match 설정**

일반적으로 OR 연산을 수행할 경우 검색 결과가 너무 많아질 수 있다. 이 경우 텀의 개수가 몇 개 이상 매칭될 때만 검색 결과로 나오게 할 수 있는데 이때 사용하는 파라미터가 minimum_should_match다.

OR 연산으로 검색할 떄 minimum_should_match 파라미터를 사용하면 AND 연산자가 아닌 OR 연산자로도 AND 연산과 비슷한 효과를 낼 수 있다. 이 파라미터를 이용하면 문서의 결과에 포함할 텀의 최소 개수를 지정할 수 있따. 

```
POST movie_search/_search
{
    "query": {
        "match": {
            "movieNm": {
                "query": "자전차왕 엄복동",
                "minimum_should_match": 2
            }
        }
    }
}
```

**fuzziness 설정**

fuzziness 파라미터를 사용하면 단순히 같은 값을 찾는 Match Query를 유사한 값을 찾는 Fuzzy Query로 변경할 수 있다. 이는 레벤슈타인 편집 거리 알고리즘을 기반으로 문서의 필드 값을 여러 번 변경하는 방식으로 동작한다. 유사한 검색 결과를 찾기 위해 허용 범위의 텀으로 변경해 가며 문서를 찾아 결과로 출력한다. 

예를 들어, 편집 거리의 수를 2로 설정한다면 오차 범위가 두 글자 이하인 검색 결과까지 포함해서 결과로 출력한다. 오차범위 값으로 0, 1, 2, AUTO로 총 4가지 값을 사용할 수 있는데, 이는 알파벳에는 유용하지만 한국어에는 적용하기 어렵다. 하지만 영어를 많이 사용하는 국내 상황에서는 여러 가지 적용가능한 곳이 있기에 알아두면 유용할 것이다. 

```
POST movie_search/_search
{
    "query": {
        "match": {
            "movieNmEn": {
                "query": "Fli High",
                "fuzziness": 1
            }
        }
    }
}
```

**boost 설정**

이 파라미터는 관련성이 높은 필드나 키워드에 가중치를 더 줄 수 있게 해준다. 영화 데이터의 경우 한글 영화 제목과 영문 영화 제목으로 두 가지 제목 필드를 제공하고 있다. 이때 한글 영화 제목에 좀 더 가중치를 부여해서 검색 결과를 좀 더 상위로 올리고 싶을 때 사용할 수 있다. 

```
POST movie_search/_search
{
    "query": {
        "multi_match": {
            "query": "Fly",
            "fields": ["movieNm^3", "movieNmEn"]
        }
    }
}
```

## Query DSL의 주요 쿼리

### Match All Query
mach_all 파라미터를 사용하는 Match All Query는 색인에 모든 문서를 검색하는 쿼리다. 가장 단순한 쿼리로서 일반적으로 색인에 저장된 문서를 확인할 때 사용된다. 

```
POST movie_search/_search
{
    "query": {
        "match_all": {}
    }
}
```

### Match Query
텍스트, 숫자, 날짜 등이 포함된 문장을 형태소 분석을 통해 텀으로 분리한 후 이텀들을 이용해 검색 질의를 수행한다. 그러므로 검색어가 분석돼야 할 경우에 사용해야 한다. 

```
POST movie_search/_search
{
    "query": {
        "match": {
            "movieNm": "그대 장미"
        }
    }
}
```

"그대 장미" 를 "그대", "장미"로 2개의 텀으로 분리한다. 

### Multi Match Query
Match Query와 기본적인 사용 방법은 동일하나 단일 필드가 아닌 여러 개의 필드를 대상으로 검색해야 할 떄 사용하는 쿼리다.

```
POST movie_search/_search
{
    "query": {
        "multi_match": {
            "query": "가족",
            "fields": ["movieNm", "movieNmEn"]
        }
    }
}
```

### Term Query
텍스트 형태의 검색을 하기 위해 엘라스틱 서치는 두 가지 매핑 유형을 지원한다.

*문자형 데이터 타입*
|타입|설명|
|---|---|
|Text 데이터 타입| 필드에 데이터가 저장되기 전에 데이터가 분석되어 역색인 구조로 저장된다.|
|Keyword 데이터 타입| 데이터가 분석되지 않고 그대로 필드에 저장된다.|

Term Query는 별도의 분석 작업을 수행하지 않고 입력된 텍스트가 존재하는 문서를 찾는다. 따라서 Keyword 데이터 타입을 사용하는 필드를 검색하려면 Term Query를 사용해야 한다.

Term Query는 Keyword 데이터 타입을 대상으로 하기 때문에 일반적으로 숫자, Keyword, 날짜 데이터를 쿼리하는 데 사용한다.

```
POST movie_search/_search
{
    "query": {
        "term": {
            "genreAlt": "코미디"
        }
    }
}
```


### Bool Query
관계형 DB에서는 AND, OR 문으로 묶은 여러 조건을 Where절에서 사용할 수 있다. 이처럼 엘라스틱서치에서도 여러 개의 쿼리를 조합해서 사용하고 싶을때는 어떻게 해야할까?

엘라스틱서치에서는 하나의 쿼리나 여러 개의 쿼리를 조합해서 더 높은 스코어를 가진 쿼리 조건으로 검색을 수 행할 수 있다. 이러한 유형의 쿼리를 Compound Query 라 하는데, 이러한 Compound Query를 구현하기 위해 엘라스틱서치에서는 Bool Query를 제공한다. Bool Query를 상위에 두고 하위에 다른 Query들을 사용해 복잡한 조건의 쿼리문을 작성할 수 있다. 

Bool Query는 주어진 쿼리와 논리적으로 일치하는 문서를 복합적으로 검색한다. 해당 Query를 사용해 여러 가지 형태(AND/OR/NAND/FILTER)를 표현할 수 있다. 기본적으로 다음과 같은 구조로 Bool Query를 표현할 수 있다.

```
{
    "query": {
        "bool": {
            "must": [],
            "must_not": [],
            "should": [],
            "filter": []
        }
    }
}
```

*SQL 구문의 조건절 비교*
|Elasticsearch|SQL|설명|
|---|---|---|
|must:[필드]|AND 칼럼 = 조건| 반드시 조건에 만족하는 문서만 검색된다.|
|must_not:[필드]|AND 칼럼 != 조건|조건을 마나족하지 않는 문서가 검색된다.|
|should:[필드]|OR 칼럼 = 조건| 여러 조건 중 하나 이상을 만족하는 문서가 검색된다.|
|filter:[필드]|칼럼 IN (조건)|조건을 포함하고 있는 문서를 출력한다. 해당 파라미터를 사용하면 스코어별로 정렬되지는 않는다.|

```
POST movie_search/_search
{
  "query":{
    "bool": {
      "must": [
        {
          "term": {
            "repGenreNm": "코미디"
          }
        },
        {
          "match": {
            "repNationNm": "미국"
          }
        }
      ],
      "must_not": [
        {
          "match": {
            "typeNm": "단편"
          }
        }
      ]
    }
  }
}

```

### Query String
엘라스틱서치에는 기본적으로 내장된 쿼리 분석기가 있다. query_string 파라미터를 사용하는 쿼리를 작성하면 내장된 쿼리 분석기를 이용하는 질의를 작성할 수 있다. 

```
POST movie_search/_search
{
    "query": {
        "query_string": {
            "default_field": "movieNm",
            "query": "(가정) AND (어린이 날)"
        }
    }
}
```

### Prefix Query
접두어가 있는 몯느 문서를 검색
```
POST movie_search/_search
{
    "query": {
        "prefix": {
            "movieNm": "자전차"
        }
    }
}
```

### Exists Query
문서를 색인할 때 필드의 값이 없다면 필드를 생성하지 않거나 필드의 값을 null로 설정할 때가 있다. 이러한 데이터를 제외하고 실제 값이 존재하는 문서만 찾을때 사용하는 쿼리

```
POST movie_search/_search
{
    "query": {
        "exists": {
            "field": "movieNm"
        }
    }
}
```

### Wildcard Query
검색어가 와일드 카드와 일치하는 구문을 찾는다. 이때 입력된 검색어는 형태소 분석이 이뤄지지 않는다.

*와일드카드 옵션*
|와일드카드 옵션|설명|
|---|---|
|*|문자의 길이와 상관없이 와일드카드와 일치하는 모든 문서를 찾는다.|
|?|지정된 위치의 한 글자가 다른 경우의 문서를 찾는다.|

```
POST movie_search/_search
{
    "query": {
        "wildcard": {
            "typeNm" : "장?"
        }
    }
}
```
### Nested Query

분산 시스템에서 SQL에서 지원하는 조인과 유사한 기능을 수행하려면 엄청나게 많은 비용이 소모된다. 수평적으로 샤드가 얼마나 늘어날지 모르는 상황에서 모든 샤드를 검색해야 할 수도 있기 떄문이다. 하짐나 업무를 수행하다 보면 문서 간의 부모/자식 관계의 형태로 모델링되는 경우가 종종 발생할 것이다. 이러한 경우에 대비해 엘라스틱서치에서는 분산 데이터 환경에서도 SQL 조인과 유사한 기능을 수행하는 Nested Query를 제공한다. 

Nested Query는 Nested 데이터 타입의 필드를 검색할 때 사용한다. Nested 데이터 타입은 문서 내부에 다른 문서가 존재할 때 사용한다. path 오볏ㄴ으로 중첩된 필드를 명시하고, query 옵션에 Nested 필드 검색에 사용할 쿼리를 입력한다. 

```
PUT movie_nested
{
    "settings": {
        "number_of_replicas": 1,
        "number_of_shards": 5
    },
    "mapping": {
        "_doc": {
            "properties": {
                "repGenreNm": {
                    "type": "keyword"
                },
                "companies": {
                    "type": "nested",
                    "properties": {
                        "companyCd": {"type": "keyword"},
                        "companyNm": {"type": "keyword"}
                    }
                }
            }
        }
    }
}
```

생성된 인덱스에 ㅇ문서를 하나 추가한다.
```
PUT movie_nested/_doc/1
{
  "movieCd": "20184623",
  "movieNm": "바람난 아내들",
  "movieNmEn": "",
  "prdtYear": "2018",
  "openDt":"",
  "typeNm": "장편",
  "prdtStatNm": "개봉예정",
  "nationAlt":"한국",
  "genreAlt":"멜로/로맨스",
  "repNationNm": "한국",
  "repGenreNm": "멜로/로맨스",
  "companies": [
    {
      "companyCd": "20173401",
      "companyNm":"(주)케이피에이기획"
    }
  ]
}
```

```
GET movie_nested/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "repGenreNm":"멜로/로맨스"
          }
        },
        {
          "nested": {
            "path": "companies",
            "query": {
              "bool": {
                "must": [
                  {
                    "term" : {
                      "companies.companyCd":"20173401"
                    }
                  }
                ]
              }
            }
          }
        }
      ]
    }
  }
}
```