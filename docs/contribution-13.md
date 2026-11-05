# 📘 Guía Exhaustiva para Extender **sapat** con Nuevas APIs de Transcripción  
*Proyecto: **AI Transcription Tool** – Plataforma: **Algora** – Recompensa: **$150 USD***  

> **Autor:** @mau16ro (Desarrollador Senior)  
> **Fecha:** 5 nov 2026  

---  

## 1️⃣ Resumen Ejecutivo  

**Objetivo**  
Ampliar el repositorio **[sapat](https://github.com/nkkko/sapat)** para que soporte de forma nativa **todas las principales APIs de transcripción de audio** (OpenAI Whisper, Groq, Anthropic Claude, Azure Speech‑to‑Text, Google Cloud Speech‑to‑Text, etc.). La solución debe:

- Mantener la **arquitectura limpia** y **extensible** (principio Open‑Closed).  
- Proveer un **SDK tipado** en TypeScript para que los AI‑engineers lo integren en sus pipelines sin fricciones.  
- Incluir **documentación**, **tests unitarios** y **guía de instalación** para producción.  

**Beneficios**  

| Beneficio | Impacto |
|-----------|---------|
| **Flexibilidad** – Cambiar de proveedor sin tocar la lógica de negocio. | Reduce tiempo de integración y costes de migración. |
| **Escalabilidad** – Añadir futuros proveedores con un solo archivo. | Asegura longevidad del proyecto. |
| **Calidad** – Tipado estricto + pruebas automáticas. | Minimiza bugs en entornos críticos. |
| **Usabilidad** – CLI y SDK listos para consumo. | Mejora la adopción por la comunidad. |

---  

## 2️⃣ Arquitectura de la Solución  

### 2.1 Visión General  

```
src/
│
├─ config/
│   └─ providers.ts               # Configuración declarativa de proveedores
│
├─ core/
│   ├─ Transcriber.ts             # Orquestador genérico
│   └─ types.ts                   # Definiciones de tipos comunes
│
├─ providers/
│   ├─ BaseProvider.ts            # Clase abstracta + contrato
│   ├─ OpenAIProvider.ts          # Implementación Whisper
│   ├─ GroqProvider.ts            # Implementación Groq
│   ├─ AnthropicProvider.ts       # (ejemplo) Claude
│   ├─ AzureProvider.ts           # Azure Speech‑to‑Text
│   └─ GoogleProvider.ts          # Google Cloud Speech
│
├─ utils/
│   └─ http.ts                    # Wrapper fetch/axios con retries
│
└─ index.ts                       # Export público del SDK
```

### 2.2 Principios de Diseño  

| Principio | Aplicación |
|-----------|------------|
| **Open‑Closed** | `BaseProvider` define la interfaz; cada nuevo API extiende esa clase sin tocar código existente. |
| **Dependency Inversion** | `Transcriber` depende de la abstracción `IProvider` y no de implementaciones concretas. |
| **Single Responsibility** | Cada provider gestiona sólo la lógica de su API (auth, payload, parsing). |
| **Fail‑Fast & Retries** | `utils/http.ts` incorpora back‑off exponencial y circuit‑breaker básico. |
| **Configuración Declarativa** | `config/providers.ts` permite habilitar/deshabilitar proveedores mediante un JSON simple. |

### 2.3 Flujo de Ejecución  

1. **Inicialización** – El consumidor crea una instancia de `Transcriber` indicando el `providerId`.  
2. **Factory** – `Transcriber` delega al `ProviderFactory` que devuelve la clase concreta (`OpenAIProvider`, `GroqProvider`, …).  
3. **Transcripción** – El provider prepara la petición (auth, multipart/form‑data), envía la solicitud vía `http.request`, y devuelve un objeto tipado `TranscriptionResult`.  
4. **Post‑procesado** – Opcional: normalizar timestamps, segmentar por hablantes, etc.  

---  

## 3️⃣ Código Fuente – Implementación Lista para Producción  

> **Requisitos:** Node ≥ 18, TypeScript ≥ 5.0, `npm` o `yarn`.  

### 3.1 Tipos Compartidos (`src/core/types.ts`)

```ts
// src/core/types.ts
/** Tipo de audio aceptado por los proveedores */
export type AudioInput = Buffer | ReadableStream | string; // string = path o URL

/** Resultado estandarizado de una transcripción */
export interface TranscriptionResult {
  /** Texto completo transcrito */
  text: string;
  /** Duración total en segundos */
  duration: number;
  /** Opcional – segmentos por hablante o timestamps */
  segments?: Array<{
    start: number;
    end: number;
    text: string;
    speaker?: string;
  }>;
}

/** Parámetros genéricos que pueden aceptar los providers */
export interface TranscribeOptions {
  /** Idioma del audio (p.ej. "en", "es") */
  language?: string;
  /** Modelo a usar (cuando aplica) */
  model?: string;
  /** Parámetros específicos del proveedor (passthrough) */
  providerParams?: Record<string, unknown>;
}
```

### 3.2 Clase Base (`src/providers/BaseProvider.ts`)

```ts
// src/providers/BaseProvider.ts
import { AudioInput, TranscribeOptions, TranscriptionResult } from '../core/types';
import { httpRequest } from '../utils/http';

export abstract class BaseProvider {
  /** ID único del proveedor (ex.: "openai", "groq") */
  abstract readonly id: string;

  /** Cabecera de autorización o token */
  protected abstract getAuthHeaders(): Record<string, string>;

  /** URL del endpoint de transcripción */
  protected abstract getEndpoint(): string;

  /** Convierte la respuesta cruda del API en TranscriptionResult */
  protected abstract parseResponse(raw: any): TranscriptionResult;

  /** Implementación genérica de envío */
  async transcribe(
    audio: AudioInput,
    opts: TranscribeOptions = {}
  ): Promise<TranscriptionResult> {
    const headers = {
      ...this.getAuthHeaders(),
      ...this.getContentHeaders(audio),
    };

    const body = await this.buildRequestBody(audio, opts);
    const raw = await httpRequest(this.getEndpoint(), {
      method: 'POST',
      headers,
      body,
    });

    return this.parseResponse(raw);
  }

  /** Hook para crear multipart/form‑data o JSON según el provider */
  protected abstract buildRequestBody(
    audio: AudioInput,
    opts: TranscribeOptions
  ): Promise<BodyInit>;

  /** Determina el Content‑Type apropiado */
  private getContentHeaders(audio: AudioInput): Record<string, string> {
    // Si el provider usa multipart/form‑data, dejar que buildRequestBody lo gestione.
    return {}; // por defecto vacío
  }
}
```

### 3.3 Implementación OpenAI (`src/providers/OpenAIProvider.ts`)

```ts
// src/providers/OpenAIProvider.ts
import { BaseProvider } from './BaseProvider';
import { AudioInput, TranscribeOptions, TranscriptionResult } from '../core/types';
import FormData from 'form-data';
import fs from 'fs';
import path from 'path';

export class OpenAIProvider extends BaseProvider {
  readonly id = 'openai';

  constructor(private readonly apiKey: string) {
    super();
    if (!apiKey) throw new Error('OpenAI API key is required');
  }

  protected getAuthHeaders() {
    return { Authorization: `Bearer ${this.apiKey}` };
  }

  protected getEndpoint() {
    return 'https://api.openai.com/v1/audio/transcriptions';
  }

  protected async buildRequestBody(
    audio: AudioInput,
    opts: TranscribeOptions
  ): Promise<FormData> {
    const form = new FormData();

    // ---- audio ----
    if (typeof audio === 'string') {
      // path o URL
      const resolved = path.resolve(audio);
      form.append('file', fs.createReadStream(resolved));
    } else if (audio instanceof Buffer) {
      form.append('file', audio, { filename: 'audio.wav' });
    } else {
      // Stream
      form.append('file', audio as any);
    }

    // ---- parámetros OpenAI ----
    form.append('model', opts.model ?? 'whisper-1');
    if (opts.language) form.append('language', opts.language);
    // Passthrough de parámetros específicos
    if (opts.providerParams) {
      Object.entries(opts.providerParams).forEach(([k, v]) =>
        form.append(k, String(v))
      );
    }

    return form;
  }

  protected parseResponse(raw: any): TranscriptionResult {
    // OpenAI devuelve: { text: string }
    if (!raw?.text) {
      throw new Error('Invalid response from OpenAI');
    }
    return {
      text: raw.text,
      duration: raw.duration ?? 0,
    };
  }
}
```

### 3.4 Implementación Groq (`src/providers/GroqProvider.ts`)

```ts
// src/providers/GroqProvider.ts
import { BaseProvider } from './BaseProvider';
import { AudioInput, TranscribeOptions, TranscriptionResult } from '../core/types';
import { httpRequest } from '../utils/http';
import fs from 'fs';
import path from 'path';

export class GroqProvider extends BaseProvider {
  readonly id = 'groq';
  private readonly endpoint = 'https://api.groq.com/openai/v1/audio/transcriptions';

  constructor(private readonly apiKey: string) {
    super();
    if (!apiKey) throw new Error('Groq API key is required');
  }

  protected getAuthHeaders() {
    return { Authorization: `Bearer ${this.apiKey}` };
  }

  protected getEndpoint() {
    return this.endpoint;
  }

  /** Groq acepta JSON (base64) → simplificado */
  protected async buildRequestBody(
    audio: AudioInput,
    opts: TranscribeOptions
  ): Promise<string> {
    let buffer: Buffer;
    if (typeof audio === 'string') {
      buffer = await fs.promises.readFile(path.resolve(audio));
    } else if (audio instanceof Buffer) {
      buffer = audio;
    } else {
      // Stream → Buffer
      buffer = await new Promise<Buffer>((resolve, reject) => {
        const chunks: Buffer[] = [];
        (audio as any).on('data', (c: Buffer) => chunks.push(c));
        (audio as any).on('end', () => resolve(Buffer.concat(chunks)));
        (audio as any).on('error', reject);
      });
    }

    const payload = {
      model: opts.model ?? 'whisper-large-v3',
      file: buffer.toString('base64'),
      language: opts.language,
      ...opts.providerParams,
    };

    return JSON.stringify(payload);
  }

  protected async transcribe(
    audio: AudioInput,
    opts: TranscribeOptions = {}
  ): Promise<TranscriptionResult> {
    // Groq requiere Content-Type: application/json
    const body = await this.buildRequestBody(audio, opts);
    const raw = await httpRequest(this.getEndpoint(), {
      method: 'POST',
      headers: {
        ...this.getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body,
    });

    return this.parseResponse(raw);
  }

  protected parseResponse(raw: any): TranscriptionResult {
    // Respuesta típica: { text: string }
    if (!raw?.text) {
      throw new Error('Invalid response from Groq');
    }
    return { text: raw.text, duration: raw.duration ?? 0 };
  }
}
```

### 3.5 Otros Providers (Ejemplo: Anthropic, Azure, Google)  

> **Plantilla** – Copiar `BaseProvider` y reemplazar `getEndpoint`, `buildRequestBody` y `parseResponse` según la documentación oficial.  

```ts
// src/providers/AnthropicProvider.ts
export class AnthropicProvider extends BaseProvider {
  readonly id = 'anthropic';
  constructor(private readonly apiKey: string) { super(); }
  protected getAuthHeaders() { return { 'x-api-key': this.apiKey }; }
  protected getEndpoint() { return 'https://api.anthropic.com/v1/audio/transcriptions'; }
  // ... (similar a OpenAI, pero con campos "prompt", "temperature", etc.)
}
```

### 3.6 Factory y Orquestador (`src/core/Transcriber.ts`)

```ts
// src/core/Transcriber.ts
import { BaseProvider } from '../providers/BaseProvider';
import { OpenAIProvider } from '../providers/OpenAIProvider';
import { GroqProvider } from '../providers/GroqProvider';
import { AnthropicProvider } from '../providers/AnthropicProvider';
import { AudioInput, TranscribeOptions, TranscriptionResult } from './types';
import { ProviderConfig } from '../config/providers';

type ProviderCtor = new (...args: any[]) => BaseProvider;

/**
 * Registro estático de proveedores.
 * Cada entrada indica la clase y los argumentos requeridos (ej. API‑key).
 */
const PROVIDER_REGISTRY: Record<string, ProviderCtor> = {
  openai: OpenAIProvider,
  groq: GroqProvider,
  anthropic: AnthropicProvider