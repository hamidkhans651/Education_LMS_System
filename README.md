# Education_LMS_System

Microservices
User Authentication Microservice
Purpose:
Manages user registration, authentication, profile management, and role-based access control (RBAC). Integrates with Kong for JWT authentication, ACL plugin, and OAuth 2.0. Uses Kafka for event-driven user registration and profile creation.

Schema:
User
id: UUID, Primary Key
username: String, Unique
password: String, Hashed
email: String, Unique
is_type: Enum (student, teacher, admin)
consumer_id: String, Kong Consumer ID
issuer: String, Kong JWT generated key, save as iss in token
token: String, Token for password reset
Profile
id: UUID, Primary Key
user_id: UUID, Foreign Key to User ( we can use the same id as user_id here which will work as the primary key in this table as well, this will save us from having two types of id for profile and user_id, instead we will have only one id as primary key which will be same to user_id)
name: String
mobile_number: String
city: String
country: String
additional_details: JSONB, Additional data specific to the user type

API Endpoints:
Registration & Authentication
POST /auth/register
Registers a new user.
Produces a Kafka event for user registration.
Returns a confirmation message.

POST /auth/login
Authenticates user credentials.
Issues a JWT token for session management.

Here, the forget password API will have two steps
POST /auth/forgot-password
Initiates the password reset process by generating a token and sending a verification code or link through email or phone number.

POST /auth/reset-password
Resets the user's password using the provided verification code which receives through email or phone number.

Profile Management
GET /auth/profile
Retrieves the logged-in user’s profile information.

PUT /auth/profile
Updates the logged-in user’s profile details.

GET /auth/profile/{user_id}
Admin only: Retrieves the profile of a specific user by ID.


Kafka Events
User Registration Event
On user registration, it produces an event with the user data.


The event is consumed by the service itself to:
Create a Kong consumer.
Assign the user to the appropriate ACL group (student, teacher, admin)
Create the user profile based on is_type.

Plugin or Service Used
Auth 2.0 will be used in fast API to manage the verification process by that, the library will be used 
Jose for the JWT token
OAuth2PasswordBearer from fastapi.security for getting tokens using dependency
CryptoContext from passlib.context for hashing password
`iss`: This key will be added to the token of JWT to get authorised by Kong JWT plugin (each user will have their issuer key)

JWT & ACL plugin will be used in kong for each consumer
Twilio for phone number or aiosmtplib for email can be used after consuming UserRegistration event in Notification Service





Student Microservice
Purpose:
The Student Microservice manages all student-related data, including personal profiles, academic progress, and report cards. It acts as a central hub for all information pertaining to a student, ensuring efficient data retrieval and updates.
Key Responsibilities:
Student Profiles:
Managing and storing student profiles, including personal details, contact information, and any other relevant data.
Academic Progress & Report Cards:
Tracking and managing student progress across courses.
Calculating and storing report card data, including grades, attendance, and other performance metrics.
Ensuring that updates from other microservices (like the Teaching Program Microservice or Exam Microservice) are reflected in real-time via Kafka.
Schema:
    Student_profile:
student_id: UUID, Primary Key
user_id: UUID, links to the User Authentication Microservice to associate the student with their user account.
student_status: String
name: String.
email: String.
mobile_number: String
city:  String
country: String
address: String
date_of_birth:  Date
enrollment_year: Integer
program: String
additional_details: JSONB (for additional performance metrics)
    Report Card:
report_id: UUID, Primary Key
student_id: UUID, Foreign Key
course_id: UUID, Foreign Key
grade: String
completion_percentage: integer
attendance: integer
remarks: Text
metadata: JSONB (for additional performance metrics)

API Endpoints:

GET /student/profile: Allow a student to get their profile details
POST /student/updateProfile: Allow a student to update their profile details like password, mobile number, or profile picture
GET /student/courseHistory: Retrieves the student's program and course history, including completed and in-progress courses. (This endpoint can merge into program microservice as well below) (Student details or id will be fetch from token)




Teaching Program Microservice
Purpose:
The Teaching Program Microservice is responsible for managing academic programs and courses within the Panaversity ecosystem. It handles the creation, updating, organization, and deletion of programs and their associated courses, focusing on providing structure and content for educational offerings.

Schema:
Program: (id, title, description) 
Course: (id, programId, title, short_description)

API Endpoints:
Program Management
POST /programs/create: Creates a new program.
PUT /programs/{programId}/update: Updates a program’s details.
DELETE /programs/{programId}/delete: Deletes a program.
GET /programs/list: Retrieves a list of all programs.
GET /programs/{programId}/details: Retrieves details of a program.

Course Management
POST /programs/{programId}/courses/create: Creates a new course within a program.
PUT /courses/{courseId}/update: Updates an existing course's details.
DELETE /courses/{courseId}/delete: Deletes a course.
GET /programs/{programId}/courses: Retrieves a list of courses within a program.
GET /courses/{courseId}/details: Fetches detailed information about a specific course, including its Table of Contents.
GET /courses/topics: Retrieves all topics and subtopics under a specific chapter
GET /courses/schedule: Retrieves scheduled exams for a specific course.




Instructor Microservice
Purpose:
Manages instructor profiles, class assignments, and communication with students.
Schema:
Instructor: (id, name, email, assignedCourses)
API Endpoints:
POST /instructors/login: Instructor login and authentication.
GET /instructors/{instructorId}/classes: Lists all classes taught by the instructor.
GET /instructors/{instructorId}/students: Retrieves students assigned to the instructor.




Notification Microservice
Purpose:
Manages notifications to students and instructors for course updates, exams, and deadlines.
Schema:
Notification: (id, recipient_id, message, type, status)
API Endpoints:
POST /notifications/send: Sends notifications (e.g., upcoming exams, new course material).
GET /notifications/{recipientId}/list: Fetches all notifications for a recipient (student/instructor).
GET /notifications/{recipientId}/list: Fetches all notifications for a recipient (student/instructor).




Progress & Feedback Microservice
Purpose:
Manages tracking student progress throughout courses and gathering feedback on courses and instructors.

Schema:
Progress: (id, studentId, courseId, currentModule, completionPercentage)
Feedback: (id, studentId, courseId, rating, comment)

API Endpoints:
Progress Tracking
POST /progress: Records new progress data for a student.
GET /progress/{studentId}/{courseId}: Retrieves progress for a specific course for a student.
PUT /progress/{id}: Updates existing progress data.

Feedback Management
POST /feedback: Submits feedback for a course.
GET /feedback/{courseId}: Retrieves all feedback for a specific course.



Course Content Microservice
Purpose:
Manages detailed course materials, including modules, lessons, and associated resources.

Schema:
Module: (id, courseId, title, description)
Lesson: (id, moduleId, title, contentUrl, duration)

API Endpoints:
Module Management
POST /courses/{courseId}/modules: Creates a new module for a course.
GET /courses/{courseId}/modules: Retrieves all modules for a course.
PUT /modules/{moduleId}: Updates a specific module.
DELETE /modules/{moduleId}: Deletes a specific module.
Lesson Management
POST /modules/{moduleId}/lessons: Adds a new lesson to a module.
GET /modules/{moduleId}/lessons: Retrieves all lessons for a module.
PUT /lessons/{lessonId}: Updates specific lesson details.
DELETE /lessons/{lessonId}: Deletes a specific lesson.




Exam Microservice
Purpose:
Handles the creation, management, and execution of exams, including grading and result distribution.

Schema:
Exam: (id, courseId, title, duration, examDate)
Question: (id, examId, questionText, choices, correctAnswer)
Result: (id, examId, studentId, score, status)

API Endpoints:
Exam Management
POST /exams: Creates a new exam.
GET /exams/{courseId}: Retrieves all exams for a course.
PUT /exams/{examId}: Updates an exam's details.
DELETE /exams/{examId}: Deletes an exam.

Question Management
POST /exams/{examId}/questions: Adds questions to an exam.
GET /exams/{examId}/questions: Retrieves all questions for an exam.
PUT /questions/{questionId}: Updates a question.
DELETE /questions/{questionId}: Deletes a question.

Results Management
POST /results: Records exam results for students.
GET /results/{examId}/{studentId}: Retrieves results for a student for a specific exam.




I have not merged the following:
*) API: Payment
Allows a student to check payment status 
api.panaversity.com/student/payment 
Request:
{ 
"StudentRollNo": 12345zxy
"Password": "xxxxxx"
 }
Respons
{ 
“Course ”: “Generative AI”
“Quater”: “Q4”
 “Fee”: “5000 Rs”
 “VoucherNo”: “1133uud”
“Payment”: “Paid”
“Date”: “auto-generate”.   (Payment date —> auto-generated)
 }

*) API:  Course Detail - Chapter , Topics, Subtopics
Retrieves the student's program and course history, including completed and in-progress courses.
api.panaversity.com/student/courseHistory 
Request:
{ 
"StudentRollNo": 123abc
 }
Response:
{
    "StudentRollNo": “123abc”
    “Title” : “Cloud Native Applied Generative AI”   (course detail)
  
    "Chater”: [
        {
            "ChapterId": "C001",
            "Title": “Generative AI",
         "Topics": [
        {
            "TopicId": "T001",
            "Title": "AI Fundamentals ”,
            "Subtopics": [
                {
                    "SubtopicId": "ST001",
                    "Title": "Definition of AI",
                    "GitHubRepoLink": "https://github.com/…”.
		     “VideoLink": "https://video.com/ …” 
                }
            ]
         }
       ]
        },  
}

