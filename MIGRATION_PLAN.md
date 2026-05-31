# Node.js → Python Backend Migration Plan

## Phase 1 — Setup

### Step 1: Project Structure
```
backend/
├── main.py                  ← entry point (like server.js)
├── app.py                   ← FastAPI app (like app.js)
├── requirements.txt         ← dependencies (like package.json)
├── .env                     ← same env variables
└── src/
    ├── db/
    │   └── database.py      ← MongoDB connection (like db.js)
    ├── models/
    │   ├── user.py          ← Pydantic schemas (like user.model.js)
    │   ├── chat.py
    │   └── message.py
    ├── routes/
    │   ├── auth.py          ← (like auth.routes.js)
    │   └── chat.py
    ├── controllers/
    │   ├── auth.py          ← (like auth.controller.js)
    │   └── chat.py
    ├── middlewares/
    │   └── auth.py          ← JWT guard (like auth.middleware.js)
    ├── services/
    │   ├── ai.py            ← Gemini (like ai.service.js)
    │   └── vector.py        ← Pinecone (like vector.service.js)
    └── sockets/
        └── socket_server.py ← Socket.io (like socket.server.js)
```

### Step 2: Install Dependencies
```bash
pip install fastapi uvicorn motor pymongo python-jose[cryptography] passlib[bcrypt] python-socketio python-dotenv google-genai pinecone
```

### Step 3: Create `.env`
Copy the exact same values from the Node `.env` file.

---

## Phase 2 — Build Order (bottom-up)

Build in this exact order — each step depends on the previous one:

| # | File | Node equivalent | Reason |
|---|------|-----------------|--------|
| 1 | `src/db/database.py` | `db/db.js` | Everything needs DB first |
| 2 | `src/models/*.py` | `models/*.model.js` | Controllers need models |
| 3 | `src/services/ai.py` | `services/ai.service.js` | No dependencies |
| 4 | `src/services/vector.py` | `services/vector.service.js` | No dependencies |
| 5 | `src/middlewares/auth.py` | `middlewares/auth.middleware.js` | Routes need this |
| 6 | `src/controllers/auth.py` | `controllers/auth.controller.js` | Routes need controllers |
| 7 | `src/routes/auth.py` | `routes/auth.routes.js` | App needs routes |
| 8 | `src/controllers/chat.py` | `controllers/chat.controller.js` | Routes need controllers |
| 9 | `src/routes/chat.py` | `routes/chat.routes.js` | App needs routes |
| 10 | `app.py` | `src/app.js` | Assembles everything |
| 11 | `main.py` | `server.js` | Entry point |
| 12 | `src/sockets/socket_server.py` | `sockets/socket.server.js` | Needs everything, most complex |

---

## Phase 3 — Testing Each Step

### After `database.py`
```bash
python -c "from src.db.database import connect_db; import asyncio; asyncio.run(connect_db())"
```
Expected: `Connected to MongoDB` printed, no errors.

### After models
```bash
python -c "from src.models.user import UserModel"
```
Expected: No errors.

### After services
```bash
python -c "import asyncio; from src.services.ai import generate_response; print(asyncio.run(generate_response('hello')))"
```
Expected: AI text response printed.

### After auth routes — start the server
```bash
uvicorn main:app --reload
```

Test register:
```bash
curl -X POST http://localhost:8000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"123456","full_name":{"first_name":"John","last_name":"Doe"}}'
```

Test login:
```bash
curl -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"123456"}'
```
Expected: user object + cookie in response.

### After chat routes
```bash
curl -X POST http://localhost:8000/api/chat \
  -H "Content-Type: application/json" \
  --cookie "token=YOUR_TOKEN_HERE" \
  -d '{"title":"My first chat"}'
```
Expected: chat object returned.

### After socket server
- Point the React frontend socket URL to `http://localhost:8000`
- Start the React app
- Log in, create a chat, send a message
- Check that `ai-response` event comes back with AI text

### FastAPI Interactive Docs (free, no curl needed)
```
http://localhost:8000/docs
```
Every route is listed here with a form to test it directly in the browser.

---

## Phase 4 — Pre-Deploy Checklist (Render)

- [ ] All routes work via `/docs`
- [ ] Socket messages go through and AI responds
- [ ] `.env` is in `.gitignore`
- [ ] `requirements.txt` has all dependencies pinned
- [ ] Server binds to `0.0.0.0` not `localhost`
- [ ] Port reads from environment: `os.environ.get("PORT", 8000)`
- [ ] CORS origin updated from `localhost:5173` to Render frontend URL

---

## Node → Python Concept Map

| Concept | Node | Python |
|---|---|---|
| Web framework | Express | FastAPI |
| DB ODM | Mongoose | Motor (async) + Pydantic |
| Data validation | Mongoose schema | Pydantic models |
| Sockets | socket.io | python-socketio |
| JWT | jsonwebtoken | python-jose |
| Password hashing | bcryptjs | passlib[bcrypt] |
| AI | @google/genai | google-genai |
| Vector DB | pinecone | pinecone |
| Dev server | nodemon | uvicorn --reload |
| Env vars | dotenv | python-dotenv |
| Middleware | Express middleware | FastAPI Depends() |
| Async | async/await | async/await (same!) |
| package.json | dependencies | requirements.txt |

---

## Build Summary

```
Setup → DB → Models → Services → Middleware → Controllers → Routes → App → Main → Sockets → Test → Deploy
```
