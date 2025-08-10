---
title: .gitignore가 동작하지 않을 때 - 이미 커밋된 파일 완벽하게 제거하기
description: 
author: laze
date: 2025-08-10 00:00:01 +0900
categories: [Dev, Git]
tags: [Git, Staging, Tracked, Untracked]
---
# `.gitignore`가 동작하지 않을 때: 이미 커밋된 파일 완벽하게 제거하기

## 1. 서론: "`.gitignore`에 넣었는데 왜 자꾸 커밋하라고 뜨죠?"

Git으로 프로젝트를 관리하다 보면, `.idea/` 디렉토리, `.iml` 파일, `build/` 산출물처럼 버전 관리에 포함하고 싶지 않은 파일들이 계속해서 커밋 대상(Changes)으로 나타나는 성가신 상황을 마주하게 됩니다. 분명히 `.gitignore` 파일에 해당 파일들을 무시하라고 규칙을 추가했음에도 불구하고 말이죠.

```
# .gitignore
.idea/
*.iml
build/
```

"왜 내 `.gitignore`는 동작하지 않는 걸까?"

이 문제의 원인은 `.gitignore`를 잘못 작성해서가 아닙니다. 바로 Git의 **'추적(Tracking)'** 상태에 대한 미묘한 오해 때문입니다. 이 글에서는 이 문제의 근본 원인을 파헤치고, 이미 Git의 관리 대상이 되어버린 불필요한 파일들을 안전하고 완벽하게 제거하는 3단계 워크플로우를 제시합니다.

## 2. `.gitignore`의 진짜 역할: 'Tracked' vs 'Untracked'

`.gitignore`의 역할에 대한 가장 흔한 오해는, 이 파일에 경로를 적어두기만 하면 Git이 무조건 해당 파일들을 무시할 것이라고 생각하는 것입니다.

**`.gitignore`의 정확한 역할:**

> "아직 Git이 관리하고 있지 않은(Untracked) 파일들 중에서, 여기에 명시된 패턴과 일치하는 파일들은 앞으로도 계속 무시해라."
>

즉, `.gitignore`는 **새로운 파일을 스테이징(`git add`)할 때만 동작**하는 필터입니다.

핵심은 **'Tracked(추적되는)'** 상태입니다. 만약 어떤 파일이 **단 한 번이라도 `git add` 후 `git commit` 되었다면**, 그 파일은 이미 Git의 관리 대상, 즉 'Tracked' 상태가 됩니다. 한번 'Tracked' 상태가 된 파일은, 그 이후에 `.gitignore`에 추가하더라도 Git이 계속해서 변경 사항을 감시합니다. Git 입장에서는 "주인님이 예전에 중요하다고 커밋까지 한 파일인데, 이제 와서 무시할 수는 없지!" 라고 생각하는 것입니다.

## 3. 문제 해결: 3단계 워크플로우

이미 추적되고 있는 불필요한 파일들을 Git의 관리에서 안전하게 제거하고, 앞으로는 자동으로 무시되도록 만드는 명확한 3단계 절차는 다음과 같습니다.

### 1단계: `.gitignore` 파일을 올바르게 작성하기

가장 먼저, 앞으로 무시하고 싶은 모든 파일과 디렉토리 패턴을 `.gitignore` 파일에 정확하게 명시해야 합니다. 이미 작성되어 있다면, 빠진 항목은 없는지 다시 한번 확인합니다.

**Java/Gradle/IntelliJ 프로젝트의 표준 `.gitignore` 예시:**

```
# Build files
.gradle/
build/

# IDE files (IntelliJ)
.idea/
*.iml
*.iws

# Log files
*.log

# OS generated files
.DS_Store
Thumbs.db
```

> Tip: gitignore.io 사이트를 이용하면, 사용하는 운영체제, IDE, 언어/프레임워크에 맞는 표준 .gitignore 파일을 매우 쉽게 생성할 수 있습니다.
>

### 2단계 (핵심): Git 캐시에서만 파일 제거하기

이제 이미 추적되고 있는 파일들을 Git의 관리 대상에서 제외시킬 차례입니다. 이때 사용하는 명령어가 바로 `git rm --cached` 입니다.

`git rm -r --cached <파일/디렉토리>` 명령어의 의미는 다음과 같습니다.

> --cached: "네 로컬 작업 디렉토리의 실제 파일은 삭제하지 말고 그대로 둬라. 대신, Git이 관리하는 추적 대상 목록(인덱스 또는 캐시)에서만 제거해라."
>
> - **`r`:** 디렉토리일 경우, 그 안의 모든 파일과 하위 디렉토리에 대해 재귀적으로 적용해라.

**터미널에서 아래 명령어를 실행하여, 무시하고 싶은 파일들을 추적 해제합니다.**

```bash
# .idea 디렉토리 전체를 추적에서 제거
git rm -r --cached .idea

# 루트 디렉토리의 모든 .iml 파일을 추적에서 제거
git rm --cached *.iml

# build 디렉토리 전체를 추적에서 제거
git rm -r --cached build
```

**주의:** `git rm -r --cached .` 처럼 현재 디렉토리 전체(`.`)를 대상으로 실행하면, 의도치 않게 모든 파일의 추적이 해제될 수 있으므로, 제거하고 싶은 대상을 명확히 지정하는 것이 안전합니다.

### 3단계: "추적 해제" 상태를 커밋하기

2단계 명령을 실행한 후 `git status`를 확인하면, 다음과 같이 파일들이 "deleted" 상태로 스테이징된 것을 볼 수 있습니다.

```
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        deleted:    .idea/workspace.xml
        deleted:    my-project.iml
```

여기서 "deleted"는 "파일 시스템에서 파일이 삭제되었다"는 뜻이 아니라, **"Git의 버전 관리 대상에서 이 파일이 삭제(제외)되었다"**는 의미입니다. 이 '추적 해제'라는 변경 사항을 Git의 역사에 기록해야 모든 과정이 마무리됩니다.

**변경 사항을 커밋합니다.**

```bash
git commit -m "Chore: Stop tracking IDE and build files"
```

## 4. 결과 확인

이제 모든 과정이 끝났습니다. 다시 `git status`를 실행하면 작업 트리가 깨끗하다는 메시지가 나타납니다. IntelliJ와 같은 IDE의 Changes 탭에서도 더 이상 `.idea`나 `build` 관련 파일들이 나타나지 않을 것입니다.

앞으로 Gradle이 `build` 디렉토리를 새로 만들거나, IntelliJ가 `.idea` 디렉토리의 내용을 변경하더라도, `.gitignore` 파일이 올바르게 동작하여 이 파일들은 모두 자동으로 무시됩니다.

## 5. 결론: `.gitignore` Best Practice 요약

`.gitignore`와 관련된 문제를 미래에 방지하기 위해 다음 원칙들을 기억합시다.

1. **가장 중요한 규칙:** 프로젝트를 시작할 때, **첫 커밋 이전에** `.gitignore` 파일을 가장 먼저 생성하고 설정하라.
2. **이미 커밋했다면:** `git rm -r --cached <제거할_대상>` 명령을 사용하여 정확한 대상만 추적 해제하고, 그 상태를 **반드시 커밋**하여 Git 역사에 기록하라.
3. **팀 협업:** `.gitignore` 파일 자체는 버전 관리에 포함하여(`git add .gitignore` & `commit`), 모든 팀원이 동일한 파일 무시 규칙을 공유하도록 하라.
