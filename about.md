---
layout: about
image: /assets/img/blog/hydejack-9.jpg
description: >
  CPS LAB 블로그에 대해서...
hide_description: true
redirect_from:
  - /download/
---

# About

<!--author-->

## CPS LAB 블로그

해당 블로그는 정보 공유등을 위해 개설되었습니다.
{:.lead}

- 배경

기존 노션 등을 이용하는 경우, 제한 된 인원만 접근가능하고 그 외에는 링크를 통해서만 접근할 수 있다.
이 사이트를 통해서 외부(계정이 없는) 경우에도 접근할 수 있고, 인터넷만 있다면 언제든 볼 수 있게 하여서 접근성을 높입니다.

<br>

- 취급 정보

이 사이트에서는 알고리즘 외에 개발에 유용한 정보를 모아둡니다.

<br>


# 글을 작성하기 앞서..

해당 Organization 에 참여해야합니다. 그리고 아래 과정을 통해 로컬 환경을 구성합니다.


원하는 로컬 위치에서...
1. git clone https://github.com/CPS-LAB-JEJU/CPS-LAB-JEJU.github.io.git
2. 클론 받은 경로로 이동
3. 명령어 `gem install bundler` 실행
4. 명령어 `gem install jekyll` 실행
5. 명령어 `bundle install` [^1] 실행
6. 해당 레포지토리 프로젝트의 `_config.yml` 파일에서 theme을 다음과 같이 수정한다.
   <img width="288" alt="Screen Shot 2023-02-22 at 11 48 31 PM" src="https://user-images.githubusercontent.com/47859845/220658626-e9f4cf4f-9e9b-406c-94eb-d16e63a4feab.png">
7. Run `bundle exec jekyll serve`
8. 브라우저에 <http://localhost:4000> 링크 입력 접속

* git push 전에 _config.yml 파일에서 theme을 기존 값으로 돌려주세요. 로컬에서 확인할 때만 4번을 다시 진행하시면 됩니다.


글 작성 시 기존 코드들을 참고하세요. 더 세부적인 사항은 [공식 가이드](https://hydejack.com/docs/)를 참고하세요. {:.lead}