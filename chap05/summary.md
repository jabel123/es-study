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

기존에는 결과가 428964 였는데 422가 됐다. 1000으로 나눴을 때 예상했던 값은 428 이었는데 422가 됐다. 이유는 1000으로 나누는 것은 모든 합산 값에 대한 나누기가 아니라 각 문서의 개별적인 값을 1000으로 나눴기 때문이다.

이 문제를 해결하기 위해서는 정수가 아닌 실수로 계산해서 소수점까지 합산해야 한다. 이를 위해 다음과 같이 캐스팅 연산을 수행한다.

```
GET /apache-web-log/_search?size=0
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
        "script": {
          "lang": "painless",
          "source": "doc.bytes.value/ (double)params.divide_value",
          "params": {
            "divide_value": 1000
          }
        }
      }
    }
  }
}
```

### 평균 집계
평균 집계는 단일 숫자 메트릭 집계에 해당한다. 이를 통해 해당 서버로 유입된 데이터의 평균 값을 쉽게 구할 수 있다.
```
GET /apache-web-log/_search?size=0
{
    "aggs": {
        "avg_bytes": {
            "avg": {
                "field": "bytes"
            }
        }
    }
}
````

### 최솟값 집계
최솟값 집계는 단일 숫자 메트릭 집계에 해당한다. 이를 통해 서버로 유입된 데이터중 가장 작은 값을 쉽게 구할 수 있다.

```
GET /apache-web-log/_search?size=0
{
  "aggs": {
    "min_bytes": {
      "min": {
        "field": "bytes"
      }
    }
  }
}
```

### 최댓값 집계
최댓값 집계는 단일 숫자 메트릭 집계에 해당한다. 이를 통해 서버로 유입된 데이터중 가장 큰 값을 구할 수 있다.

```
GET apache-web-log/_search?size=0
{
    "aggs": {
        "max-bytes": {
            "max": {
                "field": "bytes"
            }
        }
    }
}

```

### 개수 집계

개수집계는 단일 숫자 메트릭 집계에 해당한다. 이를 통해 해당 서버로 사용자 요청이 몇 회 유입됐는지 쉽게 궇할 수 있다. 
```
GET /apache-web-log/_search?size=0
{
    "aggs": {
        "bytes_count": {
            "value_count": {
                "field": "bytes"
            }
        }
    }
}
```

### 통계 집계

통계 집계는 결괏값이 여러 개인 다중 숫자 메트릭 집계에 해당한다. 통계 집계를 사용하면 앞서 살펴본 합, 평균, 최대/최솟값, 개수를 한번의 쿼리로 집계할 수 있다. 

```
GET /apache-web-log/_search?size=0
{
    "aggs": {
        "bytes_stats": {
            "stats": {
                "field": "bytes"
            }
        }
    }
}
```

### 확장 통계 집계
확장 통계 집계는 결괏값이 여러개인 다중 숫자 메트릭 집계에 해당한다. 앞서 살펴본 통계 집계를 확장해서 표준편차 같은 통곗값이 추가됐다.

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
        "bytes_stats": {
            "extended_stats": {
                "field": "bytes"
            }
        }
    }
}
```

*결과값*
```
{
  .... 생략 ....
  "aggregations": {
    "bytes_stats": {
      "count": 21,
      "min": 1015,
      "max": 53270,
      "avg": 20426.85714285714,
      "sum": 428964,
      "sum_of_squares": 18371748404,
      "variance": 457588669.3605442,
      "std_deviation": 21391.32229107271,
      "std_deviation_bounds": {
        "upper": 63209.501725002556,
        "lower": -22355.787439288277
      }
    }
  }
}
```
- sum_of_squares : 제곱합을 의미한다.
- variance: 확률변수의 분산
- std_deviation: 표준편차
- std_deviation_bounds: 표준편차의 범위
- upper: 표준편차의 상한값
- lower: 표준편차의 하한값

### 카디널리티 집계 

카디널리티 집계는 단일 숫자 메트릭 집계에 해당한다. 개수 집합과 유하게 횟수를 계산하는데, 중복된 값은 제외한 고유 값에 대한 집계를 수행한다. 하지만 모든 문서에 대해 중복된 값을 집계하는 것은 성능에 큰 영향을 줄 수 있기 때문에 근사치를 통해 집계를 수행한다.

```
GET /apache-web-log/_search?size=0
{
  "query": {
    "constant_score": {
      "filter": {
        "match": {
          "geoip.country_name": "United States"
        }
      }
    }
  },
  "aggs": {
    "us_city_names": {
      "terms": {
        "field": "geoip.city_name.keyword"
      }
    }
  }
}
```
- terms: 텀즈 쿼리를 사용한다.
- field: 개수를 집계할 필드를 명시한다. 각 문서에 명시된 필드별로 개수가 집계된다. string 타입의 필드의 경우 text 타입과 keyword 타입의 두 타입이 공존하게 되는데, text 타입의 경우 텍스트 검색용으로 사용하기 위해 분석기를 수행한후 색인되기 때문에 집계시에는 keyword 타입을 사용한다.

*결과*
```
... 생략
"aggregations": {
    "us_city_names": {
      "doc_count_error_upper_bound": 30,
      "sum_other_doc_count": 1955,
      "buckets": [
        {
          "key": "Leander",
          "doc_count": 539
        },
        {
          "key": "Lititz",
          "doc_count": 273
        },
        {
          "key": "San Francisco",
          "doc_count": 230
        },
        {
          "key": "Ashburn",
          "doc_count": 153
        },
        {
          "key": "New York",
          "doc_count": 134
        },
        {
          "key": "San Jose",
          "doc_count": 121
        },
        {
          "key": "Chantilly",
          "doc_count": 118
        },
        {
          "key": "Boydton",
          "doc_count": 115
        },
        {
          "key": "Morganton",
          "doc_count": 108
        },
        {
          "key": "Sunnyvale",
          "doc_count": 108
        }
      ]
    }
  }
  ```
집계된 결과를 살펴보면 미국 내에서 요청 수가 가장 많은 도시 순으로 결과가 반환된다. 그렇다면 미국내 몇 개의 도시에서 유입이 있었는지 확인하고 싶다면 어떻게 해야할까? 이러할 경우 카디널리트를 사용한다.

```
GET /apache-web-log/_search?size=0
{
  "query": {
    "constant_score": {
      "filter": {
        "match": {
          "geoip.country_name" :"United States"
        }
      }
    }
  },
  "aggs": {
    "us_cardinality": {
      "cardinality": {
        "field": "geoip.city_name.keyword"
      }
    }
  }
}
```


### 백분위 수 집계

백분위 수 집계는 다중 숫자 메트릭 집계에 해당한다. 백분위 수는 크기가 있는 값들로 이뤄진 자료를 순서대로 나열했을 때 백분율로 나타낸 특정 위치의 값을 이르는 용어다. 

아파트 웹로그에 이를 활용하면 유입된 데이터가 어느 정도 크기로 분포돼있는지 확인해볼 수 있다. 

```
GET /apache-web-log/_search?size=0
{
  "aggs": {
    "bytes_percentiles": {
      "percentiles": {
        "field": "bytes"
      }
    }
  }
}
```

### 백분위 수 랭크 집계

백분위 수 랭크 집계는 다중 숫자 메트릭 집계에 해당한다. 백분위 수 집계의 경우 백분위를 지정해서 각 백분위 수에 해당하는 수치를 확인할 수 있었다면 백분위수 랭크의 경우 특정 필드 수치를 통해 백분위의 어느 구간에 속하는지 확인할 수 있다.

```
GET /apache-web-log/_search?size=0
{
  "aggs": {
    "bytes_percentile_ranks": {
      "percentile_ranks": {
        "field": "bytes",
        "values": [5000, 10000]
      }
    }
  }
}
```

### 지형 집계

지형 경계 집계는 지형 좌표를 포함하고 있는 필드에 대해 해당 지역 경계 상자를 계산하는 메트릭 집계다. 
```
GET /apache-web-log-applied-mapping/_search?size=0
{
  "query": {
    "constant_score": {
      "filter": {
        "match": {
          "geoip.continent_code": "EU"
        }
      }
    }
  },
  "aggs": {
    "viewport": {
      "geo_bounds": {
        "field": "geoip.location",
        "wrap_longitude": true
      }
    }
  }
}
```

### 지형 중심 집계
```
GET /apache-web-log-applied-mapping/_search?size=0
{
  "aggs": {
    "centroid": {
      "geo_centroid": {
        "field": "geoip.location"
      }
    }
  }
}
```

---
## 버킷 집계

버킷 집계는 메트릭 집계와는 다르게 메트릭을 계산하지 않고 버킷을 생성한다. 생성되는 버킷은 쿼리와 함께 수행되어 쿼리 결과에 따른 컨텍스트 내에서 집계가 이뤄진다. 이렇게 집계된 버킷은 또 다시 하위에서 집계를 한 번 더 수행해서 집계된 결과에 대해 중첩된 집계를 수행하는 것이 가능하다. 

버킷을 생성한다는 것은 집계된 결과 데이터 집합을 메모리에 저장한다는 의미이기 떄문에 중첩되는 단계가 깊어질수록 메모리 사용량은 점점 더 증가해서 성능에 악영향을 줄 수 있다. 이러한 문제점 때문에 엘라스틱서치에서는 기본적으로 사용 가능한 최대 버킷 수가 미리 정의돼 있다. 그리고 이는 search.max_buckets 값을 변경하는 것으로 조정할 수 있다. 

만약 집계 질의를 요청할 때 버킷 크기를 -1(전체 대상) 또는 10000 이상의 값을 지정할 경우에는 엘라스틱서치에서 경고 메시지를 반환한다. 그러므로 안정적인 ㄱ집계를 위해서는 성능 측면을 충분하게 고려한 후 집계를 수행해야 한다. 

### 범위 집계

사용자가 지정한 범위 내에서 집계를 수행하는 다중 버킷 집계다. 짖ㅂ계가 수행되면 추출된 문서가 범위에 해당하는지 검증하게 되고, 범위에 해당하는 문서들에 대해서만 집계가 수행된다. 

아파치 웹로그에서 데이터 크기가 1000바이트에서 2000바이트 사이의 데이터를 대상으로 집계를 수행해본다.

```
GET /apache-web-log/_search?size=0
{
  "aggs": {
    "byte_range": {
      "range": {
        "field": "bytes",
        "ranges": [
          {
            "from": 1000,
            "to": 2000
          }
        ]
      }
    }
  }
}
```
질의 시 range 속성의 타입을 보면 배열 형태인 것을 볼 수 있는데, 짐작할 수 있듯이 여러 개의 범위를 지정해 각가 결과를 반홚받을 수 있다. 
```
GET /apache-web-log/_search?size=0
{
  "aggs": {
    "bytes_range": {
      "range": {
        "field": "bytes",
        "ranges": [
          {
            "to": 1000
          },
          {
            "from": 1000,
            "to": 2000
          },
          {
            "from": 2000,
            "to": 3000
          }
        ]
      }
    }
  }
}
```

### 날짜 범위 집계

범위집계와 유사하지만 숫자 값을 범위로 사용했던 범위 집계와는 달리 날짜 값을 범위로 집계를 수행한다. from 속성에는 시작 날짜 값을 설정하고, to 속성에는 범위의 마지막 날짜 값을 설정한다. 이때 범위 집계와 마찬가지로 마지막 날짜는 제외된다. 
```
GET /apache-web-log/_search?size=0
{
  "aggs": {
    "request count with date range" :{
      "date_range": {
        "field": "timestamp",
        "ranges": [
          {
            "from": "2015-01-04T05:14:00.000Z",
            "to": "2015-01-04T05:16:00.000Z"
          }
        ]
      }
    }
  }
}
```
### 히스토그램 집계

히스토그램 집계는 숫자 범위를 처리하기 위한 집계다. 지정한 범위 내에서 집계를 수행하는 범위 집계와는 달리 지정한 수치가 간격을 나타내고, 이 간격의 범위 내에서 집계를 수행한다. 

```
GET /apache-web-log/_search?size=0
{
  "aggs": {
    "bytes_histogram": {
      "histogram": {
        "field": "bytes",
        "interval": 10000
      }
    }
  }
}
```

10000을 범위로 하는 문서의 수가 합산된 결과를 확인할 수 있는데 문서 개수가 0인 간격도 포함돼 있다. 만약 문서가 존재하지 않는 구간은 필요하지 않다면 다음과 같이 최소 문서 수를 설정해서 해당 구간을 제외시킬 수 있다.

```
GET /apache-web-log/_search?size=0
{
  "aggs": {
    "bytes_histogram": {
      "histogram": {
        "field": "bytes",
        "interval": 10000,
        "min_doc_count": 1
      }
    }
  }
}
```

### 날짜 히스토그램 집계

날짜 히스토그램 집계는 다중 버킷 집계에 속하며 히스토그램 집계와 유사하다. 히스토그램 집계는 숫자 값을 간격으로 삼아 구간별 집계를 수행한 반면 날짜 히스토그램 집계는 분, 시간, 월, 연도를 구간으로 집계를 수행할 수 있다. 분 단위로 얼마만큼의 사용자 유입이 있었는지 확인해보기 위해 다음과 같은 집계를 사용한다. 
```
GET /apache-web-log/_search?size=0
{
  "aggs": {
    "daily_request_count": {
      "date_histogram": {
        "field": "timestamp",
        "interval": "minute"
      }
    }
  }
}
```

```
GET /apache-web-log/_search?size=0
{
  "aggs": {
    "daily_request_count": {
      "date_histogram": {
        "field": "timestamp",
        "interval": "day",
        "format": "yyyy-MM-dd"
      }
    }
  }
}
```


xkdlawhsdmf gksrnrtlfh tjfwjd
```
GET /apache-web-log/_search?size=0
{
  "aggs": {
    "daily_request_count": {
      "date_histogram": {
        "field": "timestamp",
        "interval": "day",
        "format": "yyyy-MM-dd",
        "time_zone": "+09:00"
      }
    }
  }
}
```

타임존과는 다르게 offset을 사용하면 집계 기준이 되는 날짜 값의 시작 일자를 조정할 수 있다. 예를 들어, 일자별로 집계를 수행했다면 offfset값을 "+3h"로 지정한 경우 3시부터 집계가 시작되도록 변경할 수 있다.
```
GET /apache-web-log/_search?size=0
{
  "aggs": {
    "daily_request_count": {
      "date_histogram": {
        "field": "timestamp",
        "interval": "day",
        "offset": "+3h"
      }
    }
  }
}
```

### 텀즈 집계
텀즈 집계(Terms Aggregations)는 버킷이 동적으로 생성되는 다중 버킷 집계다. 집계 시 지정한 필드에 대해 빈도수가 높은 텀의 순위로 결과가 반환된다. 이를 통해 가장 많이 접속하는 사용자를 알아낸다거나 국가별로 어느 정도의 빈도로 서버에 접속하는지 등의 집계를 수행할 수 있다. 텀즈 집계를 통해 아파치 서버로 얼마만큼의 요청이 들어왔는지를 국가별로 집계해보자.

```
GET /apache-web-log/_search?size=0
{
  "aggs": {
    "request count by country": {
      "terms": {
        "field": "geoip.country_name.keyword"
      }
    }
  }
}
```

집계를 수행할 때는 각 샤드에 집계 요청을 전달하고, 각 샤드는 집계 결과에 대해 정렬을 수행한 자체 뷰를 갖게 된다. 이것들을 병합해서 최종 뷰를 만들기 때문에 포함되지 않은 문서가 존재하는 경우에는 집계 결과가 정확하지 않을 수 있다.

모든 문서를 포함할 수 있도록 size값을 100으로 지정한 후 질의를 수행하도록 한다. 
```
GET /apache-web-log/_search?size=0
{
  "aggs": {
    "request count by country": {
      "terms": {
        "field": "geoip.country_name.keyword",
        "size":100
      }
    }
  }
}
```

집계에 포함되지 않은 문서들을 포함시키기 위해 size값을 늘리면 그만큼 집계의 정확도가 높아진다. 하지만 버킷에 더 많은 양의 데이터를 담아야 하기 때문에 메모리 사용량과 결과를 계산하는데 드는 처리 비용 또한 증가한다.

```
집계와 샤드 크기
텀즈 집계가 수행될 때 검색 프로세스를 관장하는 노드에서는 각 샤드에게 최상위 버킷을 제공하도록 요청한 후에 모든 샤드로 부터 결과를 받을 떄까지 기다린다. 결과를 기다리다가 모든 샤드로부터 결과를 받으면 설정된 size에 맞춰 하나로 병합한 후 클라이언트에 결과로 전달한다.
각 샤드에서는 정확성을 위해 size의 크기만큼이 아닌 샤드 크기를 이용한 경험적인 방법(샤드크기 * 1.5+10)을 사용해 내부적으로 집계를 수행하는데, 텀즈 집계의 결과로 받을 텀의 개수를 정확하게 파악할 수 있는 경우에는 shard_size속성을 사용해 각 샤드에서 집계할 크기를 직접 지정해 불필요한 연산을 줄이면서도 정확성을 높일 수 있다.
앞서 설명한 바와 같이 shard_size가 기본값(-1)으로 설저된 경우에는 엘라스틱서치가 샤드 크기를 기준으로 자동으로 추정한다. 만약 shard_size를 직접 설정할 경우에는 size보다 작은 값은 설정할 수 없음에 주의해야 한다. 
```

## **파이프라인 집계**

파이프라인 집게는 다른 집계와 달리 쿼리 조건에 부합하는 문서에 대해 집계를 수행하는 것이 아니라 다른 집계로 생성된 버킷을 참조해서 집계를 수행한다. 집계 또는 중첩된 집계를 통해 생성된 버킷을 사용해 추가적으로 계산을 수행한다고 보면 된다. 파이프라인 집계에는 부모, 형제라는 두 가지 유형이 있따. 
집계를 참조하는 방법은 다음과 같다.
```
AGG_SEPARATOR = '>';
METRIC_SEPARATOR = '.';
AGG_NAME = <집계 이름> ;
METRIC = <메트릭 집계 이름(다중 메트릭 집계인 경우)>;
PATH = <AGG_NAME> [ <AGG_SEPARATOR>, <AGG_NAME> ]* <METRIC_SEPARATOR>, <METRIC> ];
```

### 형제 집계

형제 집계(Silbling Aggregation)는 동일 선상의 위치에서 수행되는 새 집계를 의미한다. 즉, 형제 집계를 통해 수행되는 집계는 기존 버킷에 추가되는 혀태가 아니라 동일 선상의 위치에 새 집계가 생성되는 파이프라인 집계다. 형제 집계에는 다음과 같은 집계가 포함된다. 

- 평균 버킷 집계
- 최대 버킷 집계
- 최소 버킷 집계
- 합계 버킷 집계
- 통계 버킷 집계
- 확장 통계 버킷 집계
- 백분위수 버킷 집게
- 이동 평균 집계

아파치 웹로그에서 분 단위로 합산된 데이터량 중 가장 큰 데이터량을 구하고 싶은경우는 어떻게 할까? 기존 방법으로는 date_histogram과 그 하위 집계로 sum집계를 수행한 후 가장 큰 값을 추려야 한다.

파이프라인 집계 중 최대 버킷 집계를 통해 분단위 데이터량 합산과 더불어 가장 큰 데이터량을 구해본다.

```
GET /apache-web-log/_search?size=0
{
  "aggs": {
    "histo": {
      "date_histogram": {
        "field": "timestamp",
        "interval": "minute"
      },
      "aggs": {
        "bytes_sum": {
          "sum": {
            "field": "bytes"
          }
        }
      }
    },
    "max_bytes": {
      "max_bucket": {
        "buckets_path": "histo>bytes_sum"
      }
    }
  }
}
```

### 부모집계

부모집계는 집계를 통해 생성된 버킷을 사용해 계산을 수행하고, 그 결과를 기존 집계에 반영한다. 부모 집계에 해당하는 집계는 다음과 같다. 

- 파생집계
- 누적집계
- 버킷 스크립트 집계
- 버킷 셀렉터 집계
- 시계열 차분 집계

아파치 웹 로그를 통해 수집된 데이터가 시간이 지남에 따라 변화하는 값의 변경폭 추이를 확인하고 싶은 경우 파생 집계를 활용할 수 있다. 파생 집계는 부모 히스토그램 또는 날짜 히스토그램 집계에서 지정된 메트릭의 파생 값을 계산하는 상위 파이프라인 집계다. 

이는 부모 히스토그램 집계의 측정 항목에 대해 작동하고, 히스토그램 집계에 의한 각 버킷의 집계 값을 비교해서 차이를 계산한다. 지정된 메트릭은 숫자여야 하고, 상위에 해당하는 집계의 min_doc_count가 0보다 큰 값으로 설정되는 경우 일부 간격이 결과에서 생략될 수가 있기 떄문에 min_doc_count 값을 0으로 설정해야 한다. 

파생 집계의 경우에는 이처럼 선행되는 데이터가 존재하지 않으면 집계를 수행할 수가 없는데, 실제 데이터를 다루다 보면 종종 노이즈가 포함되기도 하고, 필요한 필드에 값이 존재하지 않을수도 있댜. 이러한 부분을 갭(gap)이라 할 수 있는데, 쉽게 말해 데이터가 존재하지 않는 부분을 의미한다. 갭은 여러가지 이유로 발생할 수 있으며, 일반적인 이유는 다음과 같다.

- 하나 이상의 버킷에 대한 쿼리와 일치하는 문서가 존재하지 않는 경우
- 다른 종속된 버킷에 누락되어 계산된 메트릭이 값을 생성할 수가 없는 경우

이러한 경우에는 파이프라인 집계에 원하는 동작을 알리는 메커니즘이 필요하다. 이러한 역할을 하는 것이 갭 정책(gap_policy)다. 모든 파이프라인 집계에서는 gap_policy 파라미터를 허용한다. 현재 두 가지 갭 정책을 선택할 수 있다.

- skip: 이 옵션은 누락된 데이터를 버킷이 존재하지 않는 것으로 간주한다. 버킷을 건너뛰고 다음으로 사용 가능한 값을 사용해 계산을 계속해서 수행한다.
- insert_zeros: 이 옵션은 누락된 값을 0으로 대체하며 파이프라인 집계 계산은 정상적으로 진행된다. 

```
GET /apache-web-log/_search?size=0
{
  "aggs": {
    "histo": { --- 1)
      "date_histogram": {
        "field": "timestamp",
        "interval": "day"
      },
      "aggs": {
        "bytes_sum": { --- 2)
          "sum": {
            "field": "bytes"
          }
        },
        "sum_deriv": { --- 3)
          "derivative": {
            "buckets_path": "bytes_sum"
          }
        }
      }
    }
  }
}
```
1. 부모 히스토그램. 일 단위로 집계를 수행한다.
2. 일 단위로 데이터의 크기를 합산하는 sum집계를 수행한다.
3. 일 단위로 데이터 크기의 합산된 값을 버킷의 현재 값과 이전 값을 비교하는 집계를 수행한다. 

이처럼 부모 집계는 부모에 해당하는 상위 집계를 통해 생성된 버킷에 대해 집계를 수행한다. 부모 집게에 해당하는 집계로는 앞서 언급했던 것처럼 파생/누적/버킷스크립트/버킷셀렉터/시계열 차분 집계가 있다. 이러한 집계 연산은 다음과 같이 수행한다.

```
// 파생집계
{
  "derivative": {
    "buckets_path": "bytes_sum"
  }
}

// 누적집계
{
  "cumulative_sum": {
    "buckets_path": "bytes_sum"
  }
}

// 버킷 스크립트 집계
{
  "bucket_script": {
    "buckets_path": {
      "my_var1": "bytes_sum",
      "my_var2": "total_count"
    },
    "script": "params.my_var1 / params.my_var2"
  }
}

// 버킷 셀렉터
{
  "bucket_selector": {
    "buckets_path": {
      "my_var1": "bytes_sum",
      "my_var2": "total_count"
    },
    "script": "params.my_var1 > params.my_var2"
  }
}

// 시계열 차분 집계
{
  "serial_diff": {
    "buckets_path": "bytes_sum",
    "lag": 7
  }
}
```

## 근사값으로 제공되는 집계 연산

엘라스틱서치에서는 다양한 종류의 집계 연산을 제공한다. 간단한 숫자 연산인 Sum, Max, Avg부터 복잡한 좌표 기반의 연산까지 다양하게 제공된다. 대부분의 경우 전체 데이터를 대상으로 집계가 일어나기 때문에 결과도 매우 정확하게 제공된다. 하지만 일부 집계 연산의 경우에는 성능 문제로 근삿값을 기반으로 한 결과를 제공한다. 

일반적으로 사용자들은 모든 연산의 계산 결과가 정확할 것이라고 생각하기 때문에 어떠한 집계 연산이 근삿값을 제공하는지 명확히 알지 못하면 자칫 큰 낭패를 볼 수 있다. 

### 집계 연산과 정확도 

현재 엘라스틱서치에서 제공하는 집계 연산의 종류가 많고 다양하다. 집계 기능은 다른 검색엔진과의 차별화를 위해 엘라스틱서치에서 강력하게 내세우는 핵심 기술로서 버전이 올라갈수록 집계 여낫ㄴ의 성능도 좋아지고 그 종류도 지속적으로 증가하고 있다.

앞서 엘라스틱서치에서 제공하는 대부분의 집계 연산을 소개했지만 집계 연산의 종류가 워낙 많다보니 엘라스틱서치를 처음 접하는 독자는 매우 혼란스러울 수도 있을 것이다. 하지만 기능별로 집게 연산들이 잘 분류돼 있기 때문에 분류 기준만 파악하면 쉽게 이해할 수 있을 것이다.

- 버킷 집계(Buckets Aggregations): 특정 기준을 충족하는 문서들을 버킷으로 분류하는 집계
- 메트릭 집계(Metrics Aggregations):버킷에 존재하는 문서들을 이용해 각종 통계 지표를 생성하는 집계
- 파이프라인 집계(Pipeline Aggregations): 서로 다른 메트릭 집계의 출력을 연결하는 집계 패밀리
- 행렬 집계(Matrix Aggregations): 추출한 값을 기반으로 결과를 행렬로 생성하는 집계 패밀리

### 일반적인 계산을 위한 집계 연산
```
평균 집계(Avg Aggregation)
- 문서에서 추출한 숫자값의 평균을 계산하는 메트릭 집계
- 키워드: avg

카디널리티 집계(Cardinality Aggregation)
- 문서의 특정 필드에 대한 Unique연산을 근삿값으로 계산하는 메트릭 집계
- 성능 문제로 내부적으로 HyperLogLog++ 알고리즘을 이용해 근삿값으로 계산된다.
- 정확도를 위해 precision_threshold 옵션을 제공한다.
- 키워드: cardinality

최댓값 집계(Max Aggregation)
- 문서에서 추출한 숫자값의 최댓값을 계산하는 메트릭 집계
- 키워드: max

최소값 집계(Min Aggregation)
- 문서에서 추출한 숫자값의 최솟값을 계산하는 메트릭 집계
- 키워드: min

합산 집계(Sum Aggregation)
- 문서에서 추출한 숫자값의 합계를 계산하는 메트릭 집계
- 키워드: sum
```

### 고차원 계산을 위한 집계 연산
```
지형 경계 집계(Geo Bounds Aggregation)
- geo_point 타입의 필드를 기반으로 경곗값을 계산하는 메트릭 집계
- 위도, 경도를 계산하여 데이터 그룹의 top_left 좌표와 buttom_right좌표를 제공한다.
- 키워드: geo_bounds

지형 중심 집계(Geo Centroid Aggregation)
- geo_point 타입의 필드를 기반으로 중앙값을 계산하는 메트릭 집계
- 위도, 경도를 계산해서 데이터 그룹의 중앙 좌표를 제공한다.
- 키워드: geo_centroid

백분위 수 집계(Percentiles Aggregation)
- 문서에서 추출한 숫자값의 백분위 수를 계산하는 메트릭 집계
- 성능 문제로 내부적으로 T-Digetst 알고리즘을 이용해 근삿값으로 게산된다.
- 버킷의 양이 작을수록 100%의 정확도를 제공하고 커질수록 정확도가 떨어진다.
- 키워드: percentiles

백분위 수 랭크 집계(Percentaile Ranks Aggregation)-
- 백분위 수 랭킹을 계산하는 메트릭 집계
- 백분위 수 집계와 마찬가지로 근사값으로 계산된다.
- 키워드: percentile_ranks
```

### 특수한 목적을 위한 집계 연산
```
통계 집계(Stats Aggregation)
- 문서의 각종 통계 지표를 한번에 계산하는 매트릭 집계
- count, min, max, avg, sum 등을 제공한다.
- 키워드: stats

확장 통계 집계(Extended Stats Aggregation)
- 문서의 각종 통계 지표를 한번에 계산하는 확장된 메트릭 집계
- count, min, max, avg, sum 등을 제공한다.
- 키워드: extended_stats

탑 히트 집계(Top Hits Aggregation)
- 전체 문서 중에서 가장 관련성이 높은 문서만을 대상으로 하는 메트릭 집계
- 다양한 기준으로 관련성을 계산할 수 있다.

스크립트 메트릭 집계(Scripted Metric Aggregation)
- 스크립트만을 이용한 메트릭 집계

추출값 카운트 집계(Value Count Aggregation)
- 집계되는 결과의 카운트를 설정하는 메트릭 집계
```

ES에서 제공하는 대부분의 집계 연산은 100%의 정확한 결과를 제공한다. 일부 집계 연산은 근삿값을 기반으로 한 집계 결과를 제공하며, 다수의 집계 연산 중 근삿값을 제공하는 집계 연산은 다음 세 가지다. 

- 카디널리티 집계
- 백분위 수 집계
- 백분위 수 랭크 집계

### 분산환경에서 집계 연산의 어려움

ES는 루씬에서 기본적으로 제공하는 그루핑 기술인 Facet API를 사용하지 않는다. 루씬은 분산 처리를 지원하지 않기 때문에 Facet에서 제공하는 집계 기능이 대용량의 분산 처리에는 적합하지 않기 때문이다. 이러한 이유로 엘라스틱서치는 독자적인 그루핑 기술인 Aggregation API를 제공한다.

Aggregation API는 버킷 기반의 집계 기능을 독자적으로 구현했기 때문에 분산되어 저장된 대용량 데이터에 대한 집계 연산을 매우 빠르게 처리할 수 있다. 데이터가 샤드의 개수만큼 분산되어 보관되고 있기 때문에 요청이 들어올 경우 각 샤드 내에서 1차적으로 집계 작업이 수행되고, 중간 결과를 모두 취합해서 최종 결과를 만들어 제공하는 방식으로 동작한다. 

검색 작업과 집계 작업도 모두 이 같은 방식으로 처리되기 때문에 검색을 실행하는 동시에 집계 결과를 반환할 수도 있다. 한 번의 실행으로 검색 결과와 집계 결과를 한 번에 제공할 수 있어 성능 측면에서도 효율적이다. 이러한 방식은 분산된 노드가 데이터라 할지라도 다수의 샤드에 적절히 분산돼 이싿는 가정만 성립된다면 빠르고 정확한 결과를 제공하는 것이 가능해진다. 

