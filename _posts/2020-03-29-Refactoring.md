---
title: "리펙토링 해보기"
date: 2020-03-20 10:20:20 -0400
categories: enum interface
---


꿈에서도 생각난 api 리펙토링부분 .. 무한 if 지옥에서 탈출하는 방법을 찾아보고 꼭 맞는걸 찾아 적용하기로 하였다.

참고링크

https://junspapa-itdev.tistory.com/36<Br>
https://junspapa-itdev.tistory.com/37

enum 을 사용해 하나의 인터페이스를 상속받은 인스턴스를 전달받아 로직 실행..

아래와 같은 코드가 작성되었다 . (테스트는 아직)


맨 처음 통신을 위한 DTO 구성
```

import com.guiving.refactoring.interfaces.CodeEnum;
import com.guiving.refactoring.typehandler.CodeEnumTypeHandler;

import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.ToString;

@NoArgsConstructor
@Getter
@ToString
public class TripDto {
	private int tripIdx;
	private int tripNum;
	private int guivingIdx;
	private TripStatus status;

	@AllArgsConstructor
	public enum TripStatus implements CodeEnum{
		
		STANDBY("1","호출대기"),
		CALLING("2","호출대기"),
		DEPARTING("3","운행시작"),
		BOARDED("4","승객승차"),
		ARRIVED("5","목적지도착"),
		ALIGHTED("6","승객하차"),
		AWAITED("7","기사대기"),
		TERMINATED("8","운행완료"),
		NEWTRAVEL("9","다음운행"),
		CANCELED("10","호출취소");
		

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

		public static class TypeHandler extends CodeEnumTypeHandler<TripStatus> {
			public TypeHandler() {
				super(TripStatus.class);
			}
		}
	}
}

```

그 다음은 인터페이스 작성

```

import com.guiving.refactoring.dto.TripDto;

public interface TripHandlerInterface {
	public TripDto tripUpdate(TripDto trip) throws Exception;
}

```

인터페이스를 토대로 레거시에 따른 handler클래스 작성

```
package com.guiving.refactoring.service;

import org.springframework.stereotype.Service;

import com.guiving.refactoring.dto.TripDto;
import com.guiving.refactoring.interfaces.TripHandlerInterface;

import lombok.AllArgsConstructor;
import lombok.RequiredArgsConstructor;

@Service
@RequiredArgsConstructor
public class TripHandler {

	/*
	 * 통신방식은 Request DTO 와 Response DTO 를 나눠서 진행
	 * legacy의 파라미터 자료형 변경 필요
	 */
	public TripDto tripUpdate(TripDto trip) throws Exception {
		
		TripHandler.Status status = TripHandler.Status.valueOf(trip.getStatus().toString());
		TripHandlerInterface handler = status.getHandler();
		handler.tripUpdate(trip);
		
		/*legacy		 
		
    	*/
		
		return trip;
	}

	@AllArgsConstructor
	public enum Status {

		STANDBY(TripDto.TripStatus.STANDBY, new StandbyHandler()),
		CALLING(TripDto.TripStatus.CALLING, new CallingHandler()),
		DEPARTING(TripDto.TripStatus.DEPARTING, new DepartingHandler()),
		BOARDED(TripDto.TripStatus.BOARDED, new BoardedHandler()),
		ARRIVED(TripDto.TripStatus.ARRIVED, new ArrivedHandler()),
		ALIGHTED(TripDto.TripStatus.ALIGHTED, new AlightedHandler()),
		AWAITED(TripDto.TripStatus.AWAITED, new AwaitedHandler()),
		TERMINATED(TripDto.TripStatus.TERMINATED, new TerminatedHandler()),
		NEWTRAVEL(TripDto.TripStatus.NEWTRAVEL, new NewtravelHandler()),
		CANCELED(TripDto.TripStatus.CANCELED, new CanceledHandler());

		private TripDto.TripStatus status;
		private TripHandlerInterface handler;

		public TripDto.TripStatus getStatus() {
			return status;
		}

		public TripHandlerInterface getHandler() {
			return handler;
		}

		
		private static class StandbyHandler implements TripHandlerInterface {

			public TripDto tripUpdate(TripDto trip) throws Exception {
				/*
				//legacy
				*/
				return trip;
			}
		}

		private static class CallingHandler implements TripHandlerInterface {

			public TripDto tripUpdate(TripDto trip) throws Exception {
				/*
				//legacy
				 */
				return trip;
			}
		}

		private static class DepartingHandler implements TripHandlerInterface {
		    
			public TripDto tripUpdate(TripDto trip) throws Exception {
				
				/*
				//legacy
				
				*/
				
				return trip;
			}
		}

		private static class BoardedHandler implements TripHandlerInterface {

			public TripDto tripUpdate(TripDto trip) throws Exception {
				/*
				//legacy
				
    			*/    		
				return trip;
				
			}
		}

		private static class ArrivedHandler implements TripHandlerInterface {

			public TripDto tripUpdate(TripDto trip) throws Exception {
				/*
				//legacy
			
    			*/
				return trip;
			}
		}

		private static class AlightedHandler implements TripHandlerInterface {

			public TripDto tripUpdate(TripDto trip) throws Exception {
				/*
				//legacy
				
    			*/
				return trip;
			}
		}

		private static class AwaitedHandler implements TripHandlerInterface {

			public TripDto tripUpdate(TripDto trip) throws Exception {
				/*
				//legacy
				
    			*/
				return trip;
			}
		}

		private static class TerminatedHandler implements TripHandlerInterface {

			public TripDto tripUpdate(TripDto trip) throws Exception {
				/*
				//legacy
				
    			*/
				return trip;
			}
		}

		private static class NewtravelHandler implements TripHandlerInterface {

			public TripDto tripUpdate(TripDto trip) throws Exception {
				/*
				//legacy
			
    			
    			*/
				return trip;
			}
		}

		private static class CanceledHandler implements TripHandlerInterface {

			public TripDto tripUpdate(TripDto trip) throws Exception {
				/*
				//legacy
				
    			*/
				return trip;
			}
		}
	}
}

```

일단 legacy 코드는 모두 지운상태

아직 진행 중인 코드이기 때문에 통신 자료형을 바꿀 필요가 있다.

주석에 써놓은대로 Request와 Response DTO 를 나누어 통신할 것...

상태값이 많기도하고 로직을 처리하는 클래스들이 Enum과 연관 있어보이게 하기 위해 inner 클래스로 작성.


최상단의 TripDto 와 TripHandler.Status 가 겹치는 점이 좀 아쉽다. 같은 상태값인 코드가 두 번 들어간 상황.

어떻게 처리할 수 있을지 고민해봐야지..
