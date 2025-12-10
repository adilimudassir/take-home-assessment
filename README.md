# Laravel Developer Technical Assessment
## Online University Platform - Upgrade & Enhancement Project

---

## Overview

You have inherited a Laravel 8 codebase for "UniLearn" - an online university platform. Your task is to upgrade the system to Laravel 12 and implement critical new features that the university needs for the upcoming semester.

---

## Background

UniLearn is experiencing rapid growth with 50,000+ students across 200+ courses. The platform needs:
- Better performance through intelligent caching
- Secure document handling for assignments and course materials
- Scalable background processing for bulk operations
- Enhanced security with 2FA for instructors and admins
- AWS infrastructure for global content delivery

---

## Starting Point

- Fresh Laravel 8 installation with basic authentication
- MySQL database
- You'll create sample data via seeders
- AWS credentials provided (or use LocalStack for local development)

---

## Part 1: Framework Upgrade

### Task 1.1: Upgrade Laravel 8 to Laravel 12

Upgrade the application from Laravel 8 to Laravel 12.

**Deliverable:**
- Working Laravel 12 application
- `docs/UPGRADE_REPORT.md` documenting:
  - Major breaking changes encountered
  - Deprecated features you replaced
  - Configuration changes made
  - Your step-by-step upgrade approach

---

## Part 2: Course Material Management System

### Task 2.1: AWS S3 File Management with Queue Processing

**Scenario:** Instructors upload various course materials (lecture videos, PDFs, slides, assignments). Files are currently stored locally, causing storage issues.

**Build a file management system with:**

#### Requirements:

1. **File Upload**
   - Support: PDF, DOCX, PPTX, MP4, ZIP files
   - Max size: 100MB for videos, 50MB for documents
   - Store in S3 with structure: `{university_id}/courses/{course_id}/{material_type}/{filename}`
   - Use both public and private S3 buckets (public for course materials, private for assignments)

2. **Asynchronous Processing Pipeline**
   
   When instructor uploads a file, implement this job chain:
   - Validate file type and size (synchronous)
   - Upload to S3 (queued job)
   - Extract metadata: file size, video duration, PDF page count (queued job)
   - Generate thumbnail: video first frame or PDF first page (queued job)
   - Notify all enrolled students via email (queued job)
   - Update course material cache

3. **AWS S3 Integration**
   - Generate presigned URLs for secure downloads (15-minute expiry)
   - Implement multipart upload for large video files
   - Configure proper CORS settings
   - Handle S3 exceptions gracefully

4. **Repository Pattern**
   ```
   Create:
   - CourseMaterialRepositoryInterface
   - CourseMaterialRepository (implementation)
   
   Methods to implement:
   - store()
   - findByCourse()
   - delete()
   - search()
   ```

5. **Error Handling**
   - Retry failed jobs (max 3 attempts with exponential backoff)
   - Log failures with full context
   - Send error notifications to instructors

**Questions to Answer in `docs/FILE_PROCESSING_ARCHITECTURE.md`:**
1. How did you structure your queue jobs and handle the processing pipeline?
2. What happens if S3 upload fails midway? Explain your error handling.
3. How do you prevent students from accessing materials from courses they're not enrolled in?
4. How would you handle uploading a 500MB video file?

**Routes & Views Required:**
```
POST   /courses/{id}/materials/upload   (Upload form submission)
GET    /courses/{id}/materials          (List materials view)
GET    /materials/{id}                  (Material details view)
DELETE /materials/{id}                  (Delete action)
GET    /materials/{id}/download         (Download action)
```

---

### Task 2.2: Intelligent Caching Strategy

**Scenario:** The platform serves 50,000+ students. Every page load queries the database multiple times. Response times are slow during peak hours (8-10 AM).

**Implement comprehensive caching:**

#### Requirements:

1. **Cache These Data Points:**
   - Student's enrolled courses list (tag: `student.{id}.courses`)
   - Course details with instructor info (tag: `course.{id}.details`)
   - Course enrollment count (tag: `course.{id}.stats`)
   - Instructor's course list (tag: `instructor.{id}.courses`)
   - University-wide statistics (tag: `university.stats`)

2. **Cache TTL Configuration:**
   ```
   - Student course list: 30 minutes
   - Course details: 2 hours
   - Enrollment counts: 5 minutes
   - University stats: 10 minutes
   - S3 presigned URLs: 5 minutes
   ```

3. **Cache Invalidation Rules:**
   
   **When student enrolls in a course:**
   - Invalidate student's course list cache
   - Invalidate course enrollment count cache
   - Invalidate university stats cache
   
   **When course details updated:**
   - Invalidate course details cache
   - Invalidate all enrolled students' course lists
   
   **When instructor changed:**
   - Invalidate course details
   - Invalidate old and new instructor's course lists

4. **Implement:**
   - Cache-aside pattern with automatic fallback to database
   - Cache warming for popular courses (enrollment > 100 students)
   - Artisan command to warm cache: `php artisan cache:warm-courses`
   - Artisan command to clear specific tags: `php artisan cache:clear-tags {tag1,tag2}`

5. **Repository Caching Decorator:**
   ```
   Create CachedCourseRepository that wraps CourseRepository
   and transparently handles caching
   ```

6. **Create CacheService:**
   - Bind to IoC container
   - Centralize cache operations
   - Handle cache tags management

**Questions to Answer in `docs/CACHING_STRATEGY.md`:**
1. Explain `Cache::remember()` vs cache repository decorator. Which did you use and why?
2. How does your invalidation strategy prevent stale data?
3. How do you prevent cache stampede during high traffic?
4. What happens if Redis goes down?
5. Explain how you used Laravel's IoC container for your caching services.

---

### Task 2.3: Bulk Operations with Advanced Queues

**Scenario:** The registrar needs to process:
- Enroll 5,000 students into courses from CSV
- Generate and email certificates to 10,000 graduating students
- Send course reminders to 50,000 students

These operations currently timeout and crash the server.

**Build a bulk processing system:**

#### Requirements:

1. **CSV Batch Enrollment:**
   - Upload CSV with columns: `student_email, course_code, semester`
   - Parse and validate CSV (queued job)
   - Process enrollments in batches of 100 (batched jobs)
   - Send confirmation emails (separate queue)
   - Track progress in real-time

2. **Certificate Generation:**
   - Generate PDF certificates with student data
   - Upload certificates to S3
   - Email certificate download link to student
   - Process in batches to avoid mail server overload

3. **Queue Configuration:**
   - Use named queues: `critical`, `emails`, `default`, `low`
   - Implement proper job priorities
   - Rate limiting: max 100 emails per minute
   - Retry failed jobs: 1min, 5min, 15min (exponential backoff)

4. **Job Monitoring Dashboard:**
   
   Create simple Blade view showing:
   - Pending jobs per queue
   - Failed jobs count
   - Last 10 completed batch operations
   - Average processing time

5. **Custom Job Middleware:**
   
   Create middleware for:
   - Logging job execution time
   - Preventing duplicate job execution
   - Rate limiting external calls

6. **Artisan Commands:**
   ```
   php artisan queue:monitor           # Show queue statistics
   php artisan students:bulk-enroll {csv_path}
   php artisan certificates:generate {semester}
   ```

**Questions to Answer in `docs/QUEUE_ARCHITECTURE.md`:**
1. Explain your batch processing approach and why you chose your batch size.
2. How do you handle scenarios where 500 out of 5,000 jobs fail?
3. What's your strategy for preventing duplicate enrollments?
4. How do you prioritize certificate jobs vs regular emails?
5. Explain the difference between `dispatch()`, `dispatchSync()`, and `dispatchAfterResponse()`.
6. How does Laravel job middleware work?

---



### Task 2.5: Assignment Submission System

**Scenario:** Students submit assignments. The system needs file uploads, plagiarism checking (simulated), grading workflows, and notifications.

**Build complete submission system:**

#### Requirements:

1. **Submission Flow:**
   - Student uploads assignment file (PDF, DOCX, ZIP)
   - System validates file and deadline
   - Upload to private S3 bucket: `submissions/{course_id}/{assignment_id}/{student_id}/`
   - Run plagiarism check (simulate with 5-second delay job)
   - Calculate similarity percentage (random 0-100 for simulation)
   - Notify instructor of new submission
   - Update relevant caches

2. **Repository Pattern:**
   ```
   Create:
   - AssignmentRepositoryInterface + Implementation
   - SubmissionRepositoryInterface + Implementation
   - Both with cached decorator implementations
   ```

3. **Queue Pipeline:**
   ```
   Upload → Validate → S3 Upload → Plagiarism Check → Notify Instructor
   
   Each step is a separate job
   Handle failures gracefully at each stage
   ```

4. **Business Rules:**
   - Students cannot submit after deadline
   - Only one active submission per student per assignment
   - Students can resubmit before deadline (archives previous submission)
   - Instructors can download submissions with presigned URLs

5. **Caching:**
   - Cache assignment submissions list per assignment (tag: `assignment.{id}.submissions`)
   - Cache individual submission details (tag: `submission.{id}`)
   - Invalidate when new submission or grade added

**Routes & Views Required:**
```
POST   /assignments/{id}/submit      (Submit form submission)
GET    /assignments/{id}/submissions (Instructor view: all submissions)
GET    /submissions/{id}             (View submission details)
POST   /submissions/{id}/grade       (Grade submission form)
GET    /submissions/{id}/download    (Download submission)
DELETE /submissions/{id}             (Student action: before deadline only)
```

**Questions to Answer in `docs/ASSIGNMENT_SYSTEM.md`:**
1. How do you prevent submissions after deadline?
2. What happens if plagiarism check fails? How do you handle retries?
3. How do you prevent concurrent submissions from the same student?

4. Explain your cache invalidation strategy for this feature.
5. How would you test S3 integration without hitting AWS?

---



## Final Deliverables

### 1. Source Code Structure
```
/app
  /Http
    /Controllers
      - CourseMaterialController.php
      - AssignmentController.php
      - SubmissionController.php
    - CacheServiceProvider.php
    - StorageServiceProvider.php

/database
  /migrations
  /seeders

/docs
  - UPGRADE_REPORT.md
  - FILE_PROCESSING_ARCHITECTURE.md
  - CACHING_STRATEGY.md
  - QUEUE_ARCHITECTURE.md
  - ASSIGNMENT_SYSTEM.md

```

### 2. Configuration Files
- `.env.example` with detailed comments
- All custom config files properly documented

### 3. Database
- All migrations
- Seeders with realistic test data (at least 100 students, 20 courses, 10 instructors)

### 4. Application Documentation
- `routes/web.php` should be well organized
- Brief guide on how to navigate the app as an instructor/student

### 5. Testing (Optional but Recommended)
- Feature tests for critical workflows
- Unit tests for services
- Demonstrate mocking S3 in tests

### 6. README.md
Must include:
- Project overview
- Environment setup instructions
- AWS configuration steps
- Database setup commands
- Queue worker setup
- How to run tests
- Troubleshooting common issues

### 7. Video Walkthrough (10 minutes)
Record yourself explaining:
- Your architectural decisions
- Most challenging part
- Your architectural decisions
- Most challenging part
- Your caching strategy
- Your caching strategy
- Queue processing approach

### 8. Deployment
- Deploy the application to a hosting provider of your choice (e.g., AWS, DigitalOcean, Heroku, Railway).
- Ensure the prototype is running and accessible online.
- Verify that critical features (file uploads, background jobs) are functional in the deployed environment.

---

## Submission

1. Create a private GitHub repository
2. Make regular commits with meaningful messages
3. Include all documentation in `/docs` folder
4. Ensure `.env.example` is complete
5. Submit: Repository link + Video link + Deployed URL

---



## Important Notes

- **Document Assumptions:** If requirements are unclear, document your assumptions
- **Ask Questions:** Create `QUESTIONS.md` with questions you'd ask the team
- **Be Pragmatic:** Real-world solutions over perfect solutions
- **Security First:** Treat this as production code
- **Explain Trade-offs:** Show you understand pros/cons of your decisions

---

## Bonus (Optional)

- Implement Laravel Horizon for queue monitoring
- Add full-text search using Laravel Scout
- Implement event sourcing for submissions
- Docker configuration for local setup
- CI/CD pipeline configuration
- WebSocket notifications with Laravel Echo

---

**Good luck! Show us not just that you can code, but that you understand Laravel's architectural patterns and the "why" behind them.**