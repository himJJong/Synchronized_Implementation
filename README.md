# synchronized 기법 3가지

## 1) 자바의 synchronized 활용시 하나의 프로세스를 보장, 서버가 1대라면 문제 없지만 그 이상이라면 문제가 생김 여러개의 쓰레드가 동시 접근하기 떄문에 문제가 생길 수 있음.

Transactional 메서드를 지워야함. aService내 Transactional 메서드 사용의 경우 TransactionAService와 같이 새로 만들어 실행하게됨. 
ex) but : startTransaction() -> aServive.decrease(id, quantity) || 이부분에서 DB에 업데이트 하기전에 메서드가 실행되면서 문제가 발생할 수 있음. -> endTransaction()

---

## 2) Mysql을 활용한 다양한 방법 (redis를 사용하면 더 좋은 성능의 동시성 제어가 가능하다)
Pessimistic Lock -> 실제로 데이터에 Lock을 걸어서 정합성을 맞추는 방법입니다. exclusive lock을 걸게되면 다른 트랜잭션에는 lock이 해제되기전에 데이터를 가져갈 수 없게 됩니다.
데드락이 걸릴 수 있기 때문에 주의하여 사용해야한다. 

Optimistic Lock -> 실제로 Lock을 이용하지 않고 버전을 이용함으로써 정합성을 맞추는 방법입니다. 
먼저 데이터를 읽은 후에 update를 수행할 대 현재 내가 읽은 버전이 맞는지 확인하며 업데이트 합니다. 내가 읽은 버전에서 수정사항이 생겼을 경우에는 application에서 다시 읽은 후에 작업을 수행해야한다. 

Named Lock -> 이름을 가진 metadata locking 입니다. 이름을 가진 lock을 획득한 후 해제할때까지 다른 세션은 이 lock을 획득할 수 없도록 합니다. 
주의할점으로는 transaction 이 종료될 때 lock이 자동으로 해제되지 않습니다. 별도의 명령어로 해제를 수행해주거나 선점시간이 끝나야 해제됩니다.


### 정리
OptimisticLock은 실패했을 때 재시도하는 로직이 있지만, 별도의 Lock을 잡지 않아 PessimisticLock 보다 성능상 이점이 있다. 
단점으로는 update가 실패했을 때 재시작 로직을 개발자가 직접 작성해야한다. 충돌이 빈번하게 일어날 경우에는 PessimisticLock이 유용할 것 이고, 그렇지 않다면 OptimisticLock이 유용할 것이다.

NamedLock 주로 분산락을 구현할 때 사용. PessimisticLock은 timeout을 구현하기 힘들지만 NamedLock은 쉽게 구현할 수 있고, 
데이터 삽입시 정합성을 맞추기 위해 NamedLock을 사용할 수 있고, 트랜잭션 종료시 세션관리를 잘 해야하기 떄문에 주의해서 사용해야 한다. 구현방법이 복잡할 수 있다.

---

## 3) Redis를 통한 분산락 구현

Lettuce : setnx 명령어를 활용하여 분산락 구현, spin lock 방식
Redisson : pub-sub 기반으로 Lock 구현 제공

Lettuce
방법은 구현이 간단하다는 장점이 있다. spring data redis를 이용하면 lettuce가 기본이기에 별도의 라이브러리가 필요없다. spin Lock 방식이라 동시에 많은 쓰레드가 lock 획득 대기 상태라면 Redis에 부하를 줄 수 있다. Thread sleep으로 락 획득 재시도간에 텀을 둬야한다.

Redisson
락 획득 재시도를 기본으로 제공한다. pub-sub기반의 구현이기에 lettuce에 비해 redis에 부하를 줄여준다. 별도의 라이브러리를 사용해야 하고, Lock을 라이브러리 차원에서 제공하기 때문에 별도의 사용법을 공부해야한다.

### 정리
실무에서는 재시도가 필요하지 않은 lock은 lettuce를 활용 / 재시도가 필요한 경우에는 redisson을 활용한다.

---

< Mysql Redis 비교 >

mysql은 이미 사용하고 있다면 별도의 비용없이 사용가능하고, 어느정도의 트래픽까지는 문제없이 활용할 수 있으며, redis보다는 성능이 좋지 않다. 
Redis는 활용중이 않다면 별도의 구축비용과 인프라 관리 비용이 발생하지만, mysql보다는 성능이 좋다.
