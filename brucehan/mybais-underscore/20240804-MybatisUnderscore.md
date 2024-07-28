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

