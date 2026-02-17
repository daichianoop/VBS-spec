Venue Booking System

A robust, enterprise-grade web application designed to streamline the reservation of campus facilities. This system eliminates manual scheduling conflicts through real-time overlap detection and provides distinct workflows for Student Representatives and Administrators.

## 🚀 Tech Stack

Frontend: Next.js (React), Tailwind CSS, Lucide Icons

Backend: ASP.NET Core 8 Web API

Database: SQL Server (Entity Framework Core)

Authentication: JWT (JSON Web Tokens)

Infrastructure: Azure App Service & SQL Database (Production), Vercel (Frontend)

✨ Key Features

Real-Time Availability Dashboard: Visual grid showing Green (Available), Red (Booked), and Yellow (Pending) status for ~200 venues.

Conflict Prevention Engine: Atomic transactions ensure no two bookings can overlap, handling race conditions gracefully.

Role-Based Access Control:

Student Reps: View availability, request bookings, view history.

Admins: Approve/Reject requests, "Super-Edit" conflicting requests, manage users, view audit logs.

"Super-Edit" Workflow: Admins can modify time/venue details during the approval process to resolve conflicts without requiring a new request.

Audit Logging: Immutable history of all critical actions (Approvals, Edits, Rejections) for accountability.

🛠️ Getting Started

Prerequisites

Node.js 18+

.NET 8 SDK

SQL Server (Developer Edition or Express)

1. Database Setup

Create a local SQL Server database named VenueBookingDB.

Execute the table creation scripts found in the database/scripts folder (Users, Venues, Bookings, AuditLogs).

Important: Run the VenueSeeder console app (or SQL script) to populate the master list of 199 venues (Blocks A/B, LTs, OATs, Audis).

2. Backend Setup (.NET)

Navigate to the API directory:

cd src/VenueBookingAPI


Update appsettings.json with your SQL Connection String and a secure JWT Secret Key.

Run the application:

dotnet restore
dotnet run


API will run on https://localhost:5001 (or similar).

3. Frontend Setup (Next.js)

Navigate to the client directory:

cd src/venue-booking-client


Install dependencies:

npm install


Create a .env.local file in the root:

NEXT_PUBLIC_API_URL=https://localhost:5001/api


Start the development server:

npm run dev


App accessible at http://localhost:3000.

📂 Project Structure

src/
├── VenueBookingAPI/           # .NET Core Web API
│   ├── Controllers/           # API Endpoints (Auth, Bookings, Admin)
│   ├── Models/                # Entity Framework Models
│   ├── Services/              # Business Logic (Overlap Check)
│   └── Data/                  # DbContext & Migrations
│
├── venue-booking-client/      # Next.js Frontend
│   ├── app/                   # App Router Pages (Login, Dashboard, Admin)
│   ├── components/            # Reusable UI (VenueCard, BookingModal)
│   └── lib/                   # API Utilities & Hooks


🛡️ Robustness & Security Policies

Atomic Transactions: Overlap checks and insertions occur within a Serializable transaction scope.

Timezone Handling: All dates stored as UTC in SQL; converted to local time on the client.

Data Integrity: Strict Foreign Key constraints prevent orphaned booking records.

Zero Trust: All Admin API endpoints are protected by server-side Role validation, not just UI hiding.

📜 License

Private / Internal Campus Use Only.
