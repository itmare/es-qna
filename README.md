indexing할때, POST와 PUT의 차이점
---------------------------------

-	PUT

	-	전체 doc를 업데이트한다.
	-	반복 수행시, 하나의 document를 업데이트한다.
	-	PUT으로 생성된 document는 `_id`와 함께 생성되기 때문에 새로운 document의 추가는 없다.
	-	output의 `_version`을 통해서 업데이트 횟수를 확인할 수 있다.

-	POST

	-	minor update를 수행, 보내진 field만 없데이트하고 document에 있는 다른 field들은 바뀌지 않는다.
	-	POST로 생성된 document는 `_id`없이도 실행 가능
	-	`_id`없을경우 반복 실행시, 같은 내용의 document가 계속 추가된다.
	-	es는 임의로 `_id`를 생성한다.

-	mapping와 indexing를 구분하여 비교

-	indexing

```shell
# body가 있을때 '_doc' 사용가능

PUT my_index # default setting과 함께 껍데기 index 생성
PUT my_index/_doc {...} # 에러: id 필요
PUT my_index/_doc/1 {...} # 생성

POST my_index # 에러
POST my_index/_doc {...} # 생성
POST my_index/_doc/1 {...} # 생성
```

-	mapping

```shell
PUT my_index {"mappings":{...}} # field mapping과 함께 index 생성
POST my_index {"mappings":{...}} # 에러
```

<br><br>

es에서의 `_version`
-------------------

-	transaction 처리 시, 동시성 제어를 위해 사용된다.

<br><br>

field type을 text? keyword? 혼돈의 카오스
-----------------------------------------

-	text: search를 위한 type
-	keyword: sort, aggregation을 위한 type

<br><br>

Analyzer
--------

<br><br>

reindex 활용
------------

-	reindex시, setting값은 가져가지 않는다.

<br><br>

analysis
--------

```json
PUT blogs_analyzed
{
  "settings": {
    "analysis": {
      "char_filter": {
        "cpp_it":{
          "type": "mapping",
          "mappings": ["c++ => cpp", "C++ => cpp", "IT => _IT_"]
        }
      }
      ,"tokenizer": {

      },
      "filter": {
        "my_stop":{
          "type": "stop",
          "stopwords":["can", "we", "our", "you", "your", "all"]
        }
      },
      "analyzer": {
        "my_analyzer":{
          "type": "custom",
          "char_filter": ["cpp_it"],
          "tokenizer": "standard",
          "filter": ["lowercase", "stop", "my_stop"]
        }
      }
    }
  },


  "mappings": {
    "properties" : {
      "author" : {
        "type" : "text",
        "fields" : {
          "keyword" : {
            "type" : "keyword",
            "ignore_above" : 256
          }
        }
      },
      "category" : {
        "type" : "keyword"
      },
      "content" : {
        "type" : "text",
        "fields": {
          "my_analyzer":{
            "type" : "text",
            "analyzer": "my_analyzer"
          }
        }
      },
      "locales" : {
        "type" : "keyword"
      },
      "publish_date" : {
        "type" : "date"
      },
      "seo_title" : {
        "type" : "text",
        "fields" : {
          "keyword" : {
            "type" : "keyword",
            "ignore_above" : 256
          }
        }
      },
      "title" : {
        "type" : "text",
        "fields" : {
          "keyword" : {
            "type" : "keyword",
            "ignore_above" : 256
          },
          "my_analyzer":{
            "type" : "text",
            "analyzer": "my_analyzer"
          }
        },
        "analyzer": "my_analyzer"
      },
      "url" : {
        "type" : "text",
        "fields" : {
          "keyword" : {
            "type" : "keyword",
            "ignore_above" : 256
          }
        }
      }
    }
  }
}
```

<br><b>

match query
-----------

-	multiple field는 지원하지 않는다.

```shell
# false
"query": {
    "match": {
      "title": "open source software",
      "minimum_should_match": 2
    }
  }

# true
"query": {
  "match": {
    "title": {
      "query": "open source software",
      "minimum_should_match": 2
    }
  }
}
```

<br><br>

Query DSL (Domain Specific Language)
====================================

-	Leaf query clauses

	-	하나의 특정 필드에 대한 결과값을 볼 수 있다.
	-	`match`, `term`, `range` 등

-	Compound query clauses

	-	여러 leaf 또는 compound query를 포함하여 여러가지 쿼리를 묶어 그에 대한 결과 값을 볼 수 있다.
	-	`bool`과 같은 절에 `must`, `must_not`, `filter`와 같은 clause를 사용

<br><br>

Boolean Query
-------------

-	Compound query 중 하나
-	여러 query를 조회해야 할때 사용
-	`"bool"` 안에 사용되는 clause(query)
	-	`"must"`: 해당 clause안에 있는 모든 쿼리가 반드시 결과에 포함되어야 함(AND), score에 영향을 미친다.
	-	`"filter"`: 해당 clause안에 있는 모든 쿼리가 반드시 결과에 포함되어야 함(AND), score에 영향을 미치지 않는다.
	-	`"should"`: 해당 clause안에 있는 쿼리 적어도 하나가 결과에 포함되어야 함(OR), (`minimum_should_match`와 관련)
-	만약 `bool` query가 적어도 하나의 `should` clause만 가지고 있다면(No `must` and `filter`), `minimum_should_match`의 값은 default값인 1이다. 반대경우에는, 0이다.

<br><br>

aggregation이 정확한가?
-----------------------

-	`"terms": { ... }`안에 `show_term_doc_count_error`를 사용하여 확인

<br><br>

bulk API
--------

```shell
# Request

POST /_bulk
{"<action>": {}}

POST /<index>/_bulk
{"<action>": {}}
```

-	newline delimited JSON (NDJSON) 구조를 사용한다.
-	4가지 action 제공

	-	`index`: body 필요, URL에 index를 명시 했다면, header에 `"index":{}`만 명시해도 된다. header에 `"_id"`를 입력시, 해당 `"_id`가 index에 존재하면 replace한다.
	-	`create`: body 필요, header에 "`_id`" 반드시 필요
	-	`delete`: body 필요없음, `"_index"`(옵션), `"_doc"`(필수)만 사용
	-	`update`: body 필요하고 반드시 body안에 `"doc"` 필드를 명시해줘야 한다.

-	index와 create: "index"는 doc을 인덱싱한다. "create"는 doc이 존재하지 않을때 doc을 인덱싱한다. "create"는 index를 생성하지 않기때문에 "index"보다 빠르다.

<br><br>

scroll API
----------

-	대량의 데이터 조회시 사용
-	RDBMS의 cursor와 같은 기능

```shell
# search context 생성, 1분이 지나면 소멸
POST /<index>/_search?scroll=1m
{
  "size": 500, # 한 페이지에 볼 문서의 갯수
    "_source": [ "<field1>",  "<field2>"],  # 보고 싶은 field 리스트
    "query": {
      "match_all": {}  # 전체 문서 조회
  }
}


# 위의 결과에서 확인가능한 "_scroll_id"를 사용하여, Scroll API에서 사용
GET/POST /_search/scroll
{
  "scroll": "1m",  # search context를 1분 더 유지시키라고 es에 전달
  "scroll_id": "<my_scroll_id>" # "_scroll_id"
}
```

-	search context가 유지되는 동안 es서버의 메모리를 점유하기 때문에, 전체 데이터 fetch가 생각보다 빨리 끝난다면, 명시적으로 해당 context를 제거해 주는 것이 좋다.

```shell
DELETE _search/scroll
{
  "scroll_id": "<id>"
}
```

<br><br>

Snapshot
--------

-	fs, hdfs, s3, gcs, azure에 인덱스를 백업할 수 있다.

```shell
# snapshot을 저장할 repository 생성하기
PUT _snapshot/<snapshot_repo_name>
{
  "type": "hdfs", # hdfs에 snapshot 저장
  "settings": {
    "uri": "hdfs://<address>:8020/",  # hdfs uri와 path를 설정, fs는 "location"
    "path": "/es-snapshot"
  }
}

# snapshot할 인덱스를 선정하고 실행
PUT _snapshot/<snapshot_repo_name>/<snapshot_name>
{
  "indices": "<index_name>" ,
  "ignore_unavailable": true ,
  "include_global_state": true
}


# restore하기
POST _snapshot/<snapshot_repo_name>/<snapshot_name>/_restore
{
  "indices": "<index_name1>,<index_name2>", #multi index system 지원
  "ignore_unavailable": true ,
  "include_global_state": false
}
```

-	`"ignore_unavailable"` : 특정 인덱스가 없거나 close상태 일떄, 무시할지 여부를 결정한다. (true / false)
-	`"include_global_state"` : snapshot에 cluster global status가 저장되는 것을 방지 (true / false / partial)
-	index의 snapshot은 incremental 방식이므로, 마지막 백업이후에 변경된 부분만 추가된다.

<br>

```shell
# snapshot repository 리스트 확인
GET _snapshot

# snapshot 리스트 확인
GET _cat/snapshots/<REPO_NAME>?v
# repo 삭제
DELETE _snapshot/<REPO_NAME>

# snapshot 삭제
DELETE _snapshot/<REPO_NAME>/<SNAPSHOT_NAME>
```

<br><br>

replica갯수 변경 시
===================

```shell
# 다음과 같이 변경
PUT my_refresh_test/_settings
{
  "number_of_replicas": 1  
}

# 해당 인덱스가 존재한다고 에러메세지 나옴
PUT my_refresh_test
{
  "settings": {
    "number_of_replicas": 1
  }
}
```

<br><br>

Dynamic Templates
=================

-	mapping시, 확실치 않은 field name에 대해 custom mapping 할 수 있게 해준다.

\-

<br><br>

Query vs. Filter
================

-	query
	-	질문예제: 이 query가 document와 얼마나 잘 매치하냐? (여러 결과값, score에 따라 나열)
-	filter
	-	질문예제: 이 query는 document와 매치하냐 안하냐? (yes/no, score 필요없음)

<br><br>

Aggregation
-----------

### Metrics Aggregation

### Bucket Aggregation

### Aggregation 사용시, POST 와 GET 의 차이점

-	결과의 차이 없다.
-	개념적으로 GET에는 body가 들어가지 않는다. 그러나 언어에 따라 GET사용시, body를 가져오거나 가져오지 않는 경우가 있다.

-	GET 사용시 body 붙이기

	-	가능: curl, python requests, C# HttpClient .NET Core,
	-	불가능: Unity HttpClient(예외 발생), C# RestSharp(body 전달안됨),
	-	UnityWebRequest(상황에 따라 변경, GET 요청시 body가 붙어있으면 POST로 보냄)
	-	elasticsearch는 request body를 POST가 아닌 요청에 붙일 수 없는 라이브러리에 대해서 query string을 대신 사용

<br><br>

doc_values vs. fielddata (정리필요)
===================================

-	doc_values:
	-	기본적으로 `text` type field는 지원하지 않는다. -
-	fielddata:
	-	`keyword` type field는 지원하지 않는다.
	-	대신 `doc_values`를 사용해라.
	-	주로 field에 대해 sorting

```json
# 관련 예제: ee2, lab2-12

PUT test
{
  "mappings": {
    "properties": {
      "message": {
        "type": "text"
      },
      "level": {
          "type": "keyword",
          "doc_values": false
      }
    }
  }
}


PUT test/_bulk
{ "index" : { "_id" : "1"}}
{ "level" : "INFO", "message" : "recovered [20] indices into cluster_state"}
{ "index" : { "_id" : "2"}}
{ "level" : "WARN", "message" : "received shard failed for shard id 0"}
{ "index" : { "_id" : "3"}}
{ "level" : "INFO", "message" : "Cluster health status changed from [YELLOW] to [GREEN]"}
{ "index" : { "_id" : "4"}}
{ "level" : "INFO", "message" : "Cluster health status changed from [RED] to [YELLOW]"}


GET test/_search
{
  "size": 0,
  "aggs": {
    "top_levels": {
      "terms": {
        "field": "level",
        "size": 5
      }
    }
  }
}


GET test/_search
{
  "size": 0,
  "aggs": {
    "top_message_words": {
      "terms": {
          "field": "message",
          "size": 5
      }
    }
  }
}
```

<br><br>

Scripting
=========

```shell
# script 저장하기
POST _scripts/<SCRIPT_ID>
{
  "script":{
    "lang": "painless",
    "source": """
ctx._source.number_of_views += params.new_value
"""
  }
}
# ctx._source: update context에 _source field 사용
# number_of_views: 기존 field 중 하나
# params: script의 clause중 하나인 'params'
# new_value: script에서 variable로 사용할 parameter로써, 아무 이름이 나 설정

POST blogs/_update/<DOC_ID>
{
	"script": {
		"id": "<SCRIPT_ID>",
		"params": {
			new_value: 100
		}
	}
}
# id: script 저장 시, 정의 했던 SCRIPT_ID 사용
# new_value: script 저장 시, 사용했던 params.new_value 값을 initializing
```

<br><br>

update by query
===============

-	특정한 쿼리의 결과값에 해당하는 데이터를 한번에 update하는 쿼리

<br><br>

Ingest node
===========

-	도큐먼트를 인덱싱하기 전에, 도큐먼트를 pre-processing하기 위해 ingest node를 사용한다.
-	순서
	-	ingest pipeline 정의하기
	-	simulate을 통해 검증하기
	-	update by query를 통하여 해당 인덱스 업데이트 하기 (search query의 json body 사용하면 편함)

```shell
# ingest pipeline 정의하기
PUT _ingest/pipeline/<PIPELINE_ID>
{
	"description": "",
	"processors": [
		{
			"script": {
				"lang": "painless",
				"source": """
				<SCRIPT_CONTENT>
				"""
			}
		}
	]
}
# PIPELINE_ID: 원하는 파이프라인 이름 설정
# CONTENT: 전처리하고자 하는 스크립트 입력


# 정의된 ingest pipeline의 시뮬레이션 테스트
POST _ingest/pipeline/<PIPELINE_ID>/_simulate
{
	"docs": [
		{
			<KEYVALUE_FOR_SCRIPT>
		}
	]
}
# KEYVALUE_FOR_SCRIPT: SCRIPT_CONTENT와 관련된 도큐먼트의 _source를 포함한 key:value 등을 입력
```

-	각 프로세서의 순서 중요, 순서에 따라 결과가 달라진다.

```shell
# example 1: null인 필드의 값을 변경하고, array 구조인 값을 특정 separator로 나누어 array로 만들때,
PUT _ingest/pipeline/fix_locales
{
  "processors": [
    {
      "script": {
        "lang": "painless",
        "source": """
        if(ctx.locales == "")
        {
          ctx.locales = "en-en";
        }
        ctx.reindexBatch = 3;
        """
      }
    }
    ,
    {
      "split": {
        "field": "locales",
        "separator": ","
      }
    }
  ]
}
# output 1





# example 2: array 구조의 값을 나누고, 해당 값의 값을 변경할때,
PUT _ingest/pipeline/fix_locales
{
  "processors": [
    {
      "split": {
        "field": "locales",
        "separator": ","
      }
    }
    ,
    {
      "script": {
        "lang": "painless",
        "source": """
        if(ctx.locales == "")
        {
          ctx.locales = "en-en";
        }
        ctx.reindexBatch = 3;
        """
      }
    }
  ]
}
# output 2
{
  "docs" : [
    {
      "doc" : {
        "_index" : "_index",
        "_type" : "_doc",
        "_id" : "_id",
        "_source" : {
          "locales" : [
            ""
          ],
          "reindexBatch" : 3
        },
        "_ingest" : {
          "timestamp" : "2020-01-07T05:09:06.932058Z"
        }
      }
    }
  ]
}
```

foreach processor
-----------------

```
- array의 모든 값들이 같은 방법으로 processor의 처리가 필요할 때 사용
- split 이후 단계에서 사용
- field
```

-	ingest metadata안에 array element context의 `_ingest._value` key에 넣는다.

```shell
# example: 스페이스를 제거하고 `,`를 seperator로 사용하여 array로 변경
PUT _ingest/pipeline/fix_locales
{
  "processors": [
    {
      "split": {
        "field": "locales",
        "separator": ","

      }
    }
    ,
    {
      "foreach": {
        "field": "locales",
        "processor": {
          "trim": {
            "field": "_ingest._value"
          }
        }
      }
    }
  ]
}

# simulation
POST _ingest/pipeline/fix_locales/_simulate
{
  "docs": [
    {
      "_source": {
        "locales": "hello          , world     , this is    "
      }
    }
  ]
}

# 결과
{
  "docs" : [
    {
      "doc" : {
        "_index" : "_index",
        "_type" : "_doc",
        "_id" : "_id",
        "_source" : {
          "locales" : [
            "hello",
            "world",
            "this is"
          ],
          "reindexBatch" : 3
        },
        "_ingest" : {
          "_value" : null,
          "timestamp" : "2020-01-07T07:41:05.479768Z"
        }
      }
    }
  ]
}
```

<br><br>

script fields
=============

-	script를 통해 기존 index에 있는 field를 사용하여 원하는 값으로 변경하고 리턴한다.
-	script field는 저장되지 않는다.

```shell
# example
GET logs_server1/_search
{
  "_source": [],  # 모든 source값들 리턴
  "query": {
    "bool": {
      "filter": [
        {
          "exists": {
            "field": "geoip.city_name"  # 해당 필드값이 있을때 리턴
          }
        },
       {
          "exists": {
            "field": "geoip.region_name" # 해당 필드값이 있을때 리턴
          }
        }
      ]
    }
  },
  "script_fields": {
    "full_region_name": {   # 리턴 될 새로운 필드의 이름
      "script": {
        "lang": "painless",
        "source": """
        doc['geoip.city_name.keyword'].value + ', ' + doc['geoip.region_name.keyword'].value
        """
      }
    }
  }
}
```

<br><br>

painless 사용 시, field value 접근을 위한 syntax
================================================

| Context                                      | Syntax for accessing fields |
|:--------------------------------------------:|:---------------------------:|
| **Ingest node**: access fields using **ctx** |     **ctx.field_name**      |
|    **Updates**: use the **_source** field    | **ctx._source.field_name**  |
|             **Search and aggs**              | **doc['field_name'].value** |

<br><br>

Search Template
===============

-	mustache 언어를 사용해서 search request를 사전 작업한다.
-	`source`안에 query 구조는 기본 search 구조에서 작성하면 편하다.

```shell
# 기본 구조

# search template 정의하기
POST _scripts/<SEARCH_TEMPLATE_ID>
{
	"script": {
        "lang": "mustache",
        "source": """
				{
            "query": {
                "match": {
                    "title": "{{param1}}"
                },
								"range": {
									"@timestamp": {
										"gte": "{{param2}}",
										"lte": "{{param3}}"
									}
								}
            }
        }
				"""
    }
}

# search template 사용하여 쿼리하기
GET my_index/_search/template
{
	"id": "<SEARCH_TEMPLATE_ID>", # 정의한 search template id
	"param": {
		"param1": "your_value1", # search template에서 key값의 value
		"param2": "your_value2",
		"param3": "your_value3"
	}
}
```
