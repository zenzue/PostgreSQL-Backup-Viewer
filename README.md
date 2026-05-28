# PostgreSQL Backup Viewer

This setup allows you to view a PostgreSQL custom dump file using Docker Compose and a browser-based GUI.

It uses:

* PostgreSQL 17 for restoring the backup
* pgweb for viewing the database in a browser
* Environment variables for database credentials, ports, and GUI login

No Java-based GUI tool is required.

by `Aung Myat Thu`

---

## Folder Structure

Create a folder like this:

```bash
mkdir -p backup-viewer/backup
cd backup-viewer
```

Expected structure:

```text
backup-viewer/
├── backup/
│   └── database.dump
├── .env
├── docker-compose.yml
└── README.md
```

---

## 1. Prepare the Backup File

Copy your PostgreSQL dump file into the `backup` folder and rename it to:

```text
database.dump
```

Example:

```bash
cp /path/to/your/backup.dump ./backup/database.dump
```

Check the file:

```bash
ls -lh ./backup/database.dump
```

---

## 2. Create `.env`

Create a `.env` file:

```bash
vim .env
```

Paste:

```env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=mysecretpassword
POSTGRES_DB=appdb

DB_HOST=backup-db
DB_PORT=5432

LOCAL_DB_PORT=55432
PGWEB_PORT=8081

PGWEB_AUTH_USER=admin
PGWEB_AUTH_PASSWORD=admin123
```

You can change the username, password, database name, and ports as needed.

---

## 3. Create `docker-compose.yml`

Create the Compose file:

```bash
vim docker-compose.yml
```

Paste:

```yaml
services:
  backup-db:
    image: postgres:17
    container_name: backup-view-db
    env_file:
      - .env
    ports:
      - "${LOCAL_DB_PORT}:5432"
    volumes:
      - backup_view_db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 2s
      timeout: 3s
      retries: 30
    restart: unless-stopped

  backup-restore:
    image: postgres:17
    container_name: backup-view-restore
    env_file:
      - .env
    depends_on:
      backup-db:
        condition: service_healthy
    volumes:
      - ./backup:/backup:ro
    command: >
      bash -lc "
      echo '[+] Restoring database backup...';

      PGPASSWORD=\"$${POSTGRES_PASSWORD}\" pg_restore
      -h \"$${DB_HOST}\"
      -p \"$${DB_PORT}\"
      -U \"$${POSTGRES_USER}\"
      -d \"$${POSTGRES_DB}\"
      --clean
      --if-exists
      /backup/database.dump;

      echo '[+] Restore completed.';
      "
    restart: "no"

  backup-pgweb:
    image: sosedoff/pgweb
    container_name: backup-view-pgweb
    env_file:
      - .env
    depends_on:
      backup-db:
        condition: service_healthy
    ports:
      - "${PGWEB_PORT}:8081"
    command: >
      sh -lc "
      pgweb
      --bind=0.0.0.0
      --listen=8081
      --auth-user=\"$${PGWEB_AUTH_USER}\"
      --auth-pass=\"$${PGWEB_AUTH_PASSWORD}\"
      --url=\"postgres://$${POSTGRES_USER}:$${POSTGRES_PASSWORD}@$${DB_HOST}:$${DB_PORT}/$${POSTGRES_DB}?sslmode=disable\"
      "
    restart: unless-stopped

volumes:
  backup_view_db_data:
```

---

## 4. Start the Database

```bash
docker compose up -d backup-db
```

Check status:

```bash
docker compose ps
```

---

## 5. Restore the Backup

```bash
docker compose run --rm backup-restore
```

Expected output:

```text
[+] Restoring database backup...
[+] Restore completed.
```

---

## 6. Start the Browser GUI

```bash
docker compose up -d backup-pgweb
```

Open in browser:

```text
http://localhost:8081
```

Login using the values from `.env`:

```text
Username: admin
Password: admin123
```

---

## 7. View Tables

After logging in:

1. Open the database.
2. Browse schemas.
3. Select a table.
4. View rows and columns from the browser.

---

## 8. Restore a Different Backup

To restore another dump file:

```bash
docker compose down -v
cp /path/to/new_backup.dump ./backup/database.dump
docker compose up -d backup-db
docker compose run --rm backup-restore
docker compose up -d backup-pgweb
```

The `down -v` command removes the temporary restored database volume, so the new backup can be restored cleanly.

---

## 9. Stop Services

Stop containers but keep restored data:

```bash
docker compose down
```

Stop containers and delete restored database data:

```bash
docker compose down -v
```

---

## 10. Useful Commands

Check running containers:

```bash
docker compose ps
```

View logs:

```bash
docker compose logs -f backup-db
docker compose logs -f backup-pgweb
```

Open PostgreSQL shell:

```bash
docker compose exec backup-db psql -U postgres -d appdb
```

List tables inside `psql`:

```sql
\dt
```

Quit `psql`:

```sql
\q
```

---

## Notes

* This setup is for local backup viewing and verification.
* Do not expose the pgweb port publicly without proper access control.
* Keep the `.env` file private because it contains credentials.
* The restored database is stored in a Docker volume and can be removed with `docker compose down -v`.
