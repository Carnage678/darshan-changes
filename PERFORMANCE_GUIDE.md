# Performance & Scalability Implementation Guide

## Quick Start for Vercel Deployment

### 1. Set Up PostgreSQL (Vercel)
```bash
# 1. Create a Vercel PostgreSQL database
# Visit: https://vercel.com/dashboard/stores

# 2. Set environment variables in Vercel
POSTGRES_URL=postgres://user:password@host:5432/db
POSTGRES_PRISMA_URL=$POSTGRES_URL
POSTGRES_URL_NON_POOLING=$POSTGRES_URL
```

### 2. Update Prisma Schema
```prisma
// backend/prisma/schema.prisma
datasource db {
  provider = "postgresql"  // Change from "sqlite"
  url      = env("DATABASE_URL")
}
```

### 3. Deploy
```bash
# Run migrations
npm run prisma:migrate

# Deploy to Vercel
vercel deploy --prod
```

---

## Database Query Performance Tips

### ✅ GOOD Query Patterns (Fast)

**1. Select Only Needed Fields**
```typescript
// Fast: Only 4 fields fetched
const students = await db.student.findMany({
  select: {
    id: true,
    name: true,
    rollNo: true,
    className: true,
  },
  where: { className: 'AIML-3' },
  take: 50,
  skip: 0,
});
```

**2. Use Indexes**
```typescript
// Fast: Uses index on studentId + subjectId
const marks = await db.mark.findMany({
  where: {
    studentId: 'student-123',
    subjectId: 'subject-456',
  },
});
```

**3. Batch Operations**
```typescript
// Fast: One query for multiple records
const marks = await db.mark.findMany({
  where: {
    studentId: {
      in: ['student-1', 'student-2', 'student-3'],
    },
  },
});
```

**4. Pagination**
```typescript
// Fast: Limits result set
const page = 1;
const pageSize = 50;

const students = await db.student.findMany({
  skip: (page - 1) * pageSize,
  take: pageSize,
  where: { className: 'AIML-3' },
});
```

### ❌ BAD Query Patterns (Slow)

**1. Fetching All Fields**
```typescript
// Slow: Loads every column including large text/blob fields
const students = await db.student.findMany(); // No select
```

**2. N+1 Queries**
```typescript
// SLOW: 1 + 60 = 61 queries!
const students = await db.student.findMany();
for (const student of students) {
  student.marks = await db.mark.findMany({
    where: { studentId: student.id },
  });
}

// FAST: 1 query with include
const students = await db.student.findMany({
  include: {
    marks: true,
  },
});
```

**3. Loading All Data at Once**
```typescript
// Slow: No pagination, loads 10,000 records
const allStudents = await db.student.findMany({
  include: { marks: true, attendance: true },
});

// Fast: Paginate
const students = await db.student.findMany({
  select: { id: true, name: true },
  take: 50,
  skip: 0,
});
```

---

## Caching Strategy

### Implement Redis Caching
```typescript
// backend/src/lib/cache.ts
import Redis from 'redis';

const redis = Redis.createClient({
  url: process.env.REDIS_URL || 'redis://localhost:6379',
});

redis.on('error', (err) => console.error('Redis error:', err));

export async function getCachedStudents(className: string) {
  const cacheKey = `students:${className}`;
  
  // Try cache first
  const cached = await redis.get(cacheKey);
  if (cached) {
    console.log(`Cache hit: ${cacheKey}`);
    return JSON.parse(cached);
  }
  
  // Fetch from DB
  console.log(`Cache miss: ${cacheKey}`);
  const students = await db.student.findMany({
    select: { id: true, name: true, rollNo: true },
    where: { className },
  });
  
  // Store in cache for 1 hour
  await redis.setEx(cacheKey, 3600, JSON.stringify(students));
  return students;
}

// Invalidate when data changes
export async function invalidateStudentsCache(className: string) {
  await redis.del(`students:${className}`);
}
```

### Cache Warm-up on Startup
```typescript
// backend/src/cache-warmup.ts
export async function warmupCache() {
  console.log('Warming up cache...');
  
  const classes = await db.classSection.findMany({
    select: { name: true },
  });
  
  for (const cls of classes) {
    await getCachedStudents(cls.name);
  }
  
  console.log('Cache warmup complete');
}

// Call on app startup
app.listen(PORT, async () => {
  await warmupCache();
  console.log(`API running on port ${PORT}`);
});
```

---

## API Endpoint Optimization

### Add Response Compression
```typescript
// backend/src/index.ts
import compression from 'compression';

app.use(compression()); // Compresses responses >1KB

app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ limit: '10mb' }));
```

### Add Request Timeout
```typescript
// backend/src/middleware/timeout.ts
import timeout from 'connect-timeout';

app.use(timeout('30s')); // 30 second timeout

app.use((err, req, res, next) => {
  if (req.timedout) {
    res.status(408).json({ error: 'Request timeout' });
  } else {
    next(err);
  }
});
```

### Add Security Headers
```typescript
// backend/src/middleware/security.ts
import helmet from 'helmet';

app.use(helmet()); // Sets various HTTP headers

app.use((req, res, next) => {
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '1; mode=block');
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
  next();
});
```

---

## Frontend Performance Tips

### 1. Code Splitting
```typescript
// frontend/src/components/pages/MarksPage.tsx
import { lazy, Suspense } from 'react';

const StudentMarksView = lazy(() =>
  import('./StudentMarksView').then(m => ({ default: m.StudentMarksView }))
);

export function MarksPage() {
  return (
    <Suspense fallback={<Spinner />}>
      <StudentMarksView />
    </Suspense>
  );
}
```

### 2. Memoization (Prevent re-renders)
```typescript
import { useMemo, useCallback } from 'react';

export function StudentList({ students }) {
  // Only recompute when students changes
  const sortedStudents = useMemo(() => {
    return students.sort((a, b) => a.rollNo.localeCompare(b.rollNo));
  }, [students]);
  
  // Only recreate when handleClick actually changes
  const handleClick = useCallback((id) => {
    console.log('Clicked:', id);
  }, []);
  
  return (
    <div>
      {sortedStudents.map(s => (
        <div key={s.id} onClick={() => handleClick(s.id)}>
          {s.name}
        </div>
      ))}
    </div>
  );
}
```

### 3. Virtual Scrolling (for 10k+ records)
```typescript
import { FixedSizeList as List } from 'react-window';

export function LargeStudentList({ students }) {
  return (
    <List
      height={600}
      itemCount={students.length}
      itemSize={50}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style} className="student-row">
          {students[index].name}
        </div>
      )}
    </List>
  );
}
```

### 4. Image Optimization
```typescript
// Use next-gen formats
<picture>
  <source srcSet="/avatar.webp" type="image/webp" />
  <source srcSet="/avatar.png" type="image/png" />
  <img src="/avatar.png" alt="Avatar" width={48} height={48} />
</picture>

// Or use responsive images
<img
  src="/avatar.png"
  srcSet="/avatar-small.png 300w, /avatar-large.png 600w"
  sizes="(max-width: 600px) 300px, 600px"
  alt="Avatar"
/>
```

---

## Monitoring & Alerting

### Log Performance Metrics
```typescript
// backend/src/middleware/metrics.ts
export function metricsMiddleware(req, res, next) {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    
    // Log to console
    console.log({
      method: req.method,
      path: req.path,
      status: res.statusCode,
      duration_ms: duration,
      timestamp: new Date().toISOString(),
    });
    
    // Send to monitoring service (Sentry, DataDog, etc.)
    if (duration > 1000) {
      console.warn(`SLOW_REQUEST: ${req.method} ${req.path} (${duration}ms)`);
    }
  });
  
  next();
}

app.use(metricsMiddleware);
```

### Set Up Error Tracking
```typescript
// backend/src/lib/sentry.ts
import * as Sentry from "@sentry/node";

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 1.0,
});

app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.errorHandler());
```

---

## Load Testing

### Test with k6
```javascript
// scripts/load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 20 },   // Ramp up to 20 users
    { duration: '1m', target: 20 },    // Stay at 20 for 1 min
    { duration: '30s', target: 0 },    // Ramp down to 0
  ],
};

export default function () {
  const res = http.get('http://localhost:4000/api/students');
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  sleep(1);
}
```

Run it:
```bash
k6 run scripts/load-test.js
```

---

## Environment Variables for Production

```env
# Database
DATABASE_URL=postgres://user:pass@host:5432/db

# Cache
REDIS_URL=redis://user:pass@host:6379

# API
PORT=3000
NODE_ENV=production
CORS_ORIGIN=https://example.com

# Auth
JWT_SECRET=your-secret-key-here

# External APIs
GEMINI_API_KEY=your-key-here

# Monitoring
SENTRY_DSN=https://...

# Email (for notifications)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=app-password
```

---

## Deployment Checklist

- [ ] Update datasource to PostgreSQL in schema.prisma
- [ ] Run `npm run prisma:migrate` to update DB
- [ ] Add Redis for caching
- [ ] Add compression middleware
- [ ] Add rate limiting
- [ ] Add security headers
- [ ] Set up error tracking (Sentry)
- [ ] Configure monitoring
- [ ] Load test with k6
- [ ] Database query analysis
- [ ] Enable gzip compression
- [ ] Set up CDN for assets
- [ ] Configure email notifications
- [ ] Set up database backups
- [ ] Test failover/recovery
- [ ] Document scaling procedures

This setup handles 100k+ students with proper caching and optimization!
