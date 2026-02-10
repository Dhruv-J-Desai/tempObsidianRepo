Yep — that’s expected: **`docker images` shows what’s on your machine**, while `curl .../v2/_catalog` shows what’s inside the **registry** at `WINL09485465.TDBFG.com:5000`. They’re two different stores, so the lists won’t match.

Since you want to remove **only the repos/tags that appear in the registry (_catalog)**, you must delete them via the **Registry HTTP API** (not `docker rmi`).

Here’s the clean, repeatable way.

---

## 1) List tags for a repo (registry)

Example for `bishowcase-backend`:

```bash
curl http://WINL09485465.TDBFG.com:5000/v2/bishowcase-backend/tags/list
```

Do this for each repo shown in `_catalog`.

---

## 2) For each tag, get its digest

Example tag `1.0`:

```bash
curl -I \
  -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  http://WINL09485465.TDBFG.com:5000/v2/bishowcase-backend/manifests/1.0
```

Copy the value of:

```
Docker-Content-Digest: sha256:...
```

---

## 3) Delete that digest (this deletes the tag’s manifest)

```bash
curl -X DELETE \
  http://WINL09485465.TDBFG.com:5000/v2/bishowcase-backend/manifests/sha256:PASTE_DIGEST_HERE
```

Expected: `202 Accepted`

Repeat for all tags you want removed.

---

## 4) Garbage-collect (actually frees space)

After deleting manifests, space won’t be reclaimed until GC runs (if you have access to the registry container/host):

```bash
docker exec -it registry \
  registry garbage-collect /etc/docker/registry/config.yml
```

---

# One-command “delete all tags of a repo” helper (copy/paste)

This will delete _every tag_ under a given repo in the registry:

```bash
REG=http://WINL09485465.TDBFG.com:5000
REPO=bishowcase-backend

for tag in $(curl -s $REG/v2/$REPO/tags/list | sed -n 's/.*"tags":\[\(.*\)\].*/\1/p' | tr -d '"' | tr ',' ' '); do
  digest=$(curl -sI -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
    $REG/v2/$REPO/manifests/$tag | tr -d '\r' | awk -F': ' '/Docker-Content-Digest/ {print $2}')
  echo "Deleting $REPO:$tag -> $digest"
  curl -s -X DELETE $REG/v2/$REPO/manifests/$digest
  echo
done
```

Run it once per repo you want to wipe from `_catalog`.

---

## Important: if DELETE doesn’t work

If you get `405 Method Not Allowed` or similar, your registry has delete disabled. It must have:

- `REGISTRY_STORAGE_DELETE_ENABLED=true`
    

---

If you tell me which repos from `_catalog` you want to delete (all of them or only `bishowcase-backend`, etc.), I’ll tailor the exact commands for your list.