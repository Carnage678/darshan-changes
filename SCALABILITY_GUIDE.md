# Scalability Architecture Guide

## Overview
This application is designed for horizontal and vertical scalability from thousands to hundreds of thousands of students. The architecture supports cloud deployment, caching, and database optimization.

---

## 1. Database Layer

### Current: SQLite (Local Dev)
- ✅ Perfect for development and testing
- ❌ Not suitable for production (single-file, no concurrency)

### Production: PostgreSQL
**Advantages:**
- ✅ Multi-user concurrency support
- ✅ ACID transactions
- ✅ Advanced indexing and query optimization
- ✅ Built-in connection pooling (pgBouncer)
- ✅ Replication for high availability
- ✅ Scales to millions of records

**Setup via Vercel Postgres:**
```env
DATABASE_URL=postgres://user:password@host:5432/db
```

**Schema Optimizations:**
- Indexes on frequently queried columns: `role`, `classId`, `studentId`, `subjectId`
- Foreign keys with cascading deletes
- Partitioning by semester/academic year for historical data
- Archive old academic years to separate tables

**Scaling Recommendations:**
1. **Connection Pooling** (pgBouncer/PgBouncer):
   ```bash
   # Use Vercel's built-in connection pooling
   # Default: 3 connections per deployment
   ```

2. **Read Replicas** (for 10k+ concurrent users):
   ```
   Primary DB (writes) → Read Replica 1 → Analytics queries
                      → Read Replica 2 → Reporting
   ```

3. **Partitioning Strategy** (for 100k+ students):
   ```sql
   -- Partition Marks table by subject
   CREATE TABLE marks_partition_iai PARTITION OF marks
     FOR VALUES IN ('22AI32');
   ```

---

## 2. API Layer (Backend)

### Current Architecture
- **Framework:** Express.js (lightweight, scalable)
- **Language:** Node.js v24 (LTS)
- **Pattern:** REST API with JWT authentication

### Scalability Optimizations

#### A. Load Balancing
**Production Setup (Vercel):**
- Automatic horizontal scaling via serverless functions
- Geographic distribution across edge locations
- Auto-scales based on traffic

**For Custom Hosting:**
```
Load Balancer (nginx/HAProxy)
├── Instance 1 (API)
├── Instance 2 (API)
├── Instance 3 (API)
└── Instance N (API)
    ↓
    Shared PostgreSQL + Redis Cache
```

#### B. Caching Layer (Redis)
Add Redis for 10x performance improvement:

```typescript
// backend/src/lib/cache.ts (NEW)
import Redis from 'redis';

const redis = Redis.createClient({
  url: env.redisUrl // 'redis://localhost:6379'
});

export const cache = {
  // Cache marks for 1 hour (frequently accessed, rarely updated)
  async getStudentMarks(studentId: string) {
    const cached = await redis.get(`marks:${studentId}`);
    if (cached) return JSON.parse(cached);
    
    const marks = await db.mark.findMany({ where: { studentId } });
    await redis.setEx(`marks:${studentId}`, 3600, JSON.stringify(marks));
    return marks;
  },
  
  // Invalidate on new mark added
  async invalidateStudentMarks(studentId: string) {
    await redis.del(`marks:${studentId}`);
  }
};
```

**Cache Strategy:**
- Marks: 1 hour TTL (low write frequency)
- Attendance: 30 minutes TTL
- Timetable: 24 hours TTL (rarely changes)
- Fees: 2 hours TTL
- User profiles: 12 hours TTL

#### C. Database Query Optimization
```typescript
// ✅ GOOD: Batch queries, select only needed fields
const students = await db.student.findMany({
  select: {
    id: true,
    name: true,
    usn: true,
    rollNo: true,
  },
  take: 100,        // Pagination
  skip: offset,
  where: { className: 'AIML-3-A' },
});

// ❌ BAD: N+1 queries, fetching all fields
const students = await db.student.findMany();
for (const s of students) {
  s.marks = await db.mark.findMany({ where: { studentId: s.id } });
}
```

#### D. Rate Limiting
```typescript
// backend/middleware/rateLimit.ts (NEW)
import rateLimit from 'express-rate-limit';

export const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,    // 15 minutes
  max: 1000,                    // 1000 requests per IP
  message: 'Too many requests',
});

export const authLimiter = rateLimit({
  windowMs: 60 * 60 * 1000,    // 1 hour
  max: 10,                      // 10 login attempts per IP
});

app.use('/api/', apiLimiter);
app.use('/api/auth/login', authLimiter);
```

#### E. Job Queue (Background Processing)
For heavy operations (bulk imports, report generation):

```typescript
// backend/src/jobs/bulkImport.ts (NEW)
import Bull from 'bull';

const importQueue = new Bull('bulk-import', {
  redis: { url: env.redisUrl }
});

importQueue.process(async (job) => {
  const { fileBuffer, userId } = job.data;
  const result = await parseAndImportExcel(fileBuffer);
  
  // Notify user via email or dashboard
  await notificationService.send(userId, {
    type: 'IMPORT_COMPLETE',
    data: result
  });
  
  return result;
});

// Queue a bulk import
export async function queueBulkImport(fileBuffer, userId) {
  await importQueue.add({ fileBuffer, userId }, {
    removeOnComplete: true,
    removeOnFail: false
  });
}
```

---

## 3. Frontend Layer

### Current Architecture
- **Framework:** React 18.3 + Vite
- **State Management:** React Context + async hooks
- **Pattern:** Component-based, server state sync

### Scalability Optimizations

#### A. Code Splitting
```typescript
// frontend/src/router/routes.tsx
import { lazy, Suspense } from 'react';

const MarksPage = lazy(() => import('@/pages/MarksPage'));
const AttendancePage = lazy(() => import('@/pages/AttendancePage'));

<Suspense fallback={<Spinner />}>
  <MarksPage />
</Suspense>
```

#### B. Virtual Scrolling (for large lists)
```typescript
// For 10k+ students
import { FixedSizeList } from 'react-window';

<FixedSizeList
  height={600}
  itemCount={students.length}
  itemSize={50}
  width="100%"
>
  {({ index, style }) => (
    <div style={style}>
      {students[index].name}
    </div>
  )}
</FixedSizeList>
```

#### C. Service Worker for Offline Support
```typescript
// frontend/src/serviceWorker.ts (NEW)
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js');
}

// Cache API responses for offline access
const cacheName = 'student-tracker-v1';
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request)
      .then(response => response || fetch(event.request))
      .then(response => {
        caches.open(cacheName).then(cache => {
          cache.put(event.request, response.clone());
        });
        return response;
      })
  );
});
```

#### D. Pagination (Don't load all data at once)
```typescript
const [page, setPage] = useState(1);
const pageSize = 50;

const students = useAsync(
  () => studentsService.list({ 
    skip: (page - 1) * pageSize,
    take: pageSize
  }),
  [page]
);
```

---

## 4. File Upload Optimization

### Current: Excel Imports
Already optimized with:
- ✅ Multer memory storage (no disk I/O)
- ✅ File size limit (10MB)
- ✅ Validation before processing
- ✅ Streaming Excel parser (xlsx)

### Future Improvements:
```typescript
// backend/src/services/bulkImportV2.ts (FUTURE)
// Process large files in chunks instead of all at once
import { ReadStream } from 'fs';
import { Readable } from 'stream';

export async function importExcelStream(fileStream: ReadStream) {
  const workbook = XLSX.read(fileStream, { sheet: 0 });
  const chunkSize = 1000; // Process 1000 rows at a time
  
  for (let i = 0; i < totalRows; i += chunkSize) {
    const chunk = rows.slice(i, i + chunkSize);
    await processBatchAsync(chunk);
    
    // Allow other requests to process
    await new Promise(resolve => setImmediate(resolve));
  }
}
```

---

## 5. Infrastructure Recommendations

### For 1,000 - 10,000 Students
**Current Setup (Vercel):**
- ✅ Vercel Postgres (auto-scales)
- ✅ Vercel edge functions (auto-scales)
- ✅ Minimal DevOps required

### For 10,000 - 100,000 Students
**Recommended Stack:**
```
├── Frontend: Vercel (CDN + Edge)
├── Backend: Docker on Kubernetes (auto-scaling)
├── Database: PostgreSQL with read replicas
├── Cache: Redis cluster
├── Queue: Bull (job processing)
└── Storage: S3 (file uploads)
```

### For 100,000+ Students
**Enterprise Stack:**
```
├── Frontend: Multi-region CDN (Cloudflare/Fastly)
├── Backend: Kubernetes (multi-region)
├── Database: PostgreSQL + sharding by school
├── Cache: Redis cluster (partitioned)
├── Queue: RabbitMQ / Kafka
├── Search: Elasticsearch (for student search)
├── Storage: S3 / multi-region replication
└── Monitoring: Prometheus + Grafana
```

---

## 6. Monitoring & Performance

### Key Metrics
```typescript
// backend/src/middleware/metrics.ts (NEW)
export function metricsMiddleware(req, res, next) {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    
    // Log to monitoring service
    monitor.record({
      endpoint: req.path,
      method: req.method,
      statusCode: res.statusCode,
      duration,
      memoryUsage: process.memoryUsage().heapUsed,
    });
  });
  
  next();
}
```

**Monitor These:**
- API response times (target: <200ms)
- Database query times (target: <50ms)
- Cache hit rate (target: >80%)
- Error rate (target: <0.1%)
- Memory usage (target: <500MB)
- CPU usage (target: <70%)

---

## 7. Deployment Checklist

- [ ] Enable HTTPS/TLS everywhere
- [ ] Set up database backups (daily)
- [ ] Configure rate limiting
- [ ] Enable CORS for allowed domains only
- [ ] Use environment variables for secrets
- [ ] Set up error tracking (Sentry)
- [ ] Enable database connection pooling
- [ ] Configure Redis for caching
- [ ] Set up CDN for static assets
- [ ] Enable gzip compression
- [ ] Set security headers (HSTS, CSP, X-Frame-Options)
- [ ] Regular dependency updates
- [ ] Load testing (k6, JMeter)
- [ ] Database query analysis
- [ ] Set up alerting for system failures

---

## 8. Scaling Commands

### Production Deployment (Vercel)
```bash
# Deploy both frontend and backend
vercel deploy --prod

# View logs
vercel logs [project-name] --prod
```

### Database Scaling (PostgreSQL)
```sql
-- Create index on frequently queried columns
CREATE INDEX idx_student_usn ON student(usn);
CREATE INDEX idx_marks_student ON mark(studentId);
CREATE INDEX idx_attendance_date ON attendance(date);

-- Run VACUUM to optimize table
VACUUM ANALYZE;
```

### Cache Warm-up
```typescript
export async function warmupCache() {
  // Preload frequently accessed data
  const students = await db.student.findMany();
  for (const s of students) {
    await cache.setStudentMarks(s.id);
  }
}
```

---

## Summary

| Metric | Current | 10k Students | 100k Students |
|--------|---------|--------------|---------------|
| DB Type | SQLite | PostgreSQL | Sharded PostgreSQL |
| Cache | None | Redis | Redis Cluster |
| API Instances | 1 | 3-5 | 10-20+ |
| Load Balancer | None | Built-in | HAProxy/nginx |
| Queue System | None | Bull | RabbitMQ |
| Storage | Local | S3 | Multi-region S3 |
| Cost/Month | Free | $50-200 | $500-2000+ |

This architecture can scale from a single classroom to an entire nation's schools without major refactoring.
