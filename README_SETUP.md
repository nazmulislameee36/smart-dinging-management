# SMART DINING — FINAL SINGLE-HALL SETUP

## GitHub Pages files
Upload these to the repository ROOT:
- index.html
- admin.html
- supabase-config.js

## Supabase SQL
Run `supabase_schema_final.sql` once in Supabase SQL Editor.

## Frontend key
Edit `supabase-config.js` and paste only your `sb_publishable_...` key.
Never commit an `sb_secret_...` or service-role key.

## First hall
Admin -> System Settings -> choose the one hall name and meal prices/time slots.
Or run:
update public.system_settings set hall_name='STA Hall' where id=1;

## First admin
1. Supabase -> Authentication -> Users -> create the admin account.
2. Run:
insert into public.admins(user_id,email)
select id,email from auth.users
where email='YOUR_ADMIN_EMAIL'
on conflict(user_id) do nothing;
3. Open `/admin.html`.

## Student flow
Student -> Register -> pending -> login by Student ID + password -> Pending dashboard.
Admin -> Pending Approvals -> enter RFID -> Save RFID & Approve.
Student then gets the full dashboard.

## Password reset
Student enters Student ID + Gmail on `Forgot password?`.
The student-reset Edge Function sends the Supabase Auth recovery email.
After clicking the link, the same `index.html` opens a new password dialog.

Set Supabase Authentication -> URL Configuration -> Site URL / Redirect URLs to your GitHub Pages URL.
For first testing, email confirmation can be disabled under Authentication -> Providers -> Email. Turn it back on later if desired.

## Edge Functions
Create/deploy:
- student-login
- student-reset
- admin-delete-student
- gate-scan

The code is under `supabase/functions/...`.
Supabase Dashboard supports creating/deploying Edge Functions directly. For `student-login`, `student-reset`, `gate-scan`, built-in JWT verification must be disabled because they authenticate another way. `admin-delete-student` also verifies the user token manually inside the function.

Function secrets:
- SUPABASE_PUBLISHABLE_KEY = your sb_publishable_... key
SUPABASE_SECRET_KEY = a server-only sb_secret_... key (never put this in GitHub/ESP32)
- SITE_URL = your GitHub Pages URL
- ESP32_DEVICE_KEY = a long random secret for the gate device

The Edge Functions read `SUPABASE_URL` plus the server-only `SUPABASE_SECRET_KEY` from function secrets. Never copy the secret key into GitHub or ESP32.

## ESP32
Install ArduinoJson.
Edit `ESP32_Gate_Test.ino`:
- Wi-Fi
- ESP32_DEVICE_KEY
Upload to ESP32.

Serial commands:
E,A3 7F 21 9C
X,A3 7F 21 9C

Entry marks `entry_scanned_at`. Exit marks `consumed_at` and creates a `gate_events` row.
The server determines the current breakfast/lunch/dinner slot from `system_settings`, so the ESP32 does not need hard-coded meal times.

## Database model
- `system_settings`: one hall + prices + three scan slots
- `students`: profile, approval and RFID
- `student_meals`: one row per student/date/meal; this is the main ESP32-facing meal table
- `payments`: demo payment ledger
- `gate_events`: entry/exit audit log
- `admins`: admin identities
- `notices`: student notice board

## Important security notes
- Passwords live in Supabase Auth, not `students`.
- RLS protects student/admin data.
- ESP32 uses a separate device key for the gate function.
- `WiFiClientSecure.setInsecure()` is used only in the prototype ESP32 sketch. Before real deployment, use proper TLS certificate verification and rotate device credentials.
- bKash/Nagad in this build are demo payment records only; there is no real payment gateway charge.
