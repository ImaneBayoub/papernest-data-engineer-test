# Papernest Data Engineer Test

## Time spent
~45min

## Setup

### Prerequisites
- Python 3.12
- PostgreSQL access

### Installation
```bash
git clone https://github.com/ton-username/papernest-data-engineer-test
cd papernest-data-engineer-test
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Structure
```
papernest-data-engineer-test/
│
├── main.ipynb              # Main script
├── calls_by_quarter.json   # Output Task 1
├── clients_calls.csv       # Output Task 2
├── requirements.txt
└── README.md
```

## Approach

### Database connection
A connection was first set up using `psycopg2`:
```python
conn = psycopg2.connect(host=..., user=..., ...)
```
However, `pd.read_sql()` generated warnings recommending **SQLAlchemy** as the preferred connectable. We therefore switched to:
```python
engine = create_engine("postgresql://testread:testread@host:5432/souscritootest")
```

### Task 1 - Number of calls per quarter
SQL query using a `CASE WHEN` to categorize calls by quarter.

Business rules applied:
- **Inbound**: `clientphonenumberin` NOT NULL, or both fields NULL (hidden number)
- **Outbound**: `clientphonenumberout` NOT NULL

Output: `calls_by_quarter.json`

### Task 2 - Call statistics per client
Joining `test_client` and `test_call` on phone number required some investigation:
- `phonenumber` in `test_client` is of type `TEXT` with a leading `0` (e.g. `0620324525`)
- `clientphonenumberin/out` in `test_call` are of type `INTEGER` without the leading `0` (e.g. `620324525`)

Solution: cast to `TEXT` + `SUBSTRING` to strip the leading `0`:
```sql
ON ca.clientphonenumberin::text = SUBSTRING(c.phonenumber, 2)
OR ca.clientphonenumberout::text = SUBSTRING(c.phonenumber, 2)
```

Total duration is converted to `HH:MM:SS` via a Python function to avoid the `X days HH:MM:SS` format that `pd.to_timedelta` produces beyond 24h.

Output: `clients_calls.csv` sorted by `total_duration_calls` DESC

## Known limitations
- The `GROUP BY firstname, lastname` in task 2 may merge two different clients sharing the same name 
- Clients with no calls do not appear in the CSV: we used an `INNER JOIN` which only returns clients with at least one matching call. A `LEFT JOIN` would have included all clients with `NULL` values for call statistics, but since the CSV focuses on call activity, this was an intentional choice.