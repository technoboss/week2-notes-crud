### Week 2 Homework 1: Simple CRUD with FastAPI and SQLite

In this homework exercise, you will build a simple CRUD API using FastAPI and SQLite.

CRUD means:

- Create
- Read
- Update
- Delete

Most database-backed applications use these four actions. For example, a notes app needs to create a note, read notes, edit a note, and delete a note.

We will use a small `notes` table to keep the example simple.

#### What You Will Learn

- How to create rows from an API request.
- How to read rows from SQLite.
- How to update rows.
- How to delete rows.
- How to test non-GET endpoints using FastAPI docs.

#### Part A: Create a New Project

1. Create a new folder called:

```text
week2-notes-crud
```

2. Open it in VS Code.

3. Open a new VS Code terminal.

4. Create and activate a virtual environment.

On macOS/Linux:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

On Windows PowerShell:

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```

5. Create `requirements.txt`:

```text
fastapi==0.141.1
uvicorn==0.52.4
```

6. Install the requirements.

```bash
python -m pip install -r requirements.txt
```

7. Create `main.py`.

#### Before You Add the Code

This is a guided example. Read the code, run it, then complete the testing tasks below using your own note content.

| Action                 | HTTP method and path      | Successful response        |
| ---------------------- | ------------------------- | -------------------------- |
| Create                 | `POST /notes`             | `201` and the new note     |
| Read all               | `GET /notes`              | `200` and a list of notes  |
| Read one               | `GET /notes/{note_id}`    | `200` and one note         |
| Update selected fields | `PATCH /notes/{note_id}`  | `200` and the updated note |
| Delete                 | `DELETE /notes/{note_id}` | `200` and a confirmation   |

**Status codes:** `200` means OK, `201` means Created, `404` means Not Found, and `422` means the request failed validation.

`BaseModel` comes from Pydantic, which FastAPI installs as a dependency. `NoteCreate` describes the JSON request body: both fields are required strings. `NoteUpdate` allows a partial update, so either field may be omitted. In this simple example, an omitted field or an explicit `null` leaves its existing value unchanged.

SQL placeholders (`?`) pass values separately from the SQL command. This avoids treating user input as SQL. `(note_id,)` is a tuple containing one value. `connection.commit()` saves database changes; `lastrowid` gives the ID of the row just inserted.

#### Part B: Add the CRUD API

8. Add this code to `main.py`:

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from pathlib import Path
import sqlite3

app = FastAPI()

DATABASE_PATH = Path(__file__).resolve().parent / "notes.db"


class NoteCreate(BaseModel):
    title: str
    text: str


class NoteUpdate(BaseModel):
    title: str | None = None
    text: str | None = None


def get_connection():
    connection = sqlite3.connect(DATABASE_PATH)
    connection.row_factory = sqlite3.Row
    return connection


def create_table():
    connection = get_connection()
    cursor = connection.cursor()
    cursor.execute(
        """
        CREATE TABLE IF NOT EXISTS notes (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            text TEXT NOT NULL
        )
        """
    )
    connection.commit()
    connection.close()


create_table()


@app.get("/")
def home():
    return {"message": "Notes API is running"}


@app.get("/notes")
def get_notes():
    connection = get_connection()
    cursor = connection.cursor()
    cursor.execute("SELECT * FROM notes ORDER BY id")
    rows = cursor.fetchall()
    connection.close()
    return [dict(row) for row in rows]


@app.get("/notes/{note_id}")
def get_note(note_id: int):
    connection = get_connection()
    cursor = connection.cursor()
    cursor.execute("SELECT * FROM notes WHERE id = ?", (note_id,))
    row = cursor.fetchone()
    connection.close()

    if row is None:
        raise HTTPException(status_code=404, detail="Note not found")

    return dict(row)


@app.post("/notes", status_code=201)
def create_note(note: NoteCreate):
    connection = get_connection()
    cursor = connection.cursor()
    cursor.execute(
        "INSERT INTO notes (title, text) VALUES (?, ?)",
        (note.title, note.text)
    )
    connection.commit()
    note_id = cursor.lastrowid
    connection.close()
    return {"id": note_id, "title": note.title, "text": note.text}


@app.patch("/notes/{note_id}")
def update_note(note_id: int, note: NoteUpdate):
    connection = get_connection()
    cursor = connection.cursor()

    cursor.execute("SELECT * FROM notes WHERE id = ?", (note_id,))
    existing = cursor.fetchone()

    if existing is None:
        connection.close()
        raise HTTPException(status_code=404, detail="Note not found")

    new_title = note.title if note.title is not None else existing["title"]
    new_text = note.text if note.text is not None else existing["text"]

    cursor.execute(
        "UPDATE notes SET title = ?, text = ? WHERE id = ?",
        (new_title, new_text, note_id)
    )
    connection.commit()
    connection.close()

    return {"id": note_id, "title": new_title, "text": new_text}


@app.delete("/notes/{note_id}")
def delete_note(note_id: int):
    connection = get_connection()
    cursor = connection.cursor()

    cursor.execute("SELECT * FROM notes WHERE id = ?", (note_id,))
    existing = cursor.fetchone()

    if existing is None:
        connection.close()
        raise HTTPException(status_code=404, detail="Note not found")

    cursor.execute("DELETE FROM notes WHERE id = ?", (note_id,))
    connection.commit()
    connection.close()

    return {"message": "Note deleted"}
```

#### Part C: Run and Test

9. Run the app:

```bash
uvicorn main:app --reload
```

10. Open FastAPI docs:

```text
http://127.0.0.1:8000/docs
```

11. Expand `POST /notes` in `/docs`, click **Try it out**, replace the example request body with the JSON below, and click **Execute**:

```json
{ "title": "First note", "text": "This note is stored in SQLite." }
```

The response should have status **201** and include the created note's `id`. Record that ID; use it in all following requests. The app creates `notes.db` beside `main.py` on startup, and notes persist between server restarts.

12. Open in the browser:

```text
http://127.0.0.1:8000/notes
```

You should see your note.

13. In `/docs`, expand `GET /notes/{note_id}`, click **Try it out**, enter the ID returned by `POST`, and click **Execute**. Expect `200` and that note.

14. Test `PATCH /notes/{note_id}` with the same ID and this JSON body:

```json
{ "title": "Updated first note" }
```

Expect `200`. The title changes and the text stays the same. Repeat using your own note content and update only the `text` field.

15. Stop the server with `Ctrl + C`, restart it, and read the note again. Its changes should still be present.

16. Test `DELETE /notes/{note_id}` with the same ID. Expect `200` and `{"message":"Note deleted"}`. Read that ID again; expect `404` and `{"detail":"Note not found"}`.

17. Choose an ID that is absent from `GET /notes`, such as `9999`. Send `PATCH /notes/9999` with `{"title":"Missing note test"}`, then `DELETE /notes/9999`; both should return `404`. Send `POST /notes` with `{"title":"Incomplete note"}` and no text; expect `422`, meaning the request body did not match the required model.

18. Stop the server with `Ctrl + C` before starting Homework 2. If it is already running when you return, use that server rather than starting a second one on port `8000`.

#### Submission Checklist

Before you finish, make sure:

- `GET /notes` returns notes
- `GET /notes/{note_id}` returns one note
- `POST /notes` creates a note
- `PATCH /notes/{note_id}` updates a note
- `DELETE /notes/{note_id}` deletes a note
- missing notes return a `404` error

If PowerShell blocks activation, refer to [Week 1's activation instructions](../../class-1/lab/part-1.md). If you see `Address already in use`, stop the earlier server.

Homework 1 is complete. Continue to [Homework 2](homework-2.md).

### Week 2 Homework 2: Test APIs with Postman

In this homework exercise, you will use Postman to make API calls to your FastAPI app.

Postman is an API client. It lets you send requests such as `GET`, `POST`, `PATCH`, and `DELETE` without writing code.

#### What You Will Learn

- How to download Postman.
- How to send a `GET` request.
- How to send a `POST` request with JSON.
- How to send a `PATCH` request.
- How to send a `DELETE` request.
- How to read the response status code and JSON body.

#### Part A: Download Postman

1. Open the official Postman download page:

```text
https://www.postman.com/downloads/
```

Use the official Postman site only.

2. Download the desktop app for your operating system.

Postman supports Windows, macOS, and Linux. The official installation guide says to download the latest desktop app from the Postman download page and choose the option for your operating system.

For installation steps specific to your system, follow the [official Postman installation guide](https://learning.postman.com/docs/getting-started/installation/install-app/).

Use the **desktop app** for this homework so you can send requests directly to your local API at `127.0.0.1`.

3. Install and open Postman.

You may be asked to create or sign in to a Postman account. Follow the screen instructions.

#### Watch the Basics First

Watch [How to Send Your First API Request in Postman](https://www.youtube.com/watch?v=2oOSnxZ28fA), published on **Postman's official YouTube channel** (about 9 minutes).

Focus on creating a request, sending a GET request, and reading the JSON response. The video demonstrates a different API; for this homework, use your Notes API and the URLs below.

The video uses an earlier Postman interface, so some buttons may look different. The [current official quick-start guide](https://learning.postman.com/docs/getting-started/quick-start) provides an up-to-date reference.

#### Part B: Start Your API

4. Open your `week2-notes-crud` project in VS Code.

5. Activate your virtual environment.

6. Run the API:

```bash
uvicorn main:app --reload
```

Keep the server running while you use Postman. If the API is already running, skip the second start command. An `Address already in use` error means another process is using port `8000`.

#### Part C: Send a GET Request

7. In Postman, open a new HTTP request tab using **Add** (the **+** button) in the workbench.

8. Set the method to:

```text
GET
```

9. Enter this URL:

```text
http://127.0.0.1:8000/notes
```

10. Click **Send**.

Expect status **200** and a JSON list. If there are no notes yet, you may see:

```json
[]
```

#### Part D: Send a POST Request

11. Create a new request.

12. Set the method to:

```text
POST
```

13. Enter this URL:

```text
http://127.0.0.1:8000/notes
```

14. Open the **Body** tab.

15. Select **raw**.

16. Change the body type to **JSON**.

17. Add this JSON:

```json
{ "title": "Postman note", "text": "I created this note using Postman." }
```

18. Click **Send**.

Expect status **201** and a response with an `id`, `title`, and `text`. Copy the returned `id` for the next requests. IDs do not restart at `1` when notes are deleted, so do not assume your new note has ID `1`.

#### Part E: Send a PATCH Request

19. Create a new request.

20. Set the method to:

```text
PATCH
```

21. Enter this URL:

```text
http://127.0.0.1:8000/notes/1
```

Replace `1` with the exact `id` returned by your POST request.

22. Open the **Body** tab.

23. Select **raw** and **JSON**.

24. Add this JSON:

```json
{ "title": "Updated from Postman" }
```

25. Click **Send**.

Expect status **200** and the updated note. The original text should remain unchanged because you sent only a title.

#### Part F: Send a DELETE Request

26. Create a new request.

27. Set the method to:

```text
DELETE
```

28. Enter this URL:

```text
http://127.0.0.1:8000/notes/1
```

Replace `1` with the exact `id` returned by your POST request.

29. Click **Send**.

Expect status **200** and:

```json
{ "message": "Note deleted" }
```

30. Send `GET /notes` again to check that this note is gone. Other notes may still be listed. Also request the deleted ID with `GET /notes/{note_id}`; expect **404** and `{"detail":"Note not found"}`.

#### Part G: Stop the Server

31. Go back to the VS Code terminal.

32. Stop the server with:

```text
Ctrl + C
```

#### Troubleshooting

- **Connection refused / could not send request:** check that Uvicorn is running and the URL uses `http://127.0.0.1:8000`.
- **`422` after POST:** check that the body is set to raw JSON and contains both `title` and `text` as strings.
- **`404` after PATCH or DELETE:** use the ID from your POST response and check that the note has not already been deleted.

#### Submission Checklist

Before you finish, make sure:

- Postman is installed.
- `GET /notes` works in Postman.
- `POST /notes` creates a note.
- `PATCH /notes/{note_id}` updates a note.
- `DELETE /notes/{note_id}` deletes a note.
- You can find the response status code in Postman.

Homework 2 is complete. Follow the [homework submission instructions](submission.md) to share your public repository in Teams for feedback, then complete the [Weekend Activity: Cloud and AI](../lab/weekend-activity.md).
