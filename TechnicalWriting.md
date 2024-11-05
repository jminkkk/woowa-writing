# MySQL Full Text Index를 사용하여 검색 기능 성능 개선하기 (feat. JPA에서의 사용)

저희 코드잽 서비스는 코드를 "템플릿"으로 저장하고 조회할 수 있습니다. (템플릿이라는 용어가 많이 등장하니 기억해주시면 감사하겠습니다.)

그중에서도 핵심 기능 중 하나는 "템플릿 검색 기능"입니다. 

검색 기능을 구현하기 위한 방법으로는 여러 가지가 존재하는데, 이번 글에서는 기존 방식의 한계와 개선 방법, 그리고 도입을 하면서 발생했던 트러블 슈팅에 대해 공유하겠습니다.

## LIKE 쿼리문을 사용한 기존 검색 기능 

Like 쿼리는 대부분의 DBMS에서 지원을 하며 JPA에서도 별도의 설정 없이 기본적으로 사용이 가능하기 때문에 우리 서비스에서도 Like 쿼리와 와일드카드를 사용하여 검색 기능을 구현했었습니다.

하지만 이렇게 구현된 검색 API는 우리 서비스의 핵심 기능임에도 불구하고 가장 낮은 처리 속도를 보였습니다.

![image](https://github.com/user-attachments/assets/786a4f24-56af-4f01-8645-170a25bd6930)

바로 Like 쿼리를 사용하여 패턴 매칭으로 검색을 구현한 것 때문인데요.

## LIKE 절을 사용한 패턴 매칭

LIKE 쿼리는 SQL에서 문자열 패턴 매칭을 위해 사용되는 연산자입니다.
이를 통해 검색 키워드처럼 특정 패턴을 포함하는 데이터를 검색할 수 있습니다.

## LIKE 쿼리 사용 방법

Like 쿼리와 함께 와일드카드를 사용할 수 있는데 이를 통해 유연한 검색 기능을 사용할 수 있습니다.

- 와일드카드 사용법
  - `%`: 0개 이상의 문자를 대체
  - `_`: 단일 문자를 대체

다음의 예시는 Like 쿼리 사용 방법입니다.

  ```sql
  SELECT * FROM users WHERE name LIKE '%John%'; # 이름에 John라는 단어가 들어간 유저를 조회
  SELECT * FROM users WHERE name LIKE 'John%'; # 이름에 John라는 단어로 시작하는 유저를 조회
  SELECT * FROM users WHERE name LIKE '%John'; # 이름에 John라는 단어로 끝나는 유저를 조회
  ```


실제 우리 서비스에서 사용하는 쿼리의 실행 계획을 통해 어떠한 성능을 보이는지 확인해 보겠습니다.

(성능 비교를 극대화하기 위해 총 템플릿 10만개, 템플릿별 소스코드는 총 30만개의 더미 데이터를 추가하였습니다. 각 템플릿은 최소 하나 이상의 소스코드를 가지고 있습니다)


### 1. 검색 조건인 템플릿과 소스코드의 컬럼에 인덱스를 사용하지 않았을 경우

![image](https://github.com/user-attachments/assets/06cda943-b8cc-466a-998d-26b12e52ec8c)

템플릿 컬럼과, 소스코드에 대해 풀 테이블 스캔이 발생했으며 특히, 내부 서브쿼리(sc1_0)에서 실행 시간이 4.29 ~ 4.264초가 걸립니다.
즉, 이 부분이 쿼리 성능에 큰 영향을 미치고 있는 것을 알 수 있었습니다.


### 2. 검색 조건인 템플릿과 소스코드의 컬럼에 인덱스를 사용했을 경우

```sql
ALTER TABLE `template` ADD INDEX `title_description` (`title`, `description`(255)); 
ALTER TABLE `source_code` ADD INDEX `content_filename` (`content`, `filename`);
```

그렇다면 인덱스를 넣으면 개선이 되지 않을까요?
한번 인덱스를 추가한 후 동일한 쿼리를 실행시켜 보겠습니다.

![image](https://github.com/user-attachments/assets/ccb96380-9380-437d-99d9-6a914b90b579)


처음과 동일하게 전체 소스코드에 대해 풀 테이블 스캔이 발생하는 것을 알 수 있습니다.

## Like 쿼리의 한계

바로, 이 Like 쿼리에는 치명적인 단점인 선행 와일드카드 ('%text')를 사용할 때 인덱스를 사용하지 못하고 전체 테이블 스캔을 발생시킨다는 것 때문입니다.

# 전문 검색 인덱스란

보통 텍스트 기반의 검색을 구현하기 위해 사용되는 개념이 있는데요.
바로 전문 검색 인덱스입니다. 

## 전문 검색 인덱스 기본 개념

전문 검색 인덱스란 MySQL을 포함한 몇몇 DBMS에서 지원하는 검색 성능을 향상하기 위해 설계된 특수한 인덱스입니다.
텍스트 기반 열(CHAR, VARCHAR 또는 TEXT 컬럼)에 생성되어 사용할 수 있습니다.

> MySQL에서 전문 검색 인덱스는 InnoDB 또는 MyISAM 스토리지 엔진에서만 사용할 수 있으며 CHAR, VARCHAR 또는 TEXT 컬럼에 대해서만 적용이 가능하니, 참고해 주세요.

## 전문 검색 인덱스 사용법

이제 이 전문 검색 인덱스를 사용하여 키워드 검색을 개선해 보겠습니다.

전문 검색 인덱스를 사용하기 위해서는 먼저 테이블의 텍스트 컬럼에 대해 전문 검색 인덱스를 생성해야 합니다.

### 1.전문 검색 인덱스 생성

```sql
ALTER TABLE mytable ADD FULLTEXT(column1, column2);

# 또는 

CREATE table mytable (
	# 생략
	FULLTEXT (title,body)
) ENGINE=InnoDB;
```

### 2. MATCH() & AGAINST() 구문

생성한 전문 검색 인덱스는 MATCH(), AGAINST() 구문을 사용하여 수행됩니다. 

MATCH()는 검색할 열의 이름을 쉼표로 구분한 목록을 받고, 
AGAINST()는 검색할 문자열을 지정해 주면 사용이 가능합니다.

예시로 아래의 쿼리는 woowacourse 테이블의 name, nickname 컬럼에 대해 'moly'라는 키워드가 들어간 모든 행을 찾습니다.

  ```sql
SELECT * FROM woowacourse WHERE MATCH(name, nickname) AGAINST('moly');
  ```

### 3. Mode 지정
그 다음으로 FULLTEXT 검색에서는 사용할 수 있는 여러 가지 모드를 지정해줄 수 있습니다.

#### Natural Language Mode
기본 모드로, 자연어 쿼리 방식으로 검색을 수행하며 가장 관련성이 높은 결과를 반환합니다.

  ```sql
SELECT * FROM woowacourses WHERE MATCH(crew, coach) AGAINST('moly');
  ```

이 모드는 기본적으로 가장 많이 등장하는 단어, 위치 등을 고려하여 결과를 정렬합니다.

#### Boolean Mode
'+'와 '-' 기호를 사용하여 포함 또는 제외할 단어를 지정할 수 있는 모드입다.

  ```sql
SELECT * FROM woowacourses WHERE MATCH(crew, coach) AGAINST('+moly -pobi IN BOOLEAN MODE);
  ```

해당 쿼리는 moly라는 단어를 포함하고 pobi 단어를 제외한 로우만 조회하게 됩니다.

## 전문 검색 인덱스 작동 방식

사용 방법에 대해 알아보았는데요. 그 음으로 간단하게 어떻게 동작하는지 확인해보겠습니다.

1. 검색어 토큰화: 사용자의 검색어를 토큰화
2. 불용어 제거: 검색어에서 불용어를 제거
3. 인덱스 조회: 각 토큰에 대해 인덱스를 조회하여 관련 문서를 조회
4. 관련성 점수 계산: 문서의 관련성을 계산
5. 결과 정렬: 관련성 점수에 따라 결과를 정렬하여 반환

### 1. 토큰화(Tokenization)
토큰화 과정에서는 텍스트를 개별 단어나 토큰으로 분리합니다.
- 단어 분리: 공백, 구두점, 특수 문자 등을 기준으로 텍스트를 개별 단어로 나눔
- 대소문자 처리: 기본적으로 모든 문자를 소문자로 변환하여 대소문자 구분 없이 검색 가능하게 함
- 숫자 처리: 숫자도 토큰으로 인식되어 인덱싱
예를 들어, "I am Moly from Woowacourse"라는 문장은 "I", "am", "moly", "from", "woowacourse"로 토큰화될 수 있다.

#### 1.1 파서
파서는 토큰화 과정의 핵심 컴포넌트로, 텍스트를 의미 있는 단위인 토큰으로 분리하는 역할을 합니다. MySQL에서는 다양한 파서 옵션을 제공하여 다양한 언어와 사용 사례를 지원하는데, 기본 파서와 ngram 파서에 대해서만 간단하게 알아봅시다.

#### 1.1.1 기본 파서

- 공백과 구두점을 기준으로 단어를 분리한다.
- 알파벳과 숫자로 구성된 연속된 문자열을 하나의 토큰으로 인식한다
- 대소문자를 구분하지 않는다. (기본적으로 모든 문자를 소문자로 변환)

"나는 우아한테크코스의 몰리야"를 기본 파서로 분리하면 "나는", "우아한테크코스의", "몰리야"로 분리할 수 있습니다.

#### 1.1.2 N-gram 파서

- 주로 동아시아 언어(중국어, 일본어, 한국어 등)를 위해 사용되는데요.
- 텍스트를 연속된 N개의 문자로 분리하기 때문에 단어 경계가 명확하지 않은 언어에 유용합니다.

"나는 우아한테크코스의 몰리야"를 2-gram으로 분리하면 "나는", "우아", "한테", "크코”, “스의", “몰리", “야"로 분리할 수 있습니다.

기본 파서를 사용하면 "나는 우아한테크코스의 몰리야"를 "몰리" 라고만 키워드를 입력하면 조회가 불가능하지만 N-gram 파서로는 가능합니다.

N-gram 파서를 사용하고 싶다면 인덱스 생성 시 다음과 같이 파서 부분을 추가 작성해주면 됩니다.

  ```sql
ALTER TABLE 테이블명 ADD FULLTEXT INDEX 인덱스명(content) WITH PARSER ngram;
  ```

### 2. 불용어(Stop words) 처리
불용어는 검색에 큰 의미가 없는 일반적인 단어들을 말합니다. MySQL은 기본적으로 불용어 목록을 가지고 있으며, 이 단어들은 인덱스에서 제외하니 주의해야 합니다.

- 기본 불용어: "a", "an", "the", "in", "on", "at" 등이 포함
- 사용자 정의 불용어: 기본 불용어뿐만 아니라 사용자가 직접 불용어 목록을 수정할 수도 있다.

이처럼 불용어를 제거하면 인덱스 크기가 줄어들기 때문에 검색 성능이 향상될 수 있습니다.

<img src = https://github.com/user-attachments/assets/40dae7ae-1f2e-4958-85ef-4a994eca43b5 width= 50%>

### 3. 어간 추출(Stemming)

MySQL의 Full Text 검색은 기본적으로 정확한 단어 매칭을 수행하지만, 설정에 따라 어간 추출을 활용할 수도 있습니다. 그중에서도 어간 추출은 단어의 어미를 제거하여 기본 형태(어간)로 변환하는 과정을 말합니다.
예시로 "running", "ran", "runs"와 같은 단어들을 "run"으로 통일하여 검색할 수 있게 됩니다.

(MySQL의 기본 Full Text 검색에서는 제한적으로 지원되며, 플러그인을 통해 확장할 수 있으나 이런 기능이 있다는 것 정도만 알고 넘어가도 될 것 같습니다)

### 4. 인덱스 조회

Full Text Index는 역인덱스(Inverted Index) 라는 특별한 데이터 구조를 사용하여 빠른 검색을 가능하게 하는데요.

#### 4.1 역인덱스란
역인덱스란 이 Full Text Index의 핵심 구조입니다. 쉽게 말하면 문서가 토큰에 대해 인덱스 정보를 가지고 있는 것이 아닌, 토큰이 자신이 포함된 문서 목록을 가지고 있는 것을 말합니다.

- 일반 인덱스
  - 구조: 문서 ID -> 문서 내용
  - 검색 과정: 모든 문서를 순회하며 검색어 포함 여부 확인
- 역인덱스
  - 구조: 단어(토큰) -> 해당 단어를 포함하는 문서 ID 목록
  - 검색 과정: 검색어에 해당하는 단어의 문서 목록을 직접 조회

<img src = https://github.com/user-attachments/assets/cbe48a88-c24b-4d58-ba24-89182115b904 width= 50%>

가장 많은 문서를 가지고 있는 단어 상위 10개를 조회해 본 모습입니다.


### 역인덱스 방식 장점

이렇게 역인덱스 방식을 사용하게 되면 다음과 같은 장점을 얻을 수 있습니다.

- 단어 스캔 시 관련 없는 문서는 처음부터 배제가 가능합니다.
- 단어 각 단어의 전체 출현 빈도, 문서별 출현 빈도 등을 쉽게 계산할 수 있습니다.
- AND, OR, NOT 등의 연산을 문서 ID 목록에 대한 집합 연산으로 쉽게 처리할 수 있습니다.
- 새 문서가 추가되면 해당 문서의 단어들만 역인덱스에 추가하면 됩니다.

## 전문 검색 인덱스 성능 측정하기

전문 검색 인덱스에 대해 알아보았으니 본격적으로 해당 인덱스를 사용하기 전 성능을 측정해 보겠습니다.

먼저 해당하는 컬럼에 대해 전문 검색 인덱스를 추가합니다.

  ```sql
ALTER TABLE `template` ADD FULLTEXT INDEX `ft_title_description` (`title`, `description`); 
ALTER TABLE `source_code` ADD FULLTEXT INDEX `ft_content_filename` (`filename`, `content`);
  ```

그 후 'Match ... Against' 구절을 사용하여 실행 계획을 확인해보겠습니다.

![image](https://github.com/user-attachments/assets/9336ff87-9ad7-4090-97ef-2af9a8c24326)

각 검색 조건이 사용되는 컬럼들에 대해 인덱스를 생성하고 실행 계획을 살펴보면
서브쿼리의 'type' 칼럼에 'fulltext'로 표시되어 있고 'Extra' 칼럼에 "Using where; Ft_hints: sorted"가 표시되어 있어, FULLTEXT 인덱스가 사용되고 있음을 확실히 알 수 있습니다.

analyze를 통해 더 자세한 실행 시간을 살펴보면, 최초의 Like 쿼리를 사용했을 때의 총 소요 시간이 533이였던 것에 반해 27.2로 줄어든 것을 확인할 수 있습니다.

![image](https://github.com/user-attachments/assets/11a94217-f8c7-403f-a82c-05545e3af2cf)

# JPA 기반 기존 구현에 전문 검색 인덱스 사용하여 개선하기

확실한 성능까지 확인하였으니 이제 애플리케이션 코드에 적용해 보겠습니다.

### Hibernate의 MySQLDialect 커스텀하기

그런데 문제가 하나 있습니다. 'MATCH ... AGAINST' 구문을 지원하지 않기 때문에 직접 SQL 함수를 등록해야 합니다. 
따라서 Hibernate의 MySQLDialect를 커스텀해야 합니다. 

```java
// MySQLDialect를 확장한 FullText 검색을 위한 Dialect 클래스
public class FullTextSearchMySQLDialect extends MySQLDialect {
   @Override
   public void initializeFunctionRegistry(FunctionContributions functionContributions) {
       super.initializeFunctionRegistry(functionContributions); // 부모 클래스의 함수 등록 실행
       SqmFunctionRegistry functionRegistry = functionContributions.getFunctionRegistry();
       functionRegistry.register("match", ExactPhraseMatchFunction.INSTANCE); 
       // 'match' 함수를 ExactPhraseMatchFunction으로 등록
   }

   // MySQL의 MATCH AGAINST 구문을 구현하는 내부 클래스
   public static class ExactPhraseMatchFunction extends NamedSqmFunctionDescriptor {
   
       public static final ExactPhraseMatchFunction INSTANCE = new ExactPhraseMatchFunction();
       // 싱글톤 패턴으로 인스턴스 생성

       public ExactPhraseMatchFunction() {
           super("MATCH", false, StandardArgumentsValidators.exactly(3), null);
           // MATCH 함수명 지정, 정확히 3개의 인자를 받도록 설정
       }
       
       @Override
       public void render(SqlAppender sqlAppender, List<? extends SqlAstNode> arguments, 
                        ReturnableType<?> returnType, SqlAstTranslator<?> translator) {
           // MySQL의 전문 검색 구문 생성:
           // MATCH(column1, column2) AGAINST (search_term IN NATURAL LANGUAGE MODE)
           sqlAppender.appendSql("MATCH(");
           translator.render(arguments.get(0), SqlAstNodeRenderingMode.DEFAULT); // 첫 번째 칼럼
           sqlAppender.appendSql(", ");
           translator.render(arguments.get(1), SqlAstNodeRenderingMode.DEFAULT); // 두 번째 칼럼
           sqlAppender.appendSql(") AGAINST (");
           translator.render(arguments.get(2), SqlAstNodeRenderingMode.DEFAULT); // 검색어
           sqlAppender.appendSql(" IN NATURAL LANGUAGE MODE)");
           // NATURAL LANGUAGE MODE로 전문 검색 수행
       }
   }
}
```

### Hibernate의 커스텀 방언 등록하기
JPA는 특정 데이터베이스에 종속되지 않고 사용할 수 있도록 설계되어 있습니다.

이를 위해 Hibernate는 각 데이터베이스별로 SQL 문법의 차이를 추상화한 방언(Dialect)을 제공하는데, 앞서 커스텀한 방언을 Hibernate는 알 방법이 없기 때문에 직접 등록해주어야 합니다.

```yml
# application.yml
jpa:
  properties:
    hibernate:
      dialect: codezap.template.repository.FullTextSearchMySQLDialect # 파일 위치 
```

### 등록한 함수 호출하여 사용하기

```java
	Predicate titleDescriptionPredicate = criteriaBuilder.isTrue(
 		criteriaBuilder.function("MATCH", Boolean.class, // 등록한 함수 호출
			root.get("title"),
			root.get("description"),
			criteriaBuilder.literal(searchKeyword)));
                
		Subquery<Long> sourceCodeSubquery = query.subquery(Long.class);
		Root<SourceCode> sourceCodeRoot = sourceCodeSubquery.from(SourceCode.class);
			sourceCodeSubquery.select(sourceCodeRoot.get("template").get("id"));
        	sourceCodeSubquery.where(criteriaBuilder.isTrue(
    			criteriaBuilder.function("MATCH", Boolean.class, // 등록한 함수 호출
					sourceCodeRoot.get("content"),
					sourceCodeRoot.get("filename"),
			criteriaBuilder.literal(searchKeyword))));

```

## 테스트 시 주의점

검색에 대해 레포지토리 테스트가 있었는데 키워드 관련 조건이 있는 경우 모두 실패했습니다.

@DataJpaTest 을 사용한 이 테스트는 내부적으로 @Transactional을 부착하고 있는데, 따라서 영속성 컨텍스트의 변경사항은 트랜잭션이 커밋되는 시점에 flush 되어 데이터베이스에 반영됩니다. 

즉,테스트 내에서 저장한 데이터는 실제로 데이터베이스에 반영되지 않았던 것입니다.

앞에서 등록한 **하이버네이트 방언은 데이터베이스 엔진 단에서 처리를 해주는 것이기 때문에 반드시 데이터베이스에 저장된 컬럼만 인덱스 적용이 가능합니다.**

저희 서비스에서는 테스트 메서드 내부에서 저장하는 방식이 아닌, @Sql 을 통해 미리 DB 단에 더미데이터를 저장한 후, 조회하는 방식으로 변경하였습니다.

## 결론

모든 기술이 그러하듯 한계는 존재합니다.

때에 따라 적절한 기술을 선택하여 현재 상태에서의 최적화를 판단하시면 좋을 것 같습니다.

### 참고 자료 

[MySQL 공식 문서: information-schema-innodb-ft-index-table-table](https://dev.mysql.com/doc/refman/8.4/en/information-schema-innodb-ft-index-table-table.html)
