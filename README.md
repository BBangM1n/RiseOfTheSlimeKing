# RiseOfTheSlimeKing
## 게임 장르 : 2D RPG
## 게임 소개 : 숲속에 숨어있던 슬라임들이 '무언가'에 의해 점점 이상현상을 보이는 스토리의 2D 성장형 방식 RPG 게임입니다.
## 개발 목적 : Unity 기반의 2D RPG 프로젝트에서 데이터 기반 퀘스트, 전투, 인벤토리, 세이브 시스템을 직접 설계 및 구현했으며, Object Pooling과 Addressables 등 을 활용한 성능 최적화 경험을 하기위해 제작하였습니다.
## 사용 엔진 : UNITY 2022.3.62f2
## 개발 기간 : 2025.11.22 ~ 2026.01.22
## 사용 기술
- Unity(C#)
- Unity Addressables
- Unity UI / EventSystem
- CSV 기반 데이터 관리
- Object Pooling
- LINQ
- JSON 기반 세이브 / 로드 형식
- UniTask
- DOTween
## 포트폴리오 빌드 파일 
- //
## 포트폴리오 에셋 파일 
- //
## 유튜브 영상 링크
- //
## 주요 활용 기술
## #01) [데이터 기반 게임 구조 설계]
- CSV -> Dictionary 구조로 변환하여
  몬스터 / 아이템 / 퀘스트 / 대화 데이터 관리
- ID(Code) 기반 데이터 참조 방식 허용
  QuestID, ItemCode, MonsterID 등
- 런타임에서 데이터 변경없이 콘텐츠 확장 가능한 구조 설계
  
<details>
  
<summary>적용 코드</summary>

```
    public static async UniTask<List<MonsterData>> LoadAddressablesAsync(string addressKey)
    {
        var list = new List<MonsterData>();

        var handle = Addressables.LoadAssetAsync<TextAsset>(addressKey);
        TextAsset csv = await handle.Task;

        if (handle.Status != AsyncOperationStatus.Succeeded || csv == null)
        {
            Debug.LogError($"[MonsterCSVLoader] CSV load failed: {addressKey}");
            Addressables.Release(handle);
            return list;
        }

        Parse(csv.text, list);
        Addressables.Release(handle);

        Debug.Log($"[MonsterCSVLoader] Loaded: {list.Count} monsters");
        return list;
    }

    private static void Parse(string text, List<MonsterData> list)
    {
        string[] lines = text.Split('\n');

        for (int i = 1; i < lines.Length; i++)
        {
            string line = lines[i].Trim();
            if (string.IsNullOrWhiteSpace(line))
                continue;

            string[] v = line.Split(',');

            if (v.Length < 8)
            {
                Debug.LogWarning($"[MonsterCSVLoader] Invalid row (columns < 8): {line}");
                continue;
            }

            int maxHp = ParseIntSafe(v[3], "MaxHP", v[0], line);
            float moveSpeed = ParseFloatSafe(v[4], "MoveSpeed", v[0], line);
            int attack = ParseIntSafe(v[5], "Attack", v[0], line);
            int defense = ParseIntSafe(v[6], "Defense", v[0], line);
            int exp = ParseIntSafe(v[7], "Exp", v[0], line);

            var data = new MonsterData
            {
                ID = v[0].Trim(),
                KR = v[1].Trim(),
                PoolKey = v[2].Trim(),
                MaxHP = maxHp,
                MoveSpeed = moveSpeed,
                Attack = attack,
                Defense = defense,
                Exp = exp
            };

            list.Add(data);
        }
    }

```

</details>

***

## #02) [퀘스트 시스템]
- Quest / Objective / Reward 분리 구조
- Objective 타입 기반 처리
  Talk / Kill / Collect 등
- 퀘스트 상태 관리
  NotAccepted / InProgress / ReadyToClear / Completed
- 퀘스트 진행도 저장 및 복원
- 퀘스트 완료 시
  보상지급
  아이템 소모
  다음 퀘스트 자동 연결
  
<details>
  
<summary>적용 코드</summary>

```

public class Quest
{
    public string QuestID;
    public string Title;
    public string StartDialogueID;
    public string ClearDialogueID;
    public string RewardID;
    public string NextQuestCode;

    public List<QuestObjective> Objectives = new List<QuestObjective>();
}

public class QuestObjective
{
    public string Type;
    public string TargetID;
    public int Total;
    public int Current = 0;
}

public enum QuestState
{
    NotAccepted,
    InProgress,
    ReadyToClear,
    Completed
}

```

</details>

***

## #03) [대화(Dialogue) 시스템]
- DialogueDatabase 기반 대화 데이터 관리
- 대화 시작 / 진행 / 종료 흐름 제어
- 퀘스트 시작 / 완료 대화 분리
- Next 버튼 입력 처리 및 대화 인덱스 관리
- 대화 상태 미존재 시 입력 방어 처리
  
<details>
  
<summary>적용 코드</summary>

```

    public Dictionary<string, List<DialogueLine>> DialogueGroups = new();

    private bool isLoaded;

    public async UniTask LoadAll()
    {
        if (isLoaded)
            return;

        await LoadDialogues();
        isLoaded = true;

        Debug.Log("[대사 데이터 베이스] 모든 이름 데이터 로드 완료");
    }

    private async UniTask LoadDialogues()
    {
        DialogueGroups.Clear();

        var data = await QuestCSVLoader.LoadCSV("Data/Dialogue");

        foreach (var row in data)
        {
            if (!row.TryGetValue("DialogueID", out var dialogueID) || string.IsNullOrEmpty(dialogueID))
                continue;

            DialogueLine line = new DialogueLine
            {
                DialogueID = dialogueID,
                Order = ParseInt(row, "Order", 0),
                Speaker = row.TryGetValue("Speaker", out var sp) ? sp : "",
                Text = row.TryGetValue("Text", out var txt) ? txt : "",
                NextDialogueID = row.TryGetValue("NextDialogueID", out var next) ? next : ""
            };

            if (!DialogueGroups.ContainsKey(dialogueID))
                DialogueGroups[dialogueID] = new List<DialogueLine>();

            DialogueGroups[dialogueID].Add(line);
        }

        foreach (var key in DialogueGroups.Keys)
            DialogueGroups[key].Sort((a, b) => a.Order.CompareTo(b.Order));

        int count = 0;
        foreach (var pair in DialogueGroups)
            count += pair.Value.Count;

        Debug.Log($"[대화 데이터] 로드된 대사 수 : {count}");
    }

    public List<DialogueLine> GetDialogue(string dialogueID)
    {
        if (DialogueGroups.TryGetValue(dialogueID, out var list))
            return list;

        Debug.LogWarning($"[대화 데이터] 대사 ID를 찾을 수 없습니다: {dialogueID}");
        return null;
    }

    private int ParseInt(Dictionary<string, string> row, string key, int def = 0)
    {
        if (!row.TryGetValue(key, out var s))
            return def;

        return int.TryParse(s, out var v) ? v : def;
    }

```

</details>

***

## #04) [전투 및 몬스터 시스템]
- Enemy 추상 클래스 기반 공통 전투 로직 설계
- 몬스터별 상속 구조 (Slime, KingSlime)
- 데미지 처리
- 피격 이펙트
- 사망 처리 및 퀘스트 연동
- 경험치 지급 이벤트 처리
  
<details>
  
<summary>적용 코드</summary>

```

public class Slime : Enemy
{
    [Header("Move Settings")]
    [SerializeField] private float directionChangeInterval = 2f;
    [SerializeField] private LayerMask wallLayer;
    [SerializeField] private float wallCheckDistance = 0.1f;

    [Header("Visual")]
    [SerializeField] private Transform visualRoot;

    private Vector2 moveDirection;
    private float timer;

    protected override void OnEnable()
    {
        base.OnEnable();
        PickRandomDirection();
        timer = 0f;
    }

    private void Update()
    {
        if (!CanUpdate())
            return;

        Move();
    }

    public override void Move()
    {
        if (moveDirection != Vector2.zero)
        {
            var hit = Physics2D.Raycast(transform.position, moveDirection, wallCheckDistance, wallLayer);
            if (hit.collider != null)
            {
                moveDirection = -moveDirection;
                UpdateFlip();
                return;
            }
        }

        transform.Translate(moveDirection * data.MoveSpeed * Time.deltaTime);

        timer += Time.deltaTime;
        if (timer >= directionChangeInterval)
        {
            PickRandomDirection();
            timer = 0f;
        }
    }

    private void PickRandomDirection()
    {
        moveDirection = Random.Range(0, 3) switch
        {
            0 => Vector2.left,
            1 => Vector2.zero,
            2 => Vector2.right,
            _ => Vector2.zero
        };

        UpdateFlip();
    }

    private void UpdateFlip()
    {
        if (moveDirection.x == 0f)
            return;

        if (visualRoot == null)
            return;

        visualRoot.localScale = new Vector3(Mathf.Sign(moveDirection.x), 1f, 1f);
    }
}

```

</details>

***

## #05) [스킬 시스템]
- 플레이어 스킬 구조 분리
  SkillData / SkillBase / SkillManager
- 스킬 슬롯(QWER) 시스템
- 스킬 장착 / 해제 처리
- 스킬 발동 시 이펙트 및 데미지 처리
  
<details>
  
<summary>적용 코드</summary>

```

public class SkillData
{
    public int Code;
    public string Name;
    public string Key;
    public string Explain;

    public float Damage;

    public float CoolTime;

    public float MpCost;

    public int RequiredLevel;
}

public abstract class SkillBase : MonoBehaviour
{
    [Header("Skill Settings")]
    [SerializeField] protected float speed = 10f;
    [SerializeField] protected float lifetime = 3f;

    [Header("Stat")]
    [SerializeField] protected float damage = 100f;

    [Header("Pool")]
    [SerializeField] private string poolKey;
    public string PoolKey => poolKey;

    protected PlayerModel owner;

    public void SetOwner(PlayerModel player)
    {
        owner = player;
    }

    public float DamagePercent => damage;

    public float Damage
    {
        get
        {
            if (owner == null)
                return Mathf.Max(1f, damage);

            float baseDamage = owner.Damage;
            float result = baseDamage * (damage * 0.01f);
            return Mathf.Max(1f, result);
        }
    }

    public int Code { get; private set; } = -1;
    public float Cooldown { get; protected set; }

    protected float spawnTime;

    protected virtual void OnEnable()
    {
        spawnTime = Time.time;
    }

    public void SetCode(int code)
    {
        Code = code;
    }

    public virtual void ApplyData(SkillData data)
    {
        damage = data.Damage;
        Cooldown = data.CoolTime;
        Code = data.Code;
    }

    public virtual void ActivateSkill(Vector3 direction)
    {
        SetDirection(direction);
    }

    protected virtual void Update()
    {
        Move();
        CheckLifetime();
    }

    public abstract void SetDirection(Vector3 direction);
    protected abstract void Move();

    private void CheckLifetime()
    {
        if (Time.time - spawnTime >= lifetime)
            Release();
    }

    protected virtual void Release()
    {
        if (!string.IsNullOrEmpty(poolKey) && PoolManager.Instance != null)
            PoolManager.Instance.ReleaseObject(poolKey, gameObject);
        else
            Destroy(gameObject);
    }

    protected virtual void OnTriggerEnter2D(Collider2D collision)
    {
        if (!collision.CompareTag("Monster"))
            return;

        Enemy enemy = collision.GetComponent<Enemy>();
        if (enemy != null)
            enemy.TakeDamage(Damage);

        Release();
    }
}

```

</details>

***

## #06) [인벤토리 & 장비 시스템]
- 아이템 타입 분리
  Weapon / Consumable / Material / Etc
- 장비 슬롯 제한
  Head / Body / Weapon / Consumable
- 장착 시 PlayerModel 스탯 실시간 반영
- 드래그 앤 드롭 UI 구조
- 소비 아이템 즉시 사용 처리
  
<details>
  
<summary>적용 코드</summary>

```

public enum EquipSlot
{
    Head,
    Body,
    Weapon,
    HP,
    MP,
    Etc
}

 public void OnPickupItem(ItemDropData data)
 {
     if (data == null)
         return;

     if (data.type == DropItemType.Coin)
     {
         int amount = UnityEngine.Random.Range(data.minCoinAmount, data.maxCoinAmount + 1);
         int gold = amount * data.coinValuePerUnit;

         LogPupupManager.Instance.AddGold(gold);
         GameManager.Instance.goldUpdate(gold);
         return;
     }

     if (!ExcelReader.Instance.dicItem.TryGetValue(data.itemCode, out Item item))
     {
         Debug.LogError($"[인벤토리] 아이템 코드가 존재하지 않습니다: {data.itemCode}");
         return;
     }

     AddItem(item);

     switch (item.Type)
     {
         case ItemKind.Weapon:
             LogPupupManager.Instance.AddItem(item.Name);
             break;

         case ItemKind.Consumable:
             LogPupupManager.Instance.AddConsume(item.Name);
             ConsumableUI.Instance.Refresh();
             break;

         case ItemKind.Material:
         case ItemKind.Etc:
             LogPupupManager.Instance.AddMaterial(item.Name);
             break;
     }

     if (!string.IsNullOrEmpty(data.itemCode))
         QuestManager.Instance.CheckObjective("Collect", data.itemCode);
 }

 public void AddItem(Item item)
 {
     if (item == null)
         return;

     InventoryItemCategory category = ConvertCategory(item);

     bool stackable =
         item.Type == ItemKind.Consumable ||
         item.Type == ItemKind.Material ||
         item.Type == ItemKind.Etc;

     if (stackable)
     {
         InventoryItem exist = inventory[category]
             .Find(i => i.item.Code == item.Code && i.count < InventoryItem.MAX_STACK);

         if (exist != null)
         {
             exist.count++;
         }
         else
         {
             inventory[category].Add(new InventoryItem(item, 1));
         }
     }
     else
     {
         inventory[category].Add(new InventoryItem(item, 1));
     }

     UpdateInventoryUI(inventoryUI.CurrentCategory);
 }

```

</details>

***

## #07) [세이브 / 로드 시스템]
- 슬롯 기반 세이브 구조 (다중 슬롯)
- 저장 대상
  플레이어 스탯
  인벤토리
  장비
  퀘스트 진행도
  플레이 타임
- 씬 이동 시 자동 세이브
- 로드 후 UI및 상태 복원
  
<details>
  
<summary>적용 코드</summary>

```

    public void SaveGame()
    {
        SaveGame(CurrentSlotIndex);
    }

    public void SaveGame(int slotIndex)
    {
        var data = new SaveData();

        FillPlayerData(data);
        FillGameManagerData(data);
        FillOptionalManagersData(data);
        FillInventoryData(data);
        FillAudioData(data);
        FillTimeData(data);

        WriteToDisk(slotIndex, data);
    }

    public void LoadGame()
    {
        LoadGame(CurrentSlotIndex);
    }

    public void LoadGame(int slotIndex)
    {
        string path = GetSavePath(slotIndex);
        if (!File.Exists(path))
        {
            Debug.LogWarning($"[SaveManager] 세이브 파일이 없습니다 : {path}");
            return;
        }

        SaveData data = ReadFromDisk(path);

        ApplyPlayerData(data);
        ApplyGameManagerData(data);
        ApplyOptionalManagersData(data);
        ApplyInventoryData(data);
        ApplyTimeData(data);
        ApplyAudioData(data);

        ShouldLoadOnMain = false;

        Debug.Log($"[SaveManager] Load complete: {path}");
        GameManager.Instance.RefreshUIAfterLoad();
    }

```

</details>

***

## #08) [오브젝트 풀링 및 성능 최적화]
- 공용 풀 + 몬스터 전용 풀 분리
- 풀 매니저 중앙 관리 구조
- 데미지 텍스트 / 몬스터 / 이펙트 / 아이템 재사용
- 씬 전환 시 풀 등록 / 해제 처리
  
<details>
  
<summary>적용 코드</summary>

```

public class PoolManager : MonoBehaviour
{
    public static PoolManager Instance;

    [SerializeField] private CommonPoolManager commonPool;
    private MonsterPoolManager monsterPool;

    private void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }

        Instance = this;

        if (commonPool == null)
            commonPool = GetComponentInChildren<CommonPoolManager>(true);

        if (commonPool == null)
            Debug.LogError("[풀 매니저] CommonPoolManager가 연결되지 않았습니다.");
    }

    public void RegisterMonsterPool(MonsterPoolManager pool)
    {
        monsterPool = pool;
    }

    public void UnregisterMonsterPool(MonsterPoolManager pool)
    {
        if (monsterPool == pool)
            monsterPool = null;
    }

    public GameObject GetObject(string key, Vector3 position, Quaternion rotation)
    {
        if (commonPool != null && commonPool.HasKey(key))
            return commonPool.GetObject(key, position, rotation);

        if (monsterPool != null && monsterPool.HasKey(key))
            return monsterPool.GetObject(key, position, rotation);

        Debug.LogError($"[풀 매니저] 요청한 키를 찾을 수 없습니다 : {key}");
        return null;
    }

    public void ReleaseObject(string key, GameObject obj)
    {
        if (obj == null)
            return;

        if (commonPool != null && commonPool.HasKey(key))
        {
            commonPool.ReleaseObject(key, obj);
            return;
        }

        if (monsterPool != null && monsterPool.HasKey(key))
        {
            monsterPool.ReleaseObject(key, obj);
            return;
        }

        Destroy(obj);
    }
}

```

</details>

***

## #09) [UI 이벤트 및 상호작용 처리]
- EventSystem 기반 클릭 / 입력 처리
- NPC 상호작용 시스템
- 상점 UI 및 아이템 구매 처리
- 로그 팝업 시스템 (골드 / 경험치 / 아이템)
  
<details>
  
<summary>적용 코드</summary>

```

    private void HandleNPCInteraction()
    {
        if (ChatManager.Instance.IsUIBlockingInput)
            return;

        if (!Input.GetMouseButtonDown(0))
            return;

        Vector2 mousePos = Camera.main.ScreenToWorldPoint(Input.mousePosition);
        Collider2D[] hits = Physics2D.OverlapPointAll(mousePos);

        if (hits == null || hits.Length == 0)
            return;

        foreach (var h in hits)
        {
            NPC npc = h.GetComponentInParent<NPC>();
            if (npc != null)
            {
                npc.Talk();
                return;
            }
        }

        foreach (var h in hits)
        {
            IInteractable interactable = h.GetComponentInParent<IInteractable>();
            if (interactable != null)
            {
                interactable.Interact();
                return;
            }
        }
    }

```

</details>

***

## #10) [사운드 시스템]
- AudioManager 중심 구조
- BGM / SFX 분리 관리
- 씬 이름 기반 BGM 자동 전환
- 볼륨 조절 및 토글 UI
  
<details>
  
<summary>적용 코드</summary>

```

public class AudioManager : MonoBehaviour
{
    public static AudioManager Instance;

    [Header("BGM")]
    [SerializeField] private AudioSource bgmSource;
    [SerializeField] private BgmEntry[] bgmEntries;

    [Header("SFX")]
    [SerializeField] private AudioSource sfxSource;
    [SerializeField] private SfxEntry[] sfxEntries;

    private readonly Dictionary<BgmType, AudioClip> bgmMap = new();
    private readonly Dictionary<SfxType, AudioClip> sfxMap = new();

    private BgmType currentBgmType = BgmType.None;

    public float BgmVolume
    {
        get => bgmSource != null ? bgmSource.volume : 0f;
        set
        {
            if (bgmSource == null) return;
            bgmSource.volume = Mathf.Clamp01(value);
        }
    }

    public float SfxVolume
    {
        get => sfxSource != null ? sfxSource.volume : 0f;
        set
        {
            if (sfxSource == null) return;
            sfxSource.volume = Mathf.Clamp01(value);
        }
    }

    private void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }

        Instance = this;
        DontDestroyOnLoad(gameObject);

        foreach (var entry in sfxEntries)
        {
            if (entry.clip != null)
                sfxMap[entry.type] = entry.clip;
        }

        foreach (var entry in bgmEntries)
        {
            if (entry.clip != null)
                bgmMap[entry.type] = entry.clip;
        }

        if (bgmSource != null) bgmSource.volume = 0.3f;
        if (sfxSource != null) sfxSource.volume = 0.3f;

        SceneManager.sceneLoaded += OnSceneLoaded;
        ChangeBgmBySceneName(SceneManager.GetActiveScene().name);
    }

    private void OnDestroy()
    {
        SceneManager.sceneLoaded -= OnSceneLoaded;
    }

    private void OnSceneLoaded(Scene scene, LoadSceneMode mode)
    {
        ChangeBgmBySceneName(scene.name);
    }

    private BgmType GetBgmTypeForScene(string sceneName)
    {
        return sceneName switch
        {
            "Start" => BgmType.Title,
            "Load" or "DownLoad" or "Loading" => currentBgmType,
            "Main" or "Field1" or "Field2" or "Field3" or "Field4" => BgmType.Field,
            "Field5" => BgmType.Boss,
            _ => BgmType.None
        };
    }

    private void ChangeBgmBySceneName(string sceneName)
    {
        BgmType targetType = GetBgmTypeForScene(sceneName);
        if (targetType == currentBgmType)
            return;

        currentBgmType = targetType;

        if (targetType == BgmType.None)
        {
            StopBgm();
            return;
        }

        if (!bgmMap.TryGetValue(targetType, out AudioClip clip))
            return;

        PlayBgm(clip);
    }

    public void PlayBgm(AudioClip clip, bool loop = true)
    {
        if (bgmSource == null || clip == null) return;

        bgmSource.clip = clip;
        bgmSource.loop = loop;
        bgmSource.Play();
    }

    public void StopBgm()
    {
        if (bgmSource == null) return;
        bgmSource.Stop();
        currentBgmType = BgmType.None;
    }

    public void PlaySfx(SfxType type, float volume = 1f)
    {
        if (sfxSource == null) return;
        if (!sfxMap.TryGetValue(type, out AudioClip clip)) return;

        sfxSource.PlayOneShot(clip, volume);
    }
    public void SetBgmVolume(float value)
    {
        BgmVolume = value;
    }

    public void SetSfxVolume(float value)
    {
        SfxVolume = value;
    }
}

```

</details>

***

## #11) [씬 & 월드 관리]
- 로딩 씬 -> 메인 -> 필드 구조
- Warp / Portal 기반 씬 이동
- 스폰 포인트 ID 기반 위치 이동
- PersistentRoot 기반 DDOL 구조 관리
  
<details>
  
<summary>적용 코드</summary>

```

public class Warp : MonoBehaviour
{
    [SerializeField] private string warpSceneName;
    [SerializeField] private string targetSpawnPointID;

    private bool isPlayerInside;

    private void OnTriggerEnter2D(Collider2D collision)
    {
        if (!collision.CompareTag("Player"))
            return;

        isPlayerInside = true;
    }

    private void OnTriggerExit2D(Collider2D collision)
    {
        if (!collision.CompareTag("Player"))
            return;

        isPlayerInside = false;
    }

    private void Update()
    {
        if (!isPlayerInside)
            return;

        if (Input.GetKeyDown(KeyCode.UpArrow))
            WarpToScene();
    }

    private void WarpToScene()
    {
        if (string.IsNullOrEmpty(warpSceneName))
        {
            Debug.LogWarning("[워프] 이동할 씬 이름이 설정되지 않았습니다.");
            return;
        }

        if (SaveManager.Instance != null)
            SaveManager.Instance.SaveGame();

        SceneLoader.LoadScene(warpSceneName, targetSpawnPointID);
    }
}

public class Loading : MonoBehaviour
{
    [SerializeField] private Slider loadingBar;

    private void Start()
    {
        StartCoroutine(LoadRoutine());
    }

    private IEnumerator LoadRoutine()
    {
        yield return null;

        string targetScene = SceneLoader.TargetSceneName;
        if (string.IsNullOrEmpty(targetScene))
        {
            Debug.LogError("[Loading] Target scene is not set.");
            yield break;
        }

        AsyncOperation op = SceneManager.LoadSceneAsync(targetScene);
        op.allowSceneActivation = false;

        float t = 0f;

        while (!op.isDone)
        {
            yield return null;
            t += Time.deltaTime;

            float target = (op.progress < 0.9f) ? op.progress : 1f;
            loadingBar.value = Mathf.Lerp(loadingBar.value, target, t);

            if (loadingBar.value >= target - 0.0001f)
                t = 0f;

            if (loadingBar.value >= 0.99f)
            {
                yield return new WaitForSeconds(1f);
                op.allowSceneActivation = true;
                yield break;
            }
        }
    }
}


```

</details>

***

## #12) [MVC 패턴 기반 플레이어 구조 설계]
- PlayerModel / PlayerView / PlayerController 분리
- 책임 분리 원칙 적용
  Model : 스탯, 수치, 상태 관리
  View : 애니메이션, 시각적 표현
  Controller : 입력 처리, 행동 제어
- 전투, 장비, 스킬 시스템 확장 시
  기존 코드 영향 최소화
  
***

## #13) [이벤트 기반 구조 설계]
- C# event / Action 활용
- 직접 참조 최소화
  HP 변경 이벤트
  EXP 변경 이벤트
  UI 갱신 이벤트
- 시스템 간 결합도 감소
  
<details>
  
<summary>적용 코드</summary>

```

public static class ExpEvent
{
    public static Action<int> OnAddExp;
}

protected virtual void Die()
{
  OnHPChanged?.Invoke(0f, MaxHP);

 if (data != null)
 {
   LogPupupManager.Instance.AddExp(data.Exp);
   ExpEvent.OnAddExp?.Invoke(data.Exp);

       QuestManager.Instance.CheckObjective("Kill", data.ID);
  }

  dropper?.DropItems();

  if (data != null && !string.IsNullOrEmpty(data.PoolKey) && PoolManager.Instance != null)
      PoolManager.Instance.ReleaseObject(data.PoolKey, gameObject);
  else
      gameObject.SetActive(false);
 }

 private void OnEnable()
  {
      ExpEvent.OnAddExp += GetExp;
  }

 private void OnDisable()
  {
     ExpEvent.OnAddExp -= GetExp;
  }

```

</details>

***

## #14) [UniTask 기반 비동기 처리 구조]
- Cysharp.Threading.Tasks (UniTask) 사용
- 씬 로딩 / 전환 / 초기화 로직에서 Coroutine 대체
- GC 최소화 + 가독성 향상
- async/await 스타일로 흐름 제어
  
<details>
  
<summary>적용 코드</summary>

```

    private async UniTask TryStartDash(int direction)
    {
        if (model.isDashing) return;
        if (Time.time - model.lastDashTime < model.dashCooldown) return;

        await DashRoutine(direction);
    }

    private async UniTask DashRoutine(int direction)
    {
        model.SetDashing(true);
        model.lastDashTime = Time.time;
        dashTimer = 0f;

        AudioManager.Instance?.PlaySfx(SfxType.Dash);

        while (dashTimer < model.dashDuration)
        {
            dashTimer += Time.deltaTime;
            Vector2 velocity = view.GetVelocity();
            velocity.x = direction * model.dashSpeed;
            view.SetVelocity(velocity);
            await UniTask.Yield();
        }

        model.SetDashing(false);
    }

```

</details>

***

## #15) [Addressables 기반 리소스 관리 및 최적화]
- 라벨 기반 Addressables 다운로드 구조
- 런타임에서 필요한 리소스만 다운로드
- 업데이트 여부 판단 후 씬 진입 제어
- 대규모 리소스 확장 대비 구조
- 
<details>
  
<summary>적용 코드</summary>

```

    private IEnumerator InitializeAndCheckRoutine()
    {
        var initHandle = Addressables.InitializeAsync();
        yield return initHandle;

        yield return CheckUpdateFilesRoutine();
    }

    private IEnumerator CheckUpdateFilesRoutine()
    {
        totalDownloadSize = 0;
        labelsToDownload.Clear();
        patchMap.Clear();

        if (labels == null || labels.Length == 0)
        {
            Debug.LogWarning("[DownLoad] No labels set in inspector.");
            SetUpToDateUI();
            yield return new WaitForSeconds(1f);
            SceneLoader.LoadScene(NextSceneName);
            yield break;
        }

        foreach (var labelRef in labels)
        {
            if (labelRef == null || string.IsNullOrEmpty(labelRef.labelString))
                continue;

            string label = labelRef.labelString;

            var sizeHandle = Addressables.GetDownloadSizeAsync(label);
            yield return sizeHandle;

            long size = sizeHandle.Result;
            Addressables.Release(sizeHandle);

            if (size > 0)
            {
                totalDownloadSize += size;
                labelsToDownload.Add(label);
                patchMap[label] = 0;
            }
        }

        if (totalDownloadSize > 0 && labelsToDownload.Count > 0)
        {
            SetNeedUpdateUI(totalDownloadSize);
            yield break;
        }

        SetUpToDateUI();
        yield return new WaitForSeconds(2f);
        SceneLoader.LoadScene(NextSceneName);
    }

    public void OnClickDownloadButton()
    {
        if (totalDownloadSize <= 0 || labelsToDownload.Count == 0)
        {
            SceneLoader.LoadScene(NextSceneName);
            return;
        }

        AudioManager.Instance?.PlaySfx(SfxType.ButtonClick);
        StartCoroutine(DownloadRoutine());
    }

    private IEnumerator DownloadRoutine()
    {
        SetDownloadStartUI();

        foreach (var label in labelsToDownload)
        {
            StartCoroutine(DownloadLabelRoutine(label));
        }

        yield return TrackProgressRoutine();

        SetDownloadCompleteUI();
        SceneLoader.LoadScene(NextSceneName);
    }

    private IEnumerator DownloadLabelRoutine(string label)
    {
        var handle = Addressables.DownloadDependenciesAsync(label, false);

        while (!handle.IsDone)
        {
            var status = handle.GetDownloadStatus();
            patchMap[label] = (long)status.DownloadedBytes;
            yield return null;
        }

        var finalStatus = handle.GetDownloadStatus();
        patchMap[label] = (long)finalStatus.TotalBytes;

        Addressables.Release(handle);
    }

    private IEnumerator TrackProgressRoutine()
    {
        while (true)
        {
            long downloaded = patchMap.Values.Sum();

            float progress = totalDownloadSize > 0
                ? (float)downloaded / totalDownloadSize
                : 1f;

            progress = Mathf.Clamp01(progress);
            UpdateProgressUI(progress);

            bool allDone = patchMap.Count > 0 && patchMap.Values.All(v => v > 0);
            bool reachedTotal = downloaded >= totalDownloadSize;

            if (reachedTotal && allDone)
                yield break;

            yield return null;
        }
    }

```

</details>

***

## #16) [카메라 시스템 분리 및 제어 구조]
- 카메라 컨트롤러 분리 설계
- 플레이어 추적, 제한 영역, 이동 제어
- 씬별 카메라 동작 변경 가능 구조
  
<details>
  
<summary>적용 코드</summary>

```

public class MainCameraContoller : MonoBehaviour
{
    private Transform player;

    [SerializeField] private float smoothing = 0.2f;
    [SerializeField] private Vector2 minCameraBoundary;
    [SerializeField] private Vector2 maxCameraBoundary;

    private void OnEnable()
    {
        SceneManager.sceneLoaded += OnSceneLoaded;
    }

    private void OnDisable()
    {
        SceneManager.sceneLoaded -= OnSceneLoaded;
    }

    private void OnSceneLoaded(Scene scene, LoadSceneMode mode)
    {
        GameObject p = GameObject.FindGameObjectWithTag("Player");
        if (p != null)
            player = p.transform;
    }

    private void LateUpdate()
    {
        if (player == null) return;

        Vector3 target = new Vector3(
            Mathf.Clamp(player.position.x, minCameraBoundary.x, maxCameraBoundary.x),
            Mathf.Clamp(player.position.y, minCameraBoundary.y, maxCameraBoundary.y),
            transform.position.z
        );

        transform.position = Vector3.Lerp(transform.position, target, smoothing);
    }
}


```

</details>

***

## #17) [LINQ 및 컬렉션 최적 활용]
- Dictionary 기반 접근
- LINQ로 데이터 필터링 및 합산
- 반복 로직 간결화 및 가독성 개선
  
<details>
  
<summary>적용 코드</summary>

```

public static class MonsterDatabase
{
    private static Dictionary<string, MonsterData> dict;

    public static void Initialize(List<MonsterData> list)
    {
        dict = new Dictionary<string, MonsterData>();

        foreach (var data in list)
        {
            if (data == null || string.IsNullOrEmpty(data.ID))
                continue;

            if (!dict.ContainsKey(data.ID))
                dict.Add(data.ID, data);
        }

        Debug.Log($"[MonsterDatabase] Initialized: {dict.Count}");
    }

    public static MonsterData Get(string id)
    {
        if (dict == null)
        {
            Debug.LogError("[MonsterDatabase] Not initialized.");
            return null;
        }

        if (!dict.TryGetValue(id, out var data))
        {
            Debug.LogError($"[MonsterDatabase] MonsterData not found: {id}");
            return null;
        }

        return data;
    }
}


```

</details>

***
