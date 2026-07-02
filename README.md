# Yolo-Inspection-with-XR10
🚧 Work in Progress ⏳ Source code and supporting files are being prepared. 📦 Public release coming soon. 🙏 Please wait for the official release. Thank you for your patience!

Download UI from here: 1. <img width="571" height="1202" alt="Screenshot 2026-02-14 034627" src="https://github.com/user-attachments/assets/aaa7f9de-d523-45b9-85c6-a97428bc4596" />
2. <img width="464" height="1047" alt="Screenshot 2026-02-14 040245" src="https://github.com/user-attachments/assets/61c42a45-bc13-4821-b574-77548c1cd518" />
3. <img width="2559" height="1531" alt="Screenshot 2026-04-17 221226" src="https://github.com/user-attachments/assets/cb4854f5-a710-4cda-8d5d-eef1d6840c5b" />
4. <img width="1764" height="1003" alt="Screenshot 2026-04-28 205359" src="https://github.com/user-attachments/assets/b6b486c0-1607-45c3-b834-0756ef3a0d2a" />



//Inspection full script //

using System.Collections;
using System.Text;
using UnityEngine;
using UnityEngine.UI;
using UnityEngine.Networking;
using TMPro;

public class ClaudeAIHelper : MonoBehaviour
{
    [Header("--- Claude API ---")]
    public string apiKey = "paste-your-key-here";

    [Header("--- Panel UI ---")]
    public GameObject claudeAIPanel;
    public TextMeshProUGUI aiHeaderText;
    public TextMeshProUGUI aiStatusText;
    public TextMeshProUGUI aiResponseText;
    public GameObject btnAnalyse;
    public GameObject btnCloseAI;
    public GameObject btnAIHelp;

    [Header("--- References ---")]
    public LiveInspectionUI liveInspectionUI;
    public SimpleCameraFeed cameraFeed;

    [Header("--- Animation ---")]
    public float slideSpeed = 800f;

    private bool isPanelOpen = false;
    private bool isAnalysing = false;
    private Vector2 panelHiddenPos;
    private Vector2 panelShownPos;
    private RectTransform panelRect;
    private string lastAIResponse = "";
    private string lastAIEquipment = "";

    public string GetLastAIResponse()
    { return lastAIResponse; }
    public string GetLastAIEquipment()
    { return lastAIEquipment; }
    public bool IsPanelOpen()
    { return isPanelOpen; }

    void Start()
    {
        panelRect =
            claudeAIPanel.GetComponent<RectTransform>();
        panelShownPos = panelRect.anchoredPosition;
        panelHiddenPos = new Vector2(
            panelShownPos.x, panelShownPos.y - 500f);
        panelRect.anchoredPosition = panelHiddenPos;
        claudeAIPanel.SetActive(false);

        WireButton(btnAIHelp, TogglePanel);
        WireButton(btnAnalyse, AnalyseEquipment);
        WireButton(btnCloseAI, ClosePanel);

        Debug.Log("[ClaudeAI] Ready.");
    }

    private void WireButton(GameObject go,
        UnityEngine.Events.UnityAction action)
    {
        if (go == null) return;
        var btn = go.GetComponent<Button>();
        if (btn == null) return;
        btn.onClick.RemoveAllListeners();
        btn.onClick.AddListener(action);
    }

    // ===== Panel Control =====
    public void TogglePanel()
    {
        if (isPanelOpen) ClosePanel();
        else OpenPanel();
    }

    public void OpenPanel()
    {
        if (isPanelOpen) return;
        isPanelOpen = true;
        claudeAIPanel.SetActive(true);

        // Show professional waiting message
        SetStatus("Claude AI — Fire Safety Assistant");
        SetResponse(
            "Point your camera directly at the fire " +
            "equipment you want to inspect.\n\n" +
            "Make sure the equipment is clearly visible " +
            "in the camera view.\n\n" +
            "Then say:\n" +
            "\"Inspection Analyse\"\n" +
            "or click the Analyse button below.");

        StopAllCoroutines();
        StartCoroutine(SlidePanel(
            panelHiddenPos, panelShownPos));

        Debug.Log("[ClaudeAI] Panel opened.");
    }

    public void ClosePanel()
    {
        if (!isPanelOpen) return;
        isPanelOpen = false;
        isAnalysing = false;

        StopAllCoroutines();
        StartCoroutine(SlideThenHide(
            panelShownPos, panelHiddenPos));

        Debug.Log("[ClaudeAI] Panel closed.");
    }

    // ===== Slide Animation =====
    private IEnumerator SlidePanel(
        Vector2 from, Vector2 to)
    {
        float elapsed = 0f;
        float duration = Mathf.Clamp(
            Vector2.Distance(from, to) / slideSpeed,
            0.2f, 0.5f);

        while (elapsed < duration)
        {
            elapsed += Time.deltaTime;
            float t = Mathf.SmoothStep(
                0f, 1f, elapsed / duration);
            panelRect.anchoredPosition =
                Vector2.Lerp(from, to, t);
            yield return null;
        }
        panelRect.anchoredPosition = to;
    }

    private IEnumerator SlideThenHide(
        Vector2 from, Vector2 to)
    {
        yield return StartCoroutine(
            SlidePanel(from, to));
        claudeAIPanel.SetActive(false);
    }

    // ===== Analyse =====
    public void AnalyseEquipment()
    {
        if (isAnalysing) return;
        if (!isPanelOpen)
        {
            OpenPanel();
            StartCoroutine(AnalyseAfterOpen());
            return;
        }
        if (string.IsNullOrEmpty(apiKey) ||
            apiKey == "paste-your-key-here")
        {
            SetStatus("ERROR: API key not set!");
            return;
        }
        StartCoroutine(RunAnalysis());
    }

    private IEnumerator AnalyseAfterOpen()
    {
        yield return new WaitForSeconds(0.6f);
        if (string.IsNullOrEmpty(apiKey) ||
            apiKey == "paste-your-key-here")
        {
            SetStatus("ERROR: API key not set!");
            yield break;
        }
        StartCoroutine(RunAnalysis());
    }

    private IEnumerator RunAnalysis()
    {
        isAnalysing = true;

        SetStatus("Capturing camera image...");
        SetResponse("");
        SetButtonInteractable(btnAnalyse, false);

        // Step 1 — Capture image
        string base64Image = "";
        if (cameraFeed != null &&
            cameraFeed.IsCameraReady())
        {
            WebCamTexture camTex =
                cameraFeed.GetCameraTexture();
            if (camTex != null)
            {
                RenderTexture rt =
                    RenderTexture.GetTemporary(
                        512, 512, 0);
                Graphics.Blit(camTex, rt);
                RenderTexture prev =
                    RenderTexture.active;
                RenderTexture.active = rt;
                Texture2D snap = new Texture2D(
                    512, 512,
                    TextureFormat.RGB24, false);
                snap.ReadPixels(
                    new Rect(0, 0, 512, 512), 0, 0);
                snap.Apply();
                RenderTexture.active = prev;
                RenderTexture.ReleaseTemporary(rt);
                byte[] imgBytes = snap.EncodeToJPG(80);
                base64Image =
                    System.Convert.ToBase64String(
                        imgBytes);
                Destroy(snap);
                SetStatus(
                    "Image captured. " +
                    "Contacting Claude AI...");
            }
        }
        else
        {
            SetStatus(
                "Camera not ready. " +
                "Sending text analysis only...");
        }

        yield return new WaitForSeconds(0.1f);

        // Step 2 — Get context
        string equipmentName = "Fire Safety Equipment";
        string checklistItem = "General fire safety check";
        if (liveInspectionUI != null)
        {
            equipmentName =
                liveInspectionUI.GetCurrentEquipmentName();
            checklistItem =
                liveInspectionUI.GetCurrentChecklistItem();
        }
        lastAIEquipment = equipmentName;

        // Step 3 — Build request
        string systemPrompt =
            "You are a professional fire safety inspection " +
            "assistant integrated into a Mixed Reality " +
            "inspection system. Your ONLY role is to analyse " +
            "fire safety equipment images and provide " +
            "structured inspection reports. " +
            "You do NOT answer any other questions. " +
            "If asked anything unrelated to fire safety " +
            "equipment inspection, respond only with: " +
            "I only assist with fire safety inspection.\n\n" +
            "ALWAYS respond in EXACTLY this format:\n\n" +
            "DEFECTS FOUND:\n" +
            "- [defect or None visible]\n\n" +
            "RISK LEVEL: Low / Medium / High\n\n" +
            "CORRECTIVE ACTIONS:\n" +
            "1. [action]\n" +
            "2. [action]\n" +
            "3. [action]\n\n" +
            "IMMEDIATE ACTION REQUIRED: Yes / No";

        string userText =
            "Equipment: " + equipmentName + "\n" +
            "Current check: " + checklistItem + "\n\n" +
            "Analyse this fire safety equipment image. " +
            "Identify all visible defects including " +
            "corrosion, physical damage, missing components, " +
            "incorrect pressure readings, missing safety " +
            "pins, damaged hoses, missing inspection tags, " +
            "or any safety violations. " +
            "Provide professional corrective actions.";

        // Step 4 — Build JSON
        StringBuilder json = new StringBuilder();
        json.Append("{");
        json.Append(
            "\"model\":\"claude-haiku-4-5-20251001\",");
        json.Append("\"max_tokens\":800,");
        json.Append("\"system\":\"");
        json.Append(EscapeForJson(systemPrompt));
        json.Append("\",");
        json.Append(
            "\"messages\":[{\"role\":\"user\"," +
            "\"content\":[");

        if (!string.IsNullOrEmpty(base64Image))
        {
            json.Append(
                "{\"type\":\"image\",\"source\":{");
            json.Append("\"type\":\"base64\",");
            json.Append(
                "\"media_type\":\"image/jpeg\",");
            json.Append("\"data\":\"");
            json.Append(base64Image);
            json.Append("\"}},");
        }

        json.Append(
            "{\"type\":\"text\",\"text\":\"");
        json.Append(EscapeForJson(userText));
        json.Append("\"}]}]}");

        // Step 5 — Send request
        byte[] bodyBytes = Encoding.UTF8.GetBytes(
            json.ToString());

        using (UnityWebRequest req = new UnityWebRequest(
            "https://api.anthropic.com/v1/messages",
            "POST"))
        {
            req.uploadHandler =
                new UploadHandlerRaw(bodyBytes);
            req.downloadHandler =
                new DownloadHandlerBuffer();
            req.SetRequestHeader(
                "Content-Type", "application/json");
            req.SetRequestHeader("x-api-key", apiKey);
            req.SetRequestHeader(
                "anthropic-version", "2023-06-01");

            SetStatus(
                "Analysing equipment with Claude AI...");
            yield return req.SendWebRequest();

            if (req.result ==
                UnityWebRequest.Result.Success)
            {
                string parsed = ParseResponse(
                    req.downloadHandler.text);
                lastAIResponse = parsed;
                SetStatus("Analysis complete.");
                SetResponse(parsed);
                Debug.Log(
                    "[ClaudeAI] Response received.");
            }
            else
            {
                string errBody =
                    req.downloadHandler.text;
                Debug.LogError(
                    "[ClaudeAI] FAILED: " +
                    req.error + "\n" + errBody);
                SetStatus(
                    "Error: " + req.responseCode);
                SetResponse(
                    "Failed: " + req.error +
                    "\n\n" + errBody);
            }
        }

        SetButtonInteractable(btnAnalyse, true);
        isAnalysing = false;
    }

    private string ParseResponse(string json)
    {
        try
        {
            string marker = "\"text\":\"";
            int start = json.IndexOf(marker);
            if (start < 0)
                return "No response received.";
            start += marker.Length;
            int end = start;
            while (end < json.Length)
            {
                if (json[end] == '"' &&
                    json[end - 1] != '\\') break;
                end++;
            }
            string raw = json.Substring(
                start, end - start);
            return raw
                .Replace("\\n", "\n")
                .Replace("\\\"", "\"")
                .Replace("\\\\", "\\")
                .Replace("\\t", "  ");
        }
        catch (Exception e)
        {
            return "Parse error: " + e.Message;
        }
    }

    private void SetStatus(string msg)
    {
        if (aiStatusText != null)
            aiStatusText.text = msg;
    }

    private void SetResponse(string msg)
    {
        if (aiResponseText != null)
            aiResponseText.text = msg;
    }

    private void SetButtonInteractable(
        GameObject go, bool state)
    {
        if (go == null) return;
        var btn = go.GetComponent<Button>();
        if (btn != null) btn.interactable = state;
    }

    private static string EscapeForJson(string s)
    {
        if (s == null) return "";
        return s
            .Replace("\\", "\\\\")
            .Replace("\"", "\\\"")
            .Replace("\n", "\\n")
            .Replace("\r", "\\r")
            .Replace("\t", "\\t");
    }
}
