# echo-speaks-ax-patch

Unofficial fix for the **Echo Speaks server v2.8.0** Amazon login, broken by Amazon's new sign-in page in October 2026.

## Symptoms

When you log in through the Echo Speaks server (`http://<server-ip>:8091` → **Goto Login Page**), entering your email or mobile number and pressing **Continue** fails with either:

- `This site can't provide a secure connection` / `ERR_SSL_PROTOCOL_ERROR` (the browser is sent to `https://<server-ip>:8091/ax/claim`), or
- `Cannot GET /ax/claim` or `Cannot POST /ax/claim`.

Adding an SSL certificate does not help. The server simply has no route for the new page.

## Cause

Amazon now uses an identifier-first sign-in flow whose form posts to a root-relative path, `/ax/claim`. The login proxy inside the server, a bundled 2020 copy of `alexa-cookie2` (3.4.1) at `libs/alexa-cookie/lib/proxy.js`, only handles URLs under `/proxy/www.amazon.com/...`. The new path lands outside the proxy, and Amazon's security headers make the browser upgrade it to https.

## What the patch changes

`proxy.js` is a drop-in replacement for `libs/alexa-cookie/lib/proxy.js`. It makes three changes, each marked `ES-AX-PATCH`:

1. **Strips security headers.** Amazon's `content-security-policy` and `strict-transport-security` headers are removed from proxied responses, so the browser no longer upgrades the form post to https.
2. **Rewrites root-relative links.** In proxied pages, `action`, `formaction`, `href` and `src` values starting with `/` are rewritten to `/proxy/www.amazon.com/...`, so the sign-in form posts back through the proxy.
3. **Adds a safety net.** Requests reaching the server root at `/ax`, `/ap`, `/cvf`, `/aaut` or `/gp` get a 307 redirect into the proxy path. This catches URLs Amazon builds in script, and 307 keeps POST data intact.

Nothing else changes: login-success detection and token handling are as in the original.

**SHA-256** of `proxy.js`:

```
2a777a69f640b50c5da69415c70ac9ee73131bdd6b10cabeb43fd97680313ec0
```

## Do you need it?

Only if you have to log in again. If Echo Speaks is working now, leave it alone until your login expires.

## Apply: Docker (any host, including QNAP over SSH)

Download `proxy.js` from this repo onto the Docker host, then run these commands from the folder containing it.

1. Find your container name:

```
docker ps --format '{{.Names}}  {{.Image}}'
```

2. Confirm the file location inside the container. It should print `/usr/src/app/libs/alexa-cookie/lib/proxy.js`; if yours differs, use your path in the next steps.

```
docker exec <container> find / -path '*alexa-cookie/lib/proxy.js' 2>/dev/null
```

3. Back up the original, copy the patched file in, and restart:

```
docker cp <container>:/usr/src/app/libs/alexa-cookie/lib/proxy.js ./proxy.js.orig && docker cp ./proxy.js <container>:/usr/src/app/libs/alexa-cookie/lib/proxy.js && docker restart <container>
```

4. Check the patch is in place. It should print `6`.

```
docker exec <container> grep -c ES-AX-PATCH /usr/src/app/libs/alexa-cookie/lib/proxy.js
```

5. Log in again from a private/incognito browser window at `http://<server-ip>:8091`.

`docker cp` survives container restarts but **not** a container recreate or image update; repeat it if that happens, or use the compose method below.

## Apply: Docker compose (permanent)

Mount the patched file over the original with a compose override. See [`docker-compose.override.example.yml`](docker-compose.override.example.yml): copy it next to your `docker-compose.yml` as `docker-compose.override.yml`, set your service name and the path to `proxy.js`, then:

```
docker compose up -d
```

On SELinux hosts (RHEL, Rocky, Fedora), add `,Z` after `:ro` in the volume line.

## Apply: without Docker

In your Echo Speaks server folder, back up `libs/alexa-cookie/lib/proxy.js`, replace it with the patched `proxy.js`, and restart the server.

## Undo

- **`docker cp` method:** copy the backup back and restart:

```
docker cp ./proxy.js.orig <container>:/usr/src/app/libs/alexa-cookie/lib/proxy.js && docker restart <container>
```

- **Compose method:** remove the volume line (or the override file) and run `docker compose up -d`.
- **Without Docker:** restore your backup and restart the server.

Undo it the same way once an official server fix is released.

## Caveats

- Unofficial stopgap, not from the Echo Speaks developers.
- Tested on server v2.8.0, `amazon.com`, with OTP sign-in. Other Amazon regions use the same code paths but are untested.
- If your server is a different or modified build, compare your `proxy.js` against the stock v2.8.0 file before replacing it.
- The server still uses the 2020 login library, so whether the automatic cookie refresh keeps working long-term is unproven. If your login expires, log in again through the patched proxy.
- The server logs Amazon session cookies in plain text. Keep its logs private.
