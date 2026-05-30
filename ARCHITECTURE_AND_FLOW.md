# AI Mock Interview Assistant - Detailed Architecture and Flow Guide

## Table of Contents
1. [Project Overview](#project-overview)
2. [System Architecture](#system-architecture)
3. [Component Details](#component-details)
4. [Data Flow](#data-flow)
5. [Technology Stack](#technology-stack)
6. [Installation & Setup](#installation--setup)
7. [Usage Guide](#usage-guide)

---

## Project Overview

The **AI Mock Interview Assistant** is an intelligent Python-based application designed to simulate realistic interview scenarios. It helps candidates prepare for job interviews by:

- **Parsing** their resume and job description
- **Generating** personalized interview questions (technical, behavioral, competency-based)
- **Evaluating** candidate responses with detailed feedback
- **Recommending** learning resources based on performance gaps

This application leverages advanced NLP (Natural Language Processing), LLMs (Large Language Models), and machine learning to provide a comprehensive interview preparation experience.

---

## System Architecture

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Streamlit UI Layer                            │
│                  (User Interface & Session Management)               │
└─────────────────────────────────────────────────────────────────────┘
                                   ↓
    ┌──────────────────────────────────────────────────────────────┐
    │                   Application Logic (app.py)                 │
    │         - Orchestrates workflow                             │
    │         - Manages session state                             │
    │         - Coordinates between components                    │
    └──────────────────────────────────────────────────────────────┘
                                   ↓
    ┌──────────────────────────────────────────────────────────────┐
    │                    Core Processing Layer                     │
    ├──────────────────────────────────────────────────────────────┤
    │ ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
    │ │ CV Parser    │  │ JD Parser    │  │ Question     │         │
    │ │              │  │              │  │ Generator    │         │
    │ │ (utils/)     │  │ (utils/)     │  │ (models/)    │         │
    │ └──────────────┘  └──────────────┘  └──────────────┘         │
    │                                                               │
    │ ┌──────────────┐  ┌──────────────┐                           │
    │ │ Answer       │  │ Resource     │                           │
    │ │ Evaluator    │  │ Recommender  │                           │
    │ │ (models/)    │  │ (models/)    │                           │
    │ └──────────────┘  └──────────────┘                           │
    └──────────────────────────────────────────────────────────────┘
                                   ↓
    ┌──────────────────────────────────────────────────────────────┐
    │              External Services & Libraries                   │
    ├──────────────────────────────────────────────────────────────┤
    │  • Groq LLM (GPT-like inference)                            │
    │  • spaCy (NLP & Named Entity Recognition)                  │
    │  • pdfplumber (PDF extraction)                             │
    │  • python-docx (DOCX extraction)                           │
    │  • LangChain (LLM framework)                               │
    └──────────────────────────────────────────────────────────────┘
```

### Directory Structure

```
AI-Mock-Interview_/
├── app.py                          # Main Streamlit application
├── requirements.txt                # Project dependencies
├── utils/
│   ├── cv_parser.py               # Resume parsing logic
│   └── JD_parser.py               # Job description parsing logic
├── models/
│   ├── question_generator.py       # Question generation logic
│   ├── answer_evaluator.py         # Answer evaluation & feedback
│   └── resource_recommender.py     # Resource recommendation engine
├── logs/                           # Application logs
└── .env                            # Environment variables (GROQ_API_KEY)
```

---

## Component Details

### 1. **CVParser** (`utils/cv_parser.py`)

**Purpose**: Extracts structured information from user's resume.

**Key Responsibilities**:
- Load PDF and DOCX files
- Extract text from documents
- Parse sections: experience, education, skills
- Use spaCy NLP for entity recognition

**Key Methods**:
```python
load_file(file_path)           # Load and extract text from PDF/DOCX
parse_cv(file_path)            # Main parsing method
_extract_experience(doc)        # Extract work experience
_extract_education(doc)         # Extract education details
_extract_skills(doc)            # Extract technical & soft skills
```

**Output Example**:
```python
{
    "experience": [
        "Software Engineer at Tech Company (2020-2023)",
        "Python, AWS, Docker development"
    ],
    "education": [
        "B.S. Computer Science, University"
    ],
    "skills": ["Python", "AWS", "Docker", "REST APIs"]
}
```

---

### 2. **JobDescriptionParser** (`utils/JD_parser.py`)

**Purpose**: Extracts job requirements and responsibilities from job descriptions.

**Key Responsibilities**:
- Parse job description text
- Extract required skills using regex patterns
- Identify responsibilities and qualifications
- Determine years of experience required
- Generate interview focus areas

**Key Methods**:
```python
parse_job_description(text)    # Main parsing method
generate_interview_focus(parsed_data)  # Create focus areas
```

**Output Example**:
```python
{
    "required_skills": ["Python", "Django", "PostgreSQL"],
    "responsibilities": ["Develop REST APIs", "Manage databases"],
    "qualifications": ["3+ years experience", "B.S. Computer Science"],
    "years_experience": 3,
    "job_title": "Senior Backend Engineer"
}
```

---

### 3. **QuestionGenerator** (`models/question_generator.py`)

**Purpose**: Generates customized interview questions based on resume and job description.

**Question Types**:
1. **TECHNICAL** - Technical depth and problem-solving
2. **BEHAVIORAL** - Communication, teamwork, conflict resolution
3. **COMPETENCY** - Role-specific competency assessment

**Key Features**:
- LLM-powered (Groq) question generation
- Context-aware prompting using CV and JD data
- 15 unique questions per type
- Mix of difficulty levels and question formats

**Key Methods**:
```python
generate_questions(question_type, cv_context, jd_context)
get_question_prompt(question_type)  # Get prompt template
format_questions(response)          # Parse LLM response
```

**Example Output**:
```python
[
    "Tell me about a complex Python project you've worked on...",
    "How would you handle a disagreement with a team member?",
    "Describe your experience with REST API design..."
]
```

---

### 4. **InterviewAnswerEvaluator** (`models/answer_evaluator.py`)

**Purpose**: Evaluates candidate responses and provides detailed feedback.

**Key Responsibilities**:
- Assess technical accuracy and depth
- Evaluate communication clarity
- Score relevance to the question
- Provide constructive feedback
- Store evaluation history

**Evaluation Metrics**:
- **Relevance Score** (0-100): How well the answer addresses the question
- **Technical Accuracy**: Correctness and depth of technical details
- **Communication Clarity**: How well ideas are articulated
- **Suggestions for Improvement**: Actionable feedback

**Key Methods**:
```python
evaluate_answer(answer, question, cv_context, jd_context)
_evaluate_answer_with_groq(prompt)  # LLM evaluation
_safe_float_conversion(value)       # Parse numeric scores
initialize_session_state()          # Setup session storage
```

**Output Example**:
```python
{
    "question_number": 1,
    "relevance_score": "85",
    "technical_accuracy": "Well-explained with good examples...",
    "communication_clarity": "Clear and structured response...",
    "overall_feedback": "Strong answer with minor gaps in...",
    "suggestions": "Consider elaborating on..."
}
```

---

### 5. **ResourceRecommender** (`models/resource_recommender.py`)

**Purpose**: Recommends personalized learning resources based on performance gaps.

**Key Responsibilities**:
- Identify weak areas from interview feedback
- Suggest relevant learning materials
- Provide skill development resources
- Recommend role-specific preparation materials

**Key Methods**:
```python
get_recommendations(cv_data, interview_type, interview_feedback, technical_stack)
_identify_weak_areas(interview_feedback)  # Parse feedback for gaps
_get_groq_recommendations(weak_areas, technical_stack)  # Generate recommendations
```

**Output Example**:
```python
{
    "priority": [
        {"title": "Advanced Python Patterns", "url": "...", "type": "course"}
    ],
    "skill_development": [
        {"title": "System Design Course", "url": "...", "type": "course"}
    ],
    "interview_prep": [
        {"title": "Mock Interview Guide", "url": "...", "type": "guide"}
    ],
    "additional": [...]
}
```

---

### 6. **Main Application** (`app.py`)

**Purpose**: Orchestrates the entire interview workflow through Streamlit UI.

**Key Features**:
- Session state management
- Document upload handling
- Interview workflow control
- Real-time progress tracking

**Application Flow Steps**:
1. **Upload Phase**: Accept resume and job description
2. **Parsing Phase**: Process documents
3. **Question Generation**: Create tailored questions
4. **Interview Phase**: Present questions and collect answers
5. **Evaluation Phase**: Evaluate responses
6. **Feedback Phase**: Display results and recommendations

**Key Methods**:
```python
load_api_key()                    # Load Groq API key
save_uploadedfile(uploadedfile)  # Handle file uploads
initialize_session_state()        # Setup session variables
cleanup_temp_files()             # Cleanup resources
process_documents()              # Coordinate parsing
```

---

## Data Flow

### Complete Workflow Diagram

```
START
  ↓
┌─────────────────────────────────┐
│ User Uploads Resume & Job Desc  │
└─────────────────────────────────┘
  ↓
┌─────────────────────────────────┐
│ Save to Temporary Files         │
└─────────────────────────────────┘
  ↓
┌─────────────────────────────────────────┐
│ Parse Documents                         │
│ ├─ CV Parser → Extract skills, exp     │
│ └─ JD Parser → Extract requirements    │
└─────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────┐
│ User Selects Interview Type             │
│ (Technical / Behavioral / Competency)   │
└─────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────┐
│ Question Generator                      │
│ ├─ Send CV + JD context to Groq LLM   │
│ ├─ Generate 15 unique questions        │
│ └─ Store in session state              │
└─────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────┐
│ Interview Loop                          │
│ for each question:                      │
│ ├─ Display question                    │
│ ├─ Collect user response               │
│ ├─ Evaluate with AnswerEvaluator      │
│ ├─ Store evaluation                    │
│ └─ Move to next question               │
└─────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────┐
│ Generate Resource Recommendations       │
│ ├─ Identify weak areas                 │
│ ├─ Use ResourceRecommender             │
│ └─ Create personalized learning plan   │
└─────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────┐
│ Display Results & Feedback              │
│ ├─ Overall performance metrics         │
│ ├─ Per-question evaluations            │
│ ├─ Resource recommendations            │
│ └─ Areas for improvement               │
└─────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────┐
│ Cleanup Temp Files                      │
└─────────────────────────────────────────┘
  ↓
END
```

### Detailed Step-by-Step Flow

#### Phase 1: Document Processing
```
User Input (Resume + JD)
       ↓
    [Save to temp files]
       ↓
    ┌─────────────────────────────┐
    │ CVParser.parse_cv()         │
    ├─ Extract text (PDF/DOCX)   │
    ├─ Run spaCy NLP             │
    ├─ Extract sections          │
    └─ Return structured data    │
       ↓
    ┌─────────────────────────────┐
    │ JDParser.parse_job_desc()   │
    ├─ Extract text              │
    ├─ Regex pattern matching    │
    ├─ Extract skills/reqs       │
    └─ Return structured data    │
       ↓
    [Store in session_state]
```

#### Phase 2: Question Generation
```
Interview Type Selection
       ↓
    ┌─────────────────────────────────────┐
    │ QuestionGenerator.generate_questions │
    ├─ Select question type prompt       │
    ├─ Format context (CV + JD)          │
    ├─ Send to Groq LLM                  │
    ├─ Parse response                    │
    └─ Store 15 questions               │
       ↓
    [Display first question]
```

#### Phase 3: Interview & Evaluation
```
For Each Question:
       ↓
    [Display Question]
       ↓
    [Collect User Response]
       ↓
    ┌───────────────────────────────┐
    │ AnswerEvaluator.evaluate()     │
    ├─ Create evaluation prompt     │
    ├─ Send to Groq LLM            │
    ├─ Parse JSON response         │
    ├─ Extract metrics             │
    └─ Store in session            │
       ↓
    [Display Feedback]
       ↓
    [Continue to Next Question]
```

#### Phase 4: Resource Recommendation
```
After Interview Complete
       ↓
    ┌──────────────────────────────┐
    │ ResourceRecommender.get_recs  │
    ├─ Analyze evaluations         │
    ├─ Identify weak areas         │
    ├─ Send to Groq for ideas      │
    ├─ Match against resource DB   │
    └─ Return recommendations      │
       ↓
    [Display Resources by Priority]
```

---

## Technology Stack

### Core Framework
- **Streamlit** (v1.35.0) - Web UI framework
- **Python 3.8+** - Programming language

### NLP & Language Models
- **spaCy** - Named Entity Recognition and NLP
- **Groq API** - Fast LLM inference
- **LangChain** - LLM framework and utilities
  - `langchain-groq`: Groq integration
  - `langchain-core`: Core abstractions
  - `langchain-community`: Additional tools

### Document Processing
- **pdfplumber** - PDF text extraction
- **python-docx** - DOCX parsing
- **PyPDF2** - PDF utilities

### Data & Environment
- **pandas** - Data manipulation
- **python-dotenv** - Environment variable management

### Development & Logging
- Standard logging library - Application logging

---

## Installation & Setup

### Prerequisites
- Python 3.8 or higher
- pip package manager
- Groq API key (free from [groq.com](https://groq.com))

### Step-by-Step Installation

1. **Clone the Repository**
   ```bash
   git clone https://github.com/niti007/AI-Mock-Interview_.git
   cd AI-Mock-Interview_
   ```

2. **Create Virtual Environment**
   ```bash
   python -m venv venv
   
   # On Windows
   venv\Scripts\activate
   
   # On macOS/Linux
   source venv/bin/activate
   ```

3. **Install Dependencies**
   ```bash
   pip install -r requirements.txt
   ```

4. **Download spaCy Model**
   ```bash
   python -m spacy download en_core_web_sm
   ```

5. **Setup Environment Variables**
   ```bash
   # Create .env file in project root
   echo "GROQ_API_KEY=your_api_key_here" > .env
   ```

6. **Run the Application**
   ```bash
   streamlit run app.py
   ```

   The application will open at `http://localhost:8501`

---

## Usage Guide

### User Workflow

#### Step 1: Upload Documents
- Upload your resume (PDF or DOCX format)
- Upload the job description (PDF, DOCX, or TXT)
- Click "Process Documents"

#### Step 2: Select Interview Type
Choose from:
- **Technical Interview**: Deep technical knowledge assessment
- **Behavioral Interview**: Soft skills and communication
- **Competency-Based**: Role-specific competencies

#### Step 3: Answer Questions
- Read each question carefully
- Provide detailed, thoughtful answers
- Click "Submit Answer" after each response

#### Step 4: Review Feedback
- View evaluation scores and metrics
- Read constructive feedback
- Identify strengths and areas for improvement

#### Step 5: View Recommendations
- Review personalized learning resources
- Prioritized by your performance gaps
- Includes courses, tutorials, and guides

### Best Practices

1. **Resume Upload**
   - Ensure resume is well-formatted
   - Include all relevant skills and experience
   - Use clear section headers

2. **Job Description**
   - Use the actual JD from the job posting
   - Include all requirements and responsibilities
   - Keep formatting consistent

3. **Interview Responses**
   - Speak clearly and naturally
   - Provide specific examples
   - Address all parts of the question
   - Structure answers logically (STAR method for behavioral)

4. **Using Feedback**
   - Focus on high-priority recommendations
   - Create a study plan based on weak areas
   - Practice similar questions multiple times

---

## Key Design Patterns

### 1. **Modular Architecture**
Each component is independent, allowing easy testing and updates:
- Parsers: Document extraction
- Models: Core AI logic
- UI: User interaction layer

### 2. **Session State Management**
Streamlit session state maintains:
- Current workflow step
- Parsed documents
- Generated questions
- Evaluation history

### 3. **Error Handling**
Comprehensive logging and error recovery:
- Try-catch blocks in critical functions
- Detailed logging for debugging
- User-friendly error messages

### 4. **Resource Cleanup**
Temporary files are managed:
- Created in system temp directory
- Deleted after processing
- Prevents disk space issues

### 5. **LLM Integration Pattern**
Consistent Groq API usage:
- Centralized initialization
- Prompt engineering for consistency
- Error handling for API failures

---

## Performance Considerations

### Optimization Strategies
1. **Caching**: Session state caches parsed data
2. **Lazy Loading**: Components initialized only when needed
3. **Async Processing**: LLM calls handled efficiently
4. **File Cleanup**: Temporary files removed promptly

### Scalability Notes
- Current design supports single user per session
- Session isolation via Streamlit's session management
- API rate limits from Groq should be considered

---

## Troubleshooting

### Common Issues

**Issue**: "Groq API key not found"
- **Solution**: Ensure `.env` file contains `GROQ_API_KEY=your_key`

**Issue**: "spaCy model not found"
- **Solution**: Run `python -m spacy download en_core_web_sm`

**Issue**: "PDF extraction fails"
- **Solution**: Ensure PDF is not corrupted; try alternative format (DOCX)

**Issue**: "Questions seem generic"
- **Solution**: Provide more detailed resume and job description

**Issue**: "Session timeout or state loss"
- **Solution**: Check browser cache and Streamlit connection

---

## Future Enhancements

1. **Voice Input/Output**
   - Record spoken answers
   - Speech-to-text conversion
   - Evaluate communication skills

2. **Performance Analytics**
   - Track improvement over multiple attempts
   - Benchmark against industry standards
   - Progress visualization

3. **Multi-language Support**
   - Support for multiple languages
   - Localized resources
   - Regional job market data

4. **Database Integration**
   - Store interview history
   - User profiles and progress tracking
   - Analytics and insights

5. **Advanced Features**
   - Video recording of interviews
   - Peer comparison (anonymized)
   - Interview templates by role/company
   - Integration with job boards

---

## License & Credits

© 2024 AI Mock Interview Assistant. All rights reserved.
Licensed under the MIT License.

**Built with**:
- Groq API for fast inference
- LangChain for LLM orchestration
- Streamlit for beautiful UI
- spaCy for NLP excellence

---

## Support & Contributing

For issues, suggestions, or contributions:
- Open an issue on GitHub
- Submit a pull request with improvements
- Share feedback and use cases

**Contact**: niti007 (Repository Owner)

---

## Appendix: Code Snippets

### Example: Initialize Session State
```python
def initialize_session_state():
    session_vars = {
        'current_step': 'upload',
        'interview_data': {},
        'questions': [],
        'responses': [],
        'feedback': {},
    }
    for var, default in session_vars.items():
        if var not in st.session_state:
            st.session_state[var] = default
```

### Example: Parse Documents
```python
cv_parser = CVParser()
jd_parser = JobDescriptionParser()

cv_data = cv_parser.parse_cv(cv_file_path)
jd_data = jd_parser.parse_job_description(jd_file_path)

st.session_state.interview_data = {
    'cv_data': cv_data,
    'jd_data': jd_data
}
```

### Example: Generate Questions
```python
question_generator = QuestionGenerator()
questions = question_generator.generate_questions(
    question_type=QuestionType.TECHNICAL,
    cv_context=str(cv_data),
    jd_context=str(jd_data)
)
st.session_state.questions = questions
```

---

**Last Updated**: May 2024
**Version**: 1.0
**Status**: Active & Maintained
