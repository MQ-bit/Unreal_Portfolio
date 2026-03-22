# Warrior

포트폴리오 목적으로 제작한 Unreal Engine 5.4 기반 3D 액션 RPG / 웨이브 서바이벌 프로토타입입니다.

- 엔진: Unreal Engine 5.4
- 언어: C++
- 주요 기술: GAS, Gameplay Tags, Enhanced Input, UMG, AI, DataTable, SaveGame
- 테스트 맵: `FeatureDevMap`, `CombatTestMap`, `GameModeTestMap`, `SurvivalGameModeMap`, `MainMenuMap`

## 데모 영상

`TODO`

## 스크린샷

### 전투

`TODO`

### 서바이벌 웨이브

`TODO`

### Rage UI / Target Lock

`TODO`

## 기술 설명

### 전투 흐름

이 프로젝트는 전투 기능을 개별 블루프린트 이벤트로 연결하기보다, 상태가 어떤 경로로 바뀌는지 코드 단위로 구성했습니다.

`입력 -> Ability 활성화/취소 -> Gameplay Tag 상태 변경 -> Attribute 반영 -> UI 갱신`

### WarriorBaseCharacter

- 모든 전투 캐릭터의 공통 부모 클래스
- `AbilitySystemComponent`, `AttributeSet`, `MotionWarpingComponent`를 생성
- `PossessedBy`에서 ASC를 초기화하고 StartUpData를 사용할 준비를 함

> `WarriorBaseCharacter.cpp`

```cpp
AWarriorBaseCharacter::AWarriorBaseCharacter()
{
	PrimaryActorTick.bCanEverTick = false;
	PrimaryActorTick.bStartWithTickEnabled = false;

	GetMesh()->bReceivesDecals = false;
	WarriorAbilitySystemComponent = CreateDefaultSubobject<UWarriorAbilitySystemComponent>(TEXT("WarriorAbilitySystemComponent"));
	WarriorAttributeSet = CreateDefaultSubobject<UWarriorAttributeSet>(TEXT("WarriorAttributeSet"));
	MotionWarpingComponent = CreateDefaultSubobject<UMotionWarpingComponent>(TEXT("MotionWarpingComponent"));
}

void AWarriorBaseCharacter::PossessedBy(AController* NewController)
{
	Super::PossessedBy(NewController);

	if (WarriorAbilitySystemComponent)
	{
		WarriorAbilitySystemComponent->InitAbilityActorInfo(this, this);
		ensureMsgf(!CharacterStartUpData.IsNull(), TEXT("Forgot to assign start up data to %s"), *GetName());
	}
}
```

### WarriorHeroCharacter

- 플레이어 캐릭터 입력과 카메라 구성을 담당
- `Enhanced Input` 매핑을 등록하고, 입력 태그를 ASC에 전달
- 난이도에 따라 StartUpData 적용 레벨을 다르게 설정

> `WarriorHeroCharacter.cpp`

```cpp
void AWarriorHeroCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
	checkf(InputConfigDataAsset, TEXT("Forgot to assign a valid data asset as input config"));

	APlayerController* PlayerController = GetController<APlayerController>();
	check(PlayerController);

	ULocalPlayer* LocalPlayer = PlayerController->GetLocalPlayer();
	check(LocalPlayer);

	UEnhancedInputLocalPlayerSubsystem* Subsystem = ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(LocalPlayer);
	check(Subsystem);

	Subsystem->AddMappingContext(InputConfigDataAsset->DefaultMappingContext, 0);

	UWarriorInputComponent* WarriorInputComponent = CastChecked<UWarriorInputComponent>(PlayerInputComponent);

	WarriorInputComponent->BindNativeInputAction(InputConfigDataAsset, WarriorGameplayTags::InputTag_Move, ETriggerEvent::Triggered, this, &ThisClass::Input_Move);
	WarriorInputComponent->BindNativeInputAction(InputConfigDataAsset, WarriorGameplayTags::InputTag_Look, ETriggerEvent::Triggered, this, &ThisClass::Input_Look);
	WarriorInputComponent->BindAbilityInputAction(InputConfigDataAsset, this, &ThisClass::Input_AbilityInputPressed, &ThisClass::Input_AbilityInputReleased);
}
```

### WarriorAbilitySystemComponent

- 입력 태그와 Ability Spec의 `DynamicAbilityTags`를 연결
- 토글형 Ability와 홀드형 Ability를 같은 계층에서 처리
- 무기 장착 상태에 따라 Ability를 grant/remove

> `WarriorAbilitySystemComponent.cpp`

```cpp
void UWarriorAbilitySystemComponent::OnAbilityInputPressed(const FGameplayTag& InInputTag)
{
	if (!InInputTag.IsValid())
	{
		return;
	}

	for (const FGameplayAbilitySpec& AbilitySpec : GetActivatableAbilities())
	{
		if (!AbilitySpec.DynamicAbilityTags.HasTagExact(InInputTag)) continue;

		if (InInputTag.MatchesTag(WarriorGameplayTags::InputTag_Toggleable) && AbilitySpec.IsActive())
		{
			CancelAbilityHandle(AbilitySpec.Handle);
		}
		else
		{
			TryActivateAbility(AbilitySpec.Handle);
		}
	}
}
```

### WarriorAttributeSet

- 체력, Rage, 대미지 후처리를 담당
- `PostGameplayEffectExecute`에서 수치를 보정하고 상태 태그를 갱신
- UI 컴포넌트에 delegate를 보내 체력/Rage 게이지를 업데이트

> `WarriorAttributeSet.cpp`

```cpp
if (Data.EvaluatedData.Attribute == GetCurrentRageAttribute())
{
	const float NewCurrentRage = FMath::Clamp(GetCurrentRage(), 0.f, GetMaxRage());
	SetCurrentRage(NewCurrentRage);

	const float CurrentRageValue = GetCurrentRage();
	const float MaxRageValue = GetMaxRage();
	const bool bIsRageFull = FMath::IsNearlyEqual(CurrentRageValue, MaxRageValue, KINDA_SMALL_NUMBER);
	const bool bIsRageEmpty = FMath::IsNearlyZero(CurrentRageValue, KINDA_SMALL_NUMBER);

	if (bIsRageFull)
	{
		UWarriorFunctionLibrary::AddGameplayTagToActorIfNone(Data.Target.GetAvatarActor(), WarriorGameplayTags::Player_Status_Rage_Full);
		UWarriorFunctionLibrary::RemoveGameplayTagFromActorIfFound(Data.Target.GetAvatarActor(), WarriorGameplayTags::Player_Status_Rage_None);
	}
	else if (bIsRageEmpty)
	{
		UWarriorFunctionLibrary::AddGameplayTagToActorIfNone(Data.Target.GetAvatarActor(), WarriorGameplayTags::Player_Status_Rage_None);
		UWarriorFunctionLibrary::RemoveGameplayTagFromActorIfFound(Data.Target.GetAvatarActor(), WarriorGameplayTags::Player_Status_Rage_Full);
	}
}
```

### HeroGameplayAbility_TargetLock

- 타겟 탐색, 카메라 회전, 이동 속도 변경, 입력 컨텍스트 변경을 한 Ability에서 처리
- 시작 시 타겟을 잡고, 종료 시 위젯/이동 속도/입력 컨텍스트를 정리
- 전투 기능보다 상태 복구 흐름을 같이 보여주기 좋은 클래스

> `HeroGameplayAbility_TargetLock.cpp`

```cpp
void UHeroGameplayAbility_TargetLock::ActivateAbility(
	const FGameplayAbilitySpecHandle Handle,
	const FGameplayAbilityActorInfo* ActorInfo,
	const FGameplayAbilityActivationInfo ActivationInfo,
	const FGameplayEventData* TriggerEventData)
{
	TryLockOnTarget();
	InitTargetLockMovement();
	InitTargetLockMappingContext();

	Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);
}

void UHeroGameplayAbility_TargetLock::EndAbility(
	const FGameplayAbilitySpecHandle Handle,
	const FGameplayAbilityActorInfo* ActorInfo,
	const FGameplayAbilityActivationInfo ActivationInfo,
	bool bReplicateEndAbility,
	bool bWasCancelled)
{
	ResetTargetLockMovement();
	ResetTargetLockMappingContext();
	CleanUp();

	Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);
}
```

### WarriorSurvialGameMode

- 웨이브 상태 전이와 적 스폰을 담당
- `DataTable`에서 웨이브 정보를 읽고, 적 클래스는 비동기 로딩으로 미리 준비
- 적 처치 수에 따라 웨이브 완료 여부를 판단하고 다음 상태로 전환

> `WarriorSurvialGameMode.cpp`

```cpp
void AWarriorSurvialGameMode::BeginPlay()
{
	Super::BeginPlay();

	checkf(EnemyWaveSpawnerDataTable, TEXT("Forgot to assign a valid datat table in survial game mode blueprint"));
	TotalWavesToSpawn = EnemyWaveSpawnerDataTable->GetRowNames().Num();

	if (TotalWavesToSpawn <= 0)
	{
		SetCurrentSurvialGameModeState(EWarriorSurvialGameModeState::AllWavesDone);
		return;
	}

	SetCurrentSurvialGameModeState(EWarriorSurvialGameModeState::WaitSpawnNewWave);
	PreLoadNextWaveEnemies();
}

bool AWarriorSurvialGameMode::AreCurrentWaveEnemiesPreloaded() const
{
	for (const FWarriorEnemyWaveSpawnerInfo& SpawnerInfo : GetCurrentWaveSpawnerTableRow()->EnemyWaveSpawnerDefinitions)
	{
		if (SpawnerInfo.SoftEnemyClassToSpawn.IsNull()) continue;

		if (!PreLoadedEnemyClassMap.Contains(SpawnerInfo.SoftEnemyClassToSpawn))
		{
			return false;
		}
	}

	return true;
}

void AWarriorSurvialGameMode::PreLoadNextWaveEnemies()
{
	if (HasFinishedAllWaves())
	{
		return;
	}

	PreLoadedEnemyClassMap.Empty();

	for (const FWarriorEnemyWaveSpawnerInfo& SpawnerInfo : GetCurrentWaveSpawnerTableRow()->EnemyWaveSpawnerDefinitions)
	{
		if (SpawnerInfo.SoftEnemyClassToSpawn.IsNull()) continue;

		UAssetManager::GetStreamableManager().RequestAsyncLoad(
			SpawnerInfo.SoftEnemyClassToSpawn.ToSoftObjectPath(),
			FStreamableDelegate::CreateLambda([SpawnerInfo, this]()
			{
				if (UClass* LoadedEnemyClass = SpawnerInfo.SoftEnemyClassToSpawn.Get())
				{
					PreLoadedEnemyClassMap.Emplace(SpawnerInfo.SoftEnemyClassToSpawn, LoadedEnemyClass);
				}
			})
		);
	}
}
```

### DataAsset_StartUpDataBase / WeaponData

- 캐릭터 초기 Ability / Gameplay Effect를 데이터 기반으로 주입
- 무기별 입력, Anim 레이어, Ability 구성을 `FWarriorHeroWeaponData`에 정의
- 기능 변경 시 캐릭터 클래스보다 데이터 수정으로 대응할 수 있게 구성

> `DataAsset_StartUpDataBase.cpp`

```cpp
void UDataAsset_StartUpDataBase::GiveToAbilitySystemComponent(UWarriorAbilitySystemComponent* InASCToGive, int32 ApplyLevel)
{
	check(InASCToGive);

	GrantAbilities(ActivateOnGivenAbilities, InASCToGive, ApplyLevel);
	GrantAbilities(ReactiveAbilities, InASCToGive, ApplyLevel);

	if (!StartUpGameplayEffects.IsEmpty())
	{
		for (const TSubclassOf<UGameplayEffect>& EffectClass : StartUpGameplayEffects)
		{
			if (!EffectClass) continue;

			UGameplayEffect* EffectCDO = EffectClass->GetDefaultObject<UGameplayEffect>();
			InASCToGive->ApplyGameplayEffectToSelf(EffectCDO, ApplyLevel, InASCToGive->MakeEffectContext());
		}
	}
}
```

### WarriorGameInstance / 메뉴 흐름

- 맵 전환 전후 로딩 화면을 관리
- `Gameplay Tag -> Soft World` 형태로 레벨 데이터를 관리
- 메뉴에서 선택한 레벨이나 난이도를 실제 게임 진입 흐름과 연결할 수 있게 구성

> `WarriorGameInstance.cpp`

```cpp
void UWarriorGameInstance::Init()
{
	Super::Init();

	FCoreUObjectDelegates::PreLoadMap.AddUObject(this, &ThisClass::OnPreLoadMap);
	FCoreUObjectDelegates::PostLoadMapWithWorld.AddUObject(this, &ThisClass::OnDestinationWorldLoaded);
}

void UWarriorGameInstance::OnPreLoadMap(const FString& MapName)
{
	FLoadingScreenAttributes LoadingScreenAttributes;
	LoadingScreenAttributes.bAutoCompleteWhenLoadingCompletes = true;
	LoadingScreenAttributes.MinimumLoadingScreenDisplayTime = 2.f;
	LoadingScreenAttributes.WidgetLoadingScreen = FLoadingScreenAttributes::NewTestLoadingScreenWidget();

	GetMoviePlayer()->SetupLoadingScreen(LoadingScreenAttributes);
}
```

### HeroUIComponent

- 플레이어 UI에 필요한 이벤트를 모아두는 컴포넌트
- 체력/Rage 갱신뿐 아니라 무기 아이콘, Ability 아이콘, 쿨다운, 회복 아이템 상호작용 이벤트를 전달
- UI 위젯이 캐릭터 내부 상태를 직접 조회하기보다 delegate를 통해 반응하도록 구성

> `HeroUIComponent.h`

```cpp
UPROPERTY(BlueprintAssignable)
FOnPercentChangedDelegate OnCurrentRageChanged;

UPROPERTY(BlueprintCallable, BlueprintAssignable)
FOnEquippedWeaponChangedDelegate OnEquippedWeaponChanged;

UPROPERTY(BlueprintCallable, BlueprintAssignable)
FOnAbilityIconSlotUpdatedDelegate OnAbilityIconSlotUpdated;

UPROPERTY(BlueprintCallable, BlueprintAssignable)
FOnAbilityCooldownBeginDelegate OnAbilityCooldownBegin;
```

### Potion / Stone 소비 아이템 시스템

- 회복 아이템은 `Potion`, `Stone` 두 종류로 나누었고, 같은 상호작용 흐름을 사용하되 적용되는 회복량은 다르게 설정했습니다.
- 아이템 획득은 별도 Ability로 처리하고, 실제 아이템 오브젝트는 `PickUpBase` 계열 Actor로 분리했습니다.
- 플레이어가 아이템에 접근하면 `Player.Ability.PickUp.Potion`을 활성화하고, UI에는 `OnPotionInteracted` delegate로 상호작용 가능 상태를 전달합니다.
- 실제 소비 시에는 각 아이템 Actor가 서로 다른 `GameplayEffect`를 플레이어 ASC에 적용합니다.

> `HeroGameplayAbility_PickUpPotion.cpp`

```cpp
void UHeroGameplayAbility_PickUpPotion::ActivateAbility(
	const FGameplayAbilitySpecHandle Handle,
	const FGameplayAbilityActorInfo* ActorInfo,
	const FGameplayAbilityActivationInfo ActivationInfo,
	const FGameplayEventData* TriggerEventData)
{
	GetHeroUIComponentFromActorInfo()->OnPotionInteracted.Broadcast(true);

	Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);
}

void UHeroGameplayAbility_PickUpPotion::ConsumePotion()
{
	if (CollectedPotion.IsEmpty())
	{
		CancelAbility(GetCurrentAbilitySpecHandle(), GetCurrentActorInfo(), GetCurrentActivationInfo(), true);
		return;
	}

	for (AWarriorPotionBase* Potion : CollectedPotion)
	{
		if (Potion)
		{
			Potion->Consume(GetWarriorAbilitySystemComponentFromActorInfo(), GetAbilityLevel());
		}
	}
}
```

### WarriorFunctionLibrary

- 태그 추가/삭제, 피격 방향 계산, 블록 판정, 세이브/로드 같은 공용 로직을 담당
- GAS와 캐릭터, 메뉴 흐름 사이에서 반복되는 공통 처리를 묶는 역할

> `WarriorFunctionLibrary.cpp`

```cpp
bool UWarriorFunctionLibrary::ApplyGameplayEffectSpecHandleToTargetActor(
	AActor* InInstigator,
	AActor* InTargetActor,
	const FGameplayEffectSpecHandle& InSpecHandle)
{
	if (!InInstigator || !InTargetActor || !InSpecHandle.IsValid())
	{
		return false;
	}

	UWarriorAbilitySystemComponent* SourceASC = NativeGetWarriorASCFromActor(InInstigator);
	UWarriorAbilitySystemComponent* TargetASC = NativeGetWarriorASCFromActor(InTargetActor);
	FActiveGameplayEffectHandle ActiveGameplayEffectHandle = SourceASC->ApplyGameplayEffectSpecToTarget(*InSpecHandle.Data, TargetASC);

	return ActiveGameplayEffectHandle.WasSuccessfullyApplied();
}
```

## 시스템 메모

### 모듈 분리

- `AbilitySystemComponent`: 입력 처리, Ability 활성화, grant/remove
- `AttributeSet`: 체력 / Rage / 대미지 후처리
- `CombatComponent`: 무기 충돌과 적중 처리
- `UIComponent`: UI 이벤트 전달
- `GameMode`: 웨이브 진행, 스폰, 상태 전이
- `DataAsset`: 초기 Ability / Effect와 무기 데이터 정의

한 클래스에 기능을 몰아넣기보다, 전투 상태가 바뀌는 경로에 맞춰 모듈을 나눠 두는 쪽에 가깝습니다.

### 데이터 흐름

- 메뉴 난이도 선택
- `SaveCurrentGameDifficulty`
- `WarriorSurvialGameMode::InitGame`
- StartUpData 적용 레벨 반영
- 실제 전투 난이도 변경

이 흐름은 단순 메뉴 저장 기능보다, 메뉴 선택이 실제 전투 로직까지 어떻게 이어지는지 보여주기 좋은 부분입니다.

### 최근 안정화 작업

- `WarriorHeroCharacter`
  - 입력 바인딩 시 `PlayerController`, `LocalPlayer`를 명확히 확인하도록 수정
  - 이동 및 Ability 입력에서 null 접근 방어 코드 추가
- `WarriorAttributeSet`
  - 체력 / Rage UI 갱신 시 0으로 나누는 상황을 피하도록 보강
- `WarriorSurvialGameMode`
  - 비동기 적 클래스 로딩이 끝난 뒤에만 웨이브 스폰을 진행하도록 조정
  - preload map 조회 시 즉시 크래시하지 않도록 보수적으로 처리
  - 스폰 카운터가 음수로 내려가지 않도록 보정
- `HeroGameplayAbility_TargetLock`
  - 컨트롤러, 로컬 플레이어, 입력 매핑 컨텍스트, 위젯 생성 실패 시 방어적으로 반환
  - UI 위치 투영 실패와 이동 속도 복구 시 null 경로 보강
- `WarriorProjectileBase`, `WarriorWeaponBase`, `WarriorWidgetBase`
  - 발사자/자기 자신 충돌, invalid damage spec, UI 컴포넌트 누락 같은 예외 상황에서 크래시하지 않도록 정리

기능을 새로 추가한 작업은 아니고, 정상 동작 경로는 유지하면서 런타임 예외 상황에 더 보수적으로 대응하도록 다듬은 작업입니다.

## 트러블슈팅

### Rage UI는 100%인데 Rage Ability가 발동되지 않던 문제

- UI에서는 Rage가 가득 찬 것처럼 보였지만 Ability는 간헐적으로 발동하지 않았습니다.
- 입력 매핑, Ability grant, Activation Required Tags, AttributeSet 갱신 순서로 확인했습니다.
- 원인은 Rage 상태를 float `==`로 비교하던 로직이었습니다.
- 부동소수점 오차 때문에 `Player.Status.Rage.Full` 태그가 안정적으로 붙지 않았고, 이 때문에 Ability 조건이 막히고 있었습니다.
- 수정 후에는 `FMath::IsNearlyEqual`, `FMath::IsNearlyZero`를 사용해 판정하고, Full / Empty 태그가 배타적으로 유지되도록 정리했습니다.

이 이슈는 단순 UI 버그가 아니라 `입력 -> Ability -> Tag -> Attribute -> UI` 흐름 전체를 추적해야 해결할 수 있었던 문제였습니다.

### 웨이브 기반 적 스폰 시스템 안정화

- 웨이브 스폰은 단순히 적을 생성하는 기능이 아니라 `DataTable` 기반 적 정보 관리, `Soft Class` 참조, 비동기 로딩, Spawn Point 탐색, 웨이브 상태 전환이 함께 맞물려 동작해야 했습니다.
- 특히 로딩 시점과 스폰 타이밍이 어긋나면 적 클래스가 아직 준비되지 않은 상태에서 스폰이 시도되거나, 웨이브 흐름이 중간에 끊길 수 있어 구조적인 설계가 중요했습니다.
- 이를 해결하기 위해 웨이브 진행을 상태 단위로 분리하고, 적 데이터는 `DataTable`로 관리하되 실제 스폰 클래스는 `Soft Class + Async Load` 방식으로 처리해 참조 의존성과 초기 로딩 부담을 줄였습니다.
- 또한 Spawn Point와 `NavigationSystem`을 활용해 스폰 위치를 보정하고, 웨이브 완료 조건과 다음 웨이브 진입 시점을 `GameMode`에서 일관되게 제어하도록 구성했습니다.
- 최근에는 비동기 로딩 완료 전 스폰 진입 방지, preload map 조회 시 즉시 크래시하지 않도록 하는 처리, 스폰 카운터 음수 방지 같은 안정화 작업도 추가했습니다.

이 과정을 통해 언리얼 프로젝트에서는 기능 구현 자체보다 데이터 구조, 로딩 전략, 상태 전이까지 함께 설계해야 실제로 안정적인 시스템이 된다는 점을 체감했습니다.

## 실행 방법

1. Unreal Engine 5.4에서 `Warrior.uproject`를 엽니다.
2. 필요하면 Visual Studio 프로젝트 파일을 생성합니다.
3. 아래 맵 중 하나를 열어 기능을 확인합니다.

- `FeatureDevMap`: 기본 전투 확인
- `CombatTestMap`: 전투 기능 테스트
- `GameModeTestMap`: 게임 모드 테스트
- `SurvivalGameModeMap`: 웨이브 서바이벌 확인
- `MainMenuMap`: 메뉴 흐름 확인

## 사용 기술

- Unreal Engine 5.4
- C++
- Gameplay Ability System
- Gameplay Tags
- Enhanced Input
- UMG
- Behavior Tree / AI
- DataTable
- NavigationSystem
- SaveGame
