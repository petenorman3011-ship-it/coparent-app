# CoParent — Full Product Specification

_Version 1.0 | April 2026_

---

## 1. Product Vision

CoParent is a secure, mobile-first co-parenting platform that helps two parents communicate respectfully, manage custody schedules, and stay informed about their child's daily care. The platform is child-centred, low-conflict by design, and built for everyday use on a phone.

---

## 2. Tech Stack Recommendation

### Frontend
- **Framework:** React Native (Expo) — one codebase for iOS and Android
- **Web version:** Next.js (React) — same component logic, responsive mobile-first
- **Styling:** Tailwind CSS / NativeWind
- **State management:** Zustand
- **Push notifications:** Expo Notifications / Firebase Cloud Messaging

### Backend
- **Runtime:** Node.js + Express (or Next.js API routes)
- **Database:** PostgreSQL (via Supabase — gives auth, real-time, storage out of the box)
- **File storage:** Supabase Storage (photos, exports)
- **Auth:** Supabase Auth (email/password + magic link)
- **AI moderation:** OpenAI GPT-4o-mini (cheap, fast, accurate for text moderation)
- **Real-time messaging:** Supabase Realtime (WebSocket subscriptions)
- **Scheduled jobs:** pg_cron (handover reminders)

### Hosting
- **Frontend:** Vercel
- **Backend/DB:** Supabase (managed PostgreSQL)
- **Storage:** Supabase Storage
- **Cost at MVP scale:** ~£0–20/month

---

## 3. Database Schema

### users
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT UNIQUE NOT NULL,
  full_name TEXT NOT NULL,
  role TEXT CHECK (role IN ('parent_a', 'parent_b')) NOT NULL,
  avatar_url TEXT,
  family_id UUID REFERENCES families(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  last_seen_at TIMESTAMPTZ
);
```

### families
```sql
CREATE TABLE families (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  invite_code TEXT UNIQUE NOT NULL, -- parent B joins via code
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### children
```sql
CREATE TABLE children (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id UUID REFERENCES families(id) NOT NULL,
  name TEXT NOT NULL,
  date_of_birth DATE,
  photo_url TEXT,
  allergies TEXT,
  medical_notes TEXT,
  doctor_name TEXT,
  doctor_phone TEXT,
  emergency_contact_name TEXT,
  emergency_contact_phone TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### messages
```sql
CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id UUID REFERENCES families(id) NOT NULL,
  sender_id UUID REFERENCES users(id) NOT NULL,
  content TEXT NOT NULL,
  is_flagged BOOLEAN DEFAULT FALSE,
  sent_at TIMESTAMPTZ DEFAULT NOW(),
  read_at TIMESTAMPTZ,
  moderation_result_id UUID REFERENCES moderation_results(id)
);
```

### moderation_results
```sql
CREATE TABLE moderation_results (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  original_text TEXT NOT NULL,
  flagged_words TEXT[],
  suggested_rewrite TEXT,
  severity TEXT CHECK (severity IN ('low','medium','high')),
  action_taken TEXT CHECK (action_taken IN ('warned','blocked','sent_anyway','edited')),
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### daily_logs
```sql
CREATE TABLE daily_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id UUID REFERENCES families(id) NOT NULL,
  child_id UUID REFERENCES children(id) NOT NULL,
  logged_by UUID REFERENCES users(id) NOT NULL,
  log_date DATE NOT NULL DEFAULT CURRENT_DATE,
  meals JSONB, -- { breakfast: bool, lunch: bool, dinner: bool, snacks: bool, appetite: 'good'|'poor' }
  drinks JSONB, -- { water: bool, milk: bool, juice: bool, amount: 'good'|'little' }
  nap_start TIME,
  nap_end TIME,
  mood TEXT CHECK (mood IN ('happy','good','okay','upset','tired','unwell')),
  mood_note TEXT,
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### log_photos
```sql
CREATE TABLE log_photos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  log_id UUID REFERENCES daily_logs(id) ON DELETE CASCADE,
  uploaded_by UUID REFERENCES users(id),
  storage_path TEXT NOT NULL,
  url TEXT NOT NULL,
  caption TEXT,
  uploaded_at TIMESTAMPTZ DEFAULT NOW()
);
```

### medications
```sql
CREATE TABLE medications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  child_id UUID REFERENCES children(id) NOT NULL,
  name TEXT NOT NULL,
  dosage TEXT,
  frequency TEXT,
  notes TEXT,
  added_by UUID REFERENCES users(id),
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### medication_logs
```sql
CREATE TABLE medication_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  medication_id UUID REFERENCES medications(id),
  child_id UUID REFERENCES children(id),
  logged_by UUID REFERENCES users(id),
  given_at TIMESTAMPTZ NOT NULL,
  dose_given TEXT,
  notes TEXT
);
```

### schedules
```sql
CREATE TABLE schedules (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id UUID REFERENCES families(id) NOT NULL,
  schedule_type TEXT CHECK (schedule_type IN ('week12','custom')) DEFAULT 'week12',
  week1_pattern JSONB, -- { mon:'A', tue:'A', wed:'A', thu:'B', fri:'B', sat:'B', sun:'B' }
  week2_pattern JSONB,
  reference_week_start DATE NOT NULL, -- the Monday of a known Week 1
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### change_requests
```sql
CREATE TABLE change_requests (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id UUID REFERENCES families(id) NOT NULL,
  requested_by UUID REFERENCES users(id) NOT NULL,
  dates DATE[] NOT NULL,
  reason TEXT,
  status TEXT CHECK (status IN ('pending','accepted','rejected')) DEFAULT 'pending',
  responded_by UUID REFERENCES users(id),
  responded_at TIMESTAMPTZ,
  response_note TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### notifications
```sql
CREATE TABLE notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) NOT NULL,
  family_id UUID REFERENCES families(id),
  type TEXT NOT NULL, -- 'new_message','new_log','change_request','request_response','handover_reminder','photo_added'
  title TEXT NOT NULL,
  body TEXT,
  data JSONB, -- reference IDs etc
  read_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### handover_notes
```sql
CREATE TABLE handover_notes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id UUID REFERENCES families(id) NOT NULL,
  child_id UUID REFERENCES children(id) NOT NULL,
  written_by UUID REFERENCES users(id) NOT NULL,
  handover_date DATE NOT NULL,
  direction TEXT CHECK (direction IN ('AtoB','BtoA')),
  note TEXT NOT NULL,
  pickup_time TIME,
  dropoff_location TEXT,
  items_packed TEXT[], -- ['school bag', 'medication', 'comfort toy']
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 4. API Endpoints

### Auth
```
POST /api/auth/register
POST /api/auth/login
POST /api/auth/join-family   -- parent B joins with invite code
GET  /api/auth/me
```

### Family & Child
```
POST /api/family/create
GET  /api/family/:id
PUT  /api/child/:id
GET  /api/child/:familyId
```

### Messages
```
GET  /api/messages/:familyId?limit=50&before=<timestamp>
POST /api/messages/:familyId
POST /api/messages/moderate   -- AI moderation check
PUT  /api/messages/:id/read
```

### Daily Logs
```
GET  /api/logs/:familyId?date=2026-04-18
POST /api/logs
POST /api/logs/:logId/photos
GET  /api/logs/:id
DELETE /api/logs/:id
```

### Calendar / Schedule
```
GET  /api/schedule/:familyId
PUT  /api/schedule/:familyId
GET  /api/schedule/:familyId/month?year=2026&month=4
```

### Change Requests
```
GET  /api/requests/:familyId
POST /api/requests
PUT  /api/requests/:id/respond   -- { action: 'accepted'|'rejected', note: '' }
```

### Notifications
```
GET  /api/notifications/:userId
PUT  /api/notifications/:id/read
PUT  /api/notifications/read-all
```

### Medications
```
GET  /api/medications/:childId
POST /api/medications
POST /api/medications/:id/log
GET  /api/medications/:childId/logs
```

### Reports/Export
```
GET  /api/export/:familyId/messages?from=&to=   -- returns PDF or CSV
GET  /api/export/:familyId/logs?from=&to=
```

---

## 5. AI Moderation Logic

```javascript
// Moderation flow
async function moderateMessage(text) {
  // Step 1: Local word filter (fast, no API cost)
  const localFlags = checkLocalWordlist(text);
  
  // Step 2: If passes local filter, check with AI
  const aiResult = await openai.chat.completions.create({
    model: 'gpt-4o-mini',
    messages: [{
      role: 'system',
      content: `You are a co-parenting communication moderator. 
      Analyse the message for: swearing, abuse, threats, manipulation, 
      blame language, generalising statements (always/never), 
      or anything that could escalate conflict.
      
      Respond with JSON:
      {
        "flagged": boolean,
        "severity": "none"|"low"|"medium"|"high",
        "reasons": string[],
        "suggestion": string | null  // neutral rewrite if flagged
      }`
    }, {
      role: 'user', 
      content: `Message to moderate: "${text}"`
    }],
    response_format: { type: 'json_object' }
  });
  
  return JSON.parse(aiResult.choices[0].message.content);
}
```

**Moderation thresholds:**
- `none` → send immediately
- `low` → show soft nudge ("Consider softening this")
- `medium` → show warning + suggestion, require confirmation to send
- `high` → block send, require full rewrite

---

## 6. User Flows

### Setup Flow
1. Parent A creates account → creates family → gets invite code
2. Parent A sets up child profile, schedule pattern, names
3. Parent A shares invite code with Parent B
4. Parent B creates account → enters invite code → joins family
5. Both parents now see shared dashboard

### Message Flow
1. Parent types message → hits Send
2. Local word filter runs instantly
3. If clean → AI moderation check (async, ~500ms)
4. If flagged → show warning modal with suggestion
5. Parent can: edit message / send anyway / cancel
6. If sent → stored in DB → real-time push to other parent
7. Read receipt recorded when other parent opens thread

### Daily Log Flow
1. Parent taps "Add Log" on dashboard or log screen
2. Fills in meals, drinks, nap, mood, notes
3. Optionally adds photos (compressed before upload)
4. Saves → stored in DB → notification sent to other parent
5. Both parents can view full timeline on log screen

### Schedule Change Flow
1. Parent A taps a calendar date → "Request Change"
2. Fills in reason, submits
3. Parent B receives push notification
4. Parent B opens request → sees dates + reason
5. Parent B accepts or rejects (optionally with note)
6. Parent A notified of outcome
7. If accepted → calendar updates automatically
8. Full history preserved in change_requests table

---

## 7. Validation Rules

### Messages
- Max length: 2000 characters
- Cannot be empty or whitespace only
- Must pass moderation before delivery (unless override)
- Moderation result always stored regardless of outcome

### Daily Logs
- log_date cannot be in the future
- nap_end must be after nap_start
- At least one field required (can't submit empty log)
- Photos: max 5 per log entry, max 10MB each, JPEG/PNG/HEIC only
- Max 10 log entries per child per day

### Change Requests
- Request dates must be in the future
- Cannot request dates that already have a pending request
- Max 30 days in advance
- Reason: max 500 characters

### Users
- Email must be valid format
- Password: min 8 chars, 1 uppercase, 1 number
- Full name: 2–50 characters
- Both parents must belong to same family_id

---

## 8. Screen Descriptions

### Login / Signup
- Clean minimal screen, app logo, parent role selection
- Email + password, magic link option
- "Join family" flow for Parent B (enter invite code)
- Forgot password via email

### Dashboard / Home
- Today's custody display (large, prominent)
- Next handover countdown
- Quick action buttons (Message, Add Log, Calendar, Medications)
- Recent activity feed (last 3 logs + messages)
- Pending change requests alert if any

### Messages
- Clean chat bubbles, colour-coded by parent
- Timestamp on each message
- Read receipts (✓✓ when read)
- Moderation warning inline when flagged
- Compose bar with send button

### Daily Log Feed
- Reverse chronological timeline
- Cards per entry: parent avatar, time, tags (meals/nap/mood), notes, photos
- Filter by date or parent
- Tap to expand full entry

### Add Daily Update
- Step-by-step or single scrollable form
- Meal chips (tap to select)
- Drink chips
- Nap time pickers
- Mood selector (emoji grid)
- Notes textarea
- Photo upload (camera or gallery)
- Save + notify partner

### Calendar / Custody Schedule
- Monthly view with parent-colour-coded days
- Weekly view toggle
- Legend at bottom
- Tap day = detail panel slides up
- Pending requests shown as dots on calendar
- "Request change" button

### Change Request Form
- Date picker
- Reason text field
- Submit → confirmation screen

### Change Request Inbox
- List of all requests (pending/accepted/rejected)
- Colour-coded status
- Accept/Reject buttons for incoming pending requests
- Full history visible

### Notifications
- Grouped by type
- Mark all read
- Tap to navigate to relevant screen
- Unread dot indicators

### Settings / Profile
- Edit name, photo, password
- Child profile: name, DOB, photo, allergies, doctor, emergency contacts
- Notification preferences (toggle each type)
- Schedule settings (edit Week 1/Week 2 pattern)
- Export data (messages + logs as PDF)
- Account: sign out, delete account

---

## 9. MVP Scope

**Phase 1 — Core product (MVP)**
- [ ] Auth (register, login, invite code family join)
- [ ] Child profile
- [ ] Messages with AI moderation
- [ ] Daily log (meals, drinks, nap, mood, notes)
- [ ] Calendar (Week 1/Week 2 schedule)
- [ ] Change request system
- [ ] Push notifications
- [ ] Basic settings

**Phase 2 — Enhanced product**
- [ ] Photo uploads on logs
- [ ] Medication tracker
- [ ] Handover notes + pickup/dropoff details
- [ ] Read receipts
- [ ] Message export (PDF)
- [ ] Log export (PDF/CSV)
- [ ] Shared child profile (allergies, doctor, emergency contacts)
- [ ] Multiple children support

**Phase 3 — Growth**
- [ ] In-app subscription (premium features)
- [ ] Court-admissible export format
- [ ] Therapist/mediator view-only access
- [ ] iOS/Android native app via React Native

---

## 10. Security & Privacy

- All data encrypted at rest (Supabase default)
- TLS for all API calls
- Row-level security (RLS) in Supabase — parents can only access their own family's data
- No third-party data sharing
- Photos stored in private bucket, served via signed URLs
- JWT tokens with 7-day expiry
- Moderation logs retained for safeguarding purposes
- Account deletion removes all PII (GDPR compliant)
- Audit log of all change requests (tamper-evident record)

---

## 11. Estimated Build Time (Solo Developer)

| Phase | Time |
|-------|------|
| DB + API setup | 3 days |
| Auth + family setup | 2 days |
| Messages + moderation | 3 days |
| Daily log | 2 days |
| Calendar + schedule | 3 days |
| Change requests | 2 days |
| Notifications | 2 days |
| Settings + profile | 1 day |
| Testing + polish | 3 days |
| **MVP Total** | **~3 weeks** |

With AI assistance (like this): cut that by 60–70%.

---

_Generated by Pete 🎯 | CoParent Product Spec v1.0_
