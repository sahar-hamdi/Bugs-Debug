# **General - Bugs & Resolutions Log**
all the bugs I faced while working on a QA Test Cases Generator using Ollama llama model



This document tracks the technical challenges, bugs, and architectural limitations encountered during the development of the EEBus QA Test Case Generation pipeline, and any other systems I work on along with their solutions.

### **1. Environment & Infrastructure Bugs**

**1.1. RuntimeError: Event loop is closed (Windows asyncio Crash)**

Description: The pipeline crashed at exactly 100% completion of the embedding phase during Knowledge Graph extraction.

Root Cause: The default asynchronous event loop policy on Windows (ProactorEventLoop) conflicts with heavy async batching used by llama_index, httpx, and tqdm_asyncio.

Solution: Enforced the Windows Selector Event Loop policy at the absolute beginning of the execution script (main.py / tc_pipeline_test.py) before any other library imports.

import sys
import asyncio
if sys.platform == "win32":
    asyncio.set_event_loop_policy(asyncio.WindowsSelectorEventLoopPolicy())


**1.2. VS Code Debugger: Timed out waiting for debuggee to spawn**

Description: Attempting to run the VS Code Python Debugger (F5) resulted in a timeout popup.

Root Cause: VS Code was attempting to launch the debugger inside a Git Bash (MINGW64) terminal, which frequently hangs or fails to attach the Python debugger on Windows.

Solution: Changed the default VS Code terminal profile. Accessed Ctrl + Shift + P -> Terminal: Select Default Profile and set it to Command Prompt (CMD).

**1.3. VS Code Debugger: ModuleNotFoundError: No module named 'llama_index'**

Description: The debugger launched successfully but immediately crashed, claiming core libraries were missing, despite them being installed in the virtual environment.

Root Cause: The VS Code debugger was using the global Windows Python interpreter instead of the project's virtual environment (venv).

Solution: Accessed Ctrl + Shift + P -> Python: Select Interpreter and explicitly selected the ('venv': venv) interpreter.

**1.4. httpx.ReadTimeout: timed out during LLM Call**

Description: The script crashed during the Negative Test Case generation step with a ReadTimeout error.

Root Cause: Modifying the prompt to generate an array of multiple test cases significantly increased the LLM's processing and inference time, exceeding the default HTTP client timeout threshold.

Solution: Increased the request timeout parameter in config.py to allow the local Ollama model sufficient time to complete large JSON generations.

LLM_REQUEST_TIMEOUT = 600.0  # Increased to 10 minutes


### **2. Pipeline Logic & Prompt Engineering Bugs**

**2.1. Valid Requirements Skipped (Heuristic Filter Failure)**

Description: The initial chunk filtering mechanism skipped chunks that contained valid Requirement IDs (e.g., [OHPCF-011]) because they lacked explicit keywords like "SHALL" or "SHOULD".

Root Cause: The gatekeeper logic was too strict, relying solely on text keywords.

Solution: Enhanced the filtering logic to act as a dual-gate system using Regex. Chunks are now processed if they contain EITHER a keyword OR a valid Requirement ID format.

**2.2. Monolithic and Non-Atomic Test Cases**

Description: The LLM generated a single, massive test case per chunk that attempted to combine Phase A, Phase B, and Phase C into one flow.

Root Cause: The prompt asked for "ONE positive test case" for the entire chunk_text, causing the LLM to summarize the entire section rather than testing individual atomic requirements.

Solution: Implemented the extract_requirement_sentences Python function to split the chunk by periods (.), identify sentences containing IDs, and pass these targeted sentences to the LLM. Updated the prompt to output a JSON array of separate test cases for each distinct requirement.

**2.3. Internal State Violations (Failed QA Review)**

Description: The LLM generated test cases that checked the internal phase of the system (e.g., "Expected result: Compressor is in Phase A"). The QA Reviewer agent rejected these for violating Black-Box testing principles.

Root Cause: The LLM was directly mirroring the technical specifications without translating internal states into external, observable behaviors.

Solution: Injected a strict, high-priority directive into both Positive and Negative prompts:

"IMPORTANT: Never reference internal states, phases, or implementation details. Only describe external observable behavior — messages sent, messages received, timeouts, or absence of expected messages."

### **3. Current Architectural Limitations**

**3.1. Context Fragmentation (Ongoing Challenge)**

* Description: The QA Reviewer Agent is rejecting test cases because they are logically flawed or test the wrong actor.

* Root Cause: By feeding the LLM isolated sentences to force atomic test cases, the LLM loses the Global Context (preconditions, overarching state machines, and actor definitions defined earlier in the chunk/document).

* Proposed Solution: Refactor the architecture to a Context-Aware approach. The pipeline must pass the full chunk text as background context to the LLM, while explicitly designating the extracted sentence as the specific target for test generation.


### 4. MLflow Errors

**4.1 OSError: [WinError 10022] An invalid argument was supplied**

* Cause: MLflow's newer server runs on uvicorn with 4 worker processes by default. Windows doesn't support multiple processes sharing the same listening socket the way uvicorn tries to do it.
* Fix: Force a single worker:
  
<img width="870" height="113" alt="image" src="https://github.com/user-attachments/assets/f9012b16-2c9e-48f7-8a48-d983a01ebd30" />

**4.2 Runs logged in notebook don't appear in the UI at all (client/server store mismatch)**

* Cause: As of MLflow 3.7+, the server default (when no --backend-store-uri is passed) is sqlite:///mlflow.db, but the Python client default (when mlflow.set_tracking_uri() isn't called) is the local ./mlruns folder. Notebook and UI end up reading from two different stores.
* Fix: Never rely on defaults — set the same URI explicitly in both places:

  <img width="852" height="218" alt="image" src="https://github.com/user-attachments/assets/72c18518-2a03-4b9f-ac48-0568c8a11dbe" />

**4.3 mlflow.set_tracking_uri() called after mlflow.set_experiment()**

* Cause: set_experiment() queries the backend immediately to find/create the experiment. If it's called before set_tracking_uri(), it uses whatever the default backend is at that moment — creating the experiment in the wrong store.
* Fix: Order matters — always:

  <img width="847" height="150" alt="image" src="https://github.com/user-attachments/assets/9f6fced3-246a-4ce5-9dc0-bc35b3bdc2cb" />


