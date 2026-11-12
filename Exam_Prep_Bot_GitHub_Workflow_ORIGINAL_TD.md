# AI Exam Prep Bot — Conversational workflow

This diagram is designed for the university project (React frontend + Python/FastAPI + LLM). It is intentionally detailed enough for implementation and for demonstrating repair/recovery in the course.

**START:** Student opens the bot. **END:** Bot displays a summary after the quiz is completed or ended early. A paused quiz can later resume.

**Legend:**

- **Purple, `LLM CALL`** — model invocation (question generation, short-essay grading, hints, ambiguous intent, grade review).
- **Blue, `PY LOGIC`** — deterministic backend routing, source validation, score recording, MC/true-false grading.
- **Yellow, `RECOVERY`** — repair path for errors, ambiguity or changes.
- **Grey slanted box, `STUDENT`** — user input or choice.
- **Green, `START` / `END`** — clear entry and exit points.

The bot asks for question format before generation, unless the student already specified it. Options are **Multiple choice**, **Short essay**, **True or false**, and **Mixed**. The Python backend preserves progress across clarifications, hints and format changes.

```mermaid
flowchart TD
    %% Developer workflow for the HTI.560 exam preparation chatbot.
    %% Purple double-bordered = LLM call. Blue = deterministic Python/FastAPI logic.
    %% Yellow = recovery. Grey slanted = student input. Green = start/end.

    subgraph SETUP["1. SETUP AND QUESTION-TYPE SELECTION"]
        START(["START - student opens exam prep bot"])
        SOURCE[/"STUDENT - enter topic, paste notes or upload PDF"/]
        SOURCE_CHECK{"PY LOGIC - is there usable material or a topic?"}
        SOURCE_REPAIR["RECOVERY - clarify topic, retry upload or paste notes"]
        TYPE_ASK["BOT - What kind of questions?<br/>Multiple choice / Short essay / True-False / Mixed"]
        TYPE_PICK[/"STUDENT - pick type(s) or describe preference"/]
        TYPE_CHECK{"PY LOGIC - is the question-type choice clear?"}
        TYPE_REPAIR["RECOVERY - explain formats and offer four buttons"]
        SETTINGS["PY LOGIC - save type(s), count, difficulty and source<br/>Use defaults for optional settings"]
        CONFIRM["BOT - summarize settings; offer Start or Edit"]
        START_CHOICE[/"STUDENT - Start or Edit"/]

        START --> SOURCE --> SOURCE_CHECK
        SOURCE_CHECK -->|No| SOURCE_REPAIR --> SOURCE
        SOURCE_CHECK -->|Yes| TYPE_ASK --> TYPE_PICK --> TYPE_CHECK
        TYPE_CHECK -->|No| TYPE_REPAIR --> TYPE_PICK
        TYPE_CHECK -->|Yes| SETTINGS --> CONFIRM --> START_CHOICE
        START_CHOICE -->|Edit| TYPE_ASK
    end

    subgraph GENERATE["2. GENERATE AND VALIDATE THE QUIZ"]
        GEN[["LLM CALL - generate requested question types,<br/>solutions, rubrics and supporting references"]]
        CHECK_GEN{"PY LOGIC - correct schema, types,<br/>source evidence and question count?"}
        GEN_REPAIR["RECOVERY - offer fewer valid questions,<br/>more material or different types"]
        GEN_CHOICE[/"STUDENT - choose alternative"/]

        START_CHOICE -->|Start| GEN --> CHECK_GEN
        CHECK_GEN -->|Invalid or insufficient| GEN_REPAIR --> GEN_CHOICE
        GEN_CHOICE -->|Add material| SOURCE
        GEN_CHOICE -->|Change types or count| TYPE_ASK
        GEN_CHOICE -->|Accept available valid questions| ASK
    end

    subgraph QUIZ["3. CONVERSATIONAL PRACTICE LOOP"]
        ASK["BOT and REACT - show Q n/N with controls<br/>MC: options / T-F: two buttons / Essay: text input"]
        STUDENT[/"STUDENT - answer, ask for help or choose an action"/]
        INTENT{"PY LOGIC - action clearly identified?"}
        INTENT_LLM[["LLM CALL - classify ambiguous free-text intent"]]
        INTENT_OK{"PY LOGIC - confident intent?"}
        INTENT_REPAIR["RECOVERY - ask one clarification; show actions"]
        ROUTE{"PY LOGIC - route student action"}
        KIND{"PY LOGIC - active question type?"}
        KEY_CHECK["PY LOGIC - check selected option<br/>against stored answer key"]
        ESSAY_CHECK[["LLM CALL - assess written answer<br/>against rubric and source"]]
        VERDICT{"PY LOGIC - assessment result?"}
        CLARIFY["RECOVERY - clarify answer without grading"]
        FEEDBACK["BOT - targeted feedback; Retry / Hint / Reveal"]
        RECORD["PY LOGIC - record outcome and help used"]
        HINT_LLM[["LLM CALL - hint, rephrase or explain a term<br/>without changing question ID"]]
        REVEAL["PY LOGIC - show stored answer;<br/>record skipped or revealed"]
        REVIEW_LLM[["LLM CALL - independently review disputed grade"]]
        REVIEW["PY LOGIC - update result only if justified;<br/>tell student why"]
        REDIRECT["RECOVERY - redirect off-topic message;<br/>keep current question"]
        CHANGE["BOT - confirm changing topic or question types<br/>and regenerating unanswered questions"]
        CHANGE_CHOICE[/"STUDENT - confirm or cancel"/]
        PAUSE["PY LOGIC - save paused session"]
        PAUSE_CHOICE[/"STUDENT - resume or finish"/]
        NEXT{"PY LOGIC - are there more questions?"}

        ASK --> STUDENT --> INTENT
        INTENT -->|Yes - button or known command| ROUTE
        INTENT -->|Ambiguous free text| INTENT_LLM --> INTENT_OK
        INTENT_OK -->|No| INTENT_REPAIR --> STUDENT
        INTENT_OK -->|Yes| ROUTE

        ROUTE -->|Submit answer| KIND
        KIND -->|Multiple choice or True-False| KEY_CHECK --> VERDICT
        KIND -->|Short essay| ESSAY_CHECK --> VERDICT
        VERDICT -->|Unclear / unverifiable| CLARIFY --> STUDENT
        VERDICT -->|Incorrect / partial| FEEDBACK --> STUDENT
        VERDICT -->|Correct| RECORD --> NEXT

        ROUTE -->|Hint / rephrase / explain| HINT_LLM --> ASK
        ROUTE -->|Show answer / skip| REVEAL --> NEXT
        ROUTE -->|Challenge grade / correct previous answer| REVIEW_LLM --> REVIEW --> STUDENT
        ROUTE -->|Off-topic / unclear request| REDIRECT --> STUDENT
        ROUTE -->|Change topic or question types| CHANGE --> CHANGE_CHOICE
        CHANGE_CHOICE -->|Cancel| ASK
        CHANGE_CHOICE -->|Change types - keep completed results| TYPE_ASK
        CHANGE_CHOICE -->|Change topic - keep completed results| SOURCE
        ROUTE -->|Pause| PAUSE --> PAUSE_CHOICE
        PAUSE_CHOICE -->|Resume| ASK
        ROUTE -->|Next after resolution| NEXT
        NEXT -->|Yes| ASK
    end

    subgraph FINISH["4. FINISH"]
        SUMMARIZE["PY LOGIC - calculate scores and help used<br/>by question type; suggest weak topics"]
        SUMMARY["BOT - show study summary and options for a new quiz"]
        END(["END - quiz completed or student ended it"])

        NEXT -->|No| SUMMARIZE
        ROUTE -->|End quiz| SUMMARIZE
        PAUSE_CHOICE -->|Finish| SUMMARIZE
        SUMMARIZE --> SUMMARY --> END
    end

    class START,END startEnd
    class SOURCE,TYPE_PICK,START_CHOICE,GEN_CHOICE,STUDENT,CHANGE_CHOICE,PAUSE_CHOICE user
    class SOURCE_CHECK,TYPE_CHECK,CHECK_GEN,INTENT,INTENT_OK,ROUTE,KIND,VERDICT,NEXT decision
    class SETTINGS,KEY_CHECK,RECORD,REVEAL,REVIEW,PAUSE,SUMMARIZE logic
    class GEN,INTENT_LLM,ESSAY_CHECK,HINT_LLM,REVIEW_LLM llm
    class SOURCE_REPAIR,TYPE_REPAIR,GEN_REPAIR,INTENT_REPAIR,CLARIFY,FEEDBACK,REDIRECT,CHANGE recovery
    class TYPE_ASK,CONFIRM,ASK,SUMMARY bot

    classDef startEnd fill:#166534,color:#ffffff,stroke:#14532d,stroke-width:2px
    classDef user fill:#f3f4f6,color:#111827,stroke:#6b7280,stroke-width:1px
    classDef decision fill:#eff6ff,color:#1e3a8a,stroke:#2563eb,stroke-width:2px
    classDef logic fill:#dbeafe,color:#1e3a8a,stroke:#3b82f6,stroke-width:1px
    classDef llm fill:#ede9fe,color:#4c1d95,stroke:#7c3aed,stroke-width:3px
    classDef recovery fill:#fef3c7,color:#78350f,stroke:#d97706,stroke-width:2px
    classDef bot fill:#f0fdf4,color:#14532d,stroke:#22c55e,stroke-width:1px
```

**Source file for Mermaid editors:** [`Exam_Prep_Bot_Workflow.mmd`](./Exam_Prep_Bot_Workflow.mmd).

**Detailed developer specification:** [`Exam_Prep_Bot_Conversation_Workflow_Developer_Handoff.md`](./Exam_Prep_Bot_Conversation_Workflow_Developer_Handoff.md).
