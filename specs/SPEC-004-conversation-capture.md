# SPEC-004: Conversation Capture

**Project:** alexandria-core  
**Author:** Claude (Designer) via claude.ai  
**For:** Claude Code (Engineer) on Linux Server  
**Captain:** Sean  
**Date:** January 23, 2026  
**Status:** Ready for Implementation  
**Depends on:** SPEC-002 (complete), SPEC-003 (patterns referenced)

---

## Context

We can now embed text and ingest books. The final piece: capturing conversations so the library remembers what we've discussed. This is what makes context compound across sessions.

When Sean asks "What did we decide about the embedding architecture?" six months from now, the system should find that conversation.

---

## This Spec's Goal

Build a system that:
1. Stores conversations in `memory.conversations`
2. Embeds conversation content for semantic search
3. Groups messages into sessions
4. Generates session summaries
5. Provides CLI for manual capture and search
6. (Optional) Imports Claude.ai chat exports

---

## Architecture Overview

```
Conversation Input
    │
    ▼
┌─────────────────┐
│ Store Messages  │  (memory.conversations)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Embed Content   │  (memory.embeddings, source_type='conversation')
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Session Summary │  (memory.sessions - AI-generated)
└─────────────────┘
```

---

## Lessons from SPEC-003 Implementation

Engineer identified patterns to follow:

1. **Vector formatting**: Format embeddings as pgvector string literals
   ```python
   embedding_str = "[" + ",".join(str(x) for x in embedding) + "]"
   ```

2. **ivfflat.probes**: Set before search queries
   ```python
   cursor.execute("SET ivfflat.probes = 100")
   ```

3. **Text preprocessing**: Clean text before embedding
   ```python
   text = re.sub(r'\.{3,}', '...', text)  # Collapse consecutive dots
   text = re.sub(r'\s+', ' ', text)        # Normalize whitespace
   ```

Apply these patterns throughout this spec.

---

## Requirement 1: Conversation Service

Create `/opt/alexandria/src/conversations.py`:

```python
import os
import re
import uuid
import psycopg2
from psycopg2.extras import Json
from datetime import datetime
from typing import Optional
from dotenv import load_dotenv

load_dotenv("/opt/alexandria/.env")

from embeddings import EmbeddingService

class ConversationService:
    """
    Manages conversation storage, embedding, and retrieval.
    
    Conversations are stored as individual messages grouped by session.
    Each message is embedded for semantic search.
    Sessions can have AI-generated summaries.
    """
    
    def __init__(self):
        self.embedding_service = EmbeddingService()
        self.db_config = {
            "host": "localhost",
            "port": 5433,
            "database": "alexandria",
            "user": "alexandria",
            "password": os.getenv("POSTGRES_PASSWORD")
        }
    
    def _clean_text_for_embedding(self, text: str) -> str:
        """Clean text before embedding to avoid Ollama issues."""
        # Collapse runs of 3+ dots to ellipsis
        text = re.sub(r'\.{3,}', '...', text)
        # Collapse runs of dots with spaces
        text = re.sub(r'(\.\s*){3,}', '... ', text)
        # Normalize whitespace
        text = re.sub(r'\s+', ' ', text)
        return text.strip()
    
    def _format_embedding(self, embedding: list[float]) -> str:
        """Format embedding as pgvector string literal."""
        return "[" + ",".join(str(x) for x in embedding) + "]"
    
    def create_session(
        self,
        participant: str,
        metadata: Optional[dict] = None
    ) -> str:
        """
        Create a new conversation session.
        
        Returns: session_id (UUID string)
        """
        conn = psycopg2.connect(**self.db_config)
        cursor = conn.cursor()
        
        session_id = str(uuid.uuid4())
        
        cursor.execute("""
            INSERT INTO memory.sessions (id, participant, metadata)
            VALUES (%s, %s, %s)
        """, (session_id, participant, Json(metadata or {})))
        
        conn.commit()
        conn.close()
        
        return session_id
    
    def add_message(
        self,
        content: str,
        role: str,
        participant: str,
        session_id: Optional[str] = None,
        metadata: Optional[dict] = None,
        embed: bool = True
    ) -> int:
        """
        Add a message to a conversation.
        
        Args:
            content: The message text
            role: 'user', 'assistant', or 'system'
            participant: Who sent this ('sean', 'claude_desktop', 'claude_code', etc.)
            session_id: Optional session to group messages
            metadata: Optional additional data
            embed: Whether to create an embedding (default True)
        
        Returns: message_id
        """
        conn = psycopg2.connect(**self.db_config)
        cursor = conn.cursor()
        
        # Store the message
        cursor.execute("""
            INSERT INTO memory.conversations 
            (participant, role, content, session_id, metadata)
            VALUES (%s, %s, %s, %s, %s)
            RETURNING id
        """, (
            participant,
            role,
            content,
            session_id,
            Json(metadata or {})
        ))
        
        message_id = cursor.fetchone()[0]
        
        # Create embedding if requested
        if embed and content.strip():
            clean_content = self._clean_text_for_embedding(content)
            embedding = self.embedding_service.generate_embedding(clean_content)
            embedding_str = self._format_embedding(embedding)
            
            preview = content[:500] if len(content) > 500 else content
            
            cursor.execute("""
                INSERT INTO memory.embeddings
                (source_type, source_id, content_preview, embedding, metadata)
                VALUES (%s, %s, %s, %s, %s)
            """, (
                'conversation',
                message_id,
                preview,
                embedding_str,
                Json({
                    "participant": participant,
                    "role": role,
                    "session_id": session_id,
                    "model": "nomic-embed-text"
                })
            ))
        
        conn.commit()
        conn.close()
        
        return message_id
    
    def add_exchange(
        self,
        user_message: str,
        assistant_message: str,
        user_participant: str = "sean",
        assistant_participant: str = "claude",
        session_id: Optional[str] = None,
        metadata: Optional[dict] = None
    ) -> dict:
        """
        Add a user-assistant exchange (convenience method).
        
        Returns: {"user_id": int, "assistant_id": int}
        """
        user_id = self.add_message(
            content=user_message,
            role="user",
            participant=user_participant,
            session_id=session_id,
            metadata=metadata
        )
        
        assistant_id = self.add_message(
            content=assistant_message,
            role="assistant",
            participant=assistant_participant,
            session_id=session_id,
            metadata=metadata
        )
        
        return {"user_id": user_id, "assistant_id": assistant_id}
    
    def end_session(
        self,
        session_id: str,
        summary: Optional[str] = None
    ):
        """
        Mark a session as ended, optionally with a summary.
        """
        conn = psycopg2.connect(**self.db_config)
        cursor = conn.cursor()
        
        cursor.execute("""
            UPDATE memory.sessions
            SET ended_at = NOW(), summary = %s
            WHERE id = %s
        """, (summary, session_id))
        
        conn.commit()
        conn.close()
    
    def search_conversations(
        self,
        query: str,
        limit: int = 5,
        participant: Optional[str] = None,
        session_id: Optional[str] = None
    ) -> list[dict]:
        """
        Semantic search across conversation history.
        
        Optionally filter by participant or session.
        """
        clean_query = self._clean_text_for_embedding(query)
        query_embedding = self.embedding_service.generate_embedding(clean_query)
        query_embedding_str = self._format_embedding(query_embedding)
        
        conn = psycopg2.connect(**self.db_config)
        cursor = conn.cursor()
        
        # Set probes for accurate search with ivfflat
        cursor.execute("SET ivfflat.probes = 100")
        
        # Build query with optional filters
        where_clauses = ["e.source_type = 'conversation'"]
        params = [query_embedding_str]
        
        if participant:
            where_clauses.append("e.metadata->>'participant' = %s")
            params.append(participant)
        
        if session_id:
            where_clauses.append("e.metadata->>'session_id' = %s")
            params.append(session_id)
        
        params.append(limit)
        
        cursor.execute(f"""
            SELECT 
                e.id,
                e.source_id,
                e.content_preview,
                e.embedding <=> %s::vector AS distance,
                e.metadata,
                c.role,
                c.participant,
                c.session_id,
                c.created_at
            FROM memory.embeddings e
            JOIN memory.conversations c ON e.source_id = c.id
            WHERE {" AND ".join(where_clauses)}
            ORDER BY distance
            LIMIT %s
        """, params)
        
        results = []
        for row in cursor.fetchall():
            results.append({
                "embedding_id": row[0],
                "message_id": row[1],
                "preview": row[2],
                "distance": row[3],
                "metadata": row[4],
                "role": row[5],
                "participant": row[6],
                "session_id": row[7],
                "created_at": row[8]
            })
        
        conn.close()
        return results
    
    def get_session_messages(self, session_id: str) -> list[dict]:
        """Get all messages in a session, ordered by time."""
        conn = psycopg2.connect(**self.db_config)
        cursor = conn.cursor()
        
        cursor.execute("""
            SELECT id, participant, role, content, created_at, metadata
            FROM memory.conversations
            WHERE session_id = %s
            ORDER BY created_at
        """, (session_id,))
        
        messages = []
        for row in cursor.fetchall():
            messages.append({
                "id": row[0],
                "participant": row[1],
                "role": row[2],
                "content": row[3],
                "created_at": row[4],
                "metadata": row[5]
            })
        
        conn.close()
        return messages
    
    def list_sessions(
        self,
        participant: Optional[str] = None,
        limit: int = 20
    ) -> list[dict]:
        """List recent sessions."""
        conn = psycopg2.connect(**self.db_config)
        cursor = conn.cursor()
        
        if participant:
            cursor.execute("""
                SELECT id, participant, started_at, ended_at, summary, metadata
                FROM memory.sessions
                WHERE participant = %s
                ORDER BY started_at DESC
                LIMIT %s
            """, (participant, limit))
        else:
            cursor.execute("""
                SELECT id, participant, started_at, ended_at, summary, metadata
                FROM memory.sessions
                ORDER BY started_at DESC
                LIMIT %s
            """, (limit,))
        
        sessions = []
        for row in cursor.fetchall():
            sessions.append({
                "id": row[0],
                "participant": row[1],
                "started_at": row[2],
                "ended_at": row[3],
                "summary": row[4],
                "metadata": row[5]
            })
        
        conn.close()
        return sessions
    
    def get_recent_context(
        self,
        limit: int = 10,
        participant: Optional[str] = None
    ) -> list[dict]:
        """
        Get recent conversation messages for context.
        
        Useful for providing conversation history to an AI.
        """
        conn = psycopg2.connect(**self.db_config)
        cursor = conn.cursor()
        
        if participant:
            cursor.execute("""
                SELECT id, participant, role, content, created_at
                FROM memory.conversations
                WHERE participant = %s
                ORDER BY created_at DESC
                LIMIT %s
            """, (participant, limit))
        else:
            cursor.execute("""
                SELECT id, participant, role, content, created_at
                FROM memory.conversations
                ORDER BY created_at DESC
                LIMIT %s
            """, (limit,))
        
        # Reverse to get chronological order
        messages = []
        for row in cursor.fetchall():
            messages.append({
                "id": row[0],
                "participant": row[1],
                "role": row[2],
                "content": row[3],
                "created_at": row[4]
            })
        
        conn.close()
        return list(reversed(messages))


class ConversationImporter:
    """
    Import conversations from external sources (e.g., Claude.ai exports).
    """
    
    def __init__(self):
        self.conversation_service = ConversationService()
    
    def import_claude_export(self, json_path: str) -> dict:
        """
        Import a Claude.ai conversation export.
        
        Claude.ai exports are JSON files with conversation data.
        
        Returns: {"sessions_created": int, "messages_imported": int}
        """
        import json
        
        with open(json_path, 'r', encoding='utf-8') as f:
            data = json.load(f)
        
        sessions_created = 0
        messages_imported = 0
        
        # Claude.ai export format (may vary - adjust as needed)
        # Expected: list of conversations, each with messages
        conversations = data if isinstance(data, list) else data.get('conversations', [data])
        
        for conv in conversations:
            # Create session for this conversation
            session_id = self.conversation_service.create_session(
                participant="claude_ai_import",
                metadata={
                    "source": "claude.ai",
                    "import_date": datetime.now().isoformat(),
                    "original_id": conv.get('uuid', conv.get('id', 'unknown'))
                }
            )
            sessions_created += 1
            
            # Import messages
            messages = conv.get('chat_messages', conv.get('messages', []))
            
            for msg in messages:
                # Handle different export formats
                role = msg.get('sender', msg.get('role', 'unknown'))
                if role == 'human':
                    role = 'user'
                elif role == 'assistant':
                    role = 'assistant'
                
                content = msg.get('text', msg.get('content', ''))
                
                # Handle content that might be a list of blocks
                if isinstance(content, list):
                    content = '\n'.join(
                        block.get('text', str(block)) 
                        for block in content 
                        if isinstance(block, dict)
                    )
                
                if content:
                    self.conversation_service.add_message(
                        content=content,
                        role=role,
                        participant="claude_ai_import",
                        session_id=session_id,
                        metadata={
                            "imported": True,
                            "original_timestamp": msg.get('created_at', msg.get('timestamp'))
                        }
                    )
                    messages_imported += 1
            
            # End the session
            self.conversation_service.end_session(
                session_id,
                summary=f"Imported from Claude.ai ({len(messages)} messages)"
            )
        
        return {
            "sessions_created": sessions_created,
            "messages_imported": messages_imported
        }
```

---

## Requirement 2: CLI Tool

Create `/opt/alexandria/src/convo_cli.py`:

```python
#!/usr/bin/env python3
"""CLI for conversation operations."""

import argparse
import sys
import os
from datetime import datetime

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))

from dotenv import load_dotenv
load_dotenv("/opt/alexandria/.env")

from conversations import ConversationService, ConversationImporter

def main():
    parser = argparse.ArgumentParser(description="Alexandria Conversation CLI")
    subparsers = parser.add_subparsers(dest="command")
    
    # Add message
    add_parser = subparsers.add_parser("add", help="Add a single message")
    add_parser.add_argument("content", help="Message content")
    add_parser.add_argument("--role", choices=["user", "assistant", "system"], default="user")
    add_parser.add_argument("--participant", default="sean")
    add_parser.add_argument("--session", help="Session ID to add to")
    
    # Add exchange (user + assistant pair)
    exchange_parser = subparsers.add_parser("exchange", help="Add a user-assistant exchange")
    exchange_parser.add_argument("--user", required=True, help="User message")
    exchange_parser.add_argument("--assistant", required=True, help="Assistant response")
    exchange_parser.add_argument("--session", help="Session ID")
    
    # Search
    search_parser = subparsers.add_parser("search", help="Search conversations")
    search_parser.add_argument("query", help="Search query")
    search_parser.add_argument("--limit", type=int, default=5)
    search_parser.add_argument("--participant", help="Filter by participant")
    
    # List sessions
    list_parser = subparsers.add_parser("sessions", help="List sessions")
    list_parser.add_argument("--participant", help="Filter by participant")
    list_parser.add_argument("--limit", type=int, default=20)
    
    # Show session
    show_parser = subparsers.add_parser("show", help="Show session messages")
    show_parser.add_argument("session_id", help="Session ID to show")
    
    # New session
    new_parser = subparsers.add_parser("new", help="Create new session")
    new_parser.add_argument("--participant", default="sean")
    
    # Import
    import_parser = subparsers.add_parser("import", help="Import conversations")
    import_parser.add_argument("file", help="JSON file to import")
    import_parser.add_argument("--format", choices=["claude"], default="claude", 
                               help="Import format")
    
    # Recent context
    recent_parser = subparsers.add_parser("recent", help="Show recent messages")
    recent_parser.add_argument("--limit", type=int, default=10)
    recent_parser.add_argument("--participant", help="Filter by participant")
    
    args = parser.parse_args()
    service = ConversationService()
    
    if args.command == "add":
        msg_id = service.add_message(
            content=args.content,
            role=args.role,
            participant=args.participant,
            session_id=args.session
        )
        print(f"Added message {msg_id}")
    
    elif args.command == "exchange":
        result = service.add_exchange(
            user_message=args.user,
            assistant_message=args.assistant,
            session_id=args.session
        )
        print(f"Added exchange: user={result['user_id']}, assistant={result['assistant_id']}")
    
    elif args.command == "search":
        results = service.search_conversations(
            query=args.query,
            limit=args.limit,
            participant=args.participant
        )
        
        if not results:
            print("No results found.")
            return
        
        for r in results:
            timestamp = r['created_at'].strftime('%Y-%m-%d %H:%M') if r['created_at'] else '?'
            print(f"\n[{r['participant']}/{r['role']}] {timestamp} (distance: {r['distance']:.4f})")
            print(f"  {r['preview'][:300]}...")
    
    elif args.command == "sessions":
        sessions = service.list_sessions(
            participant=args.participant,
            limit=args.limit
        )
        
        if not sessions:
            print("No sessions found.")
            return
        
        print(f"{'ID':<36}  {'Participant':<20}  {'Started':<16}  {'Summary'}")
        print("-" * 100)
        for s in sessions:
            started = s['started_at'].strftime('%Y-%m-%d %H:%M') if s['started_at'] else '?'
            summary = (s['summary'] or '')[:30]
            print(f"{s['id']}  {s['participant']:<20}  {started:<16}  {summary}")
    
    elif args.command == "show":
        messages = service.get_session_messages(args.session_id)
        
        if not messages:
            print("No messages found in session.")
            return
        
        for m in messages:
            timestamp = m['created_at'].strftime('%H:%M:%S') if m['created_at'] else '?'
            print(f"\n[{timestamp}] {m['participant']} ({m['role']}):")
            print(f"  {m['content'][:500]}{'...' if len(m['content']) > 500 else ''}")
    
    elif args.command == "new":
        session_id = service.create_session(participant=args.participant)
        print(f"Created session: {session_id}")
    
    elif args.command == "import":
        importer = ConversationImporter()
        
        if args.format == "claude":
            result = importer.import_claude_export(args.file)
            print(f"Imported {result['sessions_created']} sessions, {result['messages_imported']} messages")
    
    elif args.command == "recent":
        messages = service.get_recent_context(
            limit=args.limit,
            participant=args.participant
        )
        
        if not messages:
            print("No recent messages.")
            return
        
        for m in messages:
            timestamp = m['created_at'].strftime('%Y-%m-%d %H:%M') if m['created_at'] else '?'
            content_preview = m['content'][:100].replace('\n', ' ')
            print(f"[{timestamp}] {m['participant']}/{m['role']}: {content_preview}...")
    
    else:
        parser.print_help()

if __name__ == "__main__":
    main()
```

Make executable:
```bash
chmod +x /opt/alexandria/src/convo_cli.py
```

---

## Requirement 3: Validation

### Test 1: Create Session and Add Messages

```bash
cd /opt/alexandria
source venv/bin/activate

# Create a new session
python src/convo_cli.py new --participant sean
# Note the session ID returned

# Add an exchange
python src/convo_cli.py exchange \
  --user "What embedding model should we use?" \
  --assistant "I recommend nomic-embed-text via Ollama for local operation." \
  --session <session-id>
```

### Test 2: Search Conversations

```bash
# Search for the conversation we just added
python src/convo_cli.py search "embedding model recommendation"

# Should find the exchange with low distance score
```

### Test 3: List and Show Sessions

```bash
# List all sessions
python src/convo_cli.py sessions

# Show messages in a specific session
python src/convo_cli.py show <session-id>
```

### Test 4: Recent Context

```bash
python src/convo_cli.py recent --limit 5
```

### Test 5: Database Verification

```sql
-- Check conversations table
SELECT id, participant, role, 
       substring(content, 1, 50) as content_preview,
       session_id
FROM memory.conversations
ORDER BY created_at DESC
LIMIT 10;

-- Check embeddings were created
SELECT source_type, COUNT(*) 
FROM memory.embeddings 
GROUP BY source_type;

-- Check sessions
SELECT id, participant, started_at, summary
FROM memory.sessions
ORDER BY started_at DESC;
```

### Test 6: Import (Optional)

If Sean has a Claude.ai export:

```bash
python src/convo_cli.py import ~/claude-export.json --format claude
```

---

## Success Criteria

- [ ] `/opt/alexandria/src/conversations.py` created
- [ ] `/opt/alexandria/src/convo_cli.py` created
- [ ] Test 1 passes (can create sessions and add messages)
- [ ] Test 2 passes (semantic search finds conversations)
- [ ] Test 3 passes (can list and view sessions)
- [ ] Test 4 passes (recent context works)
- [ ] Test 5 passes (database has correct data)

---

## Notes for Claude Code

### On the Import Format
Claude.ai exports vary in structure. The importer handles common formats but may need adjustment. If import fails, check the JSON structure and adapt.

### On Embedding Every Message
The spec embeds every message by default. For very long conversations, this could be slow. The `embed=False` parameter exists if needed, but for typical use, embedding everything is fine.

### On Session Management
Sessions are optional. Messages can exist without a session_id. Sessions are useful for:
- Grouping related conversations
- Adding summaries
- Filtering searches

### On Vector Formatting
Apply the SPEC-003 lessons: use string formatting for pgvector, set ivfflat.probes, clean text before embedding.

---

## What This Enables

Once SPEC-004 is complete:
- Store any conversation with semantic search
- "What did we discuss about chunking strategies?"
- Import historical Claude.ai conversations
- Build context for future AI interactions
- Full conversation history alongside book knowledge

---

## Usage Examples (Post-Implementation)

```bash
# Daily workflow: capture key exchanges manually
python src/convo_cli.py exchange \
  --user "Should we use 500 or 1000 token chunks?" \
  --assistant "500 tokens with 50 token overlap balances context and granularity."

# Later: find that conversation
python src/convo_cli.py search "chunk size decision"

# Import old Claude.ai history
python src/convo_cli.py import ~/Downloads/claude-conversations.json

# Get recent context for a new session
python src/convo_cli.py recent --limit 20
```

---

## Future Extensions (Not in Scope)

- Automatic capture from Claude Code sessions
- Real-time sync with Claude.ai
- Conversation summarization via LLM
- Integration with Clara for research context

These would be separate specs. This spec provides the foundation.

---

*"The library remembers every conversation. Nothing is lost."*
