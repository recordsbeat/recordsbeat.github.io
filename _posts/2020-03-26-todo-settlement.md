
todo 정산
정산 서비스(도메인) 개발
1. 정산 받을 내역을 확인 후 각 대상의 수수료율을 계산하여 리스트를 보여준다.
2. 각 내역에 해당하는 입금 내역을 입력하고 정산 상태 값을 변경한다.

고려해야할 사항
1. 각 내역의 정산 상태를 부여한다.
2. 각 대상의 수수료율을 사용한 지급 금액을 도출한다.

정산 상태에 따른 상태 값을 구성하여 enum 클래스와 매핑을 진행한다.<br>
->완료

참고<br>
https://m.blog.naver.com/kkforgg/220910969555<br>
https://www.podo-dev.com/blogs/120<br>
https://www.holaxprogramming.com/2015/11/12/spring-boot-mybatis-typehandler/<br>


mybatis상태에서 entity와 dto 의 구분이 필요할지 ? jpa와 다르게 쿼리로써 데이터구성이 가능한거같은데..<br>
-> DTO or VO로 해결보는 방식 ? 

각 수수료율을 db에서 도출 후 코드 단의 금액 계산 컴포넌트를 작성한다. <br>
-> 수수료 등급에 따른 금액 계산을 의도했으나 계산식은 공통적이며 파라미터 값이 다른경우<br>
-> mysql view를 사용해 미리 계산한 값을 사용하도록 도출<br>
-> 내역은 최상단 테이블인 예약, 지급 예정금액 view(정산 수수료율 적용), 정산내역 테이블 join 정책<br>

UI상 노출되어야할 selectbox를 enum 변수로써 통신하도록 한다.<br>
-> 작업완료<br>
-> 각 Enum 클래스에 comment 필드추가 사용자 UI 전용.<br>
-> mybatis 와 통신에서는 무리 없이 진행됨 (code만 가지고 매핑하니 당연할듯..)<br>



참고<br>
https://github.com/jojoldu/blog-code/tree/master/java/enum-mapper



아래는 작성한 코드들..<br>


enum class 예제<br>

```

import lombok.AllArgsConstructor;

@AllArgsConstructor
public enum CityEnum implements CodeEnum{
	
	SEOUL("1","서울"),
	BUSAN("2","부산"),
	CEBU("3","세부"),
	MANILA("4","마닐라"),
	BORACAY("5","보라카이"),
	DANANG("6","다낭"),
	HOCHIMINH("7","호치민"),
	NHATRANG("8","나트랑"),
	BOHOL("9","보홀");
	
	private final String code;
	private final String comment;

	@Override
    public String getComment() {
        return comment;
    }
	@Override
    public String getKey() {
        return name();
    }

    @Override
    public String getCode() {
        return code;
    }
	
	//이 구문이 없으면 CityEnum을 필드로 갖는 클래스에서 매핑이 제대로 되지 않았음..
  //이유가 뭘까
	public static class TypeHandler extends CodeEnumTypeHandler<CityEnum> {
		public TypeHandler() {
			super(CityEnum.class);
		}
	}
	
}

```


implemented interface<br>


```
public interface CodeEnum {
	public String getCode();	
	public String getKey();	
	public String getComment();	
}
```

CodeEnumTypeHandler 클래스<br>
아래에서 mybatis config에 추가되는 핸들러다<br>
실질적인 db데이터와 enum 클래스간의 매핑을 도와준다.<br>
```
import java.sql.CallableStatement;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.EnumSet;

import org.apache.ibatis.type.BaseTypeHandler;
import org.apache.ibatis.type.JdbcType;

import com.yourProject.interfaces.CodeEnum;

public class CodeEnumTypeHandler<E extends Enum<E> & CodeEnum> extends BaseTypeHandler<E> {

	private Class<E> type;

	public CodeEnumTypeHandler(Class<E> type) {
		if (type == null)
			throw new IllegalArgumentException("Type argument cannot be null");
		this.type = type;
	}

	@Override
	public void setNonNullParameter(PreparedStatement ps, int i, E parameter, JdbcType jdbcType) throws SQLException {
		if (jdbcType == null) {
			ps.setString(i, parameter.getCode());
		} else {
			ps.setObject(i, parameter.getCode(), jdbcType.TYPE_CODE);
		}
	}

	@Override
	public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
		String s = rs.getString(columnName);
		return getCodeEnum(type, s);
	}

	@Override
	public E getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
		String s = rs.getString(columnIndex);
		return getCodeEnum(type, s);
	}

	@Override
	public E getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
		String s = cs.getString(columnIndex);
		return getCodeEnum(type, s);
	}

	public static <T extends Enum<T> & CodeEnum> T getCodeEnum(Class<T> enumClass, String code) {
		return EnumSet.allOf(enumClass)
				.stream()
				.filter(type -> type.getCode().equals(code))
				.findFirst()
				.orElseGet(null);
	}
}
```


mybatis config<br>
아래 typeHandler 태그를 살펴보면 위의 CodeEnum 형태일 경우 CodeEnumTypeHandler 로 매핑하도록..<br>
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>

	<settings>
		<setting name="mapUnderscoreToCamelCase" value="true"/>
	</settings>
	
	<typeAliases>
		<typeAlias alias="resultMap"  type="com.yourproject.common.util.ResultMap"/>
        <typeAlias alias="hashMap"    type="java.util.HashMap"/>  
        
	</typeAliases>
	<typeHandlers>
		 <typeHandler javaType="com.yourproject.interfaces.CodeEnum"
                     handler="com.yourproject.typehandler.CodeEnumTypeHandler" />
	</typeHandlers>
</configuration>
```


위와 같은 세팅을 마친 후 각 Enum 클래스를 사용자의 UI에서 받아 볼 수 있도록 세팅해본다<br>
(comment 필드를 만든이유..)<br>
참조링크(배민 블로그)에 있는걸 배껴왓당..<br>

```

import java.util.Arrays;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;

import com.yourproject.dto.EnumValue;
import com.yourproject.interfaces.CodeEnum;

public class EnumMapper {
    private Map<String, List<EnumValue>> factory = new HashMap<>();

    private List<EnumValue> toEnumValues(Class<? extends CodeEnum> e){
            // Java8이 아닐경우
//            List<EnumValue> enumValues = new ArrayList<>();
//            for (CodeEnum enumType : e.getEnumConstants()) {
//                enumValues.add(new EnumValue(enumType));
//            }
//            return enumValues;

        return Arrays
                .stream(e.getEnumConstants())
                .map(EnumValue::new)
                .collect(Collectors.toList());
    }

    public void put(String key, Class<? extends CodeEnum> e){
        factory.put(key, toEnumValues(e));
    }

    public Map<String, List<EnumValue>> getAll(){
        return factory;
    }

    public Map<String, List<EnumValue>> get(String keys){

            // Java8이 아닐경우
//            Map<String, List<EnumValue>> result = new LinkedHashMap<>();
//            for (String key : keys.split(",")) {
//                result.put(key, factory.get(key));
//            }
//
//            return result;
    	
        return Arrays
                .stream(keys.split(","))
                .collect(Collectors.toMap(Function.identity(), key -> factory.get(key)));
    }


}
```

위와 같이 toEnumValues를 사용해 CodeEnum 인터페이스를 상속 받는 Enum인자를 DTO로 전환한다.<br>

참고링크에서 언급한대로 Bean에 등록하면 좋을거 같아서 유틸로 등록했다.<br>

참고링크는 appconfig 클래스를 만들고 @Bean 등록을 진행했다.<br>
근데 Bean 객체를 불러오는게 귀찮아(사실 잘모르고 찾아봤는데 번거로워보였다 getBean을 따로 만들어줘야했던듯..)<br>

```

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

import com.yourproject.enums.CityEnum;
import com.yourproject.enums.UserEnum;
import com.yourproject.enums.PayEnum;

@Component
public class ValueUtils {

	protected Logger logger = LoggerFactory.getLogger(this.getClass());
	private static EnumMapper enumMapper;
	@Bean
    private EnumMapper enumMapper() {
        this.enumMapper = new EnumMapper();
        enumMapper.put("CityEnum", CityEnum.class);
        enumMapper.put("UserEnum", UserEnum.class);
        enumMapper.put("PayEnum", PayEnum.class);
        return enumMapper;
    }
    public static EnumMapper getMapper() {
    	return enumMapper;
    }
}
```

이렇게 bean에 등록된 ValueUtils을 컨트롤러 단에서 UI에 넘겨준다.<br>
굳이 getAll()뒤에 get()을 한 번 더 한이유는 EnumMapper(갓 배민이 만들어준)클래서에서 get()메소드가 <br>
콤마 , 를 기준으로 여러개의 EnumValue를 반환해 Map 형태를 띄고있었고, UI단에서 컨트롤 하기 번거로워 보여 그냥 <br>
get()을 한 번 더 사용하여 List<EnumValue>로 받기로 결정<br>
```
List<EnumValue> UserStatus = ValueUtils.getMapper().getAll().get("UserEnum");
```

해서 UI상에 아래와 같이 노출 시키기로함 <br>
말하기 창피하지만 이전에는 아무런 상태값에 대한 규정이 없어 상수선언했다.(그야말로 노가다)<br>

```
<select class="User_status" name="User_status" onclick="event.stopPropagation();">	
<c:forEach items="${UserStatus}" var="UserStatus" varStatus="status">					
  <option value="${UserStatus.key}" <c:if test="${data.User_status eq UserStatus.value}">selected</c:if>>${UserStatus.comment}</option>			
</c:forEach>
										
</select>		
```



위와 같은 세팅을 거쳐 얻은 것<br>


1. 테이블 상 모호한 상태값과 플래그를 소스단의 Enum으로 매핑하여 값의 범위를 규정하고 가독성을 높힌다.<br>
(가령 잘못된 값(Enum클래스에 선언된 값 이외)이 질의로 날라오더라도 mybatis 매핑에서 쳐낼 수 있다.)<br>

2. UI단에 공통코드를 사용하여 노출하는 효과.. DB탐색을 안해서 좀 더 나을 수도?<br>

더 있는지는 생각해봐야겠다..<br>


다음은 사용자 이용을 위한 View 페이지를 만들고 service 추가 작업이 필요하겠지..
