# 🦁 Panduan Arsitektur Enterprise NestJS: Controller ➔ Service ➔ Repository (Powered by `@badriana/ai-core`)

Dokumen ini berisi panduan arsitektur **Enterprise Modular Architecture (Controller - Service - Repository / Provider)** untuk aplikasi backend enterprise berbasis **NestJS & TypeScript** yang memanfaatkan paket `@badriana/ai-core` secara 100% komprehensif (Upload File PDF RAG, Ingesti Database 40+ Tabel, Ingesti Teks Knowledge Manual, Multimodal Vision Chat, Streaming Chat SSE, Text-to-Speech MP3, & Speech-to-Text).

---

## 🏗️ Struktur Folder Proyek Backend NestJS

```text
my-nestjs-ai-app/
├── src/
│   ├── ai/
│   │   ├── dto/
│   │   │   ├── ingest-manual.dto.ts     # DTO Ingesti Teks Manual / SOP
│   │   │   ├── ingest-database.dto.ts   # DTO Ingesti DB
│   │   │   ├── chat-query.dto.ts        # DTO Chat RAG
│   │   │   ├── text-to-speech.dto.ts    # DTO TTS
│   │   │   └── speech-to-text.dto.ts    # DTO STT
│   │   ├── ai.config.ts                 # Provider Single Instance @badriana/ai-core
│   │   ├── knowledge.repository.ts      # Repository Access Layer (PostgreSQL pgvector)
│   │   ├── ai.service.ts                # Business Logic Layer @badriana/ai-core
│   │   ├── ai.controller.ts             # Controller Layer Handler REST API & Multer
│   │   └── ai.module.ts                 # NestJS Module Definition
│   ├── app.module.ts                    # Root Module Aplikasi NestJS
│   └── main.ts                          # Bootstrap Entry Point NestJS
├── package.json
├── tsconfig.json
└── .env
```

---

## 1. ⚙️ Provider Konfigurasi AI Client (`src/ai/ai.config.ts`)

```typescript
// src/ai/ai.config.ts
import { Injectable } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import { createAI, Client } from "@badriana/ai-core";

@Injectable()
export class AIConfigProvider {
  public readonly client: Client;

  constructor(private configService: ConfigService) {
    const apiKey = this.configService.get<string>("OPENROUTER_API_KEY");
    if (!apiKey) {
      throw new Error("❌ OPENROUTER_API_KEY belum diset pada environment variables (.env)!");
    }

    this.client = createAI({
      provider: "openrouter",
      apiKey,
      chatModel: "google/gemma-2-9b-it:free",
      embeddingModel: "openai/text-embedding-3-small",
      visionModel: "meta-llama/llama-3.2-11b-vision-instruct:free",
    });
  }
}
```

---

## 2. 🗄️ Layer Repository (`src/ai/knowledge.repository.ts`)

```typescript
// src/ai/knowledge.repository.ts
import { Injectable, OnModuleDestroy } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import { Pool } from "pg";
import type { Chunk } from "@badriana/ai-core";

export interface VectorChunkRecord {
  id: string;
  documentId?: string;
  content: string;
  similarityScore: number;
}

@Injectable()
export class KnowledgeRepository implements OnModuleDestroy {
  private pgPool: Pool;

  constructor(private configService: ConfigService) {
    this.pgPool = new Pool({
      connectionString: this.configService.get<string>("DATABASE_URL"),
    });
  }

  onModuleDestroy() {
    this.pgPool.end();
  }

  async saveChunks(documentId: string, chunks: Chunk[]): Promise<void> {
    const client = await this.pgPool.connect();
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
    const result = await this.pgPool.query(query, [JSON.stringify(questionVector), limit]);
    return result.rows;
  }
}
```

---

## 3. 📝 Data Transfer Objects (DTO) Validation

```typescript
// src/ai/dto/ingest-manual.dto.ts
import { IsNotEmpty, IsString } from "class-validator";

export class IngestManualDto {
  @IsString()
  @IsNotEmpty()
  title: string;

  @IsString()
  @IsNotEmpty()
  content: string;
}

// src/ai/dto/ingest-database.dto.ts
import { IsArray, IsNotEmpty, IsString } from "class-validator";

export class IngestDatabaseDto {
  @IsArray()
  @IsNotEmpty()
  records: Record<string, any>[];

  @IsString()
  @IsNotEmpty()
  tableName: string;
}

// src/ai/dto/chat-query.dto.ts
import { IsNotEmpty, IsOptional, IsString } from "class-validator";

export class ChatQueryDto {
  @IsString()
  @IsNotEmpty()
  question: string;

  @IsString()
  @IsOptional()
  imageBase64?: string;
}

// src/ai/dto/text-to-speech.dto.ts
import { IsNotEmpty, IsOptional, IsString, IsIn } from "class-validator";

export class TextToSpeechDto {
  @IsString()
  @IsNotEmpty()
  text: string;

  @IsString()
  @IsOptional()
  @IsIn(["alloy", "echo", "fable", "onyx", "nova", "shimmer"])
  voice?: "alloy" | "echo" | "fable" | "onyx" | "nova" | "shimmer";
}

// src/ai/dto/speech-to-text.dto.ts
import { IsNotEmpty, IsOptional, IsString } from "class-validator";

export class SpeechToTextDto {
  @IsString()
  @IsNotEmpty()
  audioBase64: string;

  @IsString()
  @IsOptional()
  language?: string;
}
```

---

## 4. 🧠 Layer Service (`src/ai/ai.service.ts`)

```typescript
// src/ai/ai.service.ts
import { Injectable, InternalServerErrorException } from "@nestjs/common";
import { 
  createDocument, 
  createQuery, 
  formatDataRecordsForIngestion 
} from "@badriana/ai-core";
import { AIConfigProvider } from "./ai.config.js";
import { KnowledgeRepository } from "./knowledge.repository.js";
import { IngestManualDto } from "./dto/ingest-manual.dto.js";
import { IngestDatabaseDto } from "./dto/ingest-database.dto.js";
import { ChatQueryDto } from "./dto/chat-query.dto.js";
import { TextToSpeechDto } from "./dto/text-to-speech.dto.js";
import { SpeechToTextDto } from "./dto/speech-to-text.dto.js";

@Injectable()
export class AIService {
  constructor(
    private readonly aiConfig: AIConfigProvider,
    private readonly knowledgeRepo: KnowledgeRepository
  ) {}

  /**
   * 1. Ingesti Berkas PDF/Teks
   */
  async ingestUploadedFile(fileBuffer: Buffer, filename: string) {
    try {
      const docResult = await createDocument({
        source: fileBuffer,
        filename,
        embedder: async (texts) => {
          const embRes = await this.aiConfig.client.embedding.create({ input: texts });
          return "embeddings" in embRes ? embRes.embeddings : [embRes.embedding];
        },
      });

      await this.knowledgeRepo.saveChunks(docResult.document.id, docResult.chunks);

      return {
        documentId: docResult.document.id,
        chunksCount: docResult.chunks.length,
        stats: docResult.stats,
      };
    } catch (error: any) {
      throw new InternalServerErrorException(`Gagal meng-ingest file: ${error.message}`);
    }
  }

  /**
   * 2. Ingesti Teks Knowledge Manual / SOP
   */
  async ingestManualKnowledgeText(dto: IngestManualDto) {
    try {
      const docResult = await createDocument({
        source: dto.content,
        filename: `${dto.title.replace(/\s+/g, "_")}.txt`,
        embedder: async (texts) => {
          const embRes = await this.aiConfig.client.embedding.create({ input: texts });
          return "embeddings" in embRes ? embRes.embeddings : [embRes.embedding];
        },
      });

      await this.knowledgeRepo.saveChunks(docResult.document.id, docResult.chunks);

      return {
        documentId: docResult.document.id,
        chunksCount: docResult.chunks.length,
        stats: docResult.stats,
      };
    } catch (error: any) {
      throw new InternalServerErrorException(`Gagal meng-ingest teks manual: ${error.message}`);
    }
  }

  /**
   * 3. Ingesti Data Tabel Database
   */
  async ingestDatabaseRecords(dto: IngestDatabaseDto) {
    try {
      const formattedText = formatDataRecordsForIngestion(dto.records, {
        recordTitlePrefix: `[Record Tabel ${dto.tableName}]`,
        excludeKeys: ["password", "secret", "auth_token", "credit_card"],
      });

      const docResult = await createDocument({
        source: formattedText,
        filename: `db_table_${dto.tableName}.txt`,
        embedder: async (texts) => {
          const embRes = await this.aiConfig.client.embedding.create({ input: texts });
          return "embeddings" in embRes ? embRes.embeddings : [embRes.embedding];
        },
      });

      await this.knowledgeRepo.saveChunks(docResult.document.id, docResult.chunks);

      return {
        tableName: dto.tableName,
        recordsProcessed: dto.records.length,
        chunksGenerated: docResult.chunks.length,
      };
    } catch (error: any) {
      throw new InternalServerErrorException(`Gagal meng-ingest tabel database: ${error.message}`);
    }
  }

  /**
   * 4. Multimodal Vision Chat & RAG
   */
  async generateChatAnswer(dto: ChatQueryDto) {
    try {
      const embRes = await this.aiConfig.client.embedding.create({ input: dto.question });
      const questionVector = "embedding" in embRes ? embRes.embedding : embRes.embeddings[0]!;

      const candidateChunks = await this.knowledgeRepo.searchSimilarChunks(questionVector, 10);

      const queryResult = createQuery({
        vector: questionVector,
        keyword: dto.question,
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

      const chatOutput = await this.aiConfig.client.chat.generate({
        question: dto.question,
        contexts: queryResult.rankedChunks,
        images: dto.imageBase64 ? [{ dataUrl: dto.imageBase64 }] : undefined,
      });

      return {
        answer: chatOutput.text,
        imageAnalysis: chatOutput.imageAnalysis,
        citations: chatOutput.citations,
        usage: chatOutput.usage,
        raw: chatOutput.raw,
      };
    } catch (error: any) {
      throw new InternalServerErrorException(`Gagal memproses chat AI: ${error.message}`);
    }
  }

  /**
   * 5. Streaming Chat SSE
   */
  async streamChatAnswer(question: string, onChunk: (text: string) => void) {
    const embRes = await this.aiConfig.client.embedding.create({ input: question });
    const questionVector = "embedding" in embRes ? embRes.embedding : embRes.embeddings[0]!;

    const candidateChunks = await this.knowledgeRepo.searchSimilarChunks(questionVector, 5);

    return await this.aiConfig.client.chat.stream({
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
   * 6. Text-to-Speech
   */
  async textToSpeech(dto: TextToSpeechDto) {
    return await this.aiConfig.client.audio.speak({
      text: dto.text,
      voice: dto.voice ?? "nova",
    });
  }

  /**
   * 7. Speech-to-Text
   */
  async speechToText(dto: SpeechToTextDto) {
    return await this.aiConfig.client.audio.transcribe({
      file: dto.audioBase64,
      language: dto.language ?? "id",
    });
  }
}
```

---

## 5. 🎮 Layer Controller (`src/ai/ai.controller.ts`)

```typescript
// src/ai/ai.controller.ts
import { 
  Controller, 
  Post, 
  Body, 
  UseInterceptors, 
  UploadedFile, 
  BadRequestException,
  Res,
  HttpStatus 
} from "@nestjs/common";
import { FileInterceptor } from "@nestjs/platform-express";
import { Response } from "express";
import { AIService } from "./ai.service.js";
import { IngestManualDto } from "./dto/ingest-manual.dto.js";
import { IngestDatabaseDto } from "./dto/ingest-database.dto.js";
import { ChatQueryDto } from "./dto/chat-query.dto.js";
import { TextToSpeechDto } from "./dto/text-to-speech.dto.js";
import { SpeechToTextDto } from "./dto/speech-to-text.dto.js";

@Controller("ai")
export class AIController {
  constructor(private readonly aiService: AIService) {}

  @Post("ingest-file")
  @UseInterceptors(FileInterceptor("file"))
  async ingestFile(@UploadedFile() file: Express.Multer.File) {
    if (!file) {
      throw new BadRequestException("File PDF/Teks wajib diunggah!");
    }
    const data = await this.aiService.ingestUploadedFile(file.buffer, file.originalname);
    return { success: true, data };
  }

  @Post("ingest-manual")
  async ingestManual(@Body() dto: IngestManualDto) {
    const data = await this.aiService.ingestManualKnowledgeText(dto);
    return { success: true, data };
  }

  @Post("ingest-db")
  async ingestDatabase(@Body() dto: IngestDatabaseDto) {
    const data = await this.aiService.ingestDatabaseRecords(dto);
    return { success: true, data };
  }

  @Post("chat")
  async chat(@Body() dto: ChatQueryDto) {
    const data = await this.aiService.generateChatAnswer(dto);
    return { success: true, data };
  }

  @Post("chat-stream")
  async chatStream(@Body() dto: ChatQueryDto, @Res() res: Response) {
    res.setHeader("Content-Type", "text/event-stream");
    res.setHeader("Cache-Control", "no-cache");
    res.setHeader("Connection", "keep-alive");

    await this.aiService.streamChatAnswer(dto.question, (chunkText) => {
      res.write(`data: ${JSON.stringify({ text: chunkText })}\n\n`);
    });

    res.write(`data: [DONE]\n\n`);
    res.end();
  }

  @Post("audio/speak")
  async speak(@Body() dto: TextToSpeechDto, @Res() res: Response) {
    const audioResult = await this.aiService.textToSpeech(dto);
    res.setHeader("Content-Type", audioResult.mimeType);
    res.status(HttpStatus.OK).send(audioResult.audio);
  }

  @Post("audio/transcribe")
  async transcribe(@Body() dto: SpeechToTextDto) {
    const data = await this.aiService.speechToText(dto);
    return { success: true, data };
  }
}
```
