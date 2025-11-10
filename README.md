// SingleFileRunner.cs
// Place this single script in Assets/Scripts/ and attachez-le à un GameObject vide dans une scène vide.
// Configurez la caméra en Orthographic (Size ~5) et supprimez tout autre objet.
// Ce script crée la scène, le joueur, les obstacles, l'UI et gère le gameplay "endless runner" en un seul fichier.
// Contrôles : Tap/clic -> saut ; Swipe bas -> slide (détection simple) ; Space/Down pour tester dans l'éditeur.
// Build target: Android/iOS/PC. Compatible Unity 2019+ (API basique).

using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;

public class SingleFileRunner : MonoBehaviour
{
    // --- CONFIG ---
    [Header("Gameplay")]
    public float runSpeed = 6f;
    public float gravity = -30f;
    public float jumpVelocity = 12f;
    public float slideDuration = 0.6f;
    public float spawnIntervalStart = 1.5f;
    public float spawnIntervalMin = 0.6f;
    public float difficultyRamp = 0.995f; // spawnInterval *= ramp each spawn

    [Header("Sizes (world units)")]
    public Vector2 playerSize = new Vector2(0.9f, 1.8f);
    public Vector2 obstacleSize = new Vector2(0.8f, 1.6f);
    public float groundY = -2.5f;

    // --- INTERNAL ---
    GameObject player;
    Rect playerRect; // for AABB collision
    float playerVy = 0f;
    bool isGrounded = false;
    bool isSliding = false;
    float slideTimer = 0f;
    Vector2 playerOriginalSize;
    Vector2 playerSlideSize;

    Camera mainCam;
    Transform obstaclesRoot;
    List<GameObject> obstacles = new List<GameObject>();
    float spawnTimer = 0f;
    float currentSpawnInterval;

    // UI
    Canvas uiCanvas;
    Text scoreText;
    GameObject mainMenuPanel;
    GameObject gameOverPanel;
    Text finalScoreText;
    Button playButton;
    Button retryButton;
    bool isPlaying = false;
    float score = 0f;

    // Touch
    Vector2 touchStart;
    bool touchBegan = false;

    // For aesthetic simple colors
    Color bgColor = new Color(0.07f, 0.07f, 0.08f);
    Color groundColor = new Color(0.12f, 0.12f, 0.14f);
    Color playerColor = new Color(0.9f, 0.35f, 0.2f);
    Color obstacleColor = new Color(0.2f, 0.6f, 0.9f);

    void Awake()
    {
        // If user double-adds script, prevent duplicates content creation by cleaning scene children
        foreach (Transform t in transform) Destroy(t.gameObject);

        SetupCamera();
        BuildEnvironment();
        BuildPlayer();
        BuildUI();
        obstaclesRoot = new GameObject("Obstacles").transform;
        currentSpawnInterval = spawnIntervalStart;
        spawnTimer = currentSpawnInterval;
        ShowMainMenu();
    }

    void SetupCamera()
    {
        mainCam = Camera.main;
        if (mainCam == null)
        {
            GameObject camObj = new GameObject("Main Camera");
            mainCam = camObj.AddComponent<Camera>();
            camObj.tag = "MainCamera";
        }
        mainCam.orthographic = true;
        mainCam.orthographicSize = 5f;
        mainCam.backgroundColor = bgColor;
        mainCam.transform.position = new Vector3(0, 0, -10);
    }

    void BuildEnvironment()
    {
        // Ground (long quad)
        GameObject ground = GameObject.CreatePrimitive(PrimitiveType.Quad);
        ground.name = "Ground";
        ground.transform.parent = transform;
        ground.transform.localScale = new Vector3(40f, 6f, 1f); // wide ground strip
        ground.transform.position = new Vector3(0f, groundY - 1.5f, 0f);
        var gr = ground.GetComponent<MeshRenderer>();
        gr.material = new Material(Shader.Find("Unlit/Color"));
        gr.material.color = groundColor;

        // Decorative horizon blocks for parallax-like feel
        for (int i = -2; i <= 2; i++)
        {
            GameObject hill = GameObject.CreatePrimitive(PrimitiveType.Quad);
            hill.transform.parent = transform;
            hill.transform.localScale = new Vector3(6f, 3f, 1f);
            hill.transform.position = new Vector3(i * 7.5f, groundY + 0.6f, 5f);
            var mr = hill.GetComponent<MeshRenderer>();
            mr.material = new Material(Shader.Find("Unlit/Color"));
            mr.material.color = new Color(0.06f + 0.02f * i, 0.06f, 0.08f);
        }
    }

    void BuildPlayer()
    {
        player = GameObject.CreatePrimitive(PrimitiveType.Cube);
        player.name = "Player";
        player.transform.parent = transform;
        player.transform.localScale = new Vector3(playerSize.x, playerSize.y, 1f);
        player.transform.position = new Vector3(-4f, groundY + playerSize.y * 0.5f, 0f);
        var mr = player.GetComponent<MeshRenderer>();
        mr.material = new Material(Shader.Find("Unlit/Color"));
        mr.material.color = playerColor;

        playerOriginalSize = playerSize;
        playerSlideSize = new Vector2(playerSize.x, playerSize.y * 0.5f);

        // Collider is not used; we'll do AABB manually via bounds
    }

    void BuildUI()
    {
        // Canvas
        GameObject cObj = new GameObject("UICanvas");
        uiCanvas = cObj.AddComponent<Canvas>();
        uiCanvas.renderMode = RenderMode.ScreenSpaceOverlay;
        cObj.AddComponent<CanvasScaler>().uiScaleMode = CanvasScaler.ScaleMode.ScaleWithScreenSize;
        cObj.GetComponent<CanvasScaler>().referenceResolution = new Vector2(1080, 1920);
        cObj.AddComponent<GraphicRaycaster>();

        // Score text (top-left)
        GameObject scoreGO = new GameObject("ScoreText");
        scoreGO.transform.SetParent(cObj.transform, false);
        scoreText = scoreGO.AddComponent<Text>();
        scoreText.font = Resources.GetBuiltinResource<Font>("Arial.ttf");
        scoreText.fontSize = 48;
        scoreText.alignment = TextAnchor.UpperLeft;
        scoreText.rectTransform.anchorMin = new Vector2(0, 1);
        scoreText.rectTransform.anchorMax = new Vector2(0, 1);
        scoreText.rectTransform.pivot = new Vector2(0, 1);
        scoreText.rectTransform.anchoredPosition = new Vector2(20, -20);
        scoreText.text = "0";

        // Main Menu Panel
        mainMenuPanel = CreatePanel(cObj.transform, "MainMenu", new Vector2(0.5f, 0.5f), new Vector2(600, 400));
        CreateText(mainMenuPanel.transform, "Title", "ENDLESS RUNNER", 44, new Vector2(0, 100));
        playButton = CreateButton(mainMenuPanel.transform, "Play", "Jouer", new Vector2(0, -20), 180, 80);
        playButton.onClick.AddListener(OnPlayClicked);
        CreateText(mainMenuPanel.transform, "Sub", "Tap pour sauter - Swipe bas pour glisser", 20, new Vector2(0, -80));

        // GameOver Panel
        gameOverPanel = CreatePanel(cObj.transform, "GameOver", new Vector2(0.5f, 0.5f), new Vector2(640, 420));
        CreateText(gameOverPanel.transform, "GOT", "Game Over", 46, new Vector2(0, 100));
        finalScoreText = CreateText(gameOverPanel.transform, "FinalScore", "Score: 0", 30, new Vector2(0, 20));
        retryButton = CreateButton(gameOverPanel.transform, "Retry", "Rejouer", new Vector2(0, -110), 200, 90);
        retryButton.onClick.AddListener(OnRetryClicked);
    }

    GameObject CreatePanel(Transform parent, string name, Vector2 anchorCenter, Vector2 size)
    {
        GameObject panel = new GameObject(name);
        panel.transform.SetParent(parent, false);
        Image img = panel.AddComponent<Image>();
        img.color = new Color(0f, 0f, 0f, 0.6f);
        RectTransform rt = panel.GetComponent<RectTransform>();
        rt.sizeDelta = size;
        rt.anchorMin = anchorCenter;
        rt.anchorMax = anchorCenter;
        rt.anchoredPosition = Vector2.zero;
        return panel;
    }

    Text CreateText(Transform parent, string name, string content, int size, Vector2 anchoredPos)
    {
        GameObject go = new GameObject(name);
        go.transform.SetParent(parent, false);
        Text t = go.AddComponent<Text>();
        t.font = Resources.GetBuiltinResource<Font>("Arial.ttf");
        t.text = content;
        t.fontSize = size;
        t.alignment = TextAnchor.MiddleCenter;
        RectTransform rt = t.GetComponent<RectTransform>();
        rt.anchoredPosition = anchoredPos;
        rt.sizeDelta = new Vector2(800, 120);
        return t;
    }

    Button CreateButton(Transform parent, string name, string label, Vector2 anchoredPos, float w, float h)
    {
        GameObject go = new GameObject(name);
        go.transform.SetParent(parent, false);
        Image img = go.AddComponent<Image>();
        img.color = new Color(0.95f, 0.82f, 0.2f, 1f);
        RectTransform rt = img.GetComponent<RectTransform>();
        rt.anchoredPosition = anchoredPos;
        rt.sizeDelta = new Vector2(w, h);

        GameObject textGO = new GameObject("Label");
        textGO.transform.SetParent(go.transform, false);
        Text t = textGO.AddComponent<Text>();
        t.font = Resources.GetBuiltinResource<Font>("Arial.ttf");
        t.text = label;
        t.fontSize = 28;
        t.alignment = TextAnchor.MiddleCenter;
        RectTransform trt = t.GetComponent<RectTransform>();
        trt.anchorMin = Vector2.zero;
        trt.anchorMax = Vector2.one;
        trt.offsetMin = Vector2.zero;
        trt.offsetMax = Vector2.zero;

        Button b = go.AddComponent<Button>();
        return b;
    }

    void ShowMainMenu()
    {
        mainMenuPanel.SetActive(true);
        gameOverPanel.SetActive(false);
        isPlaying = false;
        scoreText.gameObject.SetActive(false);
    }

    void StartGame()
    {
        // reset state
        ClearObstacles();
        isPlaying = true;
        mainMenuPanel.SetActive(false);
        gameOverPanel.SetActive(false);
        scoreText.gameObject.SetActive(true);
        score = 0f;
        currentSpawnInterval = spawnIntervalStart;
        spawnTimer = currentSpawnInterval;
        player.transform.position = new Vector3(-4f, groundY + playerOriginalSize.y * 0.5f, 0f);
        player.transform.localScale = new Vector3(playerOriginalSize.x, playerOriginalSize.y, 1f);
        isSliding = false;
        slideTimer = 0f;
        playerVy = 0f;
        isGrounded = true;
    }

    void GameOver()
    {
        isPlaying = false;
        gameOverPanel.SetActive(true);
        finalScoreText.text = "Score : " + Mathf.FloorToInt(score).ToString();
        scoreText.gameObject.SetActive(false);
    }

    void Update()
    {
        if (!isPlaying) return;

        HandleInput();

        // Update player physics (simple)
        playerVy += gravity * Time.deltaTime;
        Vector3 pos = player.transform.position;
        pos.y += playerVy * Time.deltaTime;

        // Ground collision
        float halfHeight = (player.transform.localScale.y) * 0.5f;
        if (pos.y - halfHeight <= groundY)
        {
            pos.y = groundY + halfHeight;
            playerVy = 0f;
            isGrounded = true;
        }
        else
        {
            isGrounded = false;
        }

        player.transform.position = pos;

        // Sliding timer
        if (isSliding)
        {
            slideTimer -= Time.deltaTime;
            if (slideTimer <= 0f)
            {
                StopSlide();
            }
        }

        // Move world: spawn obstacles that move left (obstacles move, player x is fixed visually)
        UpdateObstacles();

        // Spawn logic
        spawnTimer -= Time.deltaTime;
        if (spawnTimer <= 0f)
        {
            SpawnObstacle();
            spawnTimer = currentSpawnInterval;
            currentSpawnInterval = Mathf.Max(spawnIntervalMin, currentSpawnInterval * difficultyRamp);
        }

        // Score
        score += Time.deltaTime * runSpeed * 0.5f;
        scoreText.text = Mathf.FloorToInt(score).ToString();

        // Collision detection AABB between player and obstacles
        UpdatePlayerRect();
        foreach (var obs in obstacles)
        {
            if (obs == null) continue;
            var r = obs.GetComponent<Renderer>();
            if (r == null) continue;
            Bounds b = r.bounds;
            Rect obsRect = new Rect(b.min.x, b.min.y, b.size.x, b.size.y);
            if (playerRect.Overlaps(obsRect))
            {
                // if sliding maybe safe if obstacle high? For simplicity, collide always unless sliding and obstacle is tall?
                bool obstacleHigh = (b.size.y > player.transform.localScale.y * 0.7f);
                if (!isSliding || obstacleHigh)
                {
                    GameOver();
                    return;
                }
            }
        }
    }

    void HandleInput()
    {
        // Editor keyboard fallback
        if (Input.GetKeyDown(KeyCode.Space)) TryJump();
        if (Input.GetKeyDown(KeyCode.DownArrow)) StartSlide();

        // Mouse click = tap
        if (Input.GetMouseButtonDown(0))
        {
            touchBegan = true;
            touchStart = Input.mousePosition;
        }
        if (Input.GetMouseButtonUp(0) && touchBegan)
        {
            Vector2 end = Input.mousePosition;
            Vector2 delta = end - touchStart;
            if (delta.magnitude < 20f)
            {
                TryJump();
            }
            else if (Mathf.Abs(delta.y) > Mathf.Abs(delta.x) && delta.y < -60f)
            {
                StartSlide();
            }
            touchBegan = false;
        }

        // Touch on device
        if (Input.touchCount > 0)
        {
            Touch t = Input.GetTouch(0);
            if (t.phase == TouchPhase.Began)
            {
                touchStart = t.position;
                touchBegan = true;
            }
            else if (t.phase == TouchPhase.Ended && touchBegan)
            {
                Vector2 delta = t.position - touchStart;
                if (delta.magnitude < 20f)
                    TryJump();
                else if (Mathf.Abs(delta.y) > Mathf.Abs(delta.x) && delta.y < -60f)
                    StartSlide();
                touchBegan = false;
            }
        }
    }

    void TryJump()
    {
        if (isGrounded && !isSliding)
        {
            playerVy = jumpVelocity;
            isGrounded = false;
        }
    }

    void StartSlide()
    {
        if (!isGrounded || isSliding) return;
        isSliding = true;
        slideTimer = slideDuration;
        player.transform.localScale = new Vector3(playerSlideSize.x, playerSlideSize.y, 1f);
        // lower player visually a bit
        Vector3 p = player.transform.position;
        p.y = groundY + playerSlideSize.y * 0.5f;
        player.transform.position = p;
    }

    void StopSlide()
    {
        isSliding = false;
        player.transform.localScale = new Vector3(playerOriginalSize.x, playerOriginalSize.y, 1f);
        Vector3 p = player.transform.position;
        p.y = groundY + playerOriginalSize.y * 0.5f;
        player.transform.position = p;
    }

    void SpawnObstacle()
    {
        float spawnX = mainCam.transform.position.x + mainCam.orthographicSize * mainCam.aspect + 2f;
        // randomize obstacle height/size a little
        float h = obstacleSize.y * Random.Range(0.85f, 1.3f);
        float w = obstacleSize.x * Random.Range(0.8f, 1.2f);
        GameObject ob = GameObject.CreatePrimitive(PrimitiveType.Cube);
        ob.name = "Obstacle";
        ob.transform.parent = obstaclesRoot;
        ob.transform.localScale = new Vector3(w, h, 1f);
        ob.transform.position = new Vector3(spawnX, groundY + h * 0.5f, 0f);
        var mr = ob.GetComponent<MeshRenderer>();
        mr.material = new Material(Shader.Find("Unlit/Color"));
        mr.material.color = obstacleColor;

        obstacles.Add(ob);
    }

    void UpdateObstacles()
    {
        float move = runSpeed * Time.deltaTime;
        for (int i = obstacles.Count - 1; i >= 0; i--)
        {
            var o = obstacles[i];
            if (o == null)
            {
                obstacles.RemoveAt(i);
                continue;
            }
            o.transform.position += Vector3.left * move;

            // destroy if offscreen left
            float leftLimit = mainCam.transform.position.x - mainCam.orthographicSize * mainCam.aspect - 3f;
            if (o.transform.position.x < leftLimit)
            {
                Destroy(o);
                obstacles.RemoveAt(i);
            }
        }
    }

    void UpdatePlayerRect()
    {
        var r = player.GetComponent<Renderer>();
        Bounds b = r.bounds;
        playerRect = new Rect(b.min.x, b.min.y, b.size.x, b.size.y);
    }

    void ClearObstacles()
    {
        foreach (var o in obstacles) if (o != null) Destroy(o);
        obstacles.Clear();
    }

    // UI callbacks
    void OnPlayClicked()
    {
        StartGame();
    }

    void OnRetryClicked()
    {
        StartGame();
    }

    // Simple debug: show instructions in editor and allow starting with S key
    void OnGUI()
    {
        if (!isPlaying)
        {
            GUIStyle s = new GUIStyle(GUI.skin.label) { alignment = TextAnchor.UpperCenter, fontSize = 18 };
            GUI.Label(new Rect(0, 5, Screen.width, 30), "Appuyez sur Jouer ou S pour démarrer (éditeur : Space = saut, Down = slide)", s);
            if (Event.current.type == EventType.KeyDown && Event.current.keyCode == KeyCode.S)
            {
                StartGame();
            }
        }
    }

    // Clean up when stopping play in editor
    void OnDestroy()
    {
        // Nothing required
    }
}
