# Ad Isolation

Ad Isolation prevents ad content from contaminating your AI model's context. Users still see ads in the UI. The AI never processes them.

## The Problem

When AI chatbots display ads inline with conversation, those ads become part of the conversation history. On the next turn, the model processes the ad text as if it were real context. This creates two problems:

1. **Prompt injection via ads** -- Attackers can embed instructions in ad copy that the model follows on subsequent turns
2. **Context pollution** -- Ad text wastes tokens and degrades response quality

## How It Works

Ad Isolation uses a tag-and-strip approach. Your application tags ad content when inserting it. Guardian strips tagged content before it reaches the model.

```
User sees:    "Here are some shoes. [Nike Ad - 20% off!] Want to know more?"
Model sees:   "Here are some shoes. [ad content removed] Want to know more?"
```

The ad is visible to the user but invisible to the AI. We call this the "glass door" -- the user walks through, the model hits glass.

### Supported Tag Formats

```html
<guardian-ad>Your ad here</guardian-ad>     (recommended)
<sponsored>Your ad here</sponsored>
<ad>Your ad here</ad>
<promoted>Your ad here</promoted>
[ad]Your ad here[/ad]
[sponsored]Your ad here[/sponsored]
<!-- ad-start -->Your ad here<!-- ad-end -->
```

You can also register custom tags via the API.

### What Gets Stripped

- The tag and everything between the opening and closing tag
- Self-closing tags (`<ad/>`)
- Unclosed tags (content from the tag to end of line)
- Orphaned closing tags

### Evasion Protection

Ad Isolation includes the same hardening as our prompt injection scanner:

- Unicode normalization (fullwidth characters, combining marks)
- Zero-width character stripping
- Homoglyph detection (Cyrillic/Greek lookalikes)
- HTML entity decoding
- Whitespace normalization inside tags
- DoS protection (bulk tag flooding)

If someone tries to sneak an ad tag past the filter using invisible characters or lookalike letters, Guardian catches it.

## Integration

### Python SDK

```python
from fas_guardian import Guardian

guardian = Guardian(api_key="fsg_your_key_here")

# Isolate ads from a single message
result = guardian.isolate("Check this out! <sponsored>Buy now!</sponsored> Pretty cool right?")
print(result.cleaned)
# "Check this out! [ad content removed] Pretty cool right?"

# Isolate ads from full conversation history
messages = [
    {"role": "user", "content": "Tell me about running shoes"},
    {"role": "assistant", "content": "Great question! <sponsored>Nike sale!</sponsored> Here are some options..."},
]
result = guardian.isolate_conversation(messages)
# Returns cleaned messages with all ad content stripped
```

### REST API

```bash
curl -X POST https://api.fallenangelsystems.com/v2/isolate \
  -H "Authorization: Bearer fsg_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{"text": "Hello <sponsored>Buy stuff!</sponsored> world"}'
```

Response:

```json
{
  "cleaned": "Hello [ad content removed] world",
  "ads_found": 1,
  "tags_matched": ["sponsored"],
  "original_length": 47,
  "cleaned_length": 30,
  "bytes_removed": 17
}
```

### Custom Tags

```python
result = guardian.isolate(text, custom_tags=["promo", "banner"])
```

Custom tag names must be alphanumeric (with hyphens). Guardian automatically generates HTML-style, BBCode-style, and self-closing patterns for each custom tag.

### Custom Placeholder

```python
result = guardian.isolate(text, placeholder="[removed]")
```

Placeholders are validated to prevent injection. Tags, code, and excessively long strings are rejected and replaced with the default `[ad content removed]`.

## What Ad Isolation Does NOT Do

- **It does not detect ads.** Your application tags the ads. You already know what content is an ad because you're the one inserting it.
- **It does not block ads from the user.** Users see everything. Only the AI model's input is filtered.
- **It is not an ad blocker.** This is an AI context protection tool for companies that serve ads alongside AI responses.
- **It does not modify the UI.** It only operates on the text stream sent to the model.

## Who Is This For

Ad Isolation is for companies building AI products that include advertising. If you display ads in or near AI-generated content, Ad Isolation prevents those ads from becoming attack vectors or degrading your model's performance.

Examples:
- AI chatbots with inline sponsored messages
- AI assistants that browse web pages containing ads
- Any pipeline where ad content might end up in model context

## Availability

Ad Isolation is included in **Pro** and **Enterprise** plans. Basic plan customers can upgrade to access it.

| Feature | Basic | Pro | Enterprise |
|---------|-------|-----|------------|
| Prompt injection scanning | Yes | Yes | Yes |
| Ad Isolation | -- | Yes | Yes |
| Custom ad tags | -- | Yes | Yes |
| Conversation-level isolation | -- | Yes | Yes |
| Custom isolation policies | -- | -- | Yes |
