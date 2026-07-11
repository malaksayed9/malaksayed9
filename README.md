# Fari2y — Academic Performance & Project Track Intelligence Platform

A graduation project that combines a full **Data Warehousing / BI pipeline** with a **Machine Learning–based project track recommender**, built to help universities track student performance, team dynamics, instructor workload, and automatically classify course project tasks into the right technical track (Backend, Frontend, Database, ML/AI).

---

## 📌 Project Overview

Fari2y collects, cleans, and models academic data (students, instructors, teams, courses, projects, tasks, and skills) into a proper **star schema data warehouse**, then surfaces it through interactive **Power BI dashboards**. On top of that, a **fine-tuned BERT multi-label classification model** analyzes task descriptions and recommends the most relevant project track(s) — helping course coordinators automatically tag and route tasks.

**Key achievements:**
- Designed and built a full ETL pipeline and star-schema data warehouse using **SSIS**.
- Built 5 interactive **Power BI** dashboards (Home, Students, Course Projects, Teams, Instructors).
- Fine-tuned a **BERT** multi-label classifier to recommend project tracks from task descriptions, achieving **87% classification accuracy**.

---

## 🏗️ Architecture

### 1. Data Pipeline (ETL — SSIS)
Task assignment data flows through a chain of lookup transformations that enrich each task record before loading it into the fact table:

```
Task Assignment Source
      │
      ▼
  TaskLookUp → TrackLookUp → CourseLookUp → StudentLookUp
      │
      ▼
  InstructorLookUp → ProjectLookUp → SkillLookup → TeamLookup
      │
      ▼
  AssignedAtLookup → SubmittedAtLookup
      │
      ▼
  FctTask (Fact Table)
```

Each lookup resolves foreign keys against its corresponding dimension (Task, Track, Course, Student, Instructor, Project, Skill, Team) before the enriched row is written to the fact table.

### 2. Star Schema (Data Warehouse)

The warehouse is modeled as a classic star schema centered on **FactTaskPerformance**:

```
        DimInstructor   DimStudent   DimTeam
              │             │          │
DimCourse ──► FactTaskPerformance ◄── DimSkills
              │             │          │
        DimTrack        DimTask   DimCourseProject
```

**Fact table:** `FactTaskPerformance`
**Dimensions:** `DimInstructor`, `DimStudent`, `DimTeam`, `DimCourse`, `DimSkills`, `DimTrack`, `DimTask`, `DimCourseProject`

This structure supports fast aggregations across any combination of student, instructor, team, course, project, track, task, or skill.

---

## 📊 Power BI Dashboards

### 🏠 Home
High-level KPIs across the whole platform:
- Total Teams (2K), Num of Students (382), Total Instructors (11)
- Avg Members per Team (4.00), Num of Course Projects (36)
- Students Avg Score (3), Avg Project Duration (11.22)

### 👩‍🎓 Students
- Num of Students, Avg/Max/Min Score
- Score distribution by course (dropdown-driven "Analyze By")
- Top 5 skills among students (UML, Java, Bash, XML, SSIS)
- Avg student score distribution (histogram) and trend by month

### 📁 Course Project
- Num of course projects, Avg Project Duration, Avg Max Size, Avg Task Weight
- Top 5 projects by task count & duration
- Top tracks required across projects (Backend, Database, Frontend, ML/AI)
- Avg task weight per track
- % of course projects per semester

### 👥 Teams
- Total Teams, Avg Members per Team, Avg Teams per Project, Performance Gap
- Best 3 teams by performance
- Team distribution by semester
- Top 5 academic projects by performance & engagement
- Impact of team size on student score

### 🧑‍🏫 Instructors
- Total Instructors, Avg Course per Instructor, Avg Instructor per Course, Avg Team per Instructor
- Instructor workload vs. average score given (evaluation patterns)
- Course distribution across instructors

---

## 🤖 ML Component — Project Track Classifier

A fine-tuned **BERT** (`bert-base-uncased`) multi-label classification model that reads a task's description and predicts which track(s) it belongs to: **Backend, Frontend, Database, ML/AI**.

### Training Pipeline (`train.py`)
1. **Data loading** — parses a nested JSON of projects → tasks, extracting `task_description` and `track` per task.
2. **Track normalization** — maps raw/free-text track labels to 4 canonical classes using keyword matching (`backend`, `frontend`, `database`, `ml`/`ai`).
3. **Label encoding** — `MultiLabelBinarizer` turns tracks into multi-hot vectors.
4. **Train/test split** — 80/20 split.
5. **Tokenization** — BERT tokenizer, max length 128, padded/truncated.
6. **Class imbalance handling** — per-class `pos_weight` computed from label frequency, fed into `BCEWithLogitsLoss`.
7. **Model** — `BertForSequenceClassification` configured for `multi_label_classification` with 4 output labels.
8. **Custom Trainer** — overrides `compute_loss` to use weighted BCE loss.
9. **Training** — 5 epochs, batch size 8, learning rate 2e-5, CPU training.
10. **Threshold tuning** — per-class decision thresholds are swept (0.25–0.75) and the value maximizing F1 is kept for each track — this is what pushes the model from a generic 0.5 cutoff to per-class optimized behavior.
11. **Model export** — model, tokenizer, and a `meta.json` (classes + thresholds) are saved for inference.

### Inference Pipeline (`predict.py`)
Given a raw task description, the model:
1. Runs the text through the fine-tuned BERT model and applies **sigmoid** to get per-class probabilities.
2. Applies a **keyword boost** (×1.1) when track-specific keywords appear in the text (e.g. "api", "endpoint" → Backend; "react", "css" → Frontend; "sql", "schema" → Database; "regression", "accuracy" → ML/AI) — this compensates for cases where the model's confidence is borderline but the text is lexically unambiguous.
3. Filters classes using the **per-class tuned thresholds** from `meta.json`. If nothing passes, falls back to the single highest-probability class.
4. Applies **conflict resolution rules** between overlapping tracks:
   - If ML/AI confidence > 0.7 and Backend is also present → down-weight Backend (×0.6).
   - If Database confidence > 0.7 and Backend is also present → down-weight Backend (×0.7).
5. **Normalizes** remaining scores so they sum to 1, and returns tracks sorted by confidence — giving a clean, interpretable multi-label output like:
   ```
   [("Backend", 0.62), ("Database", 0.38)]
   ```

### Result
**87% classification accuracy** on held-out task descriptions, used to automatically suggest the correct project track(s) for new tasks.

---

## 🛠️ Tech Stack

| Layer | Tools |
|---|---|
| ETL / Data Warehouse | SSIS, SQL Server (star schema) |
| BI / Visualization | Power BI |
| ML Model | Python, PyTorch, HuggingFace Transformers (BERT), scikit-learn |
| Data Handling | pandas, numpy, datasets |

---

## 📂 Suggested Repo Structure

```
fari2y/
├── etl/
│   └── ssis_package/            # SSIS project (Task Assignment ETL)
├── warehouse/
│   └── star_schema.sql          # Dimension & fact table DDL
├── dashboards/
│   └── fari2y.pbix              # Power BI report
├── ml/
│   ├── train.py                 # BERT multi-label training pipeline
│   ├── predict.py               # Inference + keyword boosting + conflict rules
│   └── TrackModel/              # Saved model, tokenizer, meta.json
└── README.md
```

---

## 📈 Future Improvements
- Expand track taxonomy beyond 4 classes (e.g. DevOps, QA, UI/UX design).
- Replace hand-tuned keyword boosting with a learned calibration layer.
- Automate model retraining as new course project data arrives.
- Add drill-through in Power BI from Course Project → Task-level track predictions.

---

## 👤 Author
Graduation Project — Faculty of Computers & Artificial Intelligence, Helwan University.
