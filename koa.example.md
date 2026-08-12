# 🌿 Panduan Arsitektur Enterprise Koa.js: Router ➔ Controller ➔ Service ➔ Repository (Powered by `@badriana/ai-core`)

Dokumen ini berisi panduan arsitektur **Clean Koa Middleware Architecture (Router - Controller - Service - Repository)** untuk aplikasi backend berbasis **Koa.js & TypeScript** yang memanfaatkan paket `@badriana/ai-core` secara 100% komprehensif (Upload File PDF RAG, Ingesti Database 40+ Tabel, Ingesti Teks Knowledge Manual, Multimodal Vision Chat, Streaming Chat SSE, Text-to-Speech MP3, & Speech-to-Text).

---

## 🏗️ Struktur Folder Proyek Backend Koa.js

```text
my-koa-ai-app/
├── src/
│   ├── config/
│   │   └── ai.ts                    # Single Instance Client @badriana/ai-core
│   ├── repositories/
│   │   └── knowledgeRepository.ts   # Layer Access Database (PostgreSQL pgvector)
│   ├── services/
│   │   └── aiService.ts             # Business Logic Layer @badriana/ai-core
│   ├── controllers/
│   │   └── aiController.ts          # Layer Handler Koa Context (ctx)
│   ├── routes/
│   │   └── aiRoutes.ts              # Layer Router REST API Endpoint Koa
│   └── server.ts                    # Bootstrap Entry Point Koa.js Server
├── package.json
├── tsconfig.json
└── .env
```

---

## 1. ⚙️ Layer Konfigurasi Client AI (`src/config/ai.ts`)

```typescript
// src/config/ai.ts
import { createAI } from "@badriana/ai-core";

if (!process.env.OPENROUTER_API_KEY) {
  throw new Error("❌ OPENROUTER_API_KEY belum diset pada environment variables (.env)!");
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

## 2. 🗄️ Layer Repository (`src/repositories/knowledgeRepository.ts`)

```typescript
// src/repositories/knowledgeRepository.ts
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

## 3. 🧠 Layer Service (`src/services/aiService.ts`)

```typescript
// src/services/aiService.ts
import { 
  createDocument, 
  createQuery, 
  formatDataRecordsForIngestion 
} from "@badriana/ai-core";
import { ai } from "../config/ai.js";
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

## 4. 🎮 Layer Controller (`src/controllers/aiController.ts`)

```typescript
// src/controllers/aiController.ts
import { Context } from "koa";
import { PassThrough } from "stream";
import { AIService } from "../services/aiService.js";

export class AIController {
  private aiService: AIService;

  constructor() {
    this.aiService = new AIService();
  }

  ingestFile = async (ctx: Context): Promise<void> => {
    try {
      const file = (ctx.request as any).file;
      if (!file) {
        ctx.status = 400;
        ctx.body = { success: false, error: "File wajib diunggah!" };
        return;
      }

      const result = await this.aiService.ingestUploadedFile(file.buffer, file.originalname);
      ctx.status = 200;
      ctx.body = { success: true, data: result };
    } catch (error: any) {
      ctx.status = 500;
      ctx.body = { success: false, error: error.message };
    }
  };

  ingestManual = async (ctx: Context): Promise<void> => {
    try {
      const { title, content } = (ctx.request as any).body || {};
      if (!title || !content) {
        ctx.status = 400;
        ctx.body = { success: false, error: "Parameter title dan content wajib diisi!" };
        return;
      }

      const result = await this.aiService.ingestManualKnowledgeText(title, content);
      ctx.status = 200;
      ctx.body = { success: true, data: result };
    } catch (error: any) {
      ctx.status = 500;
      ctx.body = { success: false, error: error.message };
    }
  };

  ingestDatabase = async (ctx: Context): Promise<void> => {
    try {
      const { records, tableName } = (ctx.request as any).body || {};
      const result = await this.aiService.ingestCompanyDatabaseRecords(records, tableName);
      ctx.status = 200;
      ctx.body = { success: true, data: result };
    } catch (error: any) {
      ctx.status = 500;
      ctx.body = { success: false, error: error.message };
    }
  };

  chat = async (ctx: Context): Promise<void> => {
    try {
      const { question, imageBase64 } = (ctx.request as any).body || {};
      const result = await this.aiService.generateChatAnswer(question, imageBase64);
      ctx.status = 200;
      ctx.body = { success: true, data: result };
    } catch (error: any) {
      ctx.status = 500;
      ctx.body = { success: false, error: error.message };
    }
  };

  chatStream = async (ctx: Context): Promise<void> => {
    try {
      const { question } = (ctx.request as any).body || {};
      ctx.set("Content-Type", "text/event-stream");
      ctx.set("Cache-Control", "no-cache");
      ctx.set("Connection", "keep-alive");

      const stream = new PassThrough();
      ctx.body = stream;

      await this.aiService.streamChatAnswer(question, (chunkText) => {
        stream.write(`data: ${JSON.stringify({ text: chunkText })}\n\n`);
      });

      stream.write(`data: [DONE]\n\n`);
      stream.end();
    } catch (error: any) {
      ctx.status = 500;
      ctx.body = { success: false, error: error.message };
    }
  };

  speak = async (ctx: Context): Promise<void> => {
    try {
      const { text, voice } = (ctx.request as any).body || {};
      const audioResult = await this.aiService.textToSpeech(text, voice);
      ctx.set("Content-Type", audioResult.mimeType);
      ctx.status = 200;
      ctx.body = audioResult.audio;
    } catch (error: any) {
      ctx.status = 500;
      ctx.body = { success: false, error: error.message };
    }
  };

  transcribe = async (ctx: Context): Promise<void> => {
    try {
      const { audioBase64, language } = (ctx.request as any).body || {};
      const transcriptResult = await this.aiService.speechToText(audioBase64, language);
      ctx.status = 200;
      ctx.body = { success: true, data: transcriptResult };
    } catch (error: any) {
      ctx.status = 500;
      ctx.body = { success: false, error: error.message };
    }
  };
}
```
