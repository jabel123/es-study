# 데이터 집계

RDBMS의 group by 같은? 통계를 위한 기능을 제공하는 파트로 보인다.

- 집계
- 메트릭 집계
- 버킷 집계
- 파이프라인 집계
- 근삿값(Approximate)으로 제공되는 집계 연산

---
## 집계

데이터를 그룹화하고 통계를 구하는 기능이다.

### 엘라스틱서치와 데이터 분석

대용량 데이터를 하둡이나 RDBMS로 통계를 낼 때는 배치 방식을 사용한다. 반면 엘라스틱서치는  많은 양의 데이터를 조각내어 관리하는 덕분에 좀 더 실시간에 가깝게 문서를 처리할 수 있다.

집계를 이용해 범위, 날짜, 위치 정보도 집계할 수 있다. 또한 ES는 인덱스를 활용해 분산 처리가 가능하기 때문에 SQL보다 더 많은 데이터를 빠르게 처리할 수 있다. 

### ES가 집계에 사용하는 기술
집계분석에 있어서 검색보다 많은 리소스를 사용하는데, 이에 있어서 주의사항이 있다.

**캐시**

캐시는 질의의 결과를 임시 버퍼(캐시)에 둔다. 이후 처리해야 하는 같은 질의에 대해 매번 결과를 계산하는 게 아니라 버퍼에 보관된 결과를 반환한다.

캐시의 크기는 일반적으로 힙 메모리의 1%를 할당하며, 캐시에 없는 질의의 경우 성능 향상에 별다른 도움이 되지 않는다. 

elasticsearch.yml 파일을 수정하여 이 값을 조정할 수 있다.
```
indices.requests.cache.size: 2%
```

**Node query Cache**

노드의 모든 샤드가 공유하는 LRU(Least-Recently-Used) 캐시다. 캐시 용량이 가득차면 사용량이 가장 적은 데이터를 삭제하고 새로운 결과를 캐싱한다.
```
index.queries.cache.enabled:true
```

**Shard request Cache**

샤드는 데이터를 분산 저장하기 위한 단위로서, 그 자체가 온전한 기능을 가진 독립 인덱스라고 할 수 있다. Shard request Cache는 바로 이 샤드에서 수행된 쿼리의 결과를 캐싱한다. 

**Field data Cache**

엘라스틱서치가 필드에서 집계 연산을 수행할 때는 모든 필드 값을 메모리에 로드한다. 


### 실습데이터 살펴보기

apche-web-log에 대한 통계 내보기


```
GET apache-web-log/_search?size=0
{
  "query": {
    "match_all":{}
  },
  "aggs": {
    "region_count": {
      "terms": {
        "field": "geoip.region_name.keyword",
        "size": 20
      }
    }
  }
}
```


### Aggregation API 이해하기
검색 쿼리의 결과 집계는 다음과 같이 기존 검색 쿼리에 집계 구문을 추가하는 방식으로 수행한다.
```
{
    "query": {...생략...}
    "aggs": {...생략...}
}
```

ES는 집계시 문서를 평가한 후 기준에 맞는 문서들을 하나로 그룹화 한다. 그룹화한 집합을 토대로 집계를 수행하고, 집계가 끝나면 버킷 목록에 속한 문서의 집합이 출력된다.

- 버킷집계 : 쿼리 결과로 도출된 도큐먼트 집합에 대해 특정 기준으로 나눈 다음 나눠진 도큐먼트들에 대한 산술 연산을 수행한다. 이때 나눠진 도큐먼트들의 모음들이 각 버킷에 해당된다.
- 메트릭 집계: 쿼리 결과로 도출된 도큐먼트 집합에서 필드의 값을 더하거나 평균을 내는 등의 산술 연산을 수행한다.
- 파이프라인 집계: 다른 집계 또는 관련 메트릭 연산의 결과를 집계한다.
- 행렬 집계: 버킷 대상이 되는 도큐먼트의 여러 필드에서 추출한 값으로 행렬 연산ㅇ르 수행한다. 이를 토대로 다양한 통계 정보를 제공하고 있으나 아직은 공식적인 집계 여산으로 제공되지 않고 실험적인 기능으로 제공되기 때문에 주의를 요한다. 


**집계 구문의 구조**
```
"aggregations": {
    "<aggregation_name>": {
        "<aggregation_type>": {
            <aggregation_body>
        }
        [, "meta" : { [<meta_data_body>] } ]?
        [, "aggregations": : { [<sub_aggregation>]+ } ]?
    }
    [, "<aggregation_name_2>" : {...}]
}
```

데이터 집계를 위해 aggregations or aggs를 명시한다. aggregation_name에는 하위 집계의 이름을 기입한다. 이 이름은 집계 결과의 출력에 사용된다. 

aggregation_type에는 집계의 유형을 적는다. terms, data_histogram, sum과 같은 다양한 집계 유형을 지정할 수 있다. aggregation_body에는 앞서 지정한 aggregation_type에 맞춰 내용을 작성하면 된다. 

또한 meta필드를 사용하거나 aggregations를 중첩할 수 있는데, 중첩의 경우 같은 레벨에 또 다른 집계를 정의하는 것도 가능하다. 단, 같은 레벨에 집계를 정으히ㅏㄹ 때는 부수적인 성격의 집계만 정의할 수 있다. 

**집계 영역**

```
{
    "query": { -------------------------  1)
        "constant_score": {
            "filter": {
                "match": <필드조건>
            }
        }
    },
    "aggs": { -------------------------  2)
        "<집계 이름>": {
            "<집계 타입>": {
                "field": "<필드명>"
            }
        }
    }
}
```
1) query: 질의를 수행한다. 하위 필터 조건에 의해 명시한 필드와 값이 일치하는 문서만 반환한다.
2) aggs: 질의를 통해 반환받은 문서들의 집합 내에서 집계를 수행한다. 

만약 질의가 생략된다면 내부적으로 match_all 쿼리로 수행.
```
{
    "size": 0,  ---------- 1
    "aggs": { --------- 2
        "<집계 이름>" :{
            "<집계 타입>" :{
                "field": "<필드명>"
            }
        }
    }
}
```
1. size: 질의가 명시돼 있지 않기 때문에 내부적으로는 match_all이 수행되고 size가 0이기 때문에 결과 집합에 문서들 또한 존재하지 않는다. 즉, 문서의 결과는 출력되지 않는다.
2. aggs: 결과 문서가 출력되지 않더라도 실제 검색된 문서의 대상 범위가 전체 문서이기 때문에 집계는 전체 문서에 대해 수행된다. 

한번의 집계를 통해 질의에 해당하는 문서들 내에서도 집계를 수행하고 전체 문서에 대해서도 집계를 수행할 경우

```
{
    "query": {
        "constant_score": {
            "filter": {
                "match": <필드 조건>
            }
        }
    },
    "aggs": {
        "<집계 이름>": {
            "<집계 타입>": {
                "field": "<필드명>"
            }
        },
        "<집계 이름>": {
            "global": {},
            "aggs": {
                "<집계 이름>": {
                    "<집계 타입>": {
                        "field": "<필드명>"
                    }
                }
            }
        }
    }
}
```
1. 일반 버킷: 질의 영영 내에서만 집계를 수행
2. 글로벌 버킷: 전체 문서를 대상으로 집계를 수행.

---
## 메트릭 집계

특정 필드에 대해 합이나 평균을 계산하거나 다른 집계와 중첩해서 결과에 대해 특정 필드의 _score 값에 따라 정렬을 수행하거나 지리 정보를 통해 범위 계산을 하는 등의 다양한 집계를 수행할 수 있다. 

메트릭 집계 내에서도 단일 숫자 메트릭 집계와 다중 숫자 메트릭 집ㅈ계로 나뉘는데, 단일 숫자메트릭 집계는 집계를 수행한 결과값이 하나라는 의미로 sum과 avg등이 이에속한다. 다중 숫자 메트릭 집계는 짖ㅂ계를 수행한 결괏값이 여러개가 될 수 있고, stats나 geo_bounds가 이에 속한다. 

### 합산 집계는 단일 숫자 메트릭 집계에 해당한다. 이를 통해 해당 서버로 총 얼마만큼의 데이터가 유입됐는지 집계해본다.

```
GET apache-web-log/_search?size=0
{
    "aggs": {
        "total_bytes": {
            "sum": {
                "field": "bytes"
            }
        }
    }
}
```

filter를 사용하여 특정 조건만 검색한 결과
```
GET apache-web-log/_search?size=0
{
  "query": {
    "constant_score": {
      "filter": {
        "match": {"geoip.city_name" : "Paris"}
      }
    }
  },
  "aggs": {
    "total_bytes": {
      "sum": {
        "field": "bytes"
      }
    }
  }
}
```

script를 이용하면 KB, MB, GB단위로 사용자 정의로 변경하는 것이 가능하다.
```
GET /apache-web-log/_search?size=0
{
    "query": {
        "constant_score": {
            "filter": {
                "match": {"geoip.city_name": "Paris"}
            }
        }
    },
    "aggs": {
        "total_bytes": {
            "sum": {
                "script": { --- 1
                    "lang": "painless", --- 2
                    "source": "doc.bytes.value" --- 3
                }
            }
        }
    }    
}
```
1. script: 스크립트 컨텍스트를 의미하며, 6.x부터는 그루비가 아닌 페인리스를 기본 언어로 사용한다. 
2. lang: 페인리스 언어를 사용하는 경우 기본값이기 때문에 따로 명시하지 않아도 되지만 위 예제에서는 언어 확인을 위해 명시했다.
3. source: bytes 필드를 사용하려면 doc 객체의 bytes 속성 변수를 사용하고, 값을 얻기 위해 bytes객체의 value 속성 변수를 사용해야 한다. 이에 대해서는 페인리스 부분에서 살핀다. 


```
GET /apache-web-log/_search?size=0
{
    "query": {
        "constant_score": {
            "filter": {
                "match": {"geoip.city_name": "Paris"}
            }
        }
    },
    "aggs": {
        "total_bytes": {
            "sum": {
                "script": { 
                    "lang": "painless", 
                    "source": "doc.bytes.value / params.divide_value", --- 1
                    "params": { --- 2
                        "divide_value": 1000 --- 3
                    }
                }
            }
        }
    }    
}
```
1. byte 필드의 값을 script 내의 params(변수와 같은 의미)에 명시한 값으로 나눈다.
2. script 내에서 사용할 파라미터 값들을 정의한다.
3. divide_value 파라미터의 값으로 1000을 대입한다. 

