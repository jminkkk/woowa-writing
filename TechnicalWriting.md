# MySQL 검색 기능 성능 개선기 (Like 쿼리부터 전문 검색 인덱스 적용까지)

우리 서비스는 템플릿을 저장하고 조회할 수 있는데, 핵심 기능으로 템플릿 검색 기능이 있다.
검색 기능을 구현하기 위한 방법으로는 여러가지가 존재하는데, 각 방법과 한계에 대해 알아보자.

## LIKE 절을 사용한 패턴 매칭

LIKE 쿼리는 SQL에서 문자열 패턴 매칭을 위해 사용되는 연산자이다.
이를 통해 검색 키워드처럼 특정 패턴을 포함하는 데이터를 검색할 수 있다.

## LIKE 쿼리 사용 방법

LIKE 연산자는 WHERE 절에서 사용되며, 특정 패턴과 일치하는 문자열을 찾는다.
Like 쿼리와 함께 와일드 카드를 사용할 수 있는데 이를 통해 유연한 검색 기능을 사용할 수 있다.

- 와일드카드 사용법
  - `%`: 0개 이상의 문자를 대체
  - `_`: 단일 문자를 대체

다음의 예시를 통해 Like 쿼리 사용 방법을 살펴보자.

  ```sql
  SELECT * FROM users WHERE name LIKE '%John%'; # 이름에 John라는 단어가 들어간 유저를 조회
  SELECT * FROM users WHERE name LIKE 'John%'; # 이름에 John라는 단어로 시작하는 유저를 조회
  SELECT * FROM users WHERE name LIKE '%John'; # 이름에 John라는 단어로 끝나는 유저를 조회
  ```

## LIKE 쿼리문을 사용한 기존 검색 기능 

Like 쿼리는 대부분의 DBMS에서 지원을 하며 JPA에서도 별도의 설정 없이 기본적으로 사용이 가능하기 때문에 우리 서비스에서도 Like 쿼리와 와일드카드를 사용하여 검색 기능을 구현했었다.

하지만 이렇게 구현된 검색 API는 우리 서비스의 핵심 기능임에도 불구하고 가장 낮은 처리 속도를 보였다.

![image](https://github.com/user-attachments/assets/786a4f24-56af-4f01-8645-170a25bd6930)

바로 Like 쿼리를 사용하여 패턴매칭으로 검색을 구현한 것 때문이다.
Like 쿼리에는 치명적인 단점이 존재하는데 선행 와일드카드 ('%text')를 사용할 때 인덱스를 사용하지 못하고 전체 테이블 스캔이 발생한다는 것이다.


실제 우리 서비스에서 사용하는 쿼리의 실행 계획을 통해 어떠한 성능을 보이는지 확인해보자.


(성능 비교를 극대화하기 위해 총 템플릿 10만개, 소스코드는 총 30만개의 더미데이터를 추가하겠다. 각 템플릿은 최소 하나 이상의 소스코드를 가지고 있다.)


### 1.검색 조건인 템플릿과 소스코드의 컬럼에 인덱스를 사용하지 않았을 경우

![image](https://github.com/user-attachments/assets/06cda943-b8cc-466a-998d-26b12e52ec8c)

템플릿은 필터를 통해 전체 테이블을 조회하지는 않는다.

하지만 문제는 소스코드 내부 서브쿼리(sc1_0)에서는 332908개의 row를 실제 스캔하는 풀 테이블 스캔이 발생했으며, 이 부분이 쿼리 성능에 가장 큰 영향을 미치고 있는 것을 알 수 있다.


### 2.검색 조건인 템플릿과 소스코드의 컬럼에 인덱스를 사용했을 경우

```sql
ALTER TABLE `template` ADD INDEX `title_description` (`title`, `description`(255)); 
ALTER TABLE `source_code` ADD FULLTEXT INDEX `content_filename` (`content`, `filename`);
```

인덱스를 추가한 후 동일한 쿼리를 실행시켰다.

![image](https://github.com/user-attachments/assets/ccb96380-9380-437d-99d9-6a914b90b579)


처음과 동일하게 전체 소스코드에 대해 풀 테이블 스캔이 발생하는 것을 알 수 있다.

# 전문 검색 인덱스란

## 전문 검색 인덱스 기본 개념

전문 검색 인덱스란 MySQL을 포함한 몇몇 DBMS에서 지원하는 검색 성능을 향상시키기 위해 설계된 특수한 인덱스이다.
텍스트 기반 열(CHAR, VARCHAR 또는 TEXT 열)에 생성되어 해당 열에 포함된 데이터에 대한 쿼리 및 DML 작업 속도를 높힌다.

MySQL 에서 전체 텍스트 인덱스는 InnoDB 또는 MyISAM 스토리지 엔진에서만 사용할 수 있으며 CHAR, VARCHAR 또는 TEXT 열에 대해서만 만들 수 있다.

## 전문 검색 인덱스 사용법

전문 검색 인덱스를 사용하기 위해서는 먼저 테이블의 텍스트 컬럼에 대해 전문 검색 인덱스를 생성해야 한다.

### 1.전문 검색 인덱스 생성

  ```sql
ALTER TABLE mytable ADD FULLTEXT(column1, column2);

# 또는 

CREATE table mytable (
	# 생략
	FULLTEXT (title,body)
) ENGINE=InnoDB;
  ```

생성한 전문 검색 인덱스는 MATCH(), AGAINST() 구문을 사용하여 수행된다. 

### 2. MATCH() & AGAINST() 구문

MATCH()는 검색할 열의 이름을 쉼표로 구분한 목록을 받는다.
AGAINST()는 검색할 문자열과 수행할 검색 유형을 나타내는 선택적 수정자를 받는다. 

예시로 아래의 쿼리는 articles 테이블의 title, body 컬럼에 대해 'database'라는 키워드가 들어간 모든 행을 찾는다.

  ```sql
SELECT * FROM articles WHERE MATCH(title, body) AGAINST('database');
  ```


MATCH()는 쿼리 블록의 SELECT 목록, GROUP BY 절, HAVING 절 또는 ORDER BY 절에서 사용할 수 있다.

### 3. Mode 지정
FULLTEXT 검색에서는 사용할 수 있는 여러 가지 모드가 있는데 다음과 같다.

#### Natural Language Mode
기본 모드로, 자연어 쿼리 방식으로 검색을 수행하며 가장 관련성이 높은 결과를 반환한다.

  ```sql
SELECT * FROM woowacourses WHERE MATCH(crew, coach) AGAINST('moly');
  ```

이 모드는 기본적으로 가장 많이 등장하는 단어, 위치 등을 고려하여 결과를 정렬한다.

#### Boolean Mode
'+'와 '-' 기호를 사용하여 포함 또는 제외할 단어를 지정할 수 있는 모드이다.

  ```sql
SELECT * FROM woowacourses WHERE MATCH(crew, coach) AGAINST('+moly -pobi IN BOOLEAN MODE);
  ```

해당 쿼리는 moly라는 단어를 포함하고 pobi 단어를 제외한 로우만 조회한다.

#### Query Expansion
검색 쿼리를 확장하여 더 많은 관련 결과를 찾도록 도와준다. 쿼리에서 자주 발생하는 단어를 사용하여 결과를 보강한다.
  ```sql
SELECT * FROM woowacourses WHERE MATCH(crew, coach) AGAINST('moly' WITH QUERY EXPANSION);
  ```

해당 쿼리 실행 시에 “moly”라는 단어가 완전히 일차하지 않아도 즉, "moly's", "molying", 또는 "moly-like" 같은 단어들이 검색 결과에 포함한다.
## 전문 검색 인덱스 작동 방식

사용자가 전문 검색 인덱스를 사용하면 다음의 과정을 거쳐 검색이 수행된다.
1. 검색어 토큰화: 사용자의 검색어를 토큰화
2. 불용어 제거: 검색어에서 불용어를 제거
3. 인덱스 조회: 각 토큰에 대해 인덱스를 조회하여 관련 문서를 조회
4. 관련성 점수 계산: 문서의 관련성을 계산
5. 결과 정렬: 관련성 점수에 따라 결과를 정렬하여 반환

작동 방식에 대해 이해하기 위해 각 과정을 살펴보자.

### 1. 토큰화(Tokenization)
토큰화 과정에서는 텍스트를 개별 단어나 토큰으로 분리한다.
단어 분리: 공백, 구두점, 특수 문자 등을 기준으로 텍스트를 개별 단어로 나눔
대소문자 처리: 기본적으로 모든 문자를 소문자로 변환하여 대소문자 구분 없이 검색 가능하게 함
숫자 처리: 숫자도 토큰으로 인식되어 인덱싱
예를 들어, "I am Moly from Woowacourse"라는 문장은 "I", "am", "moly", "from", "woowacourse"로 토큰화될 수 있다.

#### 1.1 Full Text Parser
Full Text Parser는 토큰화 과정의 핵심 컴포넌트로, 텍스트를 의미 있는 단위(토큰)로 분리하는 역할을 한다. MySQL에서는 다양한 파서 옵션을 제공하여 다양한 언어와 사용 사례를 지원하는데, 기본 파서와 ngram 파서에 대해서만 간단하게 알아보자.

#### 1.1.1 기본 파서

- 공백과 구두점을 기준으로 단어를 분리한다.
- 알파벳과 숫자로 구성된 연속된 문자열을 하나의 토큰으로 인식한다
- 대소문자를 구분하지 않는다. (기본적으로 모든 문자를 소문자로 변환)

"나는 우아한테크코스의 몰리야"를 기본 파서로 분리하면 "나는", "우아한테크코스의", "몰리야"로 분리할 수 있다.

#### 1.1.2 N-gram 파서

- 주로 동아시아 언어(중국어, 일본어, 한국어 등)를 위해 사용된다.
- 단어 경계가 명확하지 않은 언어에 유용하다.
- 텍스트를 연속된 N개의 문자로 분리한다.

"나는 우아한테크코스의 몰리야"를 2-gram으로 분리하면 "나는", "우아", "한테", "크코”, “스의", “몰리", “야"로 분리할 수 있다.

N-gram 파서를 사용하고 싶다면 인덱스 생성 시 다음과 같이 파서 부분을 추가 작성해주면 된다.

  ```sql
ALTER TABLE 테이블명 ADD FULLTEXT INDEX 인덱스명(content) WITH PARSER ngram;
  ```

### 2. 불용어(Stopwords) 처리
불용어는 검색에 큰 의미가 없는 일반적인 단어들을 말한다. MySQL은 기본적으로 불용어 목록을 가지고 있으며, 이 단어들은 인덱스에서 제외한다.

- 기본 불용어: "a", "an", "the", "in", "on", "at" 등이 포함
- 사용자 정의 불용어: 기본 불용어 뿐만 아니라 사용자가 직접 불용어 목록을 수정할 수도 있다.

이처럼 불용어를 제거하면 인덱스 크기가 줄어들기 때문에 검색 성능이 향상될 수 있다.

<img src = https://github.com/user-attachments/assets/40dae7ae-1f2e-4958-85ef-4a994eca43b5 width= 50%>

### 3. 어간 추출(Stemming)
MySQL의 Full Text 검색은 기본적으로 정확한 단어 매칭을 수행하지만, 설정에 따라 어간 추출을 활용할 수도 있다. 그중에서도 어간 추출은 단어의 어미를 제거하여 기본 형태(어간)로 변환하는 과정을 말한다.
예시로 “running", "ran", "runs"와 같은 단어들을 "run"으로 통일하여 검색할 수 있다.
(MySQL의 기본 Full Text 검색에서는 제한적으로 지원되며, 플러그인을 통해 확장 가능하나 이런 기능이 있다는 것 정도만 알고 넘어가자)

### 4. 인덱스 조회
Full Text Index는 역인덱스(Inverted Index) 라는 특별한 데이터 구조를 사용하여 빠른 검색을 가능하게 한다.
#### 4.1 역인덱스란
Full Text Index의 핵심 구조이다. 쉽게 말하면 문서가 토큰에 대해 인덱스 정보를 가지고 있는 것이 아닌, 토큰이  자신이 포함된 문서 목록을 가지고 있는 것이다.
- 일반 인덱스
  - 구조: 문서 ID -> 문서 내용
  - 검색 과정: 모든 문서를 순회하며 검색어 포함 여부 확인
- 역인덱스
  - 구조: 단어(토큰) -> 해당 단어를 포함하는 문서 ID 목록
  - 검색 과정: 검색어에 해당하는 단어의 문서 목록을 직접 조회

<img src = https://github.com/user-attachments/assets/cbe48a88-c24b-4d58-ba24-89182115b904 width= 50%>

가장 많은 문서를 가지고 있는 단어 상위 10개를 조회해본 모습이다.


### 역인덱스 방식 장점

이렇게 역인덱스 방식을 사용하게 되면 다음과 같은 장점을 얻을 수 있다.

단어 스캔 시 관련 없는 문서는 처음부터 배제가 가능하다.
단어 각 단어의 전체 출현 빈도, 문서별 출현 빈도 등을 쉽게 계산할 수 있습니다.
AND, OR, NOT 등의 연산을 문서 ID 목록에 대한 집합 연산으로 쉽게 처리할 수 있다.
새 문서가 추가되면 해당 문서의 단어들만 역인덱스에 추가하면 된다.

## 전문 검색 인덱스 사용하여 개선하기

전문 검색 인덱스에 대해 알아보았으니 본격적으로 해당 인덱스를 사용하여 개선해보자.

  ```sql
ALTER TABLE `template` ADD FULLTEXT INDEX `ft_title_description` (`title`, `description`); 
ALTER TABLE `source_code` ADD FULLTEXT INDEX `ft_content_filename` (`filename`, `content`);
  ```

![image](https://github.com/user-attachments/assets/9336ff87-9ad7-4090-97ef-2af9a8c24326)

각 검색 조건이 사용되는 컬럼들에 대해 인덱스를 생성하고 실행 계획을 살펴보면
서브쿼리의 'type' 칼럼에 'fulltext'로 표시되어 있고 'Extra' 칼럼에 "Using where; Ft_hints: sorted"가 표시되어 있어, FULLTEXT 인덱스가 사용되고 있음을 확실히 알 수 있다.

analyze를 통해 더 자세한 실행 시간을 살펴보자.

![image](https://github.com/user-attachments/assets/11a94217-f8c7-403f-a82c-05545e3af2cf)

최초의 Like 쿼리를 사용했을 때의 총 소요 시간이 533이였던 것에 반해 27.2로 줄어든 것을 확인할 수 있다.


## 전문 검색 인덱스의 한계 

속도를 눈에 띄게 향상 시킬 수 있는 전문 검색 인덱스에도 한계는 존재한다.

1. 지원되는 데이터 형식 제한

FULLTEXT 인덱스는 CHAR, VARCHAR, TEXT 유형에서만 작동하기 때문에 다른 형식에 대해서는 적용할 수 없다.

2. 인덱스 크기와 검색 속도

FULLTEXT 인덱스는 테이블의 모든 텍스트 열에서 단어를 추출하여 관련된 정보를 저장한다. 이 과정에서 생성된 인덱스는 데이터의 양이 많아질수록 점점 커지며, 대량의 데이터를 검색할 때 성능 저하를 초래할 수 있다.

또한 데이터가 많을수록 FULLTEXT 인덱스의 크기가 커지며, 이는 저장소에 많은 공간을 차지하게 된다. 인덱스는 테이블의 모든 텍스트 열에서 발생하는 모든 단어를 기록하므로, 데이터의 양에 비례하여 인덱스 크기도 증가한다.

따라서 대량의 데이터를 저장하는 경우, FULLTEXT 인덱스는 상당한 양의 디스크 공간을 요구할 수 있습니다. 이는 데이터베이스의 성능에 영향을 줄 뿐만 아니라, 추가적인 저장 공간을 확보해야 할 가능성이 있다.

3. 최소 단어 길이, 불용어 설정

검색 가능한 최소 단어 길이, 불용어 등이 기본적으로 지정되어 있어 커스텀하기 위해서는 별도의 설정이 필요하다.

---

모든 기술이 그러하듯 한계는 존재한다.

때에 따라 적절한 기술을 선택하여 현재 상태에서의 최적화를 하는 것이 좋을 듯하다.

### 참고자료 

[MySQL 공식 문서: information-schema-innodb-ft-index-table-table](https://dev.mysql.com/doc/refman/8.4/en/information-schema-innodb-ft-index-table-table.html)
