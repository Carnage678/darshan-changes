# Vercel Deployment & Database Migration Guide

## Quick Start for Vercel Deployment

### 1. Update Local Development (Optional)
Your project now uses PostgreSQL instead of SQLite for better Vercel compatibility.

**Option A: Keep local SQLite for quick dev**
```bash
# Use the old database URL in backend/.env
DATABASE_URL="file:./dev.db"
```

**Option B: Use local PostgreSQL (recommended)**
```bash
# Install PostgreSQL locally and create a database:
# createdb student_tracker

# Then update DATABASE_URL in backend/.env:
DATABASE_URL="postgresql://postgres:password@localhost:5432/student_tracker"

# Run migrations:
cd backend
npm run db:push
npm run db:seed
```

### 2. Set Up Vercel Postgres (Recommended)

Go to https://vercel.com/docs/storage/vercel-postgres

```bash
# Install Vercel CLI
npm install -g vercel

# Link your project
vercel link

# Add PostgreSQL storage
vercel env pull
```

This will auto-populate `DATABASE_URL` in your Vercel environment.

### 3. Set Environment Variables in Vercel

In Vercel Project Settings > Environment Variables, add:
- `DATABASE_URL`: Your PostgreSQL connection string (auto-set if using Vercel Postgres)
- `JWT_SECRET`: A strong random secret
- `CORS_ORIGIN`: Your frontend URL (e.g., https://app.example.com)
- `GEMINI_API_KEY`: Your Gemini API key
- `NODE_ENV`: `production`

### 4. Deploy

```bash
vercel deploy --prod
```

The backend and frontend will deploy automatically.

### 5. Troubleshooting

**"DATABASE_URL not set"**
- Check Vercel Project Settings > Environment Variables
- Make sure the DATABASE_URL is visible to the backend function

**"Connection timeout"**
- Verify your PostgreSQL database is accessible
- Check firewall/security group settings if using external database

**Prisma migration errors**
- These run automatically during build
- Check build logs in Vercel dashboard

## Local Database Migration

If you had SQLite data and want to migrate to PostgreSQL:

```bash
# Backup SQLite data first (optional)
cp backend/dev.db backend/dev.db.backup

# With new PostgreSQL connection string in .env:
cd backend
npm run db:push           # Create all tables in PostgreSQL
npm run db:seed           # Seed sample data (or import your own)
```

## Files Changed

- `backend/prisma/schema.prisma` - Changed from SQLite to PostgreSQL
- `backend/.env.production` - Production environment template
- `frontend/.env.production` - Frontend production config
- `backend/.env.local` - Local dev options (SQLite or PostgreSQL)
- `vercel.json` - Vercel build configuration

---

**Questions?** Check:
- Vercel docs: https://vercel.com/docs
- Prisma docs: https://www.prisma.io/docs
