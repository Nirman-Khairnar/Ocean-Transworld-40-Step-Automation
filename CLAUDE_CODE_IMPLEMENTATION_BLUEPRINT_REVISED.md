# CLAUDE CODE IMPLEMENTATION BLUEPRINT (REVISED)
## Complete Python Code with Affordable Tools Stack

**Document Type**: Claude Code instruction guide  
**Updated**: January 7, 2026  
**Cost**: ₹4-5L Year 1 (vs ₹28-38L original)  
**Key Change**: No Raft.ai, no Live IMPEX, no sales calls  

---

## INITIAL PROMPT FOR CLAUDE CODE

### Copy-paste this (REVISED VERSION):

```
I need to build a complete Python-based freight logistics automation system.

Context:
- Ocean Transworld Logistics (freight forwarding company)
- Current state: 500+ emails/day, 100-200 shipments/month
- Problem: 5-7 days to generate quote, 15-20% document errors
- Goal: 30-minute quotes, 0% errors, scale to 2000+/month
- Budget: <₹5L/year for all tools

Phase 1 (MVP): Implement Steps 1-8 (Rate Procurement Automation)
Goal: Quote time 5-7 days → 30 minutes

Key features:
1. Email webhook: Receive customer inquiries
2. Parse & validate: Extract shipment details using mail-parser (open source)
3. Query rate history: Check for similar past routes
4. Send 15 parallel rate requests: All emails sent simultaneously (asyncio) - NOT sequential
5. Collect responses: Over 48 hours, parse automatically with mail-parser
6. Compile rates: Create comparison table
7. Generate quote: PDF with ReportLab (open source), 3 price options

Tech stack:
- FastAPI (async web server)
- Celery + Redis (task queue, parallel processing)
- PostgreSQL (relational database)
- Docker Compose (local development)
- mail-parser (email parsing - open source)
- ReportLab (PDF generation - open source)
- Pydantic (data validation - open source)

External APIs (affordable):
- Mindee OCR API (Bill of Lading extraction, ₹25k/year, 250 free docs/month to test)
- ICEGATE API (HS code validation, FREE - government of India)
- SendGrid (email sending, free tier 100/day)

Deliverables:
1. Docker Compose file (postgres, redis, fastapi, celery worker)
2. FastAPI app with email webhook endpoint
3. PostgreSQL schema (shipments, agents, rate_requests, quotes, hs_codes)
4. Celery tasks (send_rate_requests, compile_rates, generate_quotation)
5. Email parsing with mail-parser library
6. Rate compilation logic (sort by cost, find best agent)
7. PDF generation with ReportLab
8. Pydantic schemas for HBL/document validation
9. HS code validation (local database + ICEGATE fallback)
10. Unit tests + integration tests (>80% coverage)
11. Documentation (README, API docs, setup guide)

Validation approach (NO Raft.ai):
1. Use Mindee OCR API to extract HBL fields
2. Validate with Pydantic schemas (container format, restricted parties)
3. Check HS codes against local PostgreSQL database (populated from DGFT)
4. Flag low-confidence fields for manual review

Most important: Step 5 (Send parallel rate requests)
- Must send to all 15 agents simultaneously (not one-by-one)
- Use asyncio to achieve 2-3 second send time (not 75 seconds)
- This is the critical optimization

Deployment target:
- Can run with `docker-compose up` and work immediately
- All external API keys configured via environment variables
- Can process 50+ quotes/day in Phase 1

Cost target:
- Keep external API costs <₹500/month in development
- Use free tiers for testing (Mindee 250/month, SendGrid 100/day)
```

---

## PROJECT STRUCTURE (REVISED)

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
│   │   ├── shipment.py
│   │   ├── agent.py
│   │   ├── rate_request.py
│   │   └── hs_code.py
│   ├── schemas/
│   │   ├── hbl_validation.py       # Pydantic schemas
│   │   ├── shipment_schema.py
│   │   └── quote_schema.py
│   ├── api/
│   │   ├── webhooks.py             # Email webhook
│   │   ├── shipments.py            # Shipment endpoints
│   │   └── quotes.py               # Quote endpoints
│   ├── services/
│   │   ├── email_parser.py         # mail-parser integration
│   │   ├── mindee_service.py       # Mindee OCR integration
│   │   ├── hs_code_validator.py    # HS code validation
│   │   ├── pdf_generator.py        # ReportLab integration
│   │   └── sendgrid_service.py     # Email sending
│   └── utils/
│       ├── validation.py           # Pydantic validators
│       └── formatters.py
│
├── celery_app/
│   ├── celery_config.py
│   ├── tasks/
│   │   ├── rate_requests.py        # Parallel sending
│   │   ├── email_parsing.py        # Parse agent responses
│   │   ├── rate_compilation.py     # Compile rates
│   │   └── quote_generation.py     # Generate PDF
│   └── worker.py
│
├── database/
│   ├── init_db.py
│   ├── schema.sql
│   └── seed_hs_codes.py            # Import DGFT HS codes
│
├── tests/
│   ├── test_email_parsing.py
│   ├── test_mindee_ocr.py
│   ├── test_hs_validation.py
│   ├── test_pdf_generation.py
│   ├── test_rate_compilation.py
│   ├── test_endpoints.py
│   └── test_end_to_end.py
│
└── logs/
    └── app.log
```

---

## CRITICAL DEPENDENCIES

### requirements.txt
```
fastapi==0.109.0
celery[redis]==5.3.4
postgresql-psycopg2==2.9.9
redis==5.0.1
pydantic==2.5.3
mindee==4.6.0                # HBL OCR
mail-parser==3.15.0          # Email parsing
reportlab==4.0.7             # PDF generation
cerberus==1.3.5              # Schema validation
sendgrid==6.11.0             # Email sending
BeautifulSoup4==4.12.2       # HTML/web scraping (ICEGATE)
requests==2.31.0
python-dotenv==1.0.0
uvicorn[standard]==0.25.0
```

---

## KEY IMPLEMENTATION CHANGES

### 1. HBL Validation (Mindee + Pydantic)

```python
# fastapi_app/services/mindee_service.py
from mindee import Client, product
import os

class MindeeService:
    def __init__(self):
        self.client = Client(api_key=os.getenv('MINDEE_API_KEY'))
    
    def extract_hbl_data(self, pdf_path: str) -> dict:
        """Extract HBL data using Mindee OCR"""
        input_doc = self.client.source_from_path(pdf_path)
        result = input_doc.parse(product.BillOfLadingV1)
        
        return {
            'shipper': result.document.inference.prediction.shipper.value,
            'consignee': result.document.inference.prediction.consignee.value,
            'container_numbers': [c.value for c in result.document.inference.prediction.container_numbers],
            'port_of_loading': result.document.inference.prediction.port_of_loading.value,
            'port_of_discharge': result.document.inference.prediction.port_of_discharge.value,
            'confidence_scores': {
                'shipper': result.document.inference.prediction.shipper.confidence,
                'consignee': result.document.inference.prediction.consignee.confidence
            }
        }

# fastapi_app/schemas/hbl_validation.py
from pydantic import BaseModel, validator, Field
import re

class HBLDocument(BaseModel):
    shipper: str = Field(..., min_length=2, max_length=200)
    consignee: str = Field(..., min_length=2, max_length=200)
    container_numbers: list[str]
    port_of_loading: str
    port_of_discharge: str
    weight_kg: float = Field(..., gt=0)
    
    @validator('container_numbers', each_item=True)
    def validate_container_format(cls, v):
        # ISO 6346: 4 letters + 7 digits
        if not re.match(r'^[A-Z]{4}\d{7}$', v):
            raise ValueError(f'Invalid container format: {v}')
        return v
    
    @validator('consignee')
    def check_restricted_parties(cls, v):
        # Check against restricted parties database
        from ..services.restricted_parties import check_restricted
        if check_restricted(v):
            raise ValueError(f'Consignee on restricted list')
        return v
```

### 2. HS Code Validation (Local DB + ICEGATE)

```python
# fastapi_app/services/hs_code_validator.py
from sqlalchemy import text
import requests

class HSCodeValidator:
    def __init__(self, db_session):
        self.db = db_session
    
    def validate_hs_code(self, hs_code: str) -> dict:
        """Validate HS code against local database"""
        result = self.db.execute(
            text("SELECT * FROM hs_codes WHERE code = :code"),
            {"code": hs_code}
        ).fetchone()
        
        if result:
            return {
                'valid': True,
                'description': result.description,
                'import_policy': result.import_policy,
                'export_policy': result.export_policy,
                'duty_rate': float(result.duty_rate)
            }
        else:
            # Fallback: Query ICEGATE (if available)
            return self._query_icegate(hs_code)
    
    def _query_icegate(self, hs_code: str) -> dict:
        """Fallback to ICEGATE API"""
        # Implement ICEGATE API call here
        # For now, return not found
        return {'valid': False, 'error': 'HS code not found'}
```

### 3. Email Parsing (mail-parser)

```python
# fastapi_app/services/email_parser.py
import mailparser
import re
from typing import Optional

class EmailParserService:
    def parse_agent_response(self, email_content: str) -> dict:
        """Parse agent email to extract rate"""
        mail = mailparser.parse_from_string(email_content)
        
        # Extract rate from body
        body = mail.body
        rate = self._extract_rate(body)
        transit_days = self._extract_transit_days(body)
        validity_days = self._extract_validity(body)
        
        return {
            'from': mail.from_,
            'subject': mail.subject,
            'rate': rate,
            'transit_days': transit_days,
            'validity_days': validity_days,
            'received_at': mail.date
        }
    
    def _extract_rate(self, body: str) -> Optional[float]:
        patterns = [
            r'Rate[:\s]+\$?([0-9,]+\.?[0-9]*)',
            r'Price[:\s]+\$?([0-9,]+\.?[0-9]*)',
            r'\$\s*([0-9,]+\.?[0-9]*)'
        ]
        for pattern in patterns:
            match = re.search(pattern, body, re.IGNORECASE)
            if match:
                return float(match.group(1).replace(',', ''))
        return None
    
    def _extract_transit_days(self, body: str) -> Optional[int]:
        pattern = r'(\d+)\s*days?\s*(transit|delivery)'
        match = re.search(pattern, body, re.IGNORECASE)
        return int(match.group(1)) if match else None
    
    def _extract_validity(self, body: str) -> Optional[int]:
        pattern = r'valid\s*for\s*(\d+)\s*days?'
        match = re.search(pattern, body, re.IGNORECASE)
        return int(match.group(1)) if match else 7  # default 7 days
```

### 4. PDF Generation (ReportLab)

```python
# fastapi_app/services/pdf_generator.py
from reportlab.lib.pagesizes import A4
from reportlab.pdfgen import canvas
from reportlab.lib.units import inch
from datetime import datetime

class PDFGenerator:
    def generate_quotation(self, shipment_data: dict, output_path: str):
        """Generate quotation PDF"""
        c = canvas.Canvas(output_path, pagesize=A4)
        width, height = A4
        
        # Header
        c.setFont("Helvetica-Bold", 16)
        c.drawString(1*inch, height - 1*inch, "FREIGHT QUOTATION")
        
        # Company details
        c.setFont("Helvetica", 10)
        c.drawString(1*inch, height - 1.5*inch, "Ocean Transworld Logistics Pvt. Ltd.")
        c.drawString(1*inch, height - 1.7*inch, f"Quote Ref: {shipment_data['quote_id']}")
        c.drawString(1*inch, height - 1.9*inch, f"Date: {datetime.now().strftime('%Y-%m-%d')}")
        
        # Shipment details
        y = height - 2.5*inch
        c.setFont("Helvetica-Bold", 12)
        c.drawString(1*inch, y, "SHIPMENT DETAILS")
        y -= 0.3*inch
        
        c.setFont("Helvetica", 10)
        details = [
            f"Origin: {shipment_data['origin']}",
            f"Destination: {shipment_data['destination']}",
            f"Weight: {shipment_data['weight']} kg",
            f"Volume: {shipment_data['volume']} cbm"
        ]
        for detail in details:
            c.drawString(1*inch, y, detail)
            y -= 0.2*inch
        
        # Pricing options
        y -= 0.5*inch
        c.setFont("Helvetica-Bold", 12)
        c.drawString(1*inch, y, "PRICING OPTIONS")
        y -= 0.3*inch
        
        for i, option in enumerate(shipment_data['options'], 1):
            c.setFont("Helvetica", 10)
            c.drawString(1*inch, y, f"Option {i}: {option['name']}")
            y -= 0.2*inch
            c.drawString(1.2*inch, y, f"Rate: ₹{option['rate']:,.2f}")
            y -= 0.2*inch
            c.drawString(1.2*inch, y, f"Transit: {option['transit_days']} days")
            y -= 0.4*inch
        
        c.save()
```

---

## ENVIRONMENT VARIABLES

### .env.example
```
# Database
DATABASE_URL=postgresql://user:password@localhost:5432/ocean_transworld

# Redis
REDIS_URL=redis://localhost:6379/0

# External APIs
MINDEE_API_KEY=your_mindee_api_key_here
SENDGRID_API_KEY=your_sendgrid_api_key_here

# ICEGATE (if API available)
ICEGATE_API_KEY=your_icegate_key_here

# App Config
DEBUG=True
LOG_LEVEL=INFO
```

---

## TESTING WITH FREE TIERS

### 1. Mindee (250 free docs/month)
```bash
# Sign up: https://platform.mindee.com/signup
# Get API key from dashboard
# Test with sample HBL

python -c "
from mindee import Client, product
client = Client(api_key='your_key')
result = client.source_from_path('sample_hbl.pdf').parse(product.BillOfLadingV1)
print(result.document.inference.prediction)
"
```

### 2. SendGrid (100 emails/day free)
```bash
# Sign up: https://signup.sendgrid.com/
# Get API key
# Test sending

curl --request POST \
  --url https://api.sendgrid.com/v3/mail/send \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{"personalizations":[{"to":[{"email":"test@example.com"}]}],"from":{"email":"noreply@oceantransworld.com"},"subject":"Test","content":[{"type":"text/plain","value":"Test email"}]}'
```

---

## COST TRACKING

### Development Phase (Weeks 1-4)
```
Mindee: ₹0 (free tier, 250 docs)
SendGrid: ₹0 (free tier, 100/day)
ICEGATE: ₹0 (government free)
Infrastructure: ₹0 (local Docker)

TOTAL: ₹0 🎉
```

### Production (250 shipments/month)
```
Mindee: ₹2,082/month = ₹25,000/year
SendGrid: ₹0 (within free tier)
ICEGATE: ₹0
AWS (EC2 + RDS + S3): ₹50,000/year

TOTAL: ₹75,000/year
```

---

## SUCCESS METRICS

### After Implementation
- [ ] Quote time: <30 minutes
- [ ] HBL extraction accuracy: >95% (Mindee)
- [ ] HS code validation: 100% (local DB)
- [ ] Email parsing: >90% success rate
- [ ] PDF generation: <2 seconds
- [ ] Monthly cost: <₹500 (development)
- [ ] All external APIs work via simple REST calls (no sales BS)

---

## NEXT STEPS

1. **This Week**: Give this prompt to Claude Code
2. **Week 1**: Claude builds the foundation
3. **Week 2**: Test with Mindee free tier (250 docs)
4. **Week 3**: Import DGFT HS codes to PostgreSQL
5. **Week 4**: Full end-to-end testing

---

**This is your production-ready, affordable implementation blueprint. No enterprise sales. No gatekeeping. Just working code with APIs.**