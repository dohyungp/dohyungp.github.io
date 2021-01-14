---
layout: default
title: fsevents 지옥에 빠져버렸다
---

## fsevents 지옥에 빠져버렸다

### 사건의 시작
가끔 터미널에서 작업할때는 neovim이라는 vim을 쓰는 편인데, 이 친구가 오류가 나는 것을 그동안 두고 봤다가, 기분이 내킨 날 `brew upgrade neovim`으로 업데이트를 해줬습니다(~그러지 말았어야 했습니다~).
이후 `create-react-app`으로 만드는 새로운 프로젝트들에서 어떤 시도를 해도 아래의 오류만 뱉어낼 뿐 동작하지 않았습니다.

```shell
dyld: lazy symbol binding failed: Symbol not found: _FSEventStreamCreate
  Referenced from: /Users/dohyung/Workspace/MyProject/mindyui-test/mindyland/node_modules/fsevents/build/Release/fse.node
  Expected in: flat namespace

dyld: Symbol not found: _FSEventStreamCreate
  Referenced from: /Users/dohyung/Workspace/MyProject/mindyui-test/mindyland/node_modules/fsevents/build/Release/fse.node
  Expected in: flat namespace
```

찾아보니, [이런 이슈](https://github.com/fsevents/fsevents/issues/313)가 reporting되어 있었고 사람들이 추천하는데로,

1. node, yarn을 재설치하기
2. nvm으로 버전을 변경해서 테스트해보기
3. xcode-select 삭제하고 재설치하기
4. npm cache clean && yarn cache clean

등등

여러가지 방법을 시도해봤습니다만 이 지옥에서 탈출이 되지 않았습니다(~개인의 실력부족으로~).

최종 확인한 것은 `npx create-react-app`으로 설치하는 fsevents가 node@14 ~ latest까지 지원하는 v2가 아닌 v1이라는 사실이었고 이를 수정해보기 위해 다양한 방법을 시도했습니다만 더이상 삽질할 여유가 없었던 까닭에 아래와 같은 방법으로 해결하기로 했습니다.

### 슬픈 결말

다행히 저는 최근에 docker로 개발환경을 맞추는 걸 선호하는 편이었으므로 아래와 같이 docker-compose.yml을 하나 만들고 프로젝트 생성 시에만 사용하기로 했습니다.

```
version: "3"

services:
  react:
    image: node:14
    ports:
      - 3000:3000
    stdin_open: true
    tty: true
    volumes:
      - .:/app/
```

다시 여유가 있다면 fsevents 이슈를 해결해보고 싶은데 여유가 없을 듯 합니다.
