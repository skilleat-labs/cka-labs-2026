# CKA 실습 가이드

스킬잇 CKA 온라인 강의의 따라 하기 실습 사이트 — https://skilleat-labs.github.io/cka-labs-2026/

- 채점 실습(`skilleat-labs/cka-lab`)과 짝을 이룬다. 페이지 번호 = 강의 영상 번호 = 채점 세트 번호
- 예제 리소스는 `guide-*` 네임스페이스를 쓰고, 채점 문제와 **다른 이름**을 쓴다 (답이 되지 않게)

```bash
pip install -r requirements.txt
mkdocs serve        # http://127.0.0.1:8000/cka-labs-2026/
```

`main` 에 push 하면 GitHub Actions 가 `gh-pages` 브랜치로 배포한다.
