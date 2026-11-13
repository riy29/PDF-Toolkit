# GToolkit

A Globiva-branded internal PDF processing tool with compression, bulk compression, splitting, merging, rotation, conversion, watermarking, metadata inspection, local superuser access, and optional LDAP authentication.

## Features

- **Compress**: Reduce PDF/image file size to a manually entered KB target.
- **Bulk Compress**: Compress multiple PDF/image files into one ZIP download. The temporary ZIP is kept for 30 minutes by default.
- **Split**: Fixed chunks (e.g. 3 pages per file) or custom ranges; download as ZIP.
- **Merge**: Combine multiple PDFs into one.
- **Rotate**: Rotate PDF pages by 90, 180, or 270 degrees.
- **PDF to Images**: Convert PDF pages to PNG images with DPI selection.
- **Extract Images**: Extract embedded images from PDFs.
- **Watermark**: Add text watermarks to PDFs.
- **PDF Info**: View PDF metadata and information.

## Run (local)

```powershell
docker compose up --build
```

Open: `http://127.0.0.1:3076`

## Run (production server)

Host nginx should proxy to `127.0.0.1:8001`. Start the app with the prod compose overlay:

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
```

See [DEPLOY.md](DEPLOY.md) if you see nginx `Connection refused` to port 8001.

Default local superuser:

```text
Username: superadmin
Password: ChangeMe123!
```

Change this in `docker-compose.yml` or `.env` before production use.

## LDAP Setup

Copy `.env.example` to `.env`, then configure:

```env
USE_LDAP=true
LOCAL_SUPERUSER_USERNAME=superadmin
LOCAL_SUPERUSER_PASSWORD=ChangeMe123!
PDF_TOOL_ADMINS=pdfadmin
ALLOWED_AD_USERS=jdoe,asmith
ACCESS_CONFIG_PATH=/tmp/pdf-compressor/access_control.json
LDAP_SERVER=ldap://your-ldap-server:389
LDAP_BASE_DN=dc=example,dc=com
LDAP_USER_ATTR=uid
LDAP_USER_DN_TEMPLATE={attr}={username},{base_dn}
LDAP_BIND_DN=cn=admin,dc=example,dc=com
LDAP_BIND_PASSWORD=your-password
```

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `USE_LDAP` | `false` | Enable LDAP authentication |
| `LOCAL_SUPERUSER_USERNAME` | `superadmin` | Local admin username. Leave blank to disable local superuser auth. |
| `LOCAL_SUPERUSER_PASSWORD` | `ChangeMe123!` | Local admin password. Change before production use. |
| `LOCAL_SUPERUSER_PASSWORD_HASH` | `` | Optional Werkzeug password hash. If set, it is used instead of `LOCAL_SUPERUSER_PASSWORD`. |
| `PDF_TOOL_ADMINS` | `` | Comma-separated AD usernames allowed to open frontend access management |
| `ALLOWED_AD_USERS` | `` | Comma-separated AD usernames allowed to sign in |
| `DEFAULT_ALLOWED_TOOLS` | `` | Optional fallback tool list for allowed users without mapped LDAP groups |
| `ACCESS_CONFIG_PATH` | `/tmp/pdf-compressor/access_control.json` | JSON file used by the admin page to store login/tool permissions |
| `LDAP_SERVER` | `ldap://localhost:389` | LDAP server URL |
| `LDAP_BASE_DN` | `dc=example,dc=com` | LDAP base DN for user searches |
| `LDAP_USER_ATTR` | `uid` | LDAP attribute for username |
| `LDAP_USER_DN_TEMPLATE` | `{attr}={username},{base_dn}` | Bind identity template. For AD, use `{username}@company.com` or `COMPANY\\{username}` if needed. |
| `LDAP_BIND_DN` | `` | LDAP service account DN for admin user search |
| `LDAP_BIND_PASSWORD` | `` | LDAP service account password |
| `LDAP_TIMEOUT` | `10` | LDAP connection timeout in seconds |
| `LDAP_USE_TLS` | `false` | Use TLS for LDAP connection |
| `LDAP_USE_SSL` | `false` | Use SSL for LDAP connection |
| `MAX_UPLOAD_MB` | `80` | Maximum file upload size in MB |
| `MIN_TARGET_KB` | `10` | Minimum custom compression target in KB |
| `MAX_TARGET_KB` | `10240` | Maximum custom compression target in KB |
| `BULK_RETENTION_SECONDS` | `1800` | Temporary bulk-compression ZIP retention period. Default is 30 minutes. |
| `SECRET_KEY` | `local-compressor-dev-key` | Flask secret key |

## Access Control

The local superuser can log in at `/login`, open `/admin`, and create frontend-managed users with specific tool permissions. This works even when `USE_LDAP=false`.

When LDAP is enabled:

1. The configured local superuser can always log in and manage roles.
2. Users listed in `PDF_TOOL_ADMINS` can open `/admin` after LDAP login.
3. Frontend-managed records control `login_enabled` and selected tools.
4. If no frontend-managed record exists, the env/group fallback still works: `ALLOWED_AD_USERS`, LDAP groups, and `DEFAULT_ALLOWED_TOOLS`.

Built-in LDAP role groups:

```text
pdf-compressor, pdf-splitter, pdf-merger, pdf-rotator,
pdf-converter, pdf-watermark, pdf-info, pdf-admin
```

## API Endpoints

All endpoints require authentication when local superuser auth or LDAP auth is enabled.

- `POST /compress` - Compress one PDF/image
- `POST /api/bulk-compress` - Compress multiple PDF/image files and return a temporary ZIP download link
- `GET /download/bulk/<job_id>` - Download a temporary bulk-compression ZIP
- `POST /api/split` - Split PDF by page ranges
- `POST /api/merge` - Merge multiple PDFs
- `POST /api/rotate` - Rotate PDF pages
- `POST /api/extract-images` - Extract images from PDF
- `POST /api/pdf-to-images` - Convert PDF to PNG images
- `POST /api/watermark` - Add watermark to PDF
- `POST /api/compress` - In-memory PDF compression
- `POST /api/info` - Get PDF metadata

## Notes

Single-file processing is returned immediately. Bulk compression stores the generated ZIP temporarily under `/tmp/pdf-compressor/bulk` and automatically removes expired packages after the configured retention period.
