# 코드 리팩토링을 위한 삽질기, 삽질하면서 알아본 MyBatis의 매핑 원리와 세팅 옵션 이해하기

## 개요

Spring, MyBatis 스택의 실무 환경에서 resultMap 으로 반환타입을 묶은 4천 가량의 조회 쿼리를 40% 가량 줄이면서 겪었던 삽질(?)과 해결에 도움이 됐던 MyBatis의 매핑 원리와 활용 방안을 공유드리겠습니다.

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


### Alias로 퉁쳐도 될 것 같은데 굳이 resultMap을 쓰고 있는 select문

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

이렇게 조회한 데이터는 프론트 단에서 각 급여 종류의 값을 출력하고, 합은 마지막에 _total이라는 글자를 붙여 값을 가져와 뿌려줍니다.

```javascript
$.ajax({
  ...
  success : function(result) {
    for(var i = 0; i < size; i++) {
      html += "<tr>" + result[0][type + i] + "</tr>";
    }
    html += "<tr>" + result[0][type + '_total'] + "</tr>";
  }
});
```

## 해결 과정

위에서 설명한 소스는 사원의 급여에 대해 조회해오는데 필요한 소스입니다.
만약, 급여의 종류가 다양해진다면 그에 따라 DB 테이블에 컬럼을 추가하고 데이터를 넣는 등의 작업이 요구됩니다.
위의 급여 예시를 들었지만, 차를 판매한다고 했을 때 그 차의 종류, 옵션에 따라 금액이 다를 것이고, 한 두 가지가 아니기에 관리해야 하는 종류의 수는 많아질 겁니다.

실무에서도 이와 비슷하게 출력할 컬럼과 필드의 수가 많아져서, 기존에 Alias로 바꾸고 resultMap으로 매핑하던 방식을 Alias만 사용하도록 수정할 필요가 있었습니다.
우선 resultMap을 제거하고 `<select>`문의 resultType으로 VO와 직접 연결했습니다. 값들이 map 없이도 잘 나오는 걸 확인한 후에 코드를 반영했지만, 총합을 나타내는 `_total`은 매핑이 되지 않고 있었습니다.

운영에서 발생했을 때는 서버를 일과시간에 중단할 수 없어 프론트(jsp)단에서 해결하기로 했습니다.  
프론트 단에서 `total` 값을 직접 출력하지 않고, for문으로 각 값을 합쳐 출력하는 방식으로 급하게 반영을 마무리 했습니다.

핫픽스 후 이상은 없었지만, 프론트 단에는 이전부터 분기 처리 로직이 많아 디버깅할 때 이를 육안으로 확인하는 번거로움이 예상됐습니다.
번거로움을 줄이고 이전과 같은 에러도 방지하고자 config.xml에 이미 설정되어 있던 `settings` 엘리먼트의 `mapUnderscoreToCamelCase`속성을 활용하기로 했습니다.

```xml
<settings>
    <setting name="mapUnderscoreToCamelCase" value="true"/>
</settings>
```

설정값을 true로 바꾸면 언더스코어로 반환되는 컬럼 이름은 카멜케이스로 매핑하게끔 매핑됩니다.  
이 설정을 활용하여 개발 서버에는 프론트 단에 임시로 처리했던 for문을 다시 제거하고, VO에는 `_total`이 붙은 필드를 카멜케이스로 바꿔 리팩토링을 마쳤습니다.

기존 4천 줄이던 쿼리문을 30%를 줄여 2천 8백 줄로 한결 깔끔하게 보도록 수정을 완료했습니다.
번거로운 노가다 작업도 한 줌 덜게 되었습니다.

---

## MyBatis의 매핑 원리와 세팅 옵션 이해하기

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