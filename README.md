# HexStrike AI — Complete Deep Technical Analysis
> **Repository:** `hexstrike-ai` | **Version:** v6.0 | **Files Analyzed:** [hexstrike_server.py](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_server.py) (17,290 lines), [hexstrike_mcp.py](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_mcp.py) (5,471 lines), [requirements.txt](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/requirements.txt), [hexstrike-ai-mcp.json](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike-ai-mcp.json)

---

## Table of Contents
1. [Phase 1 — Architecture Analysis](#phase-1--architecture-analysis)
2. [Phase 2 — Internal Working (Deep Dive)](#phase-2--internal-working-deep-dive)
3. [Phase 3 — Rebuild Guide From Scratch](#phase-3--rebuild-guide-from-scratch)
4. [Bonus — Improvements, Features & Converting Tools](#bonus--improvements-features--converting-tools)

---

# PHASE 1 — Architecture Analysis

## 1.1 Overview: The Two-Script System

HexStrike AI uses a **dual-process architecture**:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    HEXSTRIKE AI SYSTEM                               │
│                                                                      │
│  ┌─────────────────────────┐       ┌──────────────────────────────┐ │
│  │   hexstrike_mcp.py      │       │   hexstrike_server.py        │ │
│  │                         │       │                              │ │
│  │  🤖 MCP Server          │  HTTP │  🌐 Flask REST API Server    │ │
│  │  (Claude's Interface)   │◄─────►│  (Tool Executor & Brain)     │ │
│  │                         │ :8888 │                              │ │
│  │  - FastMCP              │       │  - Flask App                 │ │
│  │  - 100+ Tool Wrappers   │       │  - subprocess runner         │ │
│  │  - AI ↔ Tool Bridge     │       │  - IntelligentDecisionEngine │ │
│  │                         │       │  - ErrorRecovery System      │ │
│  │  Runs as MCP Server     │       │  - CVE Intelligence          │ │
│  │  Claude talks to this   │       │  - Process Manager           │ │
│  └─────────────────────────┘       └──────────────────────────────┘ │
│             ▲                                      │                 │
│             │ MCP Protocol                         │ subprocess      │
│             │ (JSON-RPC / stdio)                   ▼                 │
│  ┌──────────┴──────────┐              ┌────────────────────────┐    │
│  │   Claude Desktop    │              │  OS-Level Security     │    │
│  │   / Claude API      │              │  Tools (nmap, gobuster, │    │
│  │   (LLM / AI Brain)  │              │  nuclei, sqlmap, etc.) │    │
│  └─────────────────────┘              └────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

**Key Insight:** Claude Desktop is NOT just a user interface — it IS the LLM intelligence. The MCP layer ([hexstrike_mcp.py](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_mcp.py)) exposes tools that Claude can call as function calls. The Flask server is what actually runs the OS commands.

---

## 1.2 How MCP Is Implemented

MCP (**Model Context Protocol**) is an open standard by Anthropic that lets Claude (or any LLM) call external tools by name, passing typed arguments, just like calling a Python function.

### Implementation Detail

```python
# hexstrike_mcp.py — Line 277
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("hexstrike-ai-mcp")   # Creates the MCP server instance

@mcp.tool()
def nmap_scan(target: str, scan_type: str = "-sV", ports: str = "", additional_args: str = "") -> Dict[str, Any]:
    """
    Execute an enhanced Nmap scan against a target.
    [full docstring becomes the tool description Claude sees]
    """
    data = {"target": target, "scan_type": scan_type, ...}
    result = hexstrike_client.safe_post("api/tools/nmap", data)
    return result
```

- `FastMCP` is from the `fastmcp` Python package (version ≥ 0.2.0)
- The `@mcp.tool()` decorator registers a Python function as a callable MCP tool
- The **function name** = tool name (what Claude sees)
- The **docstring** = tool description (Claude reads this to understand what the tool does)
- The **type hints** = the schema for arguments (Claude uses this to know what to pass)
- At the bottom: `mcp.run()` starts the MCP server over **stdio** (standard input/output)

### MCP Configuration (Claude Desktop)

```json
// hexstrike-ai-mcp.json
{
  "mcpServers": {
    "hexstrike-ai": {
      "command": "python3",
      "args": ["/path/hexstrike_mcp.py", "--server", "http://IPADDRESS:8888"],
      "timeout": 300,
      "alwaysAllow": []
    }
  }
}
```

Claude Desktop launches [hexstrike_mcp.py](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_mcp.py) as a child process and communicates with it over **JSON-RPC via stdio**. This is the MCP protocol.

---

## 1.3 How Tools Are Registered and Executed

### Registration Flow

```
1. hexstrike_mcp.py starts
2. setup_mcp_server(hexstrike_client) is called
3. Inside, each @mcp.tool() decorated function is registered into FastMCP
4. mcp.run() is called → listens on stdio for JSON-RPC calls from Claude
```

Every single tool follows this exact pattern (100+ tools all identical in structure):

```python
# Pattern: MCP Tool → HTTP POST → Flask API → subprocess
@mcp.tool()
def gobuster_scan(url: str, mode: str = "dir", wordlist: str = "...", additional_args: str = "") -> Dict:
    data = {"url": url, "mode": mode, "wordlist": wordlist, "additional_args": additional_args}
    result = hexstrike_client.safe_post("api/tools/gobuster", data)   # HTTP POST to Flask
    return result
```

### Execution Flow

```
Claude wants to scan → Calls nmap_scan("192.168.1.1") via MCP
         ↓
FastMCP receives JSON-RPC call → dispatches to nmap_scan() Python function
         ↓
nmap_scan() builds JSON payload → calls hexstrike_client.safe_post("api/tools/nmap", data)
         ↓
HTTP POST → http://127.0.0.1:8888/api/tools/nmap
         ↓
Flask route on hexstrike_server.py handles the request
         ↓
subprocess.run(["nmap", ...]) executes the actual OS tool
         ↓
Result (stdout/stderr/returncode) → Flask returns JSON response
         ↓
hexstrike_mcp.py gets the JSON → returns it to FastMCP
         ↓
FastMCP sends result back to Claude via stdio JSON-RPC
         ↓
Claude reads the result and decides next action
```

---

## 1.4 How LLM / AI Integration Works

There are **two types of "AI"** in HexStrike:

### A) Claude (External LLM) — The Real Brain
- Claude Desktop / Claude API IS the AI decision-maker
- It reads tool descriptions (from docstrings), decides which tool to call, reads results, and decides what to do next
- HexStrike doesn't call any OpenAI/Anthropic API itself — it IS a tool server FOR Claude

### B) IntelligentDecisionEngine (Internal Heuristics — Not True AI)
This is NOT a real AI/LLM. It's a **rule-based heuristic system** on the server:

```python
# hexstrike_server.py — Line 572
class IntelligentDecisionEngine:
    def __init__(self):
        self.tool_effectiveness = {
            "web_application": {"nmap": 0.8, "gobuster": 0.9, "nuclei": 0.95, ...},
            "network_host":    {"nmap": 0.95, "masscan": 0.92, ...},
        }
        self.attack_patterns = {
            "web_reconnaissance": [
                {"tool": "nmap",     "priority": 1, "params": {...}},
                {"tool": "gobuster", "priority": 2, "params": {...}},
                ...
            ]
        }
    
    def analyze_target(self, target: str) -> TargetProfile:
        # Detects target type (URL → web, IP → network, .exe → binary)
        # Calculates attack surface score
        # Returns TargetProfile dataclass
    
    def select_optimal_tools(self, profile, objective) -> List[str]:
        # Returns top tools by effectiveness rating
    
    def create_attack_chain(self, profile, objective) -> AttackChain:
        # Builds a sequential list of AttackStep objects
```

The "intelligence" is hardcoded tables — it does NOT use any ML model. When Claude calls `get_attack_chain`, the engine looks up the target type in a dictionary and returns a pre-defined attack sequence.

---

## 1.5 How User Input Flows Through The System

```
User types in Claude Desktop:
"Scan 192.168.1.100 for open ports and vulnerabilities"
           │
           ▼
Claude (LLM) interprets intent → decides to call nmap_scan("192.168.1.100", "-sV")
           │
           ▼  [MCP stdio JSON-RPC]
hexstrike_mcp.py receives the call
HexStrikeClient.safe_post("api/tools/nmap", {...}) is called
           │
           ▼  [HTTP POST to :8888]
hexstrike_server.py Flask route /api/tools/nmap
IntelligentDecisionEngine may optimize parameters
subprocess.run(["nmap", "-sV", "192.168.1.100"]) executes
           │
           ▼  [JSON response]
{"success": true, "stdout": "...", "ports": [...], "execution_time": 12.3}
           │
           ▼  [Back via HTTP → MCP → Claude]
Claude reads the JSON results
Decides: "Port 80 open, run gobuster_scan next"
Calls: gobuster_scan("http://192.168.1.100")
           │
           [Repeats until objective complete]
```

---

## 1.6 Core Components — Every File and Folder

### Repository Structure

```
hexstrike-ai/
│
├── hexstrike_mcp.py         ← MCP SERVER (5,471 lines)
│   ├── HexStrikeColors       - ANSI color palette class
│   ├── ColoredFormatter      - Logging with color/emoji
│   ├── HexStrikeClient       - HTTP client for Flask API
│   │   ├── safe_get()        - GET requests to Flask
│   │   ├── safe_post()       - POST requests to Flask
│   │   └── execute_command() - Generic command execution
│   └── setup_mcp_server()    - Registers ALL 100+ @mcp.tool() functions
│       ├── Network Tools     : nmap_scan, gobuster_scan, nuclei_scan, masscan, rustscan...
│       ├── Cloud Tools       : prowler, trivy, kube_hunter, checkov, terrascan...
│       ├── Web Tools         : ffuf, nikto, sqlmap, wpscan, dalfox, arjun...
│       ├── Password Tools    : hydra, john, hashcat...
│       ├── Binary Tools      : ghidra, radare2, angr, pwntools, ropper...
│       ├── File Tools        : create_file, modify_file, delete_file, list_files
│       ├── Python Env Tools  : install_python_package, execute_python_script
│       └── Meta Tools        : get_attack_chain, get_server_health, error_handling_statistics
│
├── hexstrike_server.py      ← FLASK API SERVER + BRAIN (17,290 lines)
│   ├── Flask App (app)       - REST API server on port 8888
│   ├── ModernVisualEngine    - Terminal UI rendering (banners, progress bars)
│   ├── IntelligentDecisionEngine  - Heuristic tool selection engine
│   │   ├── TargetType (Enum) - WEB_APPLICATION, NETWORK_HOST, API_ENDPOINT...
│   │   ├── TargetProfile     - Dataclass for target analysis results
│   │   ├── AttackStep        - Single step in an attack chain
│   │   ├── AttackChain       - Sequence of AttackSteps
│   │   └── optimize_parameters() - Tweaks tool flags per target type
│   ├── ErrorHandlingSystem   - Intelligent error recovery
│   │   ├── ErrorType (Enum)  - TIMEOUT, PERMISSION_DENIED, RATE_LIMITED...
│   │   ├── RecoveryAction    - RETRY, SWITCH_TOOL, ESCALATE_TO_HUMAN...
│   │   └── ErrorHandler      - Decides recovery strategy per error type
│   ├── CTFWorkflowManager    - CTF-specific workflow creation
│   ├── CTFToolManager        - 150+ CTF tool command templates
│   ├── CTFChallengeAutomator - Automated CTF challenge solving
│   ├── CTFTeamCoordinator    - Multi-member CTF team management
│   ├── TechnologyDetector    - Header/content-based tech fingerprinting
│   ├── RateLimitDetector     - Detects and responds to rate limits
│   ├── FailureRecoverySystem - Alternative tool selection on failure
│   ├── PerformanceMonitor    - Execution metrics and trend analysis
│   ├── PerformanceDashboard  - Real-time execution history tracking
│   ├── PythonEnvironmentManager - venv creation and package management
│   ├── CVEIntelligenceManager - Real-time CVE fetching from NVD API
│   ├── ProcessManager        - subprocess lifecycle management
│   │   ├── register_process()  - Track PIDs of running tools
│   │   ├── terminate_process() - Kill specific processes
│   │   ├── pause_process()     - SIGSTOP a process
│   │   └── resume_process()    - SIGCONT a process
│   └── Flask Routes          - /health, /api/tools/*, /api/files/*, /api/python/*
│
├── requirements.txt         ← Dependencies (Flask, requests, FastMCP, aiohttp, mitmproxy...)
│
├── hexstrike-ai-mcp.json    ← Claude Desktop config (how to launch the MCP server)
│
├── README.md                ← Documentation (31KB)
│
└── assets/                  ← Static assets (images, etc.)
```

---

## 1.7 End-to-End Execution Flow Diagram

```
╔══════════════════════════════════════════════════════════════════════════════╗
║           HEXSTRIKE AI — END-TO-END EXECUTION FLOW                          ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  [1] User types task in Claude Desktop                                       ║
║       "Scan example.com for vulnerabilities"                                 ║
║            │                                                                 ║
║            ▼                                                                 ║
║  [2] Claude LLM runs inference on message                                    ║
║       Reads tool schemas (from @mcp.tool docstrings)                         ║
║       Decides: call nmap_scan("example.com", "-sV -sC", "80,443")            ║
║            │                                                                 ║
║            ▼  [JSON-RPC over stdio]                                          ║
║  [3] hexstrike_mcp.py receives tool call                                     ║
║       FastMCP dispatches to nmap_scan() function                             ║
║            │                                                                 ║
║            ▼  [HTTP POST :8888]                                              ║
║  [4] HexStrikeClient.safe_post("api/tools/nmap", payload)                   ║
║            │                                                                 ║
║            ▼                                                                 ║
║  [5] hexstrike_server.py Flask route /api/tools/nmap receives request        ║
║       IntelligentDecisionEngine.optimize_parameters("nmap", profile)         ║
║       ProcessManager.register_process(pid, command, proc)                    ║
║            │                                                                 ║
║            ▼  [subprocess.run()]                                             ║
║  [6] OS executes: nmap -sV -sC -p 80,443 example.com                        ║
║       stdout/stderr captured in real-time                                    ║
║            │                                                                 ║
║            ▼                                                                 ║
║  [7] FailureRecoverySystem checks for errors                                 ║
║       If timeout → retry with reduced scope                                  ║
║       If tool not found → suggest rustscan as alternative                    ║
║            │                                                                 ║
║            ▼  [JSON Response]                                                ║
║  [8] {"success": true, "stdout": "...", "ports": [...], "execution_time": 5} ║
║            │                                                                 ║
║            ▼  [HTTP Response back to MCP]                                    ║
║  [9] hexstrike_mcp.py receives JSON result                                   ║
║       Logs result with colored output                                        ║
║       Returns Dict to FastMCP                                                ║
║            │                                                                 ║
║            ▼  [JSON-RPC result back to Claude]                               ║
║  [10] Claude reads scan results                                              ║
║        "Port 80 open, HTTP server detected → run gobuster_scan next"         ║
║        Continues autonomously or asks user for next step                     ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

# PHASE 2 — Internal Working (Deep Dive)

## 2.1 Step-by-Step: What Happens When a User Runs a Command

Let's trace [nmap_scan("192.168.1.1")](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_mcp.py#283-325) all the way through:

**Step 1: Claude decides to call nmap_scan**
- Claude receives user input, runs inference
- Sees the tool [nmap_scan](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_mcp.py#283-325) with docstring: *"Execute an enhanced Nmap scan against a target"*
- Generates a JSON-RPC call: `{"method": "tools/call", "params": {"name": "nmap_scan", "arguments": {"target": "192.168.1.1"}}}`
- Sends it over stdio to [hexstrike_mcp.py](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_mcp.py)

**Step 2: FastMCP dispatches the call (hexstrike_mcp.py)**
```python
# FastMCP internally does something like:
tool_fn = registered_tools["nmap_scan"]
result = tool_fn(target="192.168.1.1")
```

**Step 3: The nmap_scan function executes**
```python
@mcp.tool()
def nmap_scan(target: str, scan_type: str = "-sV", ...):
    data = {"target": target, "scan_type": "-sV", "use_recovery": True}
    logger.info(f"🔍 Initiating Nmap scan: {target}")
    result = hexstrike_client.safe_post("api/tools/nmap", data)
    # logs success/failure
    return result
```

**Step 4: HexStrikeClient.safe_post() sends HTTP request**
```python
def safe_post(self, endpoint, json_data):
    url = f"{self.server_url}/{endpoint}"  # http://127.0.0.1:8888/api/tools/nmap
    response = self.session.post(url, json=json_data, timeout=300)
    return response.json()
```

**Step 5: Flask route on hexstrike_server.py**
The Flask server has a route like:
```python
@app.route('/api/tools/nmap', methods=['POST'])
def run_nmap():
    data = request.json
    target = data.get('target')
    scan_type = data.get('scan_type', '-sV')
    # Build command
    cmd = ["nmap", scan_type, target]
    # Execute via subprocess
    process = subprocess.Popen(cmd, stdout=PIPE, stderr=PIPE)
    ProcessManager.register_process(process.pid, ' '.join(cmd), process)
    stdout, stderr = process.communicate(timeout=300)
    return jsonify({"success": True, "stdout": stdout.decode(), ...})
```

**Step 6: OS executes the tool**
- `nmap -sV 192.168.1.1` runs as a child process
- stdout/stderr captured
- ProcessManager tracks the PID

**Step 7: Recovery system checks output**
- If stderr contains "permission denied" → `RecoveryAction.ESCALATE_TO_HUMAN`
- If exit code 124 (timeout) → `RecoveryAction.RETRY_WITH_REDUCED_SCOPE`
- If "command not found" → `RecoveryAction.SWITCH_TO_ALTERNATIVE_TOOL` (rustscan)

**Step 8: Response flows back**
JSON → HTTP response → HexStrikeClient → `@mcp.tool()` function → FastMCP → Claude

---

## 2.2 How MCP Handles Requests Internally

FastMCP uses the **stdio transport** (standard I/O). The flow is:

```
Claude Desktop → writes JSON-RPC to hexstrike_mcp.py's stdin
hexstrike_mcp.py → reads from stdin, dispatches to tool function
tool function runs → returns dict
hexstrike_mcp.py → writes JSON-RPC response to stdout
Claude Desktop → reads from hexstrike_mcp.py's stdout
```

This is completely synchronous per call. `mcp.run()` is a blocking event loop.

**Important:** All logging in [hexstrike_mcp.py](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_mcp.py) goes to [stderr](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_server.py#6810-6820) specifically to avoid polluting the JSON-RPC stdout channel:
```python
logging.StreamHandler(sys.stderr)  # NOT stdout — that's for MCP protocol
```

---

## 2.3 How Tools Are Triggered and Return Results

The MCP client ([hexstrike_mcp.py](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_mcp.py)) has a single [HexStrikeClient](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_mcp.py#147-266) instance shared by all 100+ tool functions via **closure capture**:

```python
def setup_mcp_server(hexstrike_client: HexStrikeClient) -> FastMCP:
    mcp = FastMCP("hexstrike-ai-mcp")
    
    @mcp.tool()
    def nmap_scan(target: str, ...) -> Dict:
        # hexstrike_client is captured from the outer scope (closure)
        result = hexstrike_client.safe_post(...)
        return result
    
    # All other tools also close over hexstrike_client
    
    return mcp
```

Every tool always returns a `Dict[str, Any]` with at minimum:
- `"success": bool` — whether the tool ran without error
- `"stdout": str` — raw tool output
- `"stderr": str` — error output
- `"return_code": int` — process exit code
- `"execution_time": float` — seconds taken
- Optional: `"recovery_info"`, `"human_escalation"`, `"alternative_tool_suggested"`

---

## 2.4 How External Tools Like Nmap Are Integrated

External tools are integrated via **subprocess execution** on the Flask server side. The server uses:
- `subprocess.run()` for synchronous, short-lived commands
- `subprocess.Popen()` for long-running commands with streaming output
- [ProcessManager](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_server.py#5576-5704) to track PIDs and allow termination mid-run

The [CTFToolManager](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_server.py#3492-3850) class contains pre-built command templates for 150+ tools:
```python
self.tool_commands = {
    "nmap":      "nmap -sV -sC --top-ports 1000",
    "gobuster":  "gobuster dir -w /usr/share/wordlists/dirb/common.txt -x php,html",
    "sqlmap":    "sqlmap --batch --level 3 --risk 2 --threads 5",
    "hashcat":   "hashcat -m 0 -a 0 --potfile-disable --quiet",
    "ghidra":    "analyzeHeadless /tmp ghidra_project -import",
    # ... 145+ more
}
```

Parameter optimization happens in `IntelligentDecisionEngine.optimize_parameters()`:
```python
# Example: if target is a PHP site, nmap automatically adds PHP-relevant ports
def _optimize_nmap_params(self, profile, context):
    params = {"target": profile.target}
    if profile.target_type == TargetType.WEB_APPLICATION:
        params["scan_type"] = "-sV -sC"
        params["ports"] = "80,443,8080,8443,8000,9000"
    if context.get("stealth"):
        params["additional_args"] = "-T2"  # Slow, quiet scan
    return params
```

---

## 2.5 Use of Async, Subprocess, APIs, and Custom Logic

| Mechanism | Used Where | Purpose |
|-----------|-----------|---------|
| `subprocess.run()` | `PythonEnvironmentManager.install_package()` | pip install in venvs |
| `subprocess.Popen()` | Flask routes for tool execution | Run nmap, gobuster, etc. |
| `threading.Thread` | Process monitoring | Non-blocking progress tracking |
| `threading.Lock` | [ProcessManager](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_server.py#5576-5704), `process_lock` | Thread-safe PID registry |
| `concurrent.futures.ThreadPoolExecutor` | Imported but not heavily used | Parallel tool execution potential |
| `asyncio` + `aiohttp` | Imported in server | Async HTTP scanning (BurpSuite alternative) |
| `selenium.webdriver` | Browser agent tools | Headless Chrome for web crawling |
| `mitmproxy` | MITM proxy integration | HTTP interception and analysis |
| `requests.Session()` | HexStrikeClient | Persistent HTTP connection pool |
| NVD REST API (`requests.get`) | `CVEIntelligenceManager.fetch_latest_cves()` | Real-time CVE data |
| `venv.create()` | [PythonEnvironmentManager](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_server.py#5705-5742) | Python venv management |
| `BeautifulSoup` | Web scraping tools | HTML parsing |

**Async is partially stubbed.** The `asyncio` and `aiohttp` imports exist, but most tool execution is synchronous. The `BurpSuiteAlternative` tool is where async/Selenium is actually used.

---

## 2.6 Limitations, Bugs, and Design Weaknesses

### 🔴 Critical Weaknesses

**1. No Input Sanitization on the Flask API**
```python
# The Flask server likely does something like:
cmd = ["nmap", scan_type, target]  # target = user-supplied!
subprocess.run(cmd, ...)
```
If `additional_args` is passed as a shell string rather than a list, it's vulnerable to **command injection**. Passing `additional_args="--script; rm -rf /"` could be catastrophic.

**2. Monolithic Files — Extreme Technical Debt**
- [hexstrike_server.py](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_server.py) is **17,290 lines** in a single file — unmaintainable
- [hexstrike_mcp.py](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_mcp.py) is **5,471 lines** in a single file
- No module separation, no unit tests, no clear boundaries

**3. Import Errors on Non-Linux Systems**
```python
import mitmproxy          # Requires system dependencies
from selenium import webdriver  # Requires Chrome/ChromeDriver
from pwn import *         # pwntools - Linux-only
import angr               # Binary analysis, massive dependency
```
On Windows or minimal Linux installs, [hexstrike_server.py](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_server.py) **will crash on import** even if you only want to use nmap.

**4. Dead Code Throughout**
- The server imports `ThreadPoolExecutor` but doesn't meaningfully use it for parallel scanning
- `asyncio`/`aiohttp` is imported but most paths are synchronous
- Multiple duplicate class definitions (mentioned in v6.0 changelog: **"Removed duplicate classes"** — but remnants remain)

**5. Loose Color Code Fragments**
```python
# Lines 5691-5703 — These are class body lines floating OUTSIDE any class:
    BG_GREEN = '\033[42m'
    BG_YELLOW = '\033[43m'
    DIM = '\033[2m'
    BLINK = '\033[5m'
```
These lines appear **outside any class scope**, causing a syntax/indentation anomaly in the server file.

**6. No Authentication on Flask API**
The Flask API running on `127.0.0.1:8888` has NO authentication. Any process on the same machine can call `/api/tools/nmap` and execute arbitrary commands.

**7. Technology Detection is URL-Based Only**
```python
def _detect_technologies(self, target):
    if 'wordpress' in target.lower():   # Checks URL string, not HTTP response!
        technologies.append(TechnologyStack.WORDPRESS)
    if '.php' in target.lower():
        technologies.append(TechnologyStack.PHP)
```
This is extremely naive — it checks the URL string, not actual HTTP headers or page content.

**8. Attack Chain Success Probability is Meaningless**
```python
success_prob = effectiveness * profile.confidence_score
# effectiveness = 0.8 (hardcoded table)
# confidence_score = 0.5 base + minor additions
# Result: ~0.4 for most targets — a meaningless hardcoded number
```

**9. Tool Output Not Parsed / Structured**
Nmap results come back as raw [stdout](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_server.py#6799-6809) strings. There's no parsing of nmap XML output, no structured port/service objects. Claude has to read raw text.

**10. HexStrikeClient Tries to Connect at Startup, Continues on Failure**
```python
for i in range(MAX_RETRIES):
    # try to connect
    ...
if not connected:
    logger.error(error_msg)
    # We'll continue anyway...  ← SILENT FAILURE
```
All 100+ tools will silently fail until the Flask server is started.

---

# PHASE 3 — Rebuild Guide From Scratch

> **Goal:** Build a better, cleaner, production-grade version of HexStrike. We'll call it **NexusPen** for reference.

---

## Stage 1 — Basic MCP Setup

### What is MCP?
MCP lets Claude call Python functions as tools. You write Python functions, decorate them, and Claude can call them like: *"Run nmap on this target"* and it invokes your function directly.

### Step 1.1: Install dependencies

```bash
pip install fastmcp flask requests
```

### Step 1.2: Create the simplest possible MCP server

```python
# nexuspen_mcp.py — The MCP server (what Claude talks to)
from mcp.server.fastmcp import FastMCP
import requests

mcp = FastMCP("nexuspen")

BACKEND_URL = "http://127.0.0.1:8888"

def call_backend(endpoint: str, data: dict) -> dict:
    """Helper to call the Flask backend."""
    try:
        resp = requests.post(f"{BACKEND_URL}/{endpoint}", json=data, timeout=300)
        return resp.json()
    except Exception as e:
        return {"success": False, "error": str(e)}

@mcp.tool()
def hello_world(name: str) -> dict:
    """Say hello to a person. Use this to test the MCP connection."""
    return {"success": True, "message": f"Hello, {name}! NexusPen is connected."}

if __name__ == "__main__":
    mcp.run()
```

### Step 1.3: Create the Claude Desktop config

```json
// ~/Library/Application Support/Claude/claude_desktop_config.json  (macOS)
// %APPDATA%\Claude\claude_desktop_config.json  (Windows)
{
  "mcpServers": {
    "nexuspen": {
      "command": "python3",
      "args": ["/absolute/path/to/nexuspen_mcp.py"],
      "timeout": 300,
      "alwaysAllow": []
    }
  }
}
```

**Restart Claude Desktop.** You should now see "nexuspen" in the tools list.

---

## Stage 2 — Adding Custom Tools

### Step 2.1: Add a web recon tool

```python
# nexuspen_mcp.py — Adding real tools

@mcp.tool()
def whois_lookup(domain: str) -> dict:
    """
    Look up WHOIS information for a domain.
    
    Args:
        domain: The domain to look up (e.g. 'example.com')
    
    Returns:
        WHOIS registration data including registrar, dates, nameservers
    """
    return call_backend("api/tools/whois", {"domain": domain})

@mcp.tool()
def check_headers(url: str) -> dict:
    """
    Fetch and analyze HTTP security headers for a URL.
    Checks for missing security headers like CSP, HSTS, X-Frame-Options.
    
    Args:
        url: Full URL to check (e.g. 'https://example.com')
    
    Returns:
        Dict with headers present/missing and security score
    """
    return call_backend("api/tools/check-headers", {"url": url})
```

### Step 2.2: Create the Flask backend

```python
# nexuspen_server.py — The Flask API server

from flask import Flask, request, jsonify
import subprocess
import socket
import requests as req_lib

app = Flask(__name__)

@app.route('/health', methods=['GET'])
def health():
    return jsonify({"status": "ok", "version": "1.0.0"})

@app.route('/api/tools/whois', methods=['POST'])
def run_whois():
    data = request.json
    domain = data.get('domain', '')
    
    # Input validation
    if not domain or not is_valid_domain(domain):
        return jsonify({"success": False, "error": "Invalid domain"})
    
    # Safe subprocess call (list form, NOT shell=True)
    try:
        result = subprocess.run(
            ["whois", domain],          # List form — no shell injection
            capture_output=True,
            text=True,
            timeout=30
        )
        return jsonify({
            "success": result.returncode == 0,
            "stdout": result.stdout,
            "stderr": result.stderr,
            "return_code": result.returncode
        })
    except subprocess.TimeoutExpired:
        return jsonify({"success": False, "error": "Command timed out"})
    except FileNotFoundError:
        return jsonify({"success": False, "error": "whois command not found"})

@app.route('/api/tools/check-headers', methods=['POST'])
def check_security_headers():
    data = request.json
    url = data.get('url', '')
    
    try:
        resp = req_lib.get(url, timeout=10, allow_redirects=True)
        headers = dict(resp.headers)
        
        # Security header checks
        security_checks = {
            "Strict-Transport-Security": "HSTS",
            "Content-Security-Policy": "CSP",
            "X-Frame-Options": "Clickjacking Protection",
            "X-Content-Type-Options": "MIME Sniffing Protection",
            "Referrer-Policy": "Referrer Policy",
        }
        
        missing = []
        present = []
        for header, name in security_checks.items():
            if header in resp.headers:
                present.append(name)
            else:
                missing.append(name)
        
        score = int((len(present) / len(security_checks)) * 100)
        
        return jsonify({
            "success": True,
            "url": url,
            "status_code": resp.status_code,
            "headers": headers,
            "security_score": score,
            "headers_present": present,
            "headers_missing": missing
        })
    except Exception as e:
        return jsonify({"success": False, "error": str(e)})

def is_valid_domain(domain: str) -> bool:
    """Basic domain validation to prevent injection."""
    import re
    # Only allow alphanumeric, hyphens, dots
    return bool(re.match(r'^[a-zA-Z0-9][a-zA-Z0-9\-\.]{0,253}[a-zA-Z0-9]$', domain))

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=8888, debug=False)
```

---

## Stage 3 — Integrating System Tools (Nmap, etc.)

### The Golden Rule: NEVER use `shell=True`

```python
# ❌ WRONG — Shell injection vulnerability
subprocess.run(f"nmap {scan_type} {target}", shell=True)
# If target = "example.com; rm -rf /" → DISASTER

# ✅ CORRECT — Arguments as a list, shell=False (default)
subprocess.run(["nmap", scan_type, target], capture_output=True, text=True)
```

### Step 3.1: A Safe, Reusable Tool Runner

```python
# nexuspen_server.py — Safe tool runner

import subprocess
import shutil
import time
from typing import List, Optional

def run_tool(
    command: List[str],
    timeout: int = 300,
    input_data: Optional[str] = None
) -> dict:
    """
    Safely run an external security tool.
    
    Args:
        command: Command as list of strings ['nmap', '-sV', 'target']
        timeout: Max seconds to wait
        input_data: Optional stdin data
    
    Returns:
        Dict with success, stdout, stderr, return_code, execution_time
    """
    # Check tool exists before running
    tool_name = command[0]
    if not shutil.which(tool_name):
        return {
            "success": False,
            "error": f"Tool '{tool_name}' not found. Install it first.",
            "stdout": "",
            "stderr": "",
            "return_code": -1
        }
    
    start_time = time.time()
    
    try:
        result = subprocess.run(
            command,                    # List form — safe
            capture_output=True,
            text=True,
            timeout=timeout,
            input=input_data
        )
        
        execution_time = time.time() - start_time
        
        return {
            "success": result.returncode == 0,
            "stdout": result.stdout,
            "stderr": result.stderr,
            "return_code": result.returncode,
            "execution_time": execution_time,
            "command": " ".join(command)
        }
    
    except subprocess.TimeoutExpired:
        return {
            "success": False,
            "error": f"Command timed out after {timeout}s",
            "stdout": "",
            "stderr": "",
            "return_code": -1,
            "execution_time": timeout
        }
    except Exception as e:
        return {
            "success": False,
            "error": str(e),
            "stdout": "",
            "stderr": "",
            "return_code": -1
        }


# Using the safe runner for real tools:

@app.route('/api/tools/nmap', methods=['POST'])
def run_nmap():
    data = request.json
    target = data.get('target', '')
    scan_type = data.get('scan_type', '-sV')
    ports = data.get('ports', '')
    
    # Validate target
    if not is_valid_target(target):
        return jsonify({"success": False, "error": "Invalid target"})
    
    # Validate scan_type (whitelist approach)
    allowed_scan_types = ['-sV', '-sC', '-sS', '-sT', '-sU', '-sV -sC', '-A', '-p-']
    if scan_type not in allowed_scan_types:
        scan_type = '-sV'  # Default to safe option
    
    # Build command as list (NEVER shell=True)
    command = ["nmap", scan_type]
    if ports:
        command.extend(["-p", ports])
    command.append(target)
    
    result = run_tool(command, timeout=300)
    return jsonify(result)


def is_valid_target(target: str) -> bool:
    """Validate target is an IP or domain, not a command injection."""
    import re
    # IP address pattern
    ip_pattern = r'^(\d{1,3}\.){3}\d{1,3}(/\d{1,2})?$'
    # Domain pattern  
    domain_pattern = r'^[a-zA-Z0-9][a-zA-Z0-9\-\.]{0,253}[a-zA-Z0-9]$'
    # URL pattern
    url_pattern = r'^https?://[a-zA-Z0-9\-\.\:\/\?\=\&\#\_\%]+$'
    
    return bool(
        re.match(ip_pattern, target) or
        re.match(domain_pattern, target) or
        re.match(url_pattern, target)
    )
```

---

## Stage 4 — Adding LLM Decision Layer

### What This Means
Instead of fully hardcoded attack chains, you use an LLM API (or let Claude itself decide) what to do next based on previous scan results.

### Option A: Let Claude Decide (Recommended — No Extra LLM Calls)
Since Claude IS the AI, just structure your tool results well. Claude will decide the next action:

```python
# Make tool results rich and interpretable
@app.route('/api/tools/nmap', methods=['POST'])
def run_nmap():
    # ... run nmap ...
    
    # Parse nmap output into structured data
    ports = parse_nmap_output(result['stdout'])
    
    return jsonify({
        "success": True,
        "raw_output": result['stdout'],
        "structured": {
            "open_ports": ports,
            "services": extract_services(ports),
            "recommendations": generate_recommendations(ports),  # Simple rules
        }
    })

def generate_recommendations(ports: list) -> list:
    """Rule-based next-step suggestions (Claude reads these and decides)."""
    recommendations = []
    open_port_nums = [p['port'] for p in ports]
    
    if 80 in open_port_nums or 443 in open_port_nums:
        recommendations.append("HTTP service detected → run gobuster_scan for directories")
        recommendations.append("HTTP service detected → run nikto_scan for vulnerabilities")
    if 22 in open_port_nums:
        recommendations.append("SSH detected → check for weak credentials with hydra")
    if 445 in open_port_nums:
        recommendations.append("SMB detected → run enum4linux for share enumeration")
    
    return recommendations
```

### Option B: Add an LLM API Call in the Decision Engine

```python
# nexuspen_server.py — Smart Attack Chain with LLM API

import os
import json

OPENAI_KEY = os.environ.get('OPENAI_API_KEY')

def get_llm_next_action(scan_results: dict, objective: str) -> dict:
    """Ask an LLM what to do next based on scan results."""
    
    if not OPENAI_KEY:
        return {"action": "manual", "reason": "No LLM API key configured"}
    
    prompt = f"""You are a penetration testing AI assistant.
    
Objective: {objective}
Previous scan results: {json.dumps(scan_results, indent=2)}

Based on these results, what single tool should be run next?
Respond in JSON: {{"tool": "tool_name", "parameters": {{}}, "reason": "..."}}
Available tools: nmap, gobuster, nikto, sqlmap, nuclei, hydra, enum4linux"""
    
    try:
        import openai
        client = openai.OpenAI(api_key=OPENAI_KEY)
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"}
        )
        return json.loads(response.choices[0].message.content)
    except Exception as e:
        return {"action": "error", "reason": str(e)}
```

---

## Stage 5 — Adding User Approval Layer

### Why This Matters
Some tools (like [sqlmap](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_server.py#11016-11048), [hydra](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_server.py#11093-11144), [metasploit](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_server.py#11049-11092)) are destructive. You want Claude to *ask* before running them, with a configurable approval gate.

### MCP `alwaysAllow` Config
The [hexstrike-ai-mcp.json](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike-ai-mcp.json) has this:
```json
"alwaysAllow": []   // Empty = Claude asks user for EVERY tool call
```
If you add tool names here, Claude runs them without asking:
```json
"alwaysAllow": ["nmap_scan", "check_headers", "whois_lookup"]  // Auto-run safe tools
// But NOT: hydra_attack, sqlmap_scan, metasploit_run
```

### Building an HTTP Approval Gate in Python

```python
# approval_server.py — Terminal approval gate

import threading
import queue
import time

approval_queue = queue.Queue()
response_queue = queue.Queue()

class UserApprovalGate:
    """
    Intercept high-risk tool calls and require user approval.
    
    How it works:
    - Tool calls hit /api/approve endpoint
    - Server waits for human to type 'yes' or 'no' in terminal
    - Response returned to Claude
    """
    
    HIGH_RISK_TOOLS = {
        "sqlmap", "hydra", "metasploit", "hashcat",
        "nuclei",  # Change as needed
    }
    
    def requires_approval(self, tool_name: str) -> bool:
        return tool_name in self.HIGH_RISK_TOOLS
    
    def request_approval(self, tool_name: str, params: dict, timeout: int = 60) -> bool:
        """
        Display prompt to user and wait for approval.
        
        Returns True if approved, False if denied or timeout.
        """
        print(f"\n{'='*60}")
        print(f"⚠️  APPROVAL REQUIRED")
        print(f"Tool: {tool_name}")
        print(f"Parameters:")
        for key, val in params.items():
            print(f"  {key}: {val}")
        print(f"{'='*60}")
        
        # Non-blocking input with timeout
        answer_event = threading.Event()
        answer = [False]  # Use list for mutable closure
        
        def get_input():
            try:
                user_input = input("Approve? [y/N]: ").strip().lower()
                answer[0] = user_input in ('y', 'yes')
            except:
                answer[0] = False
            answer_event.set()
        
        t = threading.Thread(target=get_input, daemon=True)
        t.start()
        
        if answer_event.wait(timeout=timeout):
            return answer[0]
        else:
            print(f"\n⏰ Timeout after {timeout}s — Denied by default")
            return False

# Integration in Flask route:
approval_gate = UserApprovalGate()

@app.route('/api/tools/sqlmap', methods=['POST'])
def run_sqlmap():
    data = request.json
    url = data.get('url', '')
    
    # Check if approval needed
    if approval_gate.requires_approval("sqlmap"):
        approved = approval_gate.request_approval("sqlmap", {"url": url})
        if not approved:
            return jsonify({
                "success": False, 
                "error": "Denied by user approval gate",
                "human_escalation": True
            })
    
    # Proceed with execution
    command = ["sqlmap", "--batch", "--level=2", "--risk=1", "-u", url]
    result = run_tool(command, timeout=600)
    return jsonify(result)
```

### Web-Based Approval UI (Alternative)

For a better UX, create a simple web page that pops up for approval:

```python
# Flask endpoint for web-based approval
@app.route('/api/approve', methods=['POST'])
def request_approval():
    """Called by automated tools to request human approval."""
    data = request.json
    # Store approval request in database/memory
    # Return approval_id for polling
    approval_id = str(uuid.uuid4())
    pending_approvals[approval_id] = {
        "tool": data['tool'],
        "params": data['params'],
        "status": "pending",
        "created_at": time.time()
    }
    return jsonify({"approval_id": approval_id})

@app.route('/api/approve/<approval_id>/respond', methods=['POST'])
def respond_to_approval(approval_id):
    """Called by human via web UI to approve/deny."""
    data = request.json
    approved = data.get('approved', False)
    if approval_id in pending_approvals:
        pending_approvals[approval_id]['status'] = 'approved' if approved else 'denied'
    return jsonify({"success": True})
```

---

## Security Best Practices

### 1. Command Injection Prevention

```python
# RULE: Always use list form for subprocess, NEVER shell=True
# RULE: Always validate and whitelist inputs

import re
import shlex

def sanitize_target(target: str) -> str:
    """Strip dangerous characters from target input."""
    # Only allow: alphanumeric, dots, hyphens, slashes, colons
    sanitized = re.sub(r'[^a-zA-Z0-9.\-/_:@]', '', target)
    return sanitized[:253]  # Max domain length

def whitelist_args(args: str, allowed_flags: list) -> list:
    """Parse additional_args and only keep whitelisted flags."""
    parsed = shlex.split(args)
    safe = []
    for arg in parsed:
        for allowed in allowed_flags:
            if arg.startswith(allowed):
                safe.append(arg)
                break
    return safe

# Example: nmap additional_args whitelist
NMAP_ALLOWED_FLAGS = ['-T', '-p', '--top-ports', '--script=', '-v', '-n', '-Pn']
safe_args = whitelist_args(user_additional_args, NMAP_ALLOWED_FLAGS)
```

### 2. Scalability — Process Pool & Queue

```python
# Use a job queue to prevent resource exhaustion

from concurrent.futures import ThreadPoolExecutor
import queue

job_queue = queue.Queue(maxsize=50)  # Max 50 queued jobs
executor = ThreadPoolExecutor(max_workers=5)  # Max 5 concurrent scans

def submit_scan_job(tool_fn, *args, **kwargs):
    """Submit a tool execution job to the pool."""
    if job_queue.full():
        return {"success": False, "error": "Server busy, max queue size reached"}
    
    future = executor.submit(tool_fn, *args, **kwargs)
    return {"success": True, "job_id": id(future), "status": "queued"}
```

### 3. Clean Code Structure (Modular Architecture)

```
nexuspen/
├── server/
│   ├── __init__.py
│   ├── app.py              ← Flask app factory
│   ├── config.py           ← All configuration
│   └── routes/
│       ├── __init__.py
│       ├── network.py      ← nmap, masscan, rustscan routes
│       ├── web.py          ← gobuster, ffuf, nikto routes
│       ├── cloud.py        ← prowler, trivy, checkov routes
│       ├── password.py     ← hydra, john, hashcat routes
│       └── files.py        ← file management routes
│
├── mcp/
│   ├── __init__.py
│   ├── client.py           ← NexusPenClient (HTTP client)
│   ├── server.py           ← FastMCP setup
│   └── tools/
│       ├── network.py      ← @mcp.tool() network tools
│       ├── web.py          ← @mcp.tool() web tools
│       └── cloud.py        ← @mcp.tool() cloud tools
│
├── core/
│   ├── __init__.py
│   ├── runner.py           ← Safe subprocess runner
│   ├── validator.py        ← Input validation
│   ├── approval.py         ← User approval gate
│   └── decision.py         ← Attack chain logic
│
├── tests/
│   ├── test_runner.py
│   ├── test_validator.py
│   └── test_mcp.py
│
├── config.json             ← MCP config for Claude Desktop
├── requirements.txt
└── README.md
```

---

# BONUS — Improvements, Features & Converting Tools

## Improvements Over HexStrike

| Area | HexStrike Problem | Recommended Fix |
|------|------------------|-----------------|
| **Security** | No input sanitization; shell=True risk | Strict validation + list-form subprocess |
| **Modularity** | 17K line monilith | Split into modules (routes/, tools/, core/) |
| **Output Parsing** | Raw stdout strings | Parse nmap XML, gobuster JSON, nuclei JSON |
| **Error Handling** | Hardcoded heuristics | Real error parsing with exit codes + stderr |
| **Tool Detection** | URL-string based only | HTTP HEAD request + header/content analysis |
| **Auth** | No Flask API auth | Add API key or local UNIX socket |
| **Tests** | Zero test files | Unit test every validator and runner |
| **Async** | Sync with async imports | Either go fully async (aiohttp server) or remove dead imports |
| **Progress** | Complex but broken | Use NDJSON streaming or WebSockets |
| **Docs** | Docstrings only | OpenAPI spec for Flask, type stubs for MCP |

---

## Features For a Production-Level Version

### 1. Structured Output Parsing
```python
# Instead of raw stdout, parse tool output:
import xml.etree.ElementTree as ET

def parse_nmap_xml(xml_output: str) -> dict:
    """Parse nmap -oX output into structured data."""
    root = ET.fromstring(xml_output)
    hosts = []
    for host in root.findall('host'):
        ports = []
        for port in host.findall('.//port'):
            service = port.find('service')
            ports.append({
                "port": int(port.get('portid')),
                "protocol": port.get('protocol'),
                "state": port.find('state').get('state'),
                "service": service.get('name') if service else None,
                "version": service.get('version') if service else None,
            })
        hosts.append({"ports": ports})
    return {"hosts": hosts}
```

### 2. Real-Time Streaming Output
```python
# Use Server-Sent Events for live tool output
@app.route('/api/tools/nmap/stream', methods=['POST'])
def stream_nmap():
    def generate():
        process = subprocess.Popen(cmd, stdout=PIPE, stderr=STDOUT, text=True)
        for line in process.stdout:
            yield f"data: {json.dumps({'line': line.strip()})}\n\n"
        process.wait()
        yield f"data: {json.dumps({'done': True, 'returncode': process.returncode})}\n\n"
    
    return Response(generate(), mimetype='text/event-stream')
```

### 3. Scan History & Report Database
```python
# Store every scan result in SQLite for audit trail and report generation
import sqlite3

def save_scan_result(tool: str, target: str, result: dict):
    conn = sqlite3.connect('nexuspen.db')
    conn.execute("""
        INSERT INTO scans (tool, target, success, stdout, execution_time, timestamp)
        VALUES (?, ?, ?, ?, ?, ?)
    """, (tool, target, result['success'], result['stdout'], 
          result.get('execution_time', 0), time.time()))
    conn.commit()
```

### 4. Vulnerability Deduplication
Merge findings from multiple tools (nmap finds port 80, gobuster finds /admin, nuclei finds CVE-2024-xxxx on /admin) into a unified vulnerability object with full context.

### 5. Rate Limiting & Scope Management
```python
# Prevent accidental scanning of out-of-scope targets
ALLOWED_CIDR = ["192.168.1.0/24", "10.0.0.0/8"]

def is_in_scope(target: str) -> bool:
    import ipaddress
    try:
        ip = ipaddress.ip_address(target)
        return any(ip in ipaddress.ip_network(cidr) for cidr in ALLOWED_CIDR)
    except ValueError:
        return target in ALLOWED_DOMAINS
```

---

## How to Convert Existing Python Tools Into MCP Tools

Any existing Python script or tool can become an MCP tool in 3 steps:

### Step 1: Wrap the logic in a Flask route (if it's complex)

```python
# Your existing script:
# my_scanner.py
def scan_headers(url):
    resp = requests.get(url, timeout=10)
    return {"headers": dict(resp.headers), "status": resp.status_code}

# Add it to Flask:
@app.route('/api/tools/my-scanner', methods=['POST'])
def run_my_scanner():
    from my_scanner import scan_headers
    url = request.json.get('url')
    result = scan_headers(url)
    return jsonify({"success": True, **result})
```

### Step 2: Add an MCP tool wrapper

```python
# In nexuspen_mcp.py:
@mcp.tool()
def my_scanner(url: str) -> dict:
    """
    Scan HTTP headers for a target URL.
    Checks status code and returns all response headers.
    
    Args:
        url: The full URL to scan (e.g. 'https://example.com')
    
    Returns:
        Dict with headers, status_code
    """
    return call_backend("api/tools/my-scanner", {"url": url})
```

### Step 3: (Optional) Wrap a CLI tool

```python
# Convert any CLI tool into MCP:
@mcp.tool()
def my_python_tool(target: str, verbose: bool = False) -> dict:
    """Run my-python-tool against a target."""
    return call_backend("api/tools/my-python-tool", {
        "target": target,
        "verbose": verbose
    })

# On the Flask side:
@app.route('/api/tools/my-python-tool', methods=['POST'])
def run_my_python_tool():
    data = request.json
    command = ["python3", "/opt/my_tool/main.py", "--target", data['target']]
    if data.get('verbose'):
        command.append("--verbose")
    result = run_tool(command)
    return jsonify(result)
```

### The 3 Rules for Good MCP Tool Design

1. **Write a great docstring** — Claude reads it to decide when and how to use the tool. Include: what it does, when to use it, what each arg means, what the return looks like.

2. **Return structured JSON** — Don't return raw stdout. Parse it into structured fields. Claude can reason about `{"open_ports": [80, 443]}` much better than a 500-line nmap string.

3. **Always return `success: bool`** — Claude checks this first. On failure, include a helpful [error](file:///d:/Programming/Web%20App/Rudraksh_Agi.in/hexstrike-ai/hexstrike_server.py#1961-1982) field that explains what went wrong and what to try instead.

---

## Quick Compliance Checklist for Production

- [ ] All subprocesses use list form (never `shell=True`)
- [ ] All user inputs validated and sanitized
- [ ] Flask API requires authentication (API key / local socket)
- [ ] Scope management: out-of-scope targets rejected
- [ ] User approval gate for destructive tools
- [ ] Every scan result saved to audit database
- [ ] Tool output parsed into structured JSON (not raw stdout)
- [ ] Error handling tested for every tool (tool not found, timeout, permission denied)
- [ ] Unit tests for validators and runners
- [ ] Rate limiting on Flask API to prevent abuse

---

*Analysis completed: March 22, 2026 | HexStrike AI v6.0 | 22,761 lines analyzed*
