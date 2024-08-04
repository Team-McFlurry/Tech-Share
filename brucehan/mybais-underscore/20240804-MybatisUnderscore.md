# 리팩토링이라는 삽질을 통해 알아낸 MyBatis의 매핑 원리와 세팅 옵션

## 개요

Spring Framework, MyBatis 스택의 실무 환경에서 resultMap 으로 반환타입을 묶은, 4천 줄의 코드 라인을 30% 가량 줄이면서 겪었던 삽질(?)과, 이를 해결하면서 알게 된 MyBatis의 매핑 원리와 활용했던 옵션에 대해 설명한다.

---

## 기존에 있던 VO와 xml에 있던 resultMap, select문

### 필드와 getter메서드만 있는 VO 클래스

```java
public class EmpSalaryVO {
    private String bonusitem1;
    private String bonusitem2;
    private String bonusitem3;
    private String bonusitem4;
    private String bonusitem5;
    private String bonusitem6;
    private String bonusitem7;
    private String bonusitem8;
    private String bonusitem9;
    private String bonusitem10;
    private String bonusitem11;
    private String bonusitem_total;

    public String getBonusItem1() {
        return bonusitem1;
    }
    public String getBonusItem1() {
        return bonusitem1;
    }
    public String getBonusItem2() {
        return bonusitem2;
    }
    public String getBonusItem3() {
        return bonusitem3;
    }
    public String getBonusItem4() {
        return bonusitem4;
    }
    public String getBonusItem5() {
        return bonusitem5;
    }
    public String getBonusItem6() {
        return bonusitem6;
    }
    public String geBonustItem7() {
        return bonusitem7;
    }
    public String getBonusItem8() {
        return bonusitem8;
    }
    public String getBonusItem9() {
        return bonusitem9;
    }
    public String getBonusItem10() {
        return bonusitem10;
    }
    public String getBonusItem11() {
        return bonusitem11;
    }
    public String getBonusItem_total() {
        return bonusitem_total;
    }
}
```


### resultMap을 쓰고 있는 select문

```xml
...

<resultMap id="EmpSalaryMap" type="EmpSalaryVO">
    <result column="BONUSITEM1" property="bonusitem1">
    <result column="BONUSITEM2" property="bonusitem2">
    <result column="BONUSITEM3" property="bonusitem3">
    <result column="BONUSITEM4" property="bonusitem4">
    <result column="BONUSITEM5" property="bonusitem5">
    <result column="BONUSITEM6" property="bonusitem6">
    <result column="BONUSITEM7" property="bonusitem7">
    <result column="BONUSITEM8" property="bonusitem8">
    <result column="BONUSITEM9" property="bonusitem9">
    <result column="BONUSITEM10" property="bonusitem10">
    <result column="BONUSITEM11" property="bonusitem11">
    <result column="BONUSITEM_TOTAL" property="bonusitem_total">
</resultMap>

<select id="selectEmployeeSalary" parameterType="HashMap" resultMap="EmpSalaryMap">
    select
        A.ba1 AS BONUSITEM1,
        A.ba2 AS BONUSITEM2,
        A.ba3 AS BONUSITEM3,
        A.bb1 AS BONUSITEM4,
        A.bb2 AS BONUSITEM5,
        A.bb3 AS BONUSITEM6,
        A.bc1 AS BONUSITEM7,
        A.bc2 AS BONUSITEM8,
        A.bc3 AS BONUSITEM9,
        B.bd1 AS BONUSITEM10,
        B.dd2 AS BONUSITEM11,
        (A.sum + B.sum) AS BONUSITEM_TOTAL
    from emp_salarya A, emp_salaryb B
    where id=#{id}
</select>
```

#### resultMap이란?

resultMap은 **컬럼명과 프로퍼티명이 바인딩 규칙을 벗어날 때** 사용한다.

resultMap을 사용하면 반환된 result를 **사용자가 원하는 프로퍼티에 자유롭게 바인딩**할 수 있다.  
즉, 반환된 ResultSet을 지정한 result 객체에 전달하는 **중간 역할을 수행**한다.

![매핑 구문에 Result Map 지정](specifyResultMap.png)

resultMap을 구성하는 기본 단위는 `result 구성 요소`이며, `<result />`를 사용해서 지정한다.

![매핑 구문에 resultMap 지정](resultMapProcess.png)


### AJAX로 요청했던 합계데이터 뿌리기

이렇게 조회한 합계 데이터는 `_total`이라는 텍스트를 붙여 값을 뿌려준다.

```javascript
$.ajax({
  ...
  success : function(result) {
      // 합계 데이터를 tr 태그 사이로 뿌려줌
      html += "<tr>" + result[0][type + '_total'] + "</tr>";
  }
});
```


## 문제 발생 및 해결 과정

### 리팩토링 후 문제 발생

기존에는 조회하는 컬럼들을 모두 alias로 이름을 따로 지정해준 뒤 resultMap으로 한 번 더 이름을 매핑하던 방식을 썼었다.

이러한 방식은 급여의 종류가 추가됨에 따라 컬럼을 추가하고 데이터를 넣는 등의 작업을 했을 때, `<resultMap>`안에 `<result>` 요소를 일일이 스크롤을 돌려가면서 추가해야했다.

여기서 resultMap을 제거한 뒤 alias만 사용하도록 하면 코드의 양도 줄고 컬럼 및 필드를 추가하는 작업을 할 때 한결 수월해질거라 생각하여, resultType 속성으로 VO클래스를 직접 매핑하도록 했다.

값들이 map 없이도 잘 나오는 걸 확인한 후에 운영에 코드를 반영했지만, 알고 보니 각 종류의 월급 값들은 잘 나와도 합계가 안 나오는 것이었다.  
확인해보니, 총합을 나타내는 `_total`은 매핑이 되지 않고 있었다.


### 찝찝한 핫픽스 후 다시 리팩토링

서버를 직접 내렸다 올려야 하는 운영 특성상 서버를 일과시간에 중단할 수 없어 프론트(jsp)단에서 해결하기로 했다.  
프론트 단에서 `total` 값을 직접 출력하지 않고, for문으로 각 값을 합한 값을 출력하는 방식으로 급하게 반영을 마무리 했다.

핫픽스 후 이상은 없었지만, 프론트 단에는 이전부터 분기 처리 로직이 많아 디버깅할 때 이를 육안으로 확인하는 번거로움이 예상됐다.

```javascript
$.ajax({
  ...
  success : function(result) {
      let salaryTotal;
      for(해당 월급 타입의 개수) {
        salaryTotal += 월급;
      }
      
      html += "<tr>" + salaryTotal + "</tr>";
  }
});
```

번거로움을 줄이고 이전과 같은 에러도 방지하고자 MyBatis 설정 XML 파일에 이미 설정되어 있던 `settings` 엘리먼트의 `mapUnderscoreToCamelCase`속성을 활용하기로 했다.

```xml
<settings>
    <setting name="mapUnderscoreToCamelCase" value="true"/>
</settings>
```

이 설정을 활용하여 프론트(JSP) 단에 있던 **for문을 다시 제거**한 뒤 **VO에는 `_total`이 붙은 필드 이름을 카멜케이스로 바꿔** 마무리했다.

기존 4천 줄이던 쿼리문을 30%를 줄여 2천 8백 줄로 한결 깔끔하게 보도록 수정을 완료했고, 번거로운 노가다 작업도 한 줌 덜게 되었다.

---

## MyBatis의 매핑 원리와 세팅 옵션 이해하기


### MyBatis 매핑 원리

#### 매핑 구문에 리절트 타입 속성을 지정하면?

![매핑 구문에 리절트 타입 속성을 지정하면 이렇게 됩니다](mappingPrinciple.png)

#### 반환된 조회 결과는 Reuslt 객체에 어떻게 바인딩 될까?

1. ResultSet Handler가 resultType 속성 값에 지정한 result Type의 객체를 생성한 다음, ResultSet에 담긴 조회 결과를 바인딩합니다.
   - 이 과정에서 **오브젝트 팩토리와 타입 핸들러, 프로퍼티** 등이 함께 사용됩니다.
   - org.apache.ibatis.executor.resultset.DefaultResultSetHandler 클래스를 살펴보면
     - 매핑 구문을 실행한 다음 반환된 ResultSet에 결과가 존재하면, 오브젝트 팩토리는 resultType 속성 값에 지정한 Result 객체를 생성한 다음 초기화합니다.
     - 그리고 Property와 TypeHandler를 통해서 Result 객체에 결과 값을 바인딩합니다.

2. ResultSet Handler는 컬럼명과 프로퍼티명(자바빈즈)이 정해진 기준에 맞으면 프로퍼티 값에 컬럼 값을 바인딩합니다.

3. 첫 번째 객체 타입 중 자바빈즈(클래스)에 ResultSet이 바인딩되는 과정을 알아보자면
   - 조회한 컬럼명과 자바빈즈에 정의한 프로퍼티명을 각각 대문자로 변경한 다음 서로 비교했을 때 일치하면, 프로퍼티 값에 컬럼 값을 바인딩합니다.
   - 이때, 자바빈즈 명세에 맞는 setter메서드가 정의되어 있지 않아도 Reflection을 통해서 프로퍼티에 컬럼 값을 바인딩합니다.
     - 예를 들어, `BONUSITEM_TOTAL` 문자열이면, 컬럼명을 대문자로 변경한 문자열은 `BONUSITEM_TOTAL`이 됩니다.
     - 위에서 설명한 바인딩 기준에 따라 자바빈즈에 정의한 프로퍼티명을 대문자로 변경했을 때, `BONUSITEM_TOTAL`과 **동일한 프로퍼티명**을 찾습니다.
     - 대문자로 변경했을 때, `BONUSITEM_TOTAL` 문자열과 일치하는 `bonusitem_total` 프로퍼티명을 찾게 되면, 프로퍼티 값에 컬럼 값을 바인딩합니다.


**프로퍼티명을 통상 CamelCase로 하니**, 이런 상황에서 바인딩을 하려면 MyBatis 설정 XML 파일의 `<settings>` 구성 요소에 `mapUnderscoreToCamelCase` 속성을 추가하고서 속성값에 `true`를 넣으면 프로퍼티명이 CamelCase여도 문제없이 바인딩이 가능합니다.




### 위의 상황에서 해줄 수 있는 매핑 세팅

데이터베이스에서 데이터를 가져온 후 자바 객체에 값을 넣기 위해 MyBatis는 `결과 매핑`이라는 기법을 제공합니다. 그 `결과 매핑`을 처리하기 위해 매핑 구문을 정의할 때처럼 XML 엘리먼트나 애노테이션을 사용할 수 있습니다.

마이바티스의 결과 매핑은 **칼럼명과 자바 모델(클래스)의 필드명** 혹은 **setter 메서드가 설정하고자 하는 값**과 일치하면 자동으로 값을 넣어줍니다.

하지만 칼럼명과 일치하지 않는다면 별도로 값을 설정하는 규칙을 정의해야만 정확히 값을 설정할 수 있습니다.  
이렇게 일치하지 않는 경우에 제가 겪었던 상황에서 활용할 수 있는, 값을 설정해주기 위해 제공하는 XML 엘리먼트와 속성은 다음과 같습니다.

- resultMap : 결과 매핑을 사용하기 위한 가장 상위 엘리먼트
  - 결과 매핑을 구분하기 위한 id 속성과, 매핑하는 대상 클래스를 정의하는 type 속성을 사용
  - 예) 
      ```xml
      <resultMap id="EmpSalaryMap" type="EmpSalaryVO">
      ...
      </resultMap>
      ```
  - constructor : setter 메서드나 리플렉션을 통해 값을 설정하지 않고 **생성자를 통해 값을 설정할 때** 사용
    - idArg : ID 인자
    - arg : 생성자에 삽입되는 일반적인 결과


사원 연봉 예시를 다시 들고 오겠습니다

```xml
<resultMap id="EmpSalaryMap" type="EmpSalaryVO">
    <result column="BONUSITEM1" property="bonusitem1">
    <result column="BONUSITEM2" property="bonusitem2">
    <result column="BONUSITEM3" property="bonusitem3">
    <result column="BONUSITEM4" property="bonusitem4">
    <result column="BONUSITEM5" property="bonusitem5">
    <result column="BONUSITEM6" property="bonusitem6">
    <result column="BONUSITEM7" property="bonusitem7">
    <result column="BONUSITEM8" property="bonusitem8">
    <result column="BONUSITEM9" property="bonusitem9">
    <result column="BONUSITEM10" property="bonusitem10">
    <result column="BONUSITEM11" property="bonusitem11">
    <result column="BONUSITEM_TOTAL" property="bonusitem_total">
</resultMap>

<select id="selectEmployeeSalary" parameterType="HashMap" resultMap="EmpSalaryMap">
    select
        A.ba1 AS BONUSITEM1,
        A.ba2 AS BONUSITEM2,
        A.ba3 AS BONUSITEM3,
        A.bb1 AS BONUSITEM4,
        A.bb2 AS BONUSITEM5,
        A.bb3 AS BONUSITEM6,
        A.bc1 AS BONUSITEM7,
        A.bc2 AS BONUSITEM8,
        A.bc3 AS BONUSITEM9,
        B.bd1 AS BONUSITEM10,
        B.dd2 AS BONUSITEM11,
        (A.sum + B.sum) AS BONUSITEM_TOTAL
    from emp_salarya A, emp_salaryb B
    where id=#{id}
</select>
```

상여금 합계에 해당하는 `bonusitem_total` 칼럼의 값을 객체에 자동으로 설정하기 위해서는 **대상 클래스가 bonusitem_total 이름의 필드**를 갖거나 **setBonusitem_total 이름의 setter 메서드**를 가져야만 합니다.

만약, bonusitem_total 칼럼의 값을 낙타표기법으로 선언된 필드로 받고 싶다면 **bonusitemTotal 필드**나 **setBonusitemTotal 이름의 setter 메서드**를 사용해야 합니다.

```xml
<resultMap id="EmpSalaryMap" type="EmpSalaryVO">
    ... 
    <result column="bonusitem_total" property="bonusitemTotal" />
</resultMap>
```

위의 resultMap은 **SQL에서 칼럼 값을 객체에 설정하는 결과 매핑을 설정**할 수 있습니다.  
결과 매핑을 정의하는 resultMap 엘리먼트 하위에 둔 각각의 엘리먼트에서 대상 **칼럼명은 column 속성**에 선언하고, 클래스에서 설정하고자 하는 **필드명은 property 속성**에 선언하면 됩니다.

이렇게 **resultMap 엘리먼트**에 칼럼과 대상 필드 및 setter 메서드의 규칙을 정의해주면 **칼럼명과 필드명이 다르더라도 값을 설정할 수 있습니다.**

열심히 resultMap에 대해 설명을 드렸지만, 사실 resultMap을 안 써도 코드의 양을 줄이면서 충분히 매핑할 수 있는 방법이 있습니다.  
MyBatis 설정 파일에서 `mapUnserscoretoCamelCase` 설정을 활용하면 됩니다.

`결과 매핑`을 사용하지 않는다고 가정하고 각 컬럼과 setter 메서드를 사용하면 다음과 같습니다.
- bonusitem_total -> setBonusitem_total()

마이바티스 설정 파일의 `mapUnderscoreToCamelCase` 설정은 기본값이 false이며, true로 설정하면 다음과 같은 규칙이 자동으로 적용됩니다.

- bonusitem_total -> setBonusitemTotal()


