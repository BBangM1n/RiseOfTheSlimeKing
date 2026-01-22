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
- PlayerPrefs
- JSON 기반 세이브 / 로드 형식
## 포트폴리오 빌드 파일 
- //
## 포트폴리오 에셋 파일 
- //
## 유튜브 영상 링크
- //
## 주요 활용 기술
- #01)(스크립트) [데이터 기반 게임 구조 설계]
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

- #02)(스크립트) [퀘스트 시스템]
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
