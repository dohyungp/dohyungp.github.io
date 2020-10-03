---
layout: default
title: github action에서 github workflow 권한이 없어서 push가 안될때
---

## github action에서 github workflow 권한이 없어서 push가 안될때

팀원분이 작성한 github action이 build에 실패해서 `workflows/docker-publish.yml`를 수정했는데 push가 안되더군요. 사용 중인 vscode가 workflow permission이 없던 상태였기 때문인데 이럴때에는,

```shell
git config --local credential.helper ""
```

로 shell에서 configuration을 초기화해주면 됩니다.
