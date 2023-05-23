# SAP ABAP CDS(Core Data Service)

## 1. Code Push Down 개념
ABAP 7.4(NW740)부터 도입된 코딩 패러다임으로 HANA DB의 장점을 최대한 끌어올려 데이터의 집약적인 계산을 Application 계층에서 DB계층으로 위임하게 되었다.

CDS는 이러한 Code Push Down을 위한 하나의 방법이다. <br>

 
**As-Is** : DB 서버의 부하를 줄이기 위해 Application Layer에서 작업이 이루어 짐 (Data to Code) <br>
**To-Be** : Hana DB의 성능을 극대화 하기 위해 DB에서 작업이 이루어지고 최종 결과값만 Application Server로 전송 (Code to Data)

참고: https://lizarmong-water.tistory.com/19
![img](https://github.com/jobhope/TechnicalNote/assets/28692938/baf44974-fcad-4449-9be5-688a829ab95b)

### Dictinoary view 와 CDS view 차이?
### Dictionary View (Data to Code)
티코드 SE11에서 생성한다. Dictionary view는 어플리케이션 서버에 상주하며 데이터베이스에 사전에 정의된 데이터들을 가져온다. SQL문을 사용하여 application이 결과를 처리하는데 필요한 데이터를 DB로 부터 가져온다.<br><br>
 
예를 들어, 모든 제품을 출력하는 ABAP 프로그램이 있다. 이를 위해 해당 프로그램은 DB에서 제품에 필요한 모든 데이터를 가져와야 한다. 이 프로그램은 SQL문을 사용하는 대신 ABAP Dictionary view를 사용한다. Dictionary view는 SQL을 포함하고 있으며, SQL에 의해 데이터베이스의 테이블에서 제품의 데이터를 가져온다.<br><br>

 ABAP Dictionary view는 ABAP 프로그램과 데이터베이스 사이의 추가 계층과 같다. 하나 이상의 테이블을 묶어서 필요한 필드들을 반환한다. INNER JOIN이나 SELCET와 같은 제한된 기본 SQL을 사용한다. 해당 뷰를 사용하는 프로그램 사이에 매개변수가 존재하지 않는다.<br><br>
 
### CDS view (Code to Data)
CDS view는 로직을 DB 또는 클라이언트 측으로 푸시한다. 어플리케이션 서버 레벨에서 정의 및 위치하며 모든 DB에서 사용 가능하다. Open SQL을 사용하여 INNER JOIN, OUTER JOIN, UNION 등의 기본적인 명령어들을 사용할 수 있다. 매개변수를 프로그램 단에서 주고 받을 수 있으며 주 목적은 View를 만드는 것이다.


---
## 2. CDS View Syntax
### 2-1. Define
e.g. 주문정보(ZORDERH)와 주문상세(ZORDERI)에 대한 Table 및 CDS view 정의

**Table ZORDERH - 주문 헤더정보**
|Table field name|CDS alias Name|Description|
|:---:|:---:|:---:|
|ZOID|OrderID|주문ID|
|ZDATE|OrderDate|주문 일자|
|ZTAMT|TotalAmount|총 주문 금액|
|ZADDR|Address|배송 주소|
|ZSTAT|Status|주문 상태|

**Table ZORDERI - 주문 아이템정보**
|Table field name|CDS alias Name|Description|
|:---:|:---:|:---:|
|ZOIID|OrderItemID|주문상세ID|
|ZOID|OrderID|주문ID|
|ZPID|ProductID|상품ID|
|ZQUAN|Quantity|수량|
|ZPRICE|Price|상품가격|

**CDS View ZI_OrderHeader**
```
@AbapCatalog.sqlViewName: 'ZIOrderHeader'
@AbapCatalog.compiler.compareFilter: true
@AbapCatalog.preserveKey: true
@AccessControl.authorizationCheck: #CHECK
@EndUserText.label: 'Order Header Info'
define view ZI_OrderHeader as select from ZORDERH
    association [1..*] to ZI_OrderItem as _OrderI on $projection.OrderID = _OrderI.OrderID
{
  key ZOID as OrderId,
  ZDATE as OrderDate
  ZTAMT as TotalAmount,
  ZADDR as Address,
  ZSTAT as Status
  @ObjectModel.association.type: [#TO_COMPOSITION_CHILD]
  _OrderI
}
```
`@AbapCatalog.sqlViewName`: Database의 View이름 정의<br>
`@AbapCatalog.compiler.compareFilter`: 일반적으로 컴파일러는 CDS-Path 표현식에 대해 전용 JOIN문을 생성한다. 값이 True이면, 동일한 Filter 조건일 때 연관된 JOIN은 한 번만 생성한다. 그렇지 않을 경우 각각의 Filter 조건에 대해 별도의 JOIN문을 생성<br>
`@AbapCatalog.preserveKey`: True이면, Key 필드는 명시된 대로 정의. False이면 추가 Key에 관계없이 ABAP DDIC의 클래식 뷰로 결정.<br>
`@AcessControl.authorizationCheck`: Access를 위한 사용자의 권한 검사 수행
`@EndUserText.label`: CDS의 Description 정의<br>

**참고: ABAP CDS - ABAP Annotations**
https://help.sap.com/doc/abapdocu_751_index_htm/7.51/en-US/abencds_annotations_abap.htm?file=abencds_annotations_abap.htm


**CDS View ZI_OrderItem**
```
@AbapCatalog.sqlViewName: 'ZV_ORDERITEM'
@AbapCatalog.compiler.compareFilter: true
@AccessControl.authorizationCheck: #CHECK
@EndUserText.label: 'Order Item View'
define view ZI_OrderItem as select from ZORDERI
  association [1..1] to ZI_OrderHeader as _OrderH on $projection.OrderID = _OrderH.OrderID
{
  key ZOIID as OrderItemID,
  ZOID as OrderID,
  ZPID as ProductID,
  ZQUAN as Quantity,
  ZPRICE as Price,
  @ObjectModel.association.type: [#TO_COMPOSITION_PARENT, #TO_COMPOSITION_ROOT]
  _OrderH
}
```
`@ObjectModel.association.type`: Entity의 관계를 명시
1. `TO_COMPOSITION_CHILD`: 자식 개체
2. `TO_COMPOSITION_PARENT`: 부모 개체
3. `TO_COMPOSITION_ROOT`: 최상위 부모 개체


---
### 2-2. CASE문
case문은 중첩하여 사용 가능
```
define view ZI_OrderHeader as select from ZORDERH
{
  key ZOID as OrderId,
  ZDATE as OrderDate
  ZTAMT as TotalAmount,
  ZADDR as Address,
  case (ZSTAT) 
   when 'A' then 'Active'
   when 'C' then 'Cancelled'
   when 'S' then 'Shipped'
   else 'Unknown'
   end as Status
}
```
---
### 2-3. Cast
Field의 데이터타입을 새로운 타입으로 캐스팅, 중첩 가능.

**참고: ABAP CDS - cast_expr**
https://help.sap.com/doc/abapdocu_751_index_htm/7.51/en-us/abencds_f1_cast_expression.htm
```
define view ZI_PurOrderItem
 as select from ekpo
 association [1..1] to ZI_PurOrderHdr as _POHdr on
$projection.PurchaseOrder = _POHdr.PurchaseOrder
{
 key ebeln as PurchaseOrder,
 key ebelp as PurchaseOrderItem,
 ... 
 cast( netpr as abap.fltp(16) ) * 0.35 as TaxAmount,
 
 // 16자리 * 0.75
 cast( netwr as abap.fltp(16)) * 0.75 as DiscountedNetOrder
 
 // 16자리 * 0.75 값을 INT4 or INT8 Type으로
 ceil( cast(netwr as abap.fltp(16)) * 0.75 ) as DiscountedRoundedNetOrd
}
```
---
### 2-4. Numeric Function and String Functions
**참고 1: ABAP CDS - Numeric Functions**
https://help.sap.com/doc/abapdocu_751_index_htm/7.51/en-us/abencds_f1_sql_functions_numeric.htm

```  
round(netwr,1)  as Round_Op,   // 반올림
ceil(netwr)     as Ceil_Price, // 올림 
floor(netwr)    as Floor_Price,// 내림
div(netwr,3)    as Div_Op,     // 3으로 나누기
division(netwr,3,5) as Div_Op2,// 3으로 나누고 소수 5번째 반올림 
mod(10,3)       as Mod_Op,     // 나머지 구하기
abs(-10)        as Abs_Op      // 절대값
```


**참고 2: ABAP CDS - String Functions**
https://help.sap.com/doc/abapdocu_751_index_htm/7.51/en-us/abencds_f1_sql_functions_character.htm
```
upper(text) as Text_upper,     // String을 대문자로 
left(text,5) as Text_left,     // 왼쪽에서 5글자
rpad(text,8,'y') as Text_rpad, // 8자리까지 빈자리는 'y'로 채움 
substring(text,2,3) as Text_substring // 2번째 자리부터 3길이의 substring
```

---
### 2-5. Types of Annotations

1. View Annotations
    - View에 대한 속성 정의(이름, Compile 방식, 접근권한 등)
    - `@AbapCatalog.sqlViewName`,`@AbapCatalog.compiler.compareFilter`,`@AccessControl.authorizationCheck`,`@AbapCatalog.preserveKey`,`@EndUserText.label` 등
    
3. Element Annotations
    - `@Consumption`,`@UI`, `@ObjectModel` 등 element의 기본값, 주석 등을 정의
    - 참고1: https://help.sap.com/doc/abapdocu_752_index_htm/7.52/en-us/abencds_f1_element_annotation.htm
    - 참고2: https://help.sap.com/doc/saphelp_snc700_ehp04/7.0.4/de-DE/d6/0c0bf6798a481fb7412bc89934cb8a/content.htm?no_cache=true
4. Extension Annotations
    - `@AbapCatalog.sqlViewAppendName`: CDS View Extension 이름 정의(필드 추가된 View)
    - 참고: https://help.sap.com/doc/abapdocu_751_index_htm/7.51/en-us/abencds_f1_extend_view_annotations.htm
    ```
    @AbapCatalog.sqlViewAppendName: 'ZCS_APPEND'
    @EndUserText.label: '028 Extension'
    extend view ZTEST_CDS_028 with ZTEST_CDS_028_EXTN
    {
        spfli.fltime,
        spfli.deptime,
        spfli.arrtime
    }
    ```
    ![1](https://github.com/jobhope/TechnicalNote/assets/28692938/47a93b53-5ca3-4480-969b-9295b17a3cbd)
5. Parameter Annotations
    - `@Environment.systemField`: Input Parameter로 ABAP 시스템 필드를 지정. (#CLIENT: sy-mandt, #USER: sy-uname 등)
    - 참고: https://help.sap.com/doc/abapdocu_750_index_htm/7.50/en-US/abencds_f1_parameter_annotations.htm
    ```
    define view ZCDS_OP 
        with parameters
         @Environment.systemField: #CLIENT
         p1 : mandt,
         @Environment.systemField: #USER
         p2 : uname,
         ...
    ```

## 3. Joins and Associations
### 3-1. Union
2개 이상의 Table 간의 합집합
```
define view ZDemoUnion11 as select from eban {
 matnr as material
} where werks = '0002'
union
select from ekpo {
 matnr as material
} where werks = '0002'
```

### 3-2. Join 
1. Inner Join 
```
define view zdemoJoin11 as select from ekpo as po
inner join eban as pr
on po.banfn = pr
and po.bnfpo = pr.bnfpo {
...
po.ebeln as PONum,
po.matnr as Material,
...
}
```
2. Outer Join

LEFT OUTER JOIN, RIGHT OUTER JOIN 사용 가능
```
define view zdemoJoin11 as select from data_source_name
left outer join joined_data_source_name
on data_data_source_name.element_name = joined_data_data_source_name.joined_element_name{
...
...
}
```

### 3-3. Association
CDS View에서 개체 간의 관계 모델링을 위해 사용. 
관계를 정의하고 해당 관계를 통해 필요한 데이터를 조회하거나 연관된 개체 간의 연산을 수행

참고: https://help.sap.com/docs/SAP_HANA_PLATFORM/4505d0bdaf4948449b7f7379d24d0f0d/809a42308d814b7ea1c8369e55591515.html
```
define view ZI_OrderHeader as select from ZORDERH
    association [1..*] to ZI_OrderItem as _OrderI on $projection.OrderID = _OrderI.OrderID
{
  key ZOID as OrderId,
  ZDATE as OrderDate
  ZTAMT as TotalAmount,
    ...
  @ObjectModel.association.type: [#TO_COMPOSITION_CHILD]
  _OrderI
}
```

|Association|Cardinality|Explanation|
|:---:|:---|:---|
assoc1|	[0..1] |	The association has no or one target instance
assoc2|	 	|Like assoc1, this association has no or one target instance and uses the default [0..1]
assoc3|	[1]|	Like assoc1, this association has no or one target instance; the default for min is 0
assoc4|	[1..1]|	The association has one target instance
assoc5|	[0..*]|	The association has no, one, or multiple target instances
assoc6|	[]|	Like assoc4, [] is short for [0..*] (the association has no, one, or multiple target instances)
assoc7|	[2..7]|	Any numbers are possible; the user provides
assoc8|	[1, 0..*]|	The association has no, one, or multiple target instances and includes additional information about the source cardinality



### 3-4. Join과 Association의 차이 
`Join`, `Association` 모두 데이터를 결합하는데 사용되는 개념이다.

#### 1) Join의 장단점 
#### * 장점
 - JOIN은 DB 레벨에서 수행되기 때문에 DB 엔진의 최적화 기능을 활용.
 - 다양한 조인 유형을 사용하여 필요에 따라 다양한 데이터 조회 및 작업을 수행

#### * 단점 
 - 복잡한 JOIN 구조를 사용할 경우 성능 이슈발생
 - 여러 테이블을 결합할 경우 SQL 문이 복잡해지고 가독성이 저하

#### 2) Association의 장단점 
#### * 장점
 - 개체를 재사용하여 여러 개체간 데이터를 효율적으로 조회
 - 개체 간의 관계를 명시적으로 정의하므로 가독성과 코드 유지보수 용이

#### * 단점
 - 개체 모델의 논리적인 개념으로서, 실제 데이터베이스 연산을 수행하지 않으므로 복잡한 데이터 처리 작업에는 제한적일 수 있음
 - 개체 간의 관계를 명확하게 이해하고 정의해야 하므로 설계 시 주의가 필요

---

## 부록. ADT(ABAP Development Toolset) 설치 

**Eclipse 버전별 호환성 정보**
```
https://tools.hana.ondemand.com
```
**ADT S/W 설치방법**
1. Get an installation of Eclipse 2023-03 (x86_64) (e.g. Eclipse IDE for Java Developers)
2. In Eclipse, choose in the menu bar Help > Install New Software...
3. Enter the URL https://tools.hana.ondemand.com/latest
4. Press Enter to display the available features.
5. Select ABAP Development Tools and choose Next.
6. On the next wizard page, you get an overview of the features to be installed. Choose Next.
7. Confirm the license agreements and choose Finish to start the installation.
