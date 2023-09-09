# Sicarius_GoogleDrive

Github 용량부족으로 기존의 Sicarius Repository를 구글 드라이브 링크로 대체합니다.

구글 링크 : https://drive.google.com/file/d/194dARONwil3TcJasRPd3BkTnMLuuWZc9/view?usp=sharing

비디오 링크 : https://youtu.be/z4qN8cH2Qo4  
프로젝트 링크(유효하지 않음) : https://github.com/Naezan/Sicarius  
노션 링크 : https://rose-experience-136.notion.site/cf2a4bde570947cdbd5cb84890b5a02d?pvs=4  

- 프로젝트 정보
  - 
  ```shell
    플랫폼 : 윈도우
    엔진 : 언리얼 엔진5
    개발 기간 : 4개월
    개발 인원 : 1명
  ```

- 게임 설명
  - 
  ```shell
    Sicarius는 저의 세번째 프로젝트이자 언리얼 엔진을 사용한 두번째 프로젝트입니다.
    이 게임은 1~3인의 멀티 플레이 게임으로 다양한 암살 스킬을 통해 적들을 처치해야하는 암살자로의 사명을 가지고 있습니다.
    타겟이 있는 마을로 들어간 당신은 총 3명의 타겟을 처치하는 임무를 완수해야만합니다.
  ```

<a name="top"></a>
## 구현목록

> 1. [CharacterMovementComponent](#cmc)
> 2. [AbilitySystemComponent](#asc)  
>    2.1 [게임플레이 태그와 어빌리티](#gameplaytag-ability)  
>    2.2 [점프와 구르기 어빌리티](#jumpability-rollability)  
>    2.3 [던지기 어빌리티](#throwability)  
>    2.4 [공격 어빌리티와 후방 암살(뒷잡)](#attackability)  
>    2.5 [상호작용 어빌리티](#interactability)  
>    2.6 [점프암살 어빌리티](#jumpassassinability)  
>    2.7 [데미지 상호작용](#damage-interaction)  
>    2.8 [Extra : GameplayCue](#gameplaycue)  
> 3. [InventorySystem](#inventory)  
>    3.1 [TArray Or TMap](#tarray-or-tmap)  
>    3.2 [EquipmentComponent & EquipmentSlotComponent](#equipmentcomponent)  
> 4. [CameraManager](#camera-manager)  
> 5. [Animation Layering](#animation-layering)  
> 6. [AI](#ai)  
>    6.1 [순찰상태](#순찰상태)  
>    6.2 [경계상태](#경계상태)  
>    6.3 [공격상태](#공격상태)  
>    6.4 [탐색상태](#탐색상태)  
>    6.5 [AIPerception Age의 만료시점](#perception-age)  
>	 6.6 [SmartObject 탐색](#smart-object)  
> 7. [OSS Steam](#oss-steam)  
>	 7.1 [멀티 세션의 구조 설계](#멀티세션-구조설계)  
>	 7.2 [로비와 게임의 게임모드 분리](#게임모드-분리)  
>	 7.3 [세션 소스코드](#세션코드)  
> 8. [Spectator](#spectator)  
>	 8.1 [관전용 입력 맵핑](#관전-입력-맵핑)  
>	 8.2 [관전자 전환](#관전자-전환)  
>	 8.3 [리스폰](#리스폰)  
> 9. [Extra : LineTracing Attack](#linetracing-attack)
> 10. [Images](#images)


<a name="cmc"></a>
## CharacterMovementComponent

제 게임에는 클라이밍(파쿠르) 요소가 존재합니다. 이와같이 커스텀한 캐릭터의 움직임을 정의하기 위해선 **물리(속도, 충돌 등)**, **입력**, **애니메이션**에 대한 재정의가 필요했습니다.

그리고 `CharacterMovementComponent` 물리와 관련된 정보를 처리하기 위해 `CharacterMovementComponent`의 `MovementMode`를 이용했습니다.

그다음 `CMC`에서 `virtual void PhysCustom(float deltaTime, int32 Iterations);` 를 재정의하여 구현하였습니다.

 - USrCharacterMovementComponent.h
```cpp
UENUM(BlueprintType)
enum ESrMovementMode
{
	SMOVE_Parkour	UMETA(DisplayName = "Parkour"),
	SMOVE_END		UMETA(Hidden)
};

virtual void PhysCustom(float deltaTime, int32 Iterations) override;
void PhysClimb(float deltaTime, int32 Iterations);
```

`CharacterMovementComponent`은 `PerformMovement`에서 움직임과 관련된 메인 로직이 수행되는데, 아래와 같은 호출 순서를 가지게 됩니다.

![CMC호출흐름](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/CMC/CMC호출흐름.png?raw=true)

 > CMC 호출흐름

이때 `StartNewPhysics`에서 **MovementMode**에 따른 로직을 수행하는데, **MovementMode**를 변경하는 `SetMovementMode`함수를 마음대로 호출하게 되면 **서버 동기화 과정 중 어느 호출 시점에서 변경되는지 명확하게 단정짓기 어렵습니다.**

그래서 모든 **MovementMode**로직을 `StartNewPhysics`이전에 호출되는 `UpdateCharacterStateBeforeMovement`에서 수행하도록 변경하여 안정적으로 처리할 수 있게 만들어 **서버와 동기화 과정에서 문제가 발생하지 않도록 구조를 설계**했습니다.

 - USrCharacterMovementComponent.cpp
```cpp
void USrCharacterMovementComponent::UpdateCharacterStateBeforeMovement(float DeltaSeconds)
{
	if (!bWantsToClimb && ClimbingDelayTime < ClimbingDelayTimeLimit)
	{
		ClimbingDelayTime += DeltaSeconds;
	}

	//Falling상태일때 Climb상태를 지속적으로 체크해줍니다.
	if (OwningCharacter->IsFalling() || (OwningCharacter->bPressedJump && !bWantsToClimb))
	{
		if (ClimbingDelayTime >= ClimbingDelayTimeLimit)
		{
			//클라이밍 조건체크와 각도를 체크하여 유효하면 클라이밍으로 상태를 변경합니다.
			if (CanClimb() && CanClimbableAngle())
			{
				OwningCharacter->StopJumping();

				TryClimb();
			}
		}
	}
	//클라이밍 도중 점프를 하면 LedgeUp을 체크해줍니다.
	else if (OwningCharacter->bIsSpacePressed && bWantsToClimb)
	{
		if (!bIsLedgeUp && !bIsMantle && !bIsLeap)
		{
			if (CanLedgeUp())
			{
				TryLedgeUp();
			}
		}
	}

	Super::UpdateCharacterStateBeforeMovement(DeltaSeconds);
}
```

**[⬆ Back to Top](#top)**

<a name="asc"></a>
## AbilitySystemComponent

<a name="gameplaytag-ability"></a>
### 게임플레이 태그와 어빌리티

게임플레이 태그는 활용도가 굉장히 넓습니다. 단순히 표시만을 나타내기 위해서 사용하는 태그도 있고 태그와 일치하는 어빌리티를 막거나 태그에 따라서 사용중인 어빌리티가 막힐수도 있고 태그가 있어야만 호출되는 어빌리티를 만들 수도 있습니다.

![게임플레이태그](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/ASC/게임플레이태그.png?raw=true)

 > 게임플레이 태그 요소

첫번째 태그인 `Ability Tags`는 단순히 가시용입니다. 즉 보여주는 용도로 쓰이고 그외에는 아무런 의미가 없는것이죠. 

순서대로 `Activation Owned Tags`는 어빌리티가 활성화 되었을 때 어빌리티 시스템에 부여할 태그입니다.
`Activation Blocked Tags`는 어빌리티가 활성화 되었을 때 태그를 가진 어빌리티를 활성화 못하도록 합니다.
`Source Blocked Tags`도 동일한 의미지만 내부적으로는 다르게 쓰인다고 합니다. 큰 차이는 없습니다.
저는 어떤 상황에서도 `block`할 생각이였기에 2가지 모두에 적용했습니다. 일반적으로는 `Activation Blocked Tags`만 사용해도 크게 상관이 없습니다.

**[⬆ Back to Top](#top)**

<a name="jumpability-rollability"></a>
### 점프와 구르기 어빌리티

점프 어빌리티와 구르기 어빌리티는 가장 간단하게 구현할 수 있는 어빌리티입니다.
점프 어빌리티는 단순히 캐릭터의 점프 여부를 판단하여 Jumpd입력을 받게되면 점프하고 구르기 역시 동일하게 동작합니다.

 - USrAbility_Jump.cpp
```cpp
//점프키를 입력받으면 수행하는 함수입니다.
void USrAbility_Jump::CharacterJumpStart()
{
	ASrCharacter* SrCharacter = GetSrCharacterFromActorInfo();
	if (!SrCharacter)
	{
		return;
	}

	//이미 앉은 상태라면 일어섭니다.
	if (SrCharacter->bIsCrouched)
	{
		SrCharacter->UnCrouch();

		EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, false);
	}
	//점프키를 입력하지 않은상태라면(현재점프 상태가 아니라면) 점프를 수행합니다.
	else if (!SrCharacter->bPressedJump)
	{
		SrCharacter->Jump();
	}
}

//점프 여부를 체크하여 점프가 불가능하다면(땅에 붙어있거나) 어빌리티를 활성화하지 않습니다.
bool USrAbility_Jump::CanActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayTagContainer* SourceTags, const FGameplayTagContainer* TargetTags, FGameplayTagContainer* OptionalRelevantTags) const
{
	if (!Super::CanActivateAbility(Handle, ActorInfo, SourceTags, TargetTags, OptionalRelevantTags))
	{
		return false;
	}

	if (!SrCharacter->CanJump())
	{
		return false;
	}

	return true;
}
```

 - USrAbility_Roll.cpp
```cpp
void USrAbility_Roll::ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData)
{
	Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);

	//구르기 방향을 지정합니다.
	FVector ForwardVector, RightVector;
	GetDirection(OUT ForwardVector, OUT RightVector);

	//구르기 방향에 맞는 몽타주를 선택합니다.
	RollToPlayMontage = SelectDirectionalRollMontage(ForwardVector, RightVector);

	ASrCharacter* SrCharacter = Cast<ASrCharacter>(ActorInfo->AvatarActor.Get());
	if (SrCharacter && RollToPlayMontage)
	{
		//몽타주를 서버와 클라이언트 모두에게 재생하여 동기화해줍니다.
		SrCharacter->PlayMontage_Server(RollToPlayMontage);
	
		//애니메이션 델리게이트 바인딩이 멀티환경에서 제대로 동작하지 않아 비동기 딜레이 함수를 통해 어빌리티를 종료하였습니다.
		float DelayTime = RollToPlayMontage->GetPlayLength() - RollToPlayMontage->BlendOut.GetBlendTime();
		UAbilityTask_WaitDelay* WaitDelayTask = UAbilityTask_WaitDelay::WaitDelay(
			this,
			DelayTime
		);
		WaitDelayTask->OnFinish.AddDynamic(this, &ThisClass::OnRollEndAbiity);
		WaitDelayTask->ReadyForActivation();
	}
	else
	{
		//몽타주가 없으면 어빌리티를 종료합니다.
		OnRollEndAbiity();
	}
}
```

**[⬆ Back to Top](#top)**

<a name="throwability"></a>
### 던지기 어빌리티

던지기 어빌리티는 총2가지로 구성하였습니다. 포물선을 그리면서 날라가는 흘리기 형태(아무데나 던지기)의 던지기와 타겟을 향해 날라가는 던지기 형태(화살처럼 맞추기)입니다.

먼저 플레이어가 바라보는 방향으로 타겟을 탐지하고 타겟이 있으면 던지기를, 타겟이 없으면 흘리기를 수행했습니다.

**1. 흘리기**
흘리기는 키를 입력받고 `UGameplayStatics::PredictProjectilePath`함수를 통해 경로를 그립니다. 그리고 아이템 메쉬에서 `AddImpulse`에 `Velocity`값을 매개변수로 하여 호출하였습니다.

**2. 던져 맞추기**
던져 맞추기도 동일하게 `UGameplayStatics::PredictProjectilePath`를 통해서 타겟을 통해 발사하지만 이때 도착할 지점으로 타겟의 소켓정보를 받아 소켓위치로 던지게 만들었습니다.

 - USrAbility_Secondary.cpp
```cpp
//액터 스폰은 서버에서 이루어져야지만 동기화가 되기때문에 RPC를 이용하여 구현하였습니다.
void USrAbility_Secondary::ThrowToTarget_Implementation(FVector ViewStart, FRotator ViewRot, FVector LaunchVelocity, FVector SocketLocation)
{
	//던지기용 오브젝트를 스폰해줍니다.
	FTransform SpawnTransform(ViewRot, SocketLocation, FVector::OneVector);
	FActorSpawnParameters SpawnParameter;
	SpawnParameter.Owner = GetAvatarActorFromActorInfo();
	SpawnParameter.Instigator = GetSrCharacterFromActorInfo();
	SpawnParameter.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
	AActor* OutActor = GetWorld()->SpawnActor<AActor>(ActorToSpawn, SpawnTransform, SpawnParameter);

	//던질 경로를 예측하여 도착할때까지 걸리는 시간을 가져옵니다.
	FPredictProjectilePathResult PathResult;
	LastDestTime = GetLastDestTime(OUT PathResult, LaunchVelocity);

	//장비 인터페이스를 통해 Throw함수를 호출합니다.
	TScriptInterface<IEquipable> EquipableActor = OutActor;
	if (EquipableActor.GetInterface() != nullptr)
	{
		ASrCharacter* TargetActor = HitResult.bBlockingHit ? Cast<ASrCharacter>(HitResult.GetActor()) : nullptr;
		//타겟이 존재하면 타겟의 소켓위치로 던집니다.
		if (TargetActor)
		{
			FVector TargetLocation = TargetActor->GetMesh()->GetSocketLocation(TargetSocketName);
			EquipableActor->ThrownAtTarget(TargetLocation, ArcParam, bDebugThrowLine);
		}
		//타겟이 존재하지 않으면 바라보는 방향으로 던집니다.
		else if (!LaunchVelocity.IsNearlyZero())
		{
			EquipableActor->ThrownByVelocity(LaunchVelocity, LastDestTime);
		}
	}
}
```

**[⬆ Back to Top](#top)**

<a name="attackability"></a>
### 공격 어빌리티와 후방 암살(뒷잡)

먼저 공격 어빌리티에 있어서 2가지로 분기하였습니다. 하나는 `기본공격`, 다른 하나는 `후방암살`입니다.
`일반 공격`은 기본적으로 적이 없거나 적의 정면일때만 수행하고 `후방암살`은 적이 있을때 후방조건을 탐색하는 방식으로 설계하였습니다.
이때 플레이어의 바향과 적의 방향을 내적하여 `내적한 값이 양수` 일때만 후방으로 처리하였습니다.

![후방암살조건](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/ASC/후방암살조건.png?raw=true)

 > 후방암살조건

 - USrAbility_Primary.cpp
```cpp
bool USrAbility_Primary::IsBehindTarget(FVector InTargetDirection)
{
	FVector AvatarActorFowardDir = GetAvatarActorFromActorInfo()->GetActorForwardVector();
	float DotToDirection = FVector::DotProduct(AvatarActorFowardDir, InTargetDirection);

	if (DotToDirection <= 1.f && DotToDirection >= FMath::Cos(PI / 3))
	{
		return true;
	}

	return false;
}
```

그리고 `모션워핑`을 이용하여 플레이어의 위치를 보간했습니다.
위치를 보간해주는 변수를 암살 어빌리티에 선언한 다음 암살의 종류에 따라 다른 Offset값을 지정하여 워핑값을 셋팅했습니다.

![뒷잡모션워핑](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/ASC/뒷잡모션워핑.png?raw=true)

 > 뒷잡 모션워핑 변수

 **[⬆ Back to Top](#top)**

<a name="interactability"></a>
### 상호작용 어빌리티

상호작용을 하기 위해선 상호작용하기 위한 오브젝트를 찾아야 합니다. 그래서 우선 `오브젝트를 찾는 역할을 해주는 기능`이 필요했습니다. → **탐색기능이 필요하다!**

`탐색기능`에 대해 찾아보던 중 `GASShooter`와 `Lyra`에서 어빌리티로 관리하는 것을 알게 되었습니다. `탐색 기능`은 상호작용 어빌리티의 연관성을 고려해 봤을때 어빌리티로 관리하는 방법이 적절해 보였습니다.

```cpp
//탐색 어빌리티 클래스
class SICARIUS_API USrAbility_DetectInteraction : public USrGameplayAbility

//상호작용 어빌리티 클래스
class SICARIUS_API USrAbility_Interact : public USrGameplayAbility
```

다음으로 탐색구조를 다음과 같이 만들었습니다.
DetectInteraction어빌리티로 주기적으로 상호작용 오브젝트 탐색 &#8594; 오브젝트가 탐색되었다면  키 입력 기다림 &#8594; 키 입력을 받으면 Interact어빌리티 활성화

그리고 상호작용 어빌리티는 입력 이벤트를 통해서만 호출하게 설정해주었습니다.

![상호작용어빌리티이벤트](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/ASC/상호작용어빌리티이벤트.png?raw=true)

 > 상호작용 어빌리티의 호출 조건

 - USrAbility_DetectInteraction.cpp
```cpp
void USrAbility_DetectInteraction::ActivateInteractAbility()
{
	FGameplayEventData GameplayEventData = FGameplayEventData();
	GameplayEventData.TargetData = TargetData;
	GameplayEventData.Target = TargetActor;
	GameplayEventData.Instigator = GetAvatarActorFromActorInfo();
	SendGameplayEvent(FGameplayTag("Ability.Interaction"), GameplayEventData);
}
```

**1. 상호작용 오브젝트 탐색**

```cpp
//탐색은 일정 주기를 반복하며 수행합니다.
GetWorld()->GetTimerManager().SetTimer(TimerHandle, this, &UWaitForInteractableTarget::DetectTarget, InteractionRate, true);

void UWaitForInteractableTarget::DetectTarget()
{
	AActor* AvatarActor = Ability->GetCurrentActorInfo()->AvatarActor.Get();
	if (!AvatarActor)
		return;

	FCollisionQueryParams Params(SCENE_QUERY_STAT(UWaitForInteractableTarget), false);
	Params.AddIgnoredActor(AvatarActor);

	FVector TraceStart = StartLocation.GetTargetingTransform().GetLocation();
	FVector TraceEnd;

	//카메라를 기준으로 가장 끝 지점의 Trace위치를 반환합니다.
	AimWithPlayerController(AvatarActor, Params, TraceStart, InteractionRange, OUT TraceEnd);

	//플레이어의 위치에서 카메라 방향의 끝지점으로 타겟을 탐색합니다.
	FHitResult HitResult;
	LineTrace(OUT HitResult, GetWorld(), TraceStart, TraceEnd, TraceProfile.Name, Params);

	//타겟을 찾으면 타겟 변경 여부에 따라 델리게이트를 브로드캐스팅해줍니다.
	if (HitResult.bBlockingHit)
	{
		bool bTargetChanged = true;

		if (InteractTargetData.Num() > 0)
		{
			//타겟을 잃었는지 여부를 판단합니다.
			const auto* PrevTarget = InteractTargetData.Get(0)->GetHitResult()->GetActor();
			//타겟이 같다면 타겟을 잃지 않았습니다.
			if (PrevTarget == HitResult.GetActor())
			{
				bTargetChanged = false;
			}
			//타겟이 다르다면 타겟 정보를 전달하고 현재 타겟의 정보를 잃어버렸음을 nullptr로 뿌립니다.
			else if (PrevTarget != nullptr)
			{
				OnTargetLostedEvent.Broadcast(InteractTargetData, nullptr);
			}
		}

		//타겟이 변경되었다면 변경된 타겟 정보를 뿌려줍니다.
		if (bTargetChanged)
		{
			InteractTargetData = StartLocation.MakeTargetDataHandleFromHitResult(Ability, HitResult);
			OnTargetChangedEvent.Broadcast(InteractTargetData, HitResult.GetActor());
		}
	}
	else
	{
		//아무런 타겟도 찾지 못한경우엔 nullptr로 타겟정보를 무한정 잃어버림으로 뿌려줍니다.
		if (InteractTargetData.Num() > 0)
		{
			OnTargetLostedEvent.Broadcast(InteractTargetData, nullptr);
		}
	}
}
```

**2. 상호작용 키입력**
상호작용 오브젝트가 탐지되면 `UAbilityTask_WaitInputPress`태스크를 통해 입력을 기다립니다.
그리고 입력이 들어오면 `Interact`어빌리티를 활성화하였습니다.

 - USrAbility_DetectInteraction.cpp
```cpp
void USrAbility_DetectInteraction::InteractionInputEvent()
{
	WaitInputPressTask = UAbilityTask_WaitInputPress::WaitInputPress(this);

	WaitInputPressTask->OnPress.AddDynamic(this, &ThisClass::OnTriggerInteractAbility);
	WaitInputPressTask->ReadyForActivation();
}

void USrAbility_DetectInteraction::OnTriggerInteractAbility(float TimeWaited)
{
	//SendGameplayEvent 내장함수를 통해 Interact어빌리티를 활성화해줍니다.
	if (TargetData.Num() > 0)
	{
		FGameplayEventData GameplayEventData;
		GameplayEventData.EventTag = TAG_ABILITY_INTERACTION;
		GameplayEventData.TargetData = TargetData;
		GameplayEventData.Instigator = GetAvatarActorFromActorInfo();
		//타겟 정보를 Interact어빌리티에게 전달합니다. 이 코드가 없으면 상호작용 어빌리티에서 관련 로직을 수행할 수 없습니다.
		GameplayEventData.Target = TargetActor;
		SendGameplayEvent(GameplayEventData.EventTag, GameplayEventData);
	}

	//기존의 태스크를 초기화해줍니다.
	WaitInputPressTask->OnPress.Clear();
	WaitInputPressTask->EndTask();
	InteractionInputEvent();
}
```

**3. 상호작용 활성화**
Interact어빌리티가 활성화되면 TargetActor를 통해 수집 및 장착을 수행합니다.

```cpp
void USrAbility_Interact::ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData)
{
	Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);

	//기타 예외처리 로직들...

	//상호작용 로직은 서버를 통해서 처리됩니다.(액터를 삭제하는 로직이 들어 있기 때문)
	if (HasAuthority(&CurrentActivationInfo))
	{
		//상호작용만을 수행하는 아이템 로직을 먼저 수행합니다.
		if (IInteractable* InteractableTargetActor = Cast<IInteractable>(TargetActor.Get()))
		{
			if (!InteractableTargetActor->IsInteracted())
			{
				InteractableTargetActor->Interact();
			}
		}

		//PickUp할 수 없다면 Equip을 수행하지 않습니다.
		if (IPickupable* PickupableTargetActor = Cast<IPickupable>(TargetActor.Get()))
		{
			USrInventoryComponent* InventoryComponent = AvatarPawn->GetController()->FindComponentByClass<USrInventoryComponent>();
			//인벤토리에 장비를 추가해줍니다.
			PickupableTargetActor->PickupItem(InventoryComponent);

			TargetActor->SetActorEnableCollision(false);
			TargetActor->SetLifeSpan(1.f);
		}
		else
		{
			EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
			return;
		}

		//PickUp할 수 있다고해서 Equip을 할수있는건 아닙니다.
		if (IEquipable* EquipableTargetActor = Cast<IEquipable>(TargetActor.Get()))
		{
			USrEquipmentSlotComponent* EquipmentSlotComponent = AvatarPawn->FindComponentByClass<USrEquipmentSlotComponent>();
			//장비를 장착해줍니다.
			EquipableTargetActor->EquipItem(EquipmentSlotComponent);
		}
		else
		{
			EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
			return;
		}
	}

	PlayInteractMontage(InteractMontage);
}
```

**[⬆ Back to Top](#top)**

<a name="jumpassassinability"></a>
### 점프암살 어빌리티

`Styx`라는 게임에서 점프동작 중 아래에 적이 있는지 체크하고 적이 존재할때 키를 입력하면 암살을 수행하는 암살 시스템이 존재합니다.
이 기능 마침 제가 가지고 있는 애니메이션에서 비슷하게 동작했고 플레이어의 아래 방향에 적이 있을때 점프 암살을 사용하는 방식으로 설계하였습니다.

![점프암살흐름](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/ASC/점프암살흐름.png?raw=true)

 > 점프암살 흐름

점프 암살의 시퀀스에서 봤듯이 점프 암살에는 처리해야할 2가지 요소가 있었습니다.

**첫째. 아래의 적을 어떻게 체크할 것인가?**
**둘째. 어떤 키를 통해 점프액션을 구현할 것인가?**

이를 위해 몇가지 사전 조건을 만들었습니다.

`**입력 방식**` : E키를 이용해 점프 암살을 수행합니다.
`**체크 방식**` : 게임의 시작부터 끝까지 Falling상태일 때 **SingleSweepSphere**로 타겟을 지속적으로 탐지합니다.

지속적인 체크가 필요했기 때문에 어빌리티 시스템에서 제공해주는 `AbilityTask_Repeat`를 사용하였습니다.


**첫째. 적 체크 수행 방식**

 - USrAbility_JumpAssassin.cpp
```cpp
RepeatTask = UAbilityTask_Repeat::RepeatAction(this, 0.1f, 50);
	RepeatTask->OnPerformAction.AddDynamic(this, &ThisClass::OnDetectTarget);
	RepeatTask->ReadyForActivation();
```

그리고 수행 도중 타겟을 찾으면 `UAbilityTask_WaitInputPress`를 이용하여 입력이 들어올때까지 기다렸습니다.
이 함수의 경우엔 지속적으로 실행되는 `Tick과 유사`하기때문에 입력델리게이트가 `실행 중일때는 다시 입력을 받지 않도록` 하였습니다.

**둘째. 어떤 키를 사용할 것인가**

 - USrAbility_JumpAssassin.cpp
```cpp
void USrAbility_JumpAssassin::OnDetectTarget(int32 ActionNumber)
{
	ASrCharacter* PlayerCharacter = Cast<ASrCharacter>(GetAvatarActorFromActorInfo());

	if (!IsValid(PlayerCharacter))
	{
		EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, true);
		return;
	}
	
	//플레이어가 낙하중이 아니라면 탐지 로직 수행을 중단합니다.
	if (ActionNumber > 1 && !PlayerCharacter->IsFalling())
	{
		EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, true);
	}

	FVector Start = PlayerCharacter->GetActorLocation() + FVector::DownVector * StartOffset;
	FVector Direction = FVector::DownVector;
	FVector End = Start + Direction * MaxRange;

	FCollisionQueryParams Params;
	Params.AddIgnoredActor(GetAvatarActorFromActorInfo());
	bool bHit = GetWorld()->SweepSingleByProfile(
		HitResult,
		Start,
		End,
		FQuat::Identity,
		ProfileName,
		FCollisionShape::MakeSphere(SweepSphereRadius),
		Params
	);

	//타겟을 찾았다면 입력을 기다립니다.
	if(IsTargetFound())
	{
		if (WaitInputPressTask && WaitInputPressTask->IsActive())
		{
			return;
		}

		WaitInputPressTask = UAbilityTask_WaitInputPress::WaitInputPress(this);

		WaitInputPressTask->OnPress.AddDynamic(this, &ThisClass::OnTriggerAssassinAbility);
		WaitInputPressTask->ReadyForActivation();
	}
}
```

**[⬆ Back to Top](#top)**

<a name="damage-interaction"></a>
### 데미지 상호작용

언리얼에서는 `기본적`으로 **TakeDamage**라는 데미지 상호작용 기능이 있습니다.

이와 비슷하게 `GAS`에서는 **GameplayEffect**를 이용해서 자신이나 다른 플레이어의 **Attributes(스탯정보)**를 변경하는 데미지 상호작용 기능들을 제공합니다.

**ModifierMagnitudeCalculations와 GameplayEffectExecutionCalculation**

언리얼의 **`GAS`**에서는 이러한 **데미지상호작용을 보조해주는 2가지 기능**을 제공해주는데, 각각 `**ModifierMagnitudeCalculations(MMC)**`과`**GameplayEffectExecutionCalculations(ExecCalc)**`입니다.

**ModifierMagnitudeCalculations**

`MMC`는 **GameplayEffect**를 수정할 수 있는 강력한 기능을 제공해주며, 체력과 장비의 방어력, 버프 정보 등을 **정적으로 계산하여 변경된 ResultValue를 반환해주는 역할**을 합니다.
플레이어의 상태나 속성이 고정된 경우에 유용하게 쓸 수 있습니다.

**GameplayEffectExecutionCalculations**

반면에 `ExecCalc`는 `MMC`의 기능에서 한걸음 더 나아가 **GameplayEffect의 실행시점에서 대상의 데미지량과 방어력을 동적으로 계산합니다.**

두 기능 모두 제 게임에서는 크게 차이가 없습니다. 그래서 그 중 `GameplayEffectExecutionCalculations`를 이용하여 구현하였습니다.



**1. GameplayEffectExecutionCalculation의 구조**

`ExecCalc`의 구조는 정말간단합니다. 아래에 있는 기본 `UObject`를 상속받아서 구현된 것이 `UGameplayEffectCalculation`인데, 계산에 사용할 **Attribute**의 정보를 담을 배열만이 존재합니다.

 - UGameplayEffectCalculation.cpp
```cpp
UCLASS(BlueprintType, Blueprintable, Abstract)
class GAMEPLAYABILITIES_API UGameplayEffectCalculation : public UObject
{
	GENERATED_UCLASS_BODY()
protected:
	/** 계산에 사용할 Attribute정보가 담긴 배열 */
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Attributes)
	**TArray<FGameplayEffectAttributeCaptureDefinition> RelevantAttributesToCapture;**
};
```

위 `UGameplayEffectCalculation`클래스를 상속받아서 구현된 것이 `UGameplayEffectExecutionCalculation`이며 핵심만 가져오면 아래와 같습니다.

 - UGameplayEffectExecutionCalculation.cpp
```cpp
UCLASS(BlueprintType, Blueprintable, Abstract)
class GAMEPLAYABILITIES_API UGameplayEffectExecutionCalculation : public UGameplayEffectCalculation
{
	GENERATED_UCLASS_BODY()
public:
	/**
	 * GameplayEffect가 실행될때 호출됩니다.
	 * 기본적으로BlueprintNativeEvent이기 때문에 c++에서는 함수명_Implementation으로 재정의해서 사용해야합니다.
	 * 
	 * @param ExecutionParams		Parameters for the custom execution calculation
	 * @param OutExecutionOutput	[OUT] Output data populated by the execution detailing further behavior or results of the execution
	 */
	UFUNCTION(BlueprintNativeEvent, Category="Calculation")
	**void Execute(const FGameplayEffectCustomExecutionParameters& ExecutionParams, FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const;**
};
```

**2. GameplayEffectExecutionCalculation의 구현**

`Execute`함수와 `Calculation`에서 사용할 `Attribute정보`를 담을 구조체를 만들어주고 체력과 데미지를 통해 최종 체력을 계산하도록 했습니다.
그리고 생성자에서 `Caclulcation`에서 사용할 `Attribute정보`를 `RelevantAttributesToCapture`에 추가해줬습니다.

 - USrDamageExecution.h
```cpp
UCLASS()
class SICARIUS_API USrDamageExecution : public UGameplayEffectExecutionCalculation
{
	GENERATED_BODY()
public:
	USrDamageExecution();
protected:
	virtual void Execute_Implementation(const FGameplayEffectCustomExecutionParameters& ExecutionParams, FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const override;

protected:
	FGameplayEffectAttributeCaptureDefinition 	HealthDef;
	FGameplayEffectAttributeCaptureDefinition	DamageDef;
};
```

 - USrDamageExecution.cpp
```cpp
USrDamageExecution::USrDamageExecution()
{
	//Target : 이펙트를 받는 곳
	HealthDef = FGameplayEffectAttributeCaptureDefinition(USrAttributeSet::GetHealthAttribute(), EGameplayEffectAttributeCaptureSource::Target, false);

	//Source : 이펙트를 전달하는 곳
	DamageDef = FGameplayEffectAttributeCaptureDefinition(USrAttributeSet::GetDamageAttribute(), EGameplayEffectAttributeCaptureSource::Source, true);

	RelevantAttributesToCapture.Add(HealthDef);
	RelevantAttributesToCapture.Add(DamageDef);
}

//직접적인 최종 데미지를 계산하는 코드입니다.
void USrDamageExecution::Execute_Implementation(const FGameplayEffectCustomExecutionParameters& ExecutionParams, FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const
{
	//EvaluationParameters에서 Target의 Attribute중 Health에 대한 데이터를 가져옵니다.
	float CurrentHealth = 0.0f;
	ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(DamageStatics().HealthDef, EvaluationParameters, CurrentHealth);

	//EvaluationParameters에서 Source의 Attribute중 BaseDamage에 대한 데이터를 가져옵니다.
	float BaseDamage = 0.0f;
	ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(DamageStatics().DamageDef, EvaluationParameters, BaseDamage);

	//간단하게 데미지를 Clamping해줍니다.
	const float DamageDone = FMath::Clamp(BaseDamage, 0.0f, CurrentHealth);

	if (DamageDone > 0.0f)
	{
		//GameplayEffect의 최종 전달값(데미지값)을 어느 속성을 변경하는데 사용할지 지정해줍니다.
		OutExecutionOutput.AddOutputModifier(FGameplayModifierEvaluatedData(USrAttributeSet::GetHealthAttribute(), EGameplayModOp::Additive, -DamageDone));
	}
}
```

**3. GameplayEffect 실행**

데미지를 적용시키기 위해 블루프린트에서 `GE_Damage_Dagger`클래스를 생성한 뒤 **Execution**에 저희가 만든 클래스를 등록해주고 추가적인 값을 셋팅해줬습니다.

![GE_Dagger셋팅](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/ASC/GE_Dagger셋팅.png?raw=true)

 > GE_Dagger 셋팅

다음으로 공격 어빌리티에서 타겟이 탐지되었을때 호출되는 `OnTargetPresent`함수에서 `ApplyGameplayEffectToTarget`함수를 호출하여 완성했습니다.

```cpp
void USrAbility_Primary::OnTargetPresent(const FHitResult& InHitResult)
{
	//HitResult정보를 ApplyGameplayEffectToTarget에서 사용할 수 있도록 가공해주는 라이브러리함수입니다.
	FGameplayAbilityTargetDataHandle TargetData = UAbilitySystemBlueprintLibrary::AbilityTargetDataFromHitResult(InHitResult);

	//TargetData에 GameplayEffect를 적용시켜 줍니다.
	ApplyGameplayEffectToTarget(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, TargetData, DamageEffectClass, 1.f);
}
```

**[⬆ Back to Top](#top)**

<a name="gameplaycue"></a>
## Extra : GameplayCue

```cpp
//TODO
```

**[⬆ Back to Top](#top)**

<a name="inventory"></a>
## InventorySystem

제 게임의 인벤토리엔 장비들을 저장할 용도로 사용하고 필요에 따라 UI에서 인벤토리에 접근하여 순서대로 띄워주는 방식을 사용하고자 했습니다.

<a name="tarray-or-tmap"></a>
### TArray Or TMap

인벤토리의 아이템은 삭제와 추가가 빈번하게 일어날 수 있습니다. 또한 중간에서 삭제될수도있고 중간에 추가될수도있습니다.
시카리우스에서 인벤토리는 항상 아이템의 갯수만큼 공간이 존재해야하고 새로운 아이템은 맨마지막 아이템의 다음에 추가되어야 합니다. 하지만 인덱스를 통해 UI에서 접근할수도 있어야 합니다.
그래서 검색시마다 순회를 해야하는 TArray의 단점과 삭제와 검색이 빠른 TMap 중 TArray를 선택했습니다.

<a name="equipmentcomponent"></a>
### EquipmentComponent & EquipmentSlotComponent

인벤토리와 장비는 비슷한 점을 가지고 있습니다.

인벤토리는 아이템을 저장하는데 사용하지만 장비는 플레이어에게 귀속되거나 부착되는 방식으로 사용됩니다. 즉 장비 시스템에서의 핵심은 귀속과 장착입니다.
그래서 저는 장비의 장착과 인벤토리를 구분하여 컴포넌트로 관리하였습니다. 그리고 장비슬롯 컴포넌트를 보다 상위컴포넌트로 만들어 인벤토리관리, UI, 장비장착을 수행하도록 구성하였습니다.


장착용 아이템은 구조체 형태로 관리되며 장착될때 사용할 어빌리티와 아이템 정보를 가지고 있습니다.
```cpp
//아이템 한개의 정보를 담고 있는 구조체입니다.
struct FEquipmentItem
{
	GENERATED_BODY()

private:
	UPROPERTY()
	USrEquipmentItemObject* EqiupmentItemObject = nullptr;

	UPROPERTY()
	TObjectPtr<USrAbilitySet> EquipmentItemAbilitySet;
};

UCLASS(Blueprintable, BlueprintType)
class SICARIUS_API USrEquipmentItemObject : public UObject
{
	GENERATED_BODY()

public:
	//장비의 장착 정보
	UPROPERTY(EditDefaultsOnly)
		FSrEquipmentAttachData AttachData;

private:
	UPROPERTY(Replicated)
	TArray<AActor*> SpawnedActors;
};
```

그리고 장비 컴포넌트에서 장비의 어빌리티정보와 장비를 장착하는 로직을 수행하도록 구성하였습니다.

 - USrEquipmentComponent.h
```cpp
//장착할 모든 장비의 정보를 가지고 있으며 장비의 실질적인 장착을 담당합니다.
class SICARIUS_API USrEquipmentComponent : public UPawnComponent
{
	GENERATED_BODY()

public:
	USrEquipmentComponent(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

	UFUNCTION(BlueprintCallable)
	USrEquipmentItemObject* EquipItem(TSubclassOf<USrEquipmentItemObject> EquipmentItemObject);

	UFUNCTION(BlueprintCallable)
	void UnEquipItem(USrEquipmentItemObject* EquipmentItemObject);

	USrAbilitySystemComponent* GetAbilitySystemComponent() const;

private:
    //장착된 장비의 정보를 담고있는 배열입니다.
	UPROPERTY()
		TArray<FEquipmentItem> EquipmentItems;
};
```

 - USrEquipmentComponent.cpp
```cpp
USrEquipmentItemObject* USrEquipmentComponent::EquipItem(TSubclassOf<USrEquipmentItemObject> EquipmentItemObject)
{
	USrEquipmentItemObject* EquipmentItemResult = nullptr;

	check(EquipmentItemObject);

    //혹여나 서버에서 실행하고 있지 않다면 null을 반환합니다.
	if ((GetOwnerRole() != ENetRole::ROLE_Authority))
	{
		return EquipmentItemResult;
	}

    //TSubClassOf의 CDO를 가져옵니다.
	const USrEquipmentItemObject* EquipmentItemCDO = EquipmentItemObject.GetDefaultObject();

	FEquipmentItem& NewEquipmentItem = EquipmentItems.AddDefaulted_GetRef();
	NewEquipmentItem.EqiupmentItemObject = NewObject<USrEquipmentItemObject>(GetOwner());
	NewEquipmentItem.EqiupmentItemObject->SetOwner(GetOwner());
    //무기의 장착 위치
	NewEquipmentItem.EqiupmentItemObject->AttachData = EquipmentItemCDO->AttachData;

    //Ability의 정보들을 담고 있는 TObjectPtr을 깊은복사로 복사해줍니다.
    //깊은 복사를 사용하는 이유는 멀티환경에서 TObjectPtr을 그대로 가져오게되면 모든 클라와 서버가 같은 포인터를 가지게 됩니다.
	NewEquipmentItem.EquipmentItemAbilitySet = DuplicateObject<USrAbilitySet>(EquipmentItemCDO->AbilitySets, GetOwner());
	EquipmentItemResult = NewEquipmentItem.EqiupmentItemObject;

    //무기를 스폰하고 장착해줍니다.
	EquipmentItemResult->SpawnEquipmentItem();

    //AbilitySet에 있는 어빌리티들을 모두 추가해줍니다.
	USrAbilitySystemComponent* ASC = GetAbilitySystemComponent();
	if (ASC)
	{
		for (const auto& Item : EquipmentItems)
		{
			if (Item.EquipmentItemAbilitySet != nullptr)
			{
				Item.EquipmentItemAbilitySet->GiveAbilities(ASC);
			}
		}
	}

	return EquipmentItemResult;
}

void USrEquipmentComponent::UnEquipItem(USrEquipmentItemObject* EquipmentItemObject)
{
    //혹여나 서버에서 실행하고 있지 않다면 null을 반환합니다.
	if ((GetOwnerRole() != ENetRole::ROLE_Authority))
	{
		return;
	}

	if (EquipmentItemObject != nullptr)
	{
		for (auto It = EquipmentItems.CreateIterator(); It; ++It)
		{
			auto& EquipmentItem = *It;
			if (EquipmentItem.EqiupmentItemObject == EquipmentItemObject)
			{
				if (USrAbilitySystemComponent* ASC = GetAbilitySystemComponent())
				{
                    //어빌리티의 정보를 제거해줍니다.
					if (EquipmentItem.EquipmentItemAbilitySet != nullptr)
					{
						EquipmentItem.EquipmentItemAbilitySet->RemoveAbilities(ASC);
					}
				}

                //장착된 무기를 제거합니다.
				EquipmentItemObject->DestroyEqiupmentItem();


				It.RemoveCurrent();
				break;
			}
		}
	}
}
```

**[⬆ Back to Top](#top)**

위 장비 장착 함수들은 모두 Interact 어빌리티(상호작용)나 UI를 통해서 동작합니다. 그러기 떄문에 상위 EquipmentSlotComponent에서 이 모든 과정을 수행합니다.

 - USrEquipmentSlotComponent.h
```cpp
UCLASS()
class SICARIUS_API USrEquipmentSlotComponent : public UPawnComponent, public IEquipmentUIInterface
{
	GENERATED_BODY()

public:
	/*
	* 장비를 장착하는 메인 로직으로 장비장착과 UI에 이벤트를 전달해줍니다.
	*/
	UFUNCTION(Server, Reliable, BlueprintCallable)
		void SetCurrentSlotItem(int32 SlotIndex);

	//장비를 슬롯에 추가해주는 함수로 현재 장비하고 있는 아이템을 실질적으로 장착합니다.
	UFUNCTION(BlueprintCallable)
		void EquipItemInSlot();

	//장비를 슬롯에서 제거해주는 함수로 현재 장비하고 있는 아이템을 실질적으로 제거합니다.
	UFUNCTION(BlueprintCallable)
		void UnEquipItemInSlot();

private:
	//플레이어가 가지고 있는 아이템의 슬롯개수입니다.
	UPROPERTY(Replicated)
		TArray<TObjectPtr<USrInventoryItemObject>> EquipmentSlots;

	//현재 장비하고 있는 아이템 정보입니다.
	UPROPERTY(Replicated)
		TObjectPtr<USrEquipmentItemObject> EquippedItem;

	UPROPERTY()
		int32 MaxSlots = 3;

	//Current EquipmentItem Index
	UPROPERTY(Replicated)
		int32 CurrentSlotIndex = -1;
};
```

이때 모든 로직은 서버를 통해서 진행되며 기존의 장비체크 후 장착해제, 해제한 슬롯에 장착할 장비를 착용합니다.

 - USrEquipmentSlotComponent.h
```cpp
void USrEquipmentSlotComponent::SetCurrentSlotItem_Implementation(int32 SlotIndex)
{
	if (EquipmentSlots.IsValidIndex(SlotIndex) && CurrentSlotIndex != SlotIndex)
	{
		UnEquipItemInSlot();

		CurrentSlotIndex = SlotIndex;

		EquipItemInSlot();
	}
}

void USrEquipmentSlotComponent::EquipItemInSlot()
{
	USrEquipmentComponent* EquipmentComponent = FindEquipmentComponent();
	check(EquipmentComponent);

	USrInventoryItemObject* CurrentInventoryItem = GetCurrentSlotItem();
	if (CurrentInventoryItem != nullptr)
	{
		if (CurrentInventoryItem->EquipmentItemObject != nullptr)
		{
			//EquipmentComponent의 장비 장착함수를 호출합니다.
			EquippedItem = EquipmentComponent->EquipItem(CurrentInventoryItem->EquipmentItemObject);
			EquippedItem->SetOwner(GetPawn<APawn>());

			//HUD에 인벤토리 이벤트를 전달하여 인벤토리UI를 알맞는 아이템 이미지로 대체합니다.
			BroadcastEquipItemChanged_Client(CurrentInventoryItem->ItemBrush);
		}
	}
	else
	{
		FSlateBrush EmptyBrush;
		EmptyBrush.TintColor = FSlateColor(FColor::Transparent);
		BroadcastEquipItemChanged_Client(EmptyBrush);
	}
}

void USrEquipmentSlotComponent::UnEquipItemInSlot()
{
	USrEquipmentComponent* EquipmentComponent = FindEquipmentComponent();
	check(EquipmentComponent);

	if (EquippedItem != nullptr && EquipmentSlots.IsValidIndex(CurrentSlotIndex))
	{
		//EquipmentComponent의 장비 장착해제함수를 호출합니다.
		EquipmentComponent->UnEquipItem(EquippedItem);
		EquippedItem = nullptr;

		//HUD에 인벤토리 이벤트를 전달하여 인벤토리UI를 비워줍니다.
		FSlateBrush EmptyBrush;
		EmptyBrush.TintColor = FSlateColor(FColor::Transparent);
		BroadcastEquipItemChanged_Client(EmptyBrush);
	}
}

```

최종적으로 수행되는 장비 함수 흐름은 다음과 같습니다.

 > 장비 해제시 : UnEquipItemInSlot -> UnEquipItem -> RemoveAbilities

 > 장비 장착시 : EquipItemInSlot -> EquipItem -> GiveAbilities

**[⬆ Back to Top](#top)**

<a name="camera-manager"></a>
## CameraManager

유저에게 있어서 화면을 보여주는 역할을 하는 카메라가 끊기거나 난장판이 나면 유저 입장에서는 눈이 굉장히 피곤합니다.

관련 영상 : https://youtu.be/ZzHWrR-Vejc

이렇게 끊기는 원인을 찾아보면 `SpringArmComponent`에 있습니다. `SpringArmComponent`에는 `TargetArmLength`라는 것이 존재하는데 이것이 카메라와 플레이어의 거리이며 이 부분을 `SpringArmComponent`의 `BlendLocations`함수로 조절할 수 있습니다.

 - USpringArmComponent.cpp
```cpp
FVector USpringArmComponent::BlendLocations(const FVector& DesiredArmLocation, const FVector& TraceHitLocation, bool bHitSomething, float DeltaTime)
{
		return bHitSomething ? TraceHitLocation : DesiredArmLocation;
}
```

기존의 함수는 단순히 카메라가 충돌한 위치를 반환해줍니다. 저희는 이 함수를 재정의해서 아래와 같이 사용하면 충돌한 위치와 현재위치를 적절히 보간하여 부드럽게 전환시킬 수 있습니다.

 - USrSpringArmComponent.cpp
```cpp
FVector USrSpringArmComponent::BlendLocations(const FVector& DesiredArmLocation, const FVector& TraceHitLocation, bool bHitSomething, float DeltaTime)
{
	if (bHitSomething)
	{
		FVector NewInterpLocation = FMath::VInterpTo(PrevInterpLocation, TraceHitLocation, DeltaTime, HitInterpSpeed);
		PrevInterpLocation = NewInterpLocation;

		RemaningInterpTime = HitInterpTime;

		return NewInterpLocation;
	}

	if (RemaningInterpTime > 0.0f)
	{
		RemaningInterpTime -= DeltaTime;
		FVector NewInterpLocation = FMath::VInterpTo(PrevInterpLocation, DesiredArmLocation, 1.0f - (RemaningInterpTime / HitInterpTime), HitInterpSpeed);
		PrevInterpLocation = NewInterpLocation;

		return NewInterpLocation;
	}

	PrevInterpLocation = DesiredArmLocation;
	return DesiredArmLocation;
}
```

**[⬆ Back to Top](#top)**

<a name="animation-layering"></a>
## Animation Layering

애니메이션 시스템을 만들때 쓸데없이 늘어나는 state캐싱과 애니메이션 상태에 따라서 무수히 많아지는 스테이트를 줄이고 싶었습니다.
그래서 정보를 찾다가 `Lyra샘플`에서 `Animation Layer Interface`와 `Animation Layer`를 사용해 굉장히 깔끔하게 구현하는 것을 보고 괜찮은 방법이라고 생각했고 제 프로젝트에 적용해봤습니다.

![AnimLayerBP](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/AnimationLayer/AnimLayerBP.png?raw=true)

 > 애니메이션 레이어와 인터페이스

아래와 같이 인터페이스에 사용할 애니메이션 상태들을 등록해두고 애니메이션 레이어에서 인터페이스 함수를 실제로 구현하고 이를 `애니메이션BP`에서 가져와서 사용할 수 있습니다.

![AnimLayerInterface](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/AnimationLayer/AnimLayerInterface.png?raw=true)

 > Idle, Jump 등의 상태의 애니메이션 인터페이스 생성

실제로 구현된 애니메이션 레이어를 플레이어의 애니메이션 레이어에 등록시킨 다음 애니메이션 블루프린트에서는 인터페이스 함수를 호출합니다.

 - ASrCharacter.cpp
```cpp
void ASrCharacter::OnConstruction(const FTransform& Transform)
{
	Super::OnConstruction(Transform);

	/* Sets the default animation layer.*/
	if (IsValid(AnimLayerClass))
	{
		GetMesh()->LinkAnimClassLayers(AnimLayerClass);
	}
}
```

![AnimLayerCall](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/AnimationLayer/AnimLayerCall.png?raw=true)

 > 레이어의 애니메이션 호출

**[⬆ Back to Top](#top)**

<a name="ai"></a>
## AI

암살게임에서 AI는 경우 순찰, 경계, 탐색, 공격과 같이 상황에 따라 다양한 행동 패턴을 보입니다.
이를 근거로 제 게임에서도 **순찰**, **경계**, **탐색**, **공격**의 4가지의 상태를 가지도록 하였습니다.

또한 이 과정에서 `EQS`와 `SmartObject`를 통해 **공격**과 **탐색**을 다채롭게 구현하였습니다.

![AI행동흐름](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/AI/AI행동흐름.png?raw=true)

 > 상태별 흐름

`AIPerception`에 의해 반응을 하게되면 최상위 상태인 공격, 경계상태로 시작하여 최하위 상태(순찰)까지 내려가는 구조를 가집니다.
상태 변경의 경우 `AIPerceptionComponent`에서 제공해주는 `OnTargetPerceptionUpdated`델리게이트를 통해 구현하였습니다.

 - ASrEnemyBotController.cpp
```cpp
void ASrEnemyBotController::OnTargetPerceptionUpdated(AActor* Actor, FAIStimulus Stimulus)
{
	if (UBlackboardComponent* BBComponent = GetBlackboardComponent())
	{
		TSubclassOf<UAISense> AISence = UAIPerceptionSystem::GetSenseClassForStimulus(GetWorld(), Stimulus).Get();

		if (AISence == UAISense_Sight::StaticClass() || AISence == UAISense_Damage::StaticClass())
		{
			BBComponent->SetValueAsObject(TEXT("AttackTarget"), Actor);
			SetBotState(EBotState::BS_Attack);
		}
		else if (AISence == UAISense_Hearing::StaticClass())
		{
			BBComponent->SetValueAsVector(TEXT("SearchLocation"), Stimulus.StimulusLocation);
			SetBotState(EBotState::BS_Alert);
		}
	}
}
```

**[⬆ Back to Top](#top)**

<a name="순찰상태"></a>
### 순찰상태
![순찰상태설계](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/AI/순찰상태설계.png?raw=true)

 > 순찰상태의 설계구조

순찰상태에서는 SplineComponent를 이용해서 월드에 위치를 지정한 뒤 Spline의 경로에 따라 이동하도록 만들었습니다.

 - ASrPatrolSpline.cpp
```cpp
FVector ASrPatrolSpline::GetCurrentWorldSplinePoint()
{
	return SplineComponent->GetLocationAtSplinePoint(PatrolIndex, ESplineCoordinateSpace::World);
}
```
코드는 SplineComponent의 다음 SplinePoint를 가져와서 MoveTo하는 방식입니다.
![순찰움직임](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/AI/순찰움직임.png?raw=true)

 > AI같은경우엔 빠른 생산성을 위해 BP를 사용했습니다.

<a name="경계상태"></a>
### 경계상태
![경계상태설계](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/AI/경계상태설계.png?raw=true)

 > 경계상태의 설계구조

`경계상태`는 **AIPerception**의 `Hearing`이 감지되었을때 **감지된 위치로 단순하게 이동**합니다.

그리고 `경계상태`가 종료되면 `순찰상태`로 상태수준을 한 단계낮춥니다.

<a name="공격상태"></a>
### 공격상태
![공격상태설계](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/AI/공격상태설계.png?raw=true)

 > 공격상태의 설계구조

`공격상태`의 경우 **AIPerception**의 `Sight`나 `Damage`를 통해 트리거됩니다.

그리고 **공격을 수행하는 로직**과 **적을 주시하는(간보는) 로직**으로 나누어집니다.
간보기상태의 경우엔 `EQS`를 통해 적의 위치 중 적절한 위치로의 이동을 수행합니다.

<a name="탐색상태"></a>
### 탐색상태
![탐색상태설계](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/AI/탐색상태설계.png?raw=true)

 > 탐색상태의 설계구조

`탐색상태`의 경우 자체적으로 트리거되지 않으며 무조건 `공격상태`를 거쳐서 진행됩니다.
`공격상태`에서 `탐색상태`로 변경되게 되면 주변의 `SmartObject`를 탐색합니다.
만약 **SmartObject**가 주변에 존재한다면 탐색을 진행하고 그렇지 않으면 탐색을 종료하고 `순찰상태`로 상태수준을 낮춥니다.

**[⬆ Back to Top](#top)**

<a name="perception-age"></a>
### AIPerception Age의 만료시점을 통한 상태 전환
적이 시야에서 멀어지거나 **AIPerception**의 **Age**가 만료된 경우 탐색상태로 변경됩니다.
**Age의 만료된 시점**의 경우엔 **AIPerception**의 `virtual void HandleExpiredStimulus(FAIStimulus& StimulusStore) override;`함수를 재정의 함으로써 구현이 가능합니다.
제가 구현한 코드에서는 **델리게이트를 만들고 AIController의 함수를 바인딩**하여 처리했습니다.

 - USrAIPerceptionComponent.cpp
```cpp
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FPerceptionStimulusExired, const FAIStimulus&, Stimulus);

void USrAIPerceptionComponent::HandleExpiredStimulus(FAIStimulus& StimulusStore)
{
	OnStimuliExpired.Broadcast(StimulusStore);
}
```

그리고 각각의 상태에 따라 `Age`가 만료될 때, AI의 행동상태를 재설정해주었습니다.

 - ASrEnemyBotController.cpp
```cpp
void ASrEnemyBotController::OnStimuliExpired(const FAIStimulus& Stimulus)
{
	TSubclassOf<UAISense> AISence = UAIPerceptionSystem::GetSenseClassForStimulus(GetWorld(), Stimulus).Get();

	if (AISence == UAISense_Sight::StaticClass() || AISence == UAISense_Damage::StaticClass())
	{
		SetBotState(EBotState::BS_Search);
	}
	else if (AISence == UAISense_Hearing::StaticClass())
	{
		SetBotState(EBotState::BS_Roam);
	}
}
```

**[⬆ Back to Top](#top)**

<a name="smart-object"></a>
### SmartObject
언리얼 엔진에서는 `AI`와 플레이어의 상호작용에 `SmartObject`라는 최신 기술을 도입했습니다.
이 기술을 이용하면 `GameplayTag`를 이용해 AI의 상호작용을 의존성을 최소화하여 표현할 수 있습니다.

![SmartObject탐색정의](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/AI/SmartObject탐색정의.png?raw=true)

 > 탐색 SmartObject 구조

`SmartObjectDefinition`을 상속받는 탐색용 애셋을 생성해주고 `Gameplay Behavior`에 `Activaty Tags`로 `SmartObject`용 태그를 추가해줬습니다.

![SmartObject탐색로직](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/AI/SmartObject탐색로직.png?raw=true)

 > 탐색 SmartObject 로직

탐색로직의 경우 단순하게 `Search애니메이션`을 재생해주고 재생이 완료되면 행동을 종료합니다.

그리고 AI는 `FindSmartObjects`를 통해서 `SmartObject`를 찾고 이때 찾은 `SmartObject`들을 통해 `SmartObjectSubsystem`의 `Claim`함수에서 `SmartObject`에 상호작용할 수 있는지 여부를 결정하고 유효하다면 탐색로직을 수행하도록 했습니다.
(`SmartObjectSubsystem`는 World와 LifeCycle을 공유하는 `UWorldSubsystem`입니다.)

![SmartObject셋팅](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/AI/SmartObject셋팅.png?raw=true)

 > 탐색 SmartObject Claim

![SmartObject사용](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/AI/SmartObject사용.png?raw=true)

 > 탐색 SmartObject 수행(Use)

**[⬆ Back to Top](#top)**

<a name="oss-steam"></a>
## OSS Steam

<a name="멀티세션-구조설계"></a>
### 멀티 세션의 구조 설계
제 게임은 **`게임의 입장(로그인) → 방생성 → 로비입장 → ||| 진정한 게임의 시작`**의 순서를 가지고 있고 게임의 시작 전까지의 모든 과정은 실제 게임의 로직과는 연관성이 없습니다.

로비입장의 경우엔 캐릭터의 존재에 따라 연관성이 생길 수있지만 제 게임에서는 특별히 캐릭터의 외형꾸밈 같은 데이터를 고려하지 않습니다.

이를 근거로 `멀티 세션 시스템`은 단순히 현재 프로젝트에만 사용하는데 그치지 않고 **범용적으로 사용**할 수 있는 독립적인 기능을 가진 시스템이라고 판단하여 `플러그인`으로 구현하였습니다.

<a name="게임모드-분리"></a>
### 로비와 게임의 게임모드 분리
실제 게임에서 사용하는 게임모드와는 별개로 **Lobby**에 들어왔을때는 게임의 로직과는 다른 구조로 로직이 구성되고 **로비에 들어오는 모든 플레이어를 관리할 필요**가 있기 때문에 `SessionGameMode`와 `SessionPlayerController`를 통해서 관리하도록 설계했습니다.

![세션의존성관계도](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/AI/세션의존성관계도.png?raw=true)

 > OnlineGameInstanceSubsystem에 의존하고 있는 객체들

**[⬆ Back to Top](#top)**

<a name="세션코드"></a>
### 세션 소스코드

 - 세션 생성
```cpp
//처음 메인메뉴에서 로비를 생성할때 호출되는 함수입니다.
void UOnlineGameInstanceSubsystem::LaunchLobby(int32 InNumOfPlayers, bool bIsLan, FText InServerName)
{
	FNamedOnlineSession* ExistingSession = SessionInterface->GetNamedSession(NAME_GameSession);
	if (ExistingSession != nullptr)
	{
		DestroySession();
	}

	NumOfPlayers = InNumOfPlayers;
	ServerName = InServerName;

    //세션이 생성될때 호출할 델리게이트를 바인딩해줍니다.
	CreateSessionCompleteDelegateHandle = SessionInterface->AddOnCreateSessionCompleteDelegate_Handle(CreateSessionCompleteDelegate);

    //세션 생성 정보를 셋팅합니다.
	CreateSessiomSettings = MakeShareable(new FOnlineSessionSettings());
	CreateSessiomSettings->NumPublicConnections = NumOfPlayers;
	CreateSessiomSettings->bShouldAdvertise = true;
	CreateSessiomSettings->bAllowJoinInProgress = true;
	CreateSessiomSettings->bIsLANMatch = bIsLan;
	CreateSessiomSettings->bUsesPresence = true;
	CreateSessiomSettings->bAllowJoinViaPresence = true;
	CreateSessiomSettings->BuildUniqueId = bIsLan ? 0 : 1;
	CreateSessiomSettings->bUseLobbiesIfAvailable = true;
	CreateSessiomSettings->bIsDedicated = false;

	CreateSessiomSettings->Set(SETTING_MAPNAME, ServerName.ToString(), EOnlineDataAdvertisementType::ViaOnlineServiceAndPing);

    //실제로 세션을 생성해주는 로직입니다.
	const ULocalPlayer* LocalPlayer = GetWorld()->GetFirstLocalPlayerFromController();
	if (!SessionInterface->CreateSession(*LocalPlayer->GetPreferredUniqueNetId(), NAME_GameSession, *CreateSessiomSettings))
	{
		SessionInterface->ClearOnCreateSessionCompleteDelegate_Handle(CreateSessionCompleteDelegateHandle);
	}
}
```

 - 세션 탐색
```cpp
//세션을 찾는 함수로 클라이언트가 호출합니다.
bool UOnlineGameInstanceSubsystem::FindSessions(int32 MaxSessionResults, bool bIsLan)
{
	if (!SessionInterface.IsValid())
	{
		return false;
	}

    //세션을 찾을때 호출할 델리게이트를 바인딩해줍니다.
	FindSessionsCompleteDelegateHandle = SessionInterface->AddOnFindSessionsCompleteDelegate_Handle(FindSessionsCompleteDelegate);

    //세션 탐색 정보를 셋팅합니다.
	SessionSearch = MakeShareable(new FOnlineSessionSearch);
	SessionSearch->MaxSearchResults = MaxSessionResults;
	SessionSearch->bIsLanQuery = bIsLan;
	SessionSearch->QuerySettings.Set(SEARCH_PRESENCE, true, EOnlineComparisonOp::Equals);

    //이전에 검색한 세션 결과를 리셋해줍니다.
	SearchSessionResults.Empty();

    //세션을 찾습니다.
	const ULocalPlayer* LocalPlayer = GetWorld()->GetFirstLocalPlayerFromController();
	if (!SessionInterface->FindSessions(*LocalPlayer->GetPreferredUniqueNetId(), SessionSearch.ToSharedRef()))
	{
		SessionInterface->ClearOnFindSessionsCompleteDelegate_Handle(FindSessionsCompleteDelegateHandle);

		return false;
	}

	return true;
}
```

 - 세션 참가
```cpp
//세션에 참가하는 함수로 클라이언트가 호출합니다.
void UOnlineGameInstanceSubsystem::JoinSession(const FOnlineSessionSearchResult& SearchResult)
{
	if (!SessionInterface.IsValid())
	{
		return;
	}

    //세션에 참가할 때 호출할 델리게이트를 바인딩해줍니다.
	JoinSessionCompleteDelegateHandle = SessionInterface->AddOnJoinSessionCompleteDelegate_Handle(JoinSessionCompleteDelegate);

    //세션에 참가합니다.
	const ULocalPlayer* LocalPlayer = GetWorld()->GetFirstLocalPlayerFromController();
	if (!SessionInterface->JoinSession(*LocalPlayer->GetPreferredUniqueNetId(), NAME_GameSession, SearchResult))
	{
		SessionInterface->ClearOnJoinSessionCompleteDelegate_Handle(JoinSessionCompleteDelegateHandle);
	}
}
```

 - 게임 시작
```cpp
//게임 시작 함수로 실제 게임 맵으로 이동하는 로직입니다.(컨트롤러 Possess문제로 연결을 끊고 재연결하는 Non-SeamlessTravel을 사용합니다.)
void UOnlineGameInstanceSubsystem::StartGame(const FString& InGameMapPath, bool bIsAbsolute)
{
	UWorld* World = GetWorld();
	if (World)
	{
		FString PathToGame = FString::Printf(TEXT("%s?listen"), *InGameMapPath);
		if (World->ServerTravel(PathToGame, bIsAbsolute))
		{
			GEngine->AddOnScreenDebugMessage(
				-1,
				5.f,
				FColor::Red,
				FString(TEXT("Success to start game!"))
			);
		}
		else
		{
			GEngine->AddOnScreenDebugMessage(
				-1,
				5.f,
				FColor::Red,
				FString(TEXT("Fail to start game!"))
			);
		}
	}
}
```

 - 세션 생성 성공
```cpp
//세션을 성공적으로 생성하면 호출되는 함수로 호스트에서 호출됩니다.
void UOnlineGameInstanceSubsystem::OnCreateSessionComplete(FName SessionName, bool bSucceeded)
{
	if (SessionInterface)
	{
		SessionInterface->ClearOnCreateSessionCompleteDelegate_Handle(CreateSessionCompleteDelegateHandle);
	}

    //세션 생성에 성공하면 로비로 이동합니다.
	if (bSucceeded)
	{
		UWorld* World = GetWorld();
		if (World)
		{
			FString PathToLobby = FString::Printf(TEXT("%s?listen"), TEXT("/OnlineSessionSystem/Game/Level/Lobby"));
			World->ServerTravel(PathToLobby, true);
		}
	}
	else
	{
		GEngine->AddOnScreenDebugMessage(
			-1,
			5.f,
			FColor::Red,
			FString(TEXT("Failed to create session!"))
		);
	}
}
```

 - 세션 참가 성공
```cpp
//세션에 성공적으로 참가하면 호출되는 함수로 클라이언트에서 호출됩니다.
void UOnlineGameInstanceSubsystem::OnJoinSessionComplete(FName SessionName, EOnJoinSessionCompleteResult::Type Result)
{
	if (SessionInterface)
	{
		SessionInterface->ClearOnJoinSessionCompleteDelegate_Handle(JoinSessionCompleteDelegateHandle);
	}

	if (Result != EOnJoinSessionCompleteResult::Success)
	{
		GEngine->AddOnScreenDebugMessage(
			-1,
			5.f,
			FColor::Red,
			FString(TEXT("JoinSession Fail!"))
		);
		return;
	}
	if (Result == EOnJoinSessionCompleteResult::Success)
	{
		if (SessionInterface.IsValid())
		{
            //호스트의 Address(포트)를 찾습니다.
			FString Address;
			SessionInterface->GetResolvedConnectString(NAME_GameSession, Address);

            //찾은 주소로 이동하며 이때 Client RPC를 통해 이동하게 됩니다.
			APlayerController* PlayerController = GetGameInstance()->GetFirstLocalPlayerController();
			if (PlayerController)
			{
				PlayerController->ClientTravel(Address, ETravelType::TRAVEL_Absolute);
			}
		}

		GEngine->AddOnScreenDebugMessage(
			-1,
			5.f,
			FColor::Red,
			FString(TEXT("JoinSession Success!"))
		);
	}
}

```

위 내용들을 모두 통합하여 OSS Steam기반으로 구현한 영상입니다.

시연영상(리슨서버 호스트) : https://youtu.be/9apJG1NuR0I

**[⬆ Back to Top](#top)**

<a name="spectator"></a>
## Spectator

Styx : Shard of Darkness 게임에서 캐릭터가 사망 이후 특정 플레이어를 관전하고 리스폰하는 시스템에 영감을 얻어서 제 프로젝트에도 구현해보면 어떨까라는 생각으로 관전 시스템을 구현해봤습니다.

언리얼에서는 SpectatorPawn의 관전자 전용 Pawn클래스를 제공해줍니다. 이 관전용 폰에 관전과 관련된 기능을 추가하여 구현했습니다.

<a name="관전-입력-맵핑"></a>
### 관전용 입력 맵핑

관전 시점에서는 실제 게임의 모든 입력이 막혀야 하고 관전시점에서의 새로운 입력이 존재합니다.
이를 근거로하여 관전용 새로운 입력맵핑을 만들었습니다.

![관전입력맵핑](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/Spectator/관전입력맵핑.PNG?raw=true)

 > 관전용 InputMapping

![관전입력값](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/Spectator/관전입력값.PNG?raw=true)

 > 관전용 입력값들

여기서 설정한 입력값을 ASrSpectatorPawn의 SetupPlayerInputComponent에서 초기화하였습니다.

```cpp
void ASrSpectatorPawn::SetupPlayerInputComponent(UInputComponent* InInputComponent)
{
	Super::SetupPlayerInputComponent(InInputComponent);

	UWorld* World = GetWorld();
	if (!World)
	{
		return;
	}

	InitializeInputMapping();

	EnhancedInputComponent = Cast<UEnhancedInputComponent>(InInputComponent);

	//시간 딜레이를 준 이유는 리스폰 과정에서 입력딜레이 없이 바로 리스폰하게되면 게임 캐릭터에 Possess가 되지 않는 문제가 발생합니다.
	FTimerHandle TimerHandle;
	World->GetTimerManager().SetTimer(TimerHandle, [&]()
		{
			EnhancedInputComponent->BindAction(NextViewInputAction, ETriggerEvent::Triggered, this, &ThisClass::ViewNextPlayer);
			EnhancedInputComponent->BindAction(PrevViewInputAction, ETriggerEvent::Triggered, this, &ThisClass::ViewPrevPlayer);
			EnhancedInputComponent->BindAction(RespawnInputAction, ETriggerEvent::Triggered, this, &ThisClass::RespawnPlayer);
		}, 1.f, false);
}
```

```cpp
void ASrSpectatorPawn::InitializeInputMapping() const
{
	const APlayerController* PlayerController = GetController<APlayerController>();
	check(PlayerController);

	const ULocalPlayer* LocalPlayer = PlayerController->GetLocalPlayer();
	check(LocalPlayer);

	UEnhancedInputLocalPlayerSubsystem* Subsystem = LocalPlayer->GetSubsystem<UEnhancedInputLocalPlayerSubsystem>();
	check(Subsystem);

	Subsystem->ClearAllMappings();

	//관전 시스템의 입력맵핑으로 변경해줍니다.
	if (SpectatorInputConfig)
	{
		Subsystem->AddPlayerMappableConfig(SpectatorInputConfig);
	}
}
```

**[⬆ Back to Top](#top)**

<a name="관전자-전환"></a>
### 관전자 전환

관전과 관련된 기능은 기본적으로 `APlayerController`를 통해서 동작하기때문에 기존의 `SrPlayerController`에서 관전상태로 변환해주는 로직을 추가해줬습니다.

 - ASrPlayerController.cpp
```cpp
//관전 상태에서 플레이 상태로 변경해주는 함수입니다.
void ASrPlayerController::SetPlayerPlaying()
{
	if (!HasAuthority())
	{
		return;
	}

	//관전 여부를 갱신 해줍니다.
	PlayerState->SetIsSpectator(false);

	//SpectatorPawn을 삭제합니다. 이 함수는 서버에서 실행됩니다.
	ChangeState(NAME_Playing);

	bPlayerIsWaiting = false;

	//ChageState함수를 클라이언트에도 동일하게 실행해줍니다.
	ClientGotoState(NAME_Playing);

	//관전 상태에 따라 HUD를 변경해줍니다.
	HUDStateChanged_Client(EHUDState::Playing);
}
```

 - ASrPlayerController.cpp
```cpp
//플레이 상태에서 관전 상태로 변경해주는 함수입니다.
void ASrPlayerController::SetPlayerSpectate()
{
	// Only proceed if we're on the server
	if (!HasAuthority())
	{
		return;
	}

	//관전 여부를 갱신 해줍니다.
	PlayerState->SetIsSpectator(true);
	
	//SpectatorPawn을 생성하고 Possess합니다. 이 함수는 서버에서 실행됩니다.
	ChangeState(NAME_Spectating);

	bPlayerIsWaiting = true;

	//ChageState함수를 클라이언트에도 동일하게 실행해줍니다.
	ClientGotoState(NAME_Spectating);

	//관전 상태에 따라 HUD를 변경해줍니다.
	HUDStateChanged_Client(EHUDState::Spectating);
}
```

여기서 `SpectatorPawn`은 로컬에서만 생성되어야 하기때문에 `ChangeState`와 `ClientGotoState`는 서버일땐 `ChangeState`를 통해서, 클라이언트일땐 `ClientGotoState`를 통해서 `SpectatorPawn`을 생성해줍니다.

 - APlayerController.cpp
```cpp
ASpectatorPawn* APlayerController::SpawnSpectatorPawn()
{
	ASpectatorPawn* SpawnedSpectator = nullptr;

	// Only spawned for the local player
	if ((GetSpectatorPawn() == nullptr) && IsLocalController())
	{
		//Spawn Logic...
	}
}
```

그리고 관전자의 정보를 나타내는 UI를 추가하여 PageUp과 PageDown 키입력에 따라 변화하도록 구현했습니다.

 - ASrSpectatorPawn.cpp
```cpp
void ASrSpectatorPawn::ViewNextPlayer()
{
	if (APlayerController* PC = GetController<APlayerController>())
	{
		PC->ServerViewNextPlayer();

		//관전자의 이름을 업데이트 해주는 함수입니다. 스팀환경에서는 스팀이름이 적용됩니다.
		if (ASrHUD* SrHUD = Cast<ASrHUD>(PC->GetHUD()))
		{
			SrHUD->UpdateSpectatorName();
		}
	}
}

void ASrSpectatorPawn::ViewPrevPlayer()
{
	if (APlayerController* PC = GetController<APlayerController>())
	{
		PC->ServerViewPrevPlayer();

		//관전자의 이름을 업데이트 해주는 함수입니다. 스팀환경에서는 스팀이름이 적용됩니다.
		if (ASrHUD* SrHUD = Cast<ASrHUD>(PC->GetHUD()))
		{
			SrHUD->UpdateSpectatorName();
		}
	}
}
```

 - 참고 코드 : APlayerController.cpp
```cpp
//서버에만 존재하는 GameState에 접근하기 위해 Run On Server로 실행되는 함수입니다.
void APlayerController::ServerViewNextPlayer_Implementation()
{
	if (IsInState(NAME_Spectating))
	{
		ViewAPlayer(+1);
	}
}

void APlayerController::ViewAPlayer(int32 dir)
{
	//GameState의 PlayerArray에서 순방향 검색을 통해 PlayerState를 반환합니다.
	APlayerState* const NextPlayerState = GetNextViewablePlayer(dir);

	if ( NextPlayerState != nullptr )
	{
		SetViewTarget(NextPlayerState);
	}
}
```

그리고 현재 ViewTarget의 이름을 가져와서 UI에 띄워주었습니다.
![관전자이름가져오기](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/Spectator/관전자이름가져오기.PNG?raw=true)

**[⬆ Back to Top](#top)**

<a name="리스폰"></a>
### 리스폰

플레이어가 다시 스폰되는 과정은 입력은 컨트롤러에서 발생하지만 실제 스폰되는 과정은 서버를 통해서 이루어져야합니다.

그러기 때문에 모든 리스폰 로직은 게임모드에서 동작하도록 구현하였습니다.

 - ASrSpectatorPawn.cpp
```cpp
void ASrSpectatorPawn::RespawnPlayer()
{
	if (!HasAuthority())
	{
		return;
	}

	ASrPlayerController* PC = GetController<ASrPlayerController>();
	if (!PC)
	{
		return;
	}

	PC->RespawnPlayer();
}
```

 - ASrPlayerController.cpp
```cpp
void ASrPlayerController::RespawnPlayer()
{
	RespawnPlayer_Server();
}
```

```cpp
void ASrPlayerController::RespawnPlayer_Server_Implementation()
{
	if (ICharacterRespawnable* SrGameMode = Cast<ICharacterRespawnable>(GetWorld()->GetAuthGameMode()))
	{
		if (AActor* SpawnTargetActor = GetViewTarget())
		{
			SrGameMode->RespawnPlayer(this, SpawnTargetActor);
		}
	}
}
```

```cpp
void ASrGameMode::RespawnPlayer(APlayerController* InPlayerController, AActor* TargetActor)
{
	//플레이어를 Spectator상태에서 Playing상태로 변경합니다.
	SrController->SetPlayerPlay();

	FTransform SpawnTransform;

	//ViewTarget이 존재하면 Target의 앞으로 스폰합니다.
	if (Cast<ACharacter>(TargetActor))
	{
		FVector Location = TargetActor->GetActorLocation() + TargetActor->GetActorForwardVector() * 50.f;
		FRotator Rotation = TargetActor->GetActorForwardVector().Rotation();
		SpawnTransform = FTransform(Rotation, Location);
	}
	//ViewTarget이 존재하지 않으면 PlayerState에 스폰합니다.
	else
	{
		AActor* PlayerStart = UGameplayStatics::GetActorOfClass(GetWorld(), APlayerStart::StaticClass());

		if (PlayerStart)
		{
			FVector Location = PlayerStart->GetActorLocation() + PlayerStart->GetActorForwardVector() * 50.f;
			FRotator Rotation = PlayerStart->GetActorForwardVector().Rotation();
			SpawnTransform = FTransform(Rotation, Location);
		}
	}
	
	//지연 생성을통해 BeginPlay와 Possess의 호출순서를 바로잡았습니다.
	APawn* RespawnPawn = GetWorld()->SpawnActorDeferred<APawn>(DefaultPawnClass, SpawnTransform, InPlayerController, nullptr, ESpawnActorCollisionHandlingMethod::AdjustIfPossibleButAlwaysSpawn);
	
	InPlayerController->Possess(RespawnPawn);

	RespawnPawn->FinishSpawning(SpawnTransform);
}
```

게임모드에서는 먼저 기존의 관전셋팅을 게임셋팅으로 변환하고 `DefaultPawnClass`를 이용하여 플레이어를 적절한 위치에 스폰시켜줬습니다.

이때 스폰위치는 관전하고 있는 대상이 있을경우 관전자 앞으로, 관전하고 있는 대상이 없을경우 `PlayerStart`로 지정하였습니다.

이때 기존의 `SpawnActor`를 사용하면 `Possess`와 `BeginPlay`중 `BeginPlay`가 먼저 실행되어서 로직이 꼬이는 문제가 발생하였는데, 이를 `SpawnDeffered`를 통해 `Possess`와 `BeginPlay`의 호출 순서를 의도적으로 변경해주었습니다.

**[⬆ Back to Top](#top)**

<a name="linetracing-attack"></a>
## Extra : LineTracing Attack

간혹 공격 애니메이션이 너무 빠르거나 프레임저하로 인해 모션이 스킵되면 공격이 무시되는 문제가 발생했습니다.

문제의 원인은 무기의 충돌체가 빠르게 움직이면서 판정을 무시하게 되는 것이였고 이 문제를 이전 소켓위치와 다음 소켓위치를 통한 라인 트레이싱 방법을 이용하여 해결했습니다.

![라인트레이싱공격](https://github.com/Naezan/Sicarius-Portfolio/blob/main/img/Extra/라인트레이싱공격.png?raw=true)

 > 라인 트레이싱 공격 설계

결론적으로 애니메이션의 프레임 간의 모션이 멀어도 위치를 보간하여 충돌위치를 찾아낼 수있습니다.

**[⬆ Back to Top](#top)**

프로젝트 링크(PDF용) : https://github.com/Naezan/Sicarius_GoogleDrive

<a name="images"></a>
## Images

![Map](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/MapPhoto.png?raw=true)
 - Map1


![NicePhoto1](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/NicePhoto1.png?raw=true)
 - Map2


![Climing](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/Climing.png?raw=true)
 - Climing


![TrapItem](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/TrapItem.png?raw=true)
 - TrapItem


![Hiding](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/Hiding.png?raw=true)
 - Hiding


![InjectPoison](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/InjectPoison.png?raw=true)
 - InjectPoison


![Waiting](https://github.com/Naezan/Sicarius_GoogleDrive/blob/main/img/Waiting.png?raw=true)
 - Waiting

**[⬆ Back to Top](#top)**