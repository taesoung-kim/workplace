FastAPI 기반 GitHub 연동 API 전체 정리 (샘플 코드 포함)


---

1. 개요

GitHub REST API를 사용하는 FastAPI 서버를 구축하여 다음을 수행한다.

1. 특정 이름의 Repository가 존재하는지 확인


2. 없을 경우 템플릿 기반으로 자동 생성


3. 사용자로부터 .py 파일 코드를 받아 새 브랜치 생성 및 커밋


4. PR 생성 직전까지 자동화 후 URL 반환




---

2. 엔드포인트 개요

기능	Endpoint	메서드	설명

Repository 동기화	/github/repo/sync	POST	Repo 존재 여부 확인 후 필요 시 템플릿 기반 생성
파일 업로드 및 커밋	/github/repo/push	POST	.py 파일을 받아 새 브랜치 생성, 커밋 후 PR URL 반환



---

3. /github/repo/sync 구현

요청 모델

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import httpx
import os

app = FastAPI()

GITHUB_TOKEN = os.getenv("GITHUB_TOKEN")
OWNER = "your-org"             # 실제 조직명
TEMPLATE_OWNER = "your-org"    # 템플릿 저장소 소유자
TEMPLATE_REPO = "template-repo"

class RepoRequest(BaseModel):
    name: str

엔드포인트 코드

@app.post("/github/repo/sync")
async def sync_repo(req: RepoRequest):
    headers = {
        "Authorization": f"token {GITHUB_TOKEN}",
        "Accept": "application/vnd.github.v3+json"
    }

    async with httpx.AsyncClient() as client:
        # 1. Repo 존재 여부 확인
        repo_url = f"https://api.github.com/repos/{OWNER}/{req.name}"
        resp = await client.get(repo_url, headers=headers)

        if resp.status_code == 200:
            data = resp.json()
            return {"exists": True, "url": data["html_url"]}

        elif resp.status_code == 404:
            # 2. 템플릿 기반 Repo 생성
            gen_url = f"https://api.github.com/repos/{TEMPLATE_OWNER}/{TEMPLATE_REPO}/generate"
            payload = {
                "owner": OWNER,
                "name": req.name,
                "include_all_branches": False,
                "private": True
            }
            create_resp = await client.post(gen_url, headers=headers, json=payload)

            if create_resp.status_code in [200, 201]:
                data = create_resp.json()
                return {"exists": False, "created": True, "url": data["html_url"]}
            else:
                raise HTTPException(status_code=create_resp.status_code, detail=create_resp.text)

        else:
            raise HTTPException(status_code=resp.status_code, detail=resp.text)

사용 예시

입력

{ "name": "rise-automation-service" }

출력

{ "exists": false, "created": true, "url": "https://github.com/your-org/rise-automation-service" }


---

4. /github/repo/push 구현

요청 모델

from pydantic import BaseModel
import base64

class PushRequest(BaseModel):
    branch_name: str
    file_path: str
    file_content: str  # 전체 Python 코드(base64 인코딩 권장)
    commit_message: str

엔드포인트 코드

@app.post("/github/repo/push")
async def push_to_repo(req: PushRequest):
    headers = {
        "Authorization": f"token {GITHUB_TOKEN}",
        "Accept": "application/vnd.github.v3+json"
    }

    async with httpx.AsyncClient() as client:
        # 1. main 브랜치 SHA 조회
        ref_url = f"https://api.github.com/repos/{OWNER}/{REPO}/git/ref/heads/main"
        ref_resp = await client.get(ref_url, headers=headers)
        if ref_resp.status_code != 200:
            raise HTTPException(status_code=ref_resp.status_code, detail=ref_resp.text)

        main_sha = ref_resp.json()["object"]["sha"]

        # 2. 새 브랜치 생성
        new_branch_ref = f"refs/heads/{req.branch_name}"
        create_ref_url = f"https://api.github.com/repos/{OWNER}/{REPO}/git/refs"
        ref_payload = {"ref": new_branch_ref, "sha": main_sha}
        ref_create_resp = await client.post(create_ref_url, headers=headers, json=ref_payload)

        if ref_create_resp.status_code not in [200, 201]:
            raise HTTPException(status_code=ref_create_resp.status_code, detail=ref_create_resp.text)

        # 3. 파일 커밋
        encoded_content = base64.b64encode(req.file_content.encode("utf-8")).decode("utf-8")
        commit_url = f"https://api.github.com/repos/{OWNER}/{REPO}/contents/{req.file_path}"
        commit_payload = {
            "message": req.commit_message,
            "content": encoded_content,
            "branch": req.branch_name
        }
        commit_resp = await client.put(commit_url, headers=headers, json=commit_payload)
        if commit_resp.status_code not in [200, 201]:
            raise HTTPException(status_code=commit_resp.status_code, detail=commit_resp.text)

        # 4. Pull Request URL 반환
        pr_preview_url = f"https://github.com/{OWNER}/{REPO}/compare/main...{req.branch_name}?expand=1"

        return {
            "branch": req.branch_name,
            "commit_done": True,
            "pr_url": pr_preview_url
        }

사용 예시

입력

{
  "branch_name": "feature/add_hello",
  "file_path": "src/hello.py",
  "file_content": "print('hello world')",
  "commit_message": "Add hello.py"
}

출력

{
  "branch": "feature/add_hello",
  "commit_done": true,
  "pr_url": "https://github.com/your-org/target-repo/compare/main...feature/add_hello?expand=1"
}


---

5. file_content 전송 제약

구분	설명

FastAPI 제한	없음 (Python 문자열 그대로 수신 가능)
NGINX 등 프록시	client_max_body_size 기본 1 MB → 10~50MB로 확장 필요
권장 방식	Base64 인코딩 후 JSON 전송
대용량 파일	multipart/form-data(UploadFile) 방식 사용


요약 기준

파일 크기	권장 방식

< 10 KB	일반 문자열
10 KB ~ 5 MB	Base64 문자열
> 5 MB	multipart 업로드



---

6. Python 파일 크기 참고

라인 수	평균 길이	예상 크기

10,000줄	60~100자/줄	약 0.6~1.2 MB
한글·주석 많을 경우		약 2 MB 이내


→ JSON 전송에 충분히 안전하며 GitHub의 단일 파일 제한(100 MB)보다 훨씬 작음.


---

7. 확장 포인트

.env 기반 설정 (TOKEN, OWNER, TEMPLATE_REPO 등)

GitHub Enterprise 지원 (api.github.company.com)

예외 로깅 / 재시도 정책

자동 PR 생성 (POST /repos/{owner}/{repo}/pulls)



---

✅ 결론
FastAPI + GitHub REST API 조합으로

Repository 자동 생성

새 브랜치 생성 및 커밋

PR URL 자동 반환
까지 완전 자동화 가능하며,
.py 파일 10,000줄 이하는 JSON(Base64)로 안전하게 전송 가능하다.
