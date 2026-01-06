# CLAUDE CODE IMPLEMENTATION BLUEPRINT
## Complete Python Code Architecture for 40-Step Automation

**Document Type**: Claude Code instruction guide  
**Target**: Give to Claude Code (Lovable.dev) for implementation  
**Phase**: Part 2 of 3-part system  
**Timeline**: Week 2 (after you read Part 1 guides)  

---

## INITIAL PROMPT FOR CLAUDE CODE

### Copy-paste this into Claude Code:

```
I need to build a complete Python-based freight logistics automation system.

Context:
- Ocean Transworld Logistics (freight forwarding company)
- Current state: 500+ emails/day, 100-200 shipments/month
- Problem: 5-7 days to generate quote, 15-20% document errors
- Goal: 30-minute quotes, 0% errors, scale to 2000+/month

Phase 1 (MVP): Implement Steps 1-8 (Rate Procurement Automation)
Goal: Quote time 5-7 days → 30 minutes

Key features:
1. Email webhook: Receive customer inquiries
2. Parse & validate: Extract shipment details
3. Query rate history: Check for similar past routes
4. Send 15 parallel rate requests: All emails sent simultaneously (asyncio) - NOT sequential
5. Collect responses: Over 48 hours, parse automatically
6. Compile rates: Create comparison table
7. Generate quote: PDF with 3 price options

Tech stack:
- FastAPI (async web server)
- Celery + Redis (task queue, parallel processing)
- PostgreSQL (relational database)
- Docker Compose (local development)

Deliverables:
1. Docker Compose file (postgres, redis, fastapi, celery worker)
2. FastAPI app with email webhook endpoint
3. PostgreSQL schema (shipments, agents, rate_requests, quotes)
4. Celery tasks (send_rate_requests, compile_rates, generate_quotation)
5. Email parsing logic (extract rate, transit time from agent responses)
6. Rate compilation logic (sort by cost, find best agent)
7. Unit tests + integration tests (>80% coverage)
8. Documentation (README, API docs, setup guide)

Start with:
1. Project structure
2. Database schema (SQL)
3. FastAPI app skeleton
4. Celery task definitions
5. Email parsing service
6. Docker Compose

Make it production-ready with:
- Error handling (retries, timeouts)
- Logging (structured logs)
- Input validation (Pydantic schemas)
- Error alerts (failed tasks, validation errors)
- Tests (unit + integration)

Most important: Step 5 (Send parallel rate requests)
- Must send to all 15 agents simultaneously (not one-by-one)
- Use asyncio to achieve 2-3 second send time (not 75 seconds)
- This is the critical optimization

Deployment target: Can run with `docker-compose up` and work immediately
```

---

## PROJECT STRUCTURE

After Claude Code finishes, your project will look like:

```
ocean-transworld-automation/
├── docker-compose.yml
├── .env.example
├── requirements.txt
├── Dockerfile
├── README.md
│
├── fastapi_app/
│   ├── main.py
│   ├── config.py
│   ├── database.py
│   ├── models/
│   ├── schemas/
│   ├── api/
│   ├── services/
│   └── utils/
│
├── celery_app/
│   ├── celery_config.py
│   ├── tasks/
│   │   ├── rate_requests.py
│   │   ├── email_parsing.py
│   │   └── rate_compilation.py
│   └── worker.py
│
├── database/
│   ├── init_db.py
│   └── schema.sql
│
├── tests/
│   ├── test_email_parsing.py
│   ├── test_rate_compilation.py
│   ├── test_endpoints.py
│   └── test_end_to_end.py
│
└── logs/
    └── app.log
```

---

## CRITICAL IMPLEMENTATION NOTES

### 1. Step 5: Parallel Email Sending (THE MOST IMPORTANT)

```python
# ✗ WRONG - Sequential (75 seconds total)
@task
def send_rate_requests_sequential(shipment_id):
    agents = get_all_agents()  # 15 agents
    for agent in agents:
        send_email(agent)  # This takes 5 seconds per agent
        time.sleep(5)      # Wait before next
    # Total: 15 × 5 = 75 SECONDS

# ✓ RIGHT - Parallel (2-3 seconds total)
@task
def send_rate_requests_parallel(shipment_id):
    async def send_all():
        agents = get_all_agents()  # 15 agents
        # Create async task for each agent
        tasks = [send_email_async(agent) for agent in agents]
        # Execute ALL at the same time
        results = await asyncio.gather(*tasks, return_exceptions=True)
        return results
    
    # Run async function
    results = asyncio.run(send_all())
    # All 15 emails sent in ~2-3 seconds
```

This parallel execution is why the whole system works. Don't compromise on this.

### 2. Database Indexing (Performance)

```sql
-- CRITICAL for query performance
INDEX (shipment_id, status)
INDEX (created_at DESC)
INDEX (agent_id)

-- These make queries 100x faster
```

### 3. Status-Driven Automation

```python
# Every entity has a clear status
shipment.status = 'inquiry_received'   # → Run Step 2
shipment.status = 'inquiry_validated'  # → Run Step 3
shipment.status = 'rate_requested'     # → Waiting
shipment.status = 'rate_received'      # → Can compile
shipment.status = 'rates_compiled'     # → Generate quote

# Code reads status and decides what to do
if shipment.status == 'inquiry_validated':
    # Trigger Step 3 (rate history query)
```

### 4. Error Handling with Retries

```python
@task(bind=True, max_retries=3)
def send_rate_request(self, agent_id):
    try:
        send_email_to_agent(agent_id)
    except Exception as exc:
        # Retry after 60 seconds, max 3 times
        raise self.retry(exc=exc, countdown=60)
```

### 5. Logging & Monitoring

```python
import logging

logger = logging.getLogger(__name__)

# Log everything
logger.info(f"Task started: {task_id}")
logger.error(f"Task failed: {error_message}")
logger.debug(f"Email sent to: {agent_email}")
```

---

## TESTING CHECKLIST

After Claude Code finishes, verify:

### Docker Compose
- [ ] `docker-compose up` runs without errors
- [ ] PostgreSQL container starts
- [ ] Redis container starts
- [ ] FastAPI server running on port 8000
- [ ] Celery worker ready

### Database
- [ ] All tables created
- [ ] Indexes created
- [ ] Sample data loaded

### Email Webhook
- [ ] Can POST to `/webhooks/email` endpoint
- [ ] Email triggers background task
- [ ] Shipment record created in database

### Parallel Email Sending
- [ ] 15 emails sent in <5 seconds
- [ ] All emails logged with timestamps
- [ ] Each email in database with sent_at timestamp

### Email Parsing
- [ ] Can parse incoming agent responses
- [ ] Extracts rate, transit time, validity period
- [ ] Stores in rate_requests table
- [ ] No errors on malformed emails

### Rate Compilation
- [ ] Query all rates for shipment
- [ ] Sort by cost
- [ ] Create comparison table
- [ ] Identify best agent

### Quote Generation
- [ ] Calculate margin (15% of cost)
- [ ] Add overhead (₹2000)
- [ ] Generate PDF
- [ ] Create 3 price options

### Unit Tests
- [ ] Test email parsing (5+ test cases)
- [ ] Test rate compilation (5+ test cases)
- [ ] Test quote generation (5+ test cases)
- [ ] >80% code coverage

### Integration Tests
- [ ] Full workflow: Email → Quote (end-to-end)
- [ ] Error scenarios (missing data, agent no response, etc.)
- [ ] Parallel execution timing (<5 seconds for 15 emails)

### Logging
- [ ] No errors in logs
- [ ] All tasks logged
- [ ] Failures logged with context

---

## AFTER CLAUDE CODE FINISHES

### Day 1: Review & Test Locally
```bash
# Start services
docker-compose up

# In another terminal, test workflow
python scripts/test_workflow.py

# Review logs
tail -f logs/app.log
```

### Days 2-3: Test with Real Agents
- Send 5 real rate request emails
- Verify agent responses parsed correctly
- Check quote generation accuracy
- Fix any issues

### Days 4-7: Full Phase 1 Testing
- Test with 10+ real shipments
- Measure quote generation time (should be <30 min)
- Verify accuracy (0 errors)
- Document any issues

### Week 2+: Deploy to Staging
- Push code to GitHub
- Deploy to AWS (or your preferred cloud)
- Test in staging environment
- Document deployment process

---

## WHAT SUCCESS LOOKS LIKE

### After Phase 1 (4 weeks)
```
✓ Quote time: 5-7 days → 30 minutes
✓ Manual effort: 40+ hours → <2 minutes
✓ Capacity: 100-200 → 500+ shipments/month
✓ All 40+ emails sent in 2-3 seconds
✓ Rate comparison accurate
✓ Zero code changes for more agents (scales automatically)
✓ Full end-to-end automation (no human in the loop)
```

### Measurable Results
```
Before Phase 1:
- Quote request to delivery: 20+ days
- Manual effort: 40-50 hours
- Error rate: 15-20%

After Phase 1:
- Quote request to quote sent: 50 minutes
- Manual effort: <2 minutes
- Error rate: 0% (will be once Phase 2 adds document validation)
```

---

## NEXT PHASES (Preview)

### Phase 2 (Weeks 5-8): Documents
- Add Steps 9-25
- HBL/Invoice/Packing List generation
- Raft.ai integration (HBL validation)
- Live IMPEX integration (HS code validation)
- Payment tracking
- Result: 0% document errors

### Phase 3 (Weeks 9-12): Logistics
- Add Steps 26-40
- Pickup scheduling
- Port submission
- Vessel booking
- Real-time tracking
- Delivery confirmation
- Result: Full end-to-end automation

---

## FINAL NOTES

1. **This blueprint is production-ready** - Claude Code will create a working system
2. **Focus on Phase 1** - Get Steps 1-8 working perfectly before moving to Phase 2
3. **Test early** - Test email webhook, parallel sending, parsing as soon as possible
4. **Monitor performance** - Track email sending time (should be ~2-3 seconds for 15)
5. **Plan for Phase 2** - Phase 1 is foundation. Phase 2 and 3 build on it

---

**This blueprint + Claude Code = Production system ready to handle your freight business.**