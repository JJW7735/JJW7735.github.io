---
title: "커스텀 스탯 시스템을 GAS로 옮기기 (1) - 블루프린트를 안 깨는 법"
date: 2026-11-07 00:00:00 +0900
categories: [Unreal, GAS]
tags: [ue5, gas, gameplayabilitysystem, refactoring, facade]
---

> 3인 팀으로 만들고 있는 탑뷰 로그라이크(UE 5.8)의 체력·스탯 시스템을 Gameplay Ability System(GAS)으로 옮긴 과정을 정리한 글이다. 이번 편은 0~1단계(ASC 붙이기, 체력/스탯 Facade 전환)를 다룬다.

## 왜 GAS인가

처음엔 GAS 없이 시작했다. 체력은 `URLHealthComponent`, 공격력·치명타 같은 전투 스탯은 `URLStatComponent`에 `float` 변수로 들고 있는 단순한 구조였다. 게임이 작을 때는 이걸로 충분했다.

문제는 스탯을 바꾸는 곳이 늘어나면서 생겼다. 레벨업 보상, 상점, 패시브 아이템, 영구 강화(메타), 세이브 복원... 스탯을 바꾸는 곳이 다섯 군데가 됐고, 각자 이런 코드를 들고 있었다.

```cpp
/* RLInventoryComponent.cpp - 전환 전 */
switch (Bonus.TargetStat)
{
case ERLTargetStat::AttackPower:
    if (StatComp) StatComp->AttackPower += Delta;
    break;
case ERLTargetStat::CritChance:
    if (StatComp) StatComp->CritChance = FMath::Clamp(StatComp->CritChance + Delta, 0.0f, 100.0f);
    break;
case ERLTargetStat::MoveSpeed:
    if (StatComp)
    {
        StatComp->MoveSpeed += Delta;
        if (UCharacterMovementComponent* MoveComp = OwnerChar->GetCharacterMovement())
        {
            MoveComp->MaxWalkSpeed = StatComp->MoveSpeed;
        }
    }
    break;
// ... 8개 스탯 전부
}
```

같은 8-case switch가 인벤토리, 보상, 상점에 거의 똑같이 복사돼 있었다. 클램프 범위(치명타 0~100, 뎀감 0~80)도 각자 들고 있어서, 한 곳만 바꾸면 나머지와 어긋나는 구조였다. 꽤나 위험한 코드다.

그리고 이런 요구사항이 계속 들어왔다.

- 패시브 아이템은 **빼면 스탯도 돌아와야** 한다
- 버프 스킬은 **N초 뒤에 풀려야** 한다
- 보스 기믹은 **일정 시간 방어력을 무시**해야 한다

"영구 / 해제 가능 / 시간제 / 조건부" 같은 효과의 수명을 직접 관리하려면 결국 GAS의 GameplayEffect를 다시 만드는 꼴이 된다. 그래서 옮기기로 했다. 공모전·포트폴리오 목적상 언리얼 표준 구조를 써보는 것 자체에도 의미가 있었다.

## 전환 원칙 - 한 번에 다 바꾸지 않는다

가장 걱정한 건 **블루프린트**였다. UI(`WBP_HUD`, 스탯 창), 몬스터 BP(`BP_EnemyBase`), 보스 BT 태스크가 전부 `HealthComp->GetCurrentHealth()`, `OnHealthChanged`, `StatComp->AttackPower` 같은 기존 API에 물려 있었다. 컴포넌트를 지우고 ASC를 직접 쓰게 바꾸면 팀원들이 만든 BP를 전부 다시 배선해야 한다.

그래서 원칙을 하나 정했다.

> **컴포넌트는 지우지 않는다. 겉(API·델리게이트)은 그대로 두고, 속(실제 값)만 GAS로 옮긴다.**

Facade 패턴이다. 그리고 단계를 나눠서 단계마다 PIE로 확인하고 커밋했다.

| 단계 | 내용 |
|---|---|
| 0 | GAS 모듈, ASC/AttributeSet, 네이티브 태그 |
| 1 | HealthComponent, StatComponent를 Facade로 |
| 2 | 스탯 변경을 지속시간별 GE로 |
| 3 | 데미지 파이프라인을 ExecutionCalculation으로 |
| 4 | 스킬을 GameplayAbility로 |
| 5 | 스킬 VFX를 GameplayCue로 |

## 0단계 - ASC 붙이기

### 고민1 - ASC를 어디에 둘까

GAS 예제들을 보면 ASC를 `PlayerState`에 두는 경우가 많다. 멀티플레이에서 캐릭터가 죽고 다시 생성돼도 어빌리티·스탯을 유지하려는 이유다.

이 게임은 싱글플레이고, 맵 이동은 `OpenLevel` + 런 세이브로 상태를 넘기는 구조다. PlayerState에 둘 이유가 없다. 오히려 몬스터도 같은 `ARLCharacter`를 상속하기 때문에, 캐릭터에 두면 **플레이어와 몬스터가 같은 코드 경로**를 탄다. 그래서 `ARLCharacter`에 ASC를 붙이고 Owner = Avatar = 캐릭터 자신으로 했다.

### 고민2 - 태그는 어디서 정의할까

`DefaultGameplayTags.ini`에 문자열로 두면 C++에서 `FGameplayTag::RequestGameplayTag("State.Dead")`처럼 문자열로 꺼내야 하고, 오타가 나도 런타임까지 모른다. 그래서 `RLGameplayTags`에 네이티브 태그로 선언했다.

```cpp
/* RLGameplayTags.h */
namespace RLGameplayTags
{
    // ── 상태 (기존 bool 플래그 대체) ──
    ROGUELIKETOPVIEW_API UE_DECLARE_GAMEPLAY_TAG_EXTERN(State_Dead);
    // ...

    // ── SetByCaller 데이터 키 (GE에 런타임 수치를 넘길 때) ──
    ROGUELIKETOPVIEW_API UE_DECLARE_GAMEPLAY_TAG_EXTERN(Data_Damage);
    // ...
}
```

이후 규칙으로 "C++ 태그는 `RLGameplayTags`에 네이티브로 추가"를 프로젝트 가이드(CLAUDE.md)에 적어뒀다.

## 1단계 - 체력을 Facade로

### 기존 구조

기존 `URLHealthComponent`는 `CurrentHealth`를 직접 깎고, 0이 되면 `OnDeath`를 방송했다. 방어력 계산은 호출하는 쪽마다 제각각이었다.

### 구현

이제 피해는 전부 `ApplyIncomingDamage` 하나로 들어오고, 안에서는 **공격자의 ASC가 대상에게 Damage GE를 적용**한다.

```cpp
/* RLHealthComponent.cpp */
float URLHealthComponent::ApplyIncomingDamage(float RawDamage, AActor* DamageCauser, bool bTrueDamage)
{
    if (RawDamage <= 0.0f || bIsDead || !AbilitySystemComp)
    {
        return 0.0f;
    }

    // 공격자의 ASC가 GE를 "보내는" 쪽(Source). 환경 피해처럼 공격자 ASC가 없으면 자기 자신이 Source.
    UAbilitySystemComponent* SourceASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(DamageCauser);
    if (!SourceASC)
    {
        SourceASC = AbilitySystemComp;
    }

    FGameplayEffectContextHandle Context = SourceASC->MakeEffectContext();
    Context.AddInstigator(DamageCauser, DamageCauser);

    const FGameplayEffectSpecHandle Spec = SourceASC->MakeOutgoingSpec(URLGE_Damage::StaticClass(), 1.0f, Context);
    Spec.Data->SetSetByCallerMagnitude(RLGameplayTags::Data_Damage, RawDamage);
    Spec.Data->SetSetByCallerMagnitude(RLGameplayTags::Data_TrueDamage, bTrueDamage ? 1.0f : 0.0f);

    SourceASC->ApplyGameplayEffectSpecToTarget(*Spec.Data.Get(), AbilitySystemComp);
    // ...
}
```

피해량은 GE 에셋에 박지 않고 **SetByCaller**(`Data.Damage`)로 넘긴다. 스킬마다 데미지가 다르니 GE를 스킬 수만큼 만들 수는 없다.

중요한 건 함수 이름과 시그니처가 그대로라는 점이다. `OnHealthChanged`, `OnDeath` 델리게이트도 Health 어트리뷰트가 바뀔 때 Facade가 다시 방송해준다. 그래서 체력바 UI와 `BP_EnemyBase`의 OnDeath 그래프는 **하나도 안 건드렸다.** 사망 판정은 `State.Dead` 태그로 바꿨다.

## 1단계 - 스탯을 Facade로

### 고민3 - 미러 변수를 남길까, 지울까

`StatComp->AttackPower`를 직접 읽는 BP가 많았다. 변수를 지우면 그 BP가 전부 깨진다. 그래서 변수는 남기되 **`BlueprintReadOnly` 미러**로 바꿨다. 실제 값은 AttributeSet에 있고, 어트리뷰트가 바뀌면 미러가 갱신된다. BP는 읽기만 하던 대로 읽으면 된다.

반대로 "쓰기"는 막았다. 쓰는 길은 하나만 열어뒀다.

```cpp
/* RLStatComponent.cpp */
bool URLStatComponent::AddStatBonus(ERLTargetStat Stat, float Delta)
{
    const FGameplayAttribute Attribute = GetAttributeForStat(Stat);
    if (!Attribute.IsValid() || !AbilitySystemComp)
    {
        return false;
    }

    if (!FMath::IsNearlyZero(Delta))
    {
        TGuardValue<bool> SuppressGuard(bSuppressBroadcast, true);
        RLGameplayEffectUtils::ApplyStatBonusEffect(AbilitySystemComp, URLGE_StatBonus_Instant::StaticClass(), { { Attribute, Delta } });
    }
    return true;
}
```

그리고 아까 그 switch는 이렇게 됐다.

```cpp
/* RLInventoryComponent.cpp - 전환 후 */
// 전투 스탯 8종은 StatComp가 처리 (클램프/이동속도 반영 포함)
if (StatComp && StatComp->AddStatBonus(Bonus.TargetStat, Delta))
{
    continue;
}

switch (Bonus.TargetStat)
{
case ERLTargetStat::MaxHealth:
    if (HealthComp) HealthComp->IncreaseMaxHealth(Delta, bApply);
    break;
// 체력 관련만 남음
}
```

클램프와 `MaxWalkSpeed` 반영은 AttributeSet/StatComponent 한 곳으로 모였다. 인벤토리·보상·상점에서 같은 8-case switch를 걷어냈고, `RLRewardSubsystem.cpp`는 이 커밋에서 69줄이 바뀌었는데 거의 다 삭제였다. (커밋 전체 9 files, +275 / -167)

### 고민4 - BP 디폴트 값은 어떻게 들어가나

캐릭터마다 시작 스탯이 다르고, 그 값은 기획 쪽에서 BP 클래스 디폴트로 튜닝한다. 이걸 유지하고 싶었다.

그래서 컴포넌트 BeginPlay에서 **BP 디폴트 값을 AttributeSet의 시작값으로 밀어넣고**, 세이브 복원과 메타 강화는 그 뒤 `ARLCharacter::BeginPlay` 후반에 적용되도록 순서를 고정했다. 튜닝하는 입장에서는 예전처럼 디폴트 값만 바꾸면 된다.

## 결과

- 체력바, 스탯 창, 몬스터 BP, 보스 BT 태스크 — **BP 수정 0건**
- 스탯을 바꾸는 길이 `AddStatBonus` 하나로 통일
- PIE에서 3캐릭터 피격/회복/사망, 아이템 획득·해제 확인 후 단계별 커밋

<!-- 이미지: 전환 전후 구조 다이어그램 (컴포넌트 → ASC) -->

## 마치며

0~1단계는 사실 "GAS를 쓴다"기보다 "GAS로 옮길 수 있는 모양을 만든다"에 가까웠다. 아직 이 시점에서는 모든 스탯 변경이 Instant GE라서, 패시브 아이템을 빼면 스탯이 돌아오지 않는다. 그리고 데미지 경감 계산도 아직 호출하는 쪽에 흩어져 있다.

다음 편에서는 GE를 지속시간별(Instant/Infinite/Duration)로 나누고, 데미지를 ExecutionCalculation으로 옮긴 과정을 다룬다. 그 과정에서 **몬스터 공격이 지금까지 플레이어 방어력을 무시하고 있었다**는 걸 발견했는데, 그 얘기도 같이 정리할 예정이다.
