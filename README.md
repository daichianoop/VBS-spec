Venue Booking System - Technical Design Document
Version: 1.0
Status: Detailed Design & Wireframes
Target Stack: Next.js (Frontend), .NET Core 8 Web API (Backend), SQL Server (Database)
Scale: ~200 Users, ~200 Venues
1. Executive Summary
The Venue Booking System is a web-based application designed to streamline the reservation of campus facilities. It eliminates manual conflicts by providing a real-time dashboard of room availability. The system supports two distinct user roles: Student Representatives (who request venues) and Administrators (who manage approvals, conflicts, and master data).
2. System Architecture
2.1 High-Level Diagram
Client (Frontend): Next.js application using Tailwind CSS for responsive design. Hosted on Vercel or similar static edge network.
API Gateway (Backend): ASP.NET Core Web API. Handles business logic, authentication, and data validation.
Data Layer: SQL Server. Relational database ensuring data integrity.
2.2 Tech Stack Justification
Next.js: Provides server-side rendering (SSR) for fast dashboard loads and SEO.
.NET Core: Enterprise-grade security, strong typing (C#), and native integration with SQL Server.
SQL Server: Robust relational management required for complex booking schedules and avoiding double-booking anomalies.
3. Database Schema Design (SQL)
The following SQL scripts define the exact structure required to handle the complexity of the system.
3.1 Tables (SQL Definition)
A. Users
Stores login credentials and role information.
CREATE TABLE Users (
    UserID INT PRIMARY KEY IDENTITY(1,1),
    Username NVARCHAR(50) NOT NULL UNIQUE, -- e.g., "Rep_MusicSoc"
    PasswordHash NVARCHAR(255) NOT NULL,   -- BCrypt/Argon2 Hash
    Role NVARCHAR(20) NOT NULL CHECK (Role IN ('Admin', 'StudentRep')),
    FullName NVARCHAR(100) NOT NULL,       -- Admin Name or Rep Name
    ContactNo NVARCHAR(15) NOT NULL,
    SocietyName NVARCHAR(100) NULL,        -- NULL for Admins
    IsActive BIT DEFAULT 1,                -- Soft Delete
    CreatedAt DATETIME2 DEFAULT GETUTCDATE()
);


B. Venues
Stores the physical rooms available for booking.
CREATE TABLE Venues (
    VenueID INT PRIMARY KEY IDENTITY(1,1),
    VenueCode NVARCHAR(20) NOT NULL UNIQUE, -- e.g., "B001", "LT1"
    VenueType NVARCHAR(50) NOT NULL CHECK (VenueType IN ('Classroom', 'LectureTheatre', 'OAT', 'Auditorium')),
    Block NVARCHAR(10) NOT NULL,            -- 'A', 'B', 'LT', 'OAT', 'Audi'
    Floor INT NULL,                         -- 0, 1, 2, 3
    Capacity INT DEFAULT 60,
    IsActive BIT DEFAULT 1
);


C. Bookings
The core transactional table.
CREATE TABLE Bookings (
    BookingID INT PRIMARY KEY IDENTITY(1,1),
    VenueID INT NOT NULL FOREIGN KEY REFERENCES Venues(VenueID),
    RequestedByUserID INT NOT NULL FOREIGN KEY REFERENCES Users(UserID),
    
    -- Booking Details
    SocietyName NVARCHAR(100) NOT NULL,     -- Snapshot of society name at time of booking
    Purpose NVARCHAR(500) NOT NULL,
    FICName NVARCHAR(100) NOT NULL,         -- Faculty In Charge
    FICContact NVARCHAR(15) NOT NULL,
    
    -- Timing (UTC)
    StartTime DATETIME2 NOT NULL,
    EndTime DATETIME2 NOT NULL,
    
    -- Status
    BookingStatus NVARCHAR(20) DEFAULT 'Pending' CHECK (BookingStatus IN ('Pending', 'Approved', 'Rejected', 'Cancelled')),
    
    -- Approval/Admin Tracking
    ApprovedByUserID INT NULL FOREIGN KEY REFERENCES Users(UserID),
    ApprovedAt DATETIME2 NULL,
    AdminNotes NVARCHAR(MAX) NULL,          -- Reason for rejection or edit details
    
    -- Audit Stamps
    CreatedAt DATETIME2 DEFAULT GETUTCDATE(),
    LastEditedByUserID INT NULL FOREIGN KEY REFERENCES Users(UserID),
    LastEditedAt DATETIME2 NULL
);


D. AuditLogs (For Legal/Robustness)
Tracks every critical action to prevent "He said, She said" disputes.
CREATE TABLE AuditLogs (
    LogID INT PRIMARY KEY IDENTITY(1,1),
    Action NVARCHAR(50) NOT NULL,        -- 'CreateRequest', 'Approve', 'SuperEdit', 'Reject'
    PerformedByUserID INT NOT NULL,
    TargetBookingID INT NULL,
    OldValues NVARCHAR(MAX) NULL,        -- JSON string of previous state
    NewValues NVARCHAR(MAX) NULL,        -- JSON string of new state
    Timestamp DATETIME2 DEFAULT GETUTCDATE()
);


3.2 Master Venue List (Seeding Data)
You must seed these ~200 venues into the database initially. Do not do this manually; use a script.
1. Classrooms (Block A & B)
Naming Convention: [Block][Floor][RoomNumber] (2-digit room number).
Block A: A001-A022 (GF), A101-A122 (1F), A201-A222 (2F), A301-A322 (3F). Total: 88.
Block B: B001-B022 (GF), B101-B122 (1F), B201-B222 (2F), B301-B322 (3F). Total: 88.
2. Lecture Theatres (LT)
Venues: LT1 - LT12. Total: 12.
3. Open Air Theatres (OAT)
Venues: OAT1 - OAT5. Total: 5.
4. Auditoriums (Audi)
Venues: Audi1 - Audi6. Total: 6.
Grand Total: 199 Venues.
4. Business Logic & Policies
4.1 The Overlap Policy (Critical)
The system must strictly prevent double bookings unless a booking is Rejected/Cancelled.
Algorithm:
A new booking (NewStart, NewEnd) overlaps with an existing booking (ExistingStart, ExistingEnd) if:
(NewStart < ExistingEnd) AND (NewEnd > ExistingStart)
AND ExistingBooking.Status != "Rejected"
AND ExistingBooking.Status != "Cancelled"


This check runs before any Insert or Update operation.
4.2 Admin "Super-Edit" Policy
Admins have the authority to edit a request at the moment of approval.
Trigger: Admin clicks "Review" on a Pending Request.
Edit: Admin notices a conflict (e.g., Duration too long) and changes EndTime or VenueID.
Save: System runs the Overlap Check on the new details.
If Valid: Status becomes 'Approved', ApprovedByUserID is set.
If Invalid: Error message shown to Admin.
4.3 User Management Policy
Access Control: Only Admins can create Student Representative accounts.
Data Retention: If a Representative leaves, IsActive = False. Never delete user rows.
5. API Specification (RESTful)
Authentication
POST /api/auth/login -> Returns JWT Token (contains Role claim).
Dashboard Data
GET /api/venues/status?date={YYYY-MM-DD}&type={VenueType}&block={BlockID}
Booking Operations
POST /api/bookings (Student/Admin) -> Create new request.
GET /api/bookings/my-history (Student) -> View own requests.
GET /api/bookings/pending (Admin) -> View queue.
PUT /api/bookings/{id}/approve (Admin) -> Updates status, assigns ApproverID.
PUT /api/bookings/{id}/reject (Admin).
Admin Operations
POST /api/admin/users -> Create new Rep.
PUT /api/admin/users/{id} -> Deactivate Rep.
6. Frontend Layout Pages (Overview)
Login Page (/): Dual login interface.
Student Dashboard (/dashboard): Venue grid, filters, status indicators.
Booking Form: Modal/Overlay for inputting details.
Admin Dashboard (/admin/dashboard): Approval queue and conflict management.
Admin User Management (/admin/users): Add/Remove Representatives.
7. User Interaction Flow & Wireframes (Detailed)
This section details the application "Frames" (Screens) and the navigation flow between them.
Flow A: The Student Representative Journey
Frame 1: The Login Screen
Layout: Centered Card on a blurred campus background.
Elements: Logo, "Username" Input, "Password" Input, "Login" Button.
Logic: No role selection toggle needed; the backend determines role from the credentials.
Transition: Success -> Redirects to Frame 2. Failure -> Shake animation + Error toast.
Frame 2: The Venue Dashboard (Main)
Layout:
Top Bar: Logo (Left), "Welcome, Music Soc" (Right), "My Bookings" Button, Logout Icon.
Filter Ribbon (Sticky):
Date Picker: Default = Today.
Venue Type Dropdown: (Classroom, LT, OAT, Audi).
Block Dropdown: (Visible only if Type=Classroom).
Main Content Area: A Responsive Grid (CSS Grid).
Visual States (The Cards):
Green Card: Text "B101". Subtext "Available". Action: Clickable.
Red Card: Text "B102". Subtext "Booked: Dance Soc (2pm-4pm)". Action: Disabled/Tooltip only.
Yellow Card: Text "B103". Subtext "Pending Approval". Action: Click to view status.
Transition: Click Green Card -> Opens Frame 3.
Frame 3: The Booking Modal (Overlay)
Layout: Modal dialog centered on screen. Background dimmed.
Header: "Requesting Booking for B101" (Date: 12th Oct).
Form Body:
Rep/Society: Read-only (Auto-filled from JWT).
Time Selection: "From" (Dropdown) and "To" (Dropdown).
Purpose: Textarea (Required).
FIC Details: Input Name, Input Phone.
Footer: "Cancel" (Ghost button), "Submit Request" (Primary button).
Transition: Click Submit -> Loading Spinner -> Success Toast -> Modal Closes -> Frame 2 refreshes (Card turns Yellow).
Flow B: The Admin Approval Journey
Frame 4: Admin Dashboard
Layout: Sidebar Navigation (Left), Main Content (Right).
Sidebar Items: "Dashboard", "Pending Requests (Badge: 5)", "All Bookings", "User Management".
Main Content (Pending View):
Data Table: Rows representing requests.
Columns: Date, Time, Venue, Society, FIC, Status.
Action Column: "Review" Button (Blue).
Transition: Click Review -> Opens Frame 5.
Frame 5: The "Super-Edit" Approval Modal
Layout: Split View Modal.
Left Side (Original Request): Read-only details of what the student asked for.
Right Side (Approval Actions): Editable Form.
Scenario: Student asked for LT1. Admin sees LT1 has a maintenance issue.
Action: Admin changes "Venue" dropdown from LT1 to LT2.
Action: Admin changes "Time" to fix a 15-min overlap.
Buttons: "Reject" (Red), "Approve with Changes" (Green).
Transition: Click Approve -> Backend validates new details -> Success Toast -> Row removed from Frame 4.
8. Detailed Component Hierarchy (Frontend Architecture)
To ensure the codebase is clean and maintainable, use this component structure in Next.js.
src/
├── app/
│   ├── (auth)/page.tsx           <-- Frame 1 (Login)
│   ├── (student)/dashboard/      <-- Frame 2
│   │   └── page.tsx
│   ├── (admin)/admin/
│   │   ├── dashboard/page.tsx    <-- Frame 4
│   │   └── users/page.tsx
├── components/
│   ├── ui/                       <-- Reusable UI (Buttons, Inputs, Modals)
│   ├── features/
│   │   ├── dashboard/
│   │   │   ├── FilterBar.tsx     <-- Date/Venue Type selectors
│   │   │   ├── VenueGrid.tsx     <-- The grid layout container
│   │   │   └── VenueCard.tsx     <-- Individual Green/Red card logic
│   │   ├── booking/
│   │   │   └── BookingModal.tsx  <-- Frame 3 Form Logic
│   │   ├── admin/
│   │   │   ├── RequestsTable.tsx <-- Frame 4 Data Table
│   │   │   ├── ApprovalModal.tsx <-- Frame 5 Super-Edit Logic
│   │   │   └── UserForm.tsx      <-- Add/Edit Student Reps
├── lib/
│   ├── api.ts                    <-- Axios instance with Interceptors
│   ├── utils.ts                  <-- Date formatting helpers
│   └── hooks/
│       ├── useAuth.ts            <-- Manages JWT and Roles
│       └── useVenues.ts          <-- React Query hooks for fetching status




9. Implementation & Deployment Steps
Step 1: Database Setup (Local)
Install SQL Server Developer Edition.
Run the SQL scripts provided in Section 3.1 to create tables.
Crucial: Write a C# Console App to loop through the hierarchy in Section 3.2 and INSERT the 199 venues. Do not skip this.
Step 2: Backend Development (.NET 8)
Create Project: dotnet new webapi -n VenueBookingAPI.
Packages: Install Microsoft.EntityFrameworkCore.SqlServer, Microsoft.AspNetCore.Authentication.JwtBearer.
Services: Implement BookingService.cs containing the IsOverlapping(...) method defined in Section 4.1.
Step 3: Frontend Development (Next.js)
Create Project: npx create-next-app@latest venue-booking-client.
Dependencies: npm install axios date-fns lucide-react @tanstack/react-query.
Environment: Create .env.local pointing to localhost API.
Step 4: Deployment Strategy
A. Database (Azure SQL)
Create an Azure SQL Database (Serverless tier is cheapest for <200 users).
Get the Connection String from Azure Portal.
Run your migration/SQL scripts on the Azure DB instance.
B. Backend (Azure App Service)
Create an App Service (Linux or Windows).
In "Configuration", add:
ConnectionStrings:DefaultConnection =
JwtSettings:Key =
Publish API via Visual Studio Right-Click -> Publish.
C. Frontend (Vercel)
Push Next.js code to GitHub.
Import project in Vercel.
Add Environment Variable in Vercel:
NEXT_PUBLIC_API_URL =
Deploy.
10. Edge Cases & Robustness Considerations
The "Race Condition":
Scenario: User A and User B click "Book" for B101 at 2:00 PM at the exact same second.
Solution: The Database Overlap Check (Section 4.1) inside the transaction will catch the second request and fail it. The API must return a 409 Conflict error, and the Frontend must show "Room just taken, please refresh".
Admin Override:
Scenario: An urgent university event needs the Auditorium, but the Drama Club has already booked it.
Solution: Admin rejects the Drama Club booking (triggers email/notification) and creates a new booking for the University Event. The system allows this because the first booking status becomes 'Rejected', clearing the slot.
Multi-Day Bookings:
Solution: The "Booking Form" should have a "Recurring" checkbox. If checked, the Backend should generate 7 separate booking entries (one for each day). This allows specific days to be cancelled later if needed without deleting the whole block.
11. Mandatory Robustness Conditions & Enforcement
This section defines the non-negotiable technical requirements to ensure the system remains "Legal, Easy, and Robust" under all conditions.
11.1 Data Integrity Musts
MUST: Prevent Orphaned Records
Condition: A Booking cannot exist without a valid UserID and VenueID.
How to Ensure: Enforce strict Foreign Keys in SQL (as defined in Schema 3.1). Configure Entity Framework OnDelete behavior to Restrict (prevent deleting a User if they have active bookings).
MUST: Prevent Logical Time Errors
Condition: EndTime must strictly be greater than StartTime.
How to Ensure:
Frontend: Validation in BookingModal.tsx using zod schema: refine((data) => data.end > data.start).
Backend: API Controller returns 400 Bad Request if Start >= End.
Database: Add SQL Check Constraint: ALTER TABLE Bookings ADD CONSTRAINT CHK_Time CHECK (EndTime > StartTime);
MUST: Atomic Booking Creation
Condition: The check for overlap and the insertion of the booking must happen as a single atomic unit.
How to Ensure: Wrap the logic in a Serializable Transaction.
using var transaction = _context.Database.BeginTransaction(System.Data.IsolationLevel.Serializable);
try {
   // 1. Check Overlap
   // 2. Insert
   // 3. Commit
} catch {
   // Rollback
}


11.2 Security Musts
MUST: Zero Trust for Admin Actions
Condition: An attacker cannot curl the /approve endpoint without being an actual Admin.
How to Ensure: Do not rely on UI hiding.
Decorate Controllers: [Authorize(Roles = "Admin")].
Verify the JWT signature matches the server's secret key.
MUST: Immutable Audit History
Condition: Nobody (including Admins) can delete an entry from AuditLogs via the API.
How to Ensure: Do not create a DELETE endpoint for the AuditLogs controller. The table is strictly Append-Only.
11.3 Operational Musts
MUST: Timezone Standardization
Condition: 2 PM in India must not show as 2 AM in New York.
How to Ensure:
Database: Always store dates as UTC (DateTime.UtcNow).
API: Return UTC strings (ISO 8601: 2023-10-12T14:00:00Z).
Frontend: Browser converts to local time using new Date(utcString).toLocaleTimeString().
MUST: Database Resilience
Condition: System must handle database connection blips.
How to Ensure: Enable "Retry Logic" in Entity Framework Connection String.
options.UseSqlServer(connectionString, builder => builder.EnableRetryOnFailure(5, TimeSpan.FromSeconds(10), null));
