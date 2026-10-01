# AgenticVideo

AgenticVideo is an agentic AI video generation system that transforms a user request into a complete generated video.

The system uses LangGraph to orchestrate a multi-stage AI workflow including content moderation, prompt enhancement, human-in-the-loop review, script generation, scene planning, video generation, text-to-speech, subtitle alignment, and final video assembly.

The project is designed as a modular pipeline where LLM reasoning, generative video models, speech generation, subtitle synchronization, and deterministic media processing work together to produce the final video.

---

## Features

- Content moderation before generation
- LLM-based prompt enhancement
- Human-in-the-loop prompt review
- Structured script generation
- Automatic scene planning
- Scene-level video prompt generation
- AI video generation
- Text-to-speech narration
- Forced subtitle alignment
- Background audio and narration mixing
- Burned-in subtitle support
- FFmpeg-based final video assembly
- LangGraph-based workflow orchestration
- Structured state shared across pipeline stages

---

## System Architecture

```text
User Input
    ↓
Content Moderation
    ↓
Prompt Enhancement
    ↓
Human Review
    ↓
Script Generation
    ↓
Scene Planning
    ↓
Video Prompt Generation
    ↓
Video Generation
    ↓
Speech Generation
    ↓
Subtitle Generation
    ↓
Video Assembly
    ↓
Final Video
```

LangGraph acts as the orchestration layer of the system. Each stage is implemented as a node that reads information from the shared video state and writes its result back to the state.

Conditional routing is used for moderation and human review, allowing the workflow to stop, retry, or continue depending on the result.

---

## Pipeline

### 1. Input and Content Moderation

The user provides:

- A text prompt
- Optional required words or phrases
- An optional reference image

The input is checked before entering the generation pipeline. Unsafe input can be rejected or revised before processing continues.

### 2. Prompt Enhancement

The user's request is converted into structured video requirements using an LLM.

The enhanced prompt contains information such as:

- Topic
- Objective
- Target audience
- Tone
- Visual style
- Video duration
- Key message

Structured outputs are used to make the information predictable for later pipeline stages.

### 3. Human-in-the-Loop Review

LangGraph's interrupt mechanism pauses the workflow after prompt enhancement.

The user can:

- Approve the enhanced prompt
- Edit the requirements
- Reject the request

Edited prompts can be moderated again before generation continues.

### 4. Script Generation

After approval, the system generates a structured narration script based on the enhanced requirements.

Required words supplied by the user are incorporated into the generated content.

### 5. Scene Planning

The narration is divided into individual scenes.

Each scene contains information such as:

- Scene ID
- Narration
- Visual description
- Duration
- Scene continuity information

This converts the high-level script into a structure suitable for video generation.

### 6. Video Prompt Generation

A video-generation prompt is created for each scene.

The prompts describe the visual content required by the generative video model while avoiding spoken narration because narration is generated separately using text-to-speech.

Background music and ambient audio may be generated as part of the video clips.

### 7. Video Generation

Each scene prompt is submitted to a generative video model.

Video generation is asynchronous:

```text
Submit Request
      ↓
Generation Job
      ↓
Poll Status
      ↓
Completed
      ↓
Download Scene
```

The current implementation supports Google Veo as the video generation provider.

Generated scenes are stored individually before final assembly.

### 8. Speech Generation

The complete narration script is converted into speech using text-to-speech.

The current implementation uses OpenAI TTS to generate the narration audio.

The narration is generated independently from the video so that speech can remain consistent across scenes.

### 9. Subtitle Generation

The narration script and generated speech are used to produce synchronized subtitles.

Forced alignment is used to determine subtitle timing rather than estimating timestamps directly from the script.

The result is exported as an SRT subtitle file.

### 10. Video Assembly

FFmpeg performs the deterministic media-processing stage.

The assembly pipeline:

```text
Generated Scene Videos
        ↓
Concatenate Scenes
        ↓
Veo Background Audio ──┐
                       ├── Audio Mixing
TTS Narration ─────────┘
        ↓
Subtitle Rendering
        ↓
Final MP4
```

Background audio is reduced so that narration remains clear.

If the generated video is shorter than the narration, the final frame can be extended to match the narration duration.

The final output is encoded as an MP4 using H.264 video and AAC audio.

---

## Technology Stack

### AI and Orchestration

- Python
- LangChain
- LangGraph
- Pydantic
- Structured LLM outputs
- Human-in-the-loop interrupts

### Generative AI

- Large Language Models for planning and content generation
- Google Veo for video generation
- OpenAI TTS for speech generation

### Media Processing

- FFmpeg
- FFprobe
- libass
- SRT subtitles
- H.264 / AAC

### Development

- Jupyter Notebook
- uv
- Git
- GitHub
- VS Code

---

## Project Structure

```text
agentic-video/
│
├── notebooks/
│   ├── 01_core_pipeline.ipynb
│   ├── 02_video_generation.ipynb
│   ├── 03_speech_generation.ipynb
│   ├── 04_subtitle_generation.ipynb
│   ├── 05_end_to_end_pipeline.ipynb
│   └── reference_images/
│
├── .env.example
├── .gitignore
├── .python-version
├── pyproject.toml
├── uv.lock
└── README.md
```

The first four notebooks develop and test individual pipeline components. `05_end_to_end_pipeline.ipynb` integrates the components into the complete LangGraph workflow.

Generated media is written to an `outputs/` directory, which is created automatically when the notebooks run and is not committed to Git.

---

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/YukPingLee/agentic-video.git
cd agentic-video
```

### 2. Install dependencies

This project uses `uv` for Python dependency management.

```bash
uv sync
```

### 3. Configure environment variables

Copy the example file and fill in your API keys:

```bash
cp .env.example .env
```

| Variable | Required | Used for |
| --- | --- | --- |
| `AI_MODEL_API_KEY` | Yes | OpenAI — moderation, LLM, TTS, and Whisper alignment |
| `GEMINI_API_KEY` | Yes | Google Veo video generation |
| `LANGSMITH_API_KEY`, `LANGSMITH_TRACING`, `LANGSMITH_PROJECT` | No | LangSmith tracing |

`.env` is listed in `.gitignore`. Do not commit it to Git.

### 4. Install FFmpeg

FFmpeg is required for final video assembly.

On macOS with Homebrew:

```bash
brew install ffmpeg-full
```

The full build is used because burned-in subtitles require the `subtitles` filter and `libass`.

Verify support with:

```bash
ffmpeg -filters | grep subtitles
```

The output should include the `subtitles` filter.

---

## Running the Pipeline

Open:

```text
notebooks/05_end_to_end_pipeline.ipynb
```

and run the notebook cells in order.

A typical input state contains:

```python
initial_state = {
    "user_prompt": "Create a technical introduction video about AgenticVideo.",
    "required_words": [
        "AgenticVideo",
        "LangGraph",
        "Agentic AI"
    ],
    "reference_image": None,
    "moderation_retry_count": 0
}
```

The LangGraph workflow then processes the request through the complete generation pipeline.

---

## LangGraph State

The workflow uses a shared `VideoState` to pass structured information between nodes.

Conceptually:

```text
VideoState
├── User requirements
├── Moderation result
├── Enhanced prompt
├── Human review decision
├── Script
├── Scene plan
├── Video prompts
├── Generated videos
├── Speech result
├── Subtitle result
└── Final video result
```

This allows individual components to remain modular while LangGraph controls execution order and routing.

---

## Design Principles

### Agentic orchestration

LangGraph controls the workflow rather than placing the entire generation process inside one large function.

### Structured outputs

Pydantic models are used between AI stages to reduce ambiguity and make downstream processing more reliable.

### Human control

The user can review and modify AI-generated requirements before expensive media generation begins.

### Provider separation

LLMs, video generation, TTS, subtitle alignment, and FFmpeg are treated as separate components so that providers can be changed without redesigning the entire workflow.

### AI where reasoning is needed, deterministic tools where it is not

LLMs are used for tasks such as prompt enhancement, script generation, and scene planning.

FFmpeg is used for deterministic operations such as concatenation, audio mixing, subtitle rendering, and final encoding.

---

## Current Status

The current prototype implements the core end-to-end architecture:

- Input moderation
- Prompt enhancement
- Human-in-the-loop review
- Script generation
- Scene planning
- Video prompt generation
- Video generation
- TTS narration
- Subtitle generation
- FFmpeg video assembly

The project is currently focused on validating the complete generation workflow before moving toward a production web application.

---

## Planned Improvements

Future development may include:

- Web-based user interface
- REST API
- Asynchronous generation jobs
- Persistent LangGraph state
- Cloud object storage
- User authentication
- Improved scene-level audio/video synchronization
- Failed-scene regeneration
- Quality-control agents
- Additional video and speech providers
- Reference image workflows
- MCP integration
- Automated testing and CI/CD
- Production deployment

---

## Disclaimer

This project is currently an experimental prototype. Generative AI outputs may vary between runs, and external model availability, generation limits, and API quotas may affect the pipeline.

