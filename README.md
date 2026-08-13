# @badriana/ai-core

[![npm version](https://img.shields.io/npm/v/@badriana/ai-core.svg?style=flat-square&color=blue)](https://www.npmjs.com/package/@badriana/ai-core)
[![license](https://img.shields.io/npm/l/@badriana/ai-core.svg?style=flat-square)](./LICENSE)
[![bundle size](https://img.shields.io/bundlephobia/minzip/@badriana/ai-core?style=flat-square&color=success)](https://bundlephobia.com/package/@badriana/ai-core)
[![typescript](https://img.shields.io/badge/TypeScript-Strict-blue?style=flat-square&logo=typescript)](https://www.typescriptlang.org/)
[![vitest](https://img.shields.io/badge/Unit%20Tests-60%2F60%20Passed-brightgreen?style=flat-square&logo=vitest)](./unittests)

> ⚡ **Universal AI Engineering Framework & Prisma-style Client untuk Seluruh Ekosistem Node.js (Express.js, NestJS, Fastify, Hono, Next.js).**  
> *Solusi All-in-One untuk Ingesti Dokumen RAG (PDF/TXT/MD), Vision OCR, Text-to-Speech (TTS), Speech-to-Text (STT), Batch Vector Embedding, RRF Query Search, serta Chat Completion & Streaming across OpenRouter, OpenAI, dan Gemini.*

---

## 💡 Apa Itu `@badriana/ai-core`?

`@badriana/ai-core` dirancang untuk menyederhanakan pengembangan aplikasi berbasis AI dan RAG (Retrieval-Augmented Generation). Dengan antarmuka terpusat ala Prisma Client (`createAI()`), Anda tidak perlu lagi menulis kode berulang (*boilerplate*) untuk HTTP requests, parsing PDF, ekstraksi gambar, chunking teks, pencarian vektor, transkripsi audio, sintesis suara, atau kalkulasi FinOps token usage.

---

## 🛠️ Fitur Utama Framework

1. **Client Unified `createAI()`**:
   Satu antarmuka terpusat untuk mengelola `ai.chat`, `ai.embedding`, `ai.image`, `ai.audio`, `ai.context`, `ai.document`, `ai.query`, dan `ai.utils`.
2. **Ingesti Dokumen Multimodal 100% Jujur (`createDocument`)**:
   Mendukung file PDF, Markdown, TXT, dan Base64. Mengekstrak teks, tabel, rumus, dan gambar visual HD asli. **0 Gambar Sintetis Palsu (Zero Synthetic Fabrication)**.
3. **Penyaringan Gambar 0-Token Cost**:
   Secara fisik menyaring foto sampul Halaman 1, logo/ikon `< 120px`, dan banner pipih tanpa memakan token API sama sekali (0 Token API Cost).
4. **Contextual Caption Linking**:
   Mendeteksi judul diagram/figure bernomor (`Gambar 1: ...`) dan mengikatnya secara presisi ke `ImageChunk` untuk akurasi RAG 100%.
5. **Dukungan Audio Mandiri (`ai.audio`)**:
   - `ai.audio.speak()` — Text-to-Speech sintesis suara AI (MP3).
   - `ai.audio.transcribe()` — Speech-to-Text transkripsi suara mikrofon (Whisper).
6. **Indikator Progres Real-Time (`onProgress`)**:
   Mengirimkan persentase progres (`0%` ➔ `100%`) dan status tahapan (`parsing`, `chunking`, `processing_images`, `embedding`, `completed`).
7. **Chat Completion & Streaming (`ai.chat.generate` & `ai.chat.stream`)**:
   Mendukung generasi jawaban terstruktur maupun *streaming*, lengkap dengan konteks RAG, sitasi otomatis (*citations*), dan respons mentah provider (*raw payload*).
8. **Batch Vector Embedding & Quality Assertion (`ai.embedding.create`)**:
   Generasi vektor otomatis 1536-D dengan validasi dimensi ketat (*dimension assertion*) untuk mencegah vektor rusak/kosong.

---

## 📦 Instalasi

```bash
npm install @badriana/ai-core
```

---

## 🚀 Quick Start (Contoh Penggunaan Singkat)

### 1. Inisialisasi AI Client

```typescript
import { createAI } from '@badriana/ai-core';

const ai = createAI({
  provider: 'openrouter',
  apiKey: process.env.OPENROUTER_API_KEY,
  chatModel: 'google/gemma-2-9b-it:free',
  embeddingModel: 'openai/text-embedding-3-small',
});
```

---

### 2. Tanya Jawab AI (`ai.chat.generate`)

```typescript
const result = await ai.chat.generate({
  question: 'Jelaskan apa itu Kurikulum Merdeka secara singkat!',
  system: 'Kamu adalah Guru AI yang ramah dan profesional.',
});

console.log(result.text);       // Jawaban teks AI
console.log(result.usage);      // { inputTokens: 45, outputTokens: 120, totalTokens: 165 }
console.log(result.raw);        // Raw JSON response dari OpenRouter/OpenAI
```

---

### 3. Ingesti Dokumen RAG & Progress Bar (`createDocument`)

```typescript
import { createDocument } from '@badriana/ai-core';

const docResult = await createDocument({
  source: pdfBuffer,
  filename: 'Modul_Fisika.pdf',
  // 1. Embedder callback
  embedder: async (texts) => {
    const res = await ai.embedding.create({ input: texts });
    return "embeddings" in res ? res.embeddings : [res.embedding];
  },
  // 2. Real-time Progress callback (0% - 100%)
  onProgress: (progress) => {
    console.log(`[${progress.stage.toUpperCase()}] ${progress.percentage}% - ${progress.message}`);
  },
});

console.log(docResult.chunks); // Array Knowledge Chunks (Text, Image, Table)
console.log(docResult.stats);  // Metrik waktu & statistik dokumen
```

---

### 4. Audio Text-to-Speech & Speech-to-Text (`ai.audio`)

```typescript
// 🔊 Synthesize Speech (Text -> MP3 Buffer)
const audioResult = await ai.audio.speak({
  text: "Selamat datang di sistem RAG Multimodal ai-core!",
  voice: "nova",
});

// 🎤 Transcribe Speech (Audio Base64 -> Text)
const transcript = await ai.audio.transcribe({
  file: audioBase64String,
  language: "id",
});
```

---

## 🧪 Unit Testing Suite

Repository `@badriana/ai-core` dilengkapi dengan test suite terpusat berbasis **Vitest** yang menguji 100% modul `src/` (60 Unit Test Cases Passed):

```bash
# Menjalankan seluruh unit testing sekali jalan
npm test

# Menjalankan mode interaktif watch mode
npx vitest
```

---

## 📖 Dokumentasi Spesifikasi & Arsitektur Framework

1. 🎓 **[TUTORIAL.md](./TUTORIAL.md)** — Panduan tutorial terpisah lengkap dari nol untuk Next.js, Express, NestJS, Fastify, & Koa.
2. 🏛️ **[ARSITEKTUR.md](./ARSITEKTUR.md)** — Spesifikasi arsitektur 5-layer internal, data flow, dan skema interface TypeScript.
3. 💼 **[BISNIS.md](./BISNIS.md)** — Arsitektur alur bisnis, model monetisasi, dan studi kasus enterprise.

---

### 📚 Contoh Arsitektur Backend Framework (Router ➔ Controller ➔ Service ➔ Repository):
1. 🚀 Express.js: **[express.example.md](./express.example.md)**
2. 🦁 NestJS: **[nestjs.example.md](./nestjs.example.md)**
3. ⚛️ Next.js App Router: **[nextjs.example.md](./nextjs.example.md)**
4. ⚡ Fastify: **[fastify.example.md](./fastify.example.md)**
5. 🌿 Koa.js: **[koa.example.md](./koa.example.md)**

---

## 📄 Lisensi

MIT © [badriana](https://github.com/badriana400952)
