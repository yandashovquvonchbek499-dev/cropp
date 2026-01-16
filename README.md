<!DOCTYPE html>
<html lang="uz">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>AI Chat — Minimal Clone</title>
  <style>
    :root {
      --bg: #0b0b0b;
      --panel: #121212;
      --text: #eaeaea;
      --muted: #9a9a9a;
      --accent: #2dd4bf;
      --danger: #ef4444;
      --mono: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
      --sans: Inter, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, "Apple Color Emoji", "Segoe UI Emoji";
    }
    * { box-sizing: border-box; }
    html, body { height: 100%; background: var(--bg); color: var(--text); font-family: var(--sans); }
    body { margin: 0; display: grid; grid-template-rows: auto 1fr auto; }
    header {
      padding: 16px 20px; border-bottom: 1px solid #1f1f1f; background: var(--panel);
      display: flex; align-items: center; justify-content: space-between;
    }
    header .brand { font-weight: 600; letter-spacing: 0.2px; }
    header .status { font-size: 12px; color: var(--muted); }
    main { padding: 20px; overflow-y: auto; }
    .chat {
      max-width: 820px; margin: 0 auto; display: grid; gap: 12px;
    }
    .msg {
      display: grid; gap: 6px; padding: 12px 14px; border-radius: 10px; background: #141414;
      border: 1px solid #1f1f1f;
    }
    .msg.user { background: #101010; border-color: #222; }
    .msg.assistant { background: #121212; border-color: #222; }
    .role { font-size: 12px; color: var(--muted); text-transform: uppercase; letter-spacing: 0.6px; }
    .content { white-space: pre-wrap; line-height: 1.55; }
    .thinking { color: var(--muted); font-style: italic; }
    .error { color: var(--danger); }
    footer {
      border-top: 1px solid #1f1f1f; background: var(--panel);
      padding: 12px 20px;
    }
    .composer {
      max-width: 820px; margin: 0 auto; display: grid; grid-template-columns: 1fr auto; gap: 10px;
    }
    textarea {
      width: 100%; min-height: 60px; max-height: 160px; resize: vertical;
      padding: 12px 14px; border-radius: 10px; border: 1px solid #222; background: #0f0f0f; color: var(--text);
      font-family: var(--sans);
    }
    button {
      padding: 12px 16px; border-radius: 10px; border: 1px solid #1f1f1f; background: #0f0f0f; color: var(--text);
      cursor: pointer; font-weight: 600;
    }
    button.primary { background: var(--accent); color: #0b0b0b; border-color: transparent; }
    button:disabled { opacity: 0.6; cursor: not-allowed; }
    .row { display: flex; gap: 10px; align-items: center; }
    .token {
      width: 100%; padding: 8px 10px; border-radius: 8px; border: 1px solid #222; background: #0f0f0f; color: var(--text);
      font-family: var(--mono); font-size: 12px;
    }
    .tip { font-size: 12px; color: var(--muted); margin-top: 6px; }
    .mono { font-family: var(--mono); }
  </style>
</head>
<body>
  <header>
    <div class="brand">AI Chat — Minimal Clone</div>
    <div class="status" id="status">Ready</div>
  </header>

  <main>
    <div class="chat" id="chat"></div>
  </main>

  <footer>
    <div class="composer">
      <textarea id="input" placeholder="Savolingizni yozing... (Shift+Enter — yangi satr, Enter — yuborish)"></textarea>
      <button id="send" class="primary">Yuborish</button>
    </div>
    <div class="row" style="margin-top:10px;">
      <input id="token" class="token" type="password" placeholder="API token (faqat lokal test uchun, prod-da serverda saqlang)" />
      <button id="saveToken">Saqlash</button>
    </div>
    <div class="tip">Tokenni brauzerda saqlash xavfsiz emas—prod muhitda server orqali proxylang va tokenni hech qachon frontendga bermang.</div>
  </footer>

  <script>
    // --- CONFIG: Adjust to your provider ---
    const CONFIG = {
      endpoint: "https://your-api.example.com/v1/chat/completions", // replace with your endpoint
      model: "your-model-id", // replace with your model
      temperature: 0.7,
      maxTokens: 800,
      stream: true // set false if your provider doesn't support streaming
    };

    // --- State ---
    const chatEl = document.getElementById("chat");
    const inputEl = document.getElementById("input");
    const sendBtn = document.getElementById("send");
    const statusEl = document.getElementById("status");
    const tokenEl = document.getElementById("token");
    const saveTokenBtn = document.getElementById("saveToken");

    let token = localStorage.getItem("AI_TOKEN") || "";
    if (token) tokenEl.value = "••••••••••"; // mask if present

    const messages = [
      { role: "system", content: "You are a helpful, concise assistant. Respond in Uzbek when the user writes in Uzbek." }
    ];

    // --- UI helpers ---
    function addMessage(role, content, extraClass = "") {
      const wrap = document.createElement("div");
      wrap.className = `msg ${role} ${extraClass}`;
      const roleEl = document.createElement("div");
      roleEl.className = "role";
      roleEl.textContent = role === "user" ? "User" : role === "assistant" ? "Assistant" : role;
      const contentEl = document.createElement("div");
      contentEl.className = "content";
      contentEl.textContent = content;
      wrap.appendChild(roleEl);
      wrap.appendChild(contentEl);
      chatEl.appendChild(wrap);
      chatEl.scrollTop = chatEl.scrollHeight;
      return contentEl;
    }

    function setStatus(text) {
      statusEl.textContent = text;
    }

    function setLoading(loading) {
      sendBtn.disabled = loading;
      inputEl.disabled = loading;
      setStatus(loading ? "Yozmoqda..." : "Ready");
    }

    // --- Token save ---
    saveTokenBtn.addEventListener("click", () => {
      const raw = tokenEl.value.trim();
      if (!raw) {
        alert("Token kiriting.");
        return;
      }
      token = raw;
      localStorage.setItem("AI_TOKEN", token);
      tokenEl.value = "••••••••••";
      setStatus("Token saqlandi (lokal).");
    });

    // --- Send message ---
    async function sendMessage() {
      const text = inputEl.value.trim();
      if (!text) return;
      addMessage("user", text);
      messages.push({ role: "user", content: text });
      inputEl.value = "";
      await requestAI(messages);
    }

    // --- Keyboard shortcuts ---
    inputEl.addEventListener("keydown", (e) => {
      if (e.key === "Enter" && !e.shiftKey) {
        e.preventDefault();
        sendMessage();
      }
    });
    sendBtn.addEventListener("click", sendMessage);

    // --- AI request (supports streaming if provider does) ---
    async function requestAI(history) {
      if (!token) {
        addMessage("assistant", "Token topilmadi. Iltimos, token kiriting va qayta urinib ko‘ring.", "error");
        return;
      }
      setLoading(true);

      // Placeholder assistant bubble
      const assistantContentEl = addMessage("assistant", "", "thinking");

      try {
        const payload = {
          model: CONFIG.model,
          messages: history,
          temperature: CONFIG.temperature,
          max_tokens: CONFIG.maxTokens,
          stream: CONFIG.stream
        };

        const res = await fetch(CONFIG.endpoint, {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
            "Authorization": `Bearer ${token}`
          },
          body: JSON.stringify(payload)
        });

        if (!res.ok) {
          const errText = await res.text();
          assistantContentEl.textContent = `Xato: ${res.status} — ${errText}`;
          assistantContentEl.parentElement.classList.remove("thinking");
          assistantContentEl.parentElement.classList.add("error");
          setLoading(false);
          return;
        }

        if (CONFIG.stream && res.body) {
          const reader = res.body.getReader();
          const decoder = new TextDecoder("utf-8");
          let fullText = "";

          while (true) {
            const { value, done } = await reader.read();
            if (done) break;
            const chunk = decoder.decode(value, { stream: true });

            // Example: handle Server-Sent Events or JSON lines
            // Adjust parsing to your provider's streaming format.
            const lines = chunk.split("\n").filter(Boolean);
            for (const line of lines) {
              // If SSE: lines start with "data: ..."
              const trimmed = line.replace(/^data:\s?/, "");
              if (trimmed === "[DONE]") continue;

              try {
                const json = JSON.parse(trimmed);
                // Example OpenAI-like delta:
                const delta = json.choices?.[0]?.delta?.content ?? json.choices?.[0]?.message?.content ?? "";
                if (delta) {
                  fullText += delta;
                  assistantContentEl.textContent = fullText;
                }
              } catch {
                // Fallback: append raw text if not JSON
                fullText += trimmed;
                assistantContentEl.textContent = fullText;
              }
            }
          }

          messages.push({ role: "assistant", content: fullText });
        } else {
          const data = await res.json();
          const text =
            data.choices?.[0]?.message?.content ??
            data.choices?.[0]?.delta?.content ??
            data.output?.text ??
            JSON.stringify(data);
          assistantContentEl.textContent = text;
          messages.push({ role: "assistant", content: text });
        }

        assistantContentEl.parentElement.classList.remove("thinking");
      } catch (e) {
        assistantContentEl.textContent = `Xato: ${e.message}`;
        assistantContentEl.parentElement.classList.remove("thinking");
        assistantContentEl.parentElement.classList.add("error");
      } finally {
        setLoading(false);
      }
    }

    // --- Welcome message ---
    window.addEventListener("DOMContentLoaded", () => {
      addMessage("assistant", "Salom, Quvonchbek. Minimal AI chat tayyor. Savolingizni yozing.");
    });
  </script>
</body>
</html>
