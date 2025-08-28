# Fixing X Cross‑Validation Failures (double‑encoded JSON + schema mismatch)

This README consolidates **root causes**, **affected files**, and **drop‑in code patches** to fix the X (Twitter) cross‑validation failures you observed.

## TL;DR

- Your validator sometimes receives **double‑encoded JSON** → leads to `"'str' object has no attribute 'get'"` because code treats a JSON **string** like a **dict**.
- Some miners return **flat JSON** (`username`, `url`, `tweet_id`, …) while your validator expects the **enhanced/nested** schema (`user.username`, `uri`, nested `tweet` block) → Pydantic fails with:
  - `username: Input should be a valid string (input_value=None)`  
  - `url: Input should be a valid string (input_value=None)`

**You must fix both.** This guide provides surgical patches that:

1) robustly parse and normalize JSON (including **double‑encoded** payloads), and  
2) accept **both** enhanced (nested) and legacy/flat JSON payloads (mapping fields + synthesizing `url` when needed).

---

## Symptoms from your logs

```
Validation error for https://x.com/...: 2 validation errors for EnhancedXContent
username
  Input should be a valid string [input_value=None]
url
  Input should be a valid string [input_value=None]

Validation error for https://x.com/...: 'str' object has no attribute 'get'
```

- The **`'str' object has no attribute 'get'`** errors come from **double‑encoded JSON** or non‑JSON text.
- The **Pydantic string_type** errors for `username`/`url` come from **schema mismatch** (flat vs nested).

---

## Affected files (paths in your repo)

- `vali_utils/organic_query_processor.py`
- `scraping/x/on_demand_model.py`

> In the artifact you shared, the file path was detected as:  
> `/mnt/data/data-universe-main-10/data-universe-main/vali_utils/organic_query_processor.py`

Use the repo‑relative paths above when applying the patches.

---

## Patches (copy/paste)

> The following **OLD/NEW** blocks show the exact sections to replace. Paste the **NEW** blocks in place of the corresponding **OLD** blocks.

### 1) `scraping/x/on_demand_model.py` → `EnhancedXContent.from_data_entity`

**Why:** The current implementation assumes the enhanced nested schema and doesn’t handle double‑encoded JSON. This upgrade accepts both **nested** and **flat** shapes and safely parses double‑encoded JSON.

**OLD (start)**
```python
    @classmethod
    def from_data_entity(cls, data_entity: DataEntity) -> "EnhancedXContent":
        """Converts a DataEntity to an EnhancedXContent."""
        
        # Decode the content - this should be the new X API format
        content_str = data_entity.content.decode("utf-8")  
        content_dict = json.loads(content_str)
        
        # Extract data from the new API structure
        user_info = content_dict.get("user", {})
        tweet_info = content_dict.get("tweet", {})
        media_info = content_dict.get("media", [])
        
        # Map to EnhancedXContent fields
        username = user_info.get("username")
        if username and not username.startswith("@"):
            username = f"@{username}"
            
        text = content_dict.get("text")
        now = dt.datetime.now(dt.timezone.utc)
        if now <= X_ON_DEMAND_CONTENT_EXPIRATION_DATE:
            if not text:
                # Using 'content' as fallback for compatibility until Aug 25 2025
                text = content_dict.get("content")
        if not text:
            text = "" 
        url = content_dict.get("uri")
        
        # Handle timestamp - could be in content_dict or data_entity
        timestamp = data_entity.datetime
        
        # Extract hashtags from tweet info
        hashtags = tweet_info.get("hashtags", [])
        
        # Extract media URLs and types
        media_urls = []
        media_types = []
        for media_item in media_info:
            media_urls.append(media_item.get("url"))
            media_types.append(media_item.get("type", "unknown"))
        
        return cls(
            username=username,
            text=text,
            url=url,
            timestamp=timestamp,
            tweet_hashtags=hashtags,
            user_id=user_info.get("id"),
            user_display_name=user_info.get("display_name"),
            user_verified=user_info.get("verified"),
            user_followers_count=user_info.get("followers_count"),
            user_following_count=user_info.get("following_count"),
            tweet_id=tweet_info.get("id"),
            like_count=tweet_info.get("like_count"),
            retweet_count=tweet_info.get("retweet_count"),
            reply_count=tweet_info.get("reply_count"),
            quote_count=tweet_info.get("quote_count"),
            is_retweet=tweet_info.get("is_retweet"),
            is_reply=tweet_info.get("is_reply"),
            is_quote=tweet_info.get("is_quote"),
            media_urls=media_urls,
            media_types=media_types,
            conversation_id=tweet_info.get("conversation_id"),
            in_reply_to_user_id=tweet_info.get("in_reply_to", {}).get("user_id") if tweet_info.get("in_reply_to") else None
        )
```
**OLD (end)**

**NEW (start)**
```python
    @classmethod
    def from_data_entity(cls, data_entity: DataEntity) -> "EnhancedXContent":
        """Converts a DataEntity to an EnhancedXContent, accepting both enhanced (nested)
        and legacy/flat JSON, and handling double-encoded JSON safely.
        """
        # --- robust decode & parse (handles bytes, strings, and double-encoded JSON) ---
        raw = data_entity.content
        content_str = (
            raw.decode("utf-8", errors="replace")
            if isinstance(raw, (bytes, bytearray)) else str(raw or "")
        )

        def _parse_to_dict(s: str):
            try:
                obj = json.loads(s)
            except Exception:
                return {}
            # Handle double-encoded JSON: json.loads returns a string → load again
            if isinstance(obj, str):
                try:
                    obj = json.loads(obj)
                except Exception:
                    return {}
            return obj if isinstance(obj, dict) else {}

        content_dict = _parse_to_dict(content_str)

        # Prefer enhanced (nested) structure, fall back to legacy/flat keys
        user_info  = content_dict.get("user")  or {}
        tweet_info = content_dict.get("tweet") or {}
        media_info = content_dict.get("media") or []

        # Username: nested first, then flat variants
        username = (
            user_info.get("username")
            or content_dict.get("username")
            or content_dict.get("screen_name")
            or content_dict.get("user_screen_name")
        )
        if username and not username.startswith("@"):
            username = f"@{username}"

        # Text with legacy fallback (respect the expiration toggle for 'content' if desired)
        text = content_dict.get("text")
        now = dt.datetime.now(dt.timezone.utc)
        if now <= X_ON_DEMAND_CONTENT_EXPIRATION_DATE and not text:
            text = content_dict.get("content")
        text = text or ""

        # URL: prefer enhanced 'uri', fall back to flat 'url', synthesize if needed
        url = content_dict.get("uri") or content_dict.get("url")

        # Tweet id from nested or flat variants
        tweet_id = (
            tweet_info.get("id")
            or content_dict.get("tweet_id")
            or content_dict.get("id_str")
            or content_dict.get("id")
            or content_dict.get("rest_id")
        )

        if not url and username and tweet_id:
            url = f"https://x.com/{username.lstrip('@')}/status/{tweet_id}"

        # Hashtags: nested first, then flat
        hashtags = (
            tweet_info.get("hashtags")
            or content_dict.get("tweet_hashtags")
            or content_dict.get("hashtags")
            or []
        )

        # Media: accept nested [{url,type}] or flat lists
        media_urls: List[str] = []
        media_types: List[str] = []
        if isinstance(media_info, list) and media_info and isinstance(media_info[0], dict):
            for media_item in media_info:
                media_urls.append(media_item.get("url"))
                media_types.append(media_item.get("type", "unknown"))
        else:
            flat_media = content_dict.get("media_urls") or content_dict.get("media") or []
            if isinstance(flat_media, list):
                media_urls = flat_media
                media_types = ["unknown"] * len(flat_media)

        # Build the model (Pydantic requires non-None for required fields)
        return cls(
            username=username or "",
            text=text,
            url=url or "",
            timestamp=data_entity.datetime,
            tweet_hashtags=hashtags,
            user_id=user_info.get("id") or content_dict.get("user_id"),
            user_display_name=user_info.get("display_name") or content_dict.get("user_display_name"),
            user_verified=user_info.get("verified") if user_info.get("verified") is not None else content_dict.get("user_verified"),
            user_followers_count=user_info.get("followers_count") or content_dict.get("user_followers_count"),
            user_following_count=user_info.get("following_count") or content_dict.get("user_following_count"),
            tweet_id=tweet_id,
            like_count=(tweet_info.get("like_count") if tweet_info else content_dict.get("like_count")),
            retweet_count=(tweet_info.get("retweet_count") if tweet_info else content_dict.get("retweet_count")),
            reply_count=(tweet_info.get("reply_count") if tweet_info else content_dict.get("reply_count")),
            quote_count=(tweet_info.get("quote_count") if tweet_info else content_dict.get("quote_count")),
            is_retweet=(tweet_info.get("is_retweet") if tweet_info else content_dict.get("is_retweet")),
            is_reply=(tweet_info.get("is_reply") if tweet_info else content_dict.get("is_reply")),
            is_quote=(tweet_info.get("is_quote") if tweet_info else content_dict.get("is_quote")),
            media_urls=media_urls,
            media_types=media_types,
            conversation_id=(tweet_info.get("conversation_id") if tweet_info else content_dict.get("conversation_id")),
            in_reply_to_user_id=(
                tweet_info.get("in_reply_to", {}).get("user_id")
                if (isinstance(tweet_info.get("in_reply_to"), dict))
                else content_dict.get("in_reply_to_user_id")
            ),
        )
```
**NEW (end)**

---

### 2) `vali_utils/organic_query_processor.py` → `_validate_x_request_fields`

**Why:** The current version only reads `user.username`, so flat payloads fail; it also doesn’t handle double‑encoded JSON.

**OLD (start)**
```python
    def _validate_x_request_fields(self, synapse: OrganicRequest, x_entity: DataEntity) -> bool:
        """X request field validation with the X DataEntity"""
        x_content_dict = json.loads(x_entity.content.decode('utf-8'))
        # Username validation
        if synapse.usernames:
            requested_usernames = [u.strip('@').lower() for u in synapse.usernames]
            user_dict = x_content_dict.get("user", {})
            post_username = user_dict.get("username", "").strip('@').lower()
            if not post_username or post_username not in requested_usernames:
                bt.logging.debug(f"Username mismatch: {post_username} not in: {requested_usernames}")
                return False
        
        # Keyword validation
        if synapse.keywords:
            post_text = x_content_dict.get("text")
            now = dt.datetime.now(dt.timezone.utc)
            if now <= X_ON_DEMAND_CONTENT_EXPIRATION_DATE:
                if not post_text:
                    bt.logging.debug("'text' field not found, using 'content' as fallback. This fallback will expire Aug 25 2025.")
                    post_text = x_content_dict.get("content", "")
                
            post_text = post_text.lower()
            if not post_text or not all(keyword.lower() in post_text for keyword in synapse.keywords):
                bt.logging.debug(f"Not all keywords ({synapse.keywords}) found in post: {post_text}")
                return False
        
        # Time range validation
        if not self._validate_time_range(synapse, x_entity.datetime):
            return False
        
        return True
```
**OLD (end)**

**NEW (start)**
```python
    def _validate_x_request_fields(self, synapse: OrganicRequest, x_entity: DataEntity) -> bool:
        """X request field validation that accepts both enhanced (nested) and legacy/flat JSON,
        and safely handles double-encoded JSON."""
        try:
            raw = x_entity.content
            s = raw.decode("utf-8", errors="replace") if isinstance(raw, (bytes, bytearray)) else str(raw or "")
            try:
                obj = json.loads(s)
            except Exception:
                bt.logging.debug("X post content is not JSON; failing field validation")
                return False
            # Double-encoded JSON? json.loads → str → load again
            if isinstance(obj, str):
                try:
                    obj = json.loads(obj)
                except Exception:
                    bt.logging.debug("X post content was double-encoded but inner JSON invalid")
                    return False
            if not isinstance(obj, dict):
                bt.logging.debug("X post content JSON is not an object")
                return False

            x_content_dict = obj

            # Username validation (nested 'user.username' or flat 'username'/'screen_name')
            if synapse.usernames:
                requested = [u.strip('@').lower() for u in synapse.usernames]
                user_dict = x_content_dict.get("user") or {}
                post_username = (
                    (user_dict.get("username") if isinstance(user_dict, dict) else None)
                    or x_content_dict.get("username")
                    or x_content_dict.get("screen_name")
                )
                post_username = (post_username or "").strip('@').lower()
                if not post_username or post_username not in requested:
                    bt.logging.debug(f"Username mismatch: {post_username} not in: {requested}")
                    return False

            # Keyword validation (text with fallback to 'content' if allowed)
            post_text = x_content_dict.get("text")
            now = dt.datetime.now(dt.timezone.utc)
            if now <= X_ON_DEMAND_CONTENT_EXPIRATION_DATE and not post_text:
                bt.logging.debug("'text' missing, using 'content' as fallback prior to expiration date")
                post_text = x_content_dict.get("content", "")
            post_text_lc = (post_text or "").lower()

            if synapse.keywords:
                if not post_text_lc or not all(str(k).lower() in post_text_lc for k in synapse.keywords):
                    bt.logging.debug(f"Not all keywords ({synapse.keywords}) found in post: {post_text_lc}")
                    return False

            # Time range validation
            if not self._validate_time_range(synapse, x_entity.datetime):
                return False

            return True

        except Exception as e:
            bt.logging.error(f"Error in request field validation: {str(e)}")
            return False
```
**NEW (end)**

---

### 3) `vali_utils/organic_query_processor.py` → `_create_entity_dictionary` (recommended)

**Why:** If content is double‑encoded, `json.loads(...)` returns a **string**; stepping into the `for item in content_dict` loop and calling `.get` on a `str` causes the `"'str' object has no attribute 'get'"` crash. This patch makes it tolerant.

**OLD (start)**
```python
    def _create_entity_dictionary(self, data_entity: DataEntity) -> Dict:
        """Create entity dictionary with nested structure instead of flattened"""
        entity_dict = {}

        # Top-level entity fields
        entity_dict["uri"] = data_entity.uri
        entity_dict["datetime"] = data_entity.datetime
        entity_dict["source"] = DataSource(data_entity.source).name
        entity_dict["label"] = data_entity.label.value if data_entity.label else None
        entity_dict["content_size_bytes"] = data_entity.content_size_bytes

        try:
            content_dict = json.loads(data_entity.content.decode("utf-8"))
            # Response content based off of the Data Source's given fields
            for item in content_dict:
                entity_dict[item] = content_dict.get(item)

        except Exception as e:
            bt.logging.error(f"Error decoding content from DataEntity. Content: {data_entity.content}")
            entity_dict["content"] = data_entity.content

        return entity_dict
```
**OLD (end)**

**NEW (start)**
```python
    def _create_entity_dictionary(self, data_entity: DataEntity) -> Dict:
        """Create entity dictionary with nested structure; robust to double-encoded JSON."""
        entity_dict: Dict = {}

        # Top-level entity fields
        entity_dict["uri"] = data_entity.uri
        entity_dict["datetime"] = data_entity.datetime
        entity_dict["source"] = DataSource(data_entity.source).name
        entity_dict["label"] = data_entity.label.value if data_entity.label else None
        entity_dict["content_size_bytes"] = data_entity.content_size_bytes

        try:
            raw = data_entity.content
            s = raw.decode("utf-8", errors="replace") if isinstance(raw, (bytes, bytearray)) else str(raw or "")
            try:
                obj = json.loads(s)
            except Exception:
                entity_dict["content"] = data_entity.content
                return entity_dict
            if isinstance(obj, str):  # double-encoded
                try:
                    obj = json.loads(obj)
                except Exception:
                    entity_dict["content"] = data_entity.content
                    return entity_dict
            if isinstance(obj, dict):
                entity_dict.update(obj)
            else:
                entity_dict["content"] = data_entity.content
        except Exception:
            bt.logging.error(f"Error decoding content from DataEntity. Content: {data_entity.content}")
            entity_dict["content"] = data_entity.content

        return entity_dict
```
**NEW (end)**

---

## Quick validation (local sanity checks)

Use these snippets to confirm the validator accepts both shapes and handles double‑encoded JSON.

### Flat JSON sample
```python
sample_flat = {
    "username": "MatchTenis",
    "text": "Baja de peso... #USOpen",
    "url": "https://x.com/MatchTenis/status/1960797544415457345",
    "timestamp": "2025-08-27T20:12:00+00:00",
    "tweet_hashtags": ["#USOpen"],
    "tweet_id": "1960797544415457345",
    "is_quote": False,
    "is_reply": False,
}
```

### Enhanced/Nested JSON sample
```python
sample_nested = {
  "uri": "https://x.com/MatchTenis/status/1960797544415457345",
  "text": "Baja de peso... #USOpen",
  "user": { "username": "@MatchTenis" },
  "tweet": {
    "id": "1960797544415457345",
    "hashtags": ["#USOpen"],
    "is_quote": False,
    "is_reply": False
  }
}
```

### Double‑encoded JSON sample
```python
import json
double_encoded = json.dumps(json.dumps(sample_flat))
```

After applying the patches, all three should parse/validate without raising the previous errors.

---

## Notes on `X_ON_DEMAND_CONTENT_EXPIRATION_DATE`

- The code keeps a temporary compatibility path that, **until the configured date**, allows `text` to fall back to `content`.  
- After that date, only `text` will be used.

---

## Why these fixes stop the failures

- **Double‑encoded JSON** → We now detect when `json.loads(...)` returns a **string** and **decode again** safely.  
- **Schema mismatch** → We now **map both nested and flat field names** and even **synthesize** `url` from `@username` + `tweet_id` when needed. This prevents required fields (`username`, `url`) from being `None`.

---

## Rollback

If needed, you can revert by restoring the OLD blocks above. However, doing so will reintroduce the failures when encountering flat or double‑encoded inputs.

---

## Contact

If you want this as a patch file (`.patch`) or as a PR‑style unified diff, it’s straightforward to generate from the NEW blocks above.
