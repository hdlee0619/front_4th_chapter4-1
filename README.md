# 프론트엔드 파이프라인

## 배포 과정

### 배포 파이프라인 다이어그램

![Image](https://github.com/user-attachments/assets/c86ca133-45a0-44ce-bd17-5007bb62e77e)

### 설명

1. 개발자 →  Main Branch로 코드 업데이트
   - 개발자가 로컬에서 작업한 후, 코드를 main 브랜치에 머지(merge)합니다.
2. GitHub Actions →  Main Branch로 코드 업데이트 감지
   - Git 저장소에 main브랜치로 변경이 머지되면, GitHub Actions가 자동으로 워크플로우(빌드 및 배포 파이프라인)를 실행합니다.
   - GitHub Actions에서 Next.js 빌드를 수행하여 최종 산출물을 out 폴더에 생성합니다.
3. S3 Bucket에 배포
   - GitHub Actions에서 AWS CLI를 사용하여 S3 버킷에 out 폴더의 파일을 업로드합니다.
   - 이때 S3는 정적 사이트 호스팅 또는 단순 객체 저장소로서 사용할 수 있습니다.
4. CloudFront와 연동
   - S3에 업로드된 파일을 CloudFront가 가져와서 CDN(콘텐츠 전송 네트워크)을 통해 전 세계 엣지 로케이션에 배포합니다.
   - 이를 통해 사용자들이 더 빠르게 정적 파일에 접근할 수 있습니다.
5. 사용자(User) → CloudFront
   - 최종 사용자는 CloudFront의 HTTPS 도메인을 통해 웹사이트에 접속합니다.
   - 요청은 CloudFront를 거쳐 S3 버킷에 있는 정적 파일을 전달받게 됩니다.

## 주요 링크

S3 버킷 웹사이트 엔드포인트 : http://hanghae-chapter4-3.s3-website.ap-northeast-2.amazonaws.com

CloudFront 배포 도메인 이름 : https://d3ksk4tg75lhhm.cloudfront.net

## 주요 개념

### GitHub Actions과 CI/CD 도구
**CI/CD 도구?**
<br/>개발과 배포 과정을 자동화를 위한 도구입니다.
- Continuous Integration(지속적 통합): 개발자들이 코드를 자주 병합(Integration)하고, 그때마다 자동으로 빌드와 테스트를 수행하여 문제를 빠르게 발견하도록 돕습니다.
- Continuous Deployment(지속적 배포): 테스트가 통과된 코드를 자동으로 운영 환경(Production) 또는 스테이징 환경에 배포(Deployment)하여 반복되는 수작업을 최소화합니다.

이번 과정에서는 빌드하는 과정만 자동화 하였습니다.

**GitHub Actions?**
GitHub에서 제공하는 자동화된 워크플로우/파이프라인 구축 도구입니다.
- 지정해둔 Repository에 푸시(push)나 풀 리퀘스트(PR)가 생성되는 등의 이벤트가 발생했을 때, 미리 정의해 둔 작업(테스트, 빌드, 배포 등)을 자동으로 실행할 수 있습니다.

### S3와 스토리지

**S3?**
AWS에서 제공하는 객체 스토리지 서비스입니다. 스토리지를 “버킷(Bucket)” 단위로 관리하며, 웹 애플리케이션에서 정적(Static) 자산(이미지, CSS, JavaScript 파일 등)을 저장하고 전달하는 데 자주 쓰입니다.

-  S3는 용량 제한 없이 빠르고 안정적으로 데이터를 저장하고 제공할 수 있어 정적 웹사이트 호스팅이나 백업, 로그 저장 등 다양한 용도로 사용됩니다.

이번 과정에서는 웹사이트를 호스팅하기 위해 사용하였습니다.

### CloudFront와 CDN

**CDN(Content Delivery Network)?**
콘텐츠(정적 파일, 미디어, 웹 페이지 등)를 지리적으로 분산된 서버(엣지 로케이션)에 캐싱하여, 사용자와 물리적으로 가까운 위치의 서버에서 콘텐츠를 전달함으로써 응답 시간을 단축하고 트래픽 부하를 분산시키는 기술입니다.
이 CDN을 AWS에서는 CloudFront라는 이름으로 제공합니다.

### 캐시 무효화(Cache Invalidation)

**캐시 무효화(Cache Invalidation)**
<br/>기존에 캐싱되어 있는 파일이나 데이터를 ‘유효하지 않다’고 선언해, CDN의 엣지 서버에서 새로운 버전(업데이트된 콘텐츠)으로 대체하도록 하는 작업입니다.

deployment.yml 파일의 순서를 참고해보면
1. 필요 dependency 설치
2. 빌드 (Next.js 빌드)
3. S3로 파일 업로드 (S3에 out 폴더의 파일 업로드)
4. CloudFront 캐시 무효화 (CloudFront에 업로드된 파일 캐시 무효화)

여기서 캐시 무효화를 하는 시점이 **새로운 파일이 성공적으로 S3에 업로드된 직후**라는 점이 중요합니다. 
<br />
새로운 파일이 아직 S3에 올라가지 않은 시점에 캐시 무효화를 먼저 해버리면, CDN 입장에서는 **아직 기존 파일이 최신**이라고 보고, 실제 배포가 반영되지 않을 수 있습니다.
<br />
따라서 업로드 완료 → 캐시 무효화 순서로 진행되어야, 최종적으로 사용자들이 CDN을 통해 **실제로 존재하는 최신 파일**을 제공받게 됩니다.
<br />
또한, 장애 상황이나 급한 버그 패치가 있을 때는 **급하게 새로운 파일(패치 버전)을 다시 빌드해서 S3에 올린 뒤, 즉시 캐시 무효화**를 진행해야 사용자에게 빠르게 반영될 수 있습니다. 
이처럼 캐시 무효화는 단순히 정적 파일 배포 순서 중 하나일 뿐 아니라, 장애 대응이나 긴급 배포 시에도 매우 중요한 단계가 됩니다.

### Repository secret과 환경변수

**Repository Secret**
<br />GitHub Actions 또는 다른 CI/CD 환경에서 민감한 정보를 안전하게 저장하고, 워크플로우 실행 시 필요한 값(API 키, 비밀번호, 접근 토큰 등)을 제공하는 기능입니다.
Secrets은 GitHub 레포지토리 설정에서 등록 가능하며, 일반 사용자는 암호화된 상태로만 확인할 수 있고, Actions에서만 해당 값에 접근할 수 있습니다.

**환경변수(Environment Variable)**
<br />애플리케이션 실행 환경에서 동적으로 참조되는 값들을 말합니다. 예를 들어, 서버 URL, 포트 번호, 데이터베이스 연결 정보 등을 환경변수로 관리하면, 코드 수정 없이도 배포 환경(개발/테스트/운영)에 맞게 설정할 수 있습니다.

# CDN과 성능 최적화

위의 설명에서 보았듯이 CDN은 콘텐츠를 사용자와 물리적으로 가까운 위치의 서버에서 전달함으로써 응답 시간을 단축하고 트래픽 부하를 분산시키는 역할을 합니다.
도로를 넓혀 통행량을 늘리는 것과 같은 역할을 하게 하여 성능을 최적화 시킵니다.

## TTFB(Time To First Byte) 최적화 결과

### Before (S3)
CDN적용 없이 S3 버킷에 직접 접근했을 때의 TTFB 결과입니다.
![Image](https://github.com/user-attachments/assets/f3838eee-dde4-473c-912b-2147eeac366a)

### After (CloudFront)
CloudFront를 통해 콘텐츠를 전달받았을 때의 TTFB 결과입니다.
![Image](https://github.com/user-attachments/assets/e128dcec-9e02-4a45-9489-1c51c284b059)

### 요약
- S3의 TTFB: 22.41ms
- CloudFront의 TTFB: 7.36ms
<br /> CDN을 적용한 것만으로 67% 정도 성능이 개선되었음을 알 수 있습니다.

## 캐싱

![Image](https://github.com/user-attachments/assets/01fc96f3-fb5c-4024-973f-990b5e8b3917)

이미지가 잘 안보일 수 있는데 빨간색 부분이 S3이고, 파란색 부분이 CDN(CloudFront)입니다.
보면 알 수 있듯이 CDN을 사용하면 Header에 `Content-Encoding: br`과 `X-Cache: Hit from cloudfront`가 추가되어 있습니다.

<br/>`Content-Encoding: br`은 구글에서 개발한 Brotli 압축 알고리즘을 사용했다는 것을 의미하며 일반적으로 gzip보다 더 높은 압축률을 제공해 파일 크기를 작게 만들 수 있습니다.
<br/>`X-Cache: Hit from cloudfront`는 해당 파일이 이미 CloudFront의 캐싱된 콘텐츠를 가져와 응답했다는 의미입니다.


