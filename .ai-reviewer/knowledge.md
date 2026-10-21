# repo reviewer notes

## Architecture

This is a small FastAPI application implemented entirely in `main.py`. It exposes CRUD and query endpoints for students, using a Pydantic request model for validation and a process-local Python list as the datastore. The README currently contains no additional architectural or operational guidance.

## Conventions

- Keep the API implementation in `main.py`; endpoints are grouped with large section comments such as `# CREATE STUDENT` and `# UPDATE STUDENT`.
- Use FastAPI decorators directly on functions, e.g. `@app.get("/students")`, `@app.post("/students")`, and `@app.delete("/students/{student_id}")` (`main.py`).
- Define request validation with Pydantic models and `Field` constraints. `Student` requires names and courses of at least two characters and ages from 1 through 100 (`main.py`).
- Use typed path parameters, such as `student_id: int`, so FastAPI performs URL parameter conversion and validation (`main.py`).
- Successful responses are plain dictionaries with stable wrapper keys: collection endpoints return `"students"`, single-resource endpoints return `"student"`, and mutations include a human-readable `"message"` (`main.py`).
- Resource lookup failures consistently raise `HTTPException(status_code=404, detail="Student not found")` rather than returning `None` or an empty success response (`main.py`).
- Student searches are case-insensitive. Name search uses substring matching, while course filtering uses case-insensitive exact matching (`search_student` and `get_students_by_course` in `main.py`).
- IDs are generated as one greater than the current maximum, with `default=0` for an empty collection (`create_student` in `main.py`).

## Intentional non-standard choices

- The application intentionally uses an in-memory global list of dictionaries (`students` in `main.py`) rather than a database or repository layer; data is process-local and is lost on restart.
- There are no separate response schemas: endpoint responses are assembled as plain dictionaries, while `Student` is used only for validated create/update input.
- The API returns custom success messages and wrapped payloads instead of conventional bare resources or HTTP `204` responses for update/delete operations.

## Watch out for

- Do not introduce persistence assumptions: changes to `students` must account for the fact that this is shared mutable process state and is not safe as durable or multi-worker storage.
- Flag ID-generation changes that create collisions under concurrent requests; `max(...) + 1` is not atomic (`create_student` in `main.py`).
- Preserve the route behavior and avoid adding broad dynamic routes before specific paths such as `/students/search`, `/students/count`, and `/students/course/{course_name}`.
- Preserve case-insensitive behavior for both name search and course matching unless the API contract is deliberately changed.
- Ensure new student fields are added consistently to the `Student` model, creation mapping, update mapping, and returned dictionaries; these are currently maintained manually in multiple places.
- Be cautious when modifying deletion during iteration (`students.remove(student)` in `delete_student`); changing the loop or collection handling can cause skipped items or altered semantics.