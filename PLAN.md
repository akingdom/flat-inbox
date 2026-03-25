# **Handover Document
## Blueprint plan for building this project
### *4‑Module System • Pluggable Storage Backend • Static‑Site Friendly*

---

## **1. Purpose & Philosophy**

This system provides a minimal, interpretable, low‑maintenance messaging channel embedded into a static site. It is intentionally small‑surface, low‑risk, and avoids unnecessary backend complexity.

Core values:

- frictionless for visitors  
- invisible for non‑admins  
- admin‑only features without a login wall  
- browser‑remembered admin capability  
- no black‑box behaviour  
- everything inspectable, durable, and simple  

It is a one‑way inbox with a lightweight admin UI.

---

## **2. Architecture Overview**

The system is implemented using four independent modules.

### **1. API Module**
- defines all HTTP endpoints  
- orchestrates calls to Auth and Storage  
- contains no business logic  
- does not know the storage format  

### **2. Auth Module**
- issues refresh tokens (long‑lived)  
- issues session tokens (short‑lived)  
- validates tokens  
- stateless except for secrets  

### **3. Storage Module (Pluggable Backend)**
- implements the storage backend selected in configuration  
- stores immutable message artifacts  
- stores mutable message state  
- maintains the message index  
- regenerates index from canonical artifacts  
- exposes a uniform interface regardless of backend  

### **4. Frontend Module**
- static HTML/JS served via GitHub Pages  
- visitor message form  
- admin UI panel  
- token handling in browser  
- polling logic  
- no secrets, no backend logic  

---

## **3. Authentication Model**

### **3.1 Two‑Token System**

**Refresh Token (long‑lived)**  
- stored in `localStorage`  
- identifies the browser as admin‑capable  
- cannot perform admin actions  
- used only to request a session token  

**Session Token (short‑lived)**  
- stored in `sessionStorage`  
- grants temporary admin access  
- expires on browser close or timeout  

### **3.2 Auto‑Login Flow**

1. no refresh token → visitor UI  
2. refresh token exists → call `/auth/refresh`  
3. if valid → session token issued → admin UI  
4. if invalid → refresh token cleared → visitor UI  

---

## **4. Message Flow (Visitor)**

### **4.1 UX**
- simple form: name (optional), email (optional), message  
- no CAPTCHA unless abuse detected  
- instant “Message sent” feedback  

### **4.2 Backend Flow**

1. visitor submits message → Frontend  
2. API receives request → API Module  
3. API validates → Storage Module  
4. Storage Module:  
   - creates immutable message artifact  
   - creates or updates message state  
   - updates index  
5. API returns success  
6. admin UI sees badge update on next poll  

---

## **5. Admin UI & Behaviour**

### **5.1 Icon States**

| Icon | Meaning |
|------|---------|
| Plain | browser not authenticated |
| Admin (no badge) | authenticated, no unread messages |
| Admin + badge | authenticated, unread messages exist |

### **5.2 Admin Panel Functions**

- list messages → loads index  
- view message → loads message artifact + state  
- mark read/unread → updates state + index  
- delete message → deletes artifact + state + index entry  

Admin UI interacts only with API endpoints.

### **5.3 Polling**
Admin browser polls unread count every 30–60 seconds.

---

## **6. Backend Endpoints**

### **Public**
- `POST /messages/submit`  
  - validates input  
  - calls storage to create message and update index  

### **Auth**
- `POST /auth/login`  
- `POST /auth/refresh`  
- `POST /auth/logout`  

### **Admin**
- `GET /admin/messages` → returns index  
- `GET /admin/messages/:id` → returns message artifact + state  
- `POST /admin/messages/:id/read` → updates state + index  
- `DELETE /admin/messages/:id` → deletes artifact + state + index entry  

---

## **7. Storage Model (Backend‑Agnostic)**

The system uses a pluggable storage backend selected via configuration.  
All backends must satisfy the same interface and invariants.

### **7.1 Message Artifact**
Each message is stored as a single immutable artifact in the chosen format.

Examples:

- `.eml` file  
- `.json` file  
- `.yaml` file  
- SQLite row  
- Maildir message  

### **7.2 Message State**
Mutable state (read/unread, timestamps, tags) is stored separately from the message artifact.

Examples:

- `.meta.json`  
- SQLite table  
- YAML sidecar  
- Maildir flags  

### **7.3 Index**
The storage backend maintains a fast‑lookup index containing:

- id  
- timestamp  
- sender  
- subject or preview  
- read/unread  

Index format is backend‑agnostic:

- JSON file  
- SQLite table  
- YAML file  

### **7.4 Regeneration**
All backends must support index regeneration from canonical message artifacts.

### **7.5 Storage Interface**
All backends must implement:

```
save_message(message_obj)
load_message(id)
delete_message(id)
list_messages()
update_state(id, state_obj)
regenerate_index()
```

The API module interacts only with this interface.

---

## **8. Security Model**

### **8.1 Threat Model**
Assumes:

- visitors are untrusted  
- admin browser is trusted  
- backend is trusted  
- no sensitive data stored  
- single‑admin system  

### **8.2 Protections**
- rate limiting  
- no admin actions without session token  
- refresh token cannot perform admin actions  
- no CORS exposure  
- escape all message content on render  
- no SQL injection (no SQL required)  

### **8.3 Not Included**
- multi‑admin  
- real‑time chat  
- CRM features  
- notification system  

---

## **9. Failure Modes**

### **9.1 Token Expiry**
- auto‑refresh fails  
- refresh token cleared  
- admin UI disappears  

### **9.2 Backend Down**
- visitor messages fail gracefully  
- admin UI shows “offline”  

### **9.3 Storage Corruption**
- storage regenerates index from canonical artifacts  
- admin UI shows “Unable to load messages” if regeneration fails  

---

## **10. Deployment & Maintenance**

### **10.1 Frontend**
- static GitHub Pages  
- no build pipeline  
- ensure Jekyll does not exclude admin JS  

### **10.2 Backend**
Four files:

```
api.py
auth.py
storage.py
config.py
```

### **10.3 Backups**
- backup message artifacts  
- backup index (optional — regenerable)  
- backup state files if applicable  

---

## **11. Invariants (Do Not Break)**

- admin UI must never appear to non‑admins  
- refresh token must never grant admin access  
- session token must never persist across browser restarts  
- message submission must always be frictionless  
- storage backend must be swappable via configuration without changes to API, Auth, or Frontend  
- canonical message artifacts must be immutable  
- system must remain inspectable and debuggable  
- no hidden complexity  
- no silent failures  

---

## **12. Summary**

This architecture provides:

- a minimal, durable, inspectable inbox  
- a clean separation of concerns  
- a pluggable storage backend  
- a static‑site‑friendly admin UI  
- a simple, secure authentication model  
- a system that is easy to maintain, extend, and reason about  

It is a small, sharp, reliable tool designed to do one job well.

---

