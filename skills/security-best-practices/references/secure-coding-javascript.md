# Secure Coding Patterns - JavaScript/TypeScript

## Helmet Security Headers

```javascript
const helmet = require('helmet');
const express = require('express');
const app = express();

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'", "https://api.yourdomain.com"],
      fontSrc: ["'self'"],
      objectSrc: ["'none'"],
      mediaSrc: ["'self'"],
      frameSrc: ["'none'"],
    },
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  },
  referrerPolicy: {
    policy: ["strict-origin-when-cross-origin"]
  }
}));
```

## CSRF Protection

```javascript
const csurf = require('csurf');
const cookieParser = require('cookie-parser');

app.use(cookieParser());
const csrfProtection = csurf({ 
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict'
  }
});

// Apply to routes that need CSRF protection
app.get('/form', csrfProtection, (req, res) => {
  res.render('send', { csrfToken: req.csrfToken() });
});

app.post('/submit', csrfProtection, (req, res) => {
  res.send('Form processed');
});
```

## Input Validation

```javascript
const { body, validationResult } = require('express-validator');

app.post('/user',
  body('email').isEmail().normalizeEmail(),
  body('password').isLength({ min: 8 }).trim(),
  body('username').trim().escape(),
  body('age').isInt({ min: 18, max: 120 }),
  (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ 
        errors: errors.array().map(err => ({
          field: err.path,
          message: err.msg
        }))
      });
    }
    
    // Process valid input
    const { email, password, username, age } = req.body;
    // ...
  }
);
```

## SQL Injection Prevention

```javascript
// ❌ DON'T: String concatenation
const query = `SELECT * FROM users WHERE id = ${userId}`;
db.query(query, callback);

// ✅ DO: Parameterized queries
const query = 'SELECT * FROM users WHERE id = ?';
db.query(query, [userId], callback);

// ✅ DO: Using prepared statements (mysql2/promise)
const [rows] = await connection.execute(
  'SELECT * FROM users WHERE email = ? AND active = ?',
  [email, true]
);

// ✅ DO: Using ORM (Sequelize)
const user = await User.findOne({
  where: {
    email: email,
    active: true
  }
});
```

## Password Hashing with bcrypt

```javascript
const bcrypt = require('bcrypt');
const SALT_ROUNDS = 12;

async function hashPassword(password) {
  try {
    const salt = await bcrypt.genSalt(SALT_ROUNDS);
    const hash = await bcrypt.hash(password, salt);
    return hash;
  } catch (error) {
    throw new Error('Password hashing failed');
  }
}

async function verifyPassword(password, hash) {
  try {
    const match = await bcrypt.compare(password, hash);
    return match;
  } catch (error) {
    throw new Error('Password verification failed');
  }
}

// Usage
const hashedPassword = await hashPassword(userPassword);
const isValid = await verifyPassword(inputPassword, hashedPassword);
```

## Secure Session Configuration

```javascript
const session = require('express-session');
const RedisStore = require('connect-redis')(session);

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  name: 'sessionId',  // Don't use default 'connect.sid'
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    sameSite: 'strict',
    maxAge: 24 * 60 * 60 * 1000  // 24 hours
  }
}));
```

## XSS Prevention with DOMPurify

```javascript
const createDOMPurify = require('dompurify');
const { JSDOM } = require('jsdom');

const window = new JSDOM('').window;
const DOMPurify = createDOMPurify(window);

function sanitizeHTML(dirty) {
  return DOMPurify.sanitize(dirty, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p'],
    ALLOWED_ATTR: ['href', 'title']
  });
}

// Usage
const userContent = req.body.content;
const cleanContent = sanitizeHTML(userContent);
```

## JWT Secure Implementation

```javascript
const jwt = require('jsonwebtoken');

const JWT_SECRET = process.env.JWT_SECRET;
const JWT_EXPIRES_IN = '1h';

function generateToken(payload) {
  return jwt.sign(
    payload,
    JWT_SECRET,
    { 
      expiresIn: JWT_EXPIRES_IN,
      issuer: 'your-app',
      audience: 'your-api'
    }
  );
}

function verifyToken(token) {
  try {
    return jwt.verify(token, JWT_SECRET, {
      issuer: 'your-app',
      audience: 'your-api'
    });
  } catch (error) {
    throw new Error('Invalid token');
  }
}

// Middleware
function authenticateToken(req, res, next) {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];  // Bearer TOKEN

  if (!token) {
    return res.sendStatus(401);
  }

  try {
    const user = verifyToken(token);
    req.user = user;
    next();
  } catch (error) {
    return res.sendStatus(403);
  }
}
```

## Rate Limiting

```javascript
const rateLimit = require('express-rate-limit');

// General API rate limit
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 100,  // Limit each IP to 100 requests per windowMs
  message: 'Too many requests from this IP',
  standardHeaders: true,
  legacyHeaders: false,
});

// Strict rate limit for authentication
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,  // 5 login attempts per 15 minutes
  skipSuccessfulRequests: true,
});

app.use('/api/', apiLimiter);
app.use('/auth/', authLimiter);
```

## Secure File Upload

```javascript
const multer = require('multer');
const path = require('path');

const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, 'uploads/');
  },
  filename: (req, file, cb) => {
    // Sanitize filename
    const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1E9);
    const ext = path.extname(file.originalname).toLowerCase();
    cb(null, file.fieldname + '-' + uniqueSuffix + ext);
  }
});

const fileFilter = (req, file, cb) => {
  const allowedMimes = ['image/jpeg', 'image/png', 'application/pdf'];
  const allowedExts = ['.jpg', '.jpeg', '.png', '.pdf'];
  
  const ext = path.extname(file.originalname).toLowerCase();
  
  if (allowedMimes.includes(file.mimetype) && allowedExts.includes(ext)) {
    cb(null, true);
  } else {
    cb(new Error('Invalid file type'), false);
  }
};

const upload = multer({
  storage: storage,
  fileFilter: fileFilter,
  limits: {
    fileSize: 5 * 1024 * 1024  // 5MB limit
  }
});

app.post('/upload', upload.single('document'), (req, res) => {
  res.json({ filename: req.file.filename });
});
```

## Environment Variable Validation

```javascript
const requiredEnvVars = [
  'DATABASE_URL',
  'JWT_SECRET',
  'SESSION_SECRET',
  'API_KEY'
];

function validateEnv() {
  const missing = requiredEnvVars.filter(varName => !process.env[varName]);
  
  if (missing.length > 0) {
    console.error('Missing required environment variables:', missing);
    process.exit(1);
  }
}

validateEnv();
```

## Safe JSON Parsing

```javascript
function safeJSONParse(str, defaultValue = null) {
  try {
    return JSON.parse(str);
  } catch (error) {
    return defaultValue;
  }
}

// Usage
const data = safeJSONParse(req.body.json, {});
```
