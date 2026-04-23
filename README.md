# PDF Form Filler

Upload a PDF with form fields, fill them in the browser, and save the filled PDF to the backend.

## Setup

### Backend
```bash
cd backend
npm start
```
Runs on http://localhost:3000

### Frontend
```bash
cd frontend
npm run dev
```
Runs on http://localhost:5173 (proxies API calls to backend)

## Usage
1. Start the backend server
2. Start the frontend dev server
3. Open http://localhost:5173 in your browser
4. Upload a PDF file containing form fields
5. Fill in the form fields displayed in the browser
6. Click "Save & Upload Filled PDF" to send the filled PDF to the backend
7. Filled PDFs are saved in `backend/uploads/`
