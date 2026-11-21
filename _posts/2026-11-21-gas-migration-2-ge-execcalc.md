---
title: "커스텀 스탯 시스템을 GAS로 옮기기 (2) - 효과의 수명과 데미지 공식"
date: 2026-11-21 00:00:00 +0900
categories: [Unreal, GAS]
tags: [ue5, gas, gameplayeffect, executioncalculation, refactoring]
---

> 3인 팀 탑뷰 로그라이크(UE 5.8)의 체력·스탯 시스템을 GAS로 옮긴 과정의 두 번째 글이다. 이번 편은 2~3단계(스탯 변경을 지속시간별 GameplayEffect로 나누기, 데미지 계산을 ExecutionCalculation으로 옮기기)를 다룬다.
> 선행 글: [커스텀 스탯 시스템을 GAS로 옮기기 (1) - 블루프린트를 안 깨는 법](/posts/gas-migration-1-facade/)

## 지난 편에서 남은 문제

1편이 끝난 시점에는 모든 스탯 변경이 **Instant GE** 하나로 들어갔다. "GAS를 쓰는 모양"만 만든 상태라서 두 가지 문제가 그대로 남아 있었다.

- 패시브 아이템을 빼도 스탯이 돌아오지 않는다. Instant GE는 기본값(Base) 자체를 바꾸고 끝나서, 되돌릴 손잡이가 없다.
- 방어력 계산이 여전히 호출하는 쪽에 흩어져 있다.

2단계는 첫 번째, 3단계는 두 번째 문제를 다룬다.

## 2단계 - 효과를 "얼마나 오래 가는가"로 나누기

### 기존 코드

GAS 전에는 패시브 아이템을 얻으면 스탯을 `+=`, 버리면 같은 값을 `-=` 했다.

```cpp
/* RLInventoryComponent.cpp - 전환 전 */
void URLInventoryComponent::ApplyPassiveItemBonuses(URLItemDataAsset* ItemData, int32 Multiplier, bool bApply)
{
    const float Sign = bApply ? 1.0f : -1.0f;

    for (const FRLStatBonus& Bonus : ItemData->PassiveStatBonuses)
    {
        const float Delta = Bonus.Value * Multiplier * Sign;
        // ... 스탯마다 += Delta
    }
}
```

`+=`와 `-=`의 짝이 정확히 맞아야만 원래 값으로 돌아오는 구조다. 클램프(치명타 0~100 등)에 한 번 걸리거나, 중간에 세이브/로드가 끼면 짝이 어긋나고 스탯이 조금씩 틀어진다.

### 고민1 - 스탯 변경을 몇 종류로 나눌까

스탯을 바꾸는 곳을 전부 모아서 "이 효과는 언제 사라지는가"로 분류해 봤다.

| 출처 | 언제 사라지나 | GE 종류 |
|---|---|---|
| 레벨업 보상, 상점 구매, 영구 메타 강화 | 안 사라짐 (런 동안 영구) | `URLGE_StatBonus_Instant` |
| 패시브 아이템 | 아이템을 뺄 때 | `URLGE_StatBonus_Infinite` |
| 버프 스킬 | N초 뒤 | `URLGE_StatBonus_Duration` |

GAS의 GameplayEffect가 원래 이 세 가지 수명(Instant / Infinite / HasDuration)을 지원하기 때문에, 분류만 잘 하면 수명 관리를 엔진에 맡길 수 있다. 수치는 GE 에셋에 박지 않고 전부 SetByCaller(`Data.Stat.*` 태그)로 넘겨서, 스탯 9종을 GE 클래스 하나로 처리한다.

```cpp
/* RLGameplayEffects.h */
// 런 동안 영구 보너스 (보상 / 상점 / 메타 강화)
UCLASS()
class ROGUELIKETOPVIEW_API URLGE_StatBonus_Instant : public URLGE_StatBonusBase { /* ... */ };

// 해제 가능한 보너스 (패시브 아이템 / 버프)
UCLASS()
class ROGUELIKETOPVIEW_API URLGE_StatBonus_Infinite : public URLGE_StatBonusBase { /* ... */ };

// 시간이 지나면 자동으로 사라지는 보너스 (버프 스킬). 지속 시간은 Data.Duration SetByCaller
UCLASS()
class ROGUELIKETOPVIEW_API URLGE_StatBonus_Duration : public URLGE_StatBonusBase { /* ... */ };
```

> Duration GE는 2단계 시점에는 없었고, 스킬을 GameplayAbility로 옮긴 4단계에서 추가됐다. 2단계에서는 버프도 Infinite GE + 타이머로 처리했다.
{: .prompt-info }

### 고민2 - 패시브 아이템을 뺄 때 무엇을 지울까

Infinite GE는 적용할 때 받은 `FActiveGameplayEffectHandle`로 정확히 그 효과만 지울 수 있다. 다만 같은 아이템을 여러 개 들 수도 있고 수량이 바뀌기도 해서, "어떤 변화가 있었는지"를 따라가며 핸들을 하나씩 붙이고 떼는 방식은 `+=`/`-=` 짝 맞추기와 같은 종류의 위험이 남는다.

그래서 **인벤토리가 바뀔 때마다 현재 슬롯 기준으로 아이템 효과 전체를 다시 걸도록** 했다.

```cpp
/* RLInventoryComponent.cpp - 전환 후 */
void URLInventoryComponent::RefreshPassiveItemEffects()
{
    // ...
    // 새 효과를 먼저 걸고 옛 효과를 나중에 지웁니다.
    // 반대로 하면 최대 체력이 잠깐 기본값까지 내려가면서 현재 체력이 clamp되어 깎여버립니다.
    TArray<FActiveGameplayEffectHandle> NewHandles;
    for (const FRLInventorySlot& Slot : Slots)
    {
        if (Slot.IsValid() && Slot.ItemData->ItemType == ERLItemType::Passive && Slot.ItemData->PassiveStatBonuses.Num() > 0)
        {
            const FActiveGameplayEffectHandle Handle = StatComp->ApplyRemovableStatBonuses(Slot.ItemData->PassiveStatBonuses, Slot.Quantity);
            if (Handle.IsValid())
            {
                NewHandles.Add(Handle);
            }
        }
    }

    for (FActiveGameplayEffectHandle& OldHandle : PassiveItemEffectHandles)
    {
        StatComp->RemoveStatBonusEffect(OldHandle);
    }
    PassiveItemEffectHandles = MoveTemp(NewHandles);
    // ...
}
```

"무엇이 바뀌었나"를 추적하지 않고 "지금 상태가 무엇인가"로 매번 다시 계산하니, 짝이 어긋날 여지 자체가 없어졌다. 인벤토리는 아이템을 얻거나 버릴 때만 바뀌므로 매번 다시 거는 비용도 크지 않다.

여기서 하나 걸린 게 순서다. 옛 효과를 먼저 지우면 최대 체력 아이템을 든 상태에서 최대 체력이 잠깐 기본값으로 내려가고, 그 순간 현재 체력이 최대치에 맞춰 잘려 버린다. 그래서 **새 효과를 먼저 걸고 옛 효과를 나중에 지운다.**

### 고민3 - 세이브에는 어떤 값을 저장할까

이전 세이브는 "아이템 효과까지 포함된 최종 스탯"을 저장했다. GE 구조에서는 이게 문제가 된다. 로드할 때 인벤토리가 복원되면서 아이템 GE가 다시 걸리는데, 저장된 값에 이미 아이템 보너스가 들어 있으니 **보너스가 두 번 적용된다.**

그래서 저장 방식을 바꿨다.

- 저장: 아이템 효과를 뺀 **기본값(Base)** 만 저장한다. 보상·상점·메타 강화는 Instant GE라서 기본값에 이미 들어 있다.
- 로드: **스탯 → 인벤토리 → 체력** 순서로 복원한다.

```cpp
/* RLSaveSubsystem.cpp */
// ★ 복원 순서: 스탯 기본값 → 인벤토리(아이템 GE 재적용) → 체력
//   체력을 마지막에 하는 이유: 저장된 현재 체력이 아이템 최대 체력 보너스를 포함할 수 있어서,
//   아이템 GE가 걸리기 전에 복원하면 기본 최대 체력에 clamp되어 깎입니다.
```

이미 만들어 둔 예전 세이브 파일도 있었다. 그래서 세이브 데이터에 `StatSaveVersion`을 추가했다. 필드가 없는 예전 파일은 기본값 0으로 읽히니, 0이면 "최종값이 저장된 파일"로 보고 방금 다시 걸린 아이템 보너스만큼 기본값에서 빼서 맞춘다.

```cpp
/* RLSaveGame.h */
// ★ 체력/스탯 저장 방식 버전 (GAS 전환으로 의미가 바뀜)
//   0 = 패시브 아이템 보너스가 포함된 최종값 (GAS 전환 전 저장 파일. 필드가 없으면 기본값 0)
//   1 = 패시브 아이템을 뺀 기본값. 아이템 보너스는 로드 시 인벤토리가 GE로 다시 적용
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Save|Run")
int32 StatSaveVersion = 0;
```

저장 데이터의 의미가 바뀌는 변경은 코드만 보면 티가 안 난다. 버전 필드를 두면 "이 파일은 어느 규칙으로 저장됐나"를 로드 쪽에서 구분할 수 있다.

### 덤 - 이동속도가 600으로 고정되던 버그

이 단계에서 기존 버그가 하나 같이 고쳐졌다. 이동속도 버프 스킬은 `MaxWalkSpeed`를 직접 곱하고, 끝나면 600으로 되돌리고 있었다.

```cpp
/* RLSkillComponent.cpp - 전환 전 */
MoveComp->MaxWalkSpeed *= Skill->SpeedMultiplier;   // 시전
// ...
OwnerCharacter->GetCharacterMovement()->MaxWalkSpeed = 600.0f;   // 종료
```

버프를 다시 쓰면 배율이 겹쳐서 곱해지고, 끝나면 아이템이나 보상으로 올린 이동속도까지 무시하고 600으로 고정됐다. 이동속도도 스탯 GE로 바꾸면서 "버프 핸들을 지운다"로 끝나게 됐고, 재시전 시에는 이전 버프 핸들을 먼저 지워서 중첩을 막았다.

## 3단계 - 데미지 계산을 한 곳으로

### 기존 구조

방어력 계산은 `ARLCharacter::ReceiveCustomDamage` 안에서 `StatComp->CalculateDefendedDamage`를 불러서 하고 있었다. 1편에서 피해의 입구를 `ApplyIncomingDamage` 하나로 모았지만, "얼마나 깎을지"는 여전히 GE 밖에서 계산된 값을 받아오는 구조였다.

### 구현 - ExecutionCalculation

GAS에서 "여러 어트리뷰트를 읽어서 하나의 결과를 계산하는" 용도로 있는 게 `UGameplayEffectExecutionCalculation`이다. 데미지 GE(`URLGE_Damage`)에 `URLDamageExecution`을 붙여서, GE가 적용되는 순간 **대상의** Defense와 DamageReduction을 읽어 경감하도록 했다.

```cpp
/* RLDamageExecution.cpp */
FRLDamageStatics()
{
    // 대상(Target)의 값을, 적용 시점에 캡처 (bSnapshot = false)
    DEFINE_ATTRIBUTE_CAPTUREDEF(URLAttributeSet, Defense, Target, false);
    DEFINE_ATTRIBUTE_CAPTUREDEF(URLAttributeSet, DamageReduction, Target, false);
}
```

마지막 인자 `false`가 스냅샷 여부다. 데미지는 맞는 순간의 방어력으로 계산해야 하니 스냅샷을 쓰지 않는다.

경감 공식 자체는 기존 것을 그대로 옮겼다.

```cpp
/* RLDamageExecution.cpp */
// 1. 방어력 배율 (점근적 공식). 음수 방어력(방어력 감소 디버프)은 추가 피해
const float DefenseMultiplier = Defense >= 0.0f
    ? 100.0f / (100.0f + Defense)
    : 2.0f - (100.0f / (100.0f - Defense));

// 2. 퍼센트 피해 감소율 (최대 80%)
const float ReductionMultiplier = 1.0f - (FMath::Clamp(DamageReduction, 0.0f, 80.0f) / 100.0f);

// 3. 최종 피해 (최소 1 보장)
const float FinalDamage = FMath::Max(1.0f, RawDamage * DefenseMultiplier * ReductionMultiplier);
```

`100 / (100 + 방어력)`은 방어력이 아무리 높아도 피해가 0이 되지 않는 점근적 공식이다. 방어력 100이면 피해 절반, 200이면 1/3이 된다.

계산 결과는 Health에 바로 빼지 않고, `IncomingDamage`라는 **메타 어트리뷰트**에 넣는다. 실제 Health 감소는 AttributeSet의 `PostGameplayEffectExecute`에서 한다. 이렇게 한 단계를 거치면 "피해가 들어왔다"는 시점에 사망 판정, 피격 알림 같은 후처리를 한 곳에서 할 수 있다.

```cpp
/* RLDamageExecution.cpp */
OutExecutionOutput.AddOutputModifier(FGameplayModifierEvaluatedData(
    URLAttributeSet::GetIncomingDamageAttribute(), EGameplayModOp::Additive, FinalDamage));
```

### 고민4 - 방어 무시는 어떻게 표현할까

트루 데미지와 보스 기믹의 "일정 시간 방어 무시"도 여기서 처리했다. 공격 쪽이 정하는 건 SetByCaller(`Data.TrueDamage`), 맞는 쪽의 상태로 정해지는 건 대상 태그(`State.IgnoreDefense`)로 나눴다.

```cpp
/* RLDamageExecution.cpp */
const bool bIgnoreDefense = Spec.GetSetByCallerMagnitude(RLGameplayTags::Data_TrueDamage, false, 0.0f) > 0.0f
    || (TargetASC && TargetASC->HasMatchingGameplayTag(RLGameplayTags::State_IgnoreDefense));
```

이렇게 해두니 나중에 보스 기믹을 만들 때 "대경직 동안 트루 데미지"가 태그 하나를 붙였다 떼는 것으로 끝났다.

### 발견한 버그 - 몬스터 공격이 방어력을 무시하고 있었다

데미지 계산을 한 곳으로 모으고 나니, 그 한 곳을 **안 거치는 경로**가 눈에 띄었다. 일반 몬스터(`BP_EnemyBase`)의 공격은 블루프린트의 엔진 기본 `ApplyDamage` 노드를 쓰고 있었고, 이 경로는 `ARLCharacter::TakeDamage`로 들어온다. GAS 전환 전 코드를 확인해 보니 이 함수는 방어력 계산 없이 엔진 기본 처리로 넘기기만 하고 있었다.

```cpp
/* RLCharacter.cpp - GAS 전환 전 */
float ARLCharacter::TakeDamage(float DamageAmount, struct FDamageEvent const& DamageEvent, class AController* EventInstigator, AActor* DamageCauser)
{
    if (bIsInvulnerable)
    {
        return 0.0f;
    }
    return Super::TakeDamage(DamageAmount, DamageEvent, EventInstigator, DamageCauser);
}
```

즉 처음부터 **플레이어 방어력은 일반 몬스터 공격에 아무 효과가 없었다.** 방어력 아이템을 사도 일반 몬스터한테는 의미가 없었던 셈이다. 계산이 흩어져 있을 때는 "어느 경로가 방어력을 적용하는지" 한눈에 안 보여서 몰랐다.

수정은 이 경로도 `ApplyIncomingDamage`(경감 적용)를 타게 하고, BP의 `Event AnyDamage`에는 경감 후 값을 알려주는 것이었다.

```cpp
/* RLCharacter.cpp - 수정 후 */
AActor* Causer = DamageCauser ? DamageCauser : (EventInstigator ? EventInstigator->GetPawn() : nullptr);
const float FinalDamage = HealthComp ? HealthComp->ApplyIncomingDamage(DamageAmount, Causer, false) : 0.0f;
if (FinalDamage <= 0.0f)
{
    return 0.0f;
}

// Super는 BP의 Event AnyDamage / OnTakeAnyDamage 알림만 보냅니다 (피해는 위에서 이미 적용). 경감 후 값으로 알림.
return Super::TakeDamage(FinalDamage, DamageEvent, EventInstigator, DamageCauser);
```

이렇게 바꾸자 이번엔 테스트용 즉사 명령이 방어력에 깎여서 안 죽는 일이 생겼다. 즉사 명령은 트루 데미지로 남은 체력만큼 넣도록 따로 바꿨다.

## 결과

| 단계 | 커밋 | 변경 |
|---|---|---|
| 2 GE 분리 | b9b3859 | 18 files, +429 / -94 |
| 3 ExecCalc | 7380ffc | 14 files, +246 / -90 |
| 3 방어력 버그 수정 | 38d8acc | 3 files, +14 / -11 |

- 패시브 아이템을 빼면 스탯이 정확히 돌아온다 (`+=`/`-=` 짝 맞추기 제거)
- 세이브/로드 시 아이템 보너스 이중 적용 없음, 예전 세이브 파일도 버전으로 보정
- 모든 피해가 `URLDamageExecution` 하나를 거친다
- 일반 몬스터 공격에도 방어력이 적용된다

## 마치며

이번 단계에서 가장 크게 남은 건 "단일 진입점"의 가치다. 피해 계산을 한 곳으로 모으는 작업 자체는 리팩터링이었는데, 그 과정에서 **한 곳을 안 거치는 경로**가 드러나면서 원래부터 있던 버그가 같이 잡혔다. 계산이 여러 군데 흩어져 있으면 "어느 경로가 무엇을 빠뜨렸는지"를 찾는 것부터가 어렵다.

아직 남은 것도 있다. 체력과 스탯은 GAS로 옮겼지만 스킬은 여전히 `RLSkillComponent`가 쿨타임과 진행 상태를 직접 들고 있다. 다음 편에서는 스킬 8종을 CastType별 GameplayAbility로, 스킬 VFX를 GameplayCue로 옮긴 4~5단계를 다룬다.
