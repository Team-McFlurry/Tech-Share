# 코드 리팩토링을 위한 삽질기, 삽질하면서 알아본 MyBatis의 매핑 원리와 세팅 옵션 이해하기

## 개요

Spring, MyBatis 스택의 실무 환경에서 resultMap 으로 반환타입을 묶은 4천 가량의 조회 쿼리를 40% 가량 줄이면서 겪었던 삽질(?)과 

## 기존에 있던 VO와 xml에 있던 resultMap, select문

### 필드와 getter메서드만 있는 VO 클래스

```java
public class EmpSalaryVO {
    private String performanceitem1;
    private String performanceitem2;
    private String performanceitem3;
    private String performanceitem4;
    private String performanceitem5;
    private String performanceitem6;
    private String performanceitem7;
    private String performanceitem8;
    private String performanceitem9;
    private String performanceitem10;
    private String performanceitem11;
    private String performanceitem_total;

    private String basicitem1;
    private String basicitem2;
    private String basicitem3;
    private String basicitem4;
    private String basicitem5;
    private String basicitem6;
    private String basicitem7;
    private String basicitem8;
    private String basicitem9;
    private String basicitem10;
    private String basicitem11;
    private String basicitem_total;

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


    public String getPerformanceItem1() {
        return performanceitem1;
    }
    public String getPerformanceItem1() {
        return performanceitem1;
    }
    public String getPerformanceItem2() {
        return performanceitem2;
    }
    public String getPerformanceItem3() {
        return performanceitem3;
    }
    public String getPerformanceItem4() {
        return performanceitem4;
    }
    public String getPerformanceItem5() {
        return performanceitem5;
    }
    public String getPerformanceItem6() {
        return performanceitem6;
    }
    public String getPerformanceItem7() {
        return performanceitem7;
    }
    public String getPerformanceItem8() {
        return performanceitem8;
    }
    public String getPerformanceItem9() {
        return performanceitem9;
    }
    public String getPerformanceItem10() {
        return performanceitem10;
    }
    public String getPerformanceItem11() {
        return performanceitem11;
    }
    public String getPerformanceItem_total() {
        return performanceitem_total;
    }

    
    public String getBasicItem1() {
        return basicitem1;
    }
    public String getBasicItem2() {
        return basicitem2;
    }
    public String getBasicItem3() {
        return basicitem3;
    }
    public String getBasicItem4() {
        return basicitem4;
    }
    public String getBasicItem5() {
        return basicitem5;
    }
    public String getBasicItem6() {
        return basicitem6;
    }
    public String getBasicItem7() {
        return basicitem7;
    }
    public String getBasicItem8() {
        return basicitem8;
    }
    public String getBasicItem9() {
        return basicitem9;
    }
    public String getBasicItem10() {
        return basicitem10;
    }
    public String getBasicItem11() {
        return basicitem11;
    }
    public String getBasicItem_total() {
        return basicitem_total;
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
    public String getBonusItem7() {
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
    public String getBonusItem_sum() {
        return bonusitem_sum;
    }
}
```


### Alias로 퉁쳐도 될 것 같은데 굳이 resultMap을 쓰고 있는 select문

```xml
...

<resultMap id="EmpSalaryMap" type="EmpSalaryVO">
    <result column="PERFORMANCEBONUSITEM1" property="performancebonusitem1">
    <result column="PERFORMANCEBONUSITEM2" property="performancebonusitem2">
    <result column="PERFORMANCEBONUSITEM3" property="performancebonusitem3">
    <result column="PERFORMANCEBONUSITEM4" property="performancebonusitem4">
    <result column="PERFORMANCEBONUSITEM5" property="performancebonusitem5">
    <result column="PERFORMANCEBONUSITEM6" property="performancebonusitem6">
    <result column="PERFORMANCEBONUSITEM7" property="performancebonusitem7">
    <result column="PERFORMANCEBONUSITEM8" property="performancebonusitem8">
    <result column="PERFORMANCEBONUSITEM9" property="performancebonusitem9">
    <result column="PERFORMANCEBONUSITEM10" property="performancebonusitem10">
    <result column="PERFORMANCEBONUSITEM11" property="performancebonusitem11">
    <result column="PERFORMANCEBONUSITEM_TOTAL" property="performancebonusitem_total">
    
    <result column="BASICITEM1" property="basicitem1">
    <result column="BASICITEM2" property="basicitem2">
    <result column="BASICITEM3" property="basicitem3">
    <result column="BASICITEM4" property="basicitem4">
    <result column="BASICITEM5" property="basicitem5">
    <result column="BASICITEM6" property="basicitem6">
    <result column="BASICITEM7" property="basicitem7">
    <result column="BASICITEM8" property="basicitem8">
    <result column="BASICITEM9" property="basicitem9">
    <result column="BASICITEM10" property="basicitem10">
    <result column="BASICITEM11" property="basicitem11">
    <result column="BASICITEM_TOTAL" property="basicitem_total">
    
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
        A.pia1 AS PERFORMANCEBONUSITEM1,
        A.pia2 AS PERFORMANCEBONUSITEM2,
        A.pia3 AS PERFORMANCEBONUSITEM3,
        A.pib1 AS PERFORMANCEBONUSITEM4,
        A.pib2 AS PERFORMANCEBONUSITEM5,
        A.pib3 AS PERFORMANCEBONUSITEM6,
        A.pic1 AS PERFORMANCEBONUSITEM7,
        A.pic2 AS PERFORMANCEBONUSITEM8,
        A.pic3 AS PERFORMANCEBONUSITEM9,
        B.pid1 AS PERFORMANCEBONUSITEM10,
        B.pid2 AS PERFORMANCEBONUSITEM11,
        (A.sum + B.sum) AS PERFORMANCEBONUSITEM_TOTAL
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


## MyBatis의 매핑 원리와 세팅 옵션 이해하기

데이터베이스에서 데이터를 가져온 후 자바 객체에 값을 넣기 위해 MyBatis는 `결과 매핑`이라는 기법을 제공합니다. 그 `결과 매핑`을 처리하기 위해 매핑 구문을 정의할 때처럼 XML 엘리먼트나 애노테이션을 사용할 수 있습니다.

마이바티스의 결과 매핑은 **칼럼명과 자바 모델(클래스)의 필드명** 혹은 **setter 메서드가 설정하고자 하는 값**과 일치하면 자동으로 값을 넣어줍니다.

하지만 칼럼명과 일치하지 않는다면 별도로 값을 설정하는 규칙을 정의해야만 정확히 값을 설정할 수 있습니다.  
이렇게 일치하지 않는 경우에 제가 겪었던 상황에서 활용할 수 있는, 값을 설정해주기 위해 제공하는 XML 엘리먼트는 다음과 같습니다.

- resultMap : 결과 매핑을 사용하기 위한 가장 상위 엘리먼트
  - 결과 매핑을 구분하기 위한 id 속성과, 매핑하는 대상 클래스를 정의하는 type 속성을 사용
  - 예) 
      ```xml
      <resultMap id="EmpSalaryMap" type="EmpSalaryVO">
      ...
      </resultMap>
      ```
  - id : 기본 키에 해당되는 값을 설정
  - result : 기본 키가 아닌 나머지 칼럼에 대해 매핑
  - constructor : setter 메서드나 리플렉션을 통해 값을 설정하지 않고 **생성자를 통해 값을 설정할 때** 사용
    - idArg : ID 인자
    - arg : 생성자에 삽입되는 일반적인 결과
  



한 경우는
- 결과 타입을 지정할 때 resultType 속성을 사용했고
- 예시 매핑 구문에서 각 칼럼에 대해 Alias를 사용했다.
다른 경우는
- 결과 타입을 지정할 때 resultMap을 사용했지만
- 여기 예시에서는 별칭을 사용하지 않았다.

댓글 번호에 해당하는 comment_no 칼럼의 값을 모델 객체에 자동으로 설정하기 위해서는 대상 모델 클래스가 comment_no 이름의 필드를 갖거나 setComment_no 이름의 setter 메서드를 가져야만 한다.
하지만 여기서 칼럼과 모델 클래스의 필드와 메서드는 이름이 다르기 때문에 자동으로 값을 설정할 수 없다.

즉, comment_no 칼럼의 값을 commentNo 필드나 setCommentNo 이름의 setter 메서드를 사용하게 규칙을 정의해줘야 하는 상황이 되는 셈이다.

그렇다면 comment_no 칼럼의 값을 commmentNo 필드에 설정하기 위한 결과 매핑을 설정해보자.

```xml
<resultMap id="BaseResultMap" type="ldg.mybatis.model.Comment">
    <id column="comment_no" jdbcType="BIGINT" property="commentNo" />
    <result column="user_id" jdbcType="VARCHAR" property="userId" />
</resultMap>
```

위의 resultMap은 댓글을 조회하는 SQL에서 별칭을 사용하지 않는 칼럼 값을 모델 객체에 설정하는 결과 매핑 설정이다.
결과 매핑을 정의하는 resultMap 엘리먼트 하위에 둔 각각의 엘리먼트에서 대상 칼럼명은 column 속성에 선언하고, 모델 클래스에서 설정하고자 하는 필드명은 property 속성에 선언하면 된다.

- 결과 매핑의 아이디는 BaseResultMap이고, resultMap 속성에서 사용한다. 대상 클래스의 타입은 ldg.mybatis.model.Comment다. 이 결과 매핑을 사용해 반환되는 타입은 댓글 또는 댓글을 갖는 List 객체다.
- id 엘리먼트는 기본 키의 칼럼을 지정할 때 사용한다. 댓클 테이블의 기본 키는 댓글 번호인 comment_no 칼럼이다.
- comment_no 칼럼은 property 속성을 commentNo로 했기 때문에 댓글 객체의 commentNo 필드에 리플렉션으로 값을 설정하거나 setCommentNo 메서드를 사용해서 값을 설정한다.
- comment_no 칼럼에서 jdbcType 속성에 정의한 bigint 값을 보고 JDBC가 내부적으로 값을 처리할 때 값의 타입을 bigint로 인식한다. JDBC 타입의 종류는 java.sql.Types 클래스를 보면 된다.
- result 엘리먼트를 사용한 user_id 칼럼은 property 속성을 userId로 설정했기 때문에 댓글 객체의 userId 필드에 리플렉션으로 ㄱ밧을 설정하거나 setUserId 메서드를 사용해서 값을 설정한다.

이렇게 resultMap 엘리먼트에 칼럼과 대상 필드 및 setter 메서드의 규칙을 정의해주면 칼럼명과 필드명이 다르더라도 값을 설정할 수 있다.

이번 경우에는 칼럼명과 필드명의 명명 규칙 차이가 일관적이었다. 칼럼명은 전통적인 데이터베이스가 선택하는 _(언더 바) 방식을 사용했고, 모델 클래스는 자바가 사용하는 낙타표기법을 사용했다.

마이바티스에서는 이러한 전통적인 명명 규칙 간의 차이를 쉽게 해결하기 위해 별도로 mapUnserscoretoCamelCase 설정을 제공한다. 다시 결과 매핑을 사용하지 않는다고 가정해보자.

각 컬럼과 setter 메서드를 생각해보면 다음고 ㅏ같다.
- comment_no -> setComment_no()
- user_id -> setUser_id()

마이바티스 설정 파일의 mapUnderscoreToCamelCase 설정을 true로 설정하면 마이바티스는 다음과 같은 규칙을 자동으로 적용한다.

- comment_no -> setCommentNo()
- user_id -> setUserId()

이렇ㅔㄱ 할 수 있는 배경은 전통적인 데이터베이스의 명명 규칙과 자바의 낙타표기법이 오랫동안 지속돼 왔고 일관적이라는 점에서 가능한 기능이다. 
결과 매핑을 별도로 XML에 정의하는 것은 매핑 구문이 많아질수록 귀찮은 작업이 될 수 있다.  
대부분의 경우 mapUnderscoreToCamelCase 설정을 사용해서 많은 부분을 자동으로 처리할 수 있다.  
하지만 mapUnderscoreToCamelCase 설정을 사용할 때 별칭이 너무 길면 오류가 발생할 수 있다고도 한다.

mapUnderscoreToCamelCase 설정을 프로젝트 전반에 사용할지는 약간의 테스트를 진행해서 판단하는 게 좋다.


`mapUnderscoreToCamelCase` 옵션은 

> 명명 규칙의 차이점이 있기 때문에 자동으로 매핑할 때는 대상을 찾기 어렵다. 하지만 데이터베이스와 자바는 사용하는 명명 규칙이 명확한 편이기 때문에 일정한 규칙을 부여하면 값을 매핑하는 데 어렵지 않다.
> 이런 경우를 위해 언더바 형태를 낙타표현식으로 자동매핑할지에 대한 옵션이다. 이 옵션을 사용하지 않으면서 테이블의 칼럼명은 언더 바로 구분하고 자바 모델 클래스는 낙타 표기법을 사용할 경우 쿼리문에 칼럼별로 별칭을 사용하거나 별도의 결과 매핑을 사용해야 한다. 디폴트는 false이고 자동으로 매핑하지 않는다.

> settings 엘리먼트를 사용해서 설정하는 각종 값은 SqlSesstionFacotry 객체가 SqlSession 객체를 만들 때 생성할 객체의 특성을 결정한다.
> settings 엘리먼트의 하위 엘리먼트들은 대부분 디폴트 값을 가진다.
