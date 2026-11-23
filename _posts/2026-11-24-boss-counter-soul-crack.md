---
title: "보스 카운터 기믹 '영혼의 균열' 만들기 - 새 시스템 대신 기존 구조 재사용"
date: 2026-11-24 00:00:00 +0900
categories: [Unreal, Combat]
tags: [ue5, gas, bossdesign, behaviortree, gameplaytag]
---

> 3인 팀 탑뷰 로그라이크(UE 5.8)에서 보스 카운터 기믹 "영혼의 균열(Soul Crack)"을 만든 과정과, 만들면서 겪은 문제를 정리한 글이다.
> 선행 글: [커스텀 스탯 시스템을 GAS로 옮기기 (2) - 효과의 수명과 데미지 공식](/posts/gas-migration-2-ge-execcalc/)

## 시작 동기

보스 패턴 5개(SwingCombo, BigSwing, SoulDrain, Subjugation, Dash)를 완성하고 보니, 보스전에서 플레이어가 하는 일이 **피하기뿐**이었다. 패턴을 읽고 피하는 재미는 있지만, 보스의 공격에 플레이어가 끼어들 수단이 없었다.

그래서 로스트아크의 카운터 방식을 가져왔다.

- 보스가 낮은 확률로 **강화판** 패턴을 쓴다
- 정해진 타이밍에, 정해진 위치에서, 전용 스킬을 맞히면 균열이 1스택 쌓인다
- 3스택이 되면 보스가 대경직에 빠지고 딜 타임이 열린다

이름은 "카운터" 대신 게임 테마에 맞춰 **영혼의 균열**로 지었다. 보스를 두른 영혼 보호막에 금이 가다가(스택) 깨진다(대균열)는 서사다.

## 설계 원칙 - 기존 구조를 최대한 재사용한다

새 기믹을 위해 새 시스템을 만들면 그만큼 새 버그가 생긴다. 그래서 이미 있는 구조로 어디까지 되는지부터 확인했다.

### 고민1 - 강화판 패턴을 어떻게 만들까

몬스터 패턴은 데이터 에셋(`DA_Enemy_Boss`)의 배열로 관리된다. 그리고 패턴 선택 로직은 쿨타임을 **PatternID 기준으로** 따로 돌린다.

그래서 강화판은 **새 PatternID + 낮은 SelectionWeight를 가진 데이터 행 하나**로 만들었다. 원본 패턴과 강화판의 쿨타임이 자동으로 따로 돌고, 낮은 가중치 덕분에 가끔만 나온다. C++ 수정이 필요 없었다.

첫 강화판은 가장 싸게 검증할 수 있는 BigSwing으로 했다. BigSwing은 보스 주변 부채꼴 판정인데, 원본 데이터 행에서 뒤쪽 안전지대 각도(`SectorSafeZoneAngle`)만 90에서 0으로 바꾸면 270° 판정이 360° 판정이 된다. 애니메이션도 코드도 건드리지 않고 "더 위험한 강화판"이 하나 생긴다.

### 고민2 - "성공"을 무엇으로 판정할까

균열 성공은 세 조건을 모두 만족해야 한다.

1. **균열을 낼 수 있는 스킬로 맞혔는가** - 전용 스킬 슬롯을 따로 만들지 않고, 스킬 데이터(`URLSkillData`)에 `bCanTriggerSoulCrack` bool 하나만 추가했다. 지금은 블레이드의 W 스킬에 켜져 있다.
2. **균열 타이밍 구간 안인가**
3. **올바른 각도 구역 안인가**

3번은 처음에 보스의 특정 부위(소켓)까지 거리로 판정하는 방법도 생각할 수 있었지만, 탑뷰에서 플레이어가 읽기 쉬운 건 "보스의 앞/뒤"다. 그래서 보스 정면 방향과 공격자 방향 사이의 각도로 판정했다. 수식은 이미 있던 부채꼴 히트박스와 같다.

```cpp
/* RLEnemyCharacter.cpp */
// ★ CreateSectorHitbox와 같은 각도 계산 (보스 forward vs 공격자 방향). 0도 = 정면, 180도 = 정후방
bool ARLEnemyCharacter::IsInSoulCrackZone(const AActor* Attacker) const
{
    // ...
    const FVector ToAttacker2D = (Attacker->GetActorLocation() - GetActorLocation()).GetSafeNormal2D();
    const FVector ForwardDir = GetActorForwardVector().GetSafeNormal2D();
    const float AngleFromForward = FMath::RadiansToDegrees(FMath::Acos(FMath::Clamp(FVector::DotProduct(ForwardDir, ToAttacker2D), -1.0f, 1.0f)));

    const bool bInFront = SoulCrackFrontAngle > 0.0f && AngleFromForward <= SoulCrackFrontAngle * 0.5f;
    const bool bInBack = SoulCrackBackAngle > 0.0f && (180.0f - AngleFromForward) <= SoulCrackBackAngle * 0.5f;
    return bInFront || bInBack;
}
```

두 벡터를 정규화해서 내적하면 사이 각의 코사인이 나오고, `Acos`로 각도로 바꾼다. 정면 구역은 "정면에서 몇 도 이내", 후방 구역은 "정후방(180°)에서 몇 도 이내"로 같은 값 하나로 둘 다 검사한다. 패턴마다 정면만, 후방만, 둘 다를 데이터로 고를 수 있다.

### 고민3 - 타이밍 구간은 어떻게 정할까

이것도 이미 있는 걸 썼다. 보스 공격 예고 데칼은 몽타주의 `Notify_Hit` 위치를 미리 훑어서, 각 타격의 `[히트 - LeadTime, 히트)` 구간 동안 장판을 띄운다(`ComputeTelegraphWindows`). 균열 구간은 그 구간 중 하나를 그대로 쓴다.

```cpp
/* RLEnemyCharacter.cpp */
// 텔레그래프와 같은 노티파이 스캔을 재사용해서 각 히트의 [히트 - LeadTime, 히트) 구간을 구하고, 그중 하나만 균열 구간으로 씀
const TArray<FVector2D> Windows = ComputeTelegraphWindows(Pattern.AttackMontage, HitNotifyName, Pattern.SoulCrackWindowLeadTime);
// ...
const int32 Index = Pattern.SoulCrackHitIndex < 0 ? Windows.Num() - 1 : FMath::Min(Pattern.SoulCrackHitIndex, Windows.Num() - 1);
SoulCrackWindow = Windows[Index];
```

판정할 때는 몽타주 재생 위치(`Montage_GetPosition`)가 이 구간 안에 있는지만 보면 된다. 재생 위치를 기준으로 하니 몽타주 재생 속도가 바뀌어도 구간이 같이 따라간다.

### 고민4 - 판정은 어디서 할까

피격 처리 함수(`ReceiveCustomDamage`)를 오버라이드해서, **부모의 데미지 처리를 먼저 끝낸 뒤** 균열 조건을 검사했다. 부모가 무시한 피격(같은 팀, 무적)은 균열도 무시해서 두 판정 결과가 어긋나지 않게 했다.

```cpp
/* RLEnemyCharacter.cpp */
void ARLEnemyCharacter::ReceiveCustomDamage(const FRLDamageData& DamageData, const FHitResult& HitInfo)
{
    const bool bWasDead = HealthComp && HealthComp->bIsDead;

    Super::ReceiveCustomDamage(DamageData, HitInfo);

    if (bWasDead || !DamageData.bCanTriggerSoulCrack || (HealthComp && HealthComp->bIsDead))
    {
        return;
    }

    // Super에서 무시된 피격(같은 팀 / 무적)은 균열도 무시
    // ...

    if (IsSoulCrackWindowOpen() && IsInSoulCrackZone(DamageData.DamageCauser))
    {
        TriggerSoulCrackHit();
    }
}
```

## 구현 흐름

```
BTT_Attack: PlayMontage → BeginSoulCrackPattern(패턴)
  └ 강화판이면: 균열 구간 계산 + 정면/후방 안내 데칼 + 녹색 아우라
Tick: 구간이 열려 있는 동안 보스 몸에 Overlay 머티리얼로 발광
피격: ReceiveCustomDamage → 3조건 충족 → TriggerSoulCrackHit
  ├ 스택 +1, 보호막 감소, 공격 몽타주 끊기, 소경직
  └ 3스택 → TriggerSoulCrackShatter: 대경직 + 보호막 소멸 + 방어 무시
```

보스는 원래 `CanBeStaggered()`가 false라서 일반 공격으로는 경직이 걸리지 않는다. 그래서 균열 성공 때만 이 검사를 거치지 않는 **별도 강제 경직 경로**를 만들었다. 그리고 대균열은 전투당 1회로 제한했다(`bHasShatteredThisFight`). 그렇지 않으면 대균열을 반복해서 보스를 계속 경직시키는 공략이 생긴다.

플레이어에게 보여주는 신호는 세 단계로 나눴다.

| 시점 | 신호 |
|---|---|
| 패턴 시작 전 | 녹색 아우라 + 녹색 예고 데칼 → "이번 공격은 강화판" |
| 패턴 진행 중 | 바닥에 카운터 방향 안내 데칼 → "어디서 쳐야 하나" |
| 균열 구간 | 보스 몸 발광 → "지금 쳐야 한다" |

녹색은 이 기믹 전용 색으로 정했다. 공격 예고는 빨강이라 두 신호가 섞이지 않는다.

## 겪은 문제들

### 문제1 - 디버프가 아무 효과가 없었다 (기획 문제)

처음 계획은 대균열 때 보스의 피해 감소(`DamageReduction`)를 0으로 눌렀다가 나중에 복구하는 것이었다. 구현하고 나서야 알았는데, **보스의 피해 감소 기본값이 원래 0이었다.** 0을 0으로 누르는 디버프라서, 대균열이 나도 데미지가 그대로였다. 코드는 정상이었고, 기획이 수치상 작동하지 않았던 것이다.

그래서 서사에 있던 "영혼 보호막"을 실제 수치로 만들었다.

- 보스는 전투 시작 때 **피해 감소 30% 보호막**을 가진다. GAS의 Infinite GE(`URLGE_SoulCrackShield`)로 걸어서 핸들로 언제든 바꿀 수 있게 했다
- 균열에 성공할 때마다 10%씩 깎인다: 30 → 20 → 10 → 대균열 시 0
- 대경직 동안에는 `State.IgnoreDefense` 태그를 붙여 방어력까지 무시하는 트루 데미지가 들어간다. 경직이 끝나면 태그만 뗀다

```cpp
/* RLEnemyCharacter.cpp - TriggerSoulCrackHit */
// 균열 성공마다 보호막이 깎임 (30 → 20 → 10 → 대균열 시 0)
if (CurrentSoulCrackShield > 0.0f)
{
    SetSoulCrackShield(SoulCrackShieldDamageReduction - SoulCrackShieldReductionPerHit * SoulCrackStackCount);
}
```

```cpp
/* RLEnemyCharacter.cpp - TriggerSoulCrackShatter */
// 보호막 완전 소멸 (이번 전투 동안 복구 안 됨)
SetSoulCrackShield(0.0f);

// 대경직 동안 방어 무시(트루뎀). 경직이 끝나면 태그만 떼고, 방어력은 원래 값 그대로 다시 적용됩니다.
AbilitySystemComp->AddLooseGameplayTag(RLGameplayTags::State_IgnoreDefense);
```

이게 가능했던 건 [GAS 2편](/posts/gas-migration-2-ge-execcalc/)에서 데미지 계산을 ExecutionCalculation 한 곳으로 모아 두었기 때문이다. 계산하는 곳에서 `State.IgnoreDefense` 태그를 이미 검사하고 있어서, 기믹 쪽에서는 **GE 하나와 태그 하나**만 붙였다 떼면 끝났다. 값을 저장해 뒀다가 복구하는 코드가 필요 없다.

### 문제2 - 대경직 중에 보스가 공격한다

처음엔 경직 동안 비헤이비어 트리를 `PauseLogic`으로 일시정지했다. 그런데 진행 중이던 `BTT_Attack`의 몽타주 대기 콜백과 꼬이면서, 대경직 중인데도 보스가 공격하는 경우가 생겼다. 그래서 사망 처리와 같은 방식으로 `StopLogic`으로 완전히 멈췄다가 `RestartLogic`으로 루트부터 다시 시작하게 바꿨다.

그래도 다른 경로에서 BT가 다시 켜지는 경우가 남아 있어서, Tick에서 감시하는 안전장치를 넣었다.

```cpp
/* RLEnemyCharacter.cpp - Tick */
UE_LOG(LogTemp, Warning, TEXT("[SoulCrack] %s: 경직 중(%.1f초 남음)인데 BT가 다시 실행 중 → 재정지"),
    *GetName(), SoulCrackAIStopEndTime - GetWorld()->GetTimeSeconds());
Brain->StopLogic(TEXT("SoulCrack"));
```

조용히 고치기만 하지 않고 로그를 남기게 한 이유는, 이 안전장치가 발동한다는 것 자체가 어딘가에 BT를 켜는 경로가 하나 더 있다는 뜻이기 때문이다.

### 문제3 - 균열에 성공했는데 보스 공격에 맞는다

경직으로 끊겼어야 할 공격의 `Notify_Hit`가 그대로 터졌다. 그래서 피격 모션을 재생하기 전에 `StopAllMontages`로 공격 몽타주를 먼저 확실히 끊도록 했다. 바닥에 남아 있는 예고 데칼도 같이 정리한다(`StopAttackTelegraph`).

### 문제4 - 새 판정 형태를 만들지 않고 설계를 바꿨다

Subjugation 강화판은 "보스 가까이가 안전하고 바깥이 위험한" 링 모양 공격으로 기획했다. 원래대로라면 판정 형태 enum에 `RadialBurst`를 새로 추가해야 했다.

그런데 enum에 값을 추가하면 BP의 `Switch on Enum` 노드에 연결 안 된 새 핀이 생기고, 실제로 BigSwing의 Sector 핀이 비어 있어서 판정이 안 나간 적이 있었다([조용히 실패하는 경우들](/posts/unreal-silent-failures/) 5번). 같은 실수를 반복할 위험을 만들기보다, 기존 `Sector`에 **안쪽 안전 반경(`SectorInnerSafeRadius`) 필드 하나만 추가**하는 방식으로 바꿨다.

SwingCombo 강화판도 원래 계획했던 차징 공격을 빼고, 전진하면서 타격마다 범위가 커지는 버전으로 바꿨다. 새 애니메이션을 만들 인력이 없는 팀이라, 기존 애니메이션으로 표현할 수 있는 방향으로 기획을 맞춘 것이다.

### 문제5 - 두 번째 보스의 잠행 강습과 충돌

두 번째 보스에 땅속에 숨었다가 튀어나오는 "잠행 강습" 패턴을 추가하면서 균열과 세 군데가 부딪혔다.

**몽타주를 일시정지하면 강화판이 끝나 버렸다.** 잠행 강습은 숨어 있는 동안 몽타주를 Pause한다. 그런데 강화판 정리 로직이 `Montage_IsPlaying`으로 "몽타주가 끝났나"를 검사하고 있었고, 이 함수는 **일시정지 중에는 false**를 돌려준다. 그래서 숨는 순간 강화판이 종료됐다. 일시정지 중에도 true인 `Montage_IsActive`로 바꿔서 해결했다.

```cpp
/* RLEnemyCharacter.cpp - Tick */
if (!AnimInstance || !AnimInstance->Montage_IsActive(SoulCrackMontage))
{
    EndSoulCrackPattern();
}
```

**숨어 있거나 무적인 동안에도 균열이 가능했다.** 균열 구간은 타격 직전 구간이라, 숨어 있는 시간과 겹쳤다. 구간 시작점을 "완전히 나타난 뒤 ~ 타격 전"으로 늦췄다.

```cpp
/* RLEnemyCharacter.cpp - BeginSoulCrackPattern */
// ★ 잠행 강습 강화판: 사라지기 전/숨어 있는 동안/솟아오르는(무적) 동안에는 균열 불가 → 다 나타난 뒤부터 타격 전까지만 열림
if (VanishStep != EVanishStep::None && VanishMontage == Pattern.AttackMontage && SoulCrackWindow.Y > VanishNotifyTime)
{
    SoulCrackWindow.X = FMath::Max(SoulCrackWindow.X, VanishNotifyTime + VanishPattern.VanishFadeInTime + 0.02f);
}
```

**숨은 위치가 드러났다.** 카운터 방향 안내 데칼이 보스를 따라다니니, 땅속에 숨어 있어도 데칼 위치로 보스가 어디 있는지 보였다. 숨어 있는 동안에는 안내 데칼도 숨기도록 했다.

### 그 밖의 문제

- 레벨에 이미 배치된 보스 인스턴스에 BP 기본값(대경직 몽타주)이 반영되지 않아서, BeginPlay에서 값이 비어 있으면 클래스 기본값으로 채우게 했다
- 대경직 몽타주가 `DefaultSlot`을 써서 재생되지 않았다. [보스 버그 글](/posts/boss-no-attack-5-causes/)의 슬롯 불일치가 재발한 것이라 `FullBody`로 고쳤다

## 마치며

돌아보면 이 기믹에서 새로 만든 것보다 **재사용한 것**이 더 많다.

| 필요한 것 | 재사용한 것 |
|---|---|
| 강화판 패턴 | 패턴 데이터 행 + PatternID별 쿨타임 |
| 타이밍 구간 | 예고 데칼의 `ComputeTelegraphWindows` |
| 위치 판정 | 부채꼴 히트박스의 각도 수식 |
| 보호막·트루 데미지 | GAS의 Infinite GE와 `State.IgnoreDefense` 태그 |

덕분에 새 코드는 "균열 상태 관리"와 "강제 경직 경로" 정도로 끝났다. 반대로 가장 크게 막혔던 건 코드가 아니라 기획이었다. 보스의 피해 감소가 원래 0이라 디버프가 작동하지 않았던 건, 구현하고 수치를 찍어 보기 전까지 아무도 몰랐다. 기획 단계에서 "이 효과가 실제로 몇 % 차이를 만드는가"를 먼저 계산해 봤어야 했다.

아직 남은 것도 있다. 실제 플레이에서 강화판이 나오는 빈도와 균열 구간 길이가 적당한지는 플레이테스트로 더 맞춰야 한다.
