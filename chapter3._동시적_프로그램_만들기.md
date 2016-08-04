# chapter3. 동시적 프로그램 만들기
## 이 챕터에서의 목표
- Observable 파이프라인에 대해서 알아봅니다.
- RxJS의 순수 함수에 대해서 이해합시다.
- Reactive Programming에서의 Subject의 각각의 종류와 개념을 알아봅니다.
- 배운지식을 활용해서 브라우저용 shoot-’em-up spaceship 게임에 적용합니다.



## Observable 파이프라인
Observable 파이프라인은 몇개의 operator가 같이 결합된 것을 의미합니다. 파이프라인에서 operator들은 하나의 observable을 입력으로 받고 하나의 observable을 반환합니다.(이것을 순수한 연산이라합니다...?) 파이프라인에서 operator간의 상태는 유지됩니다.

```
Rx.Observable
	.from(1, 2, 3, 4, 5, 6, 7, 8) 
	.filter(function(val) { return val % 2; }) 
	.map(function(val) { return val * 10; });
```

- ### 전역변수 사용을 피하기

순수함수란 같은 입력값에 대해 항상 같은 출력을 반환하는 함수를 말합니다. 이를통해 전역변수로 사용을 피할 수 있습니다.

- ### 효율적인 파이프라인

JS에서 Array를 chaining operator를 사용해서 변형하게되면 비용이 많이든다. operator를 거칠때마다 새로운 Array를 리턴하기 때문이다. 하지만 Observable 파이프라인은 중간에 새로운 Observable을 만들지 않고, 모든 operator를 거친 Observable을 리턴한다. 이러한점에서 매우 효율적이라고 할수 있습니다.

다음의 예제에서 그 차이점을 확인할 수 있습니다.

```
stringArray // represents an array of 1,000 strings 
	.map(function(str) { // #1 
		return str.toUpperCase(); }) 
	.filter(function(str) { //#2
		return /^[A-Z]+$/.test(str); })
	.forEach(function(str) { 
		console.log(str); }); //#3
```
1. 배열 전체를 반복하면서, 대문자로 변환된 1000개의 문자열을 가진 새로운 배열을 만든다.
2. 1에서 생성된 배열 전체를 반복하면서, 표현식에 맞는 1000개의 문자열을 가진 새로운 배열을 만든다.
3. 2에서 생선된 배열 전체를 반복하면서, 1000개의 문자열을 로그로 찍는다.
	
같은 상황을 Observable로 표현했을때는 다음과 같습니다.

```
stringObservable // represents an observable emitting 1,000 strings
 .map(function(str) { // #1
	return str.toUpperCase(); })
 .filter(function(str) { // #2
	return /^[A-Z]+$/.test(str); })
 .subscribe(function(str) {  //#3
 	console.log(str); });
```

1. 각각의 Observable을 대문자로 변환하는 함수를 만든다. 그리고 subscribe될 시에만, 이 함수가 적용된 값을 내보내는 Observable을 리턴한다.
2. 1의 대문자 변환함수와 함께 적용되는 filter 함수를 만든다. 그리고 subcribe될 시에만, 대문자 변환함수와 filter함수가 적용된 값을 내보내는 Observable을 리턴한다.
3. Observable을 subscribe하여, 값을 만들어 내도록 한다. 각각의 아이템들에 대한 (1,2의)변형함수는 아이템당 한번만 적용된다. 



## Rx's Subject
### Subject
Subject는 Observer와 Observable을 함께 구현한 형태의 타입입니다. Observer로써 Observable을 정기적으로 받을 수 있고,  Observable로써 값을 만들어내거나 자신을 정기적으로 받아줄 Observer를 가질수도 있습니다. 
다음의 시나리오에서는 하나의 Subject가  Observer와 Observable의 역할을 같이 수행하는 것을 볼수 있습니다. data source와 subject의 리스너 사이의 proxy객체를 만드는 것이 대표적인 예입니다.
다음은 Subject가 proxy객체로 사용되는 예 입니다.

	```
	var subject = new Rx.Subject();
	var source = Rx.Observable.interval(300)
		.map(function(v) { return 'Interval message #' + v; }) .take(5);
	   source.subscribe(subject);
	var subscription = subject.subscribe(
	function onNext(x) { console.log('onNext: ' + x); },
	function onError(e) { console.log('onError: ' + e.message); }, function onCompleted() 	{ console.log('onCompleted'); }
	);
	subject.onNext('Our message #1');
	subject.onNext('Our message #2');
	setTimeout(function() { subject.onCompleted();
	}, 1000);
	```
### AsyncSubject
  AsyncSubject는 source Observable으로부터 받게된 가장 마지막 값을 캐쉬해둡니다.그리고 source Observable이 종료되었을때 해당 값을 내보냅니다. (만약에 source에서 아무런 값을 받지 못했다면, AsyncSubject또한 아무런 값을 내보내지 않습니다.)

```
  var delayedRange = Rx.Observable.range(0, 5).delay(1000); var subject = new Rx.AsyncSubject();
   delayedRange.subscribe(subject);
subject.subscribe(
function onNext(item) { console.log('Value:', item); }, function onError(err) { console.log('Error:', err); }, function onCompleted() { console.log('Completed.'); }
);
```
  <!--AsyncSubject is similar to the Replay and Behavior subjects, however it will only store the last value, and only publish it when the sequence is completed. You can use the AsyncSubject type for situations when the source observable is hot and might complete before any observer can subscribe to it. In this case, AsyncSubject can still provide the last value and publish it to any future subscribers.-->
 
### BehaviorSubject
  BehaviorSubject는 ReplaySubject와 비슷하지만 오로지 마지막 값만 저장합니다. 여기서는 초기화때 초기값을 필요로합니다. 이 초기값은 subject가 아직 아무런 값을 받지 못한채 subscribe되었을때 observer들에게 넘겨줍니다. 즉 subject가 complete 되지 않아도 subscriber들은 값을 받을수 있다는 것을 의미합니다. <!--BehaviBehaviourSubject  similar to ReplaySubject, except that it only stored the last value it published. BehaviourSubject also requires a default value upon initialization. This value is sent to observers when no other value has been received by the subject yet. This means that all subscribers will receive a value instantly on subscribe, unless the Subject has already completed.-->
  
### ReplaySubject
  ReplaySubject는 발생시킨 모든 값들을 저장합니다. 이 값들을 정기적으로 받았을때, 당신은 자동적으로 모든값들의 전체적인 히스토리를 받을수 있습니다. 심지어 당신의 subscription이 특정값이 내보내진 다음에라도 모든 값들을 가져올 수 있습니다. <!--ReplaySubject stores all the values that it has published. Therefore, when you subscribe to it, you automatically receive an entire history of values that it has published, even though your subscription might have come in after certain values have been pushed out.
-->
	
	 

	


