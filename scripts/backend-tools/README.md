# Backend Utility Scripts

This folder contains thin wrappers over backend HTTP endpoints that currently do not have UI coverage. They rely only on Python's standard library (`urllib`), so no extra packages are needed.

All scripts expect the backend to run on `http://localhost:8000` (override via `API_BASE_URL`). For actions that require authentication, the scripts log in with the provided MAX ID by calling `/api/v1/auth/login-by-max-id`.

## Quick Start

1. Ensure backend services are running (see `TESTING.md`). Swagger UI is available at `http://localhost:8000/docs` — refer to it whenever you need to inspect request/response schemas or look up UUIDs (groups, faculties, events, etc.).
2. Export the following environment variables once (PowerShell syntax shown):

   ```powershell
   $env:API_BASE_URL="http://localhost:8000/api/v1"
   $env:ADMIN_MAX_ID="900001"        # MAX ID of a staff/admin account (register via Swagger if needed)
   $env:STUDENT_MAX_ID="1001"        # MAX ID of a student for payment tests
   ```

   Adjust the numbers to match real accounts in your database. You can always register a user through Swagger (`POST /api/v1/auth/register`) and assign the desired `max_id`.

3. Run scripts with the system Python 3:

   ```powershell
   python scripts/backend-tools/send_broadcast.py --title "Тест" --message "Добрый день!"
   ```

Each script prints the JSON response or a readable error. Use `--max-id <value>` to override the default MAX ID (otherwise the relevant env var is used).

## Available Scripts

### `send_broadcast.py`

- Calls `POST /api/v1/broadcasts`.
- Requires a staff/admin MAX ID (`ADMIN_MAX_ID` env var).
- Optional parameters `--group-id` or `--faculty-id` allow targeting a specific audience.

Example:

```powershell
python scripts/backend-tools/send_broadcast.py `
  --title "Актуальные новости" `
  --message "Сегодня в 18:00 состоится собрание" `
  --max-id 900001
```

### `create_payment.py`

- Calls `POST /api/v1/payments` as the specified user.
- Works for tuition/dormitory/event payments (see Swagger schema `PaymentCreate`).
- Accepts the amount in RUB (converted to kopeks automatically).
- Use `--event-id` when `--type=event`.

Example:

```powershell
python scripts/backend-tools/create_payment.py `
  --max-id 1001 `
  --type tuition `
  --amount 1500 `
  --period "Семестр 1" `
  --description "Добровольный взнос"
```

### `add_student.py`

- Calls `POST /api/v1/users/students/add`.
- Requires basic student data (full name, city, student card).
- If UUIDs are not supplied, the script fetches the first university/faculty/group via `/universities` endpoints and reports what was chosen.
- Use `--print-only` to preview the payload before sending.

Example:

```powershell
python scripts/backend-tools/add_student.py `
  --full-name "Иван Петров" `
  --city "Москва" `
  --student-card "STU900" `
  --print-only
```

Remove `--print-only` to actually create the student.

## Tips

- Swagger (`http://localhost:8000/docs`) remains the authoritative reference for every endpoint, including response fields, enums, and validation rules.
- When a script fails, it prints the HTTP status and backend error text — forward this to the backend team if something needs investigation.
- Feel free to extend these scripts or add new ones for other endpoints; re-use `common.py` to keep behavior consistent.
