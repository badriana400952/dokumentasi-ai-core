# 🎓 Panduan Tutorial Lengkap `@badriana/ai-core`: Framework Universal RAG & Multimodal AI untuk Node.js (Express.js, NestJS, Fastify, & Next.js)

Selamat datang! Paket `@badriana/ai-core` adalah **Universal AI Engineering Framework** yang mendukung **seluruh ekosistem Node.js (Express.js, NestJS, Fastify, Hono, Bun, Deno, maupun Next.js)**. Panduan ini disusun secara **detail baris demi baris (code-by-code)** untuk pengembang pemula hingga profesional agar bisa membangun aplikasi AI berbasis **RAG (Retrieval-Augmented Generation)** lengkap dengan **Progress Bar Ingesti (0% - 100%)**, **Multimodal Vision Chat**, **Memori Percakapan Multi-Turn**, dan **Sitasi Dokumen**.

---

## 📌 Daftar Isi Tutorial

1. [Bab 1: Dukungan Arsitektur Universal Backend Node.js (Express.js, NestJS, Fastify, Hono, Next.js)](#bab-1-dukungan-arsitektur-universal-backend-nodejs)
2. [Bab 2: Prasyarat & Instalasi Paket `@badriana/ai-core`](#bab-2-prasyarat--instalasi-paket-badrianaai-core)
3. [Bab 3: Konfigurasi Client (`createAI`) & `.env`](#bab-3-konfigurasi-client-createai--env)
4. [Bab 4: Katalog Lengkap API & Struktur Output Response (`@badriana/ai-core`)](#bab-4-katalog-lengkap-api--struktur-output-response-badrianaai-core)
5. [Bab 5: Konsep Ingesti Dokumen, Vision OCR & Penyaringan Gambar Bebas Token (0 Token)](#bab-5-konsep-ingesti-dokumen-vision-ocr--penyaringan-gambar-bebas-token-0-token)
6. [Bab 6: 5 Jenis Gambar yang Tidak Bisa atau Dibuang dari Ekstraksi Paket](#bab-6-5-jenis-gambar-yang-tidak-bisa-atau-dibuang-dari-ekstraksi-paket)
7. [Bab 7: Panduan Lengkap Ingesti Data Database Perusahaan (SQL / 40+ Tabel) ke AI Chat RAG](#bab-7-panduan-lengkap-ingesti-data-database-perusahaan-sql--40-tabel-ke-ai-chat-rag)
8. [Bab 8: Multimodal Vision Chat & Analisis Niat Gambar Otomatis (`imageAnalysis`)](#bab-8-multimodal-vision-chat--analisis-niat-gambar-otomatis-imageanalysis)
9. [Bab 9: Konsep Arsitektur Chat (Memori Multi-Turn, Sitasi Otomatis & Model Fallback Chain)](#bab-9-konsep-arsitektur-chat-memori-multi-turn-sitasi-otomatis--model-fallback-chain)
10. [Bab 10: Tutorial Step-by-Step Aplikasi Full-Stack (Next.js + React UI + Progress Bar)](#bab-10-tutorial-step-by-step-aplikasi-full-stack)
11. [Bab 11: FAQ Khusus Paket & Detail API (`@badriana/ai-core`)](#bab-11-faq-khusus-paket--detail-api-badrianaai-core)
12. [Bab 12: Troubleshooting & Solusi Error Umum](#bab-12-troubleshooting--solusi-error-umum)
13. [Bab 13: Panduan Tingkat Lanjut Lintas Skenario Produksi (Production Readiness & Advanced Patterns)](#bab-13-panduan-tingkat-lanjut-lintas-skenario-produksi-production-readiness--advanced-patterns)

---

## 🌐 Bab 1: Dukungan Arsitektur Universal Backend Node.js

Paket `@badriana/ai-core` dibangun murni berstandar **TypeScript / JavaScript (ESM & CommonJS)** tanpa ketergantungan pada React atau Next.js!

Paket ini **100% SUPPORTS** dan dapat dijalankan di **seluruh ekosistem runtime & framework Node.js**:
- **Express.js** (Backend REST API) — *Lihat contoh arsitektur terstruktur lengkap (Router-Controller-Service-Repository) di file [`express.example.md`](file:///D:/badri/npm/ai-core/express.example.md).*
- **NestJS** (Enterprise Modular Framework) — *Lihat contoh arsitektur modular lengkap (Controller-Service-Repository) di file [`nestjs.example.md`](file:///D:/badri/npm/ai-core/nestjs.example.md).*
- **Next.js** (App Router & Server Actions / Route Handlers) — *Lihat contoh arsitektur terpisah (Route Handler-Service-Repository) di file [`nextjs.example.md`](file:///D:/badri/npm/ai-core/nextjs.example.md).*
- **Fastify** (High-Performance Plugin Backend API) — *Lihat contoh arsitektur plugin lengkap (Plugin-Routes-Controller-Service-Repository) di file [`fastify.example.md`](file:///D:/badri/npm/ai-core/fastify.example.md).*
- **Koa.js** (Async/Await Middleware Backend API) — *Lihat contoh arsitektur middleware lengkap (Router-Controller-Service-Repository) di file [`koa.example.md`](file:///D:/badri/npm/ai-core/koa.example.md).*
- **Hono / AdonisJS / Bun / Deno CLI / Microservices**

---

### 💻 1. Contoh Integrasi di Express.js (`server.ts`):

```typescript
// server.ts - Express.js REST API Server
import express, { Request, Response } from "express";
import { createAI, createDocument } from "@badriana/ai-core";

const app = express();
app.use(express.json());

// Inisialisasi AI Core Client
const ai = createAI({
  provider: "openrouter",
  apiKey: process.env.OPENROUTER_API_KEY,
  chatModel: "google/gemma-2-9b-it:free",
});

// 📌 ENDPOINT 1: POST /api/ingest (Ingesti Dokumen Teks/PDF)
app.post("/api/ingest", async (req: Request, res: Response) => {
  try {
    const { text, filename } = req.body;
    
    const docResult = await createDocument({
      source: text,
      filename: filename || "document.txt",
      embedder: async (texts) => {
        const embRes = await ai.embedding.create({ input: texts });
        return "embeddings" in embRes ? embRes.embeddings : [embRes.embedding];
      },
    });

    res.json({
      success: true,
      chunksCount: docResult.chunks.length,
      stats: docResult.stats,
    });
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
});

// 📌 ENDPOINT 2: POST /api/chat (Tanya Jawab AI Chat RAG)
app.post("/api/chat", async (req: Request, res: Response) => {
  try {
    const { question, contexts } = req.body;
    
    const answer = await ai.chat.generate({
      question,
      contexts,
    });

    res.json({
      answer: answer.text,
      model: answer.model,
      usage: answer.usage,
      raw: answer.raw, // Raw JSON mentah dari OpenRouter
    });
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
});

app.listen(3000, () => console.log("🚀 Express.js AI Server running on port 3000"));
```

---

### 💻 2. Contoh Integrasi di NestJS (`ai.service.ts` & `ai.controller.ts`):

#### A. Service NestJS (`ai.service.ts`):
```typescript
import { Injectable } from "@nestjs/common";
import { createAI, AI } from "@badriana/ai-core";

@Injectable()
export class AiService {
  private readonly ai: AI;

  constructor() {
    this.ai = createAI({
      provider: "openrouter",
      apiKey: process.env.OPENROUTER_API_KEY,
      chatModel: "google/gemma-2-9b-it:free",
    });
  }

  async generateChatAnswer(question: string, contexts: any[]) {
    return await this.ai.chat.generate({
      question,
      contexts,
    });
  }
}
```

#### B. Controller NestJS (`ai.controller.ts`):
```typescript
import { Controller, Post, Body } from "@nestjs/common";
import { AiService } from "./ai.service";

@Controller("ai")
export class AiController {
  constructor(private readonly aiService: AiService) {}

  @Post("chat")
  async chat(@Body() dto: { question: string; contexts: any[] }) {
    const result = await this.aiService.generateChatAnswer(dto.question, dto.contexts);
    return {
      answer: result.text,
      model: result.model,
      usage: result.usage,
      raw: result.raw, // 🔴 100% Raw Response dari Provider!
    };
  }
}
```

---

## 🛠️ Bab 2: Prasyarat & Instalasi Paket `@badriana/ai-core`

Sebelum memulai, pastikan perangkat komputer Anda sudah terpasang:
- **Node.js**: Versi `18.0.0` atau yang lebih baru (Ketik `node -v` di terminal untuk mengecek).
- **NPM**: Versi `9.0.0` atau yang lebih baru.
- **API Key OpenRouter / OpenAI**: Kunci API untuk memanggil model AI (Gratis dari [openrouter.ai](https://openrouter.ai/keys)).

Jalankan perintah berikut di terminal untuk memasang paket `@badriana/ai-core` dan ikon `lucide-react`:

```bash
npm install @badriana/ai-core lucide-react
```

---

## ⚙️ Bab 3: Konfigurasi Client (`createAI`) & `.env`

Buat file `.env` atau `.env.local` di folder akar (*root*) proyek Anda:

```env
# File: .env.local
OPENROUTER_API_KEY=sk-or-v1-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

> 💡 **Catatan**: Jika menggunakan OpenAI langsung, Anda bisa mengganti nama variabel menjadi `OPENAI_API_KEY=sk-proj-xxxx`.

### Inisialisasi Client Terpusat (`src/lib/ai.ts`):

```typescript
// File: src/lib/ai.ts
import { createAI } from "@badriana/ai-core";

/**
 * Mengembalikan instance terpusat dari @badriana/ai-core.
 * Menggunakan OpenRouter sebagai provider default dengan model gratis.
 */
export function getAiClient() {
  const apiKey = process.env.OPENROUTER_API_KEY;

  if (!apiKey) {
    throw new Error(
      "❌ OPENROUTER_API_KEY belum dikonfigurasi di file environment!"
    );
  }

  return createAI({
    provider: "openrouter",
    apiKey: apiKey,
    chatModel: "google/gemma-2-9b-it:free",              // Model AI Chat default
    embeddingModel: "openai/text-embedding-3-small",     // Model Vektor Embedding default
  });
}
```

---

## 📖 Bab 4: Katalog Lengkap API & Struktur Output Response (`@badriana/ai-core`)

Berikut adalah daftar lengkap seluruh fungsi paket `@badriana/ai-core` yang menghasilkan response, disertai contoh penggunaan TypeScript, komentar penjelasan per baris, dan struktur JSON output mentah yang dihasilkan:

---

### 1️⃣ `ai.chat.generate(input)` — Generasi Jawaban AI Terstruktur

Mengeksekusi Chat Completion lengkap dengan konteks RAG, memori obrolan, dan pembentukan sitasi rujukan.

```typescript
import { createAI } from "@badriana/ai-core";

const ai = createAI({
  provider: "openrouter",
  apiKey: process.env.OPENROUTER_API_KEY,
  chatModel: "google/gemma-2-9b-it:free",
});

// Panggil fungsi ai.chat.generate()
const result = await ai.chat.generate({
  question: "Jelaskan Hukum Newton 1!",  // Pertanyaan dari siswa
  system: "Kamu adalah Guru Fisika.",    // System prompt kustom
  contexts: [                            // Context RAG dari Vector DB (Opsional)
    {
      id: "chunk_101",
      content: "Hukum I Newton menyatakan bahwa setiap benda akan diam atau bergerak lurus beraturan jika tidak ada gaya luar yang bekerja.",
      source: { filename: "Modul_Fisika.pdf", page: 5 },
    },
  ],
  temperature: 0.7,                      // Kreativitas jawaban (0.0 - 1.0)
});

console.log(result);
```

#### 📋 Struktur Output JSON (`result`):
```json
{
  "text": "Hukum I Newton (sering disebut Hukum Kelembaman/Inersia) menyatakan bahwa...",
  "model": "google/gemma-2-9b-it:free",
  "usage": {
    "inputTokens": 68,
    "outputTokens": 142,
    "totalTokens": 210
  },
  "citations": [
    {
      "chunkId": "chunk_101",
      "source": "Modul_Fisika.pdf",
      "page": 5,
      "title": "Modul_Fisika.pdf"
    }
  ],
  "finishReason": "stop",
  "raw": {
    "id": "gen-1786096500-a3f1b2c",
    "provider": "Google",
    "model": "google/gemma-2-9b-it:free",
    "object": "chat.completion",
    "created": 1786096500,
    "choices": [
      {
        "logprobs": null,
        "finish_reason": "stop",
        "native_finish_reason": "STOP",
        "index": 0,
        "message": {
          "role": "assistant",
          "content": "Hukum I Newton (sering disebut Hukum Kelembaman/Inersia)..."
        }
      }
    ],
    "usage": {
      "prompt_tokens": 68,
      "completion_tokens": 142,
      "total_tokens": 210
    }
  }
}
```

---

### 2️⃣ `ai.chat.stream(input)` — Generasi Jawaban AI Streaming (Typewriter Effect)

Mengeksekusi Chat Completion berformat *Streaming (AsyncIterable)* untuk efek penulisan teks otomatis kata-demi-kata di UI.

```typescript
const stream = await ai.chat.stream({
  question: "Jelaskan reaksi fotosintesis!",
  contexts: matchingChunks,
});

for await (const chunk of stream) {
  process.stdout.write(chunk.text);
}
```

#### 📋 Struktur Output Tiap Chunk (`chunk`):
```json
{
  "text": "Fotosintesis ",
  "model": "google/gemma-2-9b-it:free",
  "raw": {
    "id": "gen-1786096501-x9z",
    "choices": [
      {
        "delta": { "content": "Fotosintesis " },
        "finish_reason": null
      }
    ]
  }
}
```

---

### 3️⃣ `ai.embedding.create(input)` — Generasi Vektor Embedding Batch

Mengubah array string teks menjadi array vektor angka float 1536-dimensi untuk pencarian semantik RAG.

```typescript
const embResult = await ai.embedding.create({
  input: ["Gravitasi adalah gaya tarik-menarik", "Fotosintesis membutuhkan sinar matahari"],
});

console.log(embResult);
```

#### 📋 Struktur Output JSON (`embResult`):
```json
{
  "embeddings": [
    [0.0124, -0.0451, 0.0892, 0.0012],
    [-0.0311, 0.0192, -0.0712, 0.0441]
  ],
  "model": "openai/text-embedding-3-small",
  "usage": {
    "promptTokens": 14,
    "totalTokens": 14
  },
  "raw": {
    "object": "list",
    "data": [
      { "object": "embedding", "index": 0, "embedding": [0.0124, -0.0451] },
      { "object": "embedding", "index": 1, "embedding": [-0.0311, 0.0192] }
    ],
    "model": "text-embedding-3-small",
    "usage": { "prompt_tokens": 14, "total_tokens": 14 }
  }
}
```

---

### 4️⃣ `createDocument(options)` — Ingesti Dokumen Lengkap (Parsing, Chunking, Embedding)

Menjalankan pipeline ingesti dokumen PDF/TXT/MD, memicu filter gambar 0 token, memecah chunks, dan menghitung statistik ingesti.

```typescript
import { createDocument } from "@badriana/ai-core";

const docResult = await createDocument({
  source: pdfBuffer,
  filename: "Modul_Fisika.pdf",
  embedder: async (texts) => {
    const res = await ai.embedding.create({ input: texts });
    return "embeddings" in res ? res.embeddings : [res.embedding];
  },
  filterCoverImages: true,
});

console.log(docResult);
```

#### 📋 Struktur Output JSON (`docResult`):
```json
{
  "document": {
    "id": "doc_9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
    "content": "Isi dokumen murni yang sudah dibersihkan dari unicode terlarang...",
    "chunks": [],
    "metadata": {
      "title": "Modul_Fisika.pdf",
      "source": "Modul_Fisika.pdf",
      "mimeType": "application/pdf",
      "size": 1048576,
      "createdAt": "2026-08-07T17:30:00.000Z"
    }
  },
  "chunks": [
    {
      "id": "chunk_f47ac10b",
      "type": "text",
      "index": 0,
      "startOffset": 0,
      "endOffset": 350,
      "content": "Paragraf materi Hukum Newton 1...",
      "source": { "filename": "Modul_Fisika.pdf", "page": 1 },
      "embedding": [0.0124, -0.0451, 0.0892]
    },
    {
      "id": "chunk_e89ac10b",
      "type": "image",
      "kind": "diagram",
      "index": 1,
      "startOffset": 0,
      "endOffset": 0,
      "content": "Diagram gaya F1 dan F2...",
      "source": { "filename": "Modul_Fisika.pdf", "page": 3 },
      "image": {
        "dataUrl": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA...",
        "mimeType": "image/png",
        "page": 3,
        "width": 640,
        "height": 480
      },
      "embedding": [0.0341, -0.1120, 0.7451]
    }
  ],
  "images": [
    {
      "dataUrl": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA...",
      "mimeType": "image/png",
      "page": 3
    }
  ],
  "stats": {
    "characters": 52821,
    "words": 9312,
    "images": 1,
    "chunks": 2,
    "pages": 12,
    "parseTimeMs": 210,
    "chunkTimeMs": 15,
    "embeddingTimeMs": 532,
    "totalTimeMs": 757
  }
}
```

---

### 5️⃣ `createQuery(options)` / `ai.query.search(input)` — Pencarian Vektor Hibrida (Hybrid RRF Search)

Melakukan pencarian semantik gabungan Vektor Cosine Similarity + Keyword Match menggunakan algoritma RRF (Reciprocal Rank Fusion).

```typescript
import { createQuery } from "@badriana/ai-core";

const queryResult = await createQuery({
  vector: questionEmbeddingVector,
  keyword: "gaya gravitasi",
  chunks: storedKnowledgeChunks,
  limit: 3,
  rrfK: 60,
});

console.log(queryResult);
```

#### 📋 Struktur Output JSON (`queryResult`):
```json
{
  "rankedChunks": [
    {
      "chunk": {
        "id": "chunk_f47ac10b",
        "type": "text",
        "content": "Gaya gravitasi adalah gaya tarik-menarik antar benda...",
        "source": { "filename": "Modul_Fisika.pdf", "page": 4 }
      },
      "score": 0.03278,
      "rank": 1
    }
  ],
  "topChunk": {
    "id": "chunk_f47ac10b",
    "type": "text",
    "content": "Gaya gravitasi adalah gaya tarik-menarik antar benda..."
  },
  "stats": {
    "totalChunksEvaluated": 150,
    "queryTimeMs": 8
  }
}
```

---

### 6️⃣ `ai.context.build(input)` — Pembangun Konteks Window RAG Aman Token

Menghitung jumlah token dan memangkas potongan RAG secara presisi agar tidak melebihi batas token model AI.

```typescript
const contextResult = ai.context.build({
  chunks: rawMatchingChunks,
  maxTokens: 3000,
});

console.log(contextResult);
```

#### 📋 Struktur Output JSON (`contextResult`):
```json
{
  "text": "[1] Gaya gravitasi adalah gaya tarik-menarik...\n\n[2] Rumus gaya gravitasi F = G * (m1 * m2) / r^2",
  "tokenCount": 2450,
  "includedChunks": [
    { "id": "chunk_101", "type": "text", "content": "..." }
  ],
  "truncated": false
}
```

---

### 7️⃣ `parse(content, filename)` — Ekstraktor Teks & Gambar Mentah Dokumen

Fungsi tingkat rendah (*low-level*) untuk mengekstrak teks murni dan gambar biner dari file PDF/TXT tanpa membuat embedding.

```typescript
import { parse } from "@badriana/ai-core";

const parsedDoc = await parse(pdfBuffer, "Modul_IPS.pdf");
console.log(parsedDoc);
```

#### 📋 Struktur Output JSON (`parsedDoc`):
```json
{
  "content": "BAB 1: Pengenalan Ilmu Pengetahuan Sosial\n1.1 Pengertian Geografi...",
  "images": [
    {
      "dataUrl": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA...",
      "mimeType": "image/png",
      "page": 1
    }
  ],
  "metadata": {
    "title": "Modul_IPS.pdf",
    "source": "Modul_IPS.pdf",
    "mimeType": "application/pdf",
    "size": 524288,
    "createdAt": "2026-08-07T17:30:00.000Z"
  }
}
```

---

### 8️⃣ `chunk(text, options)` — Pemecah Teks Hierarkis (*Text Splitter*)

Fungsi tingkat rendah untuk memecah string teks panjang menjadi potongan *TextChunks*, *TableChunks*, atau *EquationChunks*.

```typescript
import { chunk } from "@badriana/ai-core";

const chunks = chunk(rawTextContent, {
  size: 1000,
  overlap: 150,
});

console.log(chunks);
```

#### 📋 Struktur Output JSON (`chunks`):
```json
[
  {
    "id": "chunk_a1b2c3d4",
    "type": "text",
    "index": 0,
    "startOffset": 0,
    "endOffset": 980,
    "content": "Paragraf pertama materi pelajaran..."
  },
  {
    "id": "chunk_e5f6g7h8",
    "type": "text",
    "index": 1,
    "startOffset": 830,
    "endOffset": 1810,
    "content": "Paragraf kedua materi pelajaran yang memiliki overlapping 150 karakter..."
  }
]
```

---

### 9️⃣ `ai.audio.speak(input)` — Generasi Suara AI (Text-to-Speech / TTS)

Mengonversi penjelasan teks dari AI/Guru menjadi berkas suara audio biner MP3/Buffer untuk didengarkan pengguna di aplikasi web atau seluler.

#### 📦 Format Payload Input JSON ke `ai.audio.speak()`:
```json
{
  "text": "Selamat datang di kelas Fisika! Hari ini kita akan mempelajari Hukum Newton 1.",
  "voice": "alloy",
  "model": "tts-1"
}
```

*Opsi Karakter Suara (`voice`)*: `alloy`, `echo`, `fable`, `onyx`, `nova`, `shimmer`.

#### 📋 Format Payload Output JSON dari `ai.audio.speak()`:
```json
{
  "mimeType": "audio/mpeg",
  "model": "tts-1",
  
  // 🔊 BERKAS SUARA MP3 DALAM BENTUK NODE.JS BUFFER:
  "audio": {
    "type": "Buffer",
    "data": [73, 68, 51, 3, 0, 0, 0, 0, 0, 35, 84, 83, 83, 69, ...]
  },
  
  "raw": "<ArrayBuffer 142050 bytes>"
}
```

#### 💻 Contoh Penggunaan di TypeScript (API Route / Express Server):
```typescript
import { createAI } from "@badriana/ai-core";

const ai = createAI({
  provider: "openrouter",
  apiKey: process.env.OPENROUTER_API_KEY,
});

// Panggil ai.audio.speak() untuk mengubah teks jawaban AI menjadi Suara MP3
const audioResult = await ai.audio.speak({
  text: "Hukum I Newton menyatakan bahwa setiap benda akan tetap diam atau bergerak lurus beraturan jika tidak ada gaya luar yang bekerja.",
  voice: "nova", // Suara ramah wanita
});

// Kirimkan berkas audio MP3 langsung ke browser/frontend
res.setHeader("Content-Type", audioResult.mimeType);
res.send(audioResult.audio);
```

---

### 🔟 `ai.audio.transcribe(input)` — Transkripsi Suara menjadi Teks (Speech-to-Text / STT)

Mengonversi rekaman suara pengguna (Audio Buffer / Base64 Data URI) menjadi string teks secara otomatis menggunakan Whisper AI.

#### 📦 Format Payload Input JSON ke `ai.audio.transcribe()`:
```json
{
  "file": "data:audio/mp3;base64,SUQzBAAAAAAAI1RTU0UAAAA...",
  "language": "id",
  "model": "whisper-1"
}
```

#### 📋 Format Payload Output JSON dari `ai.audio.transcribe()`:
```json
{
  // 📝 HASIL TRANSKRIPSI TEKS DARI REKAMAN SUARA:
  "text": "Bantu saya menjelaskan rumus gaya gravitasi bumi",
  "model": "whisper-1",
  "raw": {
    "text": "Bantu saya menjelaskan rumus gaya gravitasi bumi"
  }
}
```

#### 💻 Contoh Penggunaan di TypeScript (API Endpoint Transkripsi):
```typescript
import { createAI } from "@badriana/ai-core";

const ai = createAI({
  provider: "openrouter",
  apiKey: process.env.OPENROUTER_API_KEY,
});

// Panggil ai.audio.transcribe() dengan melewatkan rekaman suara mikrofon
const transcriptResult = await ai.audio.transcribe({
  file: req.body.audioBase64, // Base64 Data URI dari mikrofon user
  language: "id",             // Kode Bahasa Indonesia
});

console.log(transcriptResult.text); 
// Output: "Bantu saya menjelaskan rumus gaya gravitasi bumi"
```

---

## 🖼️ Bab 5: Konsep Ingesti Dokumen, Vision OCR & Penyaringan Gambar Bebas Token (0 Token)

### A. Bagaimana AI (ChatGPT / OpenRouter / `@badriana/ai-core`) "Melihat" Foto?
Saat pengguna mengunggah gambar, AI tidak memiliki mata biner. Proses pemrosesan gambar mengikuti 3 tahap:
1. **Visual Tokenization**: Gambar dipotong menjadi matriks piksel kecil (misal $16 \times 16$ px) dan diubah menjadi token angka visual (*Visual Embeddings*).
2. **Vision OCR & Feature Extraction**: *Vision Encoder* membaca garis, bentuk, warna, dan teks di dalam gambar.
3. **Multimodal Alignment**: LLM menyatukan token gambar dan token teks pertanyaan ke dalam satu ruang vektor (*Attention Mechanism*), sehingga AI memahami hubungan antara pertanyaan pengguna dan isi gambar.

---

### B. Fitur Penyaringan Gambar Bebas Token (0 Token / Gratis)
Di dalam `@badriana/ai-core`, saat ingesti dokumen PDF/Word berlangsung, paket menggunakan **Algoritma Fisik Heuristik** untuk menyaring foto non-materi **tanpa memotong token API sedikit pun**:

1. **Page 1 Cover Filter**: Otomatis mengabaikan gambar di Halaman 1 (Foto sampul bab/modul).
2. **Min Dimension Filter**: Mengabaikan logo/ikon kecil dengan ukuran `< 120px`.
3. **Aspect Ratio Filter**: Mengabaikan garis hiasan header/footer dengan rasio lebar/tinggi ekstrem (`> 4.5`).
4. **Vision Kind Filter**: Jika menggunakan `imageProcessor`, gambar berjenis `cover`, `logo`, dan `decorative` otomatis diabaikan dari database vektor RAG.

```typescript
// Contoh Kustomisasi Filter Gambar di createDocument()
const docResult = await createDocument({
  source: pdfBuffer,
  embedder,
  filterCoverImages: true,      // Mengaktifkan filter sampul Halaman 1 (Default: true, 0 Token Cost)
  minImageDimension: 150,       // Abaikan logo/ikon di bawah 150px
  maxImageAspectRatio: 4.0,     // Abaikan banner pipih
});
```

---

## 🚫 Bab 6: 5 Jenis Gambar yang Tidak Bisa atau Dibuang dari Ekstraksi Paket

Tidak semua gambar dalam dokumen PDF/Word cocok untuk disimpan ke database RAG. Paket `@badriana/ai-core` menangani dan membatasi 5 jenis gambar berikut agar database pengetahuan RAG tetap bersih dan hemat token:

---

### 1️⃣ Gambar Resolusi Sangat Rendah atau Blur Parah (`< 120px` / Pikselasi)
- **Bentuk Gambar**: Gambar ikon tombol, thumbnail pikselasi buram, atau tulisan tangan yang terdistorsi parah.
- **Alasan Tidak Bisa**: Vision AI & OCR membutuhkan kontras dan resolusi piksel minimal untuk membaca karakter teks/objek.
- **Penanganan Paket**: Paket `@badriana/ai-core` memiliki filter `minImageDimension: 120` yang **otomatis membuang thumbnail/ikon kecil (0 Token API Cost)**.

---

### 2️⃣ Gambar Dekoratif Murni (Garis Pembatas, Header/Footer, Watermark)
- **Bentuk Gambar**: Gambar garis hiasan pembatas bab, watermark transparan berulang, atau banner header/footer yang sangat pipih.
- **Alasan Dibuang**: Gambar ini tidak membawa makna pengetahuan (*knowledge*). Jika disimpan, gambar hanya mengotori database RAG.
- **Penanganan Paket**: Paket `@badriana/ai-core` menggunakan filter `maxImageAspectRatio: 4.5` yang **otomatis membuang banner/garis hiasan pipih (0 Token API Cost)**.

---

### 3️⃣ Foto Sampul Bab / Cover Halaman Utama PDF (Page 1 Cover)
- **Bentuk Gambar**: Foto hiasan di Halaman 1 PDF, logo kementerian, atau bingkai judul bab.
- **Alasan Dibuang**: Foto sampul bab bukan merupakan materi pelajaran yang dicari siswa saat mengajukan pertanyaan di chat.
- **Penanganan Paket**: Paket `@badriana/ai-core` mengaktifkan `filterCoverImages: true` yang **otomatis mengabaikan foto sampul Halaman 1 (0 Token API Cost)**.

---

### 4️⃣ Gambar Terenkripsi DRM / Kompresi PDF Non-Standar (`JBIG2` / `JPXDecode`)
- **Bentuk Gambar**: File PDF yang dilindungi kunci proteksi DRM (*Digital Rights Management*) atau stream gambar terkompresi non-standar.
- **Alasan Tidak Bisa**: Stream biner gambar tidak dapat di-decode menjadi format standar (PNG/JPEG) oleh decoder Node.js.
- **Penanganan Paket**: Paket menangkap error decode secara aman dan memancarkan log peringatan tanpa menghentikan pemrosesan teks dokumen.

---

### 5️⃣ Foto yang Melanggar Kebijakan Keamanan Provider AI (*Safety Filter Triggered*)
- **Bentuk Gambar**: Foto yang memuat konten kekerasan ekstrem, konten sensitif terlarang, atau dokumen rahasia terenkripsi.
- **Alasan Tidak Bisa**: Provider Vision AI (seperti OpenRouter, OpenAI, Gemini) secara otomatis **menolak memproses gambar** (*Safety Filter Triggered*).
- **Penanganan Paket**: Paket menangkap penolakan provider secara aman dan mengembalikan pesan error yang ramah pengembang.

---

## 🗄️ Bab 7: Panduan Lengkap Ingesti Data Database Perusahaan (SQL / 40+ Tabel) ke AI Chat RAG

Dalam dunia industri nyata, perusahaan besar memiliki **40+ tabel database (PostgreSQL, MySQL, Supabase, MongoDB)** dan ingin agar AI dapat menjawab pertanyaan seputar data tersebut **tanpa perlu mengunggah file PDF fisik**.

Paket `@badriana/ai-core` memfasilitasi kebutuhan ini secara elegan:
1. **Bebas Database (*Database Agnostic*)**: Paket tidak menanam database khusus, sehingga kompatibel 100% dengan database apa pun milik perusahaan.
2. **Redaksi Kolom Sensitif Otomatis (`formatDataRecordsForIngestion`)**: Secara otomatis menghapus kolom rahasia (seperti `password_hash`, `auth_token`, `secret_key`) dari vektor pengetahuan AI.

---

### 💻 Contoh Penggunaan Kode Lengkap (Service Terintegrasi):

```typescript
// File: src/services/companyAiService.ts
import { 
  createAI, 
  createDocument, 
  formatDataRecordsForIngestion 
} from "@badriana/ai-core";
import { prisma } from "@/lib/prisma"; // ORM Database Perusahaan (PostgreSQL/MySQL)

// 1. Inisialisasi Client Terpusat AI Core (Provider: OpenRouter)
const ai = createAI({
  provider: "openrouter",
  apiKey: process.env.OPENROUTER_API_KEY,
  chatModel: "google/gemma-2-9b-it:free",              // Model Chat AI
  embeddingModel: "openai/text-embedding-3-small",     // Model Vektor Embedding
});

/**
 * =========================================================================
 * LOKASI 1: INGESTI DATA DATABASE PERUSAHAAN KE RAG VEKTOR
 * Mengambil data dari 3 tabel (Tabel Siswa, Tabel Mata Pelajaran, & Tabel Nilai)
 * =========================================================================
 */
export async function ingestCompanyDatabaseToRag() {
  console.log("🔄 1. Mengambil data dari Database Perusahaan (3 Tabel Joined)...");

  // A. Ambil data gabungan dari database perusahaan via Prisma/SQL
  const rawDbRecords = await prisma.studentScore.findMany({
    include: {
      student: true,   // Join Tabel Siswa
      subject: true,   // Join Tabel Mata Pelajaran
    },
  });

  // B. Gunakan helper formatDataRecordsForIngestion() dari @badriana/ai-core
  //    Helper ini OTOMATIS MEMBUANG kolom password/token sensitif dan merapikan teks!
  const formattedText = formatDataRecordsForIngestion(rawDbRecords, {
    titleKey: "name",                    // Gunakan nama sebagai judul record
    excludeKeys: ["internal_note", "ip"], // Kolom tambahan yang ingin diabaikan
  });

  console.log("⚡ 2. Mengolah data dengan paket @badriana/ai-core...");

  // C. Panggil createDocument() dari paket @badriana/ai-core
  const docResult = await createDocument({
    source: formattedText,
    filename: "Live_Database_Student_Scores.txt",
    
    // Embedder callback: Mengubah baris database menjadi Vektor 1536-Dimensi
    embedder: async (texts) => {
      const embRes = await ai.embedding.create({ input: texts });
      return "embeddings" in embRes ? embRes.embeddings : [embRes.embedding];
    },
    
    // Callback Progres Real-Time (0% -> 100%)
    onProgress: (p) => {
      console.log(`[${p.stage.toUpperCase()}] ${p.percentage}% - ${p.message}`);
    },
  });

  console.log("💾 3. Menyimpan Vektor Chunks kembali ke Database Perusahaan...");

  // D. Simpan hasil Chunks + Vektor Embedding 1536-dimensi ke Database Perusahaan
  await prisma.knowledgeChunk.createMany({
    data: docResult.chunks.map((chunk) => ({
      id: chunk.id,
      type: chunk.type,
      content: chunk.content,
      embedding: chunk.embedding, // Array float [0.0124, -0.0451, ...]
      metadata: {
        sourceDbTable: "student_scores",
        totalRecords: rawDbRecords.length,
      },
    })),
  });

  console.log(`✅ Sukses! ${docResult.chunks.length} Chunks Vektor Database tersimpan.`);
  return docResult.stats;
}

/**
 * =========================================================================
 * LOKASI 2: CHAT AI DENGAN DATA DATABASE PERUSAHAAN (OPENROUTER PROVIDER)
 * Siswa/Admin bertanya seputar data yang bersumber dari Database
 * =========================================================================
 */
export async function askAiAboutCompanyDatabase(userQuestion: string) {
  // A. Ambil vektor embedding dari pertanyaan user
  const embRes = await ai.embedding.create({ input: userQuestion });
  const questionVector = "embedding" in embRes ? embRes.embedding : embRes.embeddings[0];

  // B. Cari Chunks terdekat di Database (Cosine Similarity Vector Search)
  const matchingChunksFromDb = await prisma.knowledgeChunk.findMany({
    take: 5,
    select: { id: true, content: true },
  });

  // C. Tanya AI menggunakan ai.chat.generate() dari paket @badriana/ai-core
  const chatResponse = await ai.chat.generate({
    question: userQuestion,
    system: "Kamu adalah Asisten AI Data Sekolah yang ramah, jelas, dan akurat.",
    contexts: matchingChunksFromDb.map((c) => ({
      id: c.id,
      content: c.content,
    })),
  });

  return {
    answer: chatResponse.text,         // Jawaban AI berdasarkan data DB Perusahaan
    modelUsed: chatResponse.model,     // Model OpenRouter yang merespons
    usage: chatResponse.usage,         // Jumlah token API (inputTokens, outputTokens)
    rawResponse: chatResponse.raw,     // Raw JSON mentah 100% dari Provider OpenRouter
  };
}
```

---

## 📸 Bab 8: Multimodal Vision Chat & Analisis Niat Gambar Otomatis (`imageAnalysis`)

Paket `@badriana/ai-core` mendukung **Multimodal Chat Input**, yaitu mengajukan pertanyaan teks **beserta 1 atau lebih gambar** pendukung secara otomatis. Paket mengenali 8 kategori niat gambar (*ImageIntent*):

1. **`person_face` (Deteksi Wajah / Tokoh / Pahlawan / Penemu)**: Mengenali foto wajah manusia, ekspresi, maupun tokoh ilmuwan/sejarah (misalnya foto Albert Einstein, Soekarno, atau tokoh pahlawan).
2. **`object_detection` (Deteksi Objek / Alat Laboratorium / Spesimen Biologi)**: Mengenali foto alat-alat pelajaran (misalnya mikroskop, neraca, jangka sorong, sel biologi, hewan, tumbuhan, planet).
3. **`landmark_map` (Deteksi Peta Geografi / Situs Bersejarah / Candi)**: Mengenali denah lokasi, peta wilayah IPS/Geografi, atau bangunan bersejarah.
4. **`document_text` (Deteksi Dokumen Teks Resmi)**: Mengenali lembaran surat, sertifikat, atau dokumen berformat.
5. **`exam_question` (Deteksi Foto Lembar Soal Ujian)**: AI memberikan penjelasan langkah demi langkah (*step-by-step reasoning*).
6. **`whiteboard` (Deteksi Foto Coretan Papan Tulis / Tulisan Tangan)**: Vision OCR tulisan tangan dan perapihan catatan.
7. **`diagram` (Deteksi Diagram / Grafik / Tabel)**: Penjelasan variabel grafik dan bagan.
8. **`general_photo`**: Foto pemandangan / objek umum di dunia nyata.

---

### 💻 Contoh Penggunaan di TypeScript:

```typescript
import { createAI } from "@badriana/ai-core";

const ai = createAI({
  provider: "openrouter",
  apiKey: process.env.OPENROUTER_API_KEY,
  chatModel: "meta-llama/llama-3.2-11b-vision-instruct:free", // Model Vision AI
});

// Panggil ai.chat.generate() dengan menyertakan parameter images
const result = await ai.chat.generate({
  question: "Bantu aku menjawab soal nomor 3 di foto ini!",
  
  // 📸 Array Gambar yang Dikirimkan Pengguna di Chat (Base64 atau HTTP URL)
  images: [
    { dataUrl: "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA..." }
  ],
  
  contexts: matchingRagChunks, // Tetap dapat digabung dengan RAG Context!
});

console.log(result.text);          // Jawaban solusi AI
console.log(result.imageAnalysis); // Analisis Niat Gambar Otomatis dari Paket
```

#### 📋 Output Analisis Niat Gambar (`result.imageAnalysis`):
```json
{
  "text": "Berdasarkan foto soal nomor 3:\nDiketahui 2x + 5 = 15...\nLangkah 1: Kurangi kedua ruas dengan 5 -> 2x = 10\nLangkah 2: Bagi kedua ruas dengan 2 -> x = 5.\n\nJawaban akhir adalah x = 5.",

  // 🧠 HASIL ANALISIS NIAT GAMBAR OTOMATIS DARI PAKET:
  "imageAnalysis": [
    {
      "intent": "exam_question",
      "summary": "Foto lembar soal / latihan pelajaran"
    }
  ],

  "model": "meta-llama/llama-3.2-11b-vision-instruct:free",
  "usage": { "inputTokens": 240, "outputTokens": 95, "totalTokens": 335 }
}
```

#### 🏷️ Daftar 8 Kategori Niat Gambar (*ImageIntent*) & Contoh Kasus Penggunaannya:

| Kategori `ImageIntent` | Deskripsi Niat Gambar | Contoh Pertanyaan Siswa | Perilaku Respons AI |
|---|---|---|---|
| **`exam_question`** | Foto lembar soal, LKS, atau latihan ujian. | *"Bantu jawab soal nomor 3 ini!"* | Memberikan penjelasan langkah demi langkah (*step-by-step reasoning*) & jawaban akhir dengan LaTeX. |
| **`whiteboard`** | Foto tulisan tangan di papan tulis / catatan. | *"Tolong rapikan dan rangkum catatan ini!"* | Menjalankan Vision OCR tulisan tangan dan merangkum poin-poin pentingnya. |
| **`diagram`** | Foto grafik, skema, atau tabel visual. | *"Jelaskan grafik kurva hubungan ini!"* | Menjelaskan variabel sumbu X/Y, grafik, dan kesimpulan data. |
| **`person_face`** | Foto wajah manusia / pahlawan / penemu. | *"Siapa tokoh penemu di foto ini?"* | Mengidentifikasi nama tokoh sejarah, biografi singkat, dan jasanya. |
| **`object_detection`** | Foto objek fisika, biologi, atau alat lab. | *"Apa nama alat laboratorium mikroskop ini?"* | Mengidentifikasi nama alat/spesimen, fungsi, dan cara penggunaannya. |
| **`landmark_map`** | Foto peta geografi, situs bersejarah, candi. | *"Di mana lokasi peta dan candi di foto ini?"* | Menjelaskan lokasi geografis, sejarah situs, dan koordinat wilayah. |
| **`document_text`** | Foto dokumen resmi, surat, atau sertifikat. | *"Bacakan dan terjemahkan dokumen ini!"* | Menjelaskan struktur dokumen resmi dan mengekstrak teksnya. |
| **`general_photo`** | Foto objek umum di dunia nyata / suasana. | *"Jelaskan apa yang terjadi di foto ini!"* | Mendeskripsikan objek visual umum dan memberikan penjelasan ilmiah. |

---

### 📦 Format Payload Input & Output Persis ke Paket `@badriana/ai-core`

#### 1️⃣ Payload Input JSON ke `ai.chat.generate()`:
```json
{
  "question": "Siapa tokoh penemu dan pahlawan di foto ini?",
  "images": [
    {
      "dataUrl": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAA...",
      "mimeType": "image/png"
    }
  ],
  "system": "Kamu adalah Guru AI yang ramah.",
  "temperature": 0.7
}
```

#### 2️⃣ Payload Output JSON dari `ai.chat.generate()`:
```json
{
  "text": "Foto tersebut menunjukkan Albert Einstein, salah satu fisikawan teoretis terkemuka yang menemukan Teori Relativitas ($E = mc^2$)...",
  "model": "meta-llama/llama-3.2-11b-vision-instruct:free",
  
  "imageAnalysis": [
    {
      "intent": "person_face",
      "summary": "Foto ekspresi wajah / tokoh manusia"
    }
  ],
  
  "citations": [],
  "usage": {
    "inputTokens": 210,
    "outputTokens": 85,
    "totalTokens": 295
  },
  
  "finishReason": "stop",
  "raw": { /* JSON mentah dari provider OpenRouter/OpenAI */ }
}
```

---

### 💻 Contoh Penggunaan TypeScript Per Kategori:

#### A. Mengirim Foto Soal Ujian (`exam_question`):
```typescript
const resExam = await ai.chat.generate({
  question: "Bantu aku menjawab soal nomor 5 matematika ini!",
  images: [{ dataUrl: "data:image/png;base64,soalMathBase64..." }],
});

console.log(resExam.imageAnalysis[0].intent); // "exam_question"
```

#### B. Mengirim Foto Tokoh / Pahlawan (`person_face`):
```typescript
const resFace = await ai.chat.generate({
  question: "Siapa nama tokoh pahlawan di foto wajah ini?",
  images: [{ dataUrl: "data:image/png;base64,pahlawanBase64..." }],
});

console.log(resFace.imageAnalysis[0].intent); // "person_face"
```

#### C. Mengirim Foto Alat Laboratorium (`object_detection`):
```typescript
const resObj = await ai.chat.generate({
  question: "Apa nama alat laboratorium dan benda pada foto ini?",
  images: [{ dataUrl: "data:image/png;base64,alatLabBase64..." }],
});

console.log(resObj.imageAnalysis[0].intent); // "object_detection"
```

#### D. Mengirim Foto Peta Geografi / Candi (`landmark_map`):
```typescript
const resMap = await ai.chat.generate({
  question: "Di mana lokasi situs bersejarah dan peta candi ini?",
  images: [{ dataUrl: "data:image/png;base64,candiBase64..." }],
});

console.log(resMap.imageAnalysis[0].intent); // "landmark_map"
```

---

## 💬 Bab 9: Konsep Arsitektur Chat (Memori Multi-Turn, Sitasi Otomatis & Model Fallback Chain)

### A. Format Memori Percakapan Multi-Turn (`history`)
Untuk mencegah AI dari *amnesia* saat menjawab pertanyaan susulan, paket `@badriana/ai-core` menyuntikkan array riwayat 20 obrolan terakhir ke dalam blok `systemPrompt`:

```text
--- MEMORI RIWAYAT PERCAKAPAN SEBELUMNYA (KONTEKS DISKUSI) ---
Siswa: "Buatkan tabel Pemetaan Materi IPS..."
Guru AI: "Tentu, berikut adalah tabel Pemetaan Materi IPS..."
-------------------------------------------------------------------
Penting: Gunakan riwayat percakapan di atas untuk memahami pertanyaan susulan, rincian kelanjutan, atau rujukan dari siswa!
```

---

### B. Sitasi Otomatis (*Citations & Source Referencing*)
Untuk mencegah *halusinasi AI* dan memberikan bukti referensi modul/halaman PDF:
1. Setiap potongan `contexts` menyertakan metadata `source`, `page`, `title`, dan `chunkId`.
2. Paket menjalankan fungsi `deriveCitations()` di `ai.chat.generate()` untuk melakukan de-duplikasi rujukan ganda.
3. Hasilnya dikembalikan sebagai array `citations` yang siap ditampilkan di UI:
```json
"citations": [
  { "title": "Modul_Geografi_Kelas_10.pdf", "page": 14, "source": "Modul_Geografi_Kelas_10.pdf", "chunkId": "chunk_98a1" }
]
```

---

### C. Model Fallback Chain & Auto-Router (Ketahanan Server 0 Downtime)
Untuk menjamin aplikasi tidak pernah mengalami *crash* saat server OpenRouter/OpenAI mengalami *Rate Limit (Error 429)* atau *Server Overload (Error 503)*:
1. Paket menyusun hirarki model cadangan berurutan: `Model Utama` ➔ `Llama 3.2 Vision` ➔ `DeepSeek R1` ➔ `openrouter/auto`.
2. Jika model utama mengalami error, paket secara instan (dalam hitungan milidetik) berpindah memanggil model cadangan berikutnya.
3. Properti `result.model` memberitahukan model mana yang akhirnya sukses merespon.

---

## 🚀 Bab 10: Tutorial Step-by-Step Aplikasi Full-Stack (Next.js + React UI + Progress Bar)

Berikut adalah panduan membuat aplikasi full-stack lengkap dengan UI Frontend React modern:

### Langkah 1: Membuat Proyek Next.js dari Nol

```bash
npx create-next-app@latest my-ai-app
```

Opsi konfirmasi:
```text
✔ Would you like to use TypeScript? … Yes
✔ Would you like to use ESLint? … Yes
✔ Would you like to use Tailwind CSS? … Yes
✔ Would you like to use `src/` directory? … Yes
✔ Would you like to use App Router? … No
✔ Would you like to customize the default import alias (@/*)? … Yes
```

---

### Langkah 2: Memasang Paket `@badriana/ai-core`

```bash
npm install @badriana/ai-core lucide-react
```

---

### Langkah 3: Konfigurasi API Key & `.env.local`

```env
# File: .env.local
OPENROUTER_API_KEY=sk-or-v1-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

---

### Langkah 4: Helper Client (`src/lib/ai.ts`)

```typescript
// File: src/lib/ai.ts
import { createAI } from "@badriana/ai-core";

export function getAiClient() {
  const apiKey = process.env.OPENROUTER_API_KEY;

  if (!apiKey) {
    throw new Error("❌ OPENROUTER_API_KEY belum dikonfigurasi di file .env.local!");
  }

  return createAI({
    provider: "openrouter",
    apiKey: apiKey,
    chatModel: "google/gemma-2-9b-it:free",
    embeddingModel: "openai/text-embedding-3-small",
  });
}
```

---

### Langkah 5: Membuat API Ingesti Dokumen + Progress Bar (`src/pages/api/ingest.ts`)

```typescript
// File: src/pages/api/ingest.ts
import type { NextApiRequest, NextApiResponse } from "next";
import { createDocument } from "@badriana/ai-core";
import { getAiClient } from "@/lib/ai";

export const globalKnowledgeStore: {
  id: string;
  type: string;
  content: string;
  embedding: number[];
}[] = [];

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  if (req.method !== "POST") return res.status(405).json({ error: "Method Not Allowed" });

  try {
    const { textContent, filename } = req.body;

    if (!textContent || typeof textContent !== "string") {
      return res.status(400).json({ error: "Isi teks dokumen tidak boleh kosong!" });
    }

    const ai = getAiClient();

    const result = await createDocument({
      source: textContent,
      filename: filename || "dokumen.txt",
      embedder: async (texts) => {
        const embRes = await ai.embedding.create({ input: texts });
        return "embeddings" in embRes ? embRes.embeddings : [embRes.embedding];
      },
      onProgress: (progress) => {
        console.log(
          `[INGESTI ${progress.stage.toUpperCase()}] ${progress.percentage}%: ${progress.message}`
        );
      },
    });

    const formattedChunks = result.chunks.map((c) => ({
      id: c.id,
      type: c.type || "text",
      content: c.content,
      embedding: c.embedding || [],
    }));

    globalKnowledgeStore.push(...formattedChunks);

    return res.status(200).json({
      success: true,
      message: `Dokumen "${filename}" berhasil diolah!`,
      totalChunks: result.chunks.length,
      stats: result.stats,
    });
  } catch (error: any) {
    return res.status(500).json({ error: error.message || "Gagal mengolah dokumen" });
  }
}
```

---

### Langkah 6: Membuat API Chat AI (`src/pages/api/chat.ts`)

```typescript
// File: src/pages/api/chat.ts
import type { NextApiRequest, NextApiResponse } from "next";
import { getAiClient } from "@/lib/ai";
import { globalKnowledgeStore } from "./ingest";

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  if (req.method !== "POST") return res.status(405).json({ error: "Method Not Allowed" });

  try {
    const { prompt, history } = req.body;

    if (!prompt || typeof prompt !== "string") {
      return res.status(400).json({ error: "Pertanyaan tidak boleh kosong!" });
    }

    const ai = getAiClient();

    const matchingContexts = globalKnowledgeStore
      .filter((c) => c.content.toLowerCase().includes(prompt.toLowerCase().slice(0, 10)))
      .slice(0, 5);

    let historyContext = "";
    if (Array.isArray(history) && history.length > 0) {
      const pastText = history
        .slice(-10)
        .map((h: any) => `${h.role === "user" ? "Siswa" : "Guru AI"}: ${h.content}`)
        .join("\n");
      historyContext = `\n\n--- RIWAYAT PERCAKAPAN SEBELUMNYA ---\n${pastText}\n--------------------------------------`;
    }

    const systemPrompt = `Kamu adalah Guru AI yang ramah, jelas, dan profesional.${historyContext}`;

    const chatResult = await ai.chat.generate({
      question: prompt,
      system: systemPrompt,
      contexts: matchingContexts.map((c) => ({
        id: c.id,
        content: c.content,
      })),
    });

    return res.status(200).json({
      success: true,
      data: {
        text: chatResult.text,
        model: chatResult.model,
        usage: chatResult.usage,
        citations: chatResult.citations,
      },
    });
  } catch (error: any) {
    return res.status(500).json({ error: error.message || "Gagal menghasilkan jawaban AI" });
  }
}
```

---

### Langkah 7: Membangun UI Frontend React (`src/pages/index.tsx`)

```tsx
// File: src/pages/index.tsx
import React, { useState } from "react";
import Head from "next/head";
import { Sparkles, Send, Upload, FileText } from "lucide-react";

interface Message {
  role: "user" | "assistant";
  content: string;
}

export default function Home() {
  const [docText, setDocText] = useState("");
  const [docName, setDocName] = useState("");
  const [isUploading, setIsUploading] = useState(false);
  const [uploadPct, setUploadPct] = useState(0);
  const [uploadStatusMsg, setUploadStatusMsg] = useState("");

  const [inputPrompt, setInputPrompt] = useState("");
  const [messages, setMessages] = useState<Message[]>([]);
  const [isLoadingAi, setIsLoadingAi] = useState(false);

  const handleUploadDoc = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!docText.trim()) return;

    setIsUploading(true);
    setUploadPct(10);
    setUploadStatusMsg("📄 Membaca dokumen...");

    const t1 = setTimeout(() => { setUploadPct(35); setUploadStatusMsg("✂️ Memecah teks..."); }, 800);
    const t2 = setTimeout(() => { setUploadPct(75); setUploadStatusMsg("🧠 Membuat vektor embedding..."); }, 2000);

    try {
      const res = await fetch("/api/ingest", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          textContent: docText.trim(),
          filename: docName.trim() || "Modul_Pelajaran.txt",
        }),
      });

      clearTimeout(t1);
      clearTimeout(t2);

      const data = await res.json();
      if (!res.ok) throw new Error(data.error);

      setUploadPct(100);
      setUploadStatusMsg(`✅ Sukses! ${data.totalChunks} chunks tersimpan.`);
      setDocText("");
      setDocName("");
    } catch (err: any) {
      setUploadStatusMsg(`❌ Gagal: ${err.message}`);
    } finally {
      setIsUploading(false);
    }
  };

  const handleSendChat = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!inputPrompt.trim() || isLoadingAi) return;

    const userQuery = inputPrompt.trim();
    setInputPrompt("");

    const newMessages: Message[] = [...messages, { role: "user", content: userQuery }];
    setMessages(newMessages);
    setIsLoadingAi(true);

    try {
      const res = await fetch("/api/chat", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          prompt: userQuery,
          history: newMessages.slice(-10),
        }),
      });

      const data = await res.json();
      if (!res.ok) throw new Error(data.error);

      setMessages((prev) => [
        ...prev,
        { role: "assistant", content: data.data.text },
      ]);
    } catch (err: any) {
      setMessages((prev) => [
        ...prev,
        { role: "assistant", content: `⚠️ Error: ${err.message}` },
      ]);
    } finally {
      setIsLoadingAi(false);
    }
  };

  return (
    <>
      <Head>
        <title>Aplikasi AI Sekolah - Powered by @badriana/ai-core</title>
      </Head>

      <div className="min-h-screen bg-slate-950 text-slate-100 font-sans flex flex-col md:flex-row">
        <aside className="w-full md:w-80 bg-slate-900 border-r border-slate-800 p-6 flex flex-col gap-4">
          <div className="flex items-center gap-2 text-blue-400 font-bold text-lg">
            <Sparkles className="w-5 h-5" />
            <span>Upload Knowledge</span>
          </div>

          <form onSubmit={handleUploadDoc} className="flex flex-col gap-3">
            <div>
              <label className="text-xs font-medium text-slate-400">Nama Dokumen:</label>
              <input
                type="text"
                placeholder="Modul_IPS_Kelas_10.txt"
                value={docName}
                onChange={(e) => setDocName(e.target.value)}
                className="w-full bg-slate-800 border border-slate-700 rounded-lg p-2.5 text-xs text-white placeholder-slate-500 focus:outline-none focus:border-blue-500"
              />
            </div>

            <div>
              <label className="text-xs font-medium text-slate-400">Isi Teks Materi:</label>
              <textarea
                rows={6}
                placeholder="Tempelkan isi materi modul di sini..."
                value={docText}
                onChange={(e) => setDocText(e.target.value)}
                className="w-full bg-slate-800 border border-slate-700 rounded-lg p-2.5 text-xs text-white placeholder-slate-500 focus:outline-none focus:border-blue-500"
              />
            </div>

            <button
              type="submit"
              disabled={isUploading || !docText.trim()}
              className="relative overflow-hidden w-full py-3 rounded-lg bg-blue-600 hover:bg-blue-500 text-white font-bold text-xs transition-all flex items-center justify-center gap-2 disabled:opacity-50"
            >
              <Upload className="w-4 h-4 z-10" />
              <span className="z-10">
                {isUploading ? `${uploadStatusMsg} (${uploadPct}%)` : "Simpan Dokumen Knowledge"}
              </span>

              {isUploading && (
                <div
                  className="absolute inset-y-0 left-0 bg-blue-700/80 transition-all duration-300 pointer-events-none"
                  style={{ width: `${uploadPct}%` }}
                />
              )}
            </button>
          </form>
        </aside>

        <main className="flex-1 flex flex-col h-screen bg-slate-950">
          <header className="p-4 border-b border-slate-800 bg-slate-900/50 flex items-center justify-between">
            <h1 className="font-bold text-sm text-slate-200">Chat Assistant AI Sekolah</h1>
          </header>

          <div className="flex-1 overflow-y-auto p-6 space-y-4">
            {messages.length === 0 ? (
              <div className="h-full flex flex-col items-center justify-center text-center text-slate-500">
                <FileText className="w-12 h-12 mb-2 opacity-30" />
                <p className="text-sm">Belum ada percakapan. Ketik pertanyaan di bawah!</p>
              </div>
            ) : (
              messages.map((msg, idx) => (
                <div
                  key={idx}
                  className={`flex ${msg.role === "user" ? "justify-end" : "justify-start"}`}
                >
                  <div
                    className={`max-w-xl p-4 rounded-2xl text-sm leading-relaxed ${
                      msg.role === "user"
                        ? "bg-blue-600 text-white rounded-br-none"
                        : "bg-slate-900 border border-slate-800 text-slate-200 rounded-bl-none"
                    }`}
                  >
                    {msg.content}
                  </div>
                </div>
              ))
            )}

            {isLoadingAi && (
              <div className="flex items-center gap-2 text-xs text-blue-400 font-medium animate-pulse">
                <Sparkles className="w-4 h-4 animate-spin" />
                <span>Guru AI sedang menyusun jawaban...</span>
              </div>
            )}
          </div>

          <form onSubmit={handleSendChat} className="p-4 border-t border-slate-800 flex gap-2">
            <input
              type="text"
              placeholder="Tanyakan materi pelajaran..."
              value={inputPrompt}
              onChange={(e) => setInputPrompt(e.target.value)}
              className="flex-1 bg-slate-900 border border-slate-800 rounded-xl px-4 py-3 text-sm text-white placeholder-slate-500 focus:outline-none focus:border-blue-500"
            />
            <button
              type="submit"
              disabled={isLoadingAi || !inputPrompt.trim()}
              className="px-5 py-3 rounded-xl bg-blue-600 hover:bg-blue-500 text-white font-semibold transition-all disabled:opacity-50"
            >
              <Send className="w-4 h-4" />
            </button>
          </form>
        </main>
      </div>
    </>
  );
}
```

---

### Langkah 8: Menjalankan & Menguji Aplikasi

```bash
npm run dev
```

Buka browser dan akses alamat: 👉 **`http://localhost:3000`**

---

## ❓ Bab 11: FAQ Khusus Paket & Detail API (`@badriana/ai-core`)

### Q1: Bagaimana cara kerja Prisma-style client `createAI()` dalam mengelola namespace API?
Fungsi `createAI()` bertindak sebagai **Inisialisator Client Terpusat** (mirip dengan Prisma Client `new PrismaClient()`). Fungsi ini mengembalikan instance `AI` yang mengekspos namespace domain terpisah sehingga pengembang mendapatkan pengalaman *Autocompletion TypeScript (IntelliSense)* yang sangat bersih:

```typescript
import { createAI } from "@badriana/ai-core";

const ai = createAI({
  provider: "openrouter",
  apiKey: process.env.OPENROUTER_API_KEY,
  chatModel: "google/gemma-2-9b-it:free",
  embeddingModel: "openai/text-embedding-3-small",
});

const chatRes = await ai.chat.generate({ question: "Jelaskan Hukum Newton!" });
const embRes = await ai.embedding.create({ input: ["Teks 1", "Teks 2"] });
const queryRes = await ai.query.search({ vector: [0.01, ...], chunks: [] });
const ctxRes = await ai.context.build({ chunks: [], maxTokens: 4000 });
```

---

### Q2: Bagaimana cara kerja Progress Callback (`onProgress`) dan 5 Tahapan Pipeline di `createDocument()`?
Fungsi `createDocument()` mengelola pipeline ingesti dokumen secara otomatis. Setiap tahapan memancarkan status persentase (`0%` ➔ `100%`) melalui callback `onProgress` yang sangat cocok untuk menggerakkan indikator UI *Progress Bar*:

```typescript
const result = await createDocument({
  source: pdfBuffer,
  embedder: async (texts) => {
    const res = await ai.embedding.create({ input: texts });
    return "embeddings" in res ? res.embeddings : [res.embedding];
  },
  onProgress: (progress) => {
    console.log(`[${progress.stage.toUpperCase()}] ${progress.percentage}%: ${progress.message}`);
  },
});
```

#### 📊 5 Tahapan Pipeline Internal:
1. `parsing` (`10%` ➔ `25%`): Membaca Buffer/Base64 file PDF/TXT, mengurai struktur halaman, serta mengekstrak teks mentah dan gambar biner.
2. `processing_images` (`40%` ➔ `60%`): Memasukkan gambar ke Vision AI / OCR dan mengevaluasi filter kualitas gambar.
3. `chunking` (`65%`): Memecah teks menjadi potongan *Knowledge Chunks* dengan *overlapping* yang aman.
4. `embedding` (`75%` ➔ `95%`): Mengirimkan potongan teks ke provider embedding untuk menghasilkan vektor 1536-dimensi.
5. `completed` (`100%`): Ingesti dokumen selesai dan mengembalikan objek `CreateDocumentResult`.

---

### Q3: Bagaimana cara kerja Filter Gambar Bebas Token (0 Token / Gratis) di `createDocument()`?
Untuk mencegah pemborosan token API dari pengolahan foto sampul bab, logo kementerian, atau garis hiasan header/footer, paket mengimplementasikan **Algoritma Fisik Heuristik (0 Token Cost)**:

```typescript
const docResult = await createDocument({
  source: pdfBuffer,
  embedder,
  filterCoverImages: true,
  minImageDimension: 120,
  maxImageAspectRatio: 4.5,
});
```

#### 🛠️ Alur Penyaringan Fisik:
- **Rule 1 (Cover Page)**: Gambar yang diekstrak dari **Halaman 1** otomatis ditandai sebagai `kind: 'cover'` dan diabaikan.
- **Rule 2 (Logo/Icon)**: Gambar dengan lebar/tinggi `< 120px` otomatis ditandai sebagai `kind: 'logo'` dan diabaikan.
- **Rule 3 (Banner/Divider)**: Gambar dengan rasio aspek `> 4.5` (sangat pipih) ditandai sebagai `kind: 'decorative'` dan diabaikan.

---

### Q4: Bagaimana cara `createQuery()` menggabungkan Vector Search dengan Reciprocal Rank Fusion (RRF)?
Fungsi `createQuery()` melakukan pencarian **Hybrid Search** yang menggabungkan keunggulan *Semantic Vector Search* (Cosine Similarity) dengan *Full-Text Keyword Search* menggunakan algoritma **RRF (Reciprocal Rank Fusion)**:

$$\text{RRF\_Score}(d) = \sum_{m \in M} \frac{1}{k + r_m(d)}$$

```typescript
import { createQuery } from "@badriana/ai-core";

const queryResult = await createQuery({
  vector: questionVector,
  keyword: "hukum newton 2",
  chunks: storedKnowledgeChunks,
  limit: 5,
  rrfK: 60,
});

console.log(queryResult.rankedChunks);
```

---

### Q5: Bagaimana paket mengekstrak dan membersihkan Sitasi (`citations`) di `ai.chat.generate()`?
Saat pengembang memasukkan potongan `contexts` ke dalam `ai.chat.generate()`, fungsi internal `deriveCitations()` membaca metadata dari setiap chunk dan menghasilkan daftar rujukan dokumen yang bersih tanpa duplikasi:

```typescript
const result = await ai.chat.generate({
  question: "Apa bunyi Hukum Newton 1?",
  contexts: matchingChunks,
});

console.log(result.citations);
```

---

### Q6: Bagaimana cara kerja Model Fallback Chain & Auto-Router saat Provider Error?
Paket mengimplementasikan *retry loop* otomatis. Jika model utama mengalami *Rate Limit (HTTP 429)* atau *Server Overload (HTTP 503)*, paket secara otomatis dan instan memanggil model cadangan gratis dalam hirarki tanpa menggagalkan permintaan aplikasi:

1. **Model Utama**: `google/gemma-2-9b-it:free` (Mencoba dipanggil).
2. **Fallback 1**: Jika Error 429 ➔ Otomatis memanggil `meta-llama/llama-3.2-11b-vision-instruct:free`.
3. **Fallback 2**: Jika Error 429 ➔ Otomatis memanggil `deepseek/deepseek-r1:free`.
4. **Fallback 3**: Jika Error 429 ➔ Otomatis memanggil `openrouter/auto`.

---

### Q7: Bagaimana cara `ai.context.build({ chunks, maxTokens })` mencegah Context Window Overflow?
Modul `ai.context.build()` menghitung estimasi jumlah token dari setiap potongan RAG dan menyusun pembangun konteks secara presisi agar total token prompt tidak pernah melebihi batas `maxTokens`:

```typescript
const safeContext = ai.context.build({
  chunks: rawMatchingChunks,
  maxTokens: 3500,
});
```

---

### Q8: Apa saja Jenis Potongan Knowledge Chunks (`ChunkType`) yang dihasilkan Paket?
Paket `@badriana/ai-core` mengklasifikasikan unit pengetahuan menjadi 6 kategori diskriminator:

```typescript
export type ChunkType =
  | 'text'       // Paragraf teks standar
  | 'image'      // Gambar diagram/grafik dengan URL & deskripsi Vision AI
  | 'table'      // Tabel materi pelajaran
  | 'equation'   // Rumus matematika dalam notasi LaTeX ($$ ... $$)
  | 'code'       // Blok kode pemrograman
  | 'markdown';  // Format teks terstruktur Markdown
```

---

### Q9: Bagaimana cara membaca Statistik Performa Ingesti (`result.stats`)?
Fungsi `createDocument()` mengembalikan objek `stats` yang mengukur metrik dokumen dan durasi eksekusi dalam milidetik:

```typescript
const result = await createDocument({ source: pdfBuffer, embedder });

console.log(result.stats);
// {
//   characters: 52821,
//   words: 9312,
//   images: 4,
//   chunks: 18,
//   pages: 14,
//   parseTimeMs: 210,
//   chunkTimeMs: 15,
//   embeddingTimeMs: 532,
//   totalTimeMs: 757
// }
```

---

### Q10: Bagaimana cara menguji fungsi `@badriana/ai-core` di Vitest/Jest tanpa memotong Token API?
Suntikkan *mock callback* pada parameter `embedder` untuk menjalankan unit testing 100% gratis, offline, dan instant:

```typescript
import { describe, it, expect } from "vitest";
import { createDocument } from "@badriana/ai-core";

describe("Unit Test Ingesti Dokumen", () => {
  it("harus memproses dokumen tanpa koneksi internet", async () => {
    const mockEmbedder = async (texts: readonly string[]) =>
      texts.map(() => new Array(1536).fill(0.01));

    const result = await createDocument({
      source: "Ini adalah teks materi pelajaran fisika.",
      filename: "test.txt",
      embedder: mockEmbedder,
    });

    expect(result.chunks.length).toBeGreaterThan(0);
    expect(result.chunks[0].embedding).toHaveLength(1536);
  });
});
```

---

### Q11: Apakah Paket `@badriana/ai-core` Menyembunyikan Data Response dari Provider?
**TIDAK SAMA SEKALI!** Paket mengusung prinsip **Full Output Transparency (Transparansi Penuh 100%)**. 

Di setiap fungsi (`ai.chat.generate()`, `ai.embedding.create()`, dll), paket **selalu menyertakan properti `result.raw`** yang berisi JSON mentah *unfiltered* 100% dari provider (OpenRouter/OpenAI/Gemini). Pengembang dapat membaca ID generasi, system fingerprint, logprobs, finish reason, dan HTTP headers tanpa ada yang ditutupi:

```typescript
const result = await ai.chat.generate({ question: "..." });

console.log(result.text); // Jawaban AI terstruktur
console.log(result.raw);  // 🔴 RAW JSON MENTAH DARI PROVIDER TANPA DITINGGAL/DISEMBUNYIKAN!
```

---

## 🛠️ Bab 12: Troubleshooting & Solusi Error Umum

### 1. Error: `OPENROUTER_API_KEY is missing`
- **Penyebab**: Variabel lingkungan API key belum diset di file `.env` / `.env.local`.
- **Solusi**: Pastikan file `.env` berada di folder akar proyek dan berisi `OPENROUTER_API_KEY=sk-or-v1-xxxx`.

### 2. Error: `HTTP 429 Rate Limit Exceeded`
- **Penyebab**: Kuota API provider terlampaui.
- **Solusi**: Paket secara otomatis menangani ini via Model Fallback Chain (`google/gemma-2-9b-it:free` ➔ `meta-llama/llama-3.2-11b-vision-instruct:free`).

---

## 🛡️ Bab 13: Panduan Tingkat Lanjut Lintas Skenario Produksi (*Production Readiness & Advanced Patterns*)

Bab ini membahas 5 skenario produksi nyata yang paling sering ditanyakan oleh pengembang agar aplikasi AI Anda berjalan 100% aman, cepat, dan terhindar dari *crash* atau *security breach*.

---

### 13.1 Penanganan Upload File PDF Biner Fisik (`Buffer` via Multer & Formidable)

Saat pengguna mengunggah berkas PDF dari komputer/HP melalui form `<input type="file">`, backend API Anda akan menerima data dalam bentuk **Node.js Buffer**. Paket `@badriana/ai-core` dapat menerima `req.file.buffer` secara langsung tanpa perlu menyimpannya ke disk fisik!

#### 💻 Contoh di Express.js dengan Middleware `multer`:

```typescript
import express from "express";
import multer from "multer";
import { createDocument } from "@badriana/ai-core";
import { getAiClient } from "./lib/ai";

const upload = multer({ storage: multer.memoryStorage() }); // Simpan di RAM sebagai Buffer
const app = express();

app.post("/api/upload-pdf", upload.single("document"), async (req, res) => {
  try {
    if (!req.file) return res.status(400).json({ error: "File PDF wajib diunggah!" });

    const ai = getAiClient();

    // Pass req.file.buffer secara langsung ke parameter source!
    const docResult = await createDocument({
      source: req.file.buffer,             // Node.js Buffer dari Multer
      filename: req.file.originalname,     // Nama file asli (misal: "Modul_IPS.pdf")
      embedder: async (texts) => {
        const embRes = await ai.embedding.create({ input: texts });
        return "embeddings" in embRes ? embRes.embeddings : [embRes.embedding];
      },
    });

    res.json({ success: true, totalChunks: docResult.chunks.length });
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
});
```

---

### 13.2 Integrasi Vector Database Produksi (Supabase `pgvector` & PostgreSQL)

Di lingkungan produksi skala besar dengan jutaan data, chunks RAG dan vektor embedding 1536-dimensi disimpan di database relasional yang mendukung pencarian Cosine Similarity (`<=>`):

#### 📜 1. Skema DDL PostgreSQL / Supabase (`pgvector`):
```sql
-- Aktifkan ekstensi vector di PostgreSQL / Supabase
CREATE EXTENSION IF NOT EXISTS vector;

-- Buat tabel penampung Knowledge Chunks
CREATE TABLE knowledge_chunks (
  id VARCHAR(64) PRIMARY KEY,
  document_id VARCHAR(64),
  content TEXT NOT NULL,
  embedding vector(1536), -- Kolom Vektor 1536-Dimensi
  created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Buat HNSW Index untuk pencarian vektor kilat (< 5ms)
CREATE INDEX ON knowledge_chunks USING hnsw (embedding vector_cosine_ops);
```

#### 💻 2. Query Pencarian Semantik di Node.js Backend:
```typescript
import { pgPool } from "@/lib/db";

export async function searchSimilarChunksFromPgVector(questionVector: number[]) {
  // Query Cosine Similarity (<=>) mengambil Top 5 Chunks paling relevan
  const query = `
    SELECT id, content, 1 - (embedding <=> $1::vector) AS similarity_score
    FROM knowledge_chunks
    ORDER BY embedding <=> $1::vector
    LIMIT 5;
  `;

  const result = await pgPool.query(query, [JSON.stringify(questionVector)]);
  return result.rows; // [{ id, content, similarity_score }, ...]
}
```

---

### 13.3 Menjalankan AI Lokal / Offline (Ollama / Local LLM / Internal Server)

Untuk perusahaan yang memiliki regulasi **privasi data ketat** sehingga tidak diizinkan mengirim teks ke cloud internet, Anda dapat mengarahkan `@badriana/ai-core` ke server **Ollama lokal** yang berjalan di server internal:

```typescript
import { createAI } from "@badriana/ai-core";

// Menggunakan Ollama Lokal (100% Offline & Gratis)
const ai = createAI({
  provider: "ollama",
  baseUrl: "http://localhost:11434/v1", // URL Server Ollama Lokal
  chatModel: "llama3.2:latest",
  embeddingModel: "nomic-embed-text:latest",
});

const result = await ai.chat.generate({
  question: "Jelaskan struktur sel hewan!",
});
```

---

### 13.4 Aturan Keamanan API Key (*Server-Side Execution Guarantee*)

> [!CAUTION]
> **JANGAN PERNAH MEMANGGIL `createAI()` DI DALAM KOMPONEN CLIENT-SIDE REACT (`src/pages/index.tsx` ATAU `app/page.tsx`)!**

Jika `createAI()` dipanggil di browser React Client Component, kunci API (`OPENROUTER_API_KEY` / `OPENAI_API_KEY`) akan **ter-expose di Inspect Element Network tab browser** dan bisa dicuri oleh pengguna jahat.

**Aturan Keamanan Wajib**:
- Panggil `createAI()` **hanya di Backend API Routes** (`src/pages/api/*`, server Express, NestJS controller).
- Frontend React hanya bertugas mengirim `fetch('/api/chat')` ke backend Anda sendiri.

---

### 13.5 Penanganan PDF Terkunci Password & PDF Scan Kamera

Saat pengguna mengunggah file PDF yang rusak, dilindungi kata sandi (*Password Protected*), atau PDF murni berupa foto *scan* kamera:

```typescript
import { createDocument, EmptyDocumentError } from "@badriana/ai-core";

try {
  const result = await createDocument({
    source: userUploadedBuffer,
    filename: "dokumen.pdf",
    embedder,
    // Aktifkan Vision Image Processor jika PDF merupakan scan foto kamera tanpa teks digital
    imageProcessor: async (img) => {
      // Panggil Vision AI untuk OCR teks dari foto scan
      return `[Hasil OCR Gambar Halaman ${img.page}]: Teks materi...`;
    },
  });
} catch (error) {
  if (error instanceof EmptyDocumentError) {
    console.error("❌ Dokumen kosong, terkunci password, atau tidak terbaca!");
  } else {
    console.error("❌ Error Ingesti:", error);
  }
}
```

---

Selamat! Anda kini memiliki **Katalog Lengkap, Urutan Terstruktur Logis, Dukungan Universal Backend, Kesiapan Skala Produksi, & Dokumentasi Terkuat `@badriana/ai-core`** yang 100% transparan, terstruktur, dan siap digunakan untuk pembangunan aplikasi AI di Next.js, Express.js, NestJS, Fastify, maupun Deno/Bun! 🎉

