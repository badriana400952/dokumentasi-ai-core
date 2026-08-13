# ⚛️ Panduan Arsitektur Enterprise Next.js App Router (14/15): Route Handlers ➔ Service ➔ Repository (Powered by `@badriana/ai-core`)

Dokumen ini berisi panduan arsitektur **Clean Enterprise Layered Architecture (App Router Route Handlers - Service - Repository)** untuk aplikasi full-stack berbasis **Next.js 14/15 (App Router & TypeScript)** yang menggunakan paket `@badriana/ai-core` secara 100% komprehensif, mengadaptasi logika bisnis dari proyek `tes-si-core/backend` ke dalam pola idiomatik Next.js (App Router Route Handlers, `NextRequest` & `NextResponse`, Web Standard `ReadableStream` untuk Streaming SSE, `formData()` Native, & Prisma ORM).

---

## 🏗️ Struktur Folder Proyek Next.js App Router

```text
tes-si-core-nextjs/
├── src/
│   ├── app/
│   │   ├── api/
│   │   │   └── v1/
│   │   │       ├── admin/
│   │   │       │   ├── ingest-file/
│   │   │       │   │   └── route.ts       # POST Route Handler: Upload File PDF/TXT (formData)
│   │   │       │   ├── ingest-manual/
│   │   │       │   │   └── route.ts       # POST Route Handler: Input Teks Knowledge Manual / SOP
│   │   │       │   └── ingest-db/
│   │   │       │       └── route.ts       # POST Route Handler: Ingesti Baris Database
│   │   │       └── user/
│   │   │           ├── chat/
│   │   │           │   └── route.ts       # POST Route Handler: RAG & Multimodal Vision Chat
│   │   │           ├── chat-stream/
│   │   │           │   └── route.ts       # POST Route Handler: Real-Time SSE Chat Streaming
│   │   │           └── audio/
│   │   │               ├── speak/
│   │   │               │   └── route.ts   # POST Route Handler: Text-to-Speech (MP3 Buffer)
│   │   │               └── transcribe/
│   │   │                   └── route.ts   # POST Route Handler: Speech-to-Text Whisper
│   │   └── page.tsx                       # Client Component UI Dashboard (Admin & User Portal)
│   └── lib/
│       ├── ai.ts                          # Singleton Instance Client @badriana/ai-core
│       ├── prisma.ts                      # Singleton Instance Client Prisma ORM
│       ├── cloudinary.ts                  # Integrasi Storage Gambar Publik Cloudinary
│       ├── repositories/
│       │   └── knowledgeRepository.ts     # Layer Akses Database & Cosine Similarity Vector Search
│       └── services/
│           └── aiService.ts               # Layer Logika Bisnis Utama (@badriana/ai-core)
├── prisma/
│   └── schema.prisma                      # Skema Tabel Document & KnowledgeChunk
├── package.json
├── tsconfig.json
└── .env.local
```

---

## 1. ⚙️ Layer Konfigurasi Client AI (`src/lib/ai.ts`)

Menggunakan instance terpusat `@badriana/ai-core` di lingkungan Next.js Server-Side:

```typescript
// src/lib/ai.ts
import { createAI } from "@badriana/ai-core";

const apiKey = process.env.OPENROUTER_API_KEY;
if (!apiKey) {
  throw new Error("❌ OPENROUTER_API_KEY belum diset pada environment variables (.env.local)!");
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

## 2. 🗄️ Layer Repository (`src/lib/repositories/knowledgeRepository.ts`)

Layer repository mengolah penyimpanan potongan Vektor Chunk ke database PostgreSQL / SQLite via Prisma ORM:

```typescript
// src/lib/repositories/knowledgeRepository.ts
import { prisma } from "../prisma";

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

## 3. 🧠 Layer Service (`src/lib/services/aiService.ts`)

Layer service yang melakukan pemanggilan API `@badriana/ai-core`:

```typescript
// src/lib/services/aiService.ts
import { createDocument } from "@badriana/ai-core";
import { ai, aiAudio } from "../ai";
import { prisma } from "../prisma";
import { uploadImageToCloudinary } from "../cloudinary";
import { KnowledgeRepository } from "../repositories/knowledgeRepository";

const RAG_MIN_SCORE = parseFloat(process.env.RAG_MIN_SCORE ?? "0.15") || 0.15;

export class AIService {
  private knowledgeRepo = new KnowledgeRepository();

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

    const docResult = await createDocument({
      source: fileBuffer,
      filename,
      filterCoverImages: false,
      minImageDimension: 80,
      embedder: async (texts) => await this.getEmbeddings(texts),
    });

    const chunksToSave = docResult.chunks.map((chunk) => ({
      id: chunk.id,
      content: chunk.content,
      embedding: chunk.embedding,
      type: chunk.type,
      imageUrl: null,
      imagePage: null,
    }));

    await this.knowledgeRepo.saveChunks(docMeta.id, chunksToSave);
    return { document: docMeta, stats: docResult.stats, chunksCount: chunksToSave.length };
  }

  /**
   * 2. Tanya Jawab AI Chat RAG
   */
  async generateChatAnswer(question: string) {
    const questionEmbeddings = await this.getEmbeddings([question]);
    const questionVector = questionEmbeddings[0] ?? [];

    const similarChunks = await this.knowledgeRepo.searchSimilarChunks(questionVector, 5);
    const relevantChunks = similarChunks.filter((c) => c.similarityScore >= RAG_MIN_SCORE);

    const contexts = relevantChunks.map((c) => ({
      id: c.id,
      content: c.content,
      source: { filename: `Document ${c.documentId ?? 'Knowledge'}` },
    }));

    const answer = await ai.chat.generate({
      question,
      contexts,
    });

    return {
      answer: answer.text,
      model: answer.model,
      usage: answer.usage,
      citations: answer.citations,
    };
  }

  /**
   * 3. Streaming Chat SSE RAG untuk ReadableStream Next.js App Router
   */
  async streamChatAnswer(question: string, onChunk: (text: string) => void) {
    const questionEmbeddings = await this.getEmbeddings([question]);
    const questionVector = questionEmbeddings[0] ?? [];

    const similarChunks = await this.knowledgeRepo.searchSimilarChunks(questionVector, 5);
    const contexts = similarChunks.map((c) => ({ id: c.id, content: c.content }));

    const stream = await ai.chat.stream({ question, contexts });
    for await (const chunk of stream) {
      onChunk(chunk.text);
    }
  }
}
```

---

## 4. 🚀 Next.js App Router Route Handlers (`src/app/api/.../route.ts`)

### A. Upload File PDF/TXT Route Handler (`src/app/api/v1/admin/ingest-file/route.ts`)
Menggunakan Web Standard `NextRequest` dan `request.formData()` untuk menerima file biner:

```typescript
// src/app/api/v1/admin/ingest-file/route.ts
import { NextRequest, NextResponse } from "next/server";
import { AIService } from "@/lib/services/aiService";

export const runtime = "nodejs";

export async function POST(request: NextRequest) {
  try {
    const formData = await request.formData();
    const file = formData.get("file") as File | null;

    if (!file) {
      return NextResponse.json({ success: false, error: "File PDF/TXT wajib diunggah!" }, { status: 400 });
    }

    const bytes = await file.arrayBuffer();
    const buffer = Buffer.from(bytes);

    const aiService = new AIService();
    const result = await aiService.ingestUploadedFile(buffer, file.name);

    return NextResponse.json({ success: true, data: result });
  } catch (error: any) {
    return NextResponse.json({ success: false, error: error.message }, { status: 500 });
  }
}
```

---

### B. User Chat RAG Route Handler (`src/app/api/v1/user/chat/route.ts`)

```typescript
// src/app/api/v1/user/chat/route.ts
import { NextRequest, NextResponse } from "next/server";
import { AIService } from "@/lib/services/aiService";

export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const { question } = body;

    if (!question) {
      return NextResponse.json({ success: false, error: "Parameter 'question' wajib diisi!" }, { status: 400 });
    }

    const aiService = new AIService();
    const result = await aiService.generateChatAnswer(question);

    return NextResponse.json({ success: true, data: result });
  } catch (error: any) {
    return NextResponse.json({ success: false, error: error.message }, { status: 500 });
  }
}
```

---

### C. Real-Time SSE Chat Streaming Route Handler (`src/app/api/v1/user/chat-stream/route.ts`)
Menggunakan Web Standard `ReadableStream` & `TextEncoder` khas Next.js App Router:

```typescript
// src/app/api/v1/user/chat-stream/route.ts
import { NextRequest } from "next/server";
import { AIService } from "@/lib/services/aiService";

export const runtime = "nodejs";

export async function POST(request: NextRequest) {
  const body = await request.json();
  const { question } = body;

  const encoder = new TextEncoder();
  const aiService = new AIService();

  const stream = new ReadableStream({
    async start(controller) {
      try {
        await aiService.streamChatAnswer(question, (chunkText) => {
          controller.enqueue(encoder.encode(`data: ${JSON.stringify({ text: chunkText })}\n\n`));
        });
        controller.enqueue(encoder.encode(`data: [DONE]\n\n`));
        controller.close();
      } catch (err: any) {
        controller.error(err);
      }
    },
  });

  return new Response(stream, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache, no-transform",
      "Connection": "keep-alive",
    },
  });
}
```

---
*Dokumentasi Arsitektur Resmi Next.js App Router (`@badriana/ai-core`)*
