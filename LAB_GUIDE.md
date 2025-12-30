# Adding React to Flask - Complete Guide

## Overview

This lab demonstrates how to build a full-stack web application with **React** (client) and **Flask** (server) communicating via HTTP requests. The application is a messaging platform called "Chatterbox" with full CRUD functionality.

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        BROWSER                                   │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    REACT CLIENT                          │   │
│   │                  (localhost:3000)                        │   │
│   │                                                         │   │
│   │   ┌─────────┐    ┌──────────┐    ┌─────────────────┐   │   │
│   │   │ Header  │    │ Search   │    │   MessageList   │   │   │
│   │   └─────────┘    └──────────┘    │   + NewMessage  │   │   │
│   │                                   └─────────────────┘   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                          │ HTTP (fetch)                         │
│                          ▼                                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ HTTP JSON
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      FLASK SERVER                                │
│                    (localhost:5555)                              │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                   REST API Routes                        │   │
│   │                                                         │   │
│   │   GET    /messages       → Get all messages             │   │
│   │   POST   /messages       → Create new message           │   │
│   │   PATCH  /messages/<id>  → Update a message             │   │
│   │   DELETE /messages/<id>  → Delete a message             │   │
│   │                                                         │   │
│   │   ┌─────────────────────────────────────────────────┐   │   │
│   │   │              SQLite Database                     │   │   │
│   │   │              (messages table)                   │   │   │
│   │   └─────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Client-Server Responsibilities

### React Client (`/client`)

| Responsibility       | Description                          |
| -------------------- | ------------------------------------ |
| **UI Rendering**     | Displays messages, forms, buttons    |
| **User Input**       | Handles typing, button clicks        |
| **HTTP Requests**    | Sends fetch() requests to Flask      |
| **State Management** | Stores messages, search terms, theme |

### Flask Server (`/server`)

| Responsibility     | Description                           |
| ------------------ | ------------------------------------- |
| **API Routes**     | Defines endpoints for CRUD operations |
| **Database**       | Stores and retrieves messages         |
| **Validation**     | Ensures data integrity                |
| **JSON Responses** | Returns data to React                 |

---

## 3. Running Two Servers

You MUST run two separate terminals:

### Terminal 1 - Flask Server

```bash
cd server
pipenv install          # Install dependencies
pipenv shell            # Activate virtual environment
export FLASK_APP=app.py
export FLASK_RUN_PORT=5555
flask db upgrade        # Run migrations
python seed.py          # Seed database with sample data
python app.py           # Start Flask server
```

- **URL**: http://127.0.0.1:5555

### Terminal 2 - React Client

```bash
cd client
npm install             # Install dependencies
npm start               # Start React dev server
```

- **URL**: http://localhost:3000

---

## 4. The Fetch Pattern - Most Important Concept

### GET Request - Fetching Data

```javascript
useEffect(() => {
  fetch("http://127.0.0.1:5555/messages")
    .then((r) => {
      if (!r.ok) throw new Error("Fetch failed");
      return r.json();
    })
    .then((messages) => setMessages(messages));
}, []);
```

**Step-by-step flow:**

1. React component mounts
2. useEffect runs (empty dependency array = run once)
3. HTTP GET request sent to Flask
4. Flask queries database and returns JSON
5. Promise resolves, JSON parsed
6. React state updated with new messages
7. UI re-renders with fetched data

### POST Request - Creating Data

```javascript
function handleSubmit(e) {
  e.preventDefault();

  fetch("http://127.0.0.1:5555/messages", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      username: currentUser.username,
      body: body,
    }),
  })
    .then((r) => r.json())
    .then((newMessage) => {
      onAddMessage(newMessage);
      setBody("");
    });
}
```

**Key requirements for POST/PATCH/DELETE:**

- `method`: HTTP verb (POST, PATCH, DELETE)
- `headers`: Content-Type: application/json
- `body`: Data serialized as JSON string

### PATCH Request - Updating Data

```javascript
fetch(`http://127.0.0.1:5555/messages/${id}`, {
  method: "PATCH",
  headers: {
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    body: messageBody,
  }),
});
```

### DELETE Request - Deleting Data

```javascript
fetch(`http://127.0.0.1:5555/messages/${id}`, {
  method: "DELETE",
});
```

---

## 5. Why CORS is Critical

**Cross-Origin Resource Sharing (CORS)** allows the browser to permit requests between different origins.

```
React:  localhost:3000      ← Different origin
Flask:  localhost:5555      ← Different origin
```

**Without CORS:**

```
❌ fetch() fails silently
❌ Console shows CORS error
❌ Nothing works
```

**Flask fix:**

```python
from flask_cors import CORS

app = Flask(__name__)
CORS(app)  # Enables all CORS requests
```

---

## 6. React Data Flow (Top → Down)

```
App (owns messages state)
├── Header
├── Search
├── MessageList
│   └── Message (x N)
│       ├── EditMessage (conditional)
│       └── Delete button
└── NewMessage
```

**Key principle:** Data flows downward, callbacks flow upward

```javascript
// App owns the state
function App() {
  const [messages, setMessages] = useState([]);

  function handleAddMessage(newMessage) {
    setMessages([...messages, newMessage]);
  }

  return <NewMessage onAddMessage={handleAddMessage} />;
}

// NewMessage receives callback, calls it when ready
function NewMessage({ onAddMessage }) {
  function handleSubmit() {
    // ... after fetch success
    onAddMessage(newMessage); // ← Reports back to App
  }
}
```

---

## 7. Complete API Reference

| Method | Endpoint         | Description                                  |
| ------ | ---------------- | -------------------------------------------- |
| GET    | `/messages`      | Returns all messages (ordered by created_at) |
| POST   | `/messages`      | Creates new message, returns created message |
| PATCH  | `/messages/<id>` | Updates message, returns updated message     |
| DELETE | `/messages/<id>` | Deletes message, returns `{deleted: true}`   |

### Example Responses

**GET /messages (200 OK):**

```json
[
  {
    "id": 1,
    "body": "Hello world!",
    "username": "Alex",
    "created_at": "2024-01-15T10:30:00",
    "updated_at": "2024-01-15T10:30:00"
  }
]
```

**POST /messages (201 Created):**

```json
{
  "id": 2,
  "body": "New message",
  "username": "Duane",
  "created_at": "2024-01-15T11:00:00",
  "updated_at": "2024-01-15T11:00:00"
}
```

---

## 8. Component Hierarchy

```
App.js
├── Header (theme toggle)
├── Search (filter messages)
├── MessageList
│   └── Message (x N)
│       ├── EditMessage (inline edit form)
│       └── actions (edit/delete buttons)
└── NewMessage (create form)
```

---

## 9. CodeGrade Testing Checklist

CodeGrade tests verify you can:

- [ ] Run both client and server
- [ ] Implement GET requests with fetch()
- [ ] Implement POST requests with proper headers
- [ ] Implement PATCH requests
- [ ] Implement DELETE requests
- [ ] Enable CORS on Flask
- [ ] Understand data flow between components
- [ ] Handle state updates after server responses

---

## 10. Minimal Working Example Summary

If you were to create the absolute minimum version:

**server/app.py** (8 lines core logic):

```python
from flask import Flask, jsonify, request
from flask_cors import CORS

app = Flask(__name__)
CORS(app)

messages = [{"id": 1, "username": "Alex", "body": "Hello"}]

@app.route("/messages", methods=["GET", "POST"])
def messages_route():
    if request.method == "GET":
        return jsonify(messages), 200
    data = request.json
    new_msg = {"id": len(messages) + 1, **data}
    messages.append(new_msg)
    return jsonify(new_msg), 201

app.run(port=5555)
```

**client/src/App.js** (minimal):

```javascript
import { useEffect, useState } from "react";

function App() {
  const [messages, setMessages] = useState([]);

  useEffect(() => {
    fetch("http://127.0.0.1:5555/messages")
      .then((r) => r.json())
      .then(setMessages);
  }, []);

  // ... form for POST
}
```

---

## 11. Key Takeaways

1. **Two separate apps** - React (client) and Flask (server) run independently
2. **HTTP is the glue** - fetch() connects them
3. **CORS is essential** - enables cross-origin requests
4. **State flows down, callbacks flow up** - React architecture pattern
5. **REST API** - Flask returns JSON, React consumes JSON
6. **Two terminals** - one for each server

---

## 12. Project Structure

```
/home/sumeya/python-p4-adding-react-to-flask/
├── client/                    # React application
│   ├── src/
│   │   ├── components/
│   │   │   ├── App.js         # Main component, state owner
│   │   │   ├── Header.js      # Theme toggle
│   │   │   ├── Search.js      # Message filter
│   │   │   ├── MessageList.js # Renders all messages
│   │   │   ├── Message.js     # Single message with edit/delete
│   │   │   ├── NewMessage.js  # Create new message
│   │   │   └── EditMessage.js # Edit existing message
│   │   ├── index.js           # React entry point
│   │   └── index.css          # Styles
│   └── package.json
│
├── server/                    # Flask application
│   ├── app.py                 # Flask routes & API
│   ├── models.py              # SQLAlchemy Message model
│   ├── seed.py                # Database seeding script
│   └── migrations/            # Alembic migrations
│
└── Pipfile                    # Python dependencies
```
