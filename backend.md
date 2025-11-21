# EtinuxE Backend

## Quickstart
1. Create and activate a Python 3.11+ virtual environment.
2. Open the backend folder:
   ```bash
   cd backend
   ```
3. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
4. Run the API:
   ```bash
   uvicorn app.main:app --reload
   ```
5. Use the CLI:
   ```bash
   python -m app.cli --help
   ```

The API listens on `http://127.0.0.1:8000`.
