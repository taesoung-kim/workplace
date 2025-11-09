합리적이다.
운영 환경에서는 전체 초기화(/sync_all) 와 일부 동기화(/sync) 를 분리해야
잘못된 요청 하나로 Redis 전체 캐시가 덮이는 사고를 막을 수 있다.

아래는 그 기준에 맞게 구조를 분리한 최종 안정형 main.py 버전이다.


---

🧩 main.py (운영 안전형 구조)

from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from dependencies import get_mongo, get_redis
import asyncio
import uuid
import time

app = FastAPI(title="MS Teams Chatroom API")

LOCK_EXPIRE = 10
LOCK_RETRY_INTERVAL = 0.5
LOCK_MAX_RETRY = 10
BATCH_SIZE = 100


# ---------- BaseModel ----------
class MessageRequest(BaseModel):
    chatroom_name: str
    message: str
    title: str
    recipient: str


class SyncRequest(BaseModel):
    chatroom_names: list[str]


# ---------- Redis Lock ----------
async def acquire_lock(redis, key: str, value: str, expire: int = LOCK_EXPIRE) -> bool:
    return redis.set(key, value, nx=True, ex=expire)


async def release_lock(redis, key: str, value: str):
    stored = redis.get(key)
    if stored and stored == value:
        redis.delete(key)


# ---------- Chatroom Helpers ----------
async def create_chatroom(name: str) -> str:
    """MS Teams API 호출 예시"""
    await asyncio.sleep(0.2)
    return str(uuid.uuid4())


async def send_to_ms_teams(chatroom_id: str, message: str, title: str, recipient: str):
    await asyncio.sleep(0.2)


# ---------- /send_message ----------
@app.post("/send_message")
async def send_message(body: MessageRequest, mongo=Depends(get_mongo), redis=Depends(get_redis)):
    chatroom_key = f"chatroom:{body.chatroom_name}"
    lock_key = f"lock:{body.chatroom_name}"
    lock_val = str(uuid.uuid4())

    chatroom_id = redis.hget(chatroom_key, "chatroom_id")

    if not chatroom_id:
        for _ in range(LOCK_MAX_RETRY):
            if await acquire_lock(redis, lock_key, lock_val):
                break
            await asyncio.sleep(LOCK_RETRY_INTERVAL)
        else:
            raise HTTPException(status_code=503, detail="Lock wait timeout")

        try:
            doc = mongo.chatrooms.find_one({"chatroom_name": body.chatroom_name})
            if doc:
                chatroom_id = str(doc["chatroom_id"])
                redis.hset(chatroom_key, "chatroom_id", chatroom_id)
            else:
                chatroom_id = await create_chatroom(body.chatroom_name)
                mongo.chatrooms.insert_one({
                    "chatroom_name": body.chatroom_name,
                    "chatroom_id": chatroom_id,
                    "created_at": time.time(),
                })
                redis.hset(chatroom_key, "chatroom_id", chatroom_id)
        finally:
            await release_lock(redis, lock_key, lock_val)

    asyncio.create_task(send_to_ms_teams(chatroom_id, body.message, body.title, body.recipient))
    return {"ok": True, "chatroom_id": chatroom_id, "message": "Message send task started."}


# ---------- /sync (부분/선택 동기화) ----------
@app.post("/sync")
async def sync(body: SyncRequest, mongo=Depends(get_mongo), redis=Depends(get_redis)):
    """
    MongoDB → Redis 동기화
    - 지정된 chatroom_names 만 반영
    - batch 100 단위 pipeline 처리
    """
    if not body.chatroom_names:
        raise HTTPException(status_code=400, detail="chatroom_names required")

    pipe = redis.pipeline(transaction=False)
    count = 0
    total = len(body.chatroom_names)

    for i, name in enumerate(body.chatroom_names, start=1):
        doc = mongo.chatrooms.find_one({"chatroom_name": name})
        if doc:
            pipe.hset(f"chatroom:{name}", "chatroom_id", str(doc["chatroom_id"]))
            count += 1

        if i % BATCH_SIZE == 0:
            pipe.execute()
            pipe = redis.pipeline(transaction=False)

    if total % BATCH_SIZE != 0:
        pipe.execute()

    return {"mode": "partial", "ok": True, "requested": total, "synced": count, "batch_size": BATCH_SIZE}


# ---------- /sync_all (전체 초기화 전용) ----------
@app.post("/sync_all")
async def sync_all(mongo=Depends(get_mongo), redis=Depends(get_redis)):
    """
    MongoDB 전체 → Redis 전체 동기화
    운영 주의: 전체 캐시 갱신 용도
    """
    cursor = mongo.chatrooms.find({}, {"chatroom_name": 1, "chatroom_id": 1})
    pipe = redis.pipeline(transaction=False)
    count = 0
    total = 0

    for doc in cursor:
        pipe.hset(f"chatroom:{doc['chatroom_name']}", "chatroom_id", str(doc["chatroom_id"]))
        count += 1
        total += 1

        if count >= BATCH_SIZE:
            pipe.execute()
            count = 0
            pipe = redis.pipeline(transaction=False)

    if count > 0:
        pipe.execute()

    return {"mode": "all", "ok": True, "synced": total, "batch_size": BATCH_SIZE}


---

⚙️ 운영 구분 설계 요약

Endpoint	역할	위험도	호출 주체

/sync	지정된 일부만 갱신	✅ 안전	일반 운영 또는 자동화 스크립트
/sync_all	Mongo 전체 → Redis 전체	⚠️ 주의 필요	관리자 전용 (cron, 수동 init)



---

✅ 장점 요약

/sync 와 /sync_all 분리 → 운영상 안전

/sync 는 다중 요청에서도 빠른 batch 처리

/sync_all 은 전체 초기화용으로만 사용 (명시적 관리 대상)

구조 단순, 실시간 락 안전성 확보



---

이 버전이면 운영 중 실수로 Redis 전체를 덮어쓸 일이 없다.
원하면 /sync_all 실행 시 관리자 키나 토큰을 검증하도록(예: header X-Admin-Key) 보안도 추가해줄 수 있다.
그 옵션도 추가할까?
