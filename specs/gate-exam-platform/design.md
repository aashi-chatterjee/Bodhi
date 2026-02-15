# Design Document: GATE Exam Platform

## Overview

The GATE Exam Platform is a full-stack web application built with React (Vite) on the frontend and Node.js/Express on the backend, using PostgreSQL as the primary database. The platform provides five core features: PYQ Database with filtering, AI-powered explanations with caching, subject-wise flashcards, an adaptive QuizBot with chat UI, and a realistic Exam Simulator.

The architecture follows a clean separation between presentation (React components), business logic (Express services), and data persistence (PostgreSQL). AI explanation generation is handled server-side with aggressive caching to minimize API costs. The system is designed for fast load times, minimal UI clutter, and robust error handling.

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Frontend (React)                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │   PYQ    │  │Flashcard │  │ QuizBot  │  │   Exam   │   │
│  │ Browser  │  │  Viewer  │  │   Chat   │  │Simulator │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│         │              │              │              │       │
│         └──────────────┴──────────────┴──────────────┘       │
│                          │                                    │
│                    API Client Layer                          │
└──────────────────────────┼──────────────────────────────────┘
                           │ HTTPS/REST
┌──────────────────────────┼──────────────────────────────────┐
│                    Backend (Express)                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              API Routes & Controllers                 │   │
│  └──────────────────────────────────────────────────────┘   │
│         │              │              │              │       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │Question  │  │Flashcard │  │   Quiz   │  │   Exam   │   │
│  │ Service  │  │ Service  │  │ Service  │  │ Service  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│         │              │              │              │       │
│         └──────────────┴──────────────┴──────────────┘       │
│                          │                                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │          AI Explanation Service (with cache)         │   │
│  └──────────────────────────────────────────────────────┘   │
│                          │                                    │
└──────────────────────────┼──────────────────────────────────┘
                           │
┌──────────────────────────┼──────────────────────────────────┐
│                   PostgreSQL Database                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │Questions │  │Flashcards│  │Quiz      │  │Mock      │   │
│  │          │  │          │  │Attempts  │  │Tests     │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │
│  │Users     │  │Explanation│ │Performance│                 │
│  │          │  │Cache     │  │Metrics   │                  │
│  └──────────┘  └──────────┘  └──────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

### Technology Stack

- **Frontend**: React 18+ with Vite, React Router, Axios for API calls
- **Backend**: Node.js 18+ with Express 4.x
- **Database**: PostgreSQL 14+ with pg driver
- **AI Service**: OpenAI API or compatible service
- **Styling**: CSS Modules or Tailwind CSS for minimal academic design
- **State Management**: React Context API or Zustand for lightweight state management

### Design Principles

1. **Separation of Concerns**: Frontend handles presentation, backend handles business logic and AI integration
2. **Caching First**: AI explanations are cached aggressively to minimize API costs
3. **Modular Architecture**: Each feature is self-contained with its own routes, services, and components
4. **Performance**: Indexed database queries, lazy loading, and optimized bundle sizes
5. **Error Resilience**: Graceful degradation when AI service is unavailable

## Components and Interfaces

### Frontend Components

#### 1. PYQ Browser Component
```typescript
interface PYQBrowserProps {
  // No props needed - manages own state
}

interface Question {
  id: string;
  year: number;
  subject: string;
  topic: string;
  question_type: 'MCQ' | 'NAT' | 'MSQ';
  question_text: string;
  options?: string[];
  correct_answer: string;
  marks: number;
}

interface FilterState {
  subjects: string[];
  topics: string[];
  years: number[];
  types: string[];
}
```

**Responsibilities**:
- Render filter controls for subject, topic, year, and question type
- Fetch filtered questions from backend API
- Display questions in card UI with show/hide answer toggle
- Request AI explanations on demand

#### 2. Flashcard Viewer Component
```typescript
interface FlashcardViewerProps {
  // No props needed - manages own state
}

interface Flashcard {
  id: string;
  subject: string;
  topic: string;
  card_type: 'formula' | 'concept' | 'common_mistake';
  front_content: string;
  back_content: string;
  is_difficult: boolean;
}

interface FlashcardFilters {
  subjects: string[];
  topics: string[];
  types: string[];
  difficultOnly: boolean;
}
```

**Responsibilities**:
- Render flashcard with flip animation
- Handle subject/topic/type filtering
- Allow marking cards as difficult
- Navigate between cards

#### 3. QuizBot Chat Component
```typescript
interface QuizBotProps {
  userId: string;
}

interface ChatMessage {
  id: string;
  sender: 'user' | 'bot';
  content: string;
  timestamp: Date;
  questionData?: Question;
}

interface QuizSession {
  sessionId: string;
  topic: string;
  currentDifficulty: 'easy' | 'medium' | 'hard';
  questionsAnswered: number;
  correctAnswers: number;
}
```

**Responsibilities**:
- Render chat-style UI for quiz interactions
- Send user quiz requests to backend
- Display questions as chat messages
- Collect user answers and show instant feedback
- Track session performance

#### 4. Exam Simulator Component
```typescript
interface ExamSimulatorProps {
  mode: 'practice' | 'real';
  userId: string;
}

interface ExamSession {
  sessionId: string;
  mode: 'practice' | 'real';
  startTime: Date;
  duration: number; // minutes
  sections: ExamSection[];
  currentQuestionIndex: number;
}

interface ExamSection {
  name: string;
  questions: Question[];
  timeLimit?: number;
}

interface QuestionStatus {
  questionId: string;
  answered: boolean;
  markedForReview: boolean;
  userAnswer?: string;
}
```

**Responsibilities**:
- Display countdown timer
- Render question navigation panel with status indicators
- Handle question navigation and answer recording
- Provide virtual calculator
- Submit exam and display results with analysis

### Backend API Endpoints

#### Question Endpoints
```
GET /api/questions
  Query params: subject, topic, year, type
  Response: { questions: Question[] }

GET /api/questions/:id
  Response: { question: Question }

GET /api/questions/:id/explanation
  Response: { 
    questionId: string,
    beginnerExplanation: string,
    examExplanation: string,
    cached: boolean
  }
```

#### Flashcard Endpoints
```
GET /api/flashcards
  Query params: subject, topic, type, difficultOnly
  Response: { flashcards: Flashcard[] }

POST /api/flashcards/:id/mark-difficult
  Body: { isDifficult: boolean }
  Response: { success: boolean }
```

#### Quiz Endpoints
```
POST /api/quiz/start
  Body: { userId: string, topic: string }
  Response: { 
    sessionId: string,
    question: Question
  }

POST /api/quiz/answer
  Body: { 
    sessionId: string,
    questionId: string,
    userAnswer: string
  }
  Response: {
    correct: boolean,
    correctAnswer: string,
    nextQuestion?: Question,
    sessionComplete: boolean
  }

GET /api/quiz/performance/:userId
  Response: {
    totalQuizzes: number,
    averageAccuracy: number,
    topicWisePerformance: { topic: string, accuracy: number }[]
  }
```

#### Exam Endpoints
```
POST /api/exam/start
  Body: { userId: string, mode: 'practice' | 'real' }
  Response: {
    sessionId: string,
    sections: ExamSection[],
    duration: number
  }

POST /api/exam/submit-answer
  Body: {
    sessionId: string,
    questionId: string,
    userAnswer: string,
    markedForReview: boolean
  }
  Response: { success: boolean }

POST /api/exam/submit
  Body: { sessionId: string }
  Response: {
    score: number,
    totalMarks: number,
    sectionWiseScores: { section: string, score: number }[],
    topicWiseAnalysis: { topic: string, correct: number, total: number }[],
    timeTaken: number
  }
```

### Backend Services

#### Question Service
```typescript
interface QuestionService {
  getQuestions(filters: FilterState): Promise<Question[]>;
  getQuestionById(id: string): Promise<Question>;
  getRandomQuestionsByTopic(topic: string, difficulty: string, count: number): Promise<Question[]>;
}
```

#### AI Explanation Service
```typescript
interface AIExplanationService {
  getExplanation(questionId: string): Promise<{
    beginnerExplanation: string;
    examExplanation: string;
    cached: boolean;
  }>;
  generateExplanation(question: Question): Promise<{
    beginnerExplanation: string;
    examExplanation: string;
  }>;
  cacheExplanation(questionId: string, explanations: any): Promise<void>;
}
```

**Implementation Notes**:
- Check cache first before calling AI API
- Use structured prompts for consistent explanation format
- Implement rate limiting middleware
- Handle API errors gracefully

#### Quiz Service
```typescript
interface QuizService {
  startQuizSession(userId: string, topic: string): Promise<{
    sessionId: string;
    question: Question;
  }>;
  submitAnswer(sessionId: string, questionId: string, userAnswer: string): Promise<{
    correct: boolean;
    correctAnswer: string;
    nextQuestion?: Question;
  }>;
  adjustDifficulty(sessionId: string, accuracy: number): string; // returns new difficulty
  getPerformanceMetrics(userId: string): Promise<PerformanceMetrics>;
}
```

**Adaptive Difficulty Logic**:
- Track last 5 answers in session
- If accuracy >= 80%, increase difficulty
- If accuracy <= 40%, decrease difficulty
- Otherwise maintain current difficulty

#### Exam Service
```typescript
interface ExamService {
  startExamSession(userId: string, mode: string): Promise<ExamSession>;
  submitAnswer(sessionId: string, questionId: string, answer: string, markedForReview: boolean): Promise<void>;
  submitExam(sessionId: string): Promise<ExamResults>;
  calculateScore(session: ExamSession): ExamResults;
}
```

## Data Models

### Database Schema (PostgreSQL)

```sql
-- Users table
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  name VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Questions table
CREATE TABLE questions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  year INTEGER NOT NULL,
  subject VARCHAR(100) NOT NULL,
  topic VARCHAR(100) NOT NULL,
  question_type VARCHAR(10) NOT NULL CHECK (question_type IN ('MCQ', 'NAT', 'MSQ')),
  question_text TEXT NOT NULL,
  options JSONB, -- Array of options for MCQ/MSQ
  correct_answer TEXT NOT NULL,
  marks DECIMAL(4,2) NOT NULL,
  difficulty VARCHAR(10) DEFAULT 'medium' CHECK (difficulty IN ('easy', 'medium', 'hard')),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  -- Indexes for filtering
  INDEX idx_subject (subject),
  INDEX idx_topic (topic),
  INDEX idx_year (year),
  INDEX idx_type (question_type),
  INDEX idx_difficulty (difficulty)
);

-- Explanation cache table
CREATE TABLE explanation_cache (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  question_id UUID NOT NULL REFERENCES questions(id) ON DELETE CASCADE,
  beginner_explanation TEXT NOT NULL,
  exam_explanation TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  UNIQUE(question_id),
  INDEX idx_question_id (question_id)
);

-- Flashcards table
CREATE TABLE flashcards (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  subject VARCHAR(100) NOT NULL,
  topic VARCHAR(100) NOT NULL,
  card_type VARCHAR(20) NOT NULL CHECK (card_type IN ('formula', 'concept', 'common_mistake')),
  front_content TEXT NOT NULL,
  back_content TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  INDEX idx_flashcard_subject (subject),
  INDEX idx_flashcard_topic (topic),
  INDEX idx_flashcard_type (card_type)
);

-- User flashcard difficulty tracking
CREATE TABLE user_flashcard_difficulty (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  flashcard_id UUID NOT NULL REFERENCES flashcards(id) ON DELETE CASCADE,
  is_difficult BOOLEAN DEFAULT false,
  marked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  UNIQUE(user_id, flashcard_id),
  INDEX idx_user_flashcard (user_id, flashcard_id)
);

-- Quiz sessions table
CREATE TABLE quiz_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  topic VARCHAR(100) NOT NULL,
  current_difficulty VARCHAR(10) DEFAULT 'medium',
  questions_answered INTEGER DEFAULT 0,
  correct_answers INTEGER DEFAULT 0,
  started_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  completed_at TIMESTAMP,
  
  INDEX idx_quiz_user (user_id),
  INDEX idx_quiz_topic (topic)
);

-- Quiz attempts table (individual question attempts within a session)
CREATE TABLE quiz_attempts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id UUID NOT NULL REFERENCES quiz_sessions(id) ON DELETE CASCADE,
  question_id UUID NOT NULL REFERENCES questions(id) ON DELETE CASCADE,
  user_answer TEXT NOT NULL,
  correct_answer TEXT NOT NULL,
  is_correct BOOLEAN NOT NULL,
  time_taken INTEGER, -- seconds
  answered_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  INDEX idx_attempt_session (session_id),
  INDEX idx_attempt_question (question_id)
);

-- Mock test sessions table
CREATE TABLE mock_test_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  mode VARCHAR(10) NOT NULL CHECK (mode IN ('practice', 'real')),
  duration INTEGER NOT NULL, -- minutes
  started_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  submitted_at TIMESTAMP,
  total_score DECIMAL(6,2),
  total_marks DECIMAL(6,2),
  
  INDEX idx_mock_user (user_id),
  INDEX idx_mock_mode (mode)
);

-- Mock test answers table
CREATE TABLE mock_test_answers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id UUID NOT NULL REFERENCES mock_test_sessions(id) ON DELETE CASCADE,
  question_id UUID NOT NULL REFERENCES questions(id) ON DELETE CASCADE,
  section_name VARCHAR(100) NOT NULL,
  user_answer TEXT,
  correct_answer TEXT NOT NULL,
  is_correct BOOLEAN,
  marked_for_review BOOLEAN DEFAULT false,
  time_taken INTEGER, -- seconds
  
  INDEX idx_mock_answer_session (session_id),
  INDEX idx_mock_answer_question (question_id)
);

-- Performance metrics table (aggregated statistics)
CREATE TABLE performance_metrics (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  metric_type VARCHAR(50) NOT NULL, -- 'quiz', 'mock_test', 'topic_accuracy'
  metric_key VARCHAR(100), -- topic name or subject name
  metric_value JSONB NOT NULL, -- flexible storage for various metrics
  calculated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  INDEX idx_perf_user (user_id),
  INDEX idx_perf_type (metric_type),
  INDEX idx_perf_key (metric_key)
);
```

### Entity Relationship Diagram

```
┌─────────────┐
│    Users    │
└──────┬──────┘
       │
       │ 1:N
       │
       ├──────────────────────────────────────┐
       │                                      │
       │                                      │
┌──────┴──────────────┐              ┌───────┴────────────┐
│  Quiz Sessions      │              │ Mock Test Sessions │
│  - topic            │              │ - mode             │
│  - difficulty       │              │ - duration         │
│  - questions_count  │              │ - total_score      │
└──────┬──────────────┘              └───────┬────────────┘
       │                                     │
       │ 1:N                                 │ 1:N
       │                                     │
┌──────┴──────────────┐              ┌───────┴────────────┐
│  Quiz Attempts      │              │ Mock Test Answers  │
│  - user_answer      │              │ - user_answer      │
│  - is_correct       │              │ - is_correct       │
│  - time_taken       │              │ - marked_review    │
└──────┬──────────────┘              └───────┬────────────┘
       │                                     │
       │ N:1                                 │ N:1
       │                                     │
       └─────────────┬───────────────────────┘
                     │
              ┌──────┴──────────┐
              │   Questions     │
              │  - year         │
              │  - subject      │
              │  - topic        │
              │  - type         │
              │  - text         │
              │  - options      │
              │  - answer       │
              │  - marks        │
              └──────┬──────────┘
                     │
                     │ 1:1
                     │
              ┌──────┴──────────────┐
              │ Explanation Cache   │
              │  - beginner_exp     │
              │  - exam_exp         │
              └─────────────────────┘

┌─────────────┐
│  Flashcards │
│  - subject  │
│  - topic    │
│  - type     │
│  - front    │
│  - back     │
└──────┬──────┘
       │
       │ N:M (through user_flashcard_difficulty)
       │
┌──────┴──────┐
│    Users    │
└─────────────┘
```


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Question Metadata Completeness
*For any* question object being stored in the database, it must contain all required fields (id, year, subject, topic, question_type, question_text, correct_answer, marks) with valid types, and questions missing any required field must be rejected.

**Validates: Requirements 1.1, 1.5**

### Property 2: Filter Correctness
*For any* collection of questions and any combination of filters (subject, topic, year, question_type), all returned questions must match every applied filter criterion, and no questions matching all criteria should be excluded.

**Validates: Requirements 1.2, 3.3**

### Property 3: UI State Preservation
*For any* question or flashcard, toggling visibility state (showing/hiding answers, flipping cards) must not mutate the underlying data, and the original content must remain unchanged after any number of state transitions.

**Validates: Requirements 1.3, 1.4, 3.2**

### Property 4: Cache-First Explanation Retrieval
*For any* question with a cached explanation, requesting the explanation must return the cached version without making an AI API call, and the cached explanation must be identical to the originally generated explanation.

**Validates: Requirements 2.2, 2.5**

### Property 5: Explanation Structure Completeness
*For any* question requesting an AI explanation, the response must contain both a beginner_explanation and an exam_explanation field, and both fields must be non-empty strings with proper formatting markers.

**Validates: Requirements 2.1, 2.3**

### Property 6: Flashcard Type Validation
*For any* flashcard being created or stored, the card_type must be one of 'formula', 'concept', or 'common_mistake', and flashcards with invalid types must be rejected.

**Validates: Requirements 3.1**

### Property 7: Difficulty Marking Persistence
*For any* flashcard marked as difficult by a user, the marking must persist across sessions, and filtering by difficulty status must return all and only the flashcards marked as difficult by that user.

**Validates: Requirements 3.4**

### Property 8: Flashcard Organization
*For any* set of flashcards, they must be groupable by subject and topic such that all flashcards in a group share the same subject and topic values.

**Validates: Requirements 3.5**

### Property 9: Quiz Topic Consistency
*For any* quiz session on a specific topic, all questions generated for that session must belong to the requested topic, with no questions from other topics included.

**Validates: Requirements 4.1**

### Property 10: Answer Feedback Correctness
*For any* user answer to a quiz question, the feedback must indicate correctness if and only if the user answer matches the correct answer (accounting for answer format normalization).

**Validates: Requirements 4.2**

### Property 11: Performance Tracking Completeness
*For any* quiz attempt, the system must record all required fields (timestamp, user_id, question_id, user_answer, correct_answer, time_taken, is_correct) in the database.

**Validates: Requirements 4.3, 9.4, 9.5**

### Property 12: Adaptive Difficulty Adjustment
*For any* quiz session, if the accuracy over the last 5 questions is >= 80%, the difficulty must increase; if accuracy is <= 40%, the difficulty must decrease; otherwise, difficulty must remain unchanged.

**Validates: Requirements 4.4, 4.5**

### Property 13: Chat Message Format
*For any* message in the QuizBot interface, it must have a sender field ('user' or 'bot'), a content field, and a timestamp, and messages containing questions must include the complete question data.

**Validates: Requirements 4.6**

### Property 14: Exam State Preservation During Navigation
*For any* exam session, navigating between questions must preserve all previously submitted answers and marked-for-review statuses, such that returning to a question shows the exact state it was left in.

**Validates: Requirements 5.2**

### Property 15: Review Status Synchronization
*For any* question marked for review in an exam session, the question navigation panel must reflect this status, and unmarking must remove the status from the navigation panel.

**Validates: Requirements 5.3**

### Property 16: Exam Section Organization
*For any* exam session, questions must be organized into sections, and each question must belong to exactly one section matching the GATE exam structure.

**Validates: Requirements 5.4**

### Property 17: Exam Mode Configuration
*For any* exam session, practice mode and real mode must have different time constraints, with real mode matching actual GATE exam duration and practice mode allowing flexible timing.

**Validates: Requirements 5.6**

### Property 18: Score Calculation Accuracy
*For any* completed exam session, the total score must equal the sum of marks for all correctly answered questions, and section-wise scores must sum to the total score.

**Validates: Requirements 5.7**

### Property 19: Authentication Enforcement
*For any* API request to protected endpoints, requests without valid authentication tokens must be rejected with appropriate HTTP error codes (401 or 403), and no protected operations should be performed.

**Validates: Requirements 7.1, 7.4**

### Property 20: Rate Limiting Enforcement
*For any* sequence of AI explanation requests from a single client, requests exceeding the configured rate limit must be rejected with a descriptive error, and the rejection count must not affect the rate limit counter.

**Validates: Requirements 7.2, 7.5**

### Property 21: Input Validation with Descriptive Errors
*For any* invalid input submitted to the platform (malformed data, missing required fields, invalid types), the system must reject the input and return a specific error message indicating which field is invalid and why.

**Validates: Requirements 8.3**

### Property 22: Network Retry with Exponential Backoff
*For any* failed network request, the system must retry with exponentially increasing delays (e.g., 1s, 2s, 4s) up to a maximum number of attempts before returning an error to the user.

**Validates: Requirements 8.5**

### Property 23: Referential Integrity Enforcement
*For any* attempt to create a record with a foreign key reference, the referenced record must exist, and attempts to delete a record with dependent records must either cascade delete or be rejected.

**Validates: Requirements 9.2**

### Property 24: Responsive Layout Adaptation
*For any* viewport size (desktop >= 1024px, tablet 768-1023px, mobile < 768px), the layout must adapt appropriately with no horizontal scrolling and all interactive elements remaining accessible.

**Validates: Requirements 11.2**

### Property 25: REST API Compliance
*For any* API endpoint, it must use the appropriate HTTP method (GET for retrieval, POST for creation, PUT/PATCH for updates, DELETE for deletion) and return appropriate status codes (200, 201, 400, 401, 404, 500) based on the operation result.

**Validates: Requirements 11.4**

### Property 26: Server-Side AI Prompt Handling
*For any* AI explanation request, the prompt construction and API communication must occur entirely on the backend, with no AI API keys or direct AI service calls present in frontend code.

**Validates: Requirements 11.8**

## Error Handling

### Frontend Error Handling

1. **API Request Failures**
   - Wrap all API calls in try-catch blocks
   - Display user-friendly error messages in toast notifications or error boundaries
   - Provide retry options for transient failures
   - Log errors to console in development mode

2. **Component Error Boundaries**
   - Implement React error boundaries at feature level
   - Display fallback UI when component crashes
   - Log error details for debugging
   - Provide "Return to Home" option

3. **Form Validation**
   - Validate inputs on blur and submit
   - Display inline error messages
   - Prevent submission of invalid data
   - Highlight invalid fields clearly

4. **Loading States**
   - Show loading indicators for async operations
   - Disable interactive elements during loading
   - Implement timeout for long-running requests
   - Provide cancel option for user-initiated requests

### Backend Error Handling

1. **Request Validation**
   - Validate all incoming request data
   - Return 400 Bad Request with specific error messages
   - Use validation middleware (e.g., express-validator)
   - Sanitize inputs to prevent injection attacks

2. **Database Errors**
   - Catch database connection errors
   - Handle constraint violations gracefully
   - Return 500 Internal Server Error for unexpected database issues
   - Log detailed error information for debugging
   - Implement connection pooling and retry logic

3. **AI Service Errors**
   - Implement circuit breaker pattern for AI service calls
   - Return cached explanations when AI service is down
   - Provide fallback error message to users
   - Log AI service failures for monitoring
   - Implement timeout for AI requests (30 seconds)

4. **Authentication/Authorization Errors**
   - Return 401 Unauthorized for missing/invalid tokens
   - Return 403 Forbidden for insufficient permissions
   - Log unauthorized access attempts
   - Implement rate limiting to prevent brute force attacks

5. **Rate Limiting**
   - Use express-rate-limit middleware
   - Configure different limits for different endpoint types
   - Return 429 Too Many Requests with Retry-After header
   - Implement per-user and per-IP rate limiting

### Error Response Format

All API errors should follow a consistent format:

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "User-friendly error message",
    "details": {
      "field": "specific_field",
      "reason": "Detailed reason for developers"
    }
  }
}
```

## Testing Strategy

### Dual Testing Approach

The platform requires both unit testing and property-based testing for comprehensive coverage:

- **Unit tests**: Verify specific examples, edge cases, and error conditions
- **Property tests**: Verify universal properties across all inputs
- Both approaches are complementary and necessary

### Unit Testing

**Focus Areas**:
- Specific examples demonstrating correct behavior
- Edge cases (empty inputs, boundary values, special characters)
- Error conditions (invalid inputs, service failures, network errors)
- Integration points between components
- Database constraint violations
- Authentication and authorization flows

**Testing Balance**:
- Avoid writing too many unit tests for scenarios covered by property tests
- Focus unit tests on concrete examples and integration scenarios
- Use unit tests to document expected behavior through examples

**Tools**:
- Frontend: Vitest + React Testing Library
- Backend: Jest + Supertest for API testing
- Database: In-memory PostgreSQL or test database

**Example Unit Tests**:
```javascript
// Example: Test specific error case
test('should reject question with missing subject field', async () => {
  const invalidQuestion = {
    year: 2023,
    topic: 'Algorithms',
    question_text: 'What is...',
    // missing subject field
  };
  
  const response = await request(app)
    .post('/api/questions')
    .send(invalidQuestion);
  
  expect(response.status).toBe(400);
  expect(response.body.error.details.field).toBe('subject');
});

// Example: Test authentication flow
test('should reject unauthenticated requests to protected endpoints', async () => {
  const response = await request(app)
    .get('/api/quiz/performance/user123');
  
  expect(response.status).toBe(401);
});
```

### Property-Based Testing

**Configuration**:
- Library: fast-check (JavaScript/TypeScript)
- Minimum 100 iterations per property test
- Each test must reference its design document property
- Tag format: `Feature: gate-exam-platform, Property {number}: {property_text}`

**Property Test Implementation**:
Each correctness property must be implemented as a single property-based test that:
1. Generates random valid inputs
2. Executes the operation
3. Verifies the property holds
4. Reports counterexamples when property fails

**Example Property Tests**:

```javascript
// Property 2: Filter Correctness
test('Feature: gate-exam-platform, Property 2: Filter Correctness', () => {
  fc.assert(
    fc.property(
      fc.array(questionArbitrary), // Generate random questions
      fc.record({ // Generate random filters
        subject: fc.option(fc.constantFrom('Math', 'Physics', 'CS')),
        topic: fc.option(fc.string()),
        year: fc.option(fc.integer({ min: 2010, max: 2024 })),
        type: fc.option(fc.constantFrom('MCQ', 'NAT', 'MSQ'))
      }),
      (questions, filters) => {
        const filtered = applyFilters(questions, filters);
        
        // All returned questions must match all filters
        return filtered.every(q => 
          (!filters.subject || q.subject === filters.subject) &&
          (!filters.topic || q.topic === filters.topic) &&
          (!filters.year || q.year === filters.year) &&
          (!filters.type || q.question_type === filters.type)
        );
      }
    ),
    { numRuns: 100 }
  );
});

// Property 4: Cache-First Explanation Retrieval
test('Feature: gate-exam-platform, Property 4: Cache-First Explanation Retrieval', () => {
  fc.assert(
    fc.property(
      questionArbitrary,
      async (question) => {
        // First request generates and caches
        const firstResponse = await getExplanation(question.id);
        const apiCallCount1 = mockAIService.callCount;
        
        // Second request should use cache
        const secondResponse = await getExplanation(question.id);
        const apiCallCount2 = mockAIService.callCount;
        
        // Verify cache was used (no additional API call)
        return apiCallCount2 === apiCallCount1 &&
               deepEqual(firstResponse, secondResponse);
      }
    ),
    { numRuns: 100 }
  );
});

// Property 12: Adaptive Difficulty Adjustment
test('Feature: gate-exam-platform, Property 12: Adaptive Difficulty Adjustment', () => {
  fc.assert(
    fc.property(
      fc.array(fc.boolean(), { minLength: 5, maxLength: 5 }), // Last 5 answers
      (answers) => {
        const accuracy = answers.filter(a => a).length / 5;
        const currentDifficulty = 'medium';
        const newDifficulty = adjustDifficulty(currentDifficulty, answers);
        
        if (accuracy >= 0.8) {
          return newDifficulty === 'hard';
        } else if (accuracy <= 0.4) {
          return newDifficulty === 'easy';
        } else {
          return newDifficulty === 'medium';
        }
      }
    ),
    { numRuns: 100 }
  );
});
```

**Arbitrary Generators**:
Create custom generators for domain objects:

```javascript
const questionArbitrary = fc.record({
  id: fc.uuid(),
  year: fc.integer({ min: 2010, max: 2024 }),
  subject: fc.constantFrom('Mathematics', 'Physics', 'Computer Science'),
  topic: fc.string({ minLength: 3, maxLength: 50 }),
  question_type: fc.constantFrom('MCQ', 'NAT', 'MSQ'),
  question_text: fc.string({ minLength: 10, maxLength: 500 }),
  options: fc.option(fc.array(fc.string(), { minLength: 4, maxLength: 4 })),
  correct_answer: fc.string(),
  marks: fc.constantFrom(1, 2),
  difficulty: fc.constantFrom('easy', 'medium', 'hard')
});

const flashcardArbitrary = fc.record({
  id: fc.uuid(),
  subject: fc.constantFrom('Mathematics', 'Physics', 'Computer Science'),
  topic: fc.string({ minLength: 3, maxLength: 50 }),
  card_type: fc.constantFrom('formula', 'concept', 'common_mistake'),
  front_content: fc.string({ minLength: 10, maxLength: 200 }),
  back_content: fc.string({ minLength: 10, maxLength: 500 }),
  is_difficult: fc.boolean()
});
```

### Integration Testing

**Focus Areas**:
- End-to-end API flows
- Database transactions and rollbacks
- AI service integration with caching
- Authentication middleware chain
- Rate limiting across multiple requests

**Tools**:
- Supertest for API integration tests
- Test database with seed data
- Mock AI service for predictable responses

### Performance Testing

**Key Metrics**:
- API response time < 200ms for cached data
- API response time < 2s for AI explanations
- Database query time < 100ms for filtered questions
- Frontend initial load < 2s
- Exam simulator handles 100+ questions smoothly

**Tools**:
- Artillery or k6 for load testing
- Lighthouse for frontend performance
- Database query analysis tools

### Test Coverage Goals

- Unit test coverage: > 80% for business logic
- Property test coverage: All 26 correctness properties
- Integration test coverage: All API endpoints
- E2E test coverage: Critical user flows (quiz, exam simulation)
