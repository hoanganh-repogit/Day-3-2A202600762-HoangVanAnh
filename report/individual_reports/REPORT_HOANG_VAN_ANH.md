# Individual Report: Lab 3 - Chatbot vs ReAct Agent

- **Student Name**: Hoàng Văn Anh
- **Student ID**: 2A202600762
- **Date**: 01/06/2026

---

## I. Technical Contribution (15 Points)

In this laboratory work, I developed the premium interactive Web UI interface and refined the core ReAct agent reasoning flow to guarantee factual accuracy and eliminate random database hallucinations.

- **Modules Implemented**:
  * `app.py`: Created a comprehensive web application utilizing Gradio. The UI provides a soft slate/blue aesthetic, a premium split-column layout, interactive query templates, and dedicated tabs for the **ReAct Research Agent**, **Academic Text Polisher**, and **Citation Formatter**. I later simplified the interface by removing the reasoning console trace accordion to center the user experience entirely on clean scientific Markdown outputs.
  * `src/tools/academic_tools.py`: Improved the mock database fallback query filtering logic. By restructuring the keyword matching in `_search_mock_database` to only return papers with an actual match score > 0, we completely eliminated irrelevant mock paper injections (such as showing deep learning papers when searching for unrelated topics like sound detection or medical queries) when live APIs are rate-limited or offline.
  * `src/core/openai_provider.py`: Integrated a strict `temperature=0.0` configuration inside the OpenAI Chat Completion endpoint, forcing highly precise, deterministic reasoning steps and preventing the LLM from inventing non-existent studies.
  * `src/agent/agent.py`: Restructured the prompt engineering inside `get_system_prompt` to force the agent to format all retrieved scientific studies using a strict Vietnamese layout (Tên Paper, Năm công bố, Đường dẫn, Tóm tắt).

- **Code Highlights**:
  * *Mock Fallback Precision Filtering (`src/tools/academic_tools.py`)*:
    ```python
    scored_papers = []
    for paper in MOCK_DATABASE:
        score = 0
        text_to_search = (paper["title"] + " " + paper["abstract"]).lower()
        for word in query_words:
            if word in text_to_search:
                score += 1
        scored_papers.append((score, paper))
    
    scored_papers.sort(key=lambda x: x[0], reverse=True)
    matched = [paper for score, paper in scored_papers if score > 0]
    return matched[:limit]
    ```
  * *System Prompt Structure Enforcement (`src/agent/agent.py`)*:
    ```python
    6. When presenting papers in the 'Final Answer', you MUST present each paper systematically and beautifully in Vietnamese using the following structure:
       ### 📄 [Tên Paper]
       * **Năm công bố**: [Năm phát hành paper]
       * **Đường dẫn**: [Đường dẫn link paper / PDF URL]
       * **Tóm tắt**: [Tóm tắt nội dung bài viết một cách ngắn gọn, súc tích]
    ```

- **Documentation**:
  * Authored `GRADIO_GUIDE.md` detailing the simplified Web UI architecture, visual layouts, Mermaid dataflows, and custom run commands. I also embedded a web recording demonstrating the high-accuracy simplified UI in action.

---

## II. Debugging Case Study (10 Points)

During the evaluation of the agent under heavy network traffic, the external academic APIs experienced timeouts and rate limits, causing the agent to get trapped in an infinite retry loop.

- **Problem Description**:
  When the arXiv API timed out or the Semantic Scholar API returned a `429 Too Many Requests` status, the agent would repeatedly try to call `search_arxiv` or `search_semantic_scholar` with the exact same query, believing that a retry would resolve the transient issue. This resulted in an infinite loop that exhausted LLM token budgets and hung the server.

- **Log Source** (from `logs/2026-06-01.log`):
  ```json
  {"timestamp": "2026-06-01T08:43:14.826452", "event": "TOOL_EXECUTION", "data": {"tool": "search_arxiv", "arguments": "query=\"Retrieval-Augmented Generation\", limit=1", "observation": "Error querying arXiv: HTTPSConnectionPool(host='export.arxiv.org', port=443): Read timed out. (read timeout=10)"}}
  {"timestamp": "2026-06-01T08:43:16.149764", "event": "LLM_CALL", "data": {"prompt": "...", "response": "Thought: Since the initial attempt to search the arXiv database timed out, I'll try to perform the search again to obtain the necessary paper information for citation.\n\nAction: search_arxiv(query=\"Retrieval-Augmented Generation\", limit=1)"}}
  {"timestamp": "2026-06-01T08:43:18.982880", "event": "LLM_CALL", "data": {"prompt": "...", "response": "Observation: [SYSTEM WARNING] You already executed search_arxiv(query=\"Retrieval-Augmented Generation\", limit=1) earlier. Repeating it will result in an infinite loop. Please analyze your previous observation and output your 'Final Answer:' or choose a different action."}}
  ```

- **Diagnosis**:
  When a tool call returns an error or a timeout observation, the LLM’s default parametric knowledge tends to trigger a blind "retry" since it does not have stateful awareness that a direct retry will just cause the same error in a single session. Without a state-tracking barrier, the agent is unable to adapt.

- **Solution**:
  I utilized a **Loop Prevention** mechanism inside the ReAct execution loop in `src/agent/agent.py`. The agent maintains an `action_history` list. If a proposed action matches a previously executed action in the same session, the system intercepts the loop, prevents the tool from executing again, and injects a `[SYSTEM WARNING]` warning the agent of the infinite loop. This forces the LLM to either change its query arguments, try an alternative database tool (e.g., switching to Semantic Scholar), or output its `Final Answer` based on cached knowledge.

---

## III. Personal Insights: Chatbot vs ReAct (10 Points)

1. **Reasoning**:
   The `Thought` block acts as a cognitive scratchpad for the model. Unlike a standard chatbot that starts generating the final answer immediately (often leading to premature reasoning paths or hallucinations), the `Thought` block allows the agent to break down complex goals, decide which database is most suitable, evaluate the results of tool outputs, and adjust its plan.

2. **Reliability**:
   The ReAct Agent can perform *worse* than a standard chatbot in cases of severe API rate-limiting or high-latency networks. Because ReAct is highly dependent on sequential tool executions (each requiring a separate LLM round-trip and a synchronous network request), any slow API call compounds the total latency. If the network is entirely blocked and fallbacks are poorly configured, a standard chatbot can fall back to its internal weights immediately, whereas a ReAct agent will keep attempting tool calls, leading to potential session timeouts.

3. **Observation**:
   Environment feedback (`Observation`) is the sensory input of the agent. When an observation returned a timeout or an API 429 error, it immediately steered the agent's next steps. For example, seeing `Error: Semantic Scholar API returned status code 429` prompted the agent to formulate a `Thought` recognizing the block, and subsequently switch its `Action` to `search_arxiv` or resort to a clean, factual final answer explaining the limitation rather than hallucinating fake links.

---

## IV. Future Improvements (5 Points)

To scale this scientific ReAct assistant into a highly robust production-grade system, the following architectural upgrades should be prioritized:

- **Scalability**:
  Introduce an **Asynchronous Queue / Task Runner** (e.g., using Celery or FastAPI Background Tasks) for executing tool calls. Instead of blocking the Gradio UI main thread during multi-step searches, the agent can yield step status and allow multiple users to query concurrently without memory or network bottlenecks.

- **Safety**:
  Implement a dual-LLM **Supervisor / Auditor** pattern. A smaller, faster model (or a strict rule-based guardrail) should audit the proposed tool arguments before execution to prevent Prompt Injection, and verify the final answer against the observations to ensure zero data leakage.

- **Performance**:
  Deploy a local **Vector Database** (such as Qdrant or Chroma) to index large-scale open-source paper corpuses (e.g., high-quality subsets of arXiv). This permits immediate vector similarity searches when external APIs are rate-limited, completely replacing simple keyword mock searches with high-quality semantic fallbacks.
