# 🚀 Panduan Arsitektur Enterprise Express.js: Router ➔ Controller ➔ Service ➔ Repository (Powered by `@badriana/ai-core`)

Dokumen ini berisi panduan arsitektur **Clean Layered Architecture (Router - Controller - Service - Repository)** untuk aplikasi backend berbasis **Express.js & TypeScript** yang menggunakan paket `@badriana/ai-core` secara 100% komprehensif, berdasarkan implementasi nyata dari proyek `tes-si-core/backend`.

---

## 🏗️ Struktur Folder Proyek Backend Express.js

```text
tes-si-core-backend/
├── src/
│   ├── config/
│   │   ├── ai.ts                    # Single Instance Client @badriana/ai-core & OpenRouter
│   │   ├── prisma.ts                # Client ORM Prisma Database (PostgreSQL / SQLite)
│   │   └── cloudinary.ts            # Integrasi Storage Gambar Publik Cloudinary
│   ├── repositories/
│   │   └── knowledgeRepository.ts   # Layer Akses Database & Cosine Similarity Vector Search
│   ├── services/
│   │   └── aiService.ts             # Layer Logika Bisnis Utama (Delegasi Murni @badriana/ai-core)
│   ├── controllers/
│   │   ├── adminController.ts       # Handler HTTP Admin (Ingesti File, Ingesti Text, Ingesti DB)
│   │   └── userController.ts        # Handler HTTP User (Chat RAG, Streaming SSE, TTS MP3, STT)
│   ├── routes/
│   │   ├── adminRoutes.ts           # Route Express Prefix /api/v1/admin
│   │   └── userRoutes.ts            # Route Express Prefix /api/v1/user
│   └── server.ts                    # Entry Point Utama Express.js Server
├── prisma/
│   └── schema.prisma                # Skema Tabel Document & KnowledgeChunk
├── package.json
├── tsconfig.json
└── .env
```

---

## 1. ⚙️ Layer Konfigurasi Client AI (`src/config/ai.ts`)

Inisialisasi instance terpusat `@badriana/ai-core` dengan per-capability provider:

```typescript
// src/config/ai.ts
import { createAI } from "@badriana/ai-core";

const apiKey = process.env.OPENROUTER_API_KEY;
if (!apiKey) {
  throw new Error("❌ OPENROUTER_API_KEY belum diset pada environment variables (.env)!");
}

export const ai = createAI({
  provider: "openrouter",
  apiKey: apiKey,
  chatModel: process.env.AI_CHAT_MODEL || "google/gemma-2-9b-it:free",
  embeddingModel: process.env.AI_EMBEDDING_MODEL || "openai/text-embedding-3-small",
  visionModel: process.env.AI_VISION_MODEL || "meta-llama/llama-3.2-11b-vision-instruct:free",
});

export const aiAudio = createAI({
  provider: "openai",
  apiKey: process.env.OPENAI_API_KEY || apiKey,
});
```

---

## 2. 🗄️ Layer Repository (`src/repositories/knowledgeRepository.ts`)

Layer ini menangani penyimpanan potongan Vektor Chunk ke database via Prisma ORM dan melakukan perhitungan Cosine Similarity:

```typescript
// src/repositories/knowledgeRepository.ts
import { prisma } from "../config/prisma.js";

export interface VectorChunkRecord {
  id: string;
  documentId?: string | null;
  type: string;
  content: string;
  similarityScore: number;
  imageUrl?: string | null;
  imagePage?: number | null;
}

export class KnowledgeRepository {
  /**
   * Menyimpan Chunks RAG hasil createDocument() ke tabel Database KnowledgeChunk
   */
  async saveChunks(
    documentId: string | null,
    chunks: readonly {
      id: string;
      content: string;
      type?: string;
      embedding?: readonly number[];
      imageUrl?: string | null;
      imagePage?: number | null;
    }[]
  ): Promise<void> {
    for (const chunk of chunks) {
      const emb = chunk.embedding ? Array.from(chunk.embedding) : [];
      await prisma.knowledgeChunk.upsert({
        where: { id: chunk.id },
        update: {
          content: chunk.content,
          embedding: JSON.stringify(emb),
          documentId,
          type: chunk.type ?? "text",
          imageUrl: chunk.imageUrl ?? null,
          imagePage: chunk.imagePage ?? null,
        },
        create: {
          id: chunk.id,
          documentId,
          type: chunk.type ?? "text",
          content: chunk.content,
          embedding: JSON.stringify(emb),
          imageUrl: chunk.imageUrl ?? null,
          imagePage: chunk.imagePage ?? null,
        },
      });
    }
  }

  /**
   * Cari Chunks paling relevan berdasarkan Cosine Similarity
   */
  async searchSimilarChunks(questionVector: number[], limit: number = 5): Promise<VectorChunkRecord[]> {
    const allChunks = await prisma.knowledgeChunk.findMany();

    const scoredChunks = allChunks.map((chunk) => {
      let chunkVector: number[] = [];
      try {
        chunkVector = JSON.parse(chunk.embedding);
      } catch (e) {
        chunkVector = [];
      }

      const score = this.calculateCosineSimilarity(questionVector, chunkVector);
      return {
        id: chunk.id,
        documentId: chunk.documentId,
        type: chunk.type,
        content: chunk.content,
        similarityScore: score,
        imageUrl: chunk.imageUrl,
        imagePage: chunk.imagePage,
      };
    });

    scoredChunks.sort((a, b) => b.similarityScore - a.similarityScore);
    return scoredChunks.slice(0, limit);
  }

  private calculateCosineSimilarity(vecA: number[], vecB: number[]): number {
    if (!vecA.length || !vecB.length || vecA.length !== vecB.length) return 0;
    let dotProduct = 0;
    let normA = 0;
    let normB = 0;
    for (let i = 0; i < vecA.length; i++) {
      const a = vecA[i] ?? 0;
      const b = vecB[i] ?? 0;
      dotProduct += a * b;
      normA += a * a;
      normB += b * b;
    }
    if (normA === 0 || normB === 0) return 0;
    return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
  }
}
```

---

## 3. 🧠 Layer Service (`src/services/aiService.ts`)

Service Layer melakukan delegasi murni ke API `@badriana/ai-core` tanpa peretasan regex atau kata kunci palsu:

```typescript
// src/services/aiService.ts
import { createDocument } from "@badriana/ai-core";
import { ai, aiAudio } from "../config/ai.js";
import { prisma } from "../config/prisma.js";
import { uploadImageToCloudinary } from "../config/cloudinary.js";
import { KnowledgeRepository } from "../repositories/knowledgeRepository.js";
import { formatDataRecordsForIngestion } from "../utils/formatRecords.js";

const RAG_MIN_SCORE = parseFloat(process.env.RAG_MIN_SCORE ?? "0.15") || 0.15;

export class AIService {
  private knowledgeRepo: KnowledgeRepository;

  constructor() {
    this.knowledgeRepo = new KnowledgeRepository();
  }

  private async getEmbeddings(texts: readonly string[]): Promise<number[][]> {
    const embRes = await ai.embedding.create({ input: texts });
    return "embeddings" in embRes
      ? embRes.embeddings.map((e) => Array.from(e))
      : [Array.from(embRes.embedding)];
  }

  /**
   * 1. Ingesti Berkas PDF/TXT
   */
  async ingestUploadedFile(fileBuffer: Buffer, filename: string) {
    const docMeta = await prisma.document.create({ data: { filename } });
    const pendingUploads: { dataUrl: string; counter: number; page?: number }[] = [];
    let imageCounter = 0;

    const docResult = await createDocument({
      source: fileBuffer,
      filename,
      filterCoverImages: false,
      minImageDimension: 80,
      embedder: async (texts) => await this.getEmbeddings(texts),
      imageProcessor: async (image) => {
        imageCounter++;
        pendingUploads.push({ dataUrl: image.dataUrl, counter: imageCounter, page: image.page });
        let description = image.contextCaption ?? `Gambar ${imageCounter} pada dokumen "${filename}".`;
        try {
          const vision = await ai.image.describe(image.dataUrl);
          description = (vision?.text as string | undefined) || description;
        } catch {
          // fallback description
        }
        return {
          content: description,
          title: image.contextCaption ?? `Gambar ${imageCounter}`,
          caption: image.contextCaption,
          kind: "diagram",
        };
      },
    });

    const uploaded: { dataUrl: string; url: string | null; page?: number }[] = [];
    for (const pending of pendingUploads) {
      const publicId = `doc_${docMeta.id}_img_${pending.counter}`;
      const url = await uploadImageToCloudinary(pending.dataUrl, publicId).catch(() => null);
      uploaded.push({ dataUrl: pending.dataUrl, url, page: pending.page });
    }

    const imageUrlMap = new Map<string, { url: string; page?: number }>();
    for (const up of uploaded) {
      if (up.url) imageUrlMap.set(up.dataUrl, { url: up.url, page: up.page });
    }

    const chunksToSave = docResult.chunks.map((chunk) => {
      if (chunk.type === "image") {
        const stored = imageUrlMap.get(chunk.image.dataUrl);
        return {
          id: chunk.id,
          content: chunk.content,
          embedding: chunk.embedding,
          type: "image",
          imageUrl: stored?.url || null,
          imagePage: stored?.page ?? (chunk.metadata?.page as number | undefined) ?? null,
        };
      }
      return {
        id: chunk.id,
        content: chunk.content,
        embedding: chunk.embedding,
        type: "text",
        imageUrl: null,
        imagePage: null,
      };
    });

    await this.knowledgeRepo.saveChunks(docMeta.id, chunksToSave);
    return { document: docMeta, stats: docResult.stats, chunksCount: chunksToSave.length };
  }

  /**
   * 2. Tanya Jawab AI Chat RAG
   */
  async generateChatAnswer(question: string, imageBase64?: string) {
    const questionEmbeddings = await this.getEmbeddings([question]);
    const questionVector = questionEmbeddings[0] ?? [];

    const similarChunks = await this.knowledgeRepo.searchSimilarChunks(questionVector, 5);
    const relevantChunks = similarChunks.filter((c) => c.similarityScore >= RAG_MIN_SCORE);

    const contexts = relevantChunks.map((c) => ({
      id: c.id,
      content: c.content,
      source: { filename: `Document ${c.documentId ?? 'Knowledge'}`, page: c.imagePage ?? undefined },
    }));

    const answer = await ai.chat.generate({
      question,
      contexts,
    });

    const relevantImages = relevantChunks
      .filter((c) => c.type === "image" && c.imageUrl)
      .map((c) => ({ id: c.id, url: c.imageUrl!, page: c.imagePage ?? undefined }));

    return {
      answer: answer.text,
      model: answer.model,
      usage: answer.usage,
      citations: answer.citations,
      images: relevantImages,
    };
  }
}
```

---

## 4. 🎮 Layer Controller (`src/controllers/adminController.ts` & `userController.ts`)

Layer controller mengolah HTTP Request & Response:

```typescript
// src/controllers/adminController.ts
import { Request, Response } from "express";
import { AIService } from "../services/aiService.js";

export class AdminController {
  private aiService = new AIService();

  ingestFile = async (req: Request, res: Response): Promise<void> => {
    try {
      const file: any = req.file || (req.files && Array.isArray(req.files) ? req.files[0] : undefined);
      if (!file) {
        res.status(400).json({ success: false, error: "File PDF/TXT wajib diunggah!" });
        return;
      }

      const result = await this.aiService.ingestUploadedFile(file.buffer, file.originalname);
      res.status(200).json({ success: true, data: result });
    } catch (error: any) {
      res.status(500).json({ success: false, error: error.message });
    }
  };
}
```

---

## 5. 🚥 Layer Router (`src/routes/adminRoutes.ts` & `userRoutes.ts`)

```typescript
// src/routes/adminRoutes.ts
import { Router } from "express";
import multer from "multer";
import { AdminController } from "../controllers/adminController.js";

const router = Router();
const upload = multer({ storage: multer.memoryStorage() });
const adminCtrl = new AdminController();

router.post("/ingest-file", upload.single("file"), adminCtrl.ingestFile);

export default router;
```

---

## 6. 🚀 Entry Point Server Express (`src/server.ts`)

```typescript
// src/server.ts
import express from "express";
import cors from "cors";
import dotenv from "dotenv";
import adminRoutes from "./routes/adminRoutes.js";
import userRoutes from "./routes/userRoutes.js";

dotenv.config();

const app = express();
const PORT = process.env.PORT || 3000;

app.use(cors({ origin: true, credentials: true }));
app.use(express.json({ limit: "25mb" }));

app.use("/api/v1/admin", adminRoutes);
app.use("/api/v1/user", userRoutes);

app.listen(PORT, () => {
  console.log(`🚀 Express.js Server running on http://localhost:${PORT}`);
});
```

---
*Dokumentasi Arsitektur Resmi Express.js (`@badriana/ai-core`)*
