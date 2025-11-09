
---

1️⃣ 요청 B : 락 점유 중 → 재시도 방식

요청 B가 들어왔을 때 A가 이미 락을 가지고 있으면,
즉시 429 로 실패시키는 대신 백오프(대기 → 재시도) 로 바꾸면 된다.
```python
import asyncio
import uuid
import time
from fastapi import FastAPI, Depends
from dependencies import get_mongo, get_redis

app = FastAPI()
LOCK_EXPIRE = 10          # 락 TTL
LOCK_RETRY_INTERVAL = 0.5 # 재시도 주기
LOCK_MAX_RETRY = 10       # 최대 10회 (≈5초)

async def acquire_lock(redis, key, value, expire=LOCK_EXPIRE):
    return redis.set(key, value, nx=True, ex=expire)

async def release_lock(redis, key, value):
    stored = redis.get(key)
    if stored and stored == value:
        redis.delete(key)

@app.post("/send_message")
async def send_message(body, mongo=Depends(get_mongo), redis=Depends(get_redis)):
    chatroom_key = f"chatroom:{body.chatroom_name}"
    lock_key = f"lock:{body.chatroom_name}"
    lock_val = str(uuid.uuid4())

    # 재시도 루프
    for _ in range(LOCK_MAX_RETRY):
        if await acquire_lock(redis, lock_key, lock_val):
            break
        await asyncio.sleep(LOCK_RETRY_INTERVAL)
    else:
        # 최대 재시도 초과
        raise HTTPException(status_code=503, detail="Lock wait timeout")

    try:
        # chatroom 생성 or 조회 로직 (동일)
        ...
    finally:
        await release_lock(redis, lock_key, lock_val)
```
동작 흐름

1. B가 락을 얻지 못하면 0.5초 대기
2. 다시 SETNX 시도
3. 최대 5초 동안 A의 락이 풀리면 바로 진행
4. 풀리지 않으면 503 반환

이 패턴은 Redis 기반 분산락의 표준 재시도 구조 이며,
짧은 요청 경쟁 상황에서 안정적으로 동작한다.


---

2️⃣ Replica 환경 (멀티 FastAPI / 멀티 Pod / 멀티 노드) 적용성

네, 이 구조는 Replica 여러 개에서도 그대로 유효하다.
핵심은 락을 잡는 Redis 서버가 단일 논리적 인스턴스 (Cluster 또는 Sentinel master) 라는 점이다.

환경	동작 가능 여부	설명

단일 Redis Instance	✅ 완벽히 동작	모든 FastAPI replica 가 하나의 Redis를 사용
Redis Sentinel (HA)	✅ 동작	master 노드 기준으로 SETNX 수행
Redis Cluster (Sharded)	⚠️ 키 분산 주의	동일 key hash 슬롯이 보장되면 가능
여러 Redis Instance 분리 (무관한 서버)	❌ 불가	락 정보 공유 불가능


즉, FastAPI replica가 여러 개라도, 공통 Redis를 참조하면 문제없다.


---

3️⃣ 고신뢰 환경(전역 락) 이 필요할 때

Redis Cluster 또는 멀티 리전 환경에서는
RedLock 알고리즘 (5 노드 중 3 이상 동의 락 확보)을 사용한다.

파이썬 예시:

from redis import Redis
from redis.lock import Lock
from redis.exceptions import LockError

r = Redis(host="redis-master")
with r.lock("lock:chatroom_x", timeout=10, blocking_timeout=5):
    # 임계 구역
    ...

또는 redis-py 공식 RedLock 패키지(redis.lock.RedLock)를 사용해 다중 노드 락을 구현할 수 있다.


---

✅ 정리

항목	설명

락 목적	중복 chatroom 생성 방지
재시도 방식	일정 간격 대기 → 락 해제 시 진행
적용 범위	FastAPI Replica 여러 개에서도 동일하게 적용 가능
전역 분산 환경	필요시 RedLock 사용



---
