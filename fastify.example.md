# ⚡ Panduan Arsitektur Enterprise Fastify: Plugin ➔ Routes ➔ Controller ➔ Service ➔ Repository (Powered by `@badriana/ai-core`)

Dokumen ini berisi panduan arsitektur **Clean Fastify Plugin & Layered Architecture (Plugin - Routes - Controller - Service - Repository)** untuk aplikasi backend ultra-fast berbasis **Fastify & TypeScript** yang memanfaatkan paket `@badriana/ai-core` secara 100% komprehensif (Upload File PDF RAG, Ingesti Database 40+ Tabel, Ingesti Teks Knowledge Manual, Multimodal Vision Chat, Streaming Chat SSE, Text-to-Speech MP3, & Speech-to-Text).

---

## 🏗️ Struktur Folder Proyek Backend Fastify

```text
my-fastify-ai-app/
├── src/
│   ├── plugins/
│   │   └── aiPlugin.ts               # Fastify Decorator Plugin @badriana/ai-core
│   ├── repositories/
│   │   └── knowledgeRepository.ts    # Layer Access Database (PostgreSQL pgvector)
│   ├── services/
│   │   └── aiService.ts              # Business Logic Layer @badriana/ai-core
│   ├── controllers/
│   │   └── aiController.ts           # Controller Layer Handler HTTP Req/Reply
│   ├── routes/
│   │   └── aiRoutes.ts               # Fastify Plugin Routes REST API Endpoints
│   └── server.ts                     # Bootstrap Entry Point Fastify Server
├── package.json
├── tsconfig.json
└── .env
```

---

## 1. ⚡ Layer Fastify Decorator Plugin (`src/plugins/aiPlugin.ts`)

```typescript
// src/plugins/aiPlugin.ts
import fp from "fastify-plugin";
import { FastifyInstance } from "fastify";
import { createAI, Client } from "@badriana/ai-core";

declare module "fastify" {
  interface FastifyInstance {
    ai: Client;
  }
}

export default fp(async (fastify: FastifyInstance) => {
  const apiKey = process.env.OPENROUTER_API_KEY;
  if (!apiKey) throw new Error("❌ OPENROUTER_API_KEY belum diset!");

  const aiClient = createAI({
    provider: "openrouter",
    apiKey,
    chatModel: "google/gemma-2-9b-it:free",
    embeddingModel: "openai/text-embedding-3-small",
    visionModel: "meta-llama/llama-3.2-11b-vision-instruct:free",
  });

  fastify.decorate("ai", aiClient);
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
import { FastifyInstance } from "fastify";
import { 
  createDocument, 
  createQuery, 
  formatDataRecordsForIngestion 
} from "@badriana/ai-core";
import { KnowledgeRepository } from "../repositories/knowledgeRepository.js";

export class AIService {
  private knowledgeRepo: KnowledgeRepository;

  constructor(private fastify: FastifyInstance) {
    this.knowledgeRepo = new KnowledgeRepository();
  }

  async ingestUploadedFile(fileBuffer: Buffer, filename: string) {
    const docResult = await createDocument({
      source: fileBuffer,
      filename,
      embedder: async (texts) => {
        const embRes = await this.fastify.ai.embedding.create({ input: texts });
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
        const embRes = await this.fastify.ai.embedding.create({ input: texts });
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
        const embRes = await this.fastify.ai.embedding.create({ input: texts });
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
    const embRes = await this.fastify.ai.embedding.create({ input: question });
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

    const chatOutput = await this.fastify.ai.chat.generate({
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
    const embRes = await this.fastify.ai.embedding.create({ input: question });
    const questionVector = "embedding" in embRes ? embRes.embedding : embRes.embeddings[0]!;

    const candidateChunks = await this.knowledgeRepo.searchSimilarChunks(questionVector, 5);

    return await this.fastify.ai.chat.stream({
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
    return await this.fastify.ai.audio.speak({ text, voice });
  }

  async speechToText(audioBase64: string, language: string = "id") {
    return await this.fastify.ai.audio.transcribe({ file: audioBase64, language });
  }
}
```

---

## 4. 🎮 Layer Controller (`src/controllers/aiController.ts`)

```typescript
// src/controllers/aiController.ts
import { FastifyRequest, FastifyReply } from "fastify";
import { AIService } from "../services/aiService.js";

export class AIController {
  private getService(request: FastifyRequest): AIService {
    return new AIService(request.server);
  }

  ingestFile = async (request: FastifyRequest, reply: FastifyReply): Promise<void> => {
    try {
      const fileData = await request.file();
      if (!fileData) return reply.status(400).send({ success: false, error: "File wajib diunggah!" });

      const fileBuffer = await fileData.toBuffer();
      const service = this.getService(request);
      const result = await service.ingestUploadedFile(fileBuffer, fileData.filename);
      reply.status(200).send({ success: true, data: result });
    } catch (error: any) {
      reply.status(500).send({ success: false, error: error.message });
    }
  };

  ingestManual = async (request: FastifyRequest, reply: FastifyReply): Promise<void> => {
    try {
      const { title, content } = request.body as { title: string; content: string };
      if (!title || !content) return reply.status(400).send({ success: false, error: "Parameter title dan content wajib diisi!" });

      const service = this.getService(request);
      const result = await service.ingestManualKnowledgeText(title, content);
      reply.status(200).send({ success: true, data: result });
    } catch (error: any) {
      reply.status(500).send({ success: false, error: error.message });
    }
  };

  ingestDatabase = async (request: FastifyRequest, reply: FastifyReply): Promise<void> => {
    try {
      const { records, tableName } = request.body as { records: any[]; tableName: string };
      const service = this.getService(request);
      const result = await service.ingestCompanyDatabaseRecords(records, tableName);
      reply.status(200).send({ success: true, data: result });
    } catch (error: any) {
      reply.status(500).send({ success: false, error: error.message });
    }
  };

  chat = async (request: FastifyRequest, reply: FastifyReply): Promise<void> => {
    try {
      const { question, imageBase64 } = request.body as { question: string; imageBase64?: string };
      const service = this.getService(request);
      const result = await service.generateChatAnswer(question, imageBase64);
      reply.status(200).send({ success: true, data: result });
    } catch (error: any) {
      reply.status(500).send({ success: false, error: error.message });
    }
  };

  chatStream = async (request: FastifyRequest, reply: FastifyReply): Promise<void> => {
    try {
      const { question } = request.body as { question: string };
      reply.raw.setHeader("Content-Type", "text/event-stream");
      reply.raw.setHeader("Cache-Control", "no-cache");
      reply.raw.setHeader("Connection", "keep-alive");

      const service = this.getService(request);
      await service.streamChatAnswer(question, (chunkText) => {
        reply.raw.write(`data: ${JSON.stringify({ text: chunkText })}\n\n`);
      });

      reply.raw.write(`data: [DONE]\n\n`);
      reply.raw.end();
    } catch (error: any) {
      reply.status(500).send({ success: false, error: error.message });
    }
  };

  speak = async (request: FastifyRequest, reply: FastifyReply): Promise<void> => {
    try {
      const { text, voice } = request.body as { text: string; voice?: any };
      const service = this.getService(request);
      const audioResult = await service.textToSpeech(text, voice);
      reply.header("Content-Type", audioResult.mimeType);
      reply.status(200).send(audioResult.audio);
    } catch (error: any) {
      reply.status(500).send({ success: false, error: error.message });
    }
  };

  transcribe = async (request: FastifyRequest, reply: FastifyReply): Promise<void> => {
    try {
      const { audioBase64, language } = request.body as { audioBase64: string; language?: string };
      const service = this.getService(request);
      const transcriptResult = await service.speechToText(audioBase64, language);
      reply.status(200).send({ success: true, data: transcriptResult });
    } catch (error: any) {
      reply.status(500).send({ success: false, error: error.message });
    }
  };
}
```
