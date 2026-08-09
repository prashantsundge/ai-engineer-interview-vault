# Lessons Learned — Video Transcription & PDF Section-Aware Chunking
### SharePoint AI Chatbot Project | June 2026

---

## Overview

Extending the RAG pipeline to support video transcripts (MKV, MP4, Teams recordings) and large technical PDFs (Sonus handbook, administrator guides) introduced several architectural and tooling challenges. This document captures every decision made, what failed, what worked, and why — for the benefit of future team members and production deployment.

---

## Part 1 — Video Transcription

### Background

The CONNX team stores training sessions and MS Teams meeting recordings in SharePoint as MKV and MP4 files. Users needed to ask questions like:

- *"Summarise the SIP training session"*
- *"What did the trainer say about Direct Routing at 15 minutes?"*
- *"Show me the ATLAS SMART Learn sessions on call flow"*

The challenge: video files are binary — not text. The RAG pipeline requires text chunks in Qdrant. The solution is to extract transcripts and store them as searchable chunks with timestamps.

---

### Decision 1 — No Azure Speech API (Cost Elimination)

**Initial plan:** Use Azure Speech Services batch transcription (~$1.00/audio hour).

**Problem:** With 1000s of training videos in production, this would cost hundreds to thousands of dollars per full ingestion cycle. Not viable.

**Final approach:** Two free alternatives in priority order:

| Priority | Method | Cost | When Used |
|----------|--------|------|-----------|
| 1 | Teams VTT auto-transcripts | Free | Teams recordings with .vtt sibling file |
| 2 | Local OpenAI Whisper | Free | Any video without VTT |

**Key insight:** Microsoft Teams automatically generates `.vtt` transcript files alongside every meeting recording stored in SharePoint. These are WebVTT format files with timestamps and speaker names — exactly what we need, at zero cost.

---

### Decision 2 — VTT-First, Whisper Fallback Architecture

**Design:**
```
Video file detected
        │
        ├── Does sibling .vtt exist in same SharePoint folder?
        │       │
        │       ├── YES → Parse VTT directly (free, instant, has speakers)
        │       │
        │       └── NO  → Local Whisper transcription (free, on-device)
        │
        └── Store transcript chunks in Qdrant with timestamps
```

**VTT parsing advantages:**
- Zero transcription time — file already exists
- Speaker identification from Teams `<v SpeakerName>` tags
- Accurate timestamps already embedded
- No GPU/CPU compute cost

**Whisper fallback advantages:**
- Works for any video format (MKV, MP4, MOV, AVI, WebM)
- Completely free — model runs on local CPU
- `base` model: ~150MB download, good accuracy for English technical content
- For better accuracy on accented speech: upgrade to `small` or `medium` model

---

### Lesson 1 — `list_files_in_folder` Must Be Added to Crawler for Production VTT Lookup

**Problem encountered:** The orchestrator calls `self.crawler.list_files_in_folder(parent_path)` to find sibling VTT files. This method did not exist on `SharePointCrawler`.

**Error in logs:**
```
WARNING | VTT sibling lookup failed: 'SharePointCrawler' object has no attribute 'list_files_in_folder'
INFO    | ⚠️ No VTT found for ... — using Whisper fallback
```

**Demo fix:** Added `hasattr()` guard in orchestrator — skips VTT lookup silently and proceeds to Whisper if the method doesn't exist.

**Production fix required:** Add this method to `app/graph/crawler.py`:
```python
def list_files_in_folder(self, folder_path: str) -> list:
    """List all files in a SharePoint folder by path."""
    url = (
        f"https://graph.microsoft.com/v1.0/drives/{self.drive_id}"
        f"/root:{folder_path}:/children"
        f"?$select=id,name,size,lastModifiedDateTime,webUrl,parentReference"
        f"&$top=100"
    )
    response = self.client.get(url)
    return response.json().get("value", [])
```

**Impact:** Without this method, ALL Teams recordings fall back to Whisper even when a free VTT exists alongside them. In production with 1000s of videos this wastes significant compute time.

---

### Lesson 2 — MKV Files Are Training Videos, Not Teams Recordings

**Observation from file names:**
- `ASIMS L2 - ATLAS SMART Learn Session 1.mkv` → training sessions, no Teams VTT
- `SIP Training-20220914_223629-Meeting Recording.mp4` → Teams recording, should have VTT

**MKV files** — these are screen recordings or edited training videos, not Teams meetings. They will never have a VTT sibling. Whisper is the only option.

**MP4 files with "Meeting Recording" in name** — these are Teams recordings. In production SharePoint (not demo), these WILL have `SIP Training-...-Meeting Recording.vtt` alongside them.

**Action for production:**
1. Check one real Teams recording in SharePoint manually — confirm `.vtt` exists
2. Add `list_files_in_folder` to crawler
3. Teams VTT names follow this pattern: `{meeting-title}-Meeting Recording.vtt`

---

### Lesson 3 — moviepy v2.x Has Breaking Import Change

**Error encountered in early testing:**
```python
from moviepy.editor import VideoFileClip  # ← works in v1.x
```
```
ImportError: cannot import name 'editor' from 'moviepy'
```

**Reason:** moviepy v2.x removed the `.editor` submodule.

**Fix for v2.x (installed version: 2.2.1):**
```python
from moviepy import VideoFileClip  # ← correct for v2.x
```

The `video_extractor.py` already handles this correctly. If you see this error, check your moviepy version:
```powershell
pip show moviepy
```

---

### Lesson 4 — Whisper Model Selection Trade-offs

| Model | Size | Speed (CPU) | Accuracy | Recommended For |
|-------|------|-------------|----------|-----------------|
| `tiny` | 75MB | ~32× real-time | Low | Quick testing only |
| `base` | 145MB | ~16× real-time | Good | General English content |
| `small` | 461MB | ~6× real-time | Better | Technical terminology |
| `medium` | 1.5GB | ~2× real-time | High | Accented speech, mixed terms |
| `large` | 2.9GB | ~1× real-time | Best | Production quality |

**Current setting:** `base` — good for CONNX technical content (SBC, SIP, Teams terms).

**To change model**, edit `video_extractor.py`:
```python
model = whisper.load_model("small")  # upgrade for better accuracy
```

**Real-time estimate for 7 demo files:**
- Average training video: ~45 minutes
- `base` model on CPU: ~3 minutes per video
- Total for 7 videos: ~21 minutes

---

### Lesson 5 — Chunk Strategy for Transcripts

**Decision:** 2-minute (120 second) transcript chunks.

**Reasoning:**

| Chunk Size | Problem |
|------------|---------|
| Too small (30s) | Not enough context for meaningful Q&A |
| Too large (10min) | Retrieval returns too broad a section |
| 2 minutes | Covers one topic/discussion point, enough context |

**Each chunk includes:**
- `[HH:MM:SS → HH:MM:SS]` timestamp header
- Speaker names (from VTT) or raw transcript (from Whisper)
- `modality: "video_transcript"` metadata
- `video_url` pointing back to SharePoint file
- `timestamp_start` and `timestamp_end` in seconds for future deep-link generation

**Retrieval advantage:** When user asks "what was discussed about SBC configuration at the start of session 2", the 2-minute chunk containing that segment is retrieved, and the LLM cites `**[04:30]** Trainer explains SBC trunk configuration`.

---

### Lesson 6 — Generation Prompt Must Handle Timestamp Citations

Without explicit instructions, the LLM would answer transcript questions like any document query. We added video-specific rules:

```
VIDEO TRANSCRIPT RULES (apply when context contains [HH:MM:SS] timestamps):
- Always cite the timestamp: "At 12:34, the trainer explains..."
- Provide a summary of topics covered with their timestamps
- Format: **[MM:SS]** Topic — makes it easy to jump to that point in the video
```

This makes answers look like:

> **[02:15]** Introduction to SIP trunking architecture  
> **[08:40]** Configuring the SBC for Teams Direct Routing  
> **[14:55]** Troubleshooting registration failures  

Instead of: *"The video covers SIP trunking and SBC configuration."*

---

## Part 2 — PDF Section-Aware Chunking

### Background

Large technical PDFs (Sonus SBC handbook, administrator guides) were being chunked paragraph-by-paragraph by the original extractor. A 300-page handbook produced 800+ tiny chunks, each containing one sentence with no section context. Problems:

- *"How do I configure SIP profiles?"* → Retrieved 5 chunks from 5 different chapters, none complete
- Section context lost — chunk says "Set the value to 30" with no heading saying this is about SIP timers
- Multi-question PDF sessions were incoherent — each question retrieved from a different part of the handbook

---

### Lesson 7 — Paragraph Chunking Destroys Section Context

**Original approach:**
```python
paragraphs = [p.strip() for p in text.split("\n") if p.strip()]
for para in paragraphs:
    document.add_block(ContentBlock(text=para, ...))
```

**Problem:** Every paragraph becomes a separate chunk. For:
```
Chapter 5: SIP Profile Configuration

5.1 Timer Settings
The T1 timer controls retransmission. Set to 500ms for WAN connections.

5.2 Authentication
Enable digest authentication for all trunk groups.
```

This creates 3 separate chunks: one with the chapter heading, one about T1 timers, one about authentication. When user asks "how do I configure SIP profiles?", they might get the authentication chunk with no context that it's from Chapter 5.

**Fix — section-aware chunking:**
- Detect headings using font size (heading font > body font × 1.15)
- Also detect bold short lines and numbered headings (5.1, Chapter 3, etc.)
- Group all body text under its parent heading
- Prepend heading to every chunk: `## Chapter 5: SIP Profile Configuration\n\n{body_text}`
- Max chunk size: 400-600 words to stay within context window efficiently

---

### Lesson 8 — Heading Detection Strategy

**Font-size approach (primary):**
```python
body_font_size = self._detect_body_font_size(pdf)  # modal font = body
heading_threshold = body_font_size * 1.15
is_heading = max_font_size >= heading_threshold
```

**Heuristic fallbacks (secondary):**
```python
# Numbered sections: "1.", "1.1", "Chapter 1", "Section 2.3"
if re.match(r"^(\d+\.)+\s+\w", text): return True
if re.match(r"^(chapter|section|appendix)\s+\d", text, re.I): return True
# Short all-caps lines
if text.isupper() and 3 <= len(text.split()) <= 8: return True
```

**Edge cases found:**
- Some PDFs use the same font size for headings and body (style-only differentiation)
- Bold detection via PyMuPDF `flags & 16` works for most PDFs but not all
- Technical PDFs with dense tables may not have traditional headings

**Recommendation for production:** Test the extractor on each PDF type used by your team. If heading detection is wrong, you can tune `HEADING_SIZE_RATIO` (currently 1.15) up or down.

---

### Lesson 9 — Section Context in Every Chunk is Critical for Q&A

**Key design decision:** Every chunk starts with its section heading, even if the user didn't ask about a section.

```python
chunk_text = f"## {section}\n\n{body_text}"
```

**Why:** The LLM needs to know "this text about T1 timers is from the SIP Profile Configuration section" to give a useful answer. Without the heading, the chunk is decontextualised — the LLM can't tell which part of the handbook it's from.

**Result:** When user asks "what are the SBC timer settings?", the retrieved chunk contains:
```
## 5.1 SIP Timer Settings
The T1 timer controls retransmission intervals. For WAN connections, 
set T1 to 500ms. The T2 timer should be 4 seconds...
```

Instead of just: `The T1 timer controls retransmission intervals. For WAN connections, set T1 to 500ms.`

---

### Lesson 10 — Backward Compatibility Pattern for Extractor Replacement

**Problem:** `extractor_router.py` imports `PDFExtractor` by class name. Replacing the file would break the import if the class name changed.

**Fix:** Added alias at bottom of new extractor file:
```python
# Keep backward compatibility with extractor_router.py
PDFExtractor = EnhancedPDFExtractor
```

This means the router's existing `from app.ingestion.extractors.pdf_extractor import PDFExtractor` continues to work without any change to the router.

**Pattern to follow** for all future extractor upgrades — always alias the new class to the old name.

---

### Lesson 11 — Re-ingestion Required After Extractor Changes

**Important:** Changing the PDF extractor does NOT automatically re-chunk existing PDFs in Qdrant. Old paragraph-level chunks remain until the file is re-ingested.

**To re-ingest all PDFs with the new extractor:**

Option A — Reset delta token (re-ingests everything):
```powershell
python scripts/reset_delta_token.py
python scripts/ingest.py
```

Option B — Targeted re-ingestion (delete old chunks, re-ingest specific file):
```python
# In a Python shell
from app.vectorstore.qdrant_client import QdrantManager
from app.vectorstore.repository import VectorRepository
qm = QdrantManager()
repo = VectorRepository(qm)
repo.delete_by_file_name("SBC Administrator Guide.pdf")
# Then re-run ingest.py with that file only
```

**Note:** For large collections (2260+ chunks), a full re-ingestion takes 15-30 minutes. Plan accordingly.

---

## Summary — Files Created/Modified

| File | Type | Purpose |
|------|------|---------|
| `app/ingestion/extractors/video_extractor.py` | NEW | VTT parsing + Whisper transcription |
| `app/ingestion/extractors/pdf_extractor.py` | REPLACED | Section-aware chunking for large PDFs |
| `app/ingestion/extractor_router.py` | UPDATED | Added .mkv, .mp4, .mov, .vtt, .avi, .webm routes |
| `app/ingestion/orchestrator.py` | UPDATED | VTT sibling lookup with graceful fallback |
| `app/agents/generation_node.py` | UPDATED | Video timestamp citation instructions |
| `app/agents/retrieval_node.py` | UPDATED | Video/PDF section boost scoring |

---

## Production Deployment Checklist

### For video transcription at scale (1000s of videos):
- [ ] Add `list_files_in_folder` to `SharePointCrawler` — enables free VTT parsing for all Teams recordings
- [ ] Verify Teams recordings have `.vtt` siblings in production SharePoint (open one folder in browser)
- [ ] For MKV training files without VTT: run Whisper overnight (GPU server recommended for speed)
- [ ] Consider `small` Whisper model for technical terminology accuracy
- [ ] Upgrade Celery worker timeout for long video files (`task_time_limit = 3600` for 1-hour videos)

### For PDF chunking:
- [ ] Test `EnhancedPDFExtractor` on each PDF type (handbook, admin guide, runbook)
- [ ] Verify heading detection is correct — check logs for heading/body classification
- [ ] Re-ingest all existing PDFs after deploying new extractor
- [ ] Tune `HEADING_SIZE_RATIO` (1.15) and `TARGET_CHUNK_WORDS` (400) per document type if needed

### Dependencies required:
```
webvtt-py>=0.5.1        # VTT parsing
openai-whisper>=20231117 # Local Whisper transcription
moviepy>=2.0.0           # Audio extraction from video
PyMuPDF>=1.23.0          # Enhanced PDF processing (fitz)
ffmpeg                   # System dependency for audio/video processing
```

---

*Document maintained by: Pravin Kumar Sundge | CONNX 2026 | June 2026*
