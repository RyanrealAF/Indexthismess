# Atlas Academy - Cloudflare Deployment Guide

Complete setup instructions for deploying the forensic intelligence indexing system to Cloudflare's edge network.

---

## Prerequisites

1. **Node.js 18+**
   ```bash
   node --version  # Should be v18.0.0 or higher
   ```

2. **Wrangler CLI**
   ```bash
   npm install -g wrangler
   wrangler --version
   ```

3. **Cloudflare Account**
   - Sign up at https://dash.cloudflare.com
   - Get your Account ID from dashboard

4. **Wrangler Authentication**
   ```bash
   wrangler login
   ```

---

## Quick Deploy (Automated)

```bash
chmod +x deploy.sh
./deploy.sh
```

This script will:
- Create D1 database
- Apply schema
- Create Vectorize index
- Create KV namespace
- Sync data
- Build Next.js app
- Deploy to Pages

---

## Manual Deployment (Step-by-Step)

### Step 1: Install Dependencies

```bash
npm install
```

### Step 2: Create D1 Database

```bash
wrangler d1 create atlas-academy-db
```

Copy the `database_id` from output and update `wrangler.toml`:
```toml
database_id = "your-actual-database-id-here"
```

### Step 3: Apply Database Schema

```bash
# Local (for development)
wrangler d1 execute atlas-academy-db --file=./schema.sql --local

# Remote (production)
wrangler d1 execute atlas-academy-db --file=./schema.sql --remote
```

### Step 4: Create Vectorize Index

```bash
wrangler vectorize create atlas-embeddings \
  --dimensions=768 \
  --metric=cosine \
  --description="Atlas Academy semantic search"
```

### Step 5: Create KV Namespace

```bash
wrangler kv:namespace create CACHE
```

Update `wrangler.toml` with the returned namespace ID:
```toml
id = "your-kv-namespace-id"
```

### Step 6: Sync Data to D1

```bash
python3 scripts/sync_documents.py
```

Verify:
```bash
wrangler d1 execute atlas-academy-db \
  --command "SELECT COUNT(*) FROM content_index"
```

Should return: `14` documents

### Step 7: Build Next.js Application

```bash
npm run build
```

This creates static export in `out/` directory.

### Step 8: Deploy to Cloudflare Pages

```bash
wrangler pages deploy out --project-name=atlas-academy
```

Your site will be live at:
```
https://atlas-academy.pages.dev
```

---

## Configuration

### Environment Variables

Add to Cloudflare Pages dashboard (Settings > Environment variables):

```
ENVIRONMENT=production
```

### Bindings Verification

Test that bindings work:

```bash
wrangler pages dev out
```

Visit `http://localhost:8788` and test API routes:
- `/api/search?q=gaslighting`
- `/api/decode` (POST with JSON body)

---

## Database Management

### View All Documents

```bash
wrangler d1 execute atlas-academy-db \
  --command "SELECT id, title, intel_type FROM content_index"
```

### Check Threat Models

```bash
wrangler d1 execute atlas-academy-db \
  --command "SELECT document_id, threat_model FROM document_threat_models"
```

### Query Entities

```bash
wrangler d1 execute atlas-academy-db \
  --command "SELECT name, role FROM entity_graph"
```

### Full-Text Search Test

```bash
wrangler d1 execute atlas-academy-db \
  --command "SELECT term FROM concepts_fts WHERE concepts_fts MATCH 'gaslighting'"
```

---

## Vectorize Setup (Semantic Search)

### Generate Embeddings

Create `scripts/batch_embed.py`:

```python
#!/usr/bin/env python3
import subprocess
import json

# Get all documents
docs = subprocess.check_output([
    'wrangler', 'd1', 'execute', 'atlas-academy-db',
    '--command', 'SELECT id, title, summary FROM content_index'
]).decode()

# TODO: Generate embeddings and insert into Vectorize
# See Cloudflare docs: https://developers.cloudflare.com/vectorize/
```

### Insert Vectors

```bash
# Via API or wrangler
wrangler vectorize insert atlas-embeddings \
  --file=embeddings.ndjson
```

---

## API Testing

### Semantic Search

```bash
curl -X POST https://atlas-academy.pages.dev/api/search \
  -H "Content-Type: application/json" \
  -d '{"query": "how to detect gaslighting", "limit": 5}'
```

### Tactical Decoding

```bash
curl -X POST https://atlas-academy.pages.dev/api/decode \
  -H "Content-Type: application/json" \
  -d '{
    "text": "You are too sensitive, I was just joking",
    "context": "Response to confrontation about hurtful comment"
  }'
```

### Keyword Search

```bash
curl "https://atlas-academy.pages.dev/api/search?q=breadcrumb&intel_type=COUNTER_DOCTRINE"
```

---

## Custom Domain Setup

1. Go to Cloudflare dashboard
2. Navigate to Pages > atlas-academy > Custom domains
3. Add `buildwhilebleeding.com` (or subdomain)
4. Follow DNS configuration steps
5. Enable "Always Use HTTPS"

---

## Development Workflow

### Local Development

```bash
# Run Next.js dev server
npm run dev

# Run with Cloudflare bindings (after build)
npm run pages:dev
```

### Database Migrations

When updating schema:

```bash
# 1. Modify schema.sql
# 2. Apply locally first
wrangler d1 execute atlas-academy-db --file=./schema.sql --local

# 3. Test thoroughly
# 4. Apply to production
wrangler d1 execute atlas-academy-db --file=./schema.sql --remote
```

### Sync Updates

After modifying TypeScript registries:

```bash
python3 scripts/sync_documents.py
# Then run sync scripts for entities, concepts, timeline
```

---

## Monitoring

### View Logs

```bash
wrangler pages deployment tail
```

### Analytics

Check Cloudflare dashboard:
- Pages > atlas-academy > Analytics
- D1 > atlas-academy-db > Metrics
- Vectorize > atlas-embeddings > Usage

---

## Troubleshooting

### Build Fails

```bash
# Clear cache
rm -rf .next out node_modules
npm install
npm run build
```

### D1 Connection Issues

```bash
# Verify database exists
wrangler d1 list

# Check bindings in wrangler.toml
wrangler pages project list
```

### API Routes Return 404

Ensure you're using static export:
```javascript
// next.config.mjs
output: 'export'
```

API routes won't work in static export. Use Cloudflare Workers instead.

### Vectorize Not Working

```bash
# Check index exists
wrangler vectorize list

# Verify dimensions match model
# bge-base-en-v1.5 uses 768 dimensions
```

---

## Cost Estimates

With Cloudflare's free tier:
- **Pages**: 500 builds/month, unlimited requests
- **D1**: 5GB storage, 5M reads/day
- **Vectorize**: 30M queries/month, 5M dimensions
- **Workers AI**: 10,000 neurons/day

Current project should stay **100% free** at moderate usage.

---

## Next Steps

1. **Complete Data Sync**
   ```bash
   python3 scripts/sync_entities.py
   python3 scripts/sync_concepts.py
   python3 scripts/sync_timeline.py
   ```

2. **Generate Embeddings**
   ```bash
   python3 scripts/batch_embed.py
   ```

3. **Build Additional Pages**
   - `/documents` - Full registry browser
   - `/entities` - Interactive graph
   - `/concepts` - Tactical glossary
   - `/timeline` - Chronological view

4. **Enable Analytics**
   - Set up Web Analytics in Cloudflare
   - Track search queries
   - Monitor API usage

---

## Security Checklist

- [ ] Enable Cloudflare Access for admin routes
- [ ] Set up rate limiting on API endpoints
- [ ] Configure CSP headers
- [ ] Enable DDoS protection
- [ ] Set up alert notifications
- [ ] Review D1 permissions

---

## Support

- **Cloudflare Docs**: https://developers.cloudflare.com
- **Next.js + Cloudflare**: https://github.com/cloudflare/next-on-pages
- **Project Repo**: [Add your GitHub URL]

---

**The fortress is built. Now make it unbreachable.**
