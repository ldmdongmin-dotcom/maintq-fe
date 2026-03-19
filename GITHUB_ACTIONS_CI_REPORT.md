# GitHub Actions CI Pipeline 실습 보고서

## 1. 실습 주제

이번 실습의 주제는 아래와 같다.

`GitHub 내 소스코드 -> 코드 빌드 -> 이미지 빌드 -> Harbor Registry에 이미지 Push 하는 CI 구성하기`

즉, GitHub Actions를 활용하여 소스코드 변경이 발생했을 때 자동으로 프론트엔드 애플리케이션을 빌드하고, Docker 이미지를 생성한 뒤 Harbor Registry에 push 하는 CI 파이프라인을 구성하는 것이 목표였다.

---

## 2. 실습 목적

이번 실습을 통해 다음 내용을 익히는 것을 목표로 했다.

| 항목 | 내용 |
| --- | --- |
| GitHub Actions | 워크플로 기본 구조 이해 |
| 이벤트 기반 자동화 | 어떤 이벤트에서 CI를 실행할지 설계하는 방식 이해 |
| Node.js 빌드 | 프론트엔드 프로젝트의 빌드 자동화 |
| Docker | 이미지를 활용한 애플리케이션 패키징 |
| Harbor | 이미지 저장소 연동 방식 학습 |
| CI 설계 | 실무 형태의 자동화 파이프라인 구성 방법 학습 |

---

## 3. 실습 환경

| 항목 | 값 |
| --- | --- |
| Repository | `maintq-fe` |
| Workflow file | [`.github/workflows/ci.yml`](/Users/ldm/SKALA/ldm_wsl/skala-DevOps/DevOps_2day/maintq-fe/.github/workflows/ci.yml) |
| Frontend source | [`FE`](/Users/ldm/SKALA/ldm_wsl/skala-DevOps/DevOps_2day/maintq-fe/FE) |
| Package file | [`FE/package.json`](/Users/ldm/SKALA/ldm_wsl/skala-DevOps/DevOps_2day/maintq-fe/FE/package.json) |
| Dockerfile | [`FE/Dockerfile`](/Users/ldm/SKALA/ldm_wsl/skala-DevOps/DevOps_2day/maintq-fe/FE/Dockerfile) |
| Branch | `main` |
| Registry | Harbor |

GitHub Secrets에는 아래 값을 미리 등록해 두고 사용했다.

| Secret 이름 | 용도 |
| --- | --- |
| `HARBOR_REGISTRY` | Harbor 레지스트리 주소 |
| `HARBOR_PROJECT` | Harbor 프로젝트명 |
| `HARBOR_USERNAME` | Harbor 로그인 계정 |
| `HARBOR_PASSWORD` | Harbor 로그인 비밀번호 |

---

## 4. 수행 내용

### 4-1. CI 파이프라인 구성 방향

이번 실습에서는 GitHub 저장소의 소스코드가 변경되었을 때 다음 순서로 동작하도록 파이프라인을 설계했다.

| 순서 | 단계 | 설명 |
| --- | --- | --- |
| 1 | Checkout | 저장소 소스코드를 가져온다 |
| 2 | Node.js Setup | Node.js 실행 환경을 구성한다 |
| 3 | Dependency Install | `npm ci`로 의존성을 설치한다 |
| 4 | Frontend Build | `npm run build`로 프론트엔드 코드를 빌드한다 |
| 5 | Docker Buildx | 이미지 빌드 환경을 구성한다 |
| 6 | Harbor Login | Harbor Registry 인증을 수행한다 |
| 7 | Image Build | Docker 이미지를 생성한다 |
| 8 | Image Push | Harbor Registry에 이미지를 업로드한다 |

이 과정을 통해 사람이 수동으로 빌드와 이미지 업로드를 반복하지 않아도, 코드 변경만으로 동일한 절차가 자동으로 수행되도록 만들었다.

### 4-2. 이벤트 조합

이번 워크플로는 아래 이벤트를 사용했다.

| 이벤트 | 목적 |
| --- | --- |
| `push` | `main` 브랜치에 코드가 반영되었을 때 자동 실행 |
| `pull_request` | PR 생성 또는 갱신 시 코드 빌드 검증 |
| `workflow_dispatch` | GitHub Actions 화면에서 수동 실행 |

이렇게 여러 이벤트를 조합함으로써 자동 실행과 수동 실행을 모두 지원할 수 있도록 구성했다.

### 4-3. 빌드 단계 추가

초기 워크플로는 Docker 이미지 빌드 중심이었기 때문에, 과제 요구사항인 `코드 빌드 -> 이미지 빌드` 흐름이 명확하게 드러나지 않았다. 이를 보완하기 위해 `Set up Docker Buildx` 단계 이전에 아래 단계를 추가했다.

| 추가 단계 | 역할 |
| --- | --- |
| `Set up Node.js` | Node.js 환경 구성 |
| `npm ci` | 의존성 설치 |
| `npm run build` | 프론트엔드 빌드 |

또한 프론트엔드 소스가 `FE` 하위 디렉토리에 존재하기 때문에 아래 옵션도 함께 적용했다.

| 설정 | 적용값 | 목적 |
| --- | --- | --- |
| `working-directory` | `./FE` | 프론트엔드 폴더 기준으로 명령 실행 |
| `cache-dependency-path` | `FE/package-lock.json` | npm 캐시 기준 파일 지정 |

### 4-4. Job 분리

워크플로는 `ci`와 `docker` 두 개의 job으로 분리했다.

| Job | 역할 |
| --- | --- |
| `ci` | 코드 빌드 검증 수행 |
| `docker` | 이미지 빌드 및 Harbor push 수행 |

이처럼 단계를 나누면 파이프라인 구조가 명확해지고, 과제 설명 시에도 "코드 빌드"와 "이미지 배포"를 분리해서 설명할 수 있다는 장점이 있다.

### 4-5. PR과 Push 처리 방식

실습 중 PR 이벤트에서도 워크플로가 동작하도록 구성했지만, 실제 Harbor push는 `push` 또는 수동 실행 시에만 수행되도록 설계했다. 이는 실무에서도 자주 사용하는 방식으로, PR 단계에서는 빌드와 검증 위주로 확인하고, 실제 배포성 작업은 브랜치 반영 이후에 수행하는 흐름과 유사하다.

---

## 5. 내가 실습 중에 궁금했던 내용과 정리

### 5-1. 브랜치 설정은 예시 코드 그대로 써야 하는가?

처음 예시 코드에는 `main`, `dev` 브랜치가 들어 있었지만, 실제 내 저장소는 `main` 브랜치를 중심으로 사용하고 있었다. 따라서 `branches` 항목은 예시를 그대로 복사하는 것이 아니라, 현재 저장소에서 실제로 사용하는 브랜치에 맞춰 수정해야 한다는 점을 확인했다.

정리하면 다음과 같다.

| 상황 | 적용 방법 |
| --- | --- |
| 저장소가 `main`만 사용 | `main`만 설정 |
| 개발 브랜치가 따로 존재 | 해당 브랜치 추가 |
| 예시 코드 사용 시 | 실제 브랜치 전략에 맞게 수정 |

### 5-2. 여러 이벤트 조합은 왜 필요한가?

과제에서 "여러 이벤트 조합"이라는 표현이 있었기 때문에 단순히 `push`만 사용하는 것보다 `push`, `pull_request`, `workflow_dispatch`를 함께 사용하는 쪽이 더 적절했다.

이를 통해 다음 효과를 얻을 수 있다.

| 이벤트 | 기대 효과 |
| --- | --- |
| `push` | 실제 브랜치 반영 시 자동 실행 |
| `pull_request` | 병합 전 빌드 검증 |
| `workflow_dispatch` | 수동 테스트 및 시연 가능 |

즉, 여러 이벤트를 조합하면 파이프라인 활용 범위가 더 넓어지고, 과제 설명에서도 설계 의도를 보여주기 좋다.

### 5-3. `paths`는 왜 넣었다가 뺐는가?

처음에는 `paths` 옵션을 넣어 특정 폴더나 워크플로 파일이 변경될 때만 CI가 실행되도록 제한했다. 이는 불필요한 실행을 줄인다는 장점이 있다. 다만 이번 과제는 기능 구현과 흐름 설명이 더 중요했고, 설정이 단순한 편이 제출용으로 설명하기 더 쉬웠기 때문에 최종적으로 `paths`는 제거했다.

즉, `paths`는 잘못된 설정이 아니라 선택 사항이며, 이번 실습에서는 단순성과 설명 편의성을 위해 제외했다.

### 5-4. 왜 Docker Buildx 앞에 Node.js 빌드 단계를 넣었는가?

과제 요구사항은 단순히 이미지를 만드는 것이 아니라, 먼저 소스코드를 빌드한 뒤 이미지를 생성하는 흐름을 구성하는 것이었다. 그래서 Docker 단계 이전에 `Node.js 설정 -> npm ci -> npm run build` 단계를 추가했다.

이 변경으로 인해 워크플로 흐름이 다음처럼 더 명확해졌다.

`소스코드 반영 -> 코드 빌드 -> 이미지 빌드 -> Harbor Push`

---

## 6. 트러블슈팅

### 6-1. `git commit`이 되지 않았던 문제

실습 중 아래와 같은 메시지가 발생했다.

```bash
Changes not staged for commit
no changes added to commit
```

원인은 수정한 파일을 `git add` 하지 않은 상태에서 바로 `git commit`을 실행했기 때문이다.

해결 방법은 아래와 같다.

```bash
git add .github/workflows/ci.yml
git commit -m "feat: add github actions ci pipeline"
```

또한 작업 위치가 `maintq-fe/FE` 하위 디렉토리였기 때문에 상위 폴더 파일이 `../.github/workflows/ci.yml`처럼 보였는데, 이 점도 처음에는 혼동 요소가 되었다. 이후에는 저장소 루트인 `maintq-fe`에서 Git 작업을 진행하여 해결했다.

### 6-2. 브랜치 설정 혼동

예시 코드 기준으로 `main`, `dev`를 그대로 사용하려 했지만, 실제 저장소 구조와 맞지 않을 수 있다는 점을 확인했다. 브랜치가 맞지 않으면 push 해도 워크플로가 실행되지 않을 수 있기 때문에, 현재 사용 중인 브랜치를 먼저 확인한 뒤 `branches` 항목을 맞추는 것이 중요했다.

이번 실습에서는 `main` 브랜치를 기준으로 설정하여 문제를 방지했다.

### 6-3. 과제 요구사항과 현재 워크플로 구조 불일치

초기 상태의 워크플로는 Docker 이미지 빌드와 push는 잘 수행할 수 있었지만, "코드 빌드 단계"가 별도 단계로 명확히 드러나지 않았다. 과제 요구사항에 맞추기 위해 Node.js 설정과 프론트엔드 빌드 단계를 Docker Buildx 앞에 추가했고, 그 결과 요구사항과 구현 내용이 일치하도록 수정할 수 있었다.

### 6-4. `paths` 사용 여부에 대한 판단

`paths`를 넣으면 특정 변경에만 반응하게 할 수 있어서 효율적이지만, 초반 과제에서는 설정이 복잡해 보일 수 있다. 이번 실습에서는 단순하고 설명하기 쉬운 구조를 선택하기 위해 제거했다. 이 경험을 통해 "좋은 설정"이 항상 "현재 상황에 가장 적합한 설정"과 같은 것은 아니라는 점을 배울 수 있었다.

---

## 7. 더 알면 좋은 내용

### 7-1. `npm install`과 `npm ci`의 차이

| 항목 | 설명 |
| --- | --- |
| `npm install` | 의존성을 비교적 유연하게 설치한다 |
| `npm ci` | `package-lock.json` 기준으로 더 빠르고 재현 가능한 설치를 수행한다 |

CI 환경에서는 빌드 결과를 일정하게 유지하는 것이 중요하므로, 일반적으로 `npm ci`가 더 적합하다.

### 7-2. `workflow_dispatch`의 장점

`workflow_dispatch`를 사용하면 코드를 다시 push 하지 않아도 GitHub UI에서 워크플로를 직접 실행할 수 있다. 이는 시연, 테스트, 재실행 상황에서 매우 유용하다.

### 7-3. Docker metadata action의 역할

`docker/metadata-action`은 이미지 태그를 자동 생성하는 데 도움을 준다. 예를 들어 브랜치명, PR 정보, SHA 기반 태그를 쉽게 만들 수 있다. 실무에서는 `latest` 하나만 쓰기보다 `commit SHA`, `version tag`, `branch tag`를 함께 사용하는 경우가 많다.

### 7-4. Secrets와 Variables 구분

| 구분 | 설명 |
| --- | --- |
| `Secrets` | 비밀번호, 토큰, 계정 정보 등 민감한 값 |
| `Variables` | 환경명, 프로젝트명, 일반 문자열 등 공개 가능한 설정값 |

이번 실습에서는 Harbor 로그인에 필요한 값이 민감하므로 Secrets를 사용했다.

---

## 8. 실무적 관점에서 본 이번 실습

| 관점 | 내용 |
| --- | --- |
| CI의 필요성 | 수동 빌드와 이미지 업로드를 줄이고, 반복 작업을 자동화해 표준화와 품질 확보에 도움을 준다 |
| PR과 배포 분리 | PR은 검증 중심, merge 이후는 배포 중심으로 나누는 것이 일반적이다 |
| 태그 전략 | `latest`만 쓰기보다 SHA나 버전 태그를 함께 사용해 배포 이력을 추적하는 것이 좋다 |
| 캐시 최적화 | npm 캐시와 Docker layer 캐시를 활용하면 실행 시간을 줄일 수 있다 |
| 실패 지점 분리 | job을 분리하면 어느 단계에서 실패했는지 빠르게 파악할 수 있다 |

---

## 9. 최종 결과

이번 실습을 통해 GitHub Actions를 사용하여 프론트엔드 프로젝트의 CI 파이프라인을 구성했다. 워크플로는 `push`, `pull_request`, `workflow_dispatch` 이벤트를 기반으로 동작하며, 소스코드 checkout 이후 Node.js 환경 설정, 의존성 설치, 프론트엔드 빌드, Docker 이미지 빌드, Harbor Registry push 순으로 실행되도록 구성했다.

또한 실습 과정에서 브랜치 설정, staging 후 commit, 이벤트 구성 방식, `paths` 사용 여부, 코드 빌드 단계 분리 같은 요소들을 직접 점검하고 수정하면서, 단순히 예시 코드를 복사하는 것이 아니라 현재 저장소 상황에 맞는 CI 구성을 만드는 경험을 할 수 있었다.

이번 실습은 단순 자동화 설정을 넘어서, 실무에서 왜 CI가 필요한지와 어떤 기준으로 워크플로를 설계해야 하는지를 이해하는 데 의미가 있었다.

---

## 10. 한 줄 결론

이번 실습에서는 GitHub Actions를 활용해 `소스코드 변경 -> 코드 빌드 -> Docker 이미지 빌드 -> Harbor Registry push` 흐름을 자동화하는 CI 파이프라인을 구성했고, 이벤트 조합과 단계 분리를 통해 과제 요구사항과 실무적 관점을 함께 반영했다.
