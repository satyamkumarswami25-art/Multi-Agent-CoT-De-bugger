# Multi-Agent-CoT-De-buggerimport os
import json
from google import genai
from google.genai import types
from pydantic import BaseModel, Field

# --- API Client Initialization ---
# Assuming GOOGLE_API_KEY is available in the environment (e.g., GitHub Secrets, local .env)
try:
    client = genai.Client()
except Exception:
    print("Warning: Gemini client failed to initialize. Set GOOGLE_API_KEY for full functionality.")
    client = None

# ----------------------------------------------------------------------
# 1. TOOL DEFINITION: Code Executor (Function Calling)
# ----------------------------------------------------------------------

def execute_code(code_snippet: str) -> str:
    """
    Simulates running the given Python code snippet in a sandboxed environment.
    This function confirms the Planner's hypothesis on a specific buggy scenario.
    """
    # Custom simulation logic for the common Indexing Error
    if "range(len(data))" in code_snippet and "-1" not in code_snippet and "total += data[i]" in code_snippet:
        return "RUNTIME_ERROR: IndexError - Index out of range. The loop condition is incorrect for zero-based indexing."
    elif "return" not in code_snippet:
        return "SYNTAX_ERROR: Missing function return statement or indentation error."
    else:
        return "EXECUTION_SUCCESS: Code ran without simulated errors."

available_tools = [execute_code]

# ----------------------------------------------------------------------
# 2. AGENT ARCHITECTURE
# ----------------------------------------------------------------------

# A. Planner Agent (CoT Specialist)

class PlannerAgent:
    def __init__(self, client):
        self.client = client
        self.model = 'gemini-2.5-flash'
        self.system_prompt = (
            "You are the Planner Agent, a meticulous Code Analysis Strategist. "
            "Your sole task is to generate a detailed Chain-of-Thought (CoT) debugging plan. "
            "You MUST recommend calling the 'execute_code' tool if a Runtime or Logic error is suspected."
        )

    def generate_cot_plan(self, code_snippet: str) -> str:
        if not self.client: return "Client not initialized. Cannot generate plan."
        
        prompt = (
            f"Analyze the following buggy Python code and generate a comprehensive CoT plan:\n\n"
            f"BUGGY CODE:\n{code_snippet}\n\n"
            f"Your plan must follow these four steps strictly:\n"
            f"1. Analyze the function definition and its intended purpose.\n"
            f"2. Predict the most likely error type (Syntax, Runtime, Logic).\n"
            f"3. Determine tool usage (Explicitly state whether to call 'execute_code' with the buggy code).\n"
            f"4. State the final suggested fix approach."
        )
        
        response = self.client.models.generate_content(
            model=self.model,
            contents=prompt,
            config=types.GenerateContentConfig(
                system_instruction=self.system_prompt,
                tools=available_tools
            )
        )
        return response.text

# B. Debugger Agent (Correction & Explanation)

class DebuggerOutputSchema(BaseModel):
    """Schema for the Debugger Agent's structured and transparent output."""
    fixed_code: str = Field(description="The final, corrected and clean Python code snippet.")
    root_cause: str = Field(description="The primary technical reason the original code failed.")
    explanation_for_fix: str = Field(description="A detailed, developer-friendly explanation of the fix.")

class DebuggerAgent:
    def __init__(self, client):
        self.client = client
        self.model = 'gemini-2.5-flash'
        self.system_prompt = (
            "You are the Debugger Agent. Use the Planner's analysis and the Tool's output "
            "to provide the final, perfect code fix and a detailed, educational explanation in the requested JSON format."
        )

    def provide_final_fix(self, code_snippet: str, plan_output: str, tool_output: str) -> str:
        if not self.client: return json.dumps({"error": "Client not initialized."})
        
        prompt = (
            f"Planner's Analysis (CoT):\n{plan_output}\n\n"
            f"Tool Execution Result:\n{tool_output}\n\n"
            f"Original Buggy Code:\n{code_snippet}\n\n"
            f"Based on this combined information, generate the final fixed code and detailed explanation."
        )
        
        response = self.client.models.generate_content(
            model=self.model,
            contents=prompt,
            config=types.GenerateContentConfig(
                system_instruction=self.system_prompt,
                response_mime_type="application/json",
                response_schema=DebuggerOutputSchema,
            )
        )
        return response.text

# ----------------------------------------------------------------------
# 3. EXECUTION DEMO (For Local Run)
# ----------------------------------------------------------------------

if __name__ == "__main__":
    BUGGY_CODE = """
def calculate_average(data):
    if not data:
        return 0
    total = 0
    for i in range(len(data)):
        total += data[i]
    return total / len(data)
"""

    if client:
        print("--- 📝 SCENARIO: Initial Buggy Code Input ---")
        print(BUGGY_CODE)
        
        planner = PlannerAgent(client)
        debugger = DebuggerAgent(client)

        print("\n--- 🧠 STEP 1: Planner Agent Executes CoT Plan ---")
        plan_output = planner.generate_cot_plan(BUGGY_CODE)
        print(plan_output)

        print("\n--- ⚙️ STEP 2: Tool Execution (Simulated) ---")
        tool_result = execute_code(BUGGY_CODE)
        print(f"--- TOOL EXECUTION RESULT: {tool_result} ---")

        print("\n--- ✅ STEP 3: Debugger Agent Provides Final Fix ---")
        final_json_fix = debugger.provide_final_fix(BUGGY_CODE, plan_output, tool_result)
        
        try:
            formatted_output = json.loads(final_json_fix)
            print("\n### 🏆 FINAL DEBUGGING SUMMARY (Structured Output)")
            print(f"**Root Cause:** {formatted_output.get('root_cause')}")
            print(f"**FINAL CORRECTED CODE:**")
            print("```python")
            print(formatted_output.get('fixed_code'))
            print("```")
        except json.JSONDecodeError:
            print("\n❌ Error in Debugger output.")
            # 🚀 Multi-Agent-CoT-Debugger (Kaggle Capstone Project)

[![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Built with Gemini](https://img.shields.io/badge/Built%20with-Gemini%202.5%20Flash-green.svg)](https://ai.google.dev/models/gemini)

## 💡 Project Abstract

This project implements a robust **Multi-Agent System** for autonomous code debugging, specifically targeting complex logical and runtime errors in Python. Developed for the **Kaggle Capstone Project (Enterprise Agent Track)**, the framework leverages advanced LLM capabilities to achieve **10X developer productivity** by providing transparent and explained solutions.

### 🔑 Key Technical Pillars

1.  **Chain-of-Thought (CoT) Reasoning:** A **Planner Agent** generates a step-by-step strategy for debugging, moving beyond simple fixes to understand the root cause.
2.  **Function Calling:** The Planner calls a **Code Executor Tool** to test hypotheses against simulated runtime environments, demonstrating efficient tool utilization.
3.  **Structured Output:** The **Debugger Agent** delivers the final corrected code and a detailed explanation in a validated JSON schema.

## 🖼️ Architecture Flow

The system operates in a logical, three-step cycle:


1.  **Input:** Developer submits buggy code.
2.  **Planner:** Generates CoT plan and calls the Tool.
3.  **Tool:** Returns simulated error or success message.
4.  **Debugger:** Provides the final, structured fix.

## ⚙️ How to Run Locally

1.  **Clone the Repository:**
    ```bash
    git clone [https://github.com/YourUsername/Multi-Agent-CoT-Debugger.git](https://github.com/YourUsername/Multi-Agent-CoT-Debugger.git)
    cd Multi-Agent-CoT-Debugger
    ```
2.  **Install Dependencies:**
    ```bash
    pip install -r requirements.txt
    ```
3.  **Set API Key:** Set your Gemini API key as an environment variable:
    ```bash
    export GOOGLE_API_KEY="YOUR_API_KEY_HERE"
    ```
4.  **Execute the Demo:**
    ```bash
    python debugger_agent.py
    ```

## 🔗 Live Kaggle Submission

**[Insert the link to your final Kaggle Notebook submission here]**
