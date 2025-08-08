1) Create project folder & Python venv (PowerShell)
Open PowerShell (as your normal user — admin not required) and run:

powershell
Copy
Edit
# create folder and venv
mkdir C:\webai
cd C:\webai
python -m venv venv

# activate venv
.\venv\Scripts\Activate.ps1

# upgrade pip
python -m pip install --upgrade pip
2) Create requirements.txt (exact)
Create a file requirements.txt with these lines:

nginx
Copy
Edit
openai
duckduckgo_search
requests
beautifulsoup4
python-dotenv
tkcalendar   # optional (if you want extra UI widgets) - not used in base
pyinstaller
playwright   # optional, only install if you will render JS-heavy pages
Install:

powershell
Copy
Edit
pip install -r requirements.txt
# If you want Playwright (optional), next run:
python -m playwright install
Note: tkinter is included with regular Python installs on Windows; no pip needed.

3) Create .env to store keys
In C:\webai create a file named .env with:

makefile
Copy
Edit
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o-mini
SERPAPI_KEY=           # optional: paste your SerpAPI key here if you use it
Security: never commit .env to GitHub.

4) Code — create these files
Create files exactly as below inside C:\webai.

search.py
python
Copy
Edit
# search.py
from duckduckgo_search import ddg
import requests
from bs4 import BeautifulSoup
import time

def web_search(query, max_results=5):
    """
    Returns list of dicts: {title, url, snippet}
    Uses duckduckgo_search ddg() for free results.
    """
    try:
        results = ddg(query, max_results=max_results)
    except Exception:
        results = []
    processed = []
    for r in results:
        processed.append({
            "title": r.get("title"),
            "url": r.get("href"),
            "snippet": r.get("body")
        })
    return processed

def fetch_page_text(url, max_chars=4000, timeout=8):
    """Fetches a page and returns cleaned text, truncated to max_chars."""
    try:
        headers = {"User-Agent": "Mozilla/5.0 (compatible; PersonalWebAI/1.0)"}
        resp = requests.get(url, headers=headers, timeout=timeout)
        if resp.status_code != 200:
            return ""
        soup = BeautifulSoup(resp.text, "html.parser")
        for tag in soup(["script","style","header","footer","nav","noscript","form"]):
            tag.extract()
        text = soup.get_text(separator="\n")
        lines = [ln.strip() for ln in text.splitlines() if ln.strip()]
        joined = "\n".join(lines)
        return joined[:max_chars]
    except Exception:
        return ""
ai_agent.py
python
Copy
Edit
# ai_agent.py
import os
import openai
from dotenv import load_dotenv
load_dotenv()

openai.api_key = os.getenv("OPENAI_API_KEY")
MODEL = os.getenv("OPENAI_MODEL", "gpt-4o-mini")

def build_context_snippets(search_results, fetch_pages=False):
    parts = []
    for idx, r in enumerate(search_results, start=1):
        title = r.get("title") or ""
        url = r.get("url") or ""
        snippet = r.get("snippet") or ""
        if fetch_pages and url:
            from search import fetch_page_text
            page_text = fetch_page_text(url)
            if page_text:
                snippet = page_text
        parts.append(f"[{idx}] {url}\nTitle: {title}\nSnippet: {snippet}")
    return "\n\n".join(parts)

def ask_with_web(query, search_results, fetch_pages=False, max_tokens=700):
    """
    Builds a RAG-style prompt from `search_results` and asks OpenAI model.
    Returns the assistant text.
    """
    context = build_context_snippets(search_results, fetch_pages=fetch_pages)
    system = (
        "You are a concise, expert technical assistant. Use ONLY the information "
        "in the 'Sources' provided. Cite sources inline using the bracket numbers "
        "like [1], [2] that correspond to the enumerated sources. If the answer "
        "is not supported by the sources, say you cannot confirm it."
    )
    user = f"Sources:\n{context}\n\nUser question: {query}\n\nAnswer with clear step-by-step guidance and cite sources by number."
    messages = [
        {"role":"system","content":system},
        {"role":"user","content":user}
    ]
    try:
        resp = openai.ChatCompletion.create(
            model=MODEL,
            messages=messages,
            max_tokens=max_tokens,
            temperature=0.1
        )
        return resp["choices"][0]["message"]["content"].strip()
    except Exception as e:
        return f"Error calling LLM: {e}"
app.py (Tkinter GUI — single process)
python
Copy
Edit
# app.py
import tkinter as tk
from tkinter.scrolledtext import ScrolledText
import threading
from search import web_search, fetch_page_text
from ai_agent import ask_with_web

def run_query(query, display_callback, status_callback):
    status_callback("Searching web...")
    results = web_search(query, max_results=4)
    if not results:
        status_callback("No search results found.")
        display_callback("No search results found.")
        return
    status_callback("Querying LLM (OpenAI)...")
    # Optionally set fetch_pages=True to include top page text (slower)
    answer = ask_with_web(query, results, fetch_pages=False)
    # Format result with sources list
    sources_list = "\n".join([f"[{i+1}] {r['url']}" for i,r in enumerate(results)])
    output = f"{answer}\n\nSources:\n{sources_list}"
    display_callback(output)
    status_callback("Done.")

class App:
    def __init__(self, root):
        self.root = root
        root.title("Web AI Assistant")
        root.geometry("820x600")

        self.frame_top = tk.Frame(root)
        self.frame_top.pack(fill=tk.X, padx=8, pady=6)

        self.entry = tk.Entry(self.frame_top, width=80, font=("Segoe UI", 11))
        self.entry.pack(side=tk.LEFT, padx=(0,8), pady=2)
        self.entry.bind("<Return>", lambda e: self.on_ask())

        self.ask_btn = tk.Button(self.frame_top, text="Ask", command=self.on_ask, width=10)
        self.ask_btn.pack(side=tk.LEFT)

        self.status = tk.Label(root, text="Ready", anchor="w")
        self.status.pack(fill=tk.X, padx=8, pady=(0,6))

        self.output = ScrolledText(root, wrap=tk.WORD, font=("Segoe UI", 10))
        self.output.pack(fill=tk.BOTH, expand=True, padx=8, pady=6)

    def set_status(self, txt):
        self.status.config(text=txt)

    def display_output(self, txt):
        self.output.delete("1.0", tk.END)
        self.output.insert(tk.END, txt)

    def on_ask(self):
        q = self.entry.get().strip()
        if not q:
            return
        self.ask_btn.config(state=tk.DISABLED)
        self.set_status("Starting...")
        # run search+LLM in background
        thread = threading.Thread(
            target=self._bg_work,
            args=(q,)
        )
        thread.daemon = True
        thread.start()

    def _bg_work(self, q):
        def display_cb(txt):
            # UI updates must be in main thread
            self.root.after(0, lambda: self.display_output(txt))
        def status_cb(txt):
            self.root.after(0, lambda: self.set_status(txt))
        run_query(q, display_cb, status_cb)
        self.root.after(0, lambda: self.ask_btn.config(state=tk.NORMAL))

if __name__ == "__main__":
    root = tk.Tk()
    app = App(root)
    root.mainloop()
5) Test locally (before packaging)
In the activated venv run:

powershell
Copy
Edit
python app.py
The GUI should open.

Type a tech question (e.g., How to update Windows drivers?) and press Enter or click Ask.

The app will search the web (via DuckDuckGo), call OpenAI, and display the answer + source URLs.

If you see Error calling LLM: ..., ensure .env has the correct OPENAI_API_KEY and the venv has python-dotenv installed.

6) Optional: Use SerpAPI (better search results)
If you prefer SerpAPI (paid but robust), replace web_search in search.py with a SerpAPI call. You will need to pip install google-search-results and set SERPAPI_KEY in .env. I can paste SerpAPI code if you want — let me know and I’ll include it.

7) Build single .exe with PyInstaller (Windows)
We built a single-process app so packaging is straightforward.

Install PyInstaller (already in requirements.txt) and run:

powershell
Copy
Edit
# Make sure venv is active
.\venv\Scripts\Activate.ps1

# Build one-file executable
pyinstaller --noconfirm --onefile --windowed --name WebAI app.py
--windowed prevents a console window from showing.

Output EXE will be at: C:\webai\dist\WebAI.exe.

Include .env: The exe won't automatically read .env unless present. For convenience, place the .env file in the same folder as WebAI.exe (dist folder) after building.

If you want to bundle .env automatically, use --add-data ".env;." (but keep secrets out of built binaries generally).

8) Distribution & Running
Copy dist\WebAI.exe and the .env file into the same directory on the target machine (or set OPENAI_API_KEY as an environment variable on that machine).

Double-click WebAI.exe to run.

9) Troubleshooting & common fixes
LLM errors: check OPENAI_API_KEY; test with python -c "import openai, os; openai.api_key=os.getenv('OPENAI_API_KEY'); print('ok')" (with venv active).

DuckDuckGo returns empty: ensure duckduckgo_search installed; test python -c "from duckduckgo_search import ddg; print(ddg('open shift', max_results=3))".

PyInstaller exe crashes: run python app.py to ensure code works. For packaging, test pyinstaller --clean -y app.spec or enable --console to see error messages.

Playwright pages: if you choose to fetch page text with Playwright, install and run python -m playwright install once.

10) Security & Cost Notes
Costs: Every query to OpenAI costs tokens. Limit prompt size and max_tokens to control cost. Use caching of search + answers if you plan on repetitive questions.

Privacy: API keys in .env are sensitive. Store them securely; on deployed machines prefer environment variables.

Respect robots.txt & TOS when scraping websites.

11) Next improvements you can add (quick list)
Add Embeddings + Chroma for a personal knowledge DB (RAG with your PDFs).

Swap search to SerpAPI for higher quality search results.

Use Playwright to render dynamic pages before scraping.

Replace OpenAI with a local model (ollama/llama.cpp) if you want offline capability.

Improve UI with Electron for a modern look, or use FastAPI + a web front-end for more features.

If you want, I can now:

A. Paste a ready-to-save ZIP (all files in one message) so you can copy into C:\webai (I can paste all code again compressed).

B. Replace the DuckDuckGo search with SerpAPI code and show how to set that up.

C. Add embedding + Chroma steps for remembering your own docs.

Which one should I do next? (I’ll proceed immediately with the files you pick — no waiting.)
