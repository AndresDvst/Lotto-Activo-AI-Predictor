# 🎰 Lotto Activo AI Predictor

> Sistema multi-agente de automatización en **n8n** que utiliza **5 modelos de IA** (Gemma 26B, Gemma 31B, Mistral, Llama 70B, GPT 120B) para analizar patrones estadísticos históricos de la lotería Lotto Activo Venezuela y generar predicciones de secuencias diarias de sorteos.

---

## 🏗️ Arquitectura General

El sistema está compuesto por **4 workflows** conectados entre sí que se ejecutan de forma coordinada:

```
┌─────────────────────────────────────────────────────────────┐
│                    🎯 INICIADOR                              │
│  Schedule Trigger → Scraping → Comparador → Switch          │
│  (Orquestador principal, 3x/día: 6AM, 12PM, 6PM)           │
└──────────┬──────────────┬─────────────────┬─────────────────┘
           │              │                 │
           ▼              ▼                 ▼
  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
  │ 🤖 GEMMA     │ │ 🤖 MISTRAL/  │ │ 🤖 GPT 120B  │
  │ 26B + 31B    │ │    LLAMA 70B │ │              │
  │ (Google)     │ │ (Groq/Cloud) │ │ (Groq)       │
  └──────────────┘ └──────────────┘ └──────────────┘
           │              │                 │
           └──────────────┴─────────────────┘
                          │
                          ▼
              ┌─────────────────────┐
              │  📊 Supabase        │
              │  Predicciones +     │
              │  Resultados reales  │
              └─────────────────────┘
                          │
                          ▼
              ┌─────────────────────┐
              │  📱 Telegram        │
              │  Notificaciones     │
              │  diarias            │
              └─────────────────────┘
```

---

## 📂 Workflows

### 1️⃣ `Iniciador.json` — Orquestador Principal

El cerebro del sistema. Se ejecuta 3 veces al día y coordina todos los sub-workflows.

**Flujo:**
1. ⏰ **Schedule Trigger** — dispara a las 6AM, 12PM, 6PM (hora Venezuela GMT-4)
2. 📥 **Tabla Lotto Activo** — lee predicciones anteriores desde Supabase
3. 📆 **Frecuencia HorariaLG** — genera rango de fechas de los últimos 3 días
4. 🌐 **Extraer Lotto Activo** — scraping de `loteriadehoy.com` con resultados reales
5. 🔄 **Setear DatosLG** — parsea y normaliza los resultados en HTML
6. 🧮 **Comparador** — compara predicciones guardadas vs resultados reales por **hora exacta**
7. 🔀 **Switch** — evalúa aciertos por modelo y despacha a sub-workflows

**Condiciones del Switch (12 ramas):**
| Rama | Condición | Acción |
|------|-----------|--------|
| Gemma 31B ✅ | `acerto_gemma31b = true` | Notifica acierto |
| Gemma 31B ❌ | `acerto_gemma31b = false` | Lanza análisis Gemma |
| Gemma 26B ✅ | `acerto_gemma26b = true` | Notifica acierto |
| Gemma 26B ❌ | `acerto_gemma26b = false` | Lanza análisis Gemma |
| Mistral ✅/❌ | `acerto_mistral` | Lanza análisis Mistral/Llama |
| Llama ✅/❌ | `acerto_llama` | Lanza análisis Mistral/Llama |
| GPT ✅/❌ | `acerto_gpt` | Lanza análisis GPT |
| Fecha diferente | `fecha_prediccion ≠ hoy` | Lanza todos los sub-workflows |
| Sin predicciones | `fecha_prediccion = null` | Lanza todos los sub-workflows |

---

### 2️⃣ `Lotto_Activo_Gemma.json` — Predictor Gemma 26B + 31B

Sub-workflow especializado en modelos Google Gemini/Gemma vía API de Google.

**Flujo interno:**
```
When Executed → Wait → Tabla Supabase → Estadísticas LottoActivo
→ Scraping 90 días → Setear Datos → Setear Estadísticas
→ Gemma 26B → Gemma 31B → Comparar Modelos
→ Enviar Predicción (Telegram) → Guardar en Supabase
```

**Modelos usados:** `gemma-4-26b-a4b-it`, `gemma-4-31b-a4b-it` (Google Gemini API)

---

### 3️⃣ `Lotto_Activo_Mistral_Llama.json` — Predictor Mistral + Llama 70B

Sub-workflow para modelos open-source vía Groq y Mistral Cloud.

**Flujo interno:**
```
When Executed → Wait → Tabla Supabase → Estadísticas LottoActivo
→ Scraping 90 días → Setear Datos → Setear Estadísticas
→ Llama 70B → Mistral → Comparar Modelos
→ Enviar Predicción (Telegram) → Guardar en Supabase
```

**Modelos usados:** `llama-3.3-70b` (Groq), `mistral-large-latest` (Mistral Cloud)

---

### 4️⃣ `Lotto_Activo_GPT_120B.json` — Predictor GPT 120B

Sub-workflow para el modelo GPT de mayor tamaño vía Groq.

**Flujo interno:**
```
When Executed → Wait → Tabla Supabase → Estadísticas LottoActivo
→ Scraping 90 días → Setear Datos → Setear Estadísticas
→ GPT 120B → Comparar Modelos (con Groq Chat Model)
→ Enviar Predicción (Telegram) → Guardar en Supabase
```

---

## 📊 Métricas y Estadísticas Calculadas

Para cada ejecución, el sistema calcula automáticamente:

| Métrica | Descripción |
|---------|-------------|
| 📈 **Chi²** | Prueba de aleatoriedad (gl=37, crítico=52.19) |
| 🔢 **Frecuencias** | Conteo histórico por número (0, 00, 1-36) |
| ⏰ **Frecuencia hora exacta** | Top 3 por slot horario (08AM→07PM) |
| 🐘 **Enjaulados críticos** | Números con >5 días sin salir |
| 🔗 **Pares Markov** | Transiciones consecutivas dentro del mismo día |
| 🎲 **Baseline aleatorio** | 10,000 simulaciones de predicción aleatoria |
| 📉 **Umbral de ventaja** | `baseline + 1σ` como mínimo para publicar |

---

## 🎯 Lógica de Predicción

Los modelos reciben un prompt estructurado con jerarquía de análisis en 4 pasos:

```
0. Ancla temporal    → Identifica el último sorteo real
1. Hora exacta       → Asigna top frecuencia histórica por slot
2. Enjaulados críticos → Sustituye slots con números >5 días sin salir
3. Pares Markov      → Ajusta slot N+1 si existe transición X→Y fuerte
4. Validación final  → Verifica que supere el baseline aleatorio
```

---

## 🗄️ Estructura Supabase

Tabla: **`Lotto Activo`**

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `Fecha Prediccion` | text | Fecha del día predicho |
| `Hora Prediccion` | text | Hora en que se generó |
| `Gemma 26B` | jsonb | Secuencia de 12 sorteos `{08AM: "X", ...}` |
| `Gemma 31B` | jsonb | Secuencia de 12 sorteos |
| `Mistral` | jsonb | Secuencia de 12 sorteos |
| `Llama 3.3 70B` | jsonb | Secuencia de 12 sorteos |
| `GPT` | jsonb | Secuencia de 12 sorteos |

---

## ⚙️ Requisitos e Instalación

### APIs necesarias
- 🔵 **Google Gemini API** — para Gemma 26B y Gemma 31B
- 🟠 **Groq API** — para Llama 70B y GPT 120B
- 🟣 **Mistral Cloud API** — para Mistral Large
- 📱 **Telegram Bot Token** — para notificaciones
- 🗄️ **Supabase** — base de datos (proyecto + API key)

### Instalación en n8n

1. Clona este repositorio
2. Importa los 4 archivos `.json` en n8n (Settings → Import Workflow)
3. Configura las credenciales en n8n para cada API
4. Actualiza los IDs de workflow en el nodo `Iniciador`:
   - `Ejecutar Anailis Gemma` → ID del workflow Gemma
   - `Ejecutar Analisis Mistral/Llama` → ID del workflow Mistral/Llama
   - `Execute Workflow` → ID del workflow GPT 120B
5. Activa el workflow `Iniciador`

### Configuración de Supabase

```sql
CREATE TABLE "Lotto Activo" (
  id bigserial PRIMARY KEY,
  "Fecha Prediccion" text,
  "Hora Prediccion" text,
  "Gemma 26B" jsonb,
  "Gemma 31B" jsonb,
  "Mistral" jsonb,
  "Llama 3.3 70B" jsonb,
  "GPT" jsonb,
  created_at timestamptz DEFAULT now()
);
```

---

## ⚠️ Disclaimer

> Este sistema es un **experimento de automatización e IA**. Los sorteos de Lotto Activo tienen una distribución estadísticamente compatible con aleatoriedad pura (Chi² < 52.19). Una predicción aleatoria acierta en promedio **0.32 sorteos por día** de 12 posibles. El sistema incluye un baseline de comparación para medir si la IA supera o no al azar. **No usar para decisiones financieras.**

---

## 🛠️ Stack Tecnológico

![n8n](https://img.shields.io/badge/n8n-EA4B71?style=flat&logo=n8n&logoColor=white)
![Supabase](https://img.shields.io/badge/Supabase-3ECF8E?style=flat&logo=supabase&logoColor=white)
![Telegram](https://img.shields.io/badge/Telegram-2CA5E0?style=flat&logo=telegram&logoColor=white)
![Google Gemini](https://img.shields.io/badge/Google_Gemini-4285F4?style=flat&logo=google&logoColor=white)
![Groq](https://img.shields.io/badge/Groq-F55036?style=flat&logoColor=white)
![Mistral](https://img.shields.io/badge/Mistral_AI-FF7000?style=flat&logoColor=white)
