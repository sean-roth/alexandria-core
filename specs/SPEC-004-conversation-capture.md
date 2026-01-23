# SPEC-004: Conversation Capture

**Project:** alexandria-core  
**Author:** Claude (Designer) via claude.ai  
**For:** Claude Code (Engineer) on Linux Server  
**Captain:** Sean  
**Date:** January 23, 2026  
**Status:** Ready for Implementation  
**Depends on:** SPEC-002 (complete), SPEC-003 (complete)

---

## Context

The library can now store and search books. But knowledge doesn't just come from books - it comes from conversations. Every discussion with Claude, every decision made, every problem solved represents knowledge that should compound.

This spec adds the ability to capture conversations, make them searchable, and optionally import historical chat exports.

---

## This Spec's Goal

Build a system that:
1. Stores conversation messages with metadata
2. Groups messages into sessions
3. Generates session summaries (for quick reference)
4. Embeds conversations for semantic search
5. Can import Claude.ai chat exports

After this: *"What did we decide about the embedding architecture?"* returns the actual conversation where we made that decision.

---

## Architecture Overview

```
Conversation Input
    │
    ├── Manual capture (paste into CLI)
    ├── Import from Claude.ai export (JSON)
    └── Future: Direct API integration
    │
    ▼
┌─────────────────┐
│  Parse & Store  │  (memory.conversations + memory.sessions)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│    Embedding    │  (Ollama nomic-embed-text)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Semantic Index │  (memory.embeddings)
└─────────────────┘
```

---

## Learned from Previous Specs

Per Engineer's feedback from SPEC-003:

1. **Vector formatting**: All pgvector inserts will use explicit string formatting:
   ```python
   embedding_str = "[" + ",".join(str(x) for x in embedding) + "]"
   ```

2. **ivfflat.probes**: All search queries will set probes before querying:
   ```python
   cursor.execute("SET ivfflat.probes = 100")
   ```

3. **Text preprocessing**: Conversation text will be cleaned before embedding (collapse excessive whitespace, normalize unicode).

---

## Requirement 1: Conversation Service

Create `/opt/alexandria/src/conversations.py`:

```python
import os
import re
import json
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
    Manages conversation capture, storage, and search.
    
    Stores in memory.conversations and memory.sessions.
    Embeds for semantic search via memory.embeddings.
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
    
    def _clean_text(self, text: str) -> str:
        """Clean text for embedding (preserve original for storage)."""
        # Normalize unicode
        text = text.encode('utf-8', errors='ignore').decode('utf-8')
        # Collapse excessive whitespace
        text = re.sub(r'\n{3,}', '\n\n', text)
        text = re.sub(r' {2,}', ' ', text)
        # Collapse repeated punctuation (learned from SPEC-003)
        text = re.sub(r'\.{3,}', '...', text)
        text = re.sub(r'-{3,}', '---', text)
        return text.strip()
    
    def _format_vector(self, embedding: list[float]) -> str:
        """Format embedding for pgvector insertion."""
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
        session_id: str,
        role: str,
        content: str,
        participant: str,
        metadata: Optional[dict] = None,
        embed: bool = True
    ) -> int:
        """
        Add a message to a session.
        
        Args:
            session_id: UUID of the session
            role: 'user' or 'assistant'
            content: Message text
            participant: Who sent this ('sean', 'claude_desktop', 'claude_code', etc.)
            metadata: Optional additional data
            embed: Whether to create embedding (default True)
        
        Returns: message ID
        """
        conn = psycopg2.connect(**self.db_config)
        cursor = conn.cursor()
        
        # Store message
        cursor.execute("""
            INSERT INTO memory.conversations 
            (session_id, role, content, participant, metadata)
            VALUES (%s, %s, %s, %s, %s)
            RETURNING id
        """, (session_id, role, content, participant, Json(metadata or {})))
        
        message_id = cursor.fetchone()[0]
        
        # Create embedding if requested
        if embed and content.strip():
            cleaned = self._clean_text(content)
            if len(cleaned) > 50:  # Only embed substantive messages
                embedding = self.embedding_service.generate_embedding(cleaned)
                embedding_str = self._format_vector(embedding)
                
                cursor.execute("""
                    INSERT INTO memory.embeddings
                    (source_type, source_id, content_preview, embedding, metadata)
                    VALUES (%s, %s, %s, %s, %s)
                """, (
                    'conversation',
                    message_id,
                    cleaned[:500],
                    embedding_str,
                    Json({
                        "session_id": session_id,
                        "role": role,
                        "participant": participant,
                        "model": "nomic-embed-text"
                    })
                ))
        
        conn.commit()
        conn.close()
        
        return message_id
    
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
        
        # Embed the summary if provided
        if summary and summary.strip():
            cleaned = self._clean_text(summary)
            embedding = self.embedding_service.generate_embedding(cleaned)
            embedding_str = self._format_vector(embedding)
            
            cursor.execute("""
                INSERT INTO memory.embeddings
                (source_type, source_id, content_preview, embedding, metadata)
                VALUES (%s, %s, %s, %s, %s)
            """, (
                'session_summary',
                None,
                cleaned[:500],
                embedding_str,
                Json({
                    "session_id": session_id,
                    "model": "nomic-embed-text"
                })
            ))
        
        conn.commit()
        conn.close()
    
    def capture_conversation(
        self,
        messages: list[dict],
        participant: str = "claude_desktop",
        session_metadata: Optional[dict] = None
    ) -> dict:
        """
        Capture a full conversation at once.
        
        Args:
            messages: List of {"role": "user"|"assistant", "content": "..."}
            participant: The AI participant ('claude_desktop', 'claude_code', etc.)
            session_metadata: Optional metadata for the session
        
        Returns: {"session_id": str, "messages_stored": int, "messages_embedded": int}
        """
        session_id = self.create_session(participant, session_metadata)
        
        messages_stored = 0
        messages_embedded = 0
        
        for msg in messages:
            role = msg.get("role", "user")
            content = msg.get("content", "")
            
            if not content.strip():
                continue
            
            # Determine participant based on role
            msg_participant = "sean" if role == "user" else participant
            
            self.add_message(
                session_id=session_id,
                role=role,
                content=content,
                participant=msg_participant,
                metadata=msg.get("metadata"),
                embed=True
            )
            messages_stored += 1
            if len(self._clean_text(content)) > 50:
                messages_embedded += 1
        
        return {
            "session_id": session_id,
            "messages_stored": messages_stored,
            "messages_embedded": messages_embedded
        }
    
    def import_claude_export(self, export_path: str) -> dict:
        """
        Import conversations from Claude.ai JSON export.
        
        The export format is a JSON array of conversation objects.
        Each conversation has 'uuid', 'name', 'created_at', 'updated_at',
        and 'chat_messages' array.
        
        Returns: {"conversations_imported": int, "messages_imported": int, "errors": list}
        """
        with open(export_path, 'r', encoding='utf-8') as f:
            data = json.load(f)
        
        # Handle both single conversation and array of conversations
        if isinstance(data, dict):
            conversations = [data]
        else:
            conversations = data
        
        total_conversations = 0
        total_messages = 0
        errors = []
        
        for conv in conversations:
            try:
                conv_uuid = conv.get('uuid', str(uuid.uuid4()))
                conv_name = conv.get('name', 'Untitled')
                created_at = conv.get('created_at')
                messages = conv.get('chat_messages', [])
                
                if not messages:
                    continue
                
                # Create session with original metadata
                conn = psycopg2.connect(**self.db_config)
                cursor = conn.cursor()
                
                session_id = str(uuid.uuid4())
                
                cursor.execute("""
                    INSERT INTO memory.sessions (id, participant, metadata)
                    VALUES (%s, %s, %s)
                """, (
                    session_id,
                    'claude_desktop',
                    Json({
                        "source": "claude_export",
                        "original_uuid": conv_uuid,
                        "conversation_name": conv_name,
                        "original_created_at": created_at
                    })
                ))
                conn.commit()
                conn.close()
                
                # Add messages
                for msg in messages:
                    sender = msg.get('sender', 'human')
                    role = 'user' if sender == 'human' else 'assistant'
                    
                    # Handle different content formats
                    content = msg.get('text', '')
                    if not content and 'content' in msg:
                        content_data = msg['content']
                        if isinstance(content_data, list):
                            # Extract text from content blocks
                            content = ' '.join(
                                block.get('text', '') 
                                for block in content_data 
                                if block.get('type') == 'text'
                            )
                        elif isinstance(content_data, str):
                            content = content_data
                    
                    if content.strip():
                        participant = 'sean' if role == 'user' else 'claude_desktop'
                        self.add_message(
                            session_id=session_id,
                            role=role,
                            content=content,
                            participant=participant,
                            metadata={"original_uuid": msg.get('uuid')},
                            embed=True
                        )
                        total_messages += 1
                
                total_conversations += 1
                print(f"  Imported: {conv_name[:50]}... ({len(messages)} messages)")
                
            except Exception as e:
                errors.append(f"Error importing conversation: {e}")
        
        return {
            "conversations_imported": total_conversations,
            "messages_imported": total_messages,
            "errors": errors
        }
    
    def search_conversations(
        self,
        query: str,
        limit: int = 5,
        participant: Optional[str] = None
    ) -> list[dict]:
        """
        Semantic search across conversations.
        
        Returns matching messages with session context.
        """
        query_embedding = self.embedding_service.generate_embedding(query)
        embedding_str = self._format_vector(query_embedding)
        
        conn = psycopg2.connect(**self.db_config)
        cursor = conn.cursor()
        
        # Set probes for accurate search (learned from SPEC-003)
        cursor.execute("SET ivfflat.probes = 100")
        
        if participant:
            cursor.execute("""
                SELECT 
                    e.id,
                    e.source_id,
                    e.content_preview,
                    e.embedding <=> %s::vector AS distance,
                    e.metadata,
                    c.role,
                    c.participant,
                    c.session_id,
                    s.metadata as session_metadata
                FROM memory.embeddings e
                LEFT JOIN memory.conversations c ON e.source_id = c.id AND e.source_type = 'conversation'
                LEFT JOIN memory.sessions s ON c.session_id = s.id
                WHERE e.source_type IN ('conversation', 'session_summary')
                AND (e.metadata->>'participant' = %s OR c.participant = %s)
                ORDER BY distance
                LIMIT %s
            """, (embedding_str, participant, participant, limit))
        else:
            cursor.execute("""
                SELECT 
                    e.id,
                    e.source_id,
                    e.content_preview,
                    e.embedding <=> %s::vector AS distance,
                    e.metadata,
                    c.role,
                    c.participant,
                    c.session_id,
                    s.metadata as session_metadata
                FROM memory.embeddings e
                LEFT JOIN memory.conversations c ON e.source_id = c.id AND e.source_type = 'conversation'
                LEFT JOIN memory.sessions s ON c.session_id = s.id
                WHERE e.source_type IN ('conversation', 'session_summary')
                ORDER BY distance
                LIMIT %s
            """, (embedding_str, limit))
        
        results = []
        for row in cursor.fetchall():
            session_meta = row[8] or {}
            results.append({
                "embedding_id": row[0],
                "message_id": row[1],
                "preview": row[2],
                "distance": row[3],
                "embedding_metadata": row[4],
                "role": row[5],
                "participant": row[6],
                "session_id": row[7],
                "conversation_name": session_meta.get("conversation_name", "Unknown")
            })
        
        conn.close()
        return results
    
    def list_sessions(self, limit: int = 20) -> list[dict]:
        """List recent conversation sessions."""
        conn = psycopg2.connect(**self.db_config)
        cursor = conn.cursor()
        
        cursor.execute("""
            SELECT 
                s.id,
                s.participant,
                s.started_at,
                s.ended_at,
                s.summary,
                s.metadata,
                COUNT(c.id) as message_count
            FROM memory.sessions s
            LEFT JOIN memory.conversations c ON s.id = c.session_id
            GROUP BY s.id
            ORDER BY s.started_at DESC
            LIMIT %s
        """, (limit,))
        
        sessions = []
        for row in cursor.fetchall():
            meta = row[5] or {}
            sessions.append({
                "id": row[0],
                "participant": row[1],
                "started_at": row[2],
                "ended_at": row[3],
                "summary": row[4],
                "conversation_name": meta.get("conversation_name", ""),
                "source": meta.get("source", "manual"),
                "message_count": row[6]
            })
        
        conn.close()
        return sessions
    
    def get_session_messages(self, session_id: str) -> list[dict]:
        """Get all messages in a session."""
        conn = psycopg2.connect(**self.db_config)
        cursor = conn.cursor()
        
        cursor.execute("""
            SELECT id, role, content, participant, created_at, metadata
            FROM memory.conversations
            WHERE session_id = %s
            ORDER BY created_at ASC
        """, (session_id,))
        
        messages = []
        for row in cursor.fetchall():
            messages.append({
                "id": row[0],
                "role": row[1],
                "content": row[2],
                "participant": row[3],
                "created_at": row[4],
                "metadata": row[5]
            })
        
        conn.close()
        return messages
```

---

## Requirement 2: CLI Tool

Create `/opt/alexandria/src/convo_cli.py`:

```python
#!/usr/bin/env python3
"""CLI for conversation capture and search."""

import argparse
import sys
import os
import json

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))

from dotenv import load_dotenv
load_dotenv("/opt/alexandria/.env")

from conversations import ConversationService

def main():
    parser = argparse.ArgumentParser(description="Alexandria Conversation CLI")
    subparsers = parser.add_subparsers(dest="command")
    
    # Import command
    import_parser = subparsers.add_parser("import", help="Import Claude.ai export")
    import_parser.add_argument("path", help="Path to export JSON file")
    
    # Capture command (for manual paste)
    capture_parser = subparsers.add_parser("capture", help="Capture a conversation from JSON")
    capture_parser.add_argument("path", help="Path to JSON file with messages")
    capture_parser.add_argument("--participant", default="claude_desktop", help="AI participant name")
    
    # Search command
    search_parser = subparsers.add_parser("search", help="Search conversations")
    search_parser.add_argument("query", help="Search query")
    search_parser.add_argument("--limit", type=int, default=5, help="Max results")
    search_parser.add_argument("--participant", help="Filter by participant")
    
    # List command
    list_parser = subparsers.add_parser("list", help="List conversation sessions")
    list_parser.add_argument("--limit", type=int, default=20, help="Max sessions")
    
    # Show command
    show_parser = subparsers.add_parser("show", help="Show messages in a session")
    show_parser.add_argument("session_id", help="Session UUID")
    
    args = parser.parse_args()
    service = ConversationService()
    
    if args.command == "import":
        if not os.path.exists(args.path):
            print(f"Error: File not found: {args.path}")
            sys.exit(1)
        
        print(f"Importing from {args.path}...")
        result = service.import_claude_export(args.path)
        
        print(f"\nImport complete:")
        print(f"  Conversations: {result['conversations_imported']}")
        print(f"  Messages: {result['messages_imported']}")
        
        if result['errors']:
            print(f"\nErrors ({len(result['errors'])}):")
            for err in result['errors'][:5]:
                print(f"  - {err}")
    
    elif args.command == "capture":
        if not os.path.exists(args.path):
            print(f"Error: File not found: {args.path}")
            sys.exit(1)
        
        with open(args.path, 'r') as f:
            messages = json.load(f)
        
        result = service.capture_conversation(messages, args.participant)
        
        print(f"Captured conversation:")
        print(f"  Session ID: {result['session_id']}")
        print(f"  Messages stored: {result['messages_stored']}")
        print(f"  Messages embedded: {result['messages_embedded']}")
    
    elif args.command == "search":
        results = service.search_conversations(
            args.query,
            limit=args.limit,
            participant=args.participant
        )
        
        if not results:
            print("No results found.")
            return
        
        for r in results:
            role_indicator = "👤" if r['role'] == 'user' else "🤖"
            conv_name = r['conversation_name'][:30] if r['conversation_name'] else "Unknown"
            
            print(f"\n{role_indicator} [{conv_name}] (distance: {r['distance']:.4f})")
            print(f"   {r['preview'][:200]}...")
            print(f"   Session: {r['session_id'][:8]}...")
    
    elif args.command == "list":
        sessions = service.list_sessions(limit=args.limit)
        
        if not sessions:
            print("No conversation sessions found.")
            return
        
        print(f"{'ID':<10} {'Name':<35} {'Msgs':<6} {'Participant':<15} {'Date'}")
        print("-" * 90)
        
        for s in sessions:
            sid = s['id'][:8] + "..."
            name = s['conversation_name'][:32] + "..." if len(s.get('conversation_name', '')) > 35 else s.get('conversation_name', '-')
            date = s['started_at'].strftime('%Y-%m-%d') if s['started_at'] else '?'
            print(f"{sid:<10} {name:<35} {s['message_count']:<6} {s['participant']:<15} {date}")
    
    elif args.command == "show":
        messages = service.get_session_messages(args.session_id)
        
        if not messages:
            print("No messages found for this session.")
            return
        
        for msg in messages:
            role_indicator = "👤 Sean:" if msg['role'] == 'user' else f"🤖 {msg['participant']}:"
            print(f"\n{role_indicator}")
            print(msg['content'][:500])
            if len(msg['content']) > 500:
                print("... [truncated]")
    
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

## Requirement 3: Get Claude.ai Export

Sean can export his Claude.ai conversations:

1. Go to https://claude.ai/settings
2. Click "Export Data"
3. Download the JSON export
4. Transfer to server: `/opt/alexandria/imports/`

```bash
mkdir -p /opt/alexandria/imports
# Then SCP the export file there
```

---

## Requirement 4: Validation

### Test 1: Manual Capture

Create a test conversation file `/tmp/test_convo.json`:
```json
[
    {"role": "user", "content": "What's the best way to structure a vector database?"},
    {"role": "assistant", "content": "For your use case, I'd recommend PostgreSQL with pgvector. It gives you both structured data and semantic search in one system."},
    {"role": "user", "content": "Should I use OpenAI or local embeddings?"},
    {"role": "assistant", "content": "Given your values around data sovereignty, local embeddings with Ollama make more sense. The quality difference is minimal for personal knowledge bases."}
]
```

```bash
cd /opt/alexandria
source venv/bin/activate

python src/convo_cli.py capture /tmp/test_convo.json --participant claude_designer
```

### Test 2: Search Conversations

```bash
python src/convo_cli.py search "vector database recommendations"
python src/convo_cli.py search "embedding model decision"
```

Should find the test conversation.

### Test 3: List Sessions

```bash
python src/convo_cli.py list
```

Should show the captured session.

### Test 4: Show Session

```bash
python src/convo_cli.py show <session-id-from-list>
```

Should display the conversation messages.

### Test 5: Import Claude Export (if available)

```bash
python src/convo_cli.py import /opt/alexandria/imports/claude-export.json
```

### Test 6: Verify in Database

```sql
-- Check sessions
SELECT id, participant, metadata->>'conversation_name' as name,
       (SELECT COUNT(*) FROM memory.conversations c WHERE c.session_id = s.id) as msgs
FROM memory.sessions s
ORDER BY started_at DESC
LIMIT 5;

-- Check conversation embeddings
SELECT source_type, COUNT(*) 
FROM memory.embeddings 
WHERE source_type IN ('conversation', 'session_summary')
GROUP BY source_type;

-- Sample search
SET ivfflat.probes = 100;
SELECT content_preview, embedding <=> (
    SELECT embedding FROM memory.embeddings WHERE source_type = 'conversation' LIMIT 1
)::vector as distance
FROM memory.embeddings
WHERE source_type = 'conversation'
ORDER BY distance
LIMIT 3;
```

---

## Success Criteria

- [ ] `/opt/alexandria/src/conversations.py` created
- [ ] `/opt/alexandria/src/convo_cli.py` created
- [ ] Test 1 passes (manual capture works)
- [ ] Test 2 passes (semantic search finds conversations)
- [ ] Test 3 passes (list shows sessions)
- [ ] Test 4 passes (show displays messages)
- [ ] Test 6 passes (database has correct data)

---

## Notes for Claude Code

### On the Claude Export Format
The format varies slightly between export versions. The import function handles:
- `text` field (older format)
- `content` as string (some versions)
- `content` as array of blocks (newer format)

If you encounter a format that doesn't work, document it and adapt.

### On Embedding Decisions
- Only embed messages > 50 characters (skip "ok", "thanks", etc.)
- Session summaries get embedded separately (useful for finding whole conversations)
- Metadata tracks session_id so we can link back to full context

### On Performance
- Large exports (hundreds of conversations) will take time
- Consider adding progress indicators for imports
- Each message = one embedding = ~1-2 seconds with Ollama

---

## What This Enables

Once SPEC-004 is complete:
- Search past conversations by topic/concept
- "What did we decide about X?" actually works
- Import months of Claude.ai history
- Foundation for AI agents to remember past discussions
- Complete the knowledge loop: Books + Conversations = Compounding context

---

## The Complete Library

After this spec:

```
alexandria-core
├── Books (SPEC-003)
│   └── 17 technical PDFs → searchable chunks
├── Conversations (SPEC-004)
│   └── All Claude chats → searchable messages
├── Embeddings (SPEC-002)
│   └── Everything unified in semantic space
└── Foundation (SPEC-001)
    └── PostgreSQL + pgvector + Syncthing
```

You'll be able to ask:
- "What did the LangChain book say about RAG?"
- "What did we decide about the embedding architecture?"
- "Find everything related to transformer attention mechanisms"

And get answers that span books AND conversations.

---

*"The library remembers every conversation. Knowledge compounds."*
