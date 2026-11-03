# ImTrainer

**Unity · C# | 2D Top-down Survival | 5인 팀 프로젝트**

![나는 트레이너다 타이틀 로고](./images/title.png)

> 프로젝트에서 사용한 '나는 트레이너다' 타이틀 로고. 실제 게임 플레이 화면은 기능별 설명과 함께 별도로 추가할 예정입니다.

ImTrainer는 적의 웨이브를 상대하며 증강을 선택해 캐릭터를 성장시키는 2D 탑다운 생존 게임입니다.

**제가 집중한 작업:** 증강 데이터·후보 선정·효과 적용, 적 관리 및 스폰 관련 로직, 전투 객체 재사용과 동작 검증. 게임 전체의 프레임워크와 모든 UI를 단독 개발한 것은 아닙니다.

---

## 1. 증강 시스템

레벨업 시 데이터에 정의된 증강 후보를 추첨하고, 선택 결과를 현재 플레이어 상태에 적용합니다. 플레이어에게 보이는 카드는 `AugmentWindowBehavior`가 담당하고, 증강 데이터·선택 결과·효과는 `UpgradeManager`가 관리합니다.

**동작 흐름**

`JSON 로딩 → 증강 정의 생성 → 레벨·해금 조건에 따른 Weight 계산 → 3개 후보 추첨 → UI 표시 → 선택 → Level 갱신 및 Effect 적용`

### 증강 후보 선정

각 후보의 현재 `Weight`를 기준으로 추첨하며, 한 번 선택된 후보는 같은 선택 화면에서 다시 뽑히지 않도록 후보 목록에서 제거합니다. 최대 Level에 도달하거나 조건을 만족하지 못한 증강은 `GetWeight()`가 0을 반환하여 제외됩니다.

<details>
<summary><b>코드 보기 — 후보 3개 추첨 (UpgradeManager.RefreshChoices)</b></summary>

```csharp
private void RefreshChoices()
{
    _choices.Clear();
    _candidates.Clear();
    foreach (var definition in _definitions)
    {
        if (GetWeight(definition.Id) > 0f)
        {
            _candidates.Add(definition);
        }
    }

    while (_choices.Count < CHOICE_COUNT && _candidates.Count > 0)
    {
        double totalWeight = 0;
        foreach (var candidate in _candidates) totalWeight += GetWeight(candidate.Id);
        var roll = UnityEngine.Random.value * totalWeight;
        var selectedIndex = _candidates.Count - 1;
        for (var i = 0; i < _candidates.Count; i++)
        {
            roll -= GetWeight(_candidates[i].Id);
            if (roll < 0)
            {
                selectedIndex = i;
                break;
            }
        }

        // Random.value는 1도 반환할 수 있다. 그 경계에서는 마지막 유효 후보를 사용한다.
        _choices.Add(_candidates[selectedIndex]);
        _candidates.RemoveAt(selectedIndex);
    }
}
```

*발췌: `Assets/Runtime/Scripts/Managers/UpgradeManager.cs` — 연결된 저장소 `main` 기준.*
</details>

### 증강 선택과 효과 적용

선택 인덱스와 현재 선택 가능 상태를 검증하고 증강 Level을 올립니다. 각 Effect의 대상과 Type에 따라 `PlayerManager`의 능력치 배율 및 특수 능력 값을 갱신합니다. UI 자체가 플레이어 능력치를 변경하지 않고 `UpgradeManager`의 선택 기능을 호출하도록 역할을 나눴습니다.

<details>
<summary><b>코드 보기 — 선택 검증 및 효과 적용 (UpgradeManager.TrySelectChoice)</b></summary>

```csharp
public bool TrySelectChoice(int choiceIndex)
{
    if (!CanSelect() ||
        choiceIndex < 0 ||
        choiceIndex >= _choices.Count)
    {
        return false;
    }

    var chosen = _choices[choiceIndex];

    // 이미 최대 Level에 도달했거나 Weight가 0이면 선택하지 않는다.
    if (GetWeight(chosen.Id) <= 0f)
    {
        return false;
    }

    var previousLevel = GetLevel(chosen.Id);

    // 선택한 Upgrade Level 증가
    _levels[chosen.Id] = previousLevel + 1;


    foreach (var effect in chosen.Effects)
    {
        if (effect.Target != UpgradeTarget.Player)
        {
            continue;
        }

        //move speed의 multiplier값을 증감시킨다.
        switch (effect.Type)
        {
            case UpgradeEffectType.MoveSpeed:
                _playerManager.AddMoveSpeedMultiplier(_playerManager.LocalPlayerId, effect.Value);
                break;
            case UpgradeEffectType.Experience:
                _playerManager.AddXpMultiplier(_playerManager.LocalPlayerId, effect.Value);
                break;
            case UpgradeEffectType.ReceivedDamage:
                _playerManager.AddReceiveDamageMultiplier(_playerManager.LocalPlayerId, effect.Value);
                break;
            case UpgradeEffectType.Damage:
                _playerManager.AddDamageMultiplier(_playerManager.LocalPlayerId, effect.Value);
                break;
            case UpgradeEffectType.MagnetUpgrade:
                _playerManager.AddMagnetRadius(_playerManager.LocalPlayerId, effect.Value);
                break;
            case UpgradeEffectType.Revive:
                _playerManager.AddRevivalChance(_playerManager.LocalPlayerId, Mathf.RoundToInt(effect.Value));
                break;
            default:
                Debug.LogWarning(
                    $"[증강] 처리되지 않은 Player Effect Type: {effect.Type}",
                    this
                );
                break;
        }
    }

    if (chosen.Grade == 2)
    {
        foreach (var augment in _definitions)
        {
            if(augment.Grade == 2)
            {
                augment.Weight = 0f;
            }
        }
    }


    // Upgrade Sequence 종료
    _active = false;
    Time.timeScale = 1f;
    Managers.Instance.ChangeState(GameState.Playing);

    return true;
}
```

*발췌: `Assets/Runtime/Scripts/Managers/UpgradeManager.cs` — 연결된 저장소 `main` 기준.*
</details>

### 선택지 UI와의 연결

UI는 매니저가 준비한 `CurrentChoices`를 읽어서 각 카드의 이름, 현재·다음 Level, 효과 설명을 채웁니다. 아래 코드는 **흐름을 설명하기 위한 연관 코드**입니다. 이 코드를 포함하는 것과 본인이 UI 전체를 구현했다고 주장하는 것은 구분합니다.

<details>
<summary><b>코드 보기 — UI에 선택지 표시 (AugmentWindowBehavior.FillCards)</b></summary>

```csharp
private void FillCards()
{
    if (Managers.Instance == null ||
        !Managers.Instance.TryGetManager<UpgradeManager>(out var upgradeManager))
    {
        return;
    }

    var choices = upgradeManager.CurrentChoices;

    for (var i = 0; i < _cards.Length; i++)
    {
        var card = _cards[i];

        var visible =
            choices != null &&
            i < choices.Count &&
            choices[i] != null;

        card.gameObject.SetActive(visible);

        if (!visible)
        {
            continue;
        }

        var choice = choices[i];
        var currentLevel = upgradeManager.GetLevel(choice.Id);
        var nextLevel = currentLevel + 1;

        card.Label.text =
            $"{choice.Name}\n\n" +
            $"Lv.{currentLevel} → Lv.{nextLevel}/{choice.MaxLevel}\n\n" +
            FormatEffects(choice);


        var icon = Resources.Load<Sprite>(choice.IconPath);

        if (icon != null)
        {
            // 회색 ArtBackground 위에 실제 Sprite를 표시한다.
            card.Icon.sprite = icon;
            card.Icon.color = Color.white;
            card.Icon.enabled = true;
        }
        else
        {
            // 아이콘만 숨긴다.
            // ArtBackground는 별도 Image라서 그대로 남는다.
            card.Icon.sprite = null;
            card.Icon.color = Color.clear;

            Debug.LogWarning(
                $"[Augment UI] Sprite를 찾지 못했습니다. " +
                $"Upgrade={choice.Id}, Path={choice.IconPath}",
                this
            );
        }

        card.Button.onClick.RemoveAllListeners();

        var choiceIndex = i;
        card.Button.onClick.AddListener(() => Choose(choiceIndex));

        // [트레이닝] 기존 코드는 그대로 두고, 새로 만든 SetupCard로 등급 색을 칠하고 클릭을 제외 모드와 함께 받는다.
        SetupCard(card, choice, choiceIndex);
    }
}
```

*연관 코드 발췌: `Assets/Runtime/Scripts/UI/AugmentWindowBehavior.cs`. 기여자별 소유·수정 범위는 별도 확인 후 표기합니다.*
</details>

---

## 2. 적 스폰 및 관리

`EnemyManager`는 스폰 요청을 받아 활성 적을 관리합니다. 스폰 수는 현재 Stage 상태, 플레이어 식별자, 최대 활성 수 등의 조건에 따라 제한합니다. 적의 생성·재사용은 아래 Object Pool과 연결됩니다.

`Wave 요청 → Spawn 위치 결정 → CreateEnemy → Pool.Get → 활성 목록 관리 → 사망 처리 및 반환`

<details>
<summary><b>코드 보기 — 스폰 조건과 생성 반복 (EnemyManager.SpawnWave)</b></summary>

```csharp
public int SpawnWave(int playerId, MonsterWaveProfile profile, int count, bool circularSpawn, Action<Enemy> onDied)
{
    if (!_stageActive || _bossFightActive || playerId != _playerId || profile == null || count <= 0)
    {
        return 0;
    }

    if (!TryGetPlayerPosition(out var playerPosition))
    {
        return 0;
    }

    var spawned = 0;
    for (var i = 0; i < count; i++)
    {
        if (_alive.Count >= MAX_ALIVE)
        {
            break;
        }

        var position = ResolveSpawnPosition(playerPosition, count, circularSpawn);
        if (CreateEnemy(profile, position, onDied, true) == null)
        {
            break;
        }

        spawned++;
    }

    return spawned;
}
```

*발췌: `Assets/Runtime/Scripts/Managers/EnemyManager.cs` — 연결된 저장소 `main` 기준.*
</details>

---

## 3. 전투 Object Pooling

적과 투사체의 반복적인 생성·파괴를 줄이기 위한 재사용 로직입니다. `CombatObjectPool`은 Prefab별 보관 Stack과 대여 중인 객체 집합을 유지합니다. `EnemyManager`가 전투 상태를 관리하고 Pool은 재사용·반환 여부만 결정합니다.

`Pool.Get → Enemy 초기화 및 사용 → 사망 후 상태 정리 → Pool.Return → 비활성 객체 재보관`

<details>
<summary><b>코드 보기 — 중복 반환 방지 (CombatObjectPool.Return)</b></summary>

```csharp
public bool Return(GameObject instance)
{
    if (_disposed || instance == null || !_inUse.Remove(instance))
    {
        return false;
    }

    instance.SetActive(false);
    if (_root != null)
    {
        instance.transform.SetParent(_root, false);
    }

    var prefab = _prefabs[instance];
    if (prefab != null)
    {
        _available[prefab].Push(instance);
    }

    return true;
}
```

*발췌: `Assets/Runtime/Scripts/Enemy/CombatObjectPool.cs` — 연결된 저장소 `main` 기준.*
</details>

**검증 범위:** 객체 재사용 및 중복 반환 방지 등의 동작 테스트를 진행했습니다. Pool 적용만으로 실제 플레이의 프레임 개선이 입증됐다고 단정하지 않습니다. 성능 비교는 테스트 환경과 측정값을 함께 제시할 수 있는 경우에만 결과를 추가합니다.

---

## 코드 발췌에 관한 표기

이 README는 **팀 프로젝트의 전체 코드가 아니라, 설명에 필요한 핵심 부분을 발췌한 포트폴리오 초안**입니다. 본인이 직접 구현·수정한 내용과 전체 동작을 이해시키기 위해 인용한 팀원 코드를 구별합니다. 코드를 기재한 것만으로 작성 책임을 주장하지 않으며, 공개 전 팀 코드와 게임 리소스의 공개 가능 범위를 확인해야 합니다.