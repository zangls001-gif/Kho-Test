Test
const http = require('node:http');
const fs = require('node:fs');
const path = require('node:path');
const { randomUUID } = require('node:crypto');

const PORT = process.env.PORT || 3000;
const DATA_FILE = path.join(__dirname, 'data', 'todos.json');
const PUBLIC_DIR = path.join(__dirname, 'public');

// MIME types for static assets
const MIME_TYPES = {
  '.html': 'text/html; charset=utf-8',
  '.css': 'text/css; charset=utf-8',
  '.js': 'application/javascript; charset=utf-8',
  '.json': 'application/json; charset=utf-8',
  '.svg': 'image/svg+xml',
  '.ico': 'image/x-icon'
};

// Helper to ensure data file exists
function ensureDataFile() {
  const dir = path.dirname(DATA_FILE);
  if (!fs.existsSync(dir)) {
    fs.mkdirSync(dir, { recursive: true });
  }
  if (!fs.existsSync(DATA_FILE)) {
    fs.writeFileSync(DATA_FILE, '[]', 'utf8');
  }
}

// Read todos from disk
function readTodos() {
  ensureDataFile();
  try {
    const raw = fs.readFileSync(DATA_FILE, 'utf8');
    return JSON.parse(raw || '[]');
  } catch (err) {
    console.error('Error reading todos file:', err);
    return [];
  }
}

// Write todos to disk
function writeTodos(todos) {
  ensureDataFile();
  fs.writeFileSync(DATA_FILE, JSON.stringify(todos, null, 2), 'utf8');
}

// Helper to parse JSON request body
function parseJsonBody(req) {
  return new Promise((resolve, reject) => {
    let body = '';
    req.on('data', chunk => {
      body += chunk;
      // Prevent payload flooding
      if (body.length > 1e6) {
        req.destroy();
        reject(new Error('Payload too large'));
      }
    });
    req.on('end', () => {
      if (!body) return resolve({});
      try {
        resolve(JSON.parse(body));
      } catch (err) {
        reject(new Error('Invalid JSON format'));
      }
    });
    req.on('error', reject);
  });
}

// Send standard JSON response
function sendJson(res, statusCode, data) {
  res.writeHead(statusCode, {
    'Content-Type': 'application/json; charset=utf-8',
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type'
  });
  res.end(JSON.stringify(data));
}

// Serve static files
function serveStatic(req, res, filePath) {
  const ext = path.extname(filePath).toLowerCase();
  const contentType = MIME_TYPES[ext] || 'application/octet-stream';

  fs.readFile(filePath, (err, content) => {
    if (err) {
      if (err.code === 'ENOENT') {
        res.writeHead(404, { 'Content-Type': 'text/plain; charset=utf-8' });
        res.end('404 Not Found');
      } else {
        res.writeHead(500, { 'Content-Type': 'text/plain; charset=utf-8' });
        res.end('500 Internal Server Error');
      }
      return;
    }
    res.writeHead(200, { 'Content-Type': contentType });
    res.end(content);
  });
}

const server = http.createServer(async (req, res) => {
  const parsedUrl = new URL(req.url, `http://${req.headers.host || 'localhost'}`);
  const pathname = parsedUrl.pathname;
  const method = req.method;

  // Handle CORS preflight
  if (method === 'OPTIONS') {
    res.writeHead(204, {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type'
    });
    res.end();
    return;
  }

  // --- API Routes ---
  if (pathname.startsWith('/api/todos')) {
    try {
      // GET /api/todos - list all
      if (method === 'GET' && pathname === '/api/todos') {
        const todos = readTodos();
        return sendJson(res, 200, todos);
      }

      // POST /api/todos/clear-completed - remove completed
      if (method === 'POST' && pathname === '/api/todos/clear-completed') {
        let todos = readTodos();
        todos = todos.filter(t => !t.completed);
        writeTodos(todos);
        return sendJson(res, 200, todos);
      }

      // POST /api/todos - create a new todo
      if (method === 'POST' && pathname === '/api/todos') {
        const body = await parseJsonBody(req);
        const text = (body.text || '').trim();

        if (!text) {
          return sendJson(res, 400, { error: 'Task text is required' });
        }

        const todos = readTodos();
        const newTodo = {
          id: randomUUID(),
          text,
          completed: false,
          createdAt: new Date().toISOString()
        };

        todos.unshift(newTodo);
        writeTodos(todos);
        return sendJson(res, 201, newTodo);
      }

      // PUT /api/todos/:id - update existing todo
      const putMatch = pathname.match(/^\/api\/todos\/([a-zA-Z0-9_-]+)$/);
      if (method === 'PUT' && putMatch) {
        const id = putMatch[1];
        const body = await parseJsonBody(req);
        const todos = readTodos();
        const index = todos.findIndex(t => t.id === id);

        if (index === -1) {
          return sendJson(res, 404, { error: 'Todo not found' });
        }

        if (typeof body.text === 'string' && body.text.trim()) {
          todos[index].text = body.text.trim();
        }
        if (typeof body.completed === 'boolean') {
          todos[index].completed = body.completed;
        }

        writeTodos(todos);
        return sendJson(res, 200, todos[index]);
      }

      // DELETE /api/todos/:id - delete a todo
      const deleteMatch = pathname.match(/^\/api\/todos\/([a-zA-Z0-9_-]+)$/);
      if (method === 'DELETE' && deleteMatch) {
        const id = deleteMatch[1];
        let todos = readTodos();
        const initialLength = todos.length;
        todos = todos.filter(t => t.id !== id);

        if (todos.length === initialLength) {
          return sendJson(res, 404, { error: 'Todo not found' });
        }

        writeTodos(todos);
        return sendJson(res, 200, { success: true, id });
      }

      // API route not recognized
      return sendJson(res, 404, { error: 'API endpoint not found' });
    } catch (err) {
      console.error('API Error:', err);
      return sendJson(res, 500, { error: err.message || 'Internal Server Error' });
    }
  }

  // --- Static File Serving ---
  let safePath = path.normalize(pathname).replace(/^(\.\.[\/\\])+/, '');
  if (safePath === '/' || safePath === '\\') {
    safePath = '/index.html';
  }

  const requestedFile = path.join(PUBLIC_DIR, safePath);

  // Prevent directory traversal outside PUBLIC_DIR
  if (!requestedFile.startsWith(PUBLIC_DIR)) {
    res.writeHead(403, { 'Content-Type': 'text/plain; charset=utf-8' });
    res.end('403 Forbidden');
    return;
  }

  serveStatic(req, res, requestedFile);
});

server.listen(PORT, () => {
  console.log(`\n========================================`);
  console.log(`  To-Do App is running!`);
  console.log(`  URL: http://localhost:${PORT}`);
  console.log(`========================================\n`);
});

