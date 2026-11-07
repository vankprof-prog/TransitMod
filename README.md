
using System;
using System.Collections;
using System.Collections.Generic;
using System.Reflection;
using BepInEx;
using HarmonyLib;
using UnityEngine;
using UnityEngine.InputSystem;

[BepInPlugin("com.transitmod.agesofconflict", "Transit & Presidents", "1.0.0")]
[BepInProcess("Ages of Conflict.exe")]
public class TransitPlugin : BaseUnityPlugin
{
    private void Awake()
    {
        new Harmony("com.transitmod.agesofconflict").PatchAll();
        Logger.LogInfo("Transit & Presidents loaded.");
    }
}

[HarmonyPatch(typeof(MainMenu), "Start")]
public static class TransitStartupPatch
{
    private static void Postfix()
    {
        if (TransitRuntime.Instance != null)
            return;

        GameObject go = new GameObject("TransitPresidentsRuntime");
        UnityEngine.Object.DontDestroyOnLoad(go);
        go.AddComponent<TransitRuntime>();
    }
}

public class TransitRuntime : MonoBehaviour
{
    public static TransitRuntime Instance;

    private class SubwayLine
    {
        public string name;
        public string label;
        public Color color;

        public List<Vector3> stations = new List<Vector3>();
        public List<GameObject> stationObjects = new List<GameObject>();

        public GameObject train;
        public GameObject labelObject;
        public TextMesh mapLabel;
        public LineRenderer track;

        public List<Vector3> path = new List<Vector3>();
        public List<float> distances = new List<float>();
        public List<float> stationDistances = new List<float>();

        public float trainDistance;
        public float waitTimer;
        public int nextStation = 1;
        public int direction = 1;
        public bool finished;
    }

    private class President
    {
        public object nation;
        public int nationIndex;
        public string countryName;
        public string firstName;
        public string lastName;
        public string fullName;

        public Texture2D portrait;

        public float stability;
        public float popularity;
        public float economy;
        public float military;

        public float age;
        public float decisionTimer;

        public string ideology;
        public string lastDecision;
    }

    private readonly List<SubwayLine> lines = new List<SubwayLine>();
    private readonly List<President> presidents = new List<President>();
    private readonly List<string> events = new List<string>();

    private SubwayLine drawingLine;
    private int selectedLine = -1;

    private President selectedPresident;
    private int selectedTarget = -1;

    private Camera mapCamera;

    private Sprite stationSprite;
    private Sprite trainSprite;
    private Material lineMaterial;

    private Rect windowRect = new Rect(20, 20, 430, 580);

    private Vector2 lineScroll;
    private Vector2 presidentScroll;
    private Vector2 eventScroll;

    private string lineNameInput = "New Line";
    private string lineLabelInput = "A";

    private int selectedTab;
    private int selectedColor;
    private bool showWindow = true;

    private float countryRefreshTimer;

    private readonly Color[] colors =
    {
        new Color(0.95f, 0.2f, 0.2f),
        new Color(0.2f, 0.55f, 1f),
        new Color(0.15f, 0.85f, 0.4f),
        new Color(1f, 0.75f, 0.1f),
        new Color(0.75f, 0.3f, 1f),
        new Color(0.1f, 0.85f, 0.85f),
        new Color(1f, 0.4f, 0.7f),
        new Color(1f, 0.5f, 0.15f),
        Color.white
    };

    private readonly string[] firstNames =
    {
        "Alexander", "James", "William", "Daniel", "Michael",
        "Thomas", "Edward", "Henry", "Charles", "Robert",
        "George", "Victor", "Arthur", "Nicholas", "Samuel",
        "Benjamin", "Richard", "David", "Frederick", "Martin",
        "Adrian", "Oliver", "Leon", "Marcus", "Elias",
        "Joseph", "Andrew", "Louis", "Peter", "Theodore",
        "Isabella", "Victoria", "Elizabeth", "Margaret", "Anna",
        "Maria", "Charlotte", "Amelia", "Sophia", "Catherine"
    };

    private readonly string[] lastNames =
    {
        "Anderson", "Bennett", "Carter", "Dawson", "Evans",
        "Foster", "Grayson", "Harris", "Ivanov", "Johnson",
        "Keller", "Lawrence", "Miller", "Novak", "Parker",
        "Roberts", "Stewart", "Turner", "Walker", "Young",
        "Petrov", "Morgan", "Reed", "Collins", "Brooks",
        "Hayes", "Ward", "Bishop", "Fletcher", "Russell"
    };

    private readonly string[] ideologies =
    {
        "Democratic", "Conservative", "Nationalist",
        "Socialist", "Liberal", "Reformist",
        "Authoritarian", "Centrist", "Progressive"
    };

    private readonly string[] peacefulDecisions =
    {
        "announced a national reform",
        "introduced a new education program",
        "announced an economic development plan",
        "promised to improve national infrastructure",
        "called for greater political stability",
        "announced a modernization program",
        "proposed new diplomatic talks",
        "announced a national investment initiative",
        "called for stronger international relations",
        "announced a government reorganization",
        "introduced a new public development policy",
        "promised to strengthen the national economy",
        "announced a military review",
        "called for a period of national unity",
        "proposed a new foreign policy"
    };

    private void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }

        Instance = this;

        CreateSprites();

        Shader shader = Shader.Find("Sprites/Default");

        if (shader != null)
            lineMaterial = new Material(shader);

        AddEvent("Transit & Presidents initialized.");

        RefreshCountries();
    }

    private void Update()
    {
        FindCamera();

        Keyboard keyboard = Keyboard.current;
        Mouse mouse = Mouse.current;

        if (keyboard != null)
        {
            if (keyboard.eKey.wasPressedThisFrame && drawingLine != null)
                FinishLine();

            if (keyboard.escapeKey.wasPressedThisFrame && drawingLine != null)
                FinishLine();

            if (keyboard.f1Key.wasPressedThisFrame)
                showWindow = !showWindow;
        }

        if (mouse != null && drawingLine != null &&
            mouse.leftButton.wasPressedThisFrame)
        {
            Vector2 position = mouse.position.ReadValue();

            if (!IsMouseOverWindow(position))
                AddStation(MouseWorldPosition(position));
        }

        UpdateTrains();
        UpdatePreview();
        UpdateMapLabels();

        countryRefreshTimer += Time.deltaTime;

        if (countryRefreshTimer >= 5f)
        {
            countryRefreshTimer = 0f;
            RefreshCountries();
        }

        UpdatePresidents();
    }

    // =========================================================
    // SUBWAY SYSTEM
    // =========================================================

    private void FindCamera()
    {
        if (mapCamera != null && mapCamera.isActiveAndEnabled)
            return;

        mapCamera = Camera.main;

        if (mapCamera == null)
        {
            foreach (Camera cam in Camera.allCameras)
            {
                if (cam != null && cam.isActiveAndEnabled)
                {
                    mapCamera = cam;

                    if (cam.orthographic)
                        break;
                }
            }
        }
    }

    private Vector3 MouseWorldPosition(Vector2 screenPosition)
    {
        if (mapCamera == null)
            return Vector3.zero;

        float depth = Mathf.Abs(mapCamera.transform.position.z);

        Vector3 position = mapCamera.ScreenToWorldPoint(
            new Vector3(screenPosition.x, screenPosition.y, depth)
        );

        position.z = 0f;
        return position;
    }

    private void StartNewLine()
    {
        if (drawingLine != null)
            FinishLine();

        SubwayLine line = new SubwayLine();

        line.name = string.IsNullOrWhiteSpace(lineNameInput)
            ? "New Line"
            : lineNameInput;

        line.label = string.IsNullOrWhiteSpace(lineLabelInput)
            ? "A"
            : lineLabelInput;

        line.color = colors[selectedColor];

        GameObject trackObject = new GameObject("SubwayTrack_" + line.name);

        line.track = trackObject.AddComponent<LineRenderer>();
        line.track.material = lineMaterial;
        line.track.startColor = line.color;
        line.track.endColor = line.color;
        line.track.startWidth = 0.055f;
        line.track.endWidth = 0.055f;
        line.track.useWorldSpace = true;
        line.track.sortingOrder = 20;
        line.track.numCapVertices = 6;
        line.track.numCornerVertices = 6;

        lines.Add(line);

        selectedLine = lines.Count - 1;
        drawingLine = line;

        if (Mouse.current != null)
        {
            AddStation(MouseWorldPosition(
                Mouse.current.position.ReadValue()
            ));
        }

        AddEvent("Subway line " + line.name + " is being constructed.");
    }

    private void EditSelectedLine()
    {
        if (selectedLine < 0 || selectedLine >= lines.Count)
            return;

        SubwayLine line = lines[selectedLine];

        if (line.finished)
            return;

        drawingLine = line;
    }

    private void AddStation(Vector3 position)
    {
        if (drawingLine == null)
            return;

        if (drawingLine.stations.Count > 0)
        {
            Vector3 previous = drawingLine.stations[
                drawingLine.stations.Count - 1
            ];

            if (Vector3.Distance(previous, position) < 0.15f)
                return;
        }

        drawingLine.stations.Add(position);

        GameObject station = new GameObject("SubwayStation");

        station.transform.position = new Vector3(
            position.x,
            position.y,
            -0.1f
        );

        SpriteRenderer renderer = station.AddComponent<SpriteRenderer>();
        renderer.sprite = stationSprite;
        renderer.color = Color.white;
        renderer.sortingOrder = 30;

        station.transform.localScale = Vector3.one * 0.22f;

        drawingLine.stationObjects.Add(station);

        RebuildTrack(drawingLine);
    }

    private void FinishLine()
    {
        if (drawingLine == null)
            return;

        SubwayLine line = drawingLine;
        drawingLine = null;

        if (line.stations.Count < 2)
        {
            AddEvent("Line " + line.name + " was cancelled: not enough stations.");

            lines.Remove(line);

            foreach (GameObject station in line.stationObjects)
                Destroy(station);

            if (line.track != null)
                Destroy(line.track.gameObject);

            selectedLine = -1;
            return;
        }

        line.finished = true;

        RebuildTrack(line);
        CreateTrain(line);
        CreateMapLabel(line);

        AddEvent("Subway line " + line.name + " opened.");
    }

    private void RebuildTrack(SubwayLine line)
    {
        if (line.stations.Count < 2)
        {
            line.track.positionCount = line.stations.Count;

            if (line.stations.Count == 1)
                line.track.SetPosition(0, line.stations[0]);

            return;
        }

        List<Vector3> smoothPath = new List<Vector3>();

        const int subdivisions = 12;

        for (int i = 0; i < line.stations.Count - 1; i++)
        {
            Vector3 p0 = line.stations[Mathf.Max(i - 1, 0)];
            Vector3 p1 = line.stations[i];
            Vector3 p2 = line.stations[i + 1];
            Vector3 p3 = line.stations[
                Mathf.Min(i + 2, line.stations.Count - 1)
            ];

            for (int j = 0; j < subdivisions; j++)
            {
                float t = j / (float)subdivisions;
                Vector3 point = CatmullRom(p0, p1, p2, p3, t);

                smoothPath.Add(new Vector3(point.x, point.y, -0.05f));
            }
        }

        Vector3 last = line.stations[line.stations.Count - 1];

        smoothPath.Add(new Vector3(last.x, last.y, -0.05f));

        line.path = smoothPath;
        line.distances.Clear();
        line.stationDistances.Clear();

        float totalDistance = 0f;
        line.distances.Add(0f);

        for (int i = 1; i < smoothPath.Count; i++)
        {
            totalDistance += Vector3.Distance(
                smoothPath[i - 1],
                smoothPath[i]
            );

            line.distances.Add(totalDistance);
        }

        foreach (Vector3 station in line.stations)
        {
            float bestDistance = 0f;
            float bestDifference = float.MaxValue;

            for (int i = 0; i < smoothPath.Count; i++)
            {
                float difference = Vector3.Distance(
                    station,
                    smoothPath[i]
                );

                if (difference < bestDifference)
                {
                    bestDifference = difference;
                    bestDistance = line.distances[i];
                }
            }

            line.stationDistances.Add(bestDistance);
        }

        line.track.positionCount = smoothPath.Count;
        line.track.SetPositions(smoothPath.ToArray());
    }

    private Vector3 CatmullRom(
        Vector3 p0,
        Vector3 p1,
        Vector3 p2,
        Vector3 p3,
        float t
    )
    {
        float t2 = t * t;
        float t3 = t2 * t;

        return 0.5f * (
            (2f * p1) +
            (-p0 + p2) * t +
            (2f * p0 - 5f * p1 + 4f * p2 - p3) * t2 +
            (-p0 + 3f * p1 - 3f * p2 + p3) * t3
        );
    }

    private void CreateTrain(SubwayLine line)
    {
        GameObject train = new GameObject("SubwayTrain");

        SpriteRenderer renderer = train.AddComponent<SpriteRenderer>();
        renderer.sprite = trainSprite;
        renderer.color = line.color;
        renderer.sortingOrder = 40;

        train.transform.localScale = new Vector3(0.35f, 0.18f, 1f);

        line.train = train;
        line.trainDistance = 0f;
        line.nextStation = 1;
        line.direction = 1;

        UpdateTrainPosition(line);
    }

    private void CreateMapLabel(SubwayLine line)
    {
        line.labelObject = new GameObject("SubwayLineLabel");

        line.mapLabel = line.labelObject.AddComponent<TextMesh>();

        line.mapLabel.text = line.label + " - " + line.name;
        line.mapLabel.color = line.color;
        line.mapLabel.fontSize = 35;
        line.mapLabel.characterSize = 0.12f;
        line.mapLabel.anchor = TextAnchor.MiddleCenter;
        line.mapLabel.alignment = TextAlignment.Center;

        line.labelObject.transform.position = GetLineMidpoint(line);
    }

    private Vector3 GetLineMidpoint(SubwayLine line)
    {
        if (line.path.Count == 0)
            return Vector3.zero;

        return line.path[line.path.Count / 2] + new Vector3(0f, 0.2f, -0.2f);
    }

    private void UpdateMapLabels()
    {
        foreach (SubwayLine line in lines)
        {
            if (line.mapLabel == null)
                continue;

            line.mapLabel.text = line.label + " - " + line.name;
            line.mapLabel.color = line.color;
            line.labelObject.transform.position = GetLineMidpoint(line);
        }
    }

    private void UpdateTrains()
    {
        foreach (SubwayLine line in lines)
        {
            if (!line.finished || line.train == null)
                continue;

            if (line.path.Count < 2 || line.stationDistances.Count < 2)
                continue;

            if (line.waitTimer > 0f)
            {
                line.waitTimer -= Time.deltaTime;
                continue;
            }

            float targetDistance = line.stationDistances[line.nextStation];

            float movement = 0.8f * Time.deltaTime;

            if (line.direction < 0)
                movement = -movement;

            line.trainDistance += movement;

            bool reachedStation = line.direction > 0
                ? line.trainDistance >= targetDistance
                : line.trainDistance <= targetDistance;

            if (reachedStation)
            {
                line.trainDistance = targetDistance;
                line.waitTimer = 1.5f;

                if (line.direction > 0)
                {
                    if (line.nextStation >= line.stationDistances.Count - 1)
                    {
                        line.direction = -1;
                        line.nextStation--;
                    }
                    else
                    {
                        line.nextStation++;
                    }
                }
                else
                {
                    if (line.nextStation <= 0)
                    {
                        line.direction = 1;
                        line.nextStation++;
                    }
                    else
                    {
                        line.nextStation--;
                    }
                }
            }

            UpdateTrainPosition(line);
        }
    }

    private void UpdateTrainPosition(SubwayLine line)
    {
        if (line.path.Count < 2)
            return;

        float distance = Mathf.Clamp(
            line.trainDistance,
            0f,
            line.distances[line.distances.Count - 1]
        );

        int index = 0;

        while (
            index < line.distances.Count - 2 &&
            line.distances[index + 1] < distance
        )
        {
            index++;
        }

        float segmentLength =
            line.distances[index + 1] - line.distances[index];

        float t = segmentLength <= 0f
            ? 0f
            : (distance - line.distances[index]) / segmentLength;

        Vector3 a = line.path[index];
        Vector3 b = line.path[index + 1];

        line.train.transform.position = Vector3.Lerp(a, b, t);

        Vector3 direction = b - a;

        float angle = Mathf.Atan2(direction.y, direction.x) * Mathf.Rad2Deg;

        line.train.transform.rotation = Quaternion.Euler(0f, 0f, angle);
    }

    private void UpdatePreview()
    {
        if (drawingLine == null || drawingLine.stations.Count == 0)
            return;

        if (Mouse.current == null)
            return;

        Vector3 mousePosition = MouseWorldPosition(
            Mouse.current.position.ReadValue()
        );

        List<Vector3> preview = new List<Vector3>(drawingLine.stations);
        preview.Add(mousePosition);

        drawingLine.track.positionCount = preview.Count;
        drawingLine.track.SetPositions(preview.ToArray());
    }

    private void DeleteSelectedLine()
    {
        if (selectedLine < 0 || selectedLine >= lines.Count)
            return;

        SubwayLine line = lines[selectedLine];

        foreach (GameObject station in line.stationObjects)
            if (station != null)
                Destroy(station);

        if (line.track != null)
            Destroy(line.track.gameObject);

        if (line.train != null)
            Destroy(line.train);

        if (line.labelObject != null)
            Destroy(line.labelObject);

        lines.RemoveAt(selectedLine);

        selectedLine = -1;
        drawingLine = null;

        AddEvent("A subway line was removed.");
    }

    // =========================================================
    // PRESIDENT SYSTEM
    // =========================================================

    private void RefreshCountries()
    {
        try
        {
            FieldInfo field = typeof(Events).GetField(
                "allNations",
                BindingFlags.Public |
                BindingFlags.NonPublic |
                BindingFlags.Static
            );

            if (field == null)
                return;

            object source = field.GetValue(null);

            IEnumerable collection = source as IEnumerable;

            if (collection == null)
                return;

            int index = 0;

            foreach (object item in collection)
            {
                if (item == null)
                    continue;

                object nation = item;

                PropertyInfo valueProperty = item.GetType().GetProperty("Value");

                if (valueProperty != null)
                    nation = valueProperty.GetValue(item, null);

                if (nation == null)
                    continue;

                President existing = FindPresident(nation);

                if (existing == null)
                {
                    President president = GeneratePresident(nation, index);

                    presidents.Add(president);

                    AddEvent(
                        president.fullName +
                        " became president of " +
                        president.countryName + "."
                    );
                }
                else
                {
                    existing.nationIndex = index;
                    existing.countryName = GetNationName(nation);
                }

                index++;
            }
        }
        catch (Exception ex)
        {
            Debug.LogError("[Presidents] Country refresh failed: " + ex);
        }
    }

    private President FindPresident(object nation)
    {
        foreach (President president in presidents)
        {
            if (ReferenceEquals(president.nation, nation))
                return president;
        }

        return null;
    }

    private President GeneratePresident(object nation, int index)
    {
        President president = new President();

        president.nation = nation;
        president.nationIndex = index;
        president.countryName = GetNationName(nation);

        president.firstName = firstNames[
            UnityEngine.Random.Range(0, firstNames.Length)
        ];

        president.lastName = lastNames[
            UnityEngine.Random.Range(0, lastNames.Length)
        ];

        president.fullName =
            president.firstName + " " + president.lastName;

        president.ideology = ideologies[
            UnityEngine.Random.Range(0, ideologies.Length)
        ];

        president.stability = UnityEngine.Random.Range(40f, 95f);
        president.popularity = UnityEngine.Random.Range(35f, 90f);
        president.economy = UnityEngine.Random.Range(30f, 90f);
        president.military = UnityEngine.Random.Range(30f, 95f);

        president.age = UnityEngine.Random.Range(35f, 75f);
        president.decisionTimer = UnityEngine.Random.Range(15f, 35f);

        president.portrait = GeneratePortrait();

        return president;
    }

    private string GetNationName(object nation)
    {
        string[] possibleNames =
        {
            "name", "nationName", "Name", "NationName"
        };

        foreach (string memberName in possibleNames)
        {
            object value = GetMember(nation, memberName);

            if (value != null)
                return value.ToString();
        }

        return "Country";
    }

    private object GetMember(object instance, string name)
    {
        if (instance == null)
            return null;

        Type type = instance.GetType();

        FieldInfo field = type.GetField(
            name,
            BindingFlags.Public |
            BindingFlags.NonPublic |
            BindingFlags.Instance
        );

        if (field != null)
            return field.GetValue(instance);

        PropertyInfo property = type.GetProperty(
            name,
            BindingFlags.Public |
            BindingFlags.NonPublic |
            BindingFlags.Instance
        );

        if (property != null)
            return property.GetValue(instance, null);

        return null;
    }

    private bool SetMember(object instance, string name, object value)
    {
        if (instance == null)
            return false;

        Type type = instance.GetType();

        FieldInfo field = type.GetField(
            name,
            BindingFlags.Public |
            BindingFlags.NonPublic |
            BindingFlags.Instance
        );

        if (field != null)
        {
            try
            {
                field.SetValue(instance, Convert.ChangeType(value, field.FieldType));
                return true;
            }
            catch { }
        }

        PropertyInfo property = type.GetProperty(
            name,
            BindingFlags.Public |
            BindingFlags.NonPublic |
            BindingFlags.Instance
        );

        if (property != null && property.CanWrite)
        {
            try
            {
                property.SetValue(
                    instance,
                    Convert.ChangeType(value, property.PropertyType),
                    null
                );

                return true;
            }
            catch { }
        }

        return false;
    }

    private void UpdatePresidents()
    {
        foreach (President president in presidents)
        {
            president.age += Time.deltaTime / 3600f;

            president.decisionTimer -= Time.deltaTime;

            if (president.decisionTimer <= 0f)
            {
                MakeAutomaticDecision(president);

                president.decisionTimer = UnityEngine.Random.Range(25f, 55f);
            }

            if (president.age > 85f)
                ReplacePresident(president);
        }
    }

    private void ReplacePresident(President president)
    {
        string oldName = president.fullName;

        if (president.portrait != null)
            Destroy(president.portrait);

        President replacement = GeneratePresident(
            president.nation,
            president.nationIndex
        );

        president.firstName = replacement.firstName;
        president.lastName = replacement.lastName;
        president.fullName = replacement.fullName;
        president.ideology = replacement.ideology;
        president.portrait = replacement.portrait;
        president.age = replacement.age;
        president.stability = replacement.stability;
        president.popularity = replacement.popularity;
        president.economy = replacement.economy;
        president.military = replacement.military;

        AddEvent(
            oldName + " left office. " +
            president.fullName + " became the new president of " +
            president.countryName + "."
        );
    }

    private void MakeAutomaticDecision(President president)
    {
        int decision = UnityEngine.Random.Range(0, 10);

        if (decision == 0)
        {
            president.stability = Mathf.Clamp(
                president.stability + UnityEngine.Random.Range(2f, 8f),
                0f, 100f
            );

            president.lastDecision = "introduced political reforms";

            AddEvent(
                president.fullName + " of " +
                president.countryName + " introduced political reforms."
            );
        }
        else if (decision == 1)
        {
            president.economy = Mathf.Clamp(
                president.economy + UnityEngine.Random.Range(2f, 7f),
                0f, 100f
            );

            president.lastDecision = "announced economic development";

            AddEvent(
                president.fullName + " of " +
                president.countryName + " announced an economic development plan."
            );
        }
        else if (decision == 2)
        {
            president.military = Mathf.Clamp(
                president.military + UnityEngine.Random.Range(1f, 5f),
                0f, 100f
            );

            president.lastDecision = "ordered a military review";

            AddEvent(
                president.fullName + " of " +
                president.countryName + " ordered a military review."
            );
        }
        else if (decision == 3)
        {
            president.popularity = Mathf.Clamp(
                president.popularity + UnityEngine.Random.Range(1f, 6f),
                0f, 100f
            );

            president.lastDecision = "announced a public initiative";

            AddEvent(
                president.fullName + " of " +
                president.countryName + " announced a public initiative."
            );
        }
        else if (decision == 4)
        {
            President target = GetRandomOtherPresident(president);

            if (target != null)
            {
                president.lastDecision = "proposed diplomatic talks";

                AddEvent(
                    president.fullName + " of " +
                    president.countryName +
                    " proposed diplomatic talks with " +
                    target.countryName + "."
                );
            }
        }
        else if (decision == 5)
        {
            president.lastDecision = "called for national unity";

            AddEvent(
                president.fullName + " of " +
                president.countryName + " called for national unity."
            );
        }
        else if (decision == 6)
        {
            president.lastDecision = "announced infrastructure investment";

            AddEvent(
                president.fullName + " of " +
                president.countryName +
                " announced an infrastructure investment program."
            );
        }
        else if (decision == 7)
        {
            president.lastDecision = "reorganized the government";

            AddEvent(
                president.fullName + " of " +
                president.countryName + " reorganized the government."
            );
        }
        else if (decision == 8)
        {
            President target = GetRandomOtherPresident(president);

            if (target != null)
            {
                president.lastDecision = "considered military action";

                AddEvent(
                    president.fullName + " of " +
                    president.countryName +
                    " considered military action involving " +
                    target.countryName + "."
                );
            }
        }
        else
        {
            string action = peacefulDecisions[
                UnityEngine.Random.Range(0, peacefulDecisions.Length)
            ];

            president.lastDecision = action;

            AddEvent(
                president.fullName + " of " +
                president.countryName + " " + action + "."
            );
        }
    }

    private President GetRandomOtherPresident(President current)
    {
        List<President> possible = new List<President>();

        foreach (President president in presidents)
        {
            if (president != current)
                possible.Add(president);
        }

        if (possible.Count == 0)
            return null;

        return possible[
            UnityEngine.Random.Range(0, possible.Count)
        ];
    }

    // =========================================================
    // REAL GAME WAR METHOD
    // =========================================================

    private bool DeclareWar(President attacker, President target)
    {
        if (attacker == null || target == null || attacker == target)
            return false;

        try
        {
            MethodInfo[] methods = typeof(Events).GetMethods(
                BindingFlags.Public |
                BindingFlags.NonPublic |
                BindingFlags.Static
            );

            foreach (MethodInfo method in methods)
            {
                if (method.Name != "TryWar")
                    continue;

                ParameterInfo[] parameters = method.GetParameters();
                object[] arguments = new object[parameters.Length];

                int nationArgument = 0;
                bool usable = true;

                for (int i = 0; i < parameters.Length; i++)
                {
                    Type parameterType = parameters[i].ParameterType;
                    string parameterName = parameters[i].Name.ToLowerInvariant();

                    if (parameterType == typeof(bool))
                    {
                        arguments[i] = true;
                    }
                    else if (parameterType == typeof(int))
                    {
                        if (parameterName.Contains("defend") ||
                            parameterName.Contains("target") ||
                            parameterName.Contains("enemy"))
                        {
                            arguments[i] = target.nationIndex;
                        }
                        else
                        {
                            arguments[i] = nationArgument == 0
                                ? attacker.nationIndex
                                : target.nationIndex;

                            nationArgument++;
                        }
                    }
                    else if (parameterType.Name == "Nation")
                    {
                        arguments[i] = nationArgument == 0
                            ? attacker.nation
                            : target.nation;

                        nationArgument++;
                    }
                    else if (parameterType == typeof(float))
                    {
                        arguments[i] = 0f;
                    }
                    else if (parameterType == typeof(double))
                    {
                        arguments[i] = 0d;
                    }
                    else if (parameterType == typeof(string))
                    {
                        arguments[i] = "";
                    }
                    else if (parameterType.IsEnum)
                    {
                        arguments[i] = Enum.GetValues(parameterType).GetValue(0);
                    }
                    else if (parameters[i].IsOptional)
                    {
                        arguments[i] = parameters[i].DefaultValue;
                    }
                    else
                    {
                        usable = false;
                        break;
                    }
                }

                if (!usable || nationArgument == 0)
                    continue;

                method.Invoke(null, arguments);

                AddEvent(
                    attacker.fullName + " of " +
                    attacker.countryName + " declared war on " +
                    target.countryName + "."
                );

                attacker.lastDecision = "declared war on " + target.countryName;

                return true;
            }
        }
        catch (Exception ex)
        {
            Debug.LogError("[Presidents] War declaration failed: " + ex);
        }

        AddEvent(
            "The game did not accept the war declaration from " +
            attacker.fullName + "."
        );

        return false;
    }

    private void ManualDecision(string decision)
    {
        if (selectedPresident == null)
            return;

        President president = selectedPresident;

        if (decision == "war")
        {
            if (selectedTarget < 0 || selectedTarget >= presidents.Count)
                return;

            President target = presidents[selectedTarget];

            if (target == president)
                return;

            DeclareWar(president, target);
        }
        else if (decision == "economy")
        {
            president.economy = Mathf.Clamp(
                president.economy + 10f, 0f, 100f
            );

            president.lastDecision = "ordered an economic development program";

            AddEvent(
                president.fullName + " ordered an economic development program."
            );

            object goldValue = GetMember(president.nation, "goldf");

            if (goldValue != null)
            {
                float gold = Convert.ToSingle(goldValue);
                SetMember(president.nation, "goldf", gold + 10f);
            }
        }
        else if (decision == "military")
        {
            president.military = Mathf.Clamp(
                president.military + 10f, 0f, 100f
            );

            president.lastDecision = "ordered a military review";

            AddEvent(
                president.fullName + " ordered a military review."
            );
        }
        else if (decision == "reform")
        {
            president.stability = Mathf.Clamp(
                president.stability + 10f, 0f, 100f
            );

            president.lastDecision = "introduced national reforms";

            AddEvent(
                president.fullName + " introduced national reforms."
            );
        }
        else if (decision == "diplomacy")
        {
            President target = GetRandomOtherPresident(president);

            if (target == null)
                return;

            president.lastDecision = "requested diplomatic talks with " +
                                     target.countryName;

            AddEvent(
                president.fullName + " requested diplomatic talks with " +
                target.countryName + "."
            );
        }
    }

    // =========================================================
    // GENERATED PRESIDENT PORTRAITS
    // =========================================================

    private Texture2D GeneratePortrait()
    {
        const int size = 64;

        Texture2D texture = new Texture2D(size, size);

        Color background = new Color(
            UnityEngine.Random.Range(0.15f, 0.5f),
            UnityEngine.Random.Range(0.15f, 0.5f),
            UnityEngine.Random.Range(0.15f, 0.5f)
        );

        Color skin = new Color(
            UnityEngine.Random.Range(0.55f, 0.95f),
            UnityEngine.Random.Range(0.35f, 0.8f),
            UnityEngine.Random.Range(0.25f, 0.65f)
        );

        Color hair = new Color(
            UnityEngine.Random.Range(0.05f, 0.4f),
            UnityEngine.Random.Range(0.03f, 0.3f),
            UnityEngine.Random.Range(0.02f, 0.2f)
        );

        Color suit = new Color(
            UnityEngine.Random.Range(0.03f, 0.25f),
            UnityEngine.Random.Range(0.03f, 0.25f),
            UnityEngine.Random.Range(0.08f, 0.35f)
        );

        Color eye = Color.black;

        for (int y = 0; y < size; y++)
        {
            for (int x = 0; x < size; x++)
            {
                Color pixel = background;

                if (y < 20)
                    pixel = suit;

                if (x >= 25 && x <= 39 && y >= 17 && y <= 28)
                    pixel = skin;

                bool face =
                    x >= 16 && x <= 47 &&
                    y >= 23 && y <= 53;

                if (face)
                    pixel = skin;

                if (x >= 14 && x <= 49 && y >= 45 && y <= 57)
                    pixel = hair;

                if (x >= 14 && x <= 20 && y >= 33 && y <= 52)
                    pixel = hair;

                if (x >= 43 && x <= 49 && y >= 33 && y <= 52)
                    pixel = hair;

                if ((x >= 23 && x <= 27 || x >= 36 && x <= 40) &&
                    y >= 39 && y <= 42)
                    pixel = eye;

                if (x >= 28 && x <= 35 && y >= 29 && y <= 30)
                    pixel = new Color(0.35f, 0.08f, 0.08f);

                if (x >= 27 && x <= 36 && y >= 0 && y <= 21)
                    pixel = Color.white;

                texture.SetPixel(x, y, pixel);
            }
        }

        texture.filterMode = FilterMode.Point;
        texture.Apply();

        return texture;
    }

    private void CreateSprites()
    {
        Texture2D stationTexture = new Texture2D(32, 32);
        Texture2D trainTexture = new Texture2D(32, 32);

        for (int y = 0; y < 32; y++)
        {
            for (int x = 0; x < 32; x++)
            {
                float dx = x - 15.5f;
                float dy = y - 15.5f;

                bool circle = dx * dx + dy * dy < 220f;

                stationTexture.SetPixel(
                    x, y, circle ? Color.white : Color.clear
                );

                bool train =
                    x >= 3 && x <= 28 &&
                    y >= 8 && y <= 23;

                trainTexture.SetPixel(
                    x, y, train ? Color.white : Color.clear
                );
            }
        }

        stationTexture.filterMode = FilterMode.Bilinear;
        trainTexture.filterMode = FilterMode.Bilinear;

        stationTexture.Apply();
        trainTexture.Apply();

        stationSprite = Sprite.Create(
            stationTexture,
            new Rect(0, 0, 32, 32),
            new Vector2(0.5f, 0.5f),
            32f
        );

        trainSprite = Sprite.Create(
            trainTexture,
            new Rect(0, 0, 32, 32),
            new Vector2(0.5f, 0.5f),
            32f
        );
    }

    // =========================================================
    // EVENT FEED
    // =========================================================

    private void AddEvent(string message)
    {
        string entry = DateTime.Now.ToString("HH:mm:ss") + "  " + message;

        events.Insert(0, entry);

        if (events.Count > 100)
            events.RemoveAt(events.Count - 1);

        Debug.Log("[Transit & Presidents] " + message);
    }

    // =========================================================
    // USER INTERFACE
    // =========================================================

    private bool IsMouseOverWindow(Vector2 mousePosition)
    {
        Vector2 guiPosition = new Vector2(
            mousePosition.x,
            Screen.height - mousePosition.y
        );

        return showWindow && windowRect.Contains(guiPosition);
    }

    private void OnGUI()
    {
        if (!showWindow)
            return;

        windowRect = GUI.Window(
            9281,
            windowRect,
            DrawWindow,
            "TRANSIT & PRESIDENTS"
        );
    }

    private void DrawWindow(int id)
    {
        GUILayout.BeginHorizontal();

        if (GUILayout.Toggle(selectedTab == 0, "SUBWAYS", "Button"))
            selectedTab = 0;

        if (GUILayout.Toggle(selectedTab == 1, "PRESIDENTS", "Button"))
            selectedTab = 1;

        if (GUILayout.Toggle(selectedTab == 2, "EVENTS", "Button"))
            selectedTab = 2;

        GUILayout.EndHorizontal();

        GUILayout.Space(8);

        if (selectedTab == 0)
            DrawSubwayTab();
        else if (selectedTab == 1)
            DrawPresidentTab();
        else
            DrawEventTab();

        GUI.DragWindow(new Rect(0, 0, 10000, 22));
    }

    private void DrawSubwayTab()
    {
        GUILayout.Label("SUBWAY NETWORK");

        GUILayout.Label("Line name:");
        lineNameInput = GUILayout.TextField(lineNameInput);

        GUILayout.Label("Map label:");
        lineLabelInput = GUILayout.TextField(lineLabelInput);

        GUILayout.Space(5);
        GUILayout.Label("Choose line color:");

        GUILayout.BeginHorizontal();

        for (int i = 0; i < colors.Length; i++)
        {
            GUI.backgroundColor = colors[i];

            if (GUILayout.Button(" ", GUILayout.Width(32), GUILayout.Height(25)))
                selectedColor = i;
        }

        GUI.backgroundColor = Color.white;

        GUILayout.EndHorizontal();

        GUI.backgroundColor = colors[selectedColor];

        GUILayout.Label("Selected color");
        GUI.backgroundColor = Color.white;

        if (drawingLine == null)
        {
            if (GUILayout.Button("START NEW LINE", GUILayout.Height(32)))
                StartNewLine();
        }
        else
        {
            GUILayout.Label(
                "Currently drawing: " + drawingLine.name +
                " | Stations: " + drawingLine.stations.Count
            );

            if (GUILayout.Button("FINISH LINE (E)", GUILayout.Height(32)))
                FinishLine();
        }

        GUILayout.Space(8);
        GUILayout.Label("EXISTING LINES");

        lineScroll = GUILayout.BeginScrollView(
            lineScroll,
            GUILayout.Height(150)
        );

        for (int i = 0; i < lines.Count; i++)
        {
            SubwayLine line = lines[i];

            GUI.backgroundColor = line.color;

            if (GUILayout.Button(
                (selectedLine == i ? "> " : "") +
                line.label + " - " + line.name +
                " (" + line.stations.Count + " stations)"
            ))
            {
                selectedLine = i;
                lineNameInput = line.name;
                lineLabelInput = line.label;
                drawingLine = null;
            }

            GUI.backgroundColor = Color.white;
        }

        GUILayout.EndScrollView();

        if (selectedLine >= 0 && selectedLine < lines.Count)
        {
            SubwayLine line = lines[selectedLine];

            GUILayout.Space(5);
            GUILayout.Label("SELECTED LINE");

            lineNameInput = GUILayout.TextField(lineNameInput);
            lineLabelInput = GUILayout.TextField(lineLabelInput);

            if (GUILayout.Button("APPLY NAME AND LABEL"))
            {
                line.name = lineNameInput;
                line.label = lineLabelInput;
            }

            GUILayout.BeginHorizontal();

            if (GUILayout.Button("RECOLOR"))
            {
                line.color = colors[selectedColor];

                if (line.track != null)
                {
                    line.track.startColor = line.color;
                    line.track.endColor = line.color;
                }

                if (line.train != null)
                {
                    SpriteRenderer renderer =
                        line.train.GetComponent<SpriteRenderer>();

                    if (renderer != null)
                        renderer.color = line.color;
                }
            }

            if (GUILayout.Button("DELETE LINE"))
                DeleteSelectedLine();

            GUILayout.EndHorizontal();
        }

        GUILayout.Space(8);

        GUILayout.Label("E finishes a line. F1 hides/shows this window.");
    }

    private void DrawPresidentTab()
    {
        GUILayout.Label("COUNTRY LEADERS");
        GUILayout.Label("Presidents: " + presidents.Count);

        presidentScroll = GUILayout.BeginScrollView(
            presidentScroll,
            GUILayout.Height(180)
        );

        for (int i = 0; i < presidents.Count; i++)
        {
            President president = presidents[i];

            GUILayout.BeginHorizontal();

            if (president.portrait != null)
                GUILayout.Label(
                    president.portrait,
                    GUILayout.Width(42),
                    GUILayout.Height(42)
                );

            if (GUILayout.Button(
                president.fullName + "\n" + president.countryName,
                GUILayout.Height(45)
            ))
            {
                selectedPresident = president;
            }

            GUILayout.EndHorizontal();
        }

        GUILayout.EndScrollView();

        if (selectedPresident == null)
        {
            GUILayout.Label("Select a president to view their profile.");
            return;
        }

        President p = selectedPresident;

        GUILayout.Space(8);
        GUILayout.Label("PRESIDENT PROFILE");

        GUILayout.BeginHorizontal();

        if (p.portrait != null)
            GUILayout.Label(
                p.portrait,
                GUILayout.Width(90),
                GUILayout.Height(90)
            );

        GUILayout.BeginVertical();

        GUILayout.Label(p.fullName);
        GUILayout.Label(p.countryName);
        GUILayout.Label("Age: " + Mathf.RoundToInt(p.age));
        GUILayout.Label("Ideology: " + p.ideology);

        GUILayout.EndVertical();

        GUILayout.EndHorizontal();

        GUILayout.Label("Stability: " + Mathf.RoundToInt(p.stability) + "%");
        GUILayout.Label("Popularity: " + Mathf.RoundToInt(p.popularity) + "%");
        GUILayout.Label("Economy: " + Mathf.RoundToInt(p.economy) + "%");
        GUILayout.Label("Military: " + Mathf.RoundToInt(p.military) + "%");

        GUILayout.Label("Last decision: " + p.lastDecision);

        GUILayout.Space(8);
        GUILayout.Label("MANUAL PRESIDENTIAL DECISIONS");

        if (GUILayout.Button("Economic development"))
            ManualDecision("economy");

        if (GUILayout.Button("Military review"))
            ManualDecision("military");

        if (GUILayout.Button("National reforms"))
            ManualDecision("reform");

        if (GUILayout.Button("Request diplomatic talks"))
            ManualDecision("diplomacy");

        GUILayout.Space(5);
        GUILayout.Label("Choose a country for a war declaration:");

        presidentScroll = GUILayout.BeginScrollView(
            presidentScroll,
            GUILayout.Height(100)
        );

        for (int i = 0; i < presidents.Count; i++)
        {
            if (presidents[i] == p)
                continue;

            if (GUILayout.Button(presidents[i].countryName))
                selectedTarget = i;
        }

        GUILayout.EndScrollView();

        if (selectedTarget >= 0 && selectedTarget < presidents.Count)
        {
            GUILayout.Label(
                "Target: " + presidents[selectedTarget].countryName
            );

            if (GUILayout.Button("DECLARE WAR"))
                ManualDecision("war");
        }
    }

    private void DrawEventTab()
    {
        GUILayout.Label("PRESIDENTIAL AND SUBWAY EVENTS");
        GUILayout.Label("Latest events appear first.");

        eventScroll = GUILayout.BeginScrollView(
            eventScroll,
            GUILayout.Height(460)
        );

        foreach (string entry in events)
            GUILayout.Label(entry);

        GUILayout.EndScrollView();
    }

    private void OnDestroy()
    {
        if (Instance == this)
            Instance = null;

        foreach (President president in presidents)
        {
            if (president.portrait != null)
                Destroy(president.portrait);
        }

        if (lineMaterial != null)
            Destroy(lineMaterial);
    }
}
