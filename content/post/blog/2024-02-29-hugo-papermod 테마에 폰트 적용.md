---
title: hugo-papermod 테마에 새로운 글꼴 적용
description:
date: 2024-02-29
summary: hugo-papermod 테마에 새로운 글꼴 적용
---

## 0. 서론

ssg(static site generator) 중 하나인 hugo에 글꼴을 적용해 보겠습니다.

## 1. 환경

- golang: 1.21.7
- hugo: 0.122.0+extended
- theme: hugo-PaperMod 7.0

## 2. 구축

[hugo server](https://gohugo.io/getting-started/usage/#develop-and-test-your-site)는 liveReload를 통해 파일이 변경되면 자동으로 브라우저를 새로고침합니다. 그러나 Fast Render Mode로 전부 교체하는 것이 아닌 변경된 일부만 교체되는 경우가 있기 때문에 글꼴 변경 확인에 혼돈을 줄 수 있습니다. 그러므로 안정적으로 글꼴 변경을 확인하고 싶으면 다음과 같이 hugo server를 실행해 주세요.

```sh
hugo server --disableFastRender
```

설정 전 디렉토리 트리는 다음과 같습니다.

```txt
├── archetypes
├── content
├── server
├── static
├── themes
│   └── PaperMod
│       └── assets
│           └── css
│               └── extended
└── config.yaml
```

static 디렉토리에 글꼴을 적용할 ttf 파일과 fonts.css 파일 생성하겠습니다. static 디렉토리는 stylesheets, js, 사진 등 여러 파일들을 놓고 사용하기 떄문에 fonts라는 이름의 디렉토리를 생성 후 ttf 파일과 fonts.css를 생성하겠습니다. 저는 D2Coding 글꼴을 사용하겠습니다.

```txt
.
├── archetypes
├── content
├── server
├── static
│   └── fonts
│       ├── D2Coding-Ver1.3.2-20180524.ttf
│       └── fonts.css
├── themes
│   └── PaperMod
│       └── assets
│           └── css
│               └── extended
└── config.yaml
```

fonts.css는 다음과 같이 작성합니다.

```css
@font-face {
    font-family: 'D2Coding';
    src: url('D2Coding-Ver1.3.2-20180524.ttf') format('truetype');
}
```

[hugo-PaperMod](https://github.com/adityatelange/hugo-PaperMod/wiki/FAQs#bundling-custom-css-with-themes-assets)는 사용자 정의 CSS를 추가할 디렉토리(PaperMod/assets/css/extended)를 제공합니다. 다음 위치에 fonts.css 추가 후 커스텀하게 글꼴을 적용하면 됩니다.

```css
body {
    font-family: D2Coding;
}
```

설정 후 디렉토리 트리는 다음과 같습니다.

```txt
.
├── archetypes
├── content
├── server
├── static
│   └── fonts
│       ├── D2Coding-Ver1.3.2-20180524.ttf
│       └── fonts.css               # @font-face {}
├── themes
│   └── PaperMod
│       └── assets
│           └── css
│               └── extended
│                   └── fonts.css   # body {}
└── config.yaml
```
