# Tài liệu Toàn bộ Cơ sở dữ liệu (Database Schema) — Unihub

Toàn bộ hệ thống sử dụng **1 instance PostgreSQL 16 (`pgvector/pgvector:pg16`)** nhưng tách thành **7 database độc lập** theo mô hình **Database per Service** (không join chéo DB; tham chiếu giữa các service bằng UUID và sự kiện RabbitMQ).

---

## Tổng quan 7 Database

| Database | Service sở hữu | Trạng thái | Mục đích |
|---|---|---|---|
| `identity_db` | `identity-service` | **Đã triển khai (Sprint 1)** | Tài khoản, vai trò, phiên đăng nhập, reset mật khẩu, audit log |
| `academic_db` | `academic-service` | **Đã triển khai (S1 + S2)** | Danh mục học thuật (S1) + Lớp học phần & Đăng ký học phần (S2) |
| `learning_db` | `learning-service` | **Đã triển khai (S2 + S3)** | Chương/bài học, tài liệu, tiến độ, bài tập, nộp bài (S2); hỏi đáp bài giảng (S3) |
| `assessment_db` | `assessment-service` | **Đã triển khai (S3)** | Ngân hàng câu hỏi, quiz, điểm thành phần, bảng điểm, điểm danh |
| `notification_db` | `notification-service` | **Đã triển khai (S3)** | Thông báo in-app, thông báo lớp, idempotency sự kiện |
| `report_db` | `report-service` | *Thiết kế cho Sprint 4* | Read-model tổng hợp dashboard và cảnh báo học vụ |
| `ai_db` | `ai-service` | *Thiết kế cho Sprint 4* | Vector embedding tài liệu (`pgvector` + Gemini API) và lịch sử chat RAG |

---

## 1. `identity_db` (`identity-service`) — *Đã triển khai*

- **`roles`**: `id` (SMALLSERIAL PK), `name` (VARCHAR 50 UK: `ADMIN`, `LECTURER`, `STUDENT`), `description`.
- **`users`**: `id` (UUID PK), `email` (UK), `password_hash`, `full_name`, `code` (UK), `avatar_file_id`, `status` (`ACTIVE` | `LOCKED` | `DELETED`), `department_id` (UUID), `admin_class_id` (UUID), `created_at`, `updated_at`, `deleted_at`.
- **`user_roles`**: `user_id` (FK → `users.id`), `role_id` (FK → `roles.id`), PK `(user_id, role_id)`.
- **`refresh_tokens`**: `id` (UUID PK), `user_id` (FK → `users.id`), `token_hash` (UK, SHA-256), `expires_at`, `revoked` (BOOLEAN), `created_at`.
- **`password_reset_tokens`**: `id` (UUID PK), `user_id` (FK → `users.id`), `token_hash` (UK, SHA-256), `expires_at`, `used` (BOOLEAN), `created_at`.
- **`audit_logs`**: `id` (UUID PK), `actor_id` (UUID), `actor_email`, `action`, `target_type`, `target_id`, `before_data` (JSONB), `after_data` (JSONB), `created_at`.

---

## 2. `academic_db` (`academic-service`)

### Phần đã triển khai (Sprint 1 — `ACD-*`)
- **`faculties`**: `id` (UUID PK), `code` (UK), `name`, `created_at`.
- **`majors`**: `id` (UUID PK), `faculty_id` (FK → `faculties.id`), `code` (UK), `name`, `created_at`.
- **`admin_classes`**: `id` (UUID PK), `major_id` (FK → `majors.id`), `code` (UK), `name`, `cohort_year` (INT), `created_at`.
- **`admin_class_students`**: `admin_class_id` (FK → `admin_classes.id`), `student_id` (UUID ref `identity_db.users`), `assigned_at`, PK `(admin_class_id, student_id)`.
- **`semesters`**: `id` (UUID PK), `code` (UK), `name`, `start_date`, `end_date`, `is_current` (BOOLEAN), `registration_start`, `registration_end`.
- **`subjects`**: `id` (UUID PK), `faculty_id` (FK → `faculties.id`), `code` (UK), `name`, `credits` (INT), `description` (TEXT), `created_at`.
- **`subject_prerequisites`**: `subject_id` (FK → `subjects.id`), `prerequisite_subject_id` (FK → `subjects.id`), PK `(subject_id, prerequisite_subject_id)`.

### Phần đã triển khai (Sprint 2 — `CRS-*`)
- **`course_sections`**: `id` (UUID PK), `semester_id` (FK → `semesters.id`), `subject_id` (FK → `subjects.id`), `code` (UK), `max_capacity` (INT), `enrolled_count` (INT), `room`, `status` (`OPEN` | `CLOSED`), `version` (BIGINT optimistic lock), `created_at`.
- **`course_section_schedules`**: `id` (UUID PK), `course_section_id` (FK → `course_sections.id`), `day_of_week` (SMALLINT 2–8), `start_period` (SMALLINT), `end_period` (SMALLINT), `room`.
- **`course_section_lecturers`**: `course_section_id` (FK → `course_sections.id`), `lecturer_id` (UUID ref `identity_db.users`), PK `(course_section_id, lecturer_id)`.
- **`enrollments`**: `id` (UUID PK), `course_section_id` (FK → `course_sections.id`), `student_id` (UUID ref `identity_db.users`), `status` (`ENROLLED` | `DROPPED`), `enrolled_at`, `dropped_at`, UK `(course_section_id, student_id)`.

---

## 3. `learning_db` (`learning-service`) — *Đã triển khai (Sprint 2–3)*

- **`chapters`**: `id` (UUID PK), `course_section_id` (UUID ref `academic_db`), `title`, `sort_order` (INT), `published` (BOOLEAN), `created_at`.
- **`lessons`**: `id` (UUID PK), `chapter_id` (FK → `chapters.id`), `title`, `content` (TEXT), `video_url`, `video_file_id` (ref `file-service`), `sort_order` (INT), `published` (BOOLEAN), `created_at`.
- **`lesson_attachments`**: `id` (UUID PK), `lesson_id` (FK → `lessons.id`), `file_id` (ref `file-service`), `file_name`, `content_type`, `file_size`, `created_at`.
- **`lesson_progress`**: `lesson_id` (FK → `lessons.id`), `student_id` (UUID), `completed_at`, PK `(lesson_id, student_id)`.
- **`assignments`**: `id` (UUID PK), `course_section_id` (UUID), `title`, `description` (TEXT), `attachment_file_id`, `due_at` (TIMESTAMPTZ), `max_score` (NUMERIC), `allow_late` (BOOLEAN), `published` (BOOLEAN), `created_at`.
- **`submissions`**: `id` (UUID PK), `assignment_id` (FK → `assignments.id`), `student_id` (UUID), `file_id`, `text_content` (TEXT), `version` (INT), `is_late` (BOOLEAN), `score` (NUMERIC), `feedback` (TEXT), `graded_by` (UUID), `submitted_at`, `graded_at`.
- **`lesson_comments`**: `id` (UUID PK), `lesson_id` (FK → `lessons.id`), `parent_id` (FK → `lesson_comments.id`, nullable), `author_id` (UUID), `content` (TEXT), `is_best_answer` (BOOLEAN), `hidden` (BOOLEAN), `created_at`.

---

## 4. `assessment_db` (`assessment-service`) — *Đã triển khai (Sprint 3)*

- **`questions`**: `id` (UUID PK), `subject_id` (UUID ref `academic_db`), `created_by` (UUID), `type` (`SINGLE_CHOICE` | `MULTIPLE_CHOICE`), `content` (TEXT), `options_json` (JSONB), `correct_answers_json` (JSONB), `difficulty` (`EASY` | `MEDIUM` | `HARD`), `topic`, `tags` (TEXT[]), `created_at`.
- **`quizzes`**: `id` (UUID PK), `course_section_id` (UUID), `title`, `duration_minutes` (INT), `max_attempts` (INT), `shuffle_questions` (BOOLEAN), `show_correct_answers` (BOOLEAN), `open_at`, `close_at`, `published` (BOOLEAN), `created_at`.
- **`quiz_questions`**: `quiz_id` (FK → `quizzes.id`), `question_id` (FK → `questions.id`), `sort_order` (INT), `points` (NUMERIC), PK `(quiz_id, question_id)`.
- **`quiz_attempts`**: `id` (UUID PK), `quiz_id` (FK → `quizzes.id`), `student_id` (UUID), `attempt_number` (INT), `started_at`, `submitted_at`, `score` (NUMERIC), `status` (`IN_PROGRESS` | `SUBMITTED` | `GRADED`).
- **`quiz_answers`**: `attempt_id` (FK → `quiz_attempts.id`), `question_id` (FK → `questions.id`), `selected_answers_json` (JSONB), `is_correct` (BOOLEAN), `earned_points` (NUMERIC), PK `(attempt_id, question_id)`.
- **`grade_components`**: `id` (UUID PK), `course_section_id` (UUID), `name`, `weight_percent` (NUMERIC), `linked_type` (`ASSIGNMENT` | `QUIZ` | `MANUAL`), `linked_id` (UUID), `locked` (BOOLEAN).
- **`student_grades`**: `id` (UUID PK), `course_section_id` (UUID), `student_id` (UUID), `component_id` (FK → `grade_components.id`), `score` (NUMERIC), `locked` (BOOLEAN), `updated_by` (UUID), `updated_at`, UK `(student_id, component_id)`.
- **`attendance_sessions`**: `id` (UUID PK), `course_section_id` (UUID), `session_date` (DATE), `topic`, `created_by` (UUID), `created_at`.
- **`attendance_records`**: `session_id` (FK → `attendance_sessions.id`), `student_id` (UUID), `status` (`PRESENT` | `ABSENT` | `LATE` | `EXCUSED`), `note`, PK `(session_id, student_id)`.

---

## 5. `notification_db` (`notification-service`) — *Đã triển khai (Sprint 3)*

- **`notifications`**: `id` (UUID PK), `recipient_id` (UUID), `type`, `title`, `body` (TEXT), `reference_type`, `reference_id`, `is_read` (BOOLEAN), `created_at`.
- **`class_announcements`**: `id` (UUID PK), `course_section_id` (UUID), `author_id` (UUID), `title`, `content` (TEXT), `pinned` (BOOLEAN), `created_at`, `updated_at`.
- **`processed_events`**: `event_id` (UUID PK), `event_type`, `processed_at` *(đảm bảo idempotency khi consume RabbitMQ)*.

---

## 6. `report_db` (`report-service`) — *Dự kiến Sprint 4*

- **`student_academic_summaries`**: `id` (UUID PK), `student_id` (UUID), `semester_id` (UUID), `course_section_id` (UUID), `attendance_rate` (NUMERIC), `current_score` (NUMERIC), `progress_percent` (NUMERIC), `updated_at`, UK `(student_id, course_section_id)`.
- **`alert_rules`**: `id` (UUID PK), `name`, `max_absences` (INT), `min_score` (NUMERIC), `active` (BOOLEAN).
- **`academic_alerts`**: `id` (UUID PK), `student_id` (UUID), `course_section_id` (UUID), `rule_id` (FK → `alert_rules.id`), `reason` (TEXT), `resolved` (BOOLEAN), `created_at`.

---

## 7. `ai_db` (`ai-service`, có `CREATE EXTENSION vector`) — *Dự kiến Sprint 4*

- **`document_chunks`**: `id` (UUID PK), `course_section_id` (UUID — dùng lọc phân quyền RAG), `lesson_id` (UUID), `file_id`, `chunk_index` (INT), `content` (TEXT), `embedding` (`vector(768)` — khớp dimension của model embedding Google Gemini API), `metadata` (JSONB), `created_at`.
- **`ai_conversations`**: `id` (UUID PK), `user_id` (UUID), `course_section_id` (UUID), `title`, `created_at`.
- **`ai_messages`**: `id` (UUID PK), `conversation_id` (FK → `ai_conversations.id`), `role` (`USER` | `ASSISTANT`), `content` (TEXT), `citations_json` (JSONB), `created_at`.
