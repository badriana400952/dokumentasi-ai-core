# ⚛️ Panduan Arsitektur Enterprise Next.js (App Router): Route Handlers ➔ Controller / Action ➔ Service ➔ Repository (Powered by `@badriana/ai-core`)

Dokumen ini berisi panduan arsitektur **Clean Enterprise Layered Architecture (Route Handlers / Actions - Service - Repository)** untuk aplikasi full-stack berbasis **Next.js 14/15 (App Router & TypeScript)** yang memanfaatkan paket `@badriana/ai-core` secara 100% komprehensif (Upload File PDF RAG, Ingesti Database 40+ Tabel, Ingesti Teks Knowledge Manual, Multimodal Vision Chat, Streaming Chat SSE, Text-to-Speech MP3, & Speech-to-Text).

---

## 🏗️ Struktur Folder Proyek Backend Next.js App Router

```text
my-nextjs-ai-app/
├── src/
│   ├── app/
│   │   ├── api/
│   │   │   ├── ai/
│   │   │   │   ├── ingest-file/
│   │   │   │   │   └── route.ts          # Route Handler Ingesti File PDF/Teks
│   │   │   │   ├── ingest-manual/
│   │   │   │   │   └── route.ts          # Route Handler Ingesti Teks Knowledge Manual / SOP
│   │   │   │   ├── ingest-db/
│   │   │   │   │   └── route.ts          # Route Handler Ingesti Database
│   │   │   │   ├── chat/
│   │   │   │   │   └── route.ts          # Route Handler RAG & Vision Chat
│   │   │   │   ├── chat-stream/
│   │   │   │   │   └── route.ts          # Route Handler Streaming Chat SSE
│   │   │   │   └── audio/
│   │   │   │       ├── speak/
│   │   │   │       │   └── route.ts      # Route Handler Text-to-Speech (MP3)
│   │   │   │       └── transcribe/
│   │   │   │           └── route.ts      # Route Handler Speech-to-Text
│   │   └── page.tsx                      # Client Component UI React Next.js
│   └── lib/
│       ├── ai.ts                         # Singleton Instance Client @badriana/ai-core
│       ├── repositories/
│       │   └── knowledgeRepository.ts    # Layer Access Database (PostgreSQL pgvector)
│       └── services/
│           └── aiService.ts              # Business Logic Layer @badriana/ai-core
├── package.json
├── tsconfig.json
└── .env.local
```

---

## 1. ⚙️ Layer Konfigurasi Client AI (`src/lib/ai.ts`)

```typescript
// src/lib/ai.ts
import { createAI } from "@badriana/ai-core";

if (!process.env.OPENROUTER_API_KEY) {
  throw new Error("❌ OPENROUTER_API_KEY belum diset pada environment variables (.env.local)!");
}

export const ai = createAI({
  provider: "openrouter",
  apiKey: process.env.OPENROUTER_API_KEY,
  chatModel: "google/gemma-2-9b-it:free",
  embeddingModel: "openai/text-embedding-3-small",
  visionModel: "meta-llama/llama-3.2-11b-vision-instruct:free",
});
```

---

## 2. 🗄️ Layer Repository (`src/lib/repositories/knowledgeRepository.ts`)

```typescript
// src/lib/repositories/knowledgeRepository.ts
import { Pool } from "pg";
import type { Chunk } from "@badriana/ai-core";

export const pgPool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

export interface VectorChunkRecord {
  id: string;
  documentId?: string;
  content: string;
  similarityScore: number;
}

export class KnowledgeRepository {
  async saveChunks(documentId: string, chunks: Chunk[]): Promise<void> {
    const client = await pgPool.connect();
    try {
      await client.query("BEGIN");
      for (const chunk of chunks) {
        const query = `
          INSERT INTO knowledge_chunks (id, document_id, content, embedding)
          VALUES ($1, $2, $3, $4::vector)
          ON CONFLICT (id) DO UPDATE SET content = EXCLUDED.content, embedding = EXCLUDED.embedding;
        `;
        await client.query(query, [
          chunk.id,
          documentId,
          chunk.content,
          JSON.stringify(chunk.embedding),
        ]);
      }
      await client.query("COMMIT");
    } catch (error) {
      await client.query("ROLLBACK");
      throw error;
    } finally {
      client.release();
    }
  }

  async searchSimilarChunks(questionVector: number[], limit: number = 5): Promise<VectorChunkRecord[]> {
    const query = `
      SELECT id, document_id AS "documentId", content, 1 - (embedding <=> $1::vector) AS "similarityScore"
      FROM knowledge_chunks
      ORDER BY embedding <=> $1::vector
      LIMIT $2;
    `;
    const result = await pgPool.query(query, [JSON.stringify(questionVector), limit]);
    return result.rows;
  }
}
```

---

## 3. 🧠 Layer Service (`src/lib/services/aiService.ts`)

```typescript
// src/lib/services/aiService.ts
import { 
  createDocument, 
  createQuery, 
  formatDataRecordsForIngestion 
} from "@badriana/ai-core";
import { ai } from "../ai.js";
import { KnowledgeRepository } from "../repositories/knowledgeRepository.js";

export class AIService {
  private knowledgeRepo: KnowledgeRepository;

  constructor() {
    this.knowledgeRepo = new KnowledgeRepository();
  }

  async ingestUploadedFile(fileBuffer: Buffer, filename: string) {
    const docResult = await createDocument({
      source: fileBuffer,
      filename,
      embedder: async (texts) => {
        const embRes = await ai.embedding.create({ input: texts });
        return "embeddings" in embRes ? embRes.embeddings : [embRes.embedding];
      },
    });

    await this.knowledgeRepo.saveChunks(docResult.document.id, docResult.chunks);

    return {
      documentId: docResult.document.id,
      chunksCount: docResult.chunks.length,
      stats: docResult.stats,
    };
  }

  async ingestManualKnowledgeText(title: string, content: string) {
    const docResult = await createDocument({
      source: content,
      filename: `${title.replace(/\s+/g, "_")}.txt`,
      embedder: async (texts) => {
        const embRes = await ai.embedding.create({ input: texts });
        return "embeddings" in embRes ? embRes.embeddings : [embRes.embedding];
      },
    });

    await this.knowledgeRepo.saveChunks(docResult.document.id, docResult.chunks);

    return {
      documentId: docResult.document.id,
      chunksCount: docResult.chunks.length,
      stats: docResult.stats,
    };
  }

  async ingestCompanyDatabaseRecords(records: any[], tableName: string) {
    const formattedText = formatDataRecordsForIngestion(records, {
      recordTitlePrefix: `[Record Tabel ${tableName}]`,
      excludeKeys: ["password", "secret", "auth_token", "credit_card"],
    });

    const docResult = await createDocument({
      source: formattedText,
      filename: `db_table_${tableName}.txt`,
      embedder: async (texts) => {
        const embRes = await ai.embedding.create({ input: texts });
        return "embeddings" in embRes ? embRes.embeddings : [embRes.embedding];
      },
    });

    await this.knowledgeRepo.saveChunks(docResult.document.id, docResult.chunks);

    return {
      tableName,
      recordsProcessed: records.length,
      chunksGenerated: docResult.chunks.length,
    };
  }

  async generateChatAnswer(question: string, imageBase64?: string) {
    const embRes = await ai.embedding.create({ input: question });
    const questionVector = "embedding" in embRes ? embRes.embedding : embRes.embeddings[0]!;

    const candidateChunks = await this.knowledgeRepo.searchSimilarChunks(questionVector, 10);

    const queryResult = createQuery({
      vector: questionVector,
      keyword: question,
      chunks: candidateChunks.map((c) => ({
        id: c.id,
        type: "text",
        index: 0,
        startOffset: 0,
        endOffset: c.content.length,
        content: c.content,
      })),
      limit: 5,
    });

    const chatOutput = await ai.chat.generate({
      question,
      contexts: queryResult.rankedChunks,
      images: imageBase64 ? [{ dataUrl: imageBase64 }] : undefined,
    });

    return {
      answer: chatOutput.text,
      imageAnalysis: chatOutput.imageAnalysis,
      citations: chatOutput.citations,
      usage: chatOutput.usage,
      raw: chatOutput.raw,
    };
  }

  async streamChatAnswer(question: string, onChunk: (text: string) => void) {
    const embRes = await ai.embedding.create({ input: question });
    const questionVector = "embedding" in embRes ? embRes.embedding : embRes.embeddings[0]!;

    const candidateChunks = await this.knowledgeRepo.searchSimilarChunks(questionVector, 5);

    return await ai.chat.stream({
      question,
      contexts: candidateChunks.map((c) => ({
        id: c.id,
        type: "text",
        index: 0,
        startOffset: 0,
        endOffset: c.content.length,
        content: c.content,
      })),
      onChunk: (chunk) => onChunk(chunk.text),
    });
  }

  async textToSpeech(text: string, voice: "alloy" | "echo" | "fable" | "onyx" | "nova" | "shimmer" = "nova") {
    return await ai.audio.speak({ text, voice });
  }

  async speechToText(audioBase64: string, language: string = "id") {
    return await ai.audio.transcribe({ file: audioBase64, language });
  }
}
```

---

## 4. 🌐 Next.js App Router Route Handlers (`app/api/.../route.ts`)

### 📌 4.1 Ingesti File PDF (`src/app/api/ai/ingest-file/route.ts`):
```typescript
import { NextRequest, NextResponse } from "next/server";
import { AIService } from "@/lib/services/aiService";

const aiService = new AIService();

export async function POST(req: NextRequest) {
  try {
    const formData = await req.formData();
    const file = formData.get("file") as File | null;
    if (!file) return NextResponse.json({ success: false, error: "File wajib diunggah!" }, { status: 400 });

    const buffer = Buffer.from(await file.arrayBuffer());
    const data = await aiService.ingestUploadedFile(buffer, file.name);
    return NextResponse.json({ success: true, data });
  } catch (error: any) {
    return NextResponse.json({ success: false, error: error.message }, { status: 500 });
  }
}
```

### 📌 4.2 Ingesti Manual Teks (`src/app/api/ai/ingest-manual/route.ts`):
```typescript
import { NextRequest, NextResponse } from "next/server";
import { AIService } from "@/lib/services/aiService";

const aiService = new AIService();

export async function POST(req: NextRequest) {
  try {
    const { title, content } = await req.json();
    if (!title || !content) return NextResponse.json({ success: false, error: "Parameter title dan content wajib diisi!" }, { status: 400 });

    const data = await aiService.ingestManualKnowledgeText(title, content);
    return NextResponse.json({ success: true, data });
  } catch (error: any) {
    return NextResponse.json({ success: false, error: error.message }, { status: 500 });
  }
}
```

### 📌 4.3 Chat Stream SSE (`src/app/api/ai/chat-stream/route.ts`):
```typescript
import { NextRequest } from "next/server";
import { AIService } from "@/lib/services/aiService";

const aiService = new AIService();

export async function POST(req: NextRequest) {
  const { question } = await req.json();
  const encoder = new TextEncoder();

  const customStream = new ReadableStream({
    async start(controller) {
      await aiService.streamChatAnswer(question, (chunkText) => {
        controller.enqueue(encoder.encode(`data: ${JSON.stringify({ text: chunkText })}\n\n`));
      });
      controller.enqueue(encoder.encode(`data: [DONE]\n\n`));
      controller.close();
    },
  });

  return new Response(customStream, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache",
      "Connection": "keep-alive",
    },
  });
}
```
