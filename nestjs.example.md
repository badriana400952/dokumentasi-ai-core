# 🦁 Panduan Arsitektur Enterprise NestJS: Modular Controller ➔ Service ➔ Repository (Powered by `@badriana/ai-core`)

Dokumen ini berisi panduan arsitektur **Enterprise Modular Architecture (Controller - Service - Repository / Provider)** untuk aplikasi backend berbasis **NestJS & TypeScript** yang menggunakan paket `@badriana/ai-core` secara 100% komprehensif, mengadaptasi logika bisnis dari proyek `tes-si-core/backend` ke dalam pola idiomatik NestJS (Dependency Injection, NestJS DTOs, FileInterceptor, RxJS SSE Observables, & Prisma Service).

---

## 🏗️ Struktur Folder Proyek Backend NestJS

```text
tes-si-core-nestjs/
├── src/
│   ├── ai/
│   │   ├── dto/
│   │   │   ├── ingest-manual.dto.ts     # DTO Ingesti Teks Manual / SOP
│   │   │   ├── ingest-database.dto.ts   # DTO Ingesti Baris Database
│   │   │   ├── chat-query.dto.ts        # DTO Chat RAG & Multimodal
│   │   │   └── audio-speak.dto.ts       # DTO Text-to-Speech MP3
│   │   ├── ai.config.ts                 # Provider Single Instance @badriana/ai-core
│   │   ├── prisma.service.ts            # NestJS Service Wraps Prisma ORM
│   │   ├── knowledge.repository.ts      # Repository Layer (Cos Similarity Vector Search)
│   │   ├── ai.service.ts                # Service Layer Logika Bisnis Utama
│   │   ├── admin.controller.ts          # Controller Admin (Ingesti File, Manual, DB)
│   │   ├── user.controller.ts           # Controller User (Chat RAG, SSE Stream, TTS, STT)
│   │   └── ai.module.ts                 # NestJS Feature Module
│   ├── app.module.ts                    # Root Module Aplikasi NestJS
│   └── main.ts                          # Bootstrap Entry Point NestJS
├── prisma/
│   └── schema.prisma                    # Skema Tabel Document & KnowledgeChunk
├── package.json
├── tsconfig.json
└── .env
```

---

## 1. ⚙️ Provider Konfigurasi AI Client (`src/ai/ai.config.ts`)

Menggunakan NestJS `@Injectable()` provider untuk mengelola client `@badriana/ai-core`:

```typescript
// src/ai/ai.config.ts
import { Injectable } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import { createAI } from "@badriana/ai-core";

@Injectable()
export class AIConfigProvider {
  public readonly ai: ReturnType<typeof createAI>;
  public readonly aiAudio: ReturnType<typeof createAI>;

  constructor(private configService: ConfigService) {
    const apiKey = this.configService.get<string>("OPENROUTER_API_KEY");
    if (!apiKey) {
      throw new Error("❌ OPENROUTER_API_KEY belum diset pada environment variables (.env)!");
    }

    this.ai = createAI({
      provider: "openrouter",
      apiKey,
      chatModel: this.configService.get<string>("AI_CHAT_MODEL") || "google/gemma-2-9b-it:free",
      embeddingModel: this.configService.get<string>("AI_EMBEDDING_MODEL") || "openai/text-embedding-3-small",
      visionModel: this.configService.get<string>("AI_VISION_MODEL") || "meta-llama/llama-3.2-11b-vision-instruct:free",
    });

    this.aiAudio = createAI({
      provider: "openai",
      apiKey: this.configService.get<string>("OPENAI_API_KEY") || apiKey,
    });
  }
}
```

---

## 2. 🗄️ Layer Repository (`src/ai/knowledge.repository.ts`)

Layer repository NestJS menggunakan PrismaService untuk menyimpan Vektor Chunks dan menghitung Cosine Similarity:

```typescript
// src/ai/knowledge.repository.ts
import { Injectable } from "@nestjs/common";
import { PrismaService } from "./prisma.service";

export interface VectorChunkRecord {
  id: string;
  documentId?: string | null;
  type: string;
  content: string;
  similarityScore: number;
  imageUrl?: string | null;
  imagePage?: number | null;
}

@Injectable()
export class KnowledgeRepository {
  constructor(private prisma: PrismaService) {}

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
      await this.prisma.knowledgeChunk.upsert({
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
    const allChunks = await this.prisma.knowledgeChunk.findMany();

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

## 3. 🧠 Layer Service (`src/ai/ai.service.ts`)

Injectable NestJS Service yang mengeksekusi pipeline ingesti `createDocument` dan RAG chat `ai.chat.generate`:

```typescript
// src/ai/ai.service.ts
import { Injectable } from "@nestjs/common";
import { createDocument } from "@badriana/ai-core";
import { AIConfigProvider } from "./ai.config";
import { KnowledgeRepository } from "./knowledge.repository";
import { PrismaService } from "./prisma.service";
import { Observable } from "rxjs";

const RAG_MIN_SCORE = 0.15;

@Injectable()
export class AIService {
  constructor(
    private aiConfig: AIConfigProvider,
    private knowledgeRepo: KnowledgeRepository,
    private prisma: PrismaService,
  ) {}

  private async getEmbeddings(texts: readonly string[]): Promise<number[][]> {
    const embRes = await this.aiConfig.ai.embedding.create({ input: texts });
    return "embeddings" in embRes
      ? embRes.embeddings.map((e) => Array.from(e))
      : [Array.from(embRes.embedding)];
  }

  /**
   * 1. Ingesti Berkas PDF/TXT yang diunggah via Express/NestJS Multer Buffer
   */
  async ingestUploadedFile(fileBuffer: Buffer, filename: string) {
    const docMeta = await this.prisma.document.create({ data: { filename } });

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

    const answer = await this.aiConfig.ai.chat.generate({
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
   * 3. Streaming SSE Chat RAG via RxJS Observable
   */
  streamChatAnswer(question: string): Observable<MessageEvent> {
    return new Observable((subscriber) => {
      (async () => {
        try {
          const questionEmbeddings = await this.getEmbeddings([question]);
          const questionVector = questionEmbeddings[0] ?? [];
          const similarChunks = await this.knowledgeRepo.searchSimilarChunks(questionVector, 5);
          const contexts = similarChunks.map((c) => ({ id: c.id, content: c.content }));

          const stream = await this.aiConfig.ai.chat.stream({ question, contexts });
          for await (const chunk of stream) {
            subscriber.next({ data: JSON.stringify({ text: chunk.text }) } as MessageEvent);
          }
          subscriber.next({ data: '[DONE]' } as MessageEvent);
          subscriber.complete();
        } catch (err: any) {
          subscriber.error(err);
        }
      })();
    });
  }
}
```

---

## 4. 🎮 Layer Controller (`src/ai/admin.controller.ts` & `user.controller.ts`)

Menggunakan dekorator standar NestJS `@Controller`, `@Post`, `@UploadedFile`, `@UseInterceptors(FileInterceptor)`:

```typescript
// src/ai/admin.controller.ts
import { Controller, Post, UseInterceptors, UploadedFile, BadRequestException } from "@nestjs/common";
import { FileInterceptor } from "@nestjs/platform-express";
import { AIService } from "./ai.service";

@Controller("api/v1/admin")
export class AdminController {
  constructor(private readonly aiService: AIService) {}

  @Post("ingest-file")
  @UseInterceptors(FileInterceptor("file"))
  async ingestFile(@UploadedFile() file: Express.Multer.File) {
    if (!file) {
      throw new BadRequestException("File PDF/TXT wajib diunggah!");
    }
    const result = await this.aiService.ingestUploadedFile(file.buffer, file.originalname);
    return { success: true, data: result };
  }
}
```

```typescript
// src/ai/user.controller.ts
import { Controller, Post, Body, Sse, MessageEvent } from "@nestjs/common";
import { AIService } from "./ai.service";
import { ChatQueryDto } from "./dto/chat-query.dto";
import { Observable } from "rxjs";

@Controller("api/v1/user")
export class UserController {
  constructor(private readonly aiService: AIService) {}

  @Post("chat")
  async chat(@Body() dto: ChatQueryDto) {
    const result = await this.aiService.generateChatAnswer(dto.question);
    return { success: true, data: result };
  }

  @Sse("chat-stream")
  chatStream(@Body() dto: ChatQueryDto): Observable<MessageEvent> {
    return this.aiService.streamChatAnswer(dto.question);
  }
}
```

---

## 5. 🧱 Module Definition & Bootstrap Main (`src/ai/ai.module.ts` & `src/main.ts`)

```typescript
// src/ai/ai.module.ts
import { Module } from "@nestjs/common";
import { ConfigModule } from "@nestjs/config";
import { AIConfigProvider } from "./ai.config";
import { PrismaService } from "./prisma.service";
import { KnowledgeRepository } from "./knowledge.repository";
import { AIService } from "./ai.service";
import { AdminController } from "./admin.controller";
import { UserController } from "./user.controller";

@Module({
  imports: [ConfigModule.forRoot()],
  controllers: [AdminController, UserController],
  providers: [AIConfigProvider, PrismaService, KnowledgeRepository, AIService],
  exports: [AIService],
})
export class AIModule {}
```

```typescript
// src/main.ts
import { NestFactory } from "@nestjs/core";
import { ValidationPipe } from "@nestjs/common";
import { AppModule } from "./app.module";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableCors({ origin: true, credentials: true });
  app.useGlobalPipes(new ValidationPipe({ transform: true }));

  const port = process.env.PORT || 3000;
  await app.listen(port);
  console.log(`🚀 NestJS Backend Server running on http://localhost:${port}`);
}
bootstrap();
```

---
*Dokumentasi Arsitektur Resmi NestJS (`@badriana/ai-core`)*
