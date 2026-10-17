# Smart Dining Management System — Updated Monthly Planner + RFID Gate

## Project layout
- `index.html` — Student registration/login, monthly meal calendar, booking ON/OFF, payment and refund requests.
- `admin.html` — Admin dashboard, approvals/RFID assignment, students, meals, payment records, refund approvals, reports, notices and settings.
- `supabase-config.js` — browser-safe Supabase URL + publishable key only.
- `supabase_schema_final.sql` — full reset schema plus the updated planner/refund routines. **Destructive to existing data.**
- `upgrade_meal_planning_refunds.sql` — non-destructive migration for an existing Smart Dining database. Use this for your current running project.
- `supabase/functions/student-login/index.ts`
- `supabase/functions/student-reset/index.ts`
- `supabase/functions/admin-delete-student/index.ts`
- `supabase/functions/gate-scan/index.ts`
- `esp32/ESP32_Gate_Test.ino` — ESP32 + MFRC522 RFID prototype.

## Important: choose the correct SQL
**Existing Smart Dining database with student records:** back up first, then run `upgrade_meal_planning_refunds.sql` only. Do NOT run `supabase_schema_final.sql`, because it drops/recreates Smart Dining tables.

**Brand-new/test database:** run `supabase_schema_final.sql` once. It contains the base schema and the upgrade routines at the end.

## Updated meal rules
1. Students select a month and see every date with Breakfast, Lunch and Dinner actions.
2. Same-day meal booking is not allowed.
3. Any future date may be booked several days ahead, but only until **10:00 PM Bangladesh time on the previous day**. After that cutoff, ON/OFF changes are locked. The database trigger enforces this, not just the browser.
4. A paid meal cannot be switched OFF. Consumed and refunded meals cannot be re-enabled. Their action/status shows `PAID`, `CONSUMED`, `REFUND REQUESTED` or `REFUNDED` where applicable.
5. Refund requests can be submitted only after the meal date has passed, for a paid booked meal with no entry scan and no consumption. Admin reviews requests under **Refund Requests**.
6. Approval records `refunded_at` in the app. This build uses **demo payment records only** and does not transfer real bKash/Nagad money. A real refund must be processed through a real provider/manual process and reconciled separately.

## 1. Supabase setup
1. In `supabase-config.js`, set the correct project URL and `sb_publishable_...` key. Keep secret/service-role keys out of frontend and ESP32 files.
2. On an existing database, run `upgrade_meal_planning_refunds.sql` in SQL Editor. Review and back up first.
3. Create an administrator in Supabase Authentication → Users, then allowlist it:
   ```sql
   insert into public.admins(user_id,email)
   select id,email from auth.users where lower(email)=lower('YOUR_ADMIN_EMAIL')
   on conflict(user_id) do nothing;
   ```
4. Configure Authentication → URL Configuration with the deployed student portal URL.

## 2. Deploy Edge Functions
Deploy functions with these exact names: `student-login`, `student-reset`, `admin-delete-student`, `gate-scan`. Set server-side secrets in Supabase: `SUPABASE_URL`, `SUPABASE_PUBLISHABLE_KEY`, `SUPABASE_SECRET_KEY`, `SITE_URL`, and `ESP32_DEVICE_KEY`. The secret key must never be pushed to GitHub or put in the ESP32 sketch.

Recommended CLI commands:
```bash
supabase functions deploy student-login --no-verify-jwt
supabase functions deploy student-reset --no-verify-jwt
supabase functions deploy gate-scan --no-verify-jwt
supabase functions deploy admin-delete-student
```
`student-login`, `student-reset`, and `gate-scan` check their own credential/device data. `admin-delete-student` validates the bearer token and admin allowlist internally too.

## 3. GitHub Pages
Place `index.html`, `admin.html`, and `supabase-config.js` beside each other in the published directory.

## 4. ESP32 wiring / libraries
- MFRC522 SDA/SS → GPIO 5; SCK → GPIO 18; MOSI → GPIO 23; MISO → GPIO 19; RST → GPIO 22.
- MFRC522 power → 3.3V and GND → GND.
- Relay IN → GPIO 26; buzzer → GPIO 25; mode button → GPIO 27 and GND. Confirm your own relay module's trigger logic and electrical isolation before connecting any door hardware.
- Install Arduino libraries `MFRC522` and `ArduinoJson`. Configure Wi-Fi, Supabase Edge Function URL, and device key in the sketch.
- Serial Monitor at 115200 baud, newline: `MODE,E`, `MODE,X`, `E,A3 7F 21 9C`, `X,A3 7F 21 9C`.

The sketch currently uses prototype TLS configuration. Configure certificate verification before deployment; rotate the device key if it is exposed. Test with an LED/bench setup before attaching a real gate.

## 5. Test checklist
1. Create a student; confirm it appears as pending.
2. Assign RFID and approve using Admin.
3. In Student Panel, choose a future date and turn meals ON before the previous day's 10:00 PM deadline.
4. Confirm same-day ON and post-deadline ON/OFF attempts are rejected by the database.
5. Record a demo payment; confirm Paid state cannot be toggled OFF.
6. Scan during configured slot: ENTRY should require approved student, active card, meal ON and paid. EXIT consumes the meal; consumed meals cannot be reactivated.
7. For a paid meal with no entry/consumption, after that meal date has passed request refund; Admin approves/rejects.

This is a prototype—not a production-certified access-control or payment system. Edge Function deployment, RLS, email delivery, relay wiring, TLS verification, and end-to-end hardware must be tested in your own Supabase project before real use.
