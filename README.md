# Blog Writing Agent

A multi-agent blog writer built with **LangGraph** and **Streamlit**. Give it a topic, and it plans a full blog post, optionally researches the web for current information, fans the work out to parallel "worker" agents to draft each section, then merges everything into a final Markdown file — with optional AI-generated images.

The repo is organized as a progression of notebooks, each adding a capability, culminating in a production-ready backend (`bwa_backend.py`) and a Streamlit frontend (`bwa_frontend.py`).

## How it works

The core pipeline is a LangGraph `StateGraph`:

```
START
  │
  ▼
router ──(needs research?)──► research ──► orchestrator
  │                                             │
  └───────────────(no)─────────────────────────┘
                                                 │
                                          fan-out (Send)
                                                 │
                                     ┌───────────┼───────────┐
                                     ▼           ▼           ▼
                                  worker      worker      worker
                                     │           │           │
                                     └───────────┼───────────┘
                                                 ▼
                                 reducer (merge → decide images → generate images)
                                                 │
                                                END
```

1. **Router** — decides whether the topic needs live web research (`closed_book`, `hybrid`, or `open_book` mode) and generates search queries if so.
2. **Research** — runs the queries via Tavily and synthesizes the raw results into structured, deduplicated `EvidenceItem`s, filtered by recency.
3. **Orchestrator** — plans the blog: title, audience, tone, and 5–9 sections (`Task`s), each with a goal, bullet points, and a target word count.
4. **Workers** — run in parallel (via `Send`), each writing one section in Markdown, grounded in the provided evidence when required.
5. **Reducer** — a subgraph that:
   - merges all sections into one document,
   - decides whether diagrams/images would help (max 3),
   - generates those images (via Pollinations) and inlines them into the Markdown,
   - saves the final post to disk.

The final output is a `.md` file (plus an `images/` folder if any images were generated).

## Project structure

```
├── 1_bwa_basic.ipynb                  # Minimal orchestrator → workers → reducer graph
├── 2_bwa_improved_prompting.ipynb     # Richer prompting/schemas
├── 3_bwa_research.ipynb               # Adds the router + Tavily research step
├── 4_bwa_research_fine_tuned.ipynb    # Tuned research/grounding logic
├── 5_bwa_image.ipynb                  # Adds the image planning/generation subgraph
├── bwa_backend.py                     # Final compiled LangGraph app (router→research→orchestrator→workers→reducer)
├── bwa_frontend.py                    # Streamlit UI for running the graph and browsing results
└── tavily_test.ipynb                  # Scratch notebook for testing the Tavily search tool
```

## Setup

### 1. Install dependencies

```bash
pip install langgraph langchain-groq langchain-community streamlit pandas pydantic python-dotenv requests tavily-python groq
```

### 2. Configure environment variables

Create a `.env` file in the project root:

```env
GROQ_API_KEY=your-groq-api-key          # required — planning + writing
TAVILY_API_KEY=your-tavily-api-key      # optional — enables web research
```

- `GROQ_API_KEY` is required (used for planning and writing via Groq's `openai/gpt-oss-20b`).
- `TAVILY_API_KEY` is optional. Without it, the research step simply returns no evidence and the graph falls back to closed-book writing.
- Image generation uses [Pollinations](https://pollinations.ai) (model `flux`) and needs **no API key**. If a request fails, a placeholder note is inserted into the Markdown instead of the image.

## Usage

### Run the Streamlit app

```bash
streamlit run bwa_frontend.py
```

Then, in the **Blog Settings** sidebar:
1. Enter a blog **topic**.
2. Choose **Blog Type** (Tutorial / Explainer / News / Comparison / System Design), **Audience** (Beginner / Intermediate / Advanced), **Tone** (Professional / Casual / Academic), and **Length** (Short ≈ 600 / Medium ≈ 1200 / Long ≈ 2000 words).
3. Toggle **Web Research**, **Generate diagrams**, **Add citations**, and **Include examples**.
4. Pick an **as-of date** (used for recency filtering during research).
5. Click **🚀 Generate Blog**.

Your Blog Type, Audience, and Tone are applied directly. The checkboxes hard-enable/disable each capability, while the router still decides *whether* research is actually needed (evergreen topics skip it), and the planner decides the section breakdown and where images/citations/code genuinely help.

The app streams progress through each graph node and shows:
- **🧩 Plan** — the generated outline and task table
- **🔎 Evidence** — sources pulled from research (if any)
- **📝 Markdown Preview** — the rendered blog post, with download buttons for the `.md` file and a zipped bundle (markdown + images)
- **🖼️ Images** — generated image assets
- **🧾 Logs** — raw event log of the graph run

Previously generated posts (`*.md` files in the working directory) are listed in the sidebar and can be reloaded into the viewer.

### Run programmatically

```python
from bwa_backend import app

result = app.invoke({
    "topic": "Write a blog on Self Attention",
    "prefs": {
        "blog_type": "Explainer",     # Tutorial | Explainer | News | Comparison | System Design
        "audience": "Intermediate",   # Beginner | Intermediate | Advanced
        "tone": "Professional",       # Professional | Casual | Academic
        "length": "Medium",           # Short (~600) | Medium (~1200) | Long (~2000)
        "want_research": False,
        "want_images": True,
        "want_citations": True,
        "want_code": True,
    },
    "mode": "",
    "needs_research": False,
    "queries": [],
    "evidence": [],
    "plan": None,
    "as_of": "2026-07-12",
    "recency_days": 7,
    "sections": [],
    "merged_md": "",
    "md_with_placeholders": "",
    "image_specs": [],
    "final": "",
})

print(result["final"])
```

The finished post is also written to disk as `<slugified-title>.md`, with any images saved under `images/`. (`prefs` is optional — if omitted, the graph defaults to a Medium, research-on Explainer.)

## Key concepts

- **Orchestrator–worker pattern**: a single planning step produces a set of tasks, which are dispatched to parallel workers via LangGraph's `Send` API, then reduced back into one document.
- **Grounded generation**: in `hybrid`/`open_book` modes, workers are instructed to cite only URLs present in the retrieved evidence, and to explicitly flag unsupported claims rather than inventing them.
- **Graceful degradation**: missing API keys (Tavily, Google) don't break the graph — research and image generation degrade to no-ops or placeholder text instead of failing the whole run.

## Notes

- Models are currently hardcoded to Groq's `openai/gpt-oss-20b` (text) and Pollinations' `flux` (images) inside `bwa_backend.py` — change these if you'd like to use different models.
- The notebooks (`1_bwa_basic.ipynb` → `5_bwa_image.ipynb`) are meant to be read in order — they show the incremental design decisions behind `bwa_backend.py` and are a good place to start if you want to understand or extend the pipeline.