# Awtomated Docker Deployment Troubleshooting Guide

## Common Issues and Solutions

### 1. PostgreSQL Collation Version Warnings

**Symptoms:**
```
WARNING: database "postgres" has a collation version mismatch
DETAIL: The database was created using collation version 2.36, but the operating system provides version 2.41.
HINT: Rebuild all objects in this database that use the default collation and run ALTER DATABASE postgres REFRESH COLLATION VERSION
```

**Cause:**
PostgreSQL Docker containers use newer OS locale libraries than what the database was initialized with.

**Solution:**
```bash
# Fix collation warnings for a database
docker compose exec postgres psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "ALTER DATABASE \$POSTGRES_DB REFRESH COLLATION VERSION;"
```

### 2. Database Connection Issues

**Symptoms:**
```
FATAL: database "db_name" does not exist
```

**Cause:**
- Database was not created during initialization
- Environment variable mismatch
- Volume persistence issues

**Diagnosis:**
```bash
# Check what databases exist
docker compose exec postgres psql -U "$POSTGRES_USER" -l

# Check environment variables
docker compose exec postgres env | grep POSTGRES

# Verify user permissions
docker compose exec postgres psql -U "$POSTGRES_USER" -c "\du"
```

**Solution:**
```bash
# Create missing database
docker compose exec postgres psql -U "$POSTGRES_USER" -c "CREATE DATABASE \$POSTGRES_DB;"

# Or recreate from scratch (DATA LOSS WARNING)
docker compose down
docker volume rm postgres_data_volume_name
docker compose up --build -d
```

### 3. Cache Issues After Django 5 Upgrade

**Symptoms:**
- Inconsistent behavior after deployment
- Django errors about cache format

**Solution:**
```bash
# Clear Django cache
docker compose exec django python manage.py shell -c "from django.core.cache import cache; cache.clear()"

# Or restart Redis
docker compose restart redis
```

### 4. Migration Issues

**Symptoms:**
- Migration errors during startup
- Database schema inconsistencies

**Solution:**
```bash
# Check migration status
docker compose exec django python manage.py showmigrations

# Apply pending migrations
docker compose exec django python manage.py migrate

# For complex migration issues
docker compose exec django python manage.py migrate --fake-initial
```

### 5. Admin Static Files Not Loading

**Symptoms:**
- CSS/JS files return 404 errors
- Admin interface looks broken

**Solution:**
```bash
# Collect static files
docker compose exec django python manage.py collectstatic --noinput

# Check nginx configuration
docker compose exec nginx nginx -t

# Restart nginx
docker compose restart nginx
```

### 6. SSL Certificate Issues

**Symptoms:**
- HTTPS not working
- Browser SSL warnings

**Solution:**
```bash
# Regenerate SSL certificates

# Restart nginx to pick up new certificates
docker compose restart nginx

# Check certificate validity
docker compose exec nginx openssl x509 -in /etc/ssl/certs/cert.pem -text -noout
```

## Deployment Best Practices

### Pre-Deployment Checklist

1. **Backup Database:**
   ```bash
   docker compose exec postgres pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB" > backup_$(date +%Y%m%d_%H%M%S).sql
   ```

2. **Check Environment Variables:**
   ```bash
   grep -v '^#' .env | grep -v '^$'
   ```

3. **Verify Image Availability:**
   ```bash
   docker compose config
   docker compose pull
   ```

### Post-Deployment Verification

1. **Health Checks:**
   ```bash
   # Check all services are running
   docker compose ps
   
   # Check application health
   curl -f http://localhost/health/ || echo "Health check failed"
   
   # Check database connectivity
   docker compose exec django python manage.py dbshell -c "SELECT 1;"
   ```

2. **Log Monitoring:**
   ```bash
   # Monitor startup logs
   docker compose logs -f django
   
   # Check for errors
   docker compose logs | grep -i error
   ```

### Maintenance Commands

```bash
# Update all services
docker compose pull
docker compose up -d

# Clean up old images/volumes
docker system prune -f
docker volume prune -f

# Backup and restore database
docker compose exec postgres pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB" > backup.sql
docker compose exec -T postgres psql -U "$POSTGRES_USER" "$POSTGRES_DB" < backup.sql
```

## Getting Help

1. **Check Logs:** Always start with `docker compose logs [service_name]`
2. **Verify Configuration:** Use `docker compose config` to check syntax
3. **Database Console:** Use `docker compose exec postgres psql -U "$POSTGRES_USER" "$POSTGRES_DB"` for direct DB access
4. **Django Shell:** Use `docker compose exec django python manage.py shell` for Django debugging
