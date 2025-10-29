# NIP-33: Parameterized Replaceable Events (Addressable Events)

**Technical Report on Nostr's Addressable Event System**

*Last Updated: October 2024*

---

## Executive Summary

NIP-33 defines a class of Nostr events called "Parameterized Replaceable Events," now commonly referred to as "Addressable Events." This specification has been merged into NIP-01 as a core component of the Nostr protocol. These events enable applications to maintain current state rather than append-only history, making them ideal for profiles, status updates, application settings, and other data that should be replaced rather than accumulated.

---

## 1. Introduction

### 1.1 Background

The Nostr protocol was initially designed around immutable, append-only events. While this works well for social media posts and messages, many applications require the ability to update existing data. NIP-33 addresses this need by introducing a standardized mechanism for replaceable events with unique identifiers.

### 1.2 Problem Statement

Traditional Nostr events (kinds 0-29999) are either:
- **Regular events**: Immutable and append-only
- **Replaceable events** (kinds 0, 3, 10000-19999): Can be replaced, but only one per kind per author

This limitation meant applications couldn't maintain multiple pieces of replaceable data of the same type. For example, a user couldn't have multiple editable lists or multiple status messages for different contexts.

---

## 2. Technical Specification

### 2.1 Event Kind Range

Parameterized replaceable events use kind numbers in the range:

```
30000 ≤ kind < 40000
```

Any event with a kind in this range is automatically treated as a parameterized replaceable event by compliant relays and clients.

### 2.2 The "d" Tag Identifier

The critical component of NIP-33 is the **"d" tag** (identifier tag). This tag serves as a unique identifier within the scope of the event kind and author.

**Structure:**
```json
["d", "<identifier>"]
```

**Example:**
```json
{
  "kind": 30078,
  "tags": [
    ["d", "current-status"]
  ],
  "content": "Working on Nostr development",
  "pubkey": "abc123...",
  "created_at": 1698000000,
  "id": "def456...",
  "sig": "789ghi..."
}
```

### 2.3 Replacement Logic

A relay MUST replace an older event with a newer one if and only if:

1. Both events have the same `kind`
2. Both events have the same `pubkey` (author)
3. Both events have the same `d` tag value
4. The new event has a higher `created_at` timestamp

**Pseudocode:**
```python
def should_replace(existing_event, new_event):
    return (
        existing_event.kind == new_event.kind and
        existing_event.pubkey == new_event.pubkey and
        existing_event.get_d_tag() == new_event.get_d_tag() and
        new_event.created_at > existing_event.created_at
    )
```

### 2.4 Addressable Event Identifier

Events can be uniquely addressed using the format:

```
<kind>:<pubkey>:<d-tag-value>
```

**Example:**
```
30078:3bf0c63fcb93463407af97a5e5ee64fa883d107ef9e558472c4eb9aaaefa459d:current-status
```

This addressing scheme allows clients to:
- Request specific events from relays
- Reference events without knowing their event ID
- Construct URLs and deep links

---

## 3. Common Use Cases

### 3.1 User Status Updates

**Kind:** 30078 (Application-specific data)

```json
{
  "kind": 30078,
  "tags": [
    ["d", "status"]
  ],
  "content": "🚀 Shipping new features",
  "created_at": 1698000000
}
```

### 3.2 Long-form Content (NIP-23)

**Kind:** 30023 (Long-form content)

```json
{
  "kind": 30023,
  "tags": [
    ["d", "my-first-blog-post"],
    ["title", "Understanding NIP-33"],
    ["published_at", "1698000000"],
    ["t", "nostr"],
    ["t", "protocol"]
  ],
  "content": "# Understanding NIP-33\n\nThis is a long-form article...",
  "created_at": 1698000000
}
```

### 3.3 Application Settings

**Kind:** 30078

```json
{
  "kind": 30078,
  "tags": [
    ["d", "app-settings"],
    ["theme", "dark"],
    ["language", "en"]
  ],
  "content": "{\"notifications\": true, \"autoplay\": false}",
  "created_at": 1698000000
}
```

### 3.4 Product Listings (NIP-15)

**Kind:** 30017 (Product listing)

```json
{
  "kind": 30017,
  "tags": [
    ["d", "product-abc-123"],
    ["title", "Vintage Nostr T-Shirt"],
    ["price", "21000", "sats"],
    ["image", "https://example.com/shirt.jpg"]
  ],
  "content": "Limited edition Nostr t-shirt from 2023",
  "created_at": 1698000000
}
```

---

## 4. Implementation Guide

### 4.1 Creating an Addressable Event

**Step 1: Define the event structure**

```javascript
const unsignedEvent = {
  kind: 30078,              // Parameterized replaceable kind
  created_at: Math.floor(Date.now() / 1000),
  tags: [
    ["d", "my-identifier"]  // Required: unique identifier
  ],
  content: "Event content here"
};
```

**Step 2: Sign the event**

```javascript
// Using window.nostr (browser extension)
const signedEvent = await window.nostr.signEvent(unsignedEvent);

// Or using a library like nostr-tools
import { getEventHash, signEvent } from 'nostr-tools';

unsignedEvent.pubkey = myPublicKey;
unsignedEvent.id = getEventHash(unsignedEvent);
unsignedEvent.sig = signEvent(unsignedEvent, myPrivateKey);
```

**Step 3: Publish to relays**

```javascript
const relay = new WebSocket('wss://relay.example.com');

relay.onopen = () => {
  const message = JSON.stringify(['EVENT', signedEvent]);
  relay.send(message);
};

relay.onmessage = (event) => {
  const [type, eventId, accepted, message] = JSON.parse(event.data);
  if (type === 'OK' && accepted) {
    console.log('Event published successfully');
  }
};
```

### 4.2 Querying Addressable Events

**By address (recommended):**

```javascript
const filter = {
  kinds: [30078],
  authors: ["3bf0c63fcb93463407af97a5e5ee64fa883d107ef9e558472c4eb9aaaefa459d"],
  "#d": ["current-status"]
};

const subscription = JSON.stringify(['REQ', subscriptionId, filter]);
relay.send(subscription);
```

**By kind and author:**

```javascript
const filter = {
  kinds: [30078],
  authors: ["3bf0c63fcb93463407af97a5e5ee64fa883d107ef9e558472c4eb9aaaefa459d"]
};
```

### 4.3 Relay Implementation Considerations

Relays implementing NIP-33 must:

1. **Index by composite key**: Store events indexed by `(kind, pubkey, d-tag)`
2. **Replace on duplicate**: When receiving an event with an existing address, compare timestamps
3. **Maintain only latest**: Only store the most recent event for each unique address
4. **Support tag queries**: Enable filtering by `#d` tag in REQ messages

**Database schema example:**

```sql
CREATE TABLE addressable_events (
  kind INTEGER NOT NULL,
  pubkey TEXT NOT NULL,
  d_tag TEXT NOT NULL,
  created_at INTEGER NOT NULL,
  event_id TEXT NOT NULL,
  content TEXT,
  tags JSON,
  sig TEXT NOT NULL,
  PRIMARY KEY (kind, pubkey, d_tag)
);

CREATE INDEX idx_created_at ON addressable_events(created_at);
CREATE INDEX idx_kind ON addressable_events(kind);
```

---

## 5. Comparison with Other Event Types

| Feature | Regular Events | Replaceable Events | Addressable Events |
|---------|---------------|-------------------|-------------------|
| **Kind Range** | 1000-9999, 20000-29999 | 0, 3, 10000-19999 | 30000-39999 |
| **Mutability** | Immutable | Replaceable | Replaceable |
| **Identifier** | Event ID only | Kind + Pubkey | Kind + Pubkey + d-tag |
| **Multiple per kind** | Unlimited | One per author | Unlimited per author |
| **Use Case** | Posts, messages | Profile, contact list | Articles, settings, listings |

---

## 6. Security Considerations

### 6.1 Timestamp Manipulation

**Risk:** Malicious actors could set future timestamps to prevent updates.

**Mitigation:** Relays should reject events with timestamps too far in the future (e.g., > 10 minutes).

```javascript
const MAX_FUTURE_DRIFT = 600; // 10 minutes

function isValidTimestamp(event) {
  const now = Math.floor(Date.now() / 1000);
  return event.created_at <= now + MAX_FUTURE_DRIFT;
}
```

### 6.2 Identifier Collisions

**Risk:** Using common or predictable `d` tag values could lead to unintended replacements.

**Best Practice:** Use descriptive, unique identifiers:

```javascript
// ❌ Bad: Generic identifier
["d", "post"]

// ✅ Good: Specific identifier
["d", "blog-post-understanding-nip33-2024-10-29"]

// ✅ Good: UUID for uniqueness
["d", "550e8400-e29b-41d4-a716-446655440000"]
```

### 6.3 Relay Trust

**Risk:** Malicious relays could refuse to replace events or serve outdated versions.

**Mitigation:** 
- Query multiple relays
- Verify timestamps client-side
- Use trusted relay networks

---

## 7. Real-World Applications

### 7.1 Blogging Platforms

Platforms like Habla.news and Yakihonne use NIP-33 (kind 30023) for long-form content, allowing authors to edit articles while maintaining a stable address.

### 7.2 Marketplaces

Nostr marketplaces use kinds 30017-30018 for product listings that can be updated as inventory changes.

### 7.3 Profile Extensions

Applications extend user profiles beyond NIP-01's kind 0 using addressable events for additional metadata, preferences, and settings.

### 7.4 Live Events

Event organizers use addressable events to update event details, schedules, and announcements while maintaining a consistent reference.

---

## 8. Best Practices

### 8.1 Choosing the Right Event Type

```
┌─────────────────────────────────────────┐
│ Does the data need to be updated?      │
└─────────────┬───────────────────────────┘
              │
              ├─ No ──→ Use regular event (kind 1000-9999)
              │
              └─ Yes ──┬─ Only one per author?
                       │
                       ├─ Yes ──→ Use replaceable event (kind 10000-19999)
                       │
                       └─ No ───→ Use addressable event (kind 30000-39999)
```

### 8.2 Naming Conventions for "d" Tags

1. **Use kebab-case**: `my-blog-post` instead of `myBlogPost`
2. **Be descriptive**: `settings-theme` instead of `st`
3. **Include context**: `article-nip33-guide` instead of `article`
4. **Avoid special characters**: Stick to alphanumeric and hyphens

### 8.3 Content Organization

```json
{
  "kind": 30078,
  "tags": [
    ["d", "app-config"],
    ["version", "1.0"],
    ["updated", "2024-10-29"]
  ],
  "content": "{\"structured\": \"data\", \"goes\": \"here\"}"
}
```

**Recommendation:** Use tags for queryable metadata, content for bulk data.

---

## 9. Testing Your Implementation

### 9.1 Test Event Creation

```javascript
async function testAddressableEvent() {
  const event = {
    kind: 30078,
    created_at: Math.floor(Date.now() / 1000),
    tags: [["d", "test-event"]],
    content: "Test content"
  };
  
  const signed = await window.nostr.signEvent(event);
  console.log('Signed event:', signed);
  
  // Verify structure
  assert(signed.kind === 30078);
  assert(signed.tags.some(t => t[0] === 'd'));
  assert(signed.id && signed.sig);
}
```

### 9.2 Test Replacement Logic

```javascript
async function testReplacement() {
  // Create first event
  const event1 = await createAndSign({
    kind: 30078,
    tags: [["d", "test"]],
    content: "Version 1"
  });
  
  await publishToRelay(event1);
  await sleep(1000);
  
  // Create replacement
  const event2 = await createAndSign({
    kind: 30078,
    tags: [["d", "test"]],
    content: "Version 2"
  });
  
  await publishToRelay(event2);
  
  // Query relay
  const result = await queryRelay({
    kinds: [30078],
    "#d": ["test"]
  });
  
  // Should only return latest
  assert(result.length === 1);
  assert(result[0].content === "Version 2");
}
```

---

## 10. Future Developments

### 10.1 Potential Enhancements

1. **Versioning**: Maintaining event history while still supporting replacement
2. **Partial updates**: Updating specific fields without resending entire content
3. **Collaborative editing**: Multiple authors for single addressable events
4. **Expiration**: Automatic deletion of addressable events after a time period

### 10.2 Related NIPs

- **NIP-01**: Basic protocol flow (now includes NIP-33)
- **NIP-23**: Long-form content (uses kind 30023)
- **NIP-51**: Lists (uses kinds 30000-30003)
- **NIP-72**: Moderated communities (uses kind 34550)

---

## 11. Conclusion

NIP-33 Addressable Events represent a crucial evolution in the Nostr protocol, enabling stateful applications while maintaining the protocol's decentralized nature. By providing a standardized mechanism for replaceable, identifiable events, NIP-33 unlocks use cases ranging from blogging platforms to marketplaces to application settings.

The elegant design—using kind ranges and a simple "d" tag—demonstrates Nostr's philosophy of minimal, composable specifications. As the ecosystem grows, addressable events will continue to be a foundational building block for sophisticated applications.

---

## 12. References

- [NIP-01: Basic Protocol Flow](https://github.com/nostr-protocol/nips/blob/master/01.md)
- [NIP-33: Parameterized Replaceable Events](https://github.com/nostr-protocol/nips/blob/master/33.md)
- [NIP-23: Long-form Content](https://github.com/nostr-protocol/nips/blob/master/23.md)
- [Nostr Protocol Documentation](https://nostr.com)

---

## Appendix A: Complete Example

```javascript
/**
 * Complete example: Creating and publishing an addressable event
 */

class AddressableEventManager {
  constructor(relayUrl) {
    this.relayUrl = relayUrl;
    this.relay = null;
  }
  
  async connect() {
    return new Promise((resolve, reject) => {
      this.relay = new WebSocket(this.relayUrl);
      this.relay.onopen = () => resolve();
      this.relay.onerror = (err) => reject(err);
    });
  }
  
  async createEvent(identifier, content, additionalTags = []) {
    const event = {
      kind: 30078,
      created_at: Math.floor(Date.now() / 1000),
      tags: [
        ["d", identifier],
        ...additionalTags
      ],
      content: content
    };
    
    // Sign with browser extension
    return await window.nostr.signEvent(event);
  }
  
  async publish(event) {
    return new Promise((resolve, reject) => {
      const message = JSON.stringify(['EVENT', event]);
      
      const handler = (msg) => {
        const [type, eventId, accepted, reason] = JSON.parse(msg.data);
        if (type === 'OK') {
          this.relay.removeEventListener('message', handler);
          if (accepted) {
            resolve({ success: true, eventId });
          } else {
            reject(new Error(reason));
          }
        }
      };
      
      this.relay.addEventListener('message', handler);
      this.relay.send(message);
      
      // Timeout after 5 seconds
      setTimeout(() => {
        this.relay.removeEventListener('message', handler);
        reject(new Error('Publish timeout'));
      }, 5000);
    });
  }
  
  async query(filter) {
    return new Promise((resolve) => {
      const subId = Math.random().toString(36).substring(7);
      const events = [];
      
      const handler = (msg) => {
        const data = JSON.parse(msg.data);
        if (data[0] === 'EVENT' && data[1] === subId) {
          events.push(data[2]);
        } else if (data[0] === 'EOSE' && data[1] === subId) {
          this.relay.removeEventListener('message', handler);
          resolve(events);
        }
      };
      
      this.relay.addEventListener('message', handler);
      this.relay.send(JSON.stringify(['REQ', subId, filter]));
    });
  }
  
  getAddress(event) {
    const dTag = event.tags.find(t => t[0] === 'd')?.[1] || '';
    return `${event.kind}:${event.pubkey}:${dTag}`;
  }
}

// Usage
async function main() {
  const manager = new AddressableEventManager('wss://nos.lol/');
  await manager.connect();
  
  // Create and publish
  const event = await manager.createEvent(
    'my-status',
    'Building on Nostr! 🚀',
    [['emoji', '🚀']]
  );
  
  console.log('Address:', manager.getAddress(event));
  
  await manager.publish(event);
  console.log('Published successfully!');
  
  // Query back
  const results = await manager.query({
    kinds: [30078],
    '#d': ['my-status'],
    authors: [event.pubkey]
  });
  
  console.log('Retrieved:', results);
}
```

---

**Document Version:** 1.0  
**Author:** Plebeius Garagicus  
**Date:** October 29, 2024  
**License:** Public Domain
