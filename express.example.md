# 🚀 Panduan Arsitektur Enterprise Express.js: Router ➔ Controller ➔ Service ➔ Repository (Powered by `@badriana/ai-core`)

Dokumen ini berisi panduan arsitektur **Clean Layered Architecture (Router - Controller - Service - Repository)** untuk aplikasi backend berbasis **Express.js & TypeScript** yang menggunakan paket `@badriana/ai-core` secara 100% komprehensif (Upload File PDF RAG, Ingesti Database 40+ Tabel, Ingesti Teks Knowledge Manual, Multimodal Vision Chat, Streaming Chat SSE, Text-to-Speech MP3, & Speech-to-Text).

---

## 🏗️ Struktur Folder Proyek Backend Express.js

```text
my-express-ai-app/
├── src/
│   ├── config/
│   │   └── ai.ts                    # Single Instance Client @badriana/ai-core
│   ├── repositories/
│   │   └── knowledgeRepository.ts   # Layer Akses Database (PostgreSQL pgvector / Supabase)
│   ├── services/
│   │   └── aiService.ts             # Layer Logika Bisnis & Fitur @badriana/ai-core
│   ├── controllers/
│   │   └── aiController.ts          # Layer Handler HTTP Req/Res & Validasi Input
│   ├── routes/
│   │   └── aiRoutes.ts              # Layer Routing REST API Endpoint Express
│   └── server.ts                    # Entry Point Utama Express.js Server
├── package.json
├── tsconfig.json
└── .env
```

---

## 1. ⚙️ Layer Konfigurasi Client AI (`src/config/ai.ts`)

File ini bertanggung jawab menginisialisasi client `@badriana/ai-core` sebagai *singleton instance* yang siap digunakan di seluruh layer aplikasi.

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

Layer ini menangani komunikasi langsung dengan database (misal: PostgreSQL dengan ekstensi `pgvector`). Layer ini menyimpan vektor embedding 1536-dimensi dan melakukan pencarian Cosine Similarity (`<=>`).

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
  /**
   * Menyimpan daftar Chunks RAG hasil createDocument() ke tabel Database
   */
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

  /**
   * Mencari Chunks paling relevan menggunakan pencarian Cosine Similarity (<=>)
   */
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

Layer Service berisi seluruh **Logika Bisnis AI** yang memanggil API dari `@badriana/ai-core`:
1. Ingesti Berkas PDF / TXT (`createDocument()`)
2. Ingesti Teks Knowledge Manual / SOP (`createDocument()`)
3. Ingesti 40+ Tabel Database SQL Perusahaan (`formatDataRecordsForIngestion()`)
4. Multimodal Vision Chat (`ai.chat.generate()`) & RRF Reranking (`createQuery()`)
5. Streaming Response Chat completion SSE (`ai.chat.stream()`)
6. Text-to-Speech MP3 Stream (`ai.audio.speak()`)
7. Speech-to-Text Transkripsi Suara (`ai.audio.transcribe()`)

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

  /**
   * 1. Ingesti Berkas PDF/Teks yang diunggah pengguna
   */
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

  /**
   * 2. Ingesti Teks Knowledge Manual (SOP / Artikel Pengetahuan)
   */
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

  /**
   * 3. Ingesti Data dari Database SQL Perusahaan ke RAG Vektor
   */
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

  /**
   * 4. Tanya Jawab Multimodal Vision & Hybrid RAG Chat
   */
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

  /**
   * 5. Streaming Chat Completion SSE (Server-Sent Events)
   */
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

  /**
   * 6. Text-to-Speech (Sintesis Suara AI)
   */
  async textToSpeech(text: string, voice: "alloy" | "echo" | "fable" | "onyx" | "nova" | "shimmer" = "nova") {
    return await ai.audio.speak({
      text,
      voice,
    });
  }

  /**
   * 7. Speech-to-Text (Transkripsi Suara User)
   */
  async speechToText(audioBase64: string, language: string = "id") {
    return await ai.audio.transcribe({
      file: audioBase64,
      language,
    });
  }
}
```

---

## 4. 🎮 Layer Controller (`src/controllers/aiController.ts`)

Layer Controller bertugas menerima **Request HTTP Express (`req`)**, mengesahkan input parameter, memanggil Layer Service, dan mengembalikan **Response HTTP (`res`)**.

```typescript
// src/controllers/aiController.ts
import { Request, Response } from "express";
import { AIService } from "../services/aiService.js";

export class AIController {
  private aiService: AIService;

  constructor() {
    this.aiService = new AIService();
  }

  /**
   * POST /api/v1/ai/ingest-file — Ingesti PDF / File Teks
   */
  ingestFile = async (req: Request, res: Response): Promise<void> => {
    try {
      if (!req.file) {
        res.status(400).json({ success: false, error: "File PDF/Teks wajib diunggah!" });
        return;
      }

      const result = await this.aiService.ingestUploadedFile(
        req.file.buffer,
        req.file.originalname
      );

      res.status(200).json({ success: true, data: result });
    } catch (error: any) {
      res.status(500).json({ success: false, error: error.message });
    }
  };

  /**
   * POST /api/v1/ai/ingest-manual — Ingesti Teks Knowledge Manual / SOP
   */
  ingestManual = async (req: Request, res: Response): Promise<void> => {
    try {
      const { title, content } = req.body;
      if (!title || !content) {
        res.status(400).json({ success: false, error: "Parameter 'title' dan 'content' wajib diisi!" });
        return;
      }

      const result = await this.aiService.ingestManualKnowledgeText(title, content);
      res.status(200).json({ success: true, data: result });
    } catch (error: any) {
      res.status(500).json({ success: false, error: error.message });
    }
  };

  /**
   * POST /api/v1/ai/ingest-db — Ingesti Data Tabel Database
   */
  ingestDatabase = async (req: Request, res: Response): Promise<void> => {
    try {
      const { records, tableName } = req.body;
      if (!records || !Array.isArray(records) || !tableName) {
        res.status(400).json({ success: false, error: "Body request 'records' dan 'tableName' tidak valid!" });
        return;
      }

      const result = await this.aiService.ingestCompanyDatabaseRecords(records, tableName);
      res.status(200).json({ success: true, data: result });
    } catch (error: any) {
      res.status(500).json({ success: false, error: error.message });
    }
  };

  /**
   * POST /api/v1/ai/chat — Chat RAG & Multimodal Vision AI
   */
  chat = async (req: Request, res: Response): Promise<void> => {
    try {
      const { question, imageBase64 } = req.body;
      if (!question || typeof question !== "string") {
        res.status(400).json({ success: false, error: "Parameter 'question' wajib diisi!" });
        return;
      }

      const result = await this.aiService.generateChatAnswer(question, imageBase64);
      res.status(200).json({ success: true, data: result });
    } catch (error: any) {
      res.status(500).json({ success: false, error: error.message });
    }
  };

  /**
   * POST /api/v1/ai/chat-stream — Streaming Chat Completion SSE
   */
  chatStream = async (req: Request, res: Response): Promise<void> => {
    try {
      const { question } = req.body;
      if (!question) {
        res.status(400).json({ success: false, error: "Parameter 'question' wajib diisi!" });
        return;
      }

      // Set Header Server-Sent Events (SSE)
      res.setHeader("Content-Type", "text/event-stream");
      res.setHeader("Cache-Control", "no-cache");
      res.setHeader("Connection", "keep-alive");

      await this.aiService.streamChatAnswer(question, (chunkText) => {
        res.write(`data: ${JSON.stringify({ text: chunkText })}\n\n`);
      });

      res.write(`data: [DONE]\n\n`);
      res.end();
    } catch (error: any) {
      res.status(500).json({ success: false, error: error.message });
    }
  };

  /**
   * POST /api/v1/ai/audio/speak — Text-to-Speech (TTS MP3 Stream)
   */
  speak = async (req: Request, res: Response): Promise<void> => {
    try {
      const { text, voice } = req.body;
      if (!text) {
        res.status(400).json({ success: false, error: "Parameter 'text' wajib diisi!" });
        return;
      }

      const audioResult = await this.aiService.textToSpeech(text, voice);
      
      res.setHeader("Content-Type", audioResult.mimeType);
      res.send(audioResult.audio);
    } catch (error: any) {
      res.status(500).json({ success: false, error: error.message });
    }
  };

  /**
   * POST /api/v1/ai/audio/transcribe — Speech-to-Text (STT)
   */
  transcribe = async (req: Request, res: Response): Promise<void> => {
    try {
      const { audioBase64, language } = req.body;
      if (!audioBase64) {
        res.status(400).json({ success: false, error: "Parameter 'audioBase64' wajib diisi!" });
        return;
      }

      const transcriptResult = await this.aiService.speechToText(audioBase64, language);
      res.status(200).json({ success: true, data: transcriptResult });
    } catch (error: any) {
      res.status(500).json({ success: false, error: error.message });
    }
  };
}
```

---

## 5. 🛣️ Layer Router (`src/routes/aiRoutes.ts`)

```typescript
// src/routes/aiRoutes.ts
import { Router } from "express";
import multer from "multer";
import { AIController } from "../controllers/aiController.js";

const upload = multer({ 
  storage: multer.memoryStorage(),
  limits: { fileSize: 15 * 1024 * 1024 }
});

const router = Router();
const aiController = new AIController();

// 📌 Routing 7 API Endpoints Lengkap
router.post("/ingest-file", upload.single("file"), aiController.ingestFile);
router.post("/ingest-manual", aiController.ingestManual);
router.post("/ingest-db", aiController.ingestDatabase);
router.post("/chat", aiController.chat);
router.post("/chat-stream", aiController.chatStream);
router.post("/audio/speak", aiController.speak);
router.post("/audio/transcribe", aiController.transcribe);

export default router;
```

---

## 6. 🚦 Main Server Entry Point (`src/server.ts`)

```typescript
// src/server.ts
import express from "express";
import cors from "cors";
import dotenv from "dotenv";
import aiRoutes from "./routes/aiRoutes.js";

dotenv.config();

const app = express();
const PORT = process.env.PORT || 3000;

app.use(cors());
app.use(express.json({ limit: "20mb" }));
app.use(express.urlencoded({ extended: true }));

app.use("/api/v1/ai", aiRoutes);

app.listen(PORT, () => {
  console.log(`🚀 Express.js AI Server running on http://localhost:${PORT}`);
  console.log(`📡 Endpoints:`);
  console.log(`   - POST http://localhost:${PORT}/api/v1/ai/ingest-file`);
  console.log(`   - POST http://localhost:${PORT}/api/v1/ai/ingest-manual`);
  console.log(`   - POST http://localhost:${PORT}/api/v1/ai/ingest-db`);
  console.log(`   - POST http://localhost:${PORT}/api/v1/ai/chat`);
  console.log(`   - POST http://localhost:${PORT}/api/v1/ai/chat-stream`);
  console.log(`   - POST http://localhost:${PORT}/api/v1/ai/audio/speak`);
  console.log(`   - POST http://localhost:${PORT}/api/v1/ai/audio/transcribe`);
});
```
