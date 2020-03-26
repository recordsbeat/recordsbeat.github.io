
todo 정산
정산 서비스(도메인) 개발
1. 정산 받을 내역을 확인 후 각 대상의 수수료율을 계산하여 리스트를 보여준다.
2. 각 내역에 해당하는 입금 내역을 입력하고 정산 상태 값을 변경한다.

고려해야할 사항
1. 각 내역의 정산 상태를 부여한다.
2. 각 대상의 수수료율을 사용한 지급 금액을 도출한다.

정산 상태에 따른 상태 값을 구성하여 enum 클래스와 매핑을 진행한다.
->완료

참고
https://m.blog.naver.com/kkforgg/220910969555<br>
https://www.podo-dev.com/blogs/120<br>
https://www.holaxprogramming.com/2015/11/12/spring-boot-mybatis-typehandler/


mybatis상태에서 entity와 dto 의 구분이 필요할지 ? jpa와 다르게 쿼리로써 데이터구성이 가능한거같은데..
-> DTO or VO로 해결보는 방식 ? 

각 수수료율을 db에서 도출 후 코드 단의 금액 계산 컴포넌트를 작성한다. 
-> 수수료 등급에 따른 금액 계산을 의도했으나 계산식은 공통적이며 파라미터 값이 다른경우
-> mysql view를 사용해 미리 계산한 값을 사용하도록 도출
-> 내역은 최상단 테이블인 예약, 지급 예정금액 view(정산 수수료율 적용), 정산내역 테이블 join 정책

UI상 노출되어야할 selectbox를 enum 변수로써 통신하도록 한다.
-> 작업요

참고

https://github.com/jojoldu/blog-code/tree/master/java/enum-mapper
