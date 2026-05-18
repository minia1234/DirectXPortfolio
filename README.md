# DirectX Portfolio — 던그리드 모작

DirectX 11 자체 엔진 기반 2D 던전 액션 게임.

## 시연 영상

https://youtu.be/EXwi85F40Ew

## 프로젝트 정보

| 항목    | 내용                                         |
| ----- | ------------------------------------------ |
| 프로젝트명 | 던그리드 모작                                    |
| 기간    | 2021.10 ~ 2021.12                          |
| 환경    | Visual Studio 2019                         |
| 도구    | DirectX 11, HLSL, DirectInput, FMOD, ImGui |
| 형상관리  | Github                                     |

## 씬 흐름

```
TitleScene → MainScene → BossScene → EndingScene
                  ↑↓
            던전 방 전환 (Gate)
```

- 시작 화면 (`StartScene/BackGround`, `Bird` 등)
- 던전 메인 게임 (방 단위 진행)
- 보스전 (Belial)
- 엔딩 시네마틱 (`EndingCamera`, `EndingHorse`, `EndingPlayer` 등)

---

# Player

## CPlayer 베이스

플레이어 베이스 클래스. **3개의 콜리전을 분리**해 정밀한 지형/벽 충돌 처리.

```cpp
enum class ePlayerState
{
    Idle,
    Move,
    Jump
};

class CPlayer : public CGameObject
{
private:
    CSharedPtr<CSpriteComponent>     m_Sprite;
    CSharedPtr<CSpringArm2D>         m_Arm;
    CSharedPtr<CCamera>              m_Camera;

    // 충돌 콜리전 분리 — 정밀한 방향별 충돌 처리
    CSharedPtr<CColliderBox2D>       m_Collider2D;          // 본체
    CSharedPtr<CColliderBox2D>       m_Collider2DHorizon;   // 좌/우 벽 감지
    CSharedPtr<CColliderBox2D>       m_Collider2DVertical;  // 위/아래 지형 감지

    CSharedPtr<CRigidBodyComponent>  m_Body;
    CSharedPtr<CAnimation2D_FSM>     m_Animation2D;
    CSharedPtr<CWidgetComponent>     m_WidgetComponent;     // 머리 위 위젯
    class CPlayerWorldWidget*        m_PlayerWidget;

    CEngineFSM<CPlayer>              m_BodyFSM;

    CPlayerStatus                    m_Status;
    class CWeaponArm*                m_WeaponArm;            // 무기 회전 팔
    class CWeapon*                   m_Weapon;
    std::vector<class CItem*>        m_AccItem;              // 장착 액세서리

    int                              m_Coin;
    Object_Dir                       m_Dir;
    float                            m_Angle;
    ePlayerState                     m_State;
    ePlayerState                     m_PrevState;

    std::vector<CGameObject*>        m_vDashTexture;         // 대시 잔상

public:
    // 입력 처리 (15개)
    void LeftMove(float DeltaTime);
    void RightMove(float DeltaTime);
    void DownMove(float DeltaTime);
    void JumpMove(float DeltaTime);
    void Attack(float DeltaTime);
    void Dash(float DeltaTime);
    void SkillAttack(float DeltaTime);
    void WeaponChange(float DeltaTime);
    void InventoryOnOff(float DeltaTime);
    void MapOnOff(float DeltaTime);
    void ShopUIOnOff(float DeltaTime);
    void RestaurantUIOnOff(float DeltaTime);
    void StatusUIOnOff(float DeltaTime);
    void InputInteractionInputKey(float DeltaTime);
    void CollisionRenderOnOff(float DeltaTime);   // 디버그용

    // FSM 핸들러 (Idle / Move / Jump)
    void BodyIdleStart(); void BodyIdleStay();
    void BodyMoveStart(); void BodyMoveStay();
    void BodyJumpStart(); void BodyJumpStay();

    // 충돌 콜백 - 본체 / 수평 / 수직 분리
    void CollisionBegin(const HitResult& result, CCollider* Collider);
    void CollisionHorizonBegin(const HitResult& result, CCollider* Collider);
    void CollisionHorizonMiddle(const HitResult& result, CCollider* Collider);
    void CollisionHorizonEnd(const HitResult& result, CCollider* Collider);
    void CollisionVerticalBegin(const HitResult& result, CCollider* Collider);
    void CollisionVerticalMiddle(const HitResult& result, CCollider* Collider);
    void CollisionVerticalEnd(const HitResult& result, CCollider* Collider);

    // 충돌 방향 계산
    void ColDirHorizon(float Angle, CCollider* Col);
    void ColDirVertical(float Angle, CCollider* Col);
    void ColTilePassDirVertical(float Angle, CCollider* Col);
};
```

- `CEngineFSM<CPlayer>` — 멤버 함수 포인터 콜백 등록 FSM
- Idle / Move / Jump 3-페이즈 상태머신 (**Stay + Start 2-페이즈**, End는 미사용)
- 코인 (`m_Coin`) — 상점/식당 통화
- 액세서리 (`m_AccItem`, 4슬롯) — 다중 능력 부여

### 초기화 실측값 / 컴포넌트 계층 / 입력 매핑

```cpp
bool CPlayer::Init()
{
    // 컴포넌트 생성
    m_Body                = CreateSceneComponent<CRigidBodyComponent>("Body");
    m_Collider2D          = CreateSceneComponent<CColliderBox2D>("Collider2D");
    m_Collider2DHorizon   = CreateSceneComponent<CColliderBox2D>("Collider2DHorizon");
    m_Collider2DVertical  = CreateSceneComponent<CColliderBox2D>("Collider2DVertical");
    m_Sprite              = CreateSceneComponent<CSpriteComponent>("Sprite");
    m_Arm                 = CreateSceneComponent<CSpringArm2D>("Arm");
    m_Camera              = CreateSceneComponent<CCamera>("Camera");
    m_WidgetComponent     = CreateSceneComponent<CWidgetComponent>("PlayerWidgetComponent");

    SetRootComponent(m_Sprite);
    m_Sprite->AddChild(m_WidgetComponent);
    m_Sprite->AddChild(m_Collider2D);
    m_Sprite->AddChild(m_Collider2DHorizon);
    m_Sprite->AddChild(m_Collider2DVertical);
    m_Sprite->AddChild(m_Body);
    m_Sprite->AddChild(m_Arm);
    m_Arm->AddChild(m_Camera);

    m_Sprite->SetRelativeScale(15.f, 20.f, 1.f);
    m_Camera->SetCameraZoom(4.f);

    // 3-Layer Collider Extent
    m_Collider2D        ->SetExtent(6.f,  10.f);    // 본체 (피격 판정)
    m_Collider2DHorizon ->SetExtent(6.f,  0.3f);    // 수평 — 위/아래 얇은 막대
    m_Collider2DVertical->SetExtent(0.3f, 10.f);    // 수직 — 좌/우 얇은 막대

    // Collision Profile + 콜백 (3 콜리전 × Begin/Middle/End 분리)
    m_Collider2D       ->SetCollisionProfile("Player");
    m_Collider2DHorizon->SetCollisionProfile("TileCheckCollsion");
    m_Collider2DVertical->SetCollisionProfile("TileCheckCollsion");
    m_Collider2D->AddCollisionCallbackFunction<CPlayer>(Collision_State::Begin, this,
        &CPlayer::CollisionBegin);
    m_Collider2DHorizon->AddCollisionCallbackFunction(Begin/Middle/End → CollisionHorizonBegin/Middle/End);
    m_Collider2DVertical->AddCollisionCallbackFunction(Begin/Middle/End → CollisionVerticalBegin/Middle/End);

    // SpringArm 카메라 화면 중앙 보정
    m_Arm->SetOffset(-640.f, -360.f, 0.f);

    // Animation2D_FSM 세팅
    m_Sprite->CreateAnimation2D<CAnimation2D_FSM>();
    m_Animation2D = (CAnimation2D_FSM*)m_Sprite->GetAnimation2D();
    m_Animation2D->SetIdleAnimation2D("PlayerIdle");
    m_Animation2D->SetMoveAnimation2D("PlayerRun");
    m_Animation2D->SetJumpAnimation2D("PlayerJump");

    // 14개 입력 키 콜백 등록
    CInput::GetInst()->AddKeyCallback("Left",                KT_Push, this, &LeftMove);
    CInput::GetInst()->AddKeyCallback("Right",               KT_Push, this, &RightMove);
    CInput::GetInst()->AddKeyCallback("Jump",                KT_Push, this, &JumpMove);
    CInput::GetInst()->AddKeyCallback("Attack",              KT_Down, this, &Attack);
    CInput::GetInst()->AddKeyCallback("Dash",                KT_Down, this, &Dash);
    CInput::GetInst()->AddKeyCallback("MouseWhell",          KT_Down, this, &WeaponChange);
    CInput::GetInst()->AddKeyCallback("InventoryOnOff",      KT_Down, this, &InventoryOnOff);
    CInput::GetInst()->AddKeyCallback("MapOnOff",            KT_Down, this, &MapOnOff);
    CInput::GetInst()->AddKeyCallback("InteractionInputKey", KT_Up,   this, &InputInteractionInputKey);
    CInput::GetInst()->AddKeyCallback("ShopUI",              KT_Up,   this, &ShopUIOnOff);
    CInput::GetInst()->AddKeyCallback("StatusUI",            KT_Up,   this, &StatusUIOnOff);
    CInput::GetInst()->AddKeyCallback("RestaurantUI",        KT_Up,   this, &RestaurantUIOnOff);
    CInput::GetInst()->AddKeyCallback("ESCUI",               KT_Up,   this, &UIOff);
    CInput::GetInst()->AddKeyCallback("Skill",               KT_Up,   this, &SkillAttack);

    // WeaponArm 자식으로 추가
    m_WeaponArm = m_pScene->SpawnObject<CWeaponArm>("basicWeaponArm");
    m_WeaponArm->SetRelativePos(20.f, 0.f, 0.f);
    PushObjectChild(m_WeaponArm);

    // FSM 등록 — Stay, Start만 (End 인자 미사용)
    m_BodyFSM.CreateState("Idle", this, &BodyIdleStay, &BodyIdleStart);
    m_BodyFSM.CreateState("Move", this, &BodyMoveStay, &BodyMoveStart);
    m_BodyFSM.CreateState("Jump", this, &BodyJumpStay, &BodyJumpStart);
    m_BodyFSM.ChangeState("Idle");

    SetGravity(true);
    m_AccItem.resize(4);                 // 액세서리 슬롯 4개
    SetStartTimer(0.2f);                 // 진입 0.2초 후부터 동작
    return true;
}
```

- **Horizon Collider (6 × 0.3)** — 가로로 넓고 세로로 얇은 막대 → 위·아래 충돌 감지
- **Vertical Collider (0.3 × 10)** — 세로로 길고 가로로 얇은 막대 → 좌·우 벽 감지
- **CollisionProfile** — `Player` (본체) / `TileCheckCollsion` (Horizon, Vertical)
- **3 Collider × 3 State (Begin/Middle/End) = 7개 콜백** 등록
- `SpringArm`의 Offset (-640, -360) — 화면 중앙 보정 (1280×720 기준)
- `SetCameraZoom(4.f)` — 4배 확대

### 방향 / 마우스 추적

```cpp
void CPlayer::Update(float DeltaTime)
{
    Vector2 MousePos = CInput::GetInst()->GetMouse2DWorldPos();

    // 마우스 위치로 좌/우 방향 결정
    if (GetWorldPos().x > MousePos.x)
        m_Dir = Object_Dir::Left;
    else
        m_Dir = Object_Dir::Right;
    // ...
}
```

마우스 X 좌표로 좌/우 회전 결정 — 무기 회전과 캐릭터 방향 동기화.

## 충돌 시스템 (3-Layer Collider)

플레이어 충돌을 **본체 / 수평 / 수직** 3개로 분리해 미세한 처리.

```
┌──────────────┐
│  Body Col    │  ← 데미지 판정용
├──────────────┤
│ Horizon Col  │  ← 좌/우 벽과의 충돌 (벽에 붙는 처리)
├──────────────┤
│ Vertical Col │  ← 위/아래 지형 (점프/낙하)
└──────────────┘
```

- `CollisionHorizonBegin/Middle/End` — 벽 진입·유지·이탈 별도 콜백
- `CollisionVerticalBegin/Middle/End` — 지형 진입·유지·이탈
- `ColDirHorizon` / `ColDirVertical` — 충돌 방향(Angle) 계산
- `ColTilePassDirVertical` — 통과 가능 타일(아래로 점프) 처리

## 보조 액터

| 액터 | 역할 |
|---|---|
| `CWeaponArm` | 마우스 방향으로 무기를 회전시키는 팔 |
| `CPlayerAttack` | 공격 콜리전 액터 |
| `CPlayerDash` | 대시 동작 / 무적 처리 |
| `CPlayerDustEffect` | 이동 시 먼지 이펙트 |
| `CPlayerInteractionCollision` | 상호작용(NPC, 아이템) 감지 콜리전 |

---

# Animation2D (FSM)

`CAnimation2D`를 상속한 **FSM 기반 애니메이션 컨트롤러**.
5가지 시퀀스를 미리 등록해두고 상태 전환 함수로 전환.

```cpp
class CAnimation2D_FSM : public CAnimation2D
{
protected:
    bool m_Die;

    Sequence2DInfo* m_SequenceIdle;
    Sequence2DInfo* m_SequenceMove;
    Sequence2DInfo* m_SequenceJump;
    Sequence2DInfo* m_SequenceAttack;
    Sequence2DInfo* m_SequenceDie;

public:
    void SetIdleAnimation2D(const std::string& Name, bool Loop = true);
    void SetMoveAnimation2D(const std::string& Name, bool Loop = true);
    void SetJumpAnimation2D(const std::string& Name, bool Loop = true);
    void SetAttackAnimation2D(const std::string& Name, bool Loop = true);
    void SetDieAnimation2D(const std::string& Name, bool Loop = true);

    void ChangeIdleAnimation2D();
    void ChangeMoveAnimation2D();
    void ChangeJumpAnimation2D();
    void ChangeAttackAnimation2D();
    void ChangeDieAnimation2D();

    void IsDie() { m_Die = true; }
};
```

- 캐릭터마다 한 번만 시퀀스 등록 → 게임 중 상태 변화 시 `ChangeXxx`로 즉시 교체
- `SetXxxAnimation2D(Name, Loop)` — `AddAnimationSequence2D` 호출 후 시퀀스 포인터 저장
- `ChangeXxxAnimation2D()` — 저장된 시퀀스로 `ChangeAnimation` 호출

```cpp
void CAnimation2D_FSM::SetIdleAnimation2D(const std::string& Name, bool Loop)
{
    m_SequenceIdle = AddAnimationSequence2D(Name, Loop);
}

void CAnimation2D_FSM::ChangeIdleAnimation2D()
{
    ChangeAnimation(m_SequenceIdle);
}
```

특수 애니메이션 클래스: `CBirdAnimation2D` (시작 화면), `CLifeWaveAnimation2D` (체력 파동 효과).

---

# Weapon (무기 시스템)

근거리·원거리 무기 7종. 마우스 방향에 따른 회전 처리는 `CWeaponArm`이 담당.

| 무기 | 종류 | 발사체 / 이펙트 |
|---|---|---|
| `CShortSword` | 근거리 검 | `CShortSwordEffectObject` |
| `CRevolver` | 원거리 권총 | `CRevolverBullet` + `CRevolverEffectObject` |
| `CMetalBoomerang` | 원거리 부메랑 | `CMetalBoomerangBullet` + `CMetalBoomerangEffectObject` |
| `CCosmosSword` | 광역 검 | `CCosmosSwordBullet` + `CCosmosSwordEffect` |
| `CDaisyRing` | 회전 무기 | — |
| `CMiniEarth` | 궤도 무기 | — |
| `CTaana` | 방어형 무기 | `CTaanaShieldEffect` |

- 인벤토리에서 좌/우 무기 슬롯 두 개 보유 → `WeaponChange`로 즉시 교체
- 무기마다 다른 발사 패턴 / 데미지 / 이펙트
- `CWeaponArm` — 마우스 위치 기반 각도(`m_Angle`) 계산 후 무기를 회전

---

# Camera

- `CSpringArm2D` — 플레이어와 카메라 사이 거리 유지 보간
- `CEndingCamera` — 엔딩 시네마틱용 별도 카메라 (말 추격, 산 배경 패럴랙스)

---

# 적 / 보스

## CMonster / CEnemy 베이스

`CGameObject` 상속, 데미지 처리·사망 흐름 공통화.

## 일반 적 (8종)

| 적 | 특징 | 투사체 |
|---|---|---|
| `CBanshee` | 비행형 마법 적 | `CBansheeBullet` + `CBansheeBulletFX` |
| `CGhost` | 유령 (관통/투과) | — |
| `CFirefly` | 자폭 곤충 | `CFlameEffect` |
| `CGiant_Red` | 거대 적 | `CGiant_RedBullet` + `CGiant_RedBulletMain` + `CGiant_RedBulletFX` |
| `CSmallSkel` | 근접 해골 | — |
| `CSmallSkelBow` / `CSmallSkel_Bow` | 활 해골 | `CSkelSmallDagger` |
| `CTaana` | 방패 적 | `CTaanaShieldEffect` |
| `CTeemo` | 함정 설치 적 | — |

각 적은 자체 패턴 + Animation2D + 스폰 이펙트(`CSpawnEffect`).

## CBelial (보스)

3가지 공격 패턴 + 양손 + 배경 파티클 + 등장/사망 연출.

```cpp
class CBelial : public CGameObject
{
private:
    std::vector<class CBelialWeapon*>  m_BelialWeapon;
    class CBelialHand*                 m_LeftHand;
    class CBelialHand*                 m_RightHand;
    class CBelialBackParticle*         m_BackParticleObject;

    CSharedPtr<CColliderBox2D>         m_SpawnColliderBox2D;
    CSharedPtr<CSpriteComponent>       m_Sprite;
    CSharedPtr<CSpriteComponent>       m_BackSprite;
    CSharedPtr<CColliderBox2D>         m_Collider2D;
    CEngineFSM<CEnemy>                 m_EnemyFSM;
    CBasicStatus*                      m_Status;

    Belial_Pattern                     m_Pattern;
    float                              m_PatternTimer;

    // 검 스폰 패턴
    float                              m_SwordSpawnTimer;
    float                              m_SwordSpawnTimerMax;
    bool                               m_SwordSpawn;

    // 탄막 패턴
    float                              m_BulletAngle;
    float                              m_BulletFireCount;
    float                              m_BulletFireCountMax;

    // 레이저 패턴
    int                                m_LaserCount;
    int                                m_LaserCountMax;

    // 등장/사망 연출
    bool                               m_Spawn;
    float                              m_Alpha;
    float                              m_HandAlpha;
    bool                               m_EffectEndStart;
    bool                               m_EffectEnd;
    float                              m_EffectEndTimer;

    Vector3                            m_BasicWorldPos;   // 기본 위치 회귀

public:
    void AttackSword(float DeltaTime);    // 검 소환 패턴
    void AttackBullet(float DeltaTime);   // 탄막 패턴
    void AttackLaser(float DeltaTime);    // 레이저 패턴

    void EffectEndUpdate(float DeltaTime);
    void AlphaUpdate(float DeltaTime);
    void PatternUpdate(float DeltaTime);
};
```

### 보스 구성 액터

| 액터 | 역할 |
|---|---|
| `CBelialHand` | 좌/우 손 (Alpha 페이드 처리) |
| `CBelialBackParticle` | 보스 뒤 배경 파티클 |
| `CBelialWeapon` | 소환 검 |
| `CBelialWeaponCharge` / `CBelialWeaponHit` | 검 차지·히트 단계 |
| `CBelialBullet` + `CBelialBulletEffect` | 탄막 |
| `CBelialLaserHead` + `CBelialLaserBody` | 레이저 (머리 + 몸체 분리) |
| `CBelialDeadHead` / `CBelialDeadMouth` | 사망 시 분리되는 머리·입 |
| `CBossDieEffect` / `Start` / `Last` / `Particle` | 사망 연출 단계별 이펙트 |
| `CBossTorchLight` | 보스방 횃불 |
| `CBossTresure` | 보스 처치 보상 보물 |

### Belial 초기화 / Update 실측

```cpp
CBelial::CBelial() :
    m_AttackTimerMax(3.f),           // 패턴 간 간격
    m_SwordSpawnTimerMax(0.2f),      // 검 스폰 간격
    m_BulletFireCountMax(0.2f),      // 탄막 발사 간격
    m_LaserCountMax(2)               // 레이저 2회
{
    SetStatus("Belial");
}

void CBelial::Start()
{
    // 양손 스폰 — 좌측 -100, 우측 +100 + 좌우 반전
    m_LeftHand  = m_pScene->SpawnObject<CBelialHand>("BelialLeftHand");
    m_LeftHand->SetRelativePos(GetWorldPos() + Vector3(-100, 0, 0));

    m_RightHand = m_pScene->SpawnObject<CBelialHand>("BelialRightHand");
    m_RightHand->SetHorizontalReverse2DEnable(true);    // 우측 손은 좌우 반전
    m_RightHand->SetRelativePos(GetWorldPos() + Vector3(100, 0, 0));

    m_BasicWorldPos = GetWorldPos();   // 기본 위치 회귀용
}

bool CBelial::Init()
{
    // 두 개의 Sprite — 본체 + 뒤 배경 원
    m_Sprite     = CreateSceneComponent<CSpriteComponent>("Sprite");
    m_BackSprite = CreateSceneComponent<CSpriteComponent>("BackSprite");
    m_BackSprite->SetRender2DType(Render_Type_2D::RT2D_Back2);     // 뒷 레이어

    m_Animation2D = (CAnimation2D*)m_Sprite->GetAnimation2D();
    m_Animation2D->AddAnimationSequence2D("BelialHead_Idle");
    m_Animation2D->AddAnimationSequence2D("BelialHead_Attack", false);
    m_Animation2D->SetFrameEndFunction<CBelial>(this, &CBelial::AnimationFrameEnd);

    m_BackAnimation2D->AddAnimationSequence2D("Belial_Circle");

    // 스폰 감지용 박스 — 매우 긴 막대 (2 × 1000), Y축 전체 영역 감지
    m_SpawnColliderBox2D->SetExtent(2.f, 1000);
    m_SpawnColliderBox2D->SetCollisionProfile("BossSpawn");
    m_SpawnColliderBox2D->AddCollisionCallbackFunction(Begin → CollisionBossSpawnBegin);

    m_Collider2D->SetExtent(30.f, 30.f);
    m_Collider2D->Enable(false);                        // 등장 전에는 비활성
}
```

```cpp
void CBelial::Update(float DeltaTime)
{
    if (!m_Spawn) return;            // 스폰 트리거 전엔 동작 안 함
    AlphaUpdate(DeltaTime);          // 등장 시 페이드인

    if (m_PatternStop)               // 사망 / 일시정지
    {
        m_Animation2D->StopPlay();
        m_LeftHand->StopAnimation();
        m_RightHand->StopAnimation();
        if (m_EffectEnd) EffectEndUpdate(DeltaTime);   // 사망 연출
    }
    else if (m_SpawnEnd)
    {
        PatternUpdate(DeltaTime);    // 정상 패턴 실행
    }

    // 피격 시 빨간색 깜빡임
    m_HitTimer -= DeltaTime;
    CMaterial* Material = m_Sprite->GetMaterial(0);
    if (m_HitTimer > 0.f)  Material->SetBaseColor(1.f, 0.f, 0.f, m_Alpha);
    else                   Material->SetBaseColor(1.f, 1.f, 1.f, m_Alpha);
}
```

- **양손 좌우 반전** — RightHand는 `SetHorizontalReverse2DEnable(true)`로 미러링
- **두 개의 Sprite** — 본체 + 뒤 배경 원(`Belial_Circle`, `RT2D_Back2`)
- **SpawnColliderBox** — 길이 1000짜리 박스로 플레이어 진입 감지
- **상태 플래그 3종**: `m_Spawn` (등장 트리거), `m_SpawnEnd` (등장 완료), `m_PatternStop` (정지/사망)
- **HitTimer** — 피격 시 색상 빨간색으로 깜빡임 (Alpha는 유지)

---

# Inventory (인벤토리 시스템)

무기 / 액세서리 / 아이템 3카테고리 분리 + 좌우 무기 슬롯.

```cpp
class CInventory : public CWidgetWindow
{
protected:
    CSharedPtr<CImage> m_InventoryBaseImage;
    CSharedPtr<CImage> m_AccBaseImage;
    CSharedPtr<CImage> m_WeaponSelect_Left;     // 1번 무기 슬롯
    CSharedPtr<CImage> m_WeaponSelect_Right;    // 2번 무기 슬롯
    CSharedPtr<CText>  m_CoinText;
    CSharedPtr<CText>  m_Name1;
    CSharedPtr<CText>  m_WeaponMagazineMiddle2; // 탄약 표시

    Select_Weapon m_Current;
    std::vector<CInventoryButton*> m_Weapon;    // 무기
    std::vector<CInventoryButton*> m_Accs;      // 액세서리
    std::vector<CInventoryButton*> m_Items;     // 아이템

public:
    bool AddInventoryItem(CItem* Item);
    class CItem* GetWeapon() const;
    CItem* GetInventoryWeapon(int Index) const;
    void WeaponChange();
};
```

- 좌/우 무기 슬롯 → 키 입력으로 즉시 교체
- 액세서리 다중 장착 → 스탯/능력 누적
- 아이템 카테고리 분리

CSV 기반 아이템 데이터: 적·아이템 스탯을 외부 CSV에서 로드해 게임 밸런스를 데이터로 관리.

### 인벤토리 실제 슬롯 구조

```cpp
bool CInventory::Init()
{
    // 4개 무기 슬롯 (좌 1·2, 우 3·4) — 가로 배치
    for (int i = 0; i < 4; ++i)
    {
        CInventoryButton* WeaponButton = CreateWidget<CInventoryButton>("WeaponButton" + str);
        WeaponButton->SetButtonSlot(InventoryButton_Slot::Weapon);
        m_Weapon.push_back(WeaponButton);
    }

    // 무기 슬롯 위에 Left / Right 선택 강조 이미지
    m_WeaponSelect_Left  = CreateWidget<CImage>("WeaponSelect_Left");
    m_WeaponSelect_Right = CreateWidget<CImage>("WeaponSelect_Right");

    // 액세서리 슬롯 4개 — 가로 배치
    for (int i = 0; i < 4; ++i)
    {
        CInventoryButton* AccButton = CreateWidget<CInventoryButton>("AccButton" + str);
        AccButton->SetButtonSlot(InventoryButton_Slot::Acc);
        m_Accs.push_back(AccButton);
    }

    // 일반 아이템 그리드 3×5 = 15칸
    for (int y = 0; y < 3; ++y)
        for (int x = 0; x < 5; ++x)
        {
            CInventoryButton* Btn = CreateWidget<CInventoryButton>("InventoryButton" + str);
            Btn->SetButtonSlot(InventoryButton_Slot::All);
            m_Items.push_back(Btn);
        }

    // 시작 시 7종 무기 인벤토리에 사전 등록
    Item = m_Scene->SpawnObject<CShortSword>("CShortSword1");      m_Items[5] ->SetItem(Item);
    Item = m_Scene->SpawnObject<CMetalBoomerang>("MetalBoomerang"); m_Items[2] ->SetItem(Item);
    Item = m_Scene->SpawnObject<CRevolver>("Revolver");             m_Items[4] ->SetItem(Item);
    Item = m_Scene->SpawnObject<CCosmosSword>("CosmosSword");       m_Items[7] ->SetItem(Item);
    Item = m_Scene->SpawnObject<CDaisyRing>("DaisyRing");           m_Items[9] ->SetItem(Item);
    Item = m_Scene->SpawnObject<CMiniEarth>("MiniEarth");           m_Items[10]->SetItem(Item);
    Item = m_Scene->SpawnObject<CTaana>("Taana");                   m_Items[11]->SetItem(Item);
}

void CInventory::Update(float DeltaTime)
{
    // 현재 선택 슬롯에 따라 선택 강조 이미지 토글
    if (m_Current == Select_Weapon::Left)
    {
        m_WeaponSelect_Left->Enable(true);
        m_WeaponSelect_Right->Enable(false);
    }
    else
    {
        m_WeaponSelect_Left->Enable(false);
        m_WeaponSelect_Right->Enable(true);
    }

    // 코인 텍스트 매 프레임 갱신
    CPlayer* Player = CGlobalValue::MainPlayer;
    m_CoinText->SetText(std::to_wstring(Player->GetCoin()).c_str());
}

void CInventory::WeaponChange()
{
    // 좌/우 토글
    m_Current = (m_Current == Select_Weapon::Left)
        ? Select_Weapon::Right
        : Select_Weapon::Left;
}

bool CInventory::AddInventoryItem(CItem* Item)
{
    // 빈 슬롯 찾아서 아이템 추가 (가득 차면 false)
    for (auto Slot : m_Items)
        if (!Slot->GetItem()) { Slot->SetItem(Item); return true; }
    return false;
}
```

- **슬롯 합계 23개** — 무기 4 + 액세서리 4 + 그리드 15
- **무기 7종 사전 스폰** + 인벤토리 그리드 슬롯에 미리 배치
- `CGlobalValue::MainPlayer` 정적 참조로 코인·플레이어 정보 접근
- `WeaponChange()`는 단순 토글, 실제 무기 교체는 Player가 `GetWeapon()` 호출

---

# MiniMap (셰이더 기반 미니맵)

화면 우측 상단 미니맵. 자체 셰이더 + Constant Buffer로 타일·오브젝트·적을 별도 색상으로 렌더.

```cpp
class CMiniMap : public CWidgetWindow
{
private:
    class CMiniMapWidget* m_MiniMapWidget;

public:
    void TileUpdate();

    void PushMiniMapInfoTile  (Vector2 Pos, Vector2 Size, Vector4 Color,
                               Vector4 EmvColor, float Opacity);
    void PushMiniMapInfoObject(Vector2 Pos, Vector2 Size, Vector4 Color,
                               Vector4 EmvColor, float Opacity);
    void PushMiniMapInfoEnemy (Vector2 Pos, Vector2 Size, Vector4 Color,
                               Vector4 EmvColor, float Opacity);
    void ObjectClear();
    void Clear();
};
```

- 타일 / 오브젝트 / 적 정보를 분리해 Push
- `MiniMapCBuffer`로 셰이더에 한꺼번에 전달 → 한 패스에 미니맵 렌더
- 위치별 컬러 / Emissive / Opacity 개별 제어

`CMiniMap` 자체는 얇은 래퍼 — 실제 데이터/렌더는 `CMiniMapWidget`에 위임.

```cpp
void CMiniMap::PushMiniMapInfoTile  (...) { m_MiniMapWidget->PushMiniMapInfoTile(...);  }
void CMiniMap::PushMiniMapInfoObject(...) { m_MiniMapWidget->PushMiniMapInfoObject(...); }
void CMiniMap::PushMiniMapInfoEnemy (...) { m_MiniMapWidget->PushMiniMapInfoEnemy(...);  }
void CMiniMap::TileUpdate()               { m_MiniMapWidget->TileUpdate(); }
void CMiniMap::Clear()                    { m_MiniMapWidget->Clear(); }
```

---

# 던전 맵 생성

비트마스크와 자료구조 기반 절차적 던전 맵 생성.

```
1. 방 데이터를 비트마스크로 정의 (상/하/좌/우 출입구)
2. unordered_map<RoomID, Room> 으로 방 관리
3. DFS 알고리즘으로 시작 방부터 분기 → 던전 구조 생성
4. 각 방의 입구가 비트마스크로 인접 방과 매칭
```

- **비트마스크**: `0b0001` = 위, `0b0010` = 아래, `0b0100` = 좌, `0b1000` = 우
  → 4비트로 한 방의 출입구 4방향을 한 변수에 표현
- **DFS**: 스택 기반 탐색으로 방을 연결, 분기·막다른 길 생성
- **unordered_map**: 좌표 → 방 정보 O(1) 조회
- 보스방, 상점방, 보물방 등 특수 방 타입 지정

`CGate` — 방 전환 트리거 액터. 플레이어가 닿으면 다음 방으로 텔레포트.

---

# Ability (능력 시스템)

5가지 능력 카테고리 — 액세서리 / 보상 시 능력 부여.

| 능력 | 효과 분류 |
|---|---|
| **Wrath (분노)** | 공격력·치명타 증가 계열 |
| **Swiftness (신속)** | 이동 속도·공격 속도 |
| **Patience (인내)** | 체력·방어력 |
| **Arcane (신비)** | 마법·스킬 강화 |
| **Greed (탐욕)** | 골드 획득량 |

```cpp
class CAbility : public CWidgetWindow
{
private:
    CWidgetWindow* m_Wrath;
    CWidgetWindow* m_Swiftness;
    CWidgetWindow* m_Patience;
    CWidgetWindow* m_Arcane;
    CWidgetWindow* m_Greed;
};
```

각 카테고리별 위젯 (`AbilityWidget_Wrath`, `_Swiftness`, `_Patience`, `_Arcane`, `_Greed`).

---

# UI (37종)

`CUIManager`로 위젯 흐름 관리. 인게임 + 인벤토리 + 미니맵 + NPC 인터랙션 + 보스/엔딩 풀세트.

## 인게임 HUD

| 위젯 | 기능 |
|---|---|
| `MainHUDWidget` | 메인 HUD |
| `PlayerUI` / `PlayerWorldInfoWidget` / `PlayerWorldWidget` | 체력 / SP / 머리 위 정보 |
| `WeaponUI` | 무기 / 탄약 표시 |
| `StatusUI` | 캐릭터 스탯 패널 |
| `EnemyWorldInfoWidget` | 적 머리 위 체력바 |
| `KeyboardUIObject` | 키 안내 |

## Inventory / Ability

| 위젯 | 기능 |
|---|---|
| `Inventory` + `InventoryButton` | 인벤토리 그리드 |
| `Ability` + `AbilityWidget_*` (5종) | 능력 패널 |
| `ItemInfoWidget` | 아이템 정보 툴팁 |

## 미니맵 / 스테이지

| 위젯 | 기능 |
|---|---|
| `MiniMap` + `MiniMapWidget` | 미니맵 |
| `StageMap` | 전체 던전 맵 |
| `GateButton` | 방 이동 안내 |

## NPC 인터랙션

| NPC / UI | 기능 |
|---|---|
| `CShopNPC` + `ShopUI` + `ShopButton` + `ShopInfoWidget` | 상점 |
| `CRestaurantNPC` + `RestaurantUI` + `RestaurantButton` + `ResturantInfoWidget` | 식당 (체력 회복) |

## 보스 / 엔딩 / 기타

| 위젯 | 기능 |
|---|---|
| `BossUI` | 보스 체력바 |
| `BossSpawnUI` | 보스 등장 연출 |
| `BossDieUI` | 보스 처치 연출 |
| `EndingUI` | 엔딩 화면 |
| `LoadingUI` | 로딩 |
| `FadeInOutUI` / `FadeInOut_White` | 페이드 전환 |
| `TitleWidget` / `TextWidget` | 타이틀 / 텍스트 |
| `BasicMouse` | 마우스 커서 |

---

# 펫 / 아이템 / 보물

| 액터 | 역할 |
|---|---|
| `CPet` | 플레이어를 따라다니는 펫 (능력 부여) |
| `CHPFairy` | 체력 회복 요정 |
| `CItem` | 아이템 베이스 |
| `CGold` / `CGoldBullion` | 골드 / 금괴 |
| `CBasicTresure` | 일반 보물 상자 |
| `CBossTresure` | 보스 처치 보물 |
| `CMainDoor` / `CGate` | 방 전환 문 / 게이트 |

---

# Effect

`CEffectObject` 베이스 + 종류별 이펙트.

| 이펙트 | 표현 |
|---|---|
| `CDoorEffect` | 문 열림/닫힘 |
| `CFlameEffect` | 화염 |
| `CObjectDieEffectObject` | 오브젝트 사망 |
| `CSpawnEffect` | 스폰 이펙트 |
| `CParticle2D` | 2D 파티클 |
| `CMainStageMapEffect` / `CStage1MapEffect` | 스테이지 환경 이펙트 |
| `CTorchLight` / `CBossTorchLight` | 횃불 라이트 |
| `CCloud` / `CBackGround_Tree` | 배경 (구름, 나무) |

## 보스 사망 연출 시퀀스

```
BossDieEffectStart → BossDieEffect → BossDieEffectLast
                        ↓
                  BossDieParticle
                        ↓
              BelialDeadHead / Mouth (분리)
```

여러 이펙트 액터를 단계별로 연쇄 스폰 → 일반 사망과 다른 보스 연출.

---

# 시작 화면 / 엔딩

## StartScene

- `BackGround` — 배경
- `Bird` + `CBirdAnimation2D` — 비행 새 (분위기 연출)

## EndingScene (시네마틱)

| 액터 | 역할 |
|---|---|
| `EndingCamera` | 엔딩 전용 카메라 |
| `EndingPlayer` | 엔딩용 플레이어 (말 위) |
| `EndingHorse` | 말 |
| `EndingMountain` | 산 배경 (패럴랙스) |
| `EndingCloud` | 구름 |
| `EndingTerrain` | 지형 |

각 액터 독립적으로 움직이며 시네마틱 구성.

---

# 매니저 / 글로벌

- `CClientManager` — 클라이언트 전체 흐름
- `CUIManager` — 위젯 흐름 관리
- `GlobalValue` — 게임 전역 값 (방향 enum, 무기 인덱스 등)
- `BasicStatus` — 적 스탯 베이스 데이터

---

# 기술 스택

- C++, DirectX 11, HLSL
- FMOD, ImGui
- 자체 FSM (CEngineFSM, 멤버 함수 포인터 콜백)
- Animation2D_FSM — 5종 시퀀스 등록 후 전환
- 3-Layer Collider (Body / Horizon / Vertical) 분리 처리
- 비트마스크 + DFS + unordered_map 기반 절차적 던전 생성
- CSV 데이터 드리븐 (적·아이템 스탯)
- 셰이더 기반 미니맵 (Custom CBuffer)
- 멀티 무기 (6종) + 액세서리 시스템
- Ability 5분류 능력 시스템
- 보스 다중 패턴 (Sword / Bullet / Laser) + 단계별 사망 연출
