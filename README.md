# front-notes

`.htaccess`:
```apache
<IfModule mod_rewrite.c>
    RewriteEngine On

    # Exclude /billing and /wordpress from these rules
    RewriteCond %{REQUEST_URI} ^/billing [OR]
    RewriteCond %{REQUEST_URI} ^/wordpress
    RewriteRule ^ - [L]
    
    # If a .html file exists for the request, serve it (even if a directory with the same name exists)
    RewriteCond %{REQUEST_FILENAME} -d
    RewriteCond %{REQUEST_FILENAME}.html -f
    RewriteRule ^(.*)$ $1.html [L]
    
    # For requests that aren't files or directories, try adding .html
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_FILENAME}.html -f
    RewriteRule ^(.*)$ $1.html [L]
    
    ErrorDocument 404 /404.html
  
</IfModule>

# NextJS 304 Not Modified cache fix
FileETag None

# NextJS 304 Not Modified cache fix headers
<IfModule mod_headers.c>
  # Unset ETag header
  Header unset ETag
  
  # never cache html files
  <FilesMatch "\.html$">
    Header set Cache-Control "no-store, no-cache, must-revalidate, max-age=0"
    Header set Pragma "no-cache"
    Header set Expires "0"
  </FilesMatch>
  
  # Static assets
  # <FilesMatch "\.(css|js|png|jpg|jpeg|gif|webp|svg|woff2?)$">
  #  Header set Cache-Control "public, max-age=31536000, immutable"
  # </FilesMatch>
</IfModule>

```

NextJS `304 Not Modified` cache fix:

1. LiteSpeed was sending the same `ETag` and `Last-Modified` headers for all HTML pages (since all pages were generated at same time during NextJS build). 
2. Cloudflare passed those headers along to the browser. Browser cached the first page visited and when visited a different page, it sent an `If-Modified-Since` or `If-None-Match` request.
3. Server saw the timestamps matched and responded with `304 Not Modified`, telling the browser to reuse the cached content (This is wrong). This will cause the page to load plain white HTML with broken CSS.

We must disable ETags and force `no-store, no-cache` on HTML files, so every page request gets fresh response.

`export.sh`:
```sh
#!/bin/bash
set -e # Exit immediately if any command fails

# cPanel Configuration
CPANEL_HOST="192.168.8.242"
CPANEL_USER="foxomy"
CPANEL_API_KEY="YN9DVGZJRR2B8ADWBDS7DZBE9U08FU14"

pnpm run build

RANDOM_CHARS=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 4 | head -n 1)
DATETIME=$(date +"%Y-%m-%d-%H%M%S")
FILENAME="out-${RANDOM_CHARS}-${DATETIME}.zip"
(cd ./out && zip -r "../${FILENAME}" .)
echo "Created: ${FILENAME}"

# Upload to cPanel
echo "Uploading to cPanel..."
curl -k \
  -H "Authorization: cpanel ${CPANEL_USER}:${CPANEL_API_KEY}" \
  -F "dir=public_html" \
  -F "file-1=@${FILENAME}" \
  "https://${CPANEL_HOST}:2083/execute/Fileman/upload_files"

echo ""
echo "Upload complete!"

# Delete old _next folder before extracting
echo "Deleting old _next folder..."
curl -k \
  -H "Authorization: cpanel ${CPANEL_USER}:${CPANEL_API_KEY}" \
  "https://${CPANEL_HOST}:2083/json-api/cpanel?cpanel_jsonapi_apiversion=2&cpanel_jsonapi_module=Fileman&cpanel_jsonapi_func=fileop&op=unlink&sourcefiles=/home/${CPANEL_USER}/public_html/_next"

echo ""
echo "_next folder deleted!"

# Extract the zip file on cPanel
echo "Extracting zip file..."
curl -k \
  -H "Authorization: cpanel ${CPANEL_USER}:${CPANEL_API_KEY}" \
  "https://${CPANEL_HOST}:2083/json-api/cpanel?cpanel_jsonapi_apiversion=2&cpanel_jsonapi_module=Fileman&cpanel_jsonapi_func=fileop&op=extract&sourcefiles=public_html/${FILENAME}&destfiles=/public_html"

# Delete the zip file from cPanel
echo "Deleting zip from server..."
curl -k \
  -H "Authorization: cpanel ${CPANEL_USER}:${CPANEL_API_KEY}" \
  "https://${CPANEL_HOST}:2083/json-api/cpanel?cpanel_jsonapi_apiversion=2&cpanel_jsonapi_module=Fileman&cpanel_jsonapi_func=fileop&op=unlink&sourcefiles=/home/${CPANEL_USER}/public_html/${FILENAME}"

echo ""
echo "Server zip deleted!"

# Prompt to delete local zip
read -p "Delete local zip file? (y/n): " DELETE_LOCAL
if [ "$DELETE_LOCAL" = "y" ] || [ "$DELETE_LOCAL" = "Y" ]; then
  rm "${FILENAME}"
  echo "Local zip deleted!"
else
  echo "Local zip kept: ${FILENAME}"
fi

echo ""
echo "All done!"
```