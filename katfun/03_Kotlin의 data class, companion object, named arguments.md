# Kotlin의 data class, companion object, named arguments

## 개요

2022년 12월에 카카오페이에 합류한 이래로, 업무 중 사용하는 프로그래밍 언어의 99.999% 이상을 Kotlin이 차지하고 있습니다. Kotlin에서 매일같이 사용하는 기능들 중에서, Kotlin을 사용하지 않는 분들께 소개해 볼 만한 주제를 몇 개 꼽아 보았습니다.

## Data Class

### 정의

[Data class](https://kotlinlang.org/docs/data-classes.html#properties-declared-in-the-class-body)는 코틀린 내에서 'data를 가지고 있기 위한' 클래스로서 사용합니다. 저의 경우에는 흔히 사용하는 dto, dao, ... 등을 모두 data class로 사용합니다. 조금 구체적으로는,

* POST 메서드의 API가 request body를 받을 때,
* API가 response body를 리턴할 때,
* (JPA를 사용하는 경우) entity와 양방향으로 데이터를 변환할 때,
* VO를 만들 때 (항상 이렇지는 않음)

등등 다양하게 사용합니다.

data class를 사용하는 가장 큰 이유는, 컴파일러가 데이터를 나타내기 위해 필요한 멤버 함수를 자동으로 만들어 (재정의해) 줍니다.

* `.equals()`, `.hashCode()` 메서드
* `.toString()`
* `.componentN()`
* `.copy()`

자세한 내용은 예제 코드를 참고하세요.

### Getting Hands Dirty

Effective Java Item 11에서는 "equals를 재정의하려거든 hashcode도 재정의하라"고 합니다 (물론 저는 책 보다 말았습니다). 위에서 언급한 바와 같이, data class를 생성 시에는 두 메서드를 자동으로 만들어 (재정의해) 줍니다. 

예제 코드를 decompile 시킨 것을 까 봅시다.

```java
// User.java
package com.katfun.tech.share.sample.codes.week03;

import kotlin.Metadata;
import kotlin.jvm.internal.Intrinsics;
import org.jetbrains.annotations.NotNull;
import org.jetbrains.annotations.Nullable;

@Metadata(
   mv = {1, 9, 0},
   k = 1,
   xi = 48,
   d1 = {"\u0000,\n\u0002\u0018\u0002\n\u0002\u0010\u0000\n\u0000\n\u0002\u0010\t\n\u0000\n\u0002\u0010\u000e\n\u0000\n\u0002\u0010\b\n\u0000\n\u0002\u0018\u0002\n\u0002\b\u000f\n\u0002\u0010\u000b\n\u0002\b\u0004\b\u0080\b\u0018\u00002\u00020\u0001B%\u0012\u0006\u0010\u0002\u001a\u00020\u0003\u0012\u0006\u0010\u0004\u001a\u00020\u0005\u0012\u0006\u0010\u0006\u001a\u00020\u0007\u0012\u0006\u0010\b\u001a\u00020\t¢\u0006\u0002\u0010\nJ\t\u0010\u0013\u001a\u00020\u0003HÆ\u0003J\t\u0010\u0014\u001a\u00020\u0005HÆ\u0003J\t\u0010\u0015\u001a\u00020\u0007HÆ\u0003J\t\u0010\u0016\u001a\u00020\tHÆ\u0003J1\u0010\u0017\u001a\u00020\u00002\b\b\u0002\u0010\u0002\u001a\u00020\u00032\b\b\u0002\u0010\u0004\u001a\u00020\u00052\b\b\u0002\u0010\u0006\u001a\u00020\u00072\b\b\u0002\u0010\b\u001a\u00020\tHÆ\u0001J\u0013\u0010\u0018\u001a\u00020\u00192\b\u0010\u001a\u001a\u0004\u0018\u00010\u0001HÖ\u0003J\t\u0010\u001b\u001a\u00020\u0007HÖ\u0001J\t\u0010\u001c\u001a\u00020\u0005HÖ\u0001R\u0011\u0010\b\u001a\u00020\t¢\u0006\b\n\u0000\u001a\u0004\b\u000b\u0010\fR\u0011\u0010\u0006\u001a\u00020\u0007¢\u0006\b\n\u0000\u001a\u0004\b\r\u0010\u000eR\u0011\u0010\u0002\u001a\u00020\u0003¢\u0006\b\n\u0000\u001a\u0004\b\u000f\u0010\u0010R\u0011\u0010\u0004\u001a\u00020\u0005¢\u0006\b\n\u0000\u001a\u0004\b\u0011\u0010\u0012¨\u0006\u001d"},
   d2 = {"Lcom/katfun/tech/share/sample/codes/week03/User;", "", "id", "", "name", "", "age", "", "address", "Lcom/katfun/tech/share/sample/codes/week03/Address;", "(JLjava/lang/String;ILcom/katfun/tech/share/sample/codes/week03/Address;)V", "getAddress", "()Lcom/katfun/tech/share/sample/codes/week03/Address;", "getAge", "()I", "getId", "()J", "getName", "()Ljava/lang/String;", "component1", "component2", "component3", "component4", "copy", "equals", "", "other", "hashCode", "toString", "Tech-Share-Sample-Codes_test"}
)
public final class User {
  // primitive types 외에는 @NotNull이 박혀 있다.
   private final long id;
   @NotNull
   private final String name;
   private final int age;
   @NotNull
   private final Address address;

  // constructor
   public User(long id, @NotNull String name, int age, @NotNull Address address) {
      Intrinsics.checkNotNullParameter(name, "name");
      Intrinsics.checkNotNullParameter(address, "address");
      super();
      this.id = id;
      this.name = name;
      this.age = age;
      this.address = address;
   }

  // getters
   public final long getId() {
      return this.id;
   }

   @NotNull
   public final String getName() {
      return this.name;
   }

   public final int getAge() {
      return this.age;
   }

   @NotNull
   public final Address getAddress() {
      return this.address;
   }

  // there's no setters
  
  // components
   public final long component1() {
      return this.id;
   }

   @NotNull
   public final String component2() {
      return this.name;
   }

   public final int component3() {
      return this.age;
   }

   @NotNull
   public final Address component4() {
      return this.address;
   }

  // copy
   @NotNull
   public final User copy(long id, @NotNull String name, int age, @NotNull Address address) {
      Intrinsics.checkNotNullParameter(name, "name");
      Intrinsics.checkNotNullParameter(address, "address");
      return new User(id, name, age, address);
   }

   // $FF: synthetic method
   public static User copy$default(User var0, long var1, String var3, int var4, Address var5, int var6, Object var7) {
      if ((var6 & 1) != 0) {
         var1 = var0.id;
      }

      if ((var6 & 2) != 0) {
         var3 = var0.name;
      }

      if ((var6 & 4) != 0) {
         var4 = var0.age;
      }

      if ((var6 & 8) != 0) {
         var5 = var0.address;
      }

      return var0.copy(var1, var3, var4, var5);
   }

  // toString
   @NotNull
   public String toString() {
      return "User(id=" + this.id + ", name=" + this.name + ", age=" + this.age + ", address=" + this.address + ')';
   }

  // hashCode and equals
   public int hashCode() {
      int result = Long.hashCode(this.id);
      result = result * 31 + this.name.hashCode();
      result = result * 31 + Integer.hashCode(this.age);
      result = result * 31 + this.address.hashCode();
      return result;
   }

   public boolean equals(@Nullable Object other) {
      if (this == other) {
         return true;
      } else if (!(other instanceof User)) {
         return false;
      } else {
         User var2 = (User)other;
         if (this.id != var2.id) {
            return false;
         } else if (!Intrinsics.areEqual(this.name, var2.name)) {
            return false;
         } else if (this.age != var2.age) {
            return false;
         } else {
            return Intrinsics.areEqual(this.address, var2.address);
         }
      }
   }
}
// Address.java
package com.katfun.tech.share.sample.codes.week03;

import kotlin.Metadata;
import kotlin.jvm.internal.Intrinsics;
import org.jetbrains.annotations.NotNull;
import org.jetbrains.annotations.Nullable;

@Metadata(
   mv = {1, 9, 0},
   k = 1,
   xi = 48,
   d1 = {"\u0000\"\n\u0002\u0018\u0002\n\u0002\u0010\u0000\n\u0000\n\u0002\u0010\u000e\n\u0002\b\u0004\n\u0002\u0010\b\n\u0002\b\u000f\n\u0002\u0010\u000b\n\u0002\b\u0004\b\u0080\b\u0018\u00002\u00020\u0001B-\u0012\u0006\u0010\u0002\u001a\u00020\u0003\u0012\u0006\u0010\u0004\u001a\u00020\u0003\u0012\u0006\u0010\u0005\u001a\u00020\u0003\u0012\u0006\u0010\u0006\u001a\u00020\u0003\u0012\u0006\u0010\u0007\u001a\u00020\b¢\u0006\u0002\u0010\tJ\t\u0010\u0011\u001a\u00020\u0003HÆ\u0003J\t\u0010\u0012\u001a\u00020\u0003HÆ\u0003J\t\u0010\u0013\u001a\u00020\u0003HÆ\u0003J\t\u0010\u0014\u001a\u00020\u0003HÆ\u0003J\t\u0010\u0015\u001a\u00020\bHÆ\u0003J;\u0010\u0016\u001a\u00020\u00002\b\b\u0002\u0010\u0002\u001a\u00020\u00032\b\b\u0002\u0010\u0004\u001a\u00020\u00032\b\b\u0002\u0010\u0005\u001a\u00020\u00032\b\b\u0002\u0010\u0006\u001a\u00020\u00032\b\b\u0002\u0010\u0007\u001a\u00020\bHÆ\u0001J\u0013\u0010\u0017\u001a\u00020\u00182\b\u0010\u0019\u001a\u0004\u0018\u00010\u0001HÖ\u0003J\t\u0010\u001a\u001a\u00020\bHÖ\u0001J\t\u0010\u001b\u001a\u00020\u0003HÖ\u0001R\u0011\u0010\u0004\u001a\u00020\u0003¢\u0006\b\n\u0000\u001a\u0004\b\n\u0010\u000bR\u0011\u0010\u0002\u001a\u00020\u0003¢\u0006\b\n\u0000\u001a\u0004\b\f\u0010\u000bR\u0011\u0010\u0006\u001a\u00020\u0003¢\u0006\b\n\u0000\u001a\u0004\b\r\u0010\u000bR\u0011\u0010\u0007\u001a\u00020\b¢\u0006\b\n\u0000\u001a\u0004\b\u000e\u0010\u000fR\u0011\u0010\u0005\u001a\u00020\u0003¢\u0006\b\n\u0000\u001a\u0004\b\u0010\u0010\u000b¨\u0006\u001c"},
   d2 = {"Lcom/katfun/tech/share/sample/codes/week03/Address;", "", "country", "", "city", "street", "detail", "postalNumber", "", "(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;I)V", "getCity", "()Ljava/lang/String;", "getCountry", "getDetail", "getPostalNumber", "()I", "getStreet", "component1", "component2", "component3", "component4", "component5", "copy", "equals", "", "other", "hashCode", "toString", "Tech-Share-Sample-Codes_test"}
)
public final class Address {
   @NotNull
   private final String country;
   @NotNull
   private final String city;
   @NotNull
   private final String street;
   @NotNull
   private final String detail;
   private final int postalNumber;

   public Address(@NotNull String country, @NotNull String city, @NotNull String street, @NotNull String detail, int postalNumber) {
      Intrinsics.checkNotNullParameter(country, "country");
      Intrinsics.checkNotNullParameter(city, "city");
      Intrinsics.checkNotNullParameter(street, "street");
      Intrinsics.checkNotNullParameter(detail, "detail");
      super();
      this.country = country;
      this.city = city;
      this.street = street;
      this.detail = detail;
      this.postalNumber = postalNumber;
   }

   @NotNull
   public final String getCountry() {
      return this.country;
   }

   @NotNull
   public final String getCity() {
      return this.city;
   }

   @NotNull
   public final String getStreet() {
      return this.street;
   }

   @NotNull
   public final String getDetail() {
      return this.detail;
   }

   public final int getPostalNumber() {
      return this.postalNumber;
   }

   @NotNull
   public final String component1() {
      return this.country;
   }

   @NotNull
   public final String component2() {
      return this.city;
   }

   @NotNull
   public final String component3() {
      return this.street;
   }

   @NotNull
   public final String component4() {
      return this.detail;
   }

   public final int component5() {
      return this.postalNumber;
   }

   @NotNull
   public final Address copy(@NotNull String country, @NotNull String city, @NotNull String street, @NotNull String detail, int postalNumber) {
      Intrinsics.checkNotNullParameter(country, "country");
      Intrinsics.checkNotNullParameter(city, "city");
      Intrinsics.checkNotNullParameter(street, "street");
      Intrinsics.checkNotNullParameter(detail, "detail");
      return new Address(country, city, street, detail, postalNumber);
   }

   // $FF: synthetic method
   public static Address copy$default(Address var0, String var1, String var2, String var3, String var4, int var5, int var6, Object var7) {
      if ((var6 & 1) != 0) {
         var1 = var0.country;
      }

      if ((var6 & 2) != 0) {
         var2 = var0.city;
      }

      if ((var6 & 4) != 0) {
         var3 = var0.street;
      }

      if ((var6 & 8) != 0) {
         var4 = var0.detail;
      }

      if ((var6 & 16) != 0) {
         var5 = var0.postalNumber;
      }

      return var0.copy(var1, var2, var3, var4, var5);
   }

   @NotNull
   public String toString() {
      return "Address(country=" + this.country + ", city=" + this.city + ", street=" + this.street + ", detail=" + this.detail + ", postalNumber=" + this.postalNumber + ')';
   }

  // hashCode and equals
   public int hashCode() {
      int result = this.country.hashCode();
      result = result * 31 + this.city.hashCode();
      result = result * 31 + this.street.hashCode();
      result = result * 31 + this.detail.hashCode();
      result = result * 31 + Integer.hashCode(this.postalNumber);
      return result;
   }

   public boolean equals(@Nullable Object other) {
      if (this == other) {
         return true;
      } else if (!(other instanceof Address)) {
         return false;
      } else {
         Address var2 = (Address)other;
         if (!Intrinsics.areEqual(this.country, var2.country)) {
            return false;
         } else if (!Intrinsics.areEqual(this.city, var2.city)) {
            return false;
         } else if (!Intrinsics.areEqual(this.street, var2.street)) {
            return false;
         } else if (!Intrinsics.areEqual(this.detail, var2.detail)) {
            return false;
         } else {
            return this.postalNumber == var2.postalNumber;
         }
      }
   }
}

```

위 코드의 각 재정의된 메서드를 보면, 예제에서 확인한 테스트가 어떻게 동작하였는지도 쉽게 예측할 수 있습니다.

참고: [Kotlin Intrinsics](https://discuss.kotlinlang.org/t/kotlin-primitives-source-code/2469/8)

### 사용 예시

예제 코드에서 확인한 `katfun`, `katfunAnother`가 있었죠. 만약 또 다른 katfun을 만들고 싶다면?

```kotlin
val katfunYetAnother = katfun.copy()
```

위와 같이 코드를 작성해 주면, 완전히 똑같은 값의 katfun 객체를 하나 생성할 수 있습니다. 둘은 분명 주소 상에서는 다른 값이지만, Kotlin의 정의 상 equality와 identity 모두가 동등합니다.

만들면서 값을 하나 바꾸고 싶다면?

```kotlin
val katfunWithNewId = katfun.copy(2L)	// not recommended
```

혹은

```kotlin
val katfunWithNewName = katfun.copy(name = "정카펀")
```

으로 원하는 필드에 대한 값을 지정해 주면, 해당 필드를 제외한 나머지 값들은 오리지널과 동일하게 복사해 줍니다.

### 결론

"데이터를 나타낸다"라는 역할에 맞게끔, Kotlin의 data class는 데이터를 난타내는 관점에서 class의 일부 멤버 함수들을 재정의해 줍니다.

## Companion Object

## Named Arguments



