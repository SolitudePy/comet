# 🔧 Implementation Guide - Quick Win Features

This guide provides implementation details for the highest priority, easiest to implement features from the feature suggestions.

## Table of Contents
1. [Health Check Endpoint](#1-health-check-endpoint)
2. [Enhanced Configuration Export/Import](#2-enhanced-configuration-exportimport)
3. [Scraper Status Page](#3-scraper-status-page)
4. [Cache Statistics Endpoint](#4-cache-statistics-endpoint)
5. [Request Logging System](#5-request-logging-system)

---

## 1. Health Check Endpoint

**Complexity**: Low  
**Time Estimate**: 2-4 hours  
**Files to Modify**: `comet/api/endpoints/base.py` or new file `comet/api/endpoints/health.py`

### Implementation

```python
# comet/api/endpoints/health.py
from fastapi import APIRouter
from pydantic import BaseModel

router = APIRouter()

class HealthStatus(BaseModel):
    status: str
    database: str
    scrapers: dict[str, str]
    cache: str
    background_scraper: str
    uptime_seconds: float

@router.get("/health", tags=["System"])
async def health_check():
    """
    System health check endpoint
    Returns status of all major components
    """
    health = {
        "status": "healthy",
        "database": await check_database(),
        "scrapers": await check_scrapers(),
        "cache": await check_cache(),
        "background_scraper": await check_background_scraper(),
        "uptime_seconds": get_uptime()
    }
    
    # If any component is unhealthy, mark overall status as degraded
    if any(v != "healthy" for v in health.values() if isinstance(v, str)):
        health["status"] = "degraded"
    
    return health

async def check_database():
    """Check database connectivity"""
    try:
        await database.fetch_one("SELECT 1")
        return "healthy"
    except Exception:
        return "unhealthy"

async def check_scrapers():
    """Check scraper availability"""
    # Return status for each enabled scraper
    pass

async def check_cache():
    """Check cache system"""
    # Verify cache is accessible and within size limits
    pass

async def check_background_scraper():
    """Check background scraper status"""
    # Return whether background scraper is running
    pass

def get_uptime():
    """Get server uptime in seconds"""
    # Calculate uptime since server start
    pass
```

### Testing
```bash
curl http://localhost:8000/health
```

Expected response:
```json
{
  "status": "healthy",
  "database": "healthy",
  "scrapers": {
    "torrentio": "healthy",
    "zilean": "healthy"
  },
  "cache": "healthy",
  "background_scraper": "running",
  "uptime_seconds": 3600.5
}
```

---

## 2. Enhanced Configuration Export/Import

**Complexity**: Low  
**Time Estimate**: 3-5 hours  
**Files to Modify**: `comet/api/endpoints/admin.py`, add new template or API endpoints

### Implementation

```python
# Add to comet/api/endpoints/admin.py

@router.get("/admin/api/config/export", tags=["Admin"])
async def export_configuration(admin_session: str = Cookie(None)):
    """Export current configuration as JSON"""
    await require_admin_auth(admin_session)
    
    config = {
        "version": "1.0",
        "exported_at": time.time(),
        "settings": {
            # Export non-sensitive settings
            "addon_name": settings.ADDON_NAME,
            "scrapers": {
                "torrentio": settings.SCRAPE_TORRENTIO,
                "zilean": settings.SCRAPE_ZILEAN,
                # ... other scrapers
            },
            "cache": {
                "metadata_ttl": settings.METADATA_CACHE_TTL,
                "torrent_ttl": settings.TORRENT_CACHE_TTL,
            },
            "filters": {
                "remove_adult": settings.REMOVE_ADULT_CONTENT,
                "digital_release_filter": settings.DIGITAL_RELEASE_FILTER,
            }
            # Note: Do NOT export sensitive data like API keys
        }
    }
    
    return JSONResponse(
        content=config,
        headers={
            "Content-Disposition": f"attachment; filename=comet-config-{int(time.time())}.json"
        }
    )

@router.post("/admin/api/config/import", tags=["Admin"])
async def import_configuration(
    admin_session: str = Cookie(None),
    config_file: UploadFile = File(...)
):
    """Import configuration from JSON file"""
    await require_admin_auth(admin_session)
    
    try:
        content = await config_file.read()
        config = orjson.loads(content)
        
        # Validate configuration version
        if config.get("version") != "1.0":
            raise HTTPException(400, "Unsupported configuration version")
        
        # Apply settings (would need to implement settings update logic)
        # Note: Some settings require restart to take effect
        
        return {
            "status": "success",
            "message": "Configuration imported. Restart required for changes to take effect.",
            "settings_updated": len(config.get("settings", {}))
        }
    except Exception as e:
        raise HTTPException(400, f"Invalid configuration file: {str(e)}")
```

### Usage
1. Export: `GET /admin/api/config/export`
2. Import: `POST /admin/api/config/import` with file upload

---

## 3. Scraper Status Page

**Complexity**: Low-Medium  
**Time Estimate**: 4-6 hours  
**Files to Modify**: `comet/api/endpoints/admin.py`, add template or enhance existing admin dashboard

### Implementation

```python
# Add to comet/api/endpoints/admin.py

@router.get("/admin/api/scrapers/status", tags=["Admin"])
async def get_scraper_status(admin_session: str = Cookie(None)):
    """Get detailed status of all scrapers"""
    if settings.PUBLIC_METRICS_API or await verify_admin_session(admin_session):
        statuses = []
        
        # Check each scraper
        scrapers = [
            ("torrentio", settings.SCRAPE_TORRENTIO, settings.TORRENTIO_URL),
            ("zilean", settings.SCRAPE_ZILEAN, settings.ZILEAN_URL),
            ("mediafusion", settings.SCRAPE_MEDIAFUSION, settings.MEDIAFUSION_URL),
            ("jackett", settings.SCRAPE_JACKETT, settings.JACKETT_URL),
            ("prowlarr", settings.SCRAPE_PROWLARR, settings.PROWLARR_URL),
            # ... other scrapers
        ]
        
        for name, enabled, url in scrapers:
            if not enabled:
                status = {
                    "name": name,
                    "enabled": False,
                    "status": "disabled",
                    "url": None,
                    "last_check": None,
                    "response_time_ms": None,
                }
            else:
                # Test scraper connectivity
                status = await test_scraper_connectivity(name, url)
            
            statuses.append(status)
        
        return {"scrapers": statuses, "timestamp": time.time()}
    
    raise HTTPException(401, "Authentication required")

async def test_scraper_connectivity(name: str, url: str):
    """Test if a scraper is accessible"""
    import aiohttp
    
    start_time = time.time()
    try:
        async with aiohttp.ClientSession() as session:
            async with session.get(url, timeout=aiohttp.ClientTimeout(total=5)) as resp:
                response_time = (time.time() - start_time) * 1000
                return {
                    "name": name,
                    "enabled": True,
                    "status": "healthy" if resp.status == 200 else "degraded",
                    "url": url,
                    "last_check": time.time(),
                    "response_time_ms": round(response_time, 2),
                    "http_status": resp.status
                }
    except Exception as e:
        return {
            "name": name,
            "enabled": True,
            "status": "unhealthy",
            "url": url,
            "last_check": time.time(),
            "response_time_ms": None,
            "error": str(e)
        }
```

### Add to Admin Dashboard Template
```html
<!-- Add to comet/templates/admin_dashboard.html -->
<div class="scraper-status-section">
    <h2>Scraper Status</h2>
    <div id="scraper-status-container">
        <!-- Populated by JavaScript -->
    </div>
</div>

<script>
async function loadScraperStatus() {
    const response = await fetch('/admin/api/scrapers/status');
    const data = await response.json();
    
    const container = document.getElementById('scraper-status-container');
    container.innerHTML = data.scrapers.map(scraper => `
        <div class="scraper-card ${scraper.status}">
            <h3>${scraper.name}</h3>
            <span class="status-badge ${scraper.status}">${scraper.status}</span>
            ${scraper.enabled ? `
                <p>Response Time: ${scraper.response_time_ms || 'N/A'} ms</p>
                <p>URL: ${scraper.url}</p>
            ` : '<p>Disabled</p>'}
        </div>
    `).join('');
}

// Refresh every 30 seconds
setInterval(loadScraperStatus, 30000);
loadScraperStatus();
</script>
```

---

## 4. Cache Statistics Endpoint

**Complexity**: Low  
**Time Estimate**: 2-3 hours  
**Files to Modify**: `comet/api/endpoints/admin.py`

### Implementation

```python
# Add to comet/api/endpoints/admin.py

@router.get("/admin/api/cache/stats", tags=["Admin"])
async def get_cache_statistics(admin_session: str = Cookie(None)):
    """Get detailed cache statistics"""
    if settings.PUBLIC_METRICS_API or await verify_admin_session(admin_session):
        # Query database for cache statistics
        
        # Total torrents cached
        torrent_count = await database.fetch_val(
            "SELECT COUNT(*) FROM torrents"
        )
        
        # Torrents by quality
        quality_distribution = await database.fetch_all(
            """
            SELECT quality, COUNT(*) as count 
            FROM torrents 
            GROUP BY quality 
            ORDER BY count DESC
            """
        )
        
        # Cache size estimate
        cache_size = await database.fetch_val(
            "SELECT pg_database_size(current_database())"
        ) if settings.DATABASE_TYPE == "postgresql" else None
        
        # Most cached content
        top_cached = await database.fetch_all(
            """
            SELECT imdb_id, title, COUNT(*) as torrent_count
            FROM torrents
            GROUP BY imdb_id, title
            ORDER BY torrent_count DESC
            LIMIT 10
            """
        )
        
        # Cache age statistics
        oldest_torrent = await database.fetch_one(
            "SELECT MIN(updated_at) as oldest FROM torrents"
        )
        newest_torrent = await database.fetch_one(
            "SELECT MAX(updated_at) as newest FROM torrents"
        )
        
        return {
            "total_torrents": torrent_count,
            "quality_distribution": [
                {"quality": row["quality"], "count": row["count"]}
                for row in quality_distribution
            ],
            "database_size_bytes": cache_size,
            "top_cached_content": [
                {
                    "imdb_id": row["imdb_id"],
                    "title": row["title"],
                    "torrent_count": row["torrent_count"]
                }
                for row in top_cached
            ],
            "cache_age": {
                "oldest_entry": oldest_torrent["oldest"] if oldest_torrent else None,
                "newest_entry": newest_torrent["newest"] if newest_torrent else None,
            },
            "timestamp": time.time()
        }
    
    raise HTTPException(401, "Authentication required")
```

---

## 5. Request Logging System

**Complexity**: Medium  
**Time Estimate**: 4-6 hours  
**Files to Modify**: `comet/api/app.py`, add middleware

### Implementation

```python
# Add to comet/api/app.py

from fastapi import Request
import time

@app.middleware("http")
async def log_requests(request: Request, call_next):
    """Log all incoming requests with timing and status"""
    start_time = time.time()
    
    # Get request details
    client_ip = request.client.host
    method = request.method
    path = request.url.path
    
    # Process request
    response = await call_next(request)
    
    # Calculate processing time
    process_time = (time.time() - start_time) * 1000
    
    # Log the request
    logger.info(
        f"{method} {path} - "
        f"Status: {response.status_code} - "
        f"IP: {client_ip} - "
        f"Time: {process_time:.2f}ms"
    )
    
    # Add timing header
    response.headers["X-Process-Time"] = f"{process_time:.2f}ms"
    
    # Optionally store in database for analytics
    if settings.DATABASE_REQUEST_LOGGING:
        await store_request_log(
            method=method,
            path=path,
            status=response.status_code,
            ip=client_ip,
            duration_ms=process_time,
            timestamp=start_time
        )
    
    return response

async def store_request_log(method, path, status, ip, duration_ms, timestamp):
    """Store request log in database"""
    try:
        await database.execute(
            """
            INSERT INTO request_logs 
            (method, path, status_code, ip_address, duration_ms, timestamp)
            VALUES (:method, :path, :status, :ip, :duration, :timestamp)
            """,
            {
                "method": method,
                "path": path,
                "status": status,
                "ip": ip,
                "duration": duration_ms,
                "timestamp": timestamp
            }
        )
    except Exception as e:
        logger.error(f"Failed to store request log: {e}")
```

### Database Migration

```sql
-- Add request logs table
CREATE TABLE IF NOT EXISTS request_logs (
    id SERIAL PRIMARY KEY,
    method VARCHAR(10) NOT NULL,
    path VARCHAR(500) NOT NULL,
    status_code INTEGER NOT NULL,
    ip_address VARCHAR(45) NOT NULL,
    duration_ms FLOAT NOT NULL,
    timestamp BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Add indexes for common queries
CREATE INDEX idx_request_logs_timestamp ON request_logs(timestamp);
CREATE INDEX idx_request_logs_path ON request_logs(path);
CREATE INDEX idx_request_logs_ip ON request_logs(ip_address);
```

### Analytics Endpoint

```python
@router.get("/admin/api/requests/analytics", tags=["Admin"])
async def get_request_analytics(
    admin_session: str = Cookie(None),
    hours: int = 24
):
    """Get request analytics for the specified time period"""
    await require_admin_auth(admin_session)
    
    cutoff_time = time.time() - (hours * 3600)
    
    # Total requests
    total_requests = await database.fetch_val(
        "SELECT COUNT(*) FROM request_logs WHERE timestamp > :cutoff",
        {"cutoff": cutoff_time}
    )
    
    # Requests by endpoint
    by_endpoint = await database.fetch_all(
        """
        SELECT path, COUNT(*) as count, AVG(duration_ms) as avg_duration
        FROM request_logs
        WHERE timestamp > :cutoff
        GROUP BY path
        ORDER BY count DESC
        LIMIT 20
        """,
        {"cutoff": cutoff_time}
    )
    
    # Requests by status code
    by_status = await database.fetch_all(
        """
        SELECT status_code, COUNT(*) as count
        FROM request_logs
        WHERE timestamp > :cutoff
        GROUP BY status_code
        """,
        {"cutoff": cutoff_time}
    )
    
    # Top IPs
    top_ips = await database.fetch_all(
        """
        SELECT ip_address, COUNT(*) as count
        FROM request_logs
        WHERE timestamp > :cutoff
        GROUP BY ip_address
        ORDER BY count DESC
        LIMIT 10
        """,
        {"cutoff": cutoff_time}
    )
    
    return {
        "period_hours": hours,
        "total_requests": total_requests,
        "by_endpoint": [dict(row) for row in by_endpoint],
        "by_status": [dict(row) for row in by_status],
        "top_ips": [dict(row) for row in top_ips],
    }
```

---

## Testing Checklist

For each implemented feature:

- [ ] Unit tests written
- [ ] Integration tests pass
- [ ] Manual testing completed
- [ ] Documentation updated
- [ ] Performance impact assessed
- [ ] Security review done
- [ ] Error handling verified
- [ ] Logging added
- [ ] Configuration options documented
- [ ] Backward compatibility maintained

---

## Deployment Notes

1. **Database Migrations**: Run any new migrations before deploying
2. **Configuration**: Add new environment variables to `.env-sample`
3. **Documentation**: Update README.md with new features
4. **Monitoring**: Set up alerts for new endpoints
5. **Rollback Plan**: Test rollback procedure

---

## Performance Considerations

1. **Caching**: Cache expensive operations (scraper status checks, statistics)
2. **Rate Limiting**: Limit frequency of status checks
3. **Async Operations**: Use async for all I/O operations
4. **Database Indexes**: Add indexes for frequently queried fields
5. **Connection Pooling**: Reuse database connections

---

## Security Considerations

1. **Authentication**: Verify admin session for all sensitive endpoints
2. **Input Validation**: Validate all user inputs
3. **SQL Injection**: Use parameterized queries
4. **Rate Limiting**: Prevent abuse of new endpoints
5. **Logging**: Don't log sensitive information (API keys, passwords)

---

*For more features and detailed descriptions, see [FEATURE_SUGGESTIONS.md](FEATURE_SUGGESTIONS.md)*
