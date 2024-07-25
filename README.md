## 1. 주요 링크

- S3 버킷 웹사이트 엔드포인트: http://hanghae-plus.s3-website-ap-southeast-2.amazonaws.com/
- CloudFront 배포 도메인 이름: https://d2b2equ8xsbpmd.cloudfront.net/

## 2. Architecture

![](/ci-cd-architecture.png)

## 3. Github Action

Github Action은 빌드, 테스트 및 배포 파이프라인을 자동화할 수 있는 지속적 통합 및 지속적 배포 플랫폼입니다.

### Github Actions 채택 이유

Background

- 신규 프로젝트 개발중
- 변경사항이 빠르게 일어나는 편

저희는 지금 신규 서비스를 개발해야하는 상황이고, 빠르게 CI/CD 시스템을 구축해야 합니다.

기존에 Jenkins를 써오고 있는 서비스라면 굳이 Github Action으로 마이그레이션 할 필요는 없습니다. 그러나 지금은 빠른 CI/CD를 구축하기 위해 Github Action을 채택하게 되었습니다.

제가 Github Action을 채택하게 된 이유를 설명하겠습니다. [Jenkins VS Github Action](https://nanbuja.com/entry/CICD-Jenkins-%EA%B3%BC-GitHub-Action%EC%9D%98-%EA%B0%9C%EB%85%90-%EC%B0%A8%EC%9D%B4%EC%A0%90)

1. **CI/CD 구축 편의성**

Jenkins와 큰 차이점은 Github Actions는 별도의 서버 세팅없이 script 작성으로 구축할 수 있습니다.  
클라우드에서 동작하기 때문에 별도의 설치, 서버 세팅이 필요없어, 그만큼 리소스를 줄일 수 있습니다.

2. **접근성**

Github 레포지토리 내부에서 yml을 작성하여 workflow를 자동화 할 수 있습니다.  
또한 Github와 호환성이 쉬워, 코드리뷰/리뷰어 등록/코멘트 등록과 같은 자동화를 쉽게 구축할 수 있습니다.

3. **코드 확장성**

Github Marketplace, 여러 오픈소스 코드를 참고하여 script를 작성할 수 있습니다.  
유사한 workflow 세팅이 필요하다면 기존에 작성했던 script 파일을 재사용할 수 있어 시간을 절약할 수 있습니다.

4. **단점/보완점**

물론 Jenkins 보다 문서가 적지만, Github 오픈소스를 통해 여러 scripts를 참고할 수 있습니다.  
또한 Jenkins는 캐싱 메커니즘을 지원하기 위해 플러그인을 사용하여 간편하게 구축할 수 있지만,
Github Action은 직접 캐싱 메커니즘을 작성해야합니다.

캐싱 메커니즘을 구현하기 위해 리소스가 많이 필요하게 될 수도 있지만, 여러 오픈소스를 참고하여 구축한다면 이후 다른 서비스에서 캐싱 메커니즘은 쉽게 구축할 수 있기 때문에 보완할 수 있습니다.

<br>

## 4. Action script 분석

제가 현재 작성한 Actions script에 대해 설명드리겠습니다.

1. **action trigger**

`on:` 은 action이 언제 실행될지 정의합니다.
현재는 push -> branch가 -> main일때 실행되도록 설정하였습니다.

```yaml
on:
  push:
    branches:
      - main
```

실제 서비스를 개발하게 된다면, 바로 main에 push할 일은 없을테니
수정하자면 다음과 같이 구축 가능합니다.

```yaml
on:
  pull_request:
    types:
      - opened
      - reopened
      - synchronize
```

pull_request를 생성한 목적은 내 코드가 base 브랜치에 merge 되기 위함입니다.  
또한 내 코드를 배포하기 위하여 코드를 확인하기 위해 PR을 생성합니다.

그러니 pull_request 생성할 때 workflow를 실행시켜 불필요한 workflow 실행 비용을 줄입니다.

자세히 살펴보면 types는 어떤 상황일때 실행하는지 정의합니다.

- opened: pr이 오픈되었을 떄
- reopened: pr이 close 되었다가 다시 open 되었을 때
- synchronize: pr이 생성되고 커밋하였을 떄

[Github 공식 문서 - Events that trigger workflows](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)

<br>

2. **jobs**

jobs는 특정 runner에서 실행되는 일련의 step 입니다.

```yaml
jobs:
  job_name:
    runs-on: 실행환경
    steps:
      - name: Step1
      run: ~~
      - name: Step2
      run: ~~
```

기본적으로 병렬적으로 실행되고, needs를 사용하여 순차적으로 실행하게 할 수도 있습니다.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: npm run build

  test:
    needs: build # build가 실행된 후에 test 실행
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Test
        run: npm test
```

현재 구성된 job으로는 deploy(배포)뿐이고 내부에서 세부 스텝별로 배포 과정을 거치고 있습니다.  
추후 Lint, test, Lighthouse, Github Comment 등 다양한 Job을 추가할 수 있습니다.

<br>

3. **Checkout repository**

처음으로 Github 저장소의 코드를 워크플로우 실행환경으로 가져옵니다.

```yaml
- name: Checkout repository
  uses: actions/checkout@v4
```

name은 step 명칭을 뜻하고, uses는 워크플로우를 실행하기 위한 코드단위인 action을 가져오는 역할입니다.  
다른 사람이 만든 액션을 사용하거나 내 저장소에 있는 로컬 액션을 재사용할 수 있게 해줍니다.

[Actions Marketplace](https://github.com/marketplace?type=actions)에 여러 action이 존재하고, 해당 action을 uses를 통해 재사용할 수 있습니다.

여기서 사용된 actions/checkout 액션은 github 저장소의 코드를 가져오도록 합니다.

<br>

4. **Setup Node.js & Install dependencies & Build**

```yaml
- name: Set up Node.js
  uses: actions/setup-node@v4
  with:
    node-version: "18.x"
- name: Install dependencies
  run: npm ci
- name: Build
  run: npm run build
```

Nodejs 환경을 설정합니다. with 구문에서 특정 버전을 지정합니다.  
또한 의존성을 설치하고 빌드를 실행합니다.

여기서 선언된 run은 셸에서 실행할 명령어를 지정합니다.

<br>

5. **Configure AWS credentials**

AWS CLI를 사용하기 위해 AWS 자격 증명을 설정하는 단계입니다.

`aws-actions/configure-aws-credentials@v1` 를 액션을 사용하여 엑세스 키, 시크릿 키, 리전 정보를 설정합니다.  
이때 토큰 값은 Github Repository > Settings > Secret > action 내에 저장해야 합니다.

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v1
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ secrets.AWS_REGION }}
```

<br>

6. **Deploy to S3**

빌드된 파일을 S3 버킷에 배포합니다.  
`aws s3 sync` 명령어를 사용하여 `out/` 폴더 내용을 S3 버킷과 동기화합니다.

```yaml
- name: Deploy to S3
  run: |
    aws s3 sync out/ s3://${{ secrets.S3_BUCKET_NAME }} --delete
```

`--delete` 옵션

- 소스 디렉토리(out/)와 S3 버킷 내용을 비교합니다.
- 소스 파일에 있는 S3는 업로드하거나 갱신합니다.
- S3에는 있지만 소스에는 없는 파일은 불필요하기 때분에 삭제합니다.

이때 필요한 설정은 next.config.mjs 파일에 output을 설정해주어야 합니다.

```mjs
/** @type {import('next').NextConfig} */
const nextConfig = { output: "export" };

export default nextConfig;
```

해당 설정을 해야 빌드된 파일이 `out/`폴더에 생성됩니다.

<br>

7. **Invalidate CloudFront cache**

CloudFront의 캐시를 무효화합니다.

```yaml
- name: Invalidate CloudFront cache
  run: |
    aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"
```

`aws cloudfront create-invalidation `명령어를 사용하여 지정된 CloudFront 배포의 모든 경로("/\*")에 대한 캐시를 무효화합니다.

CloudFront는 콘텐츠를 캐시하여 성능을 향상시키지만, 이로 인해 업데이트된 콘텐츠가 즉시 반영되지 않을 수 있게 됩니다.  
그런 이슈를 방지하기 위해 캐시무효화를 통해 CloudFront가 S3에서 최신 콘텐츠를 가져올 수 있도록 설정합니다.

<br>

8. **Finally...**

모든 step이 마무리되면 배포가 완료됩니다.

![](/action-step.png)

## 5. ++ Route53

Background

- 서비스를 운영하기 위해 도메인 https://www.yoosion.com 을 구입하였음

현재 저희가 배포한 웹 주소는 https://d2b2equ8xsbpmd.cloudfront.net/ 이런식으로 구성되었습니다.  
제가 직접 지정한 것이 아니라 AWS에서 지정해주었습니다.

추후에 서비스만의 도메인을 설정하여 유저에게 서비스 접근성을 향상시켜줄 수 있습니다.

이때 도메인을 연결하기 위해 Route53을 사용할 수 있습니다.

> Route53의 핵심은 도메인 네임 서버를 임대해주는 역할입니다. -> DNS

Route 53의 역할

- DNS(Domain Name System) 서비스: 도메인 이름을 IP 주소로 변환합니다.
- 도메인 등록: 새로운 도메인을 직접 등록할 수 있습니다.
- CloudFront에 Route53을 연동하여 도메인 네임으로 호스팅할 수 있습니다.

AWS에서 Route 53을 통해 도메인을 설정하는 방법은 해당 [레퍼런스](https://inpa.tistory.com/entry/AWS-%F0%9F%93%9A-Route-53-%EA%B0%9C%EB%85%90-%EC%9B%90%EB%A6%AC-%EC%82%AC%EC%9A%A9-%EC%84%B8%ED%8C%85-%F0%9F%92%AF-%EC%A0%95%EB%A6%AC)에 나와있습니다.
