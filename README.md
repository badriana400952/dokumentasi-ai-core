# @badriana/ai-core

[![npm version](https://img.shields.io/npm/v/@badriana/ai-core.svg?style=flat-square&color=blue)](https://www.npmjs.com/package/@badriana/ai-core)
[![license](https://img.shields.io/npm/l/@badriana/ai-core.svg?style=flat-square)](./LICENSE)
[![bundle size](https://img.shields.io/bundlephobia/minzip/@badriana/ai-core?style=flat-square&color=success)](https://bundlephobia.com/package/@badriana/ai-core)
[![typescript](https://img.shields.io/badge/TypeScript-Strict-blue?style=flat-square&logo=typescript)](https://www.typescriptlang.org/)

> ⚡ **Universal AI Engineering Framework & Prisma-style Client untuk Seluruh Node.js (Express.js, NestJS, Fastify, Hono, Next.js).**  
> *Solusi All-in-One untuk Ingesti Dokumen RAG (PDF/TXT/MD), Vision OCR, Batch Vector Embedding, RRF Query Search, serta Chat Completion & Streaming across OpenRouter, OpenAI, dan Gemini.*

---

## 💡 Apa Itu `@badriana/ai-core`?

`@badriana/ai-core` dirancang untuk menyederhanakan pengembangan aplikasi berbasis AI dan RAG (Retrieval-Augmented Generation). Dengan antarmuka yang bersih ala Prisma Client (`createAI()`), Anda tidak perlu lagi menulis kode berulang (*boilerplate*) untuk HTTP requests, parsing PDF, ekstraksi gambar, chunking teks, pencarian vektor, atau penanganan token usage.

---

## 🛠️ Fitur Utama Framework

1. **Client Unified `createAI()`**:
   Satu antarmuka terpusat untuk mengelola `ai.chat`, `ai.embedding`, `ai.image`, `ai.audio`, `ai.context`, dan `ai.document`.
2. **Ingesti Dokumen Multimodal (`createDocument`)**:
   Mendukung file PDF, Markdown, TXT, dan Base64. Mengekstrak teks, tabel, rumus, dan gambar via Vision OCR secara otomatis.
3. **Indikator Progres Real-Time (`onProgress`)**:
   Mengirimkan persentase progres (`0%` ➔ `100%`) dan status tahapan (`parsing`, `chunking`, `processing_images`, `embedding`, `completed`) untuk *Progress Bar* UI.
4. **Chat Completion & Streaming (`ai.chat.generate` & `ai.chat.stream`)**:
   Mendukung generasi jawaban tunggal maupun *streaming*, lengkap dengan konteks RAG, sitasi otomatis (*citations*), dan respons mentah provider (*raw payload*).
5. **Batch Vector Embedding (`ai.embedding.create`)**:
   Generasi vektor otomatis dengan deteksi dimensi, batasan batching, serta metrik biaya/token usage.
6. **Query & Reranker Engine (`createQuery`)**:
   Mesin kueri pencarian vektor semantik dengan pembangun konteks otomatis.

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

## 🎓 Tutorial Lengkap & Arsitektur Framework

Ingin membangun aplikasi AI RAG & Multimodal dari nol menggunakan `@badriana/ai-core` pada framework favorit Anda?  
Kami telah menyediakan panduan tutorial langkah demi langkah dari nol tanpa ada yang terlewat:

👉 **[Baca Tutorial Next.js Lengkap (TUTORIAL.md)](./TUTORIAL.md)**

### 📚 Contoh Arsitektur Backend Framework (Router ➔ Controller ➔ Service ➔ Repository):
1. 🚀 Express.js: **[express.example.md](./express.example.md)**
2. 🦁 NestJS: **[nestjs.example.md](./nestjs.example.md)**
3. ⚛️ Next.js App Router: **[nextjs.example.md](./nextjs.example.md)**
4. ⚡ Fastify: **[fastify.example.md](./fastify.example.md)**
5. 🌿 Koa.js: **[koa.example.md](./koa.example.md)**

### 🎨 Contoh Integrasi Frontend React/Next.js UI:
- ⚛️ React 18/19 Custom Hooks & UI Dashboard: **[react.example.md](./react.example.md)**

---

## 📄 Lisensi

MIT © [badriana](https://github.com/badriana400952)
