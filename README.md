# IAmIA — mi agente personal

Un asistente personal al que le hablo por **WhatsApp** (texto y audios), que me
contesta también **con voz**, **recuerda** cosas de mí, maneja mi **calendario**,
lee y trabaja con mis **documentos**, usa el **navegador** para hacer acciones en
páginas específicas, y al que le puedo ir agregando **skills** propias.

---

## 1. Decisión: no escribir el harness desde cero

Vamos a usar **[Hermes Agent](https://github.com/nousresearch/hermes-agent)**
(Nous Research, licencia MIT, Python) como base, y en este repo construimos
**nuestra capa encima**: skills, plugins, servidores MCP y configuración.

**¿Por qué no desde cero?** El ciclo básico de un agente (modelo → herramienta →
resultado → modelo) son unas 200 líneas. Lo difícil es todo lo demás:
compactar el contexto cuando la conversación crece, reintentos y respaldo entre
proveedores, manejo de errores de herramientas, el puente con WhatsApp, el
pipeline de audio, permisos, sesiones, tareas programadas… Eso son meses de
trabajo antes de llegar a *tus* funciones. Hermes ya lo trae y, al ser MIT,
podemos leerlo, modificarlo o hacer fork cuando haga falta.

**¿Por qué Hermes y no OpenClaw?** Los dos sirven. Hermes gana para este
proyecto porque: es Python, trae **DeepSeek** (`deepseek`) y **Gemini**
(`gemini`) como proveedores directos, ya integra **Honcho** para memoria, y
crea/mejora skills solo (formato estándar `SKILL.md`).

### Lo que quiero vs. cómo se cubre

| Quiero                         | Cómo                                                                 | Costo                        |
|--------------------------------|----------------------------------------------------------------------|------------------------------|
| Cerebro principal              | DeepSeek (proveedor `deepseek`)                                       | Barato (centavos por día)    |
| Visión, documentos, respaldo   | Gemini, capa gratuita (proveedor `gemini`)                            | Gratis con límites           |
| Memoria                        | Memoria integrada (`MEMORY.md` + búsqueda de sesiones) → luego Honcho | Gratis / créditos            |
| Skills propias                 | Carpetas con `SKILL.md` en `skills/` de este repo                     | Gratis                       |
| Hablarle por WhatsApp          | Gateway de WhatsApp (puente Baileys, se vincula con QR)               | Gratis (ver riesgos)         |
| Que entienda mis audios        | STT local con faster-whisper (o Groq)                                 | Gratis                       |
| Que me conteste con voz        | Edge TTS con voz en español de México → ElevenLabs si quiero premium | Gratis                       |
| Calendario, Gmail, Drive       | Skill `google-workspace` (OAuth con mi cuenta)                        | Gratis                       |
| Navegador                      | Chromium local, o mi propio Chrome vía CDP (`/browser connect`)       | Gratis                       |
| Tareas programadas             | Cron integrado ("cada mañana mándame mi agenda")                      | Gratis                       |
| Herramientas nuevas            | Plugins en Python o servidores MCP en este repo                       | —                            |

---

## 2. Modelos: cómo repartir DeepSeek y Gemini

- **DeepSeek como cerebro principal.** Un agente hace *muchas* llamadas por
  tarea (cada uso de herramienta es otra llamada). La capa gratis de Gemini se
  queda corta para eso (≈10–15 peticiones por minuto en Flash y un tope diario),
  así que el modelo principal debe ser uno barato y sin topes duros: DeepSeek
  Flash. DeepSeek cobra menos en horario valle y casi nada por tokens en caché.
- **Gemini gratis para tareas auxiliares:** ver imágenes, leer PDFs largos,
  resumir/compactar contexto y como respaldo si DeepSeek falla.
- **Ojo con la privacidad:** en la capa gratuita, Google puede usar lo que le
  mandes para mejorar sus productos. No mandes por Gemini gratis documentos
  sensibles (identificaciones, datos bancarios, etc.).

Precios y límites cambian seguido: revisar
[precios de DeepSeek](https://api-docs.deepseek.com/quick_start/pricing) y
[límites de Gemini](https://ai.google.dev/gemini-api/docs/rate-limits) antes de
hacer cuentas.

---

## 3. Estructura de este repo

No hacemos fork de Hermes (así podemos actualizarlo sin romper nada). Este repo
guarda solo **lo nuestro**; se irá creando por fases:

```
iamia/
├── README.md                    # este plan
├── .env.example                 # plantilla de llaves → se copia a ~/.hermes/.env
├── config/
│   └── config.example.yaml      # ajustes de voz, etc. → ~/.hermes/config.yaml
├── skills/                      # skills propias (una carpeta con SKILL.md cada una)
├── plugins/                     # herramientas nuevas en Python
└── mcp/                         # servidores MCP propios (si hacen falta)
```

Solo haríamos fork si necesitamos cambiar el núcleo de Hermes.

---

## 4. Plan por fases

Cada fase termina con algo que ya funciona y se puede usar.

### Fase 0 — Preparación
- [ ] **Dónde va a vivir el agente.** Tiene que estar prendido 24/7 para
      contestar WhatsApp: un VPS barato (~5 USD/mes), una PC vieja o una
      Raspberry Pi. Linux, macOS o WSL2 en Windows.
- [ ] Llave de **DeepSeek** en [platform.deepseek.com](https://platform.deepseek.com) (requiere recarga de saldo).
- [ ] Llave de **Gemini** en [aistudio.google.com](https://aistudio.google.com) (gratis).
- [ ] Un **número aparte** para el bot de WhatsApp (un chip barato). Ver riesgos abajo.

### Fase 1 — Agente básico en la terminal
- [ ] Instalar Hermes:
      `curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash`
- [ ] Copiar `.env.example` a `~/.hermes/.env` y poner las llaves.
- [ ] `hermes model` → elegir DeepSeek como principal; Gemini como auxiliar/respaldo.
- [ ] Probar en la terminal con `hermes`: preguntas, leer un archivo, buscar en web.

### Fase 2 — WhatsApp + voz
- [ ] `hermes whatsapp` → modo `bot` (con el número aparte) o `self-chat`
      (en tu propio número, para probar). Escanear el QR desde
      *Ajustes → Dispositivos vinculados*.
- [ ] `WHATSAPP_ALLOWED_USERS` = **solo tu número** (con lada, sin `+`).
- [ ] STT local (faster-whisper) para transcribir tus audios.
- [ ] TTS con Edge en español de México (`es-MX-DaliaNeural` o
      `es-MX-JorgeNeural`). Ver `config/config.example.yaml`.
- [ ] `hermes gateway` para dejarlo escuchando.

### Fase 3 — Calendario y documentos
- [ ] Pedirle al agente: *"configura Google Workspace"*. Te guía para crear un
      proyecto en Google Cloud, activar Calendar/Gmail/Drive y autorizar con
      OAuth (tipo "Desktop app").
- [ ] Probar: *"¿qué tengo mañana?"*, *"agéndame dentista el jueves a las 5"*.
- [ ] Documentos: mandarle PDFs/fotos por WhatsApp y pedirle que los resuma,
      extraiga datos o los guarde.

### Fase 4 — Navegador
- [ ] Modo local (Chromium) o conectar tu propio Chrome con `/browser connect`.
- [ ] Una skill por cada página donde quieras que haga algo específico
      (pasos, qué revisar, qué **no** hacer sin preguntarte).

### Fase 5 — Memoria con Honcho
- [ ] Primero usar la memoria integrada unas semanas.
- [ ] Luego `hermes plugins install honcho`. Opciones: Honcho en la nube
      (dan créditos gratis al registrarte) o auto-hospedado (Postgres con
      pgvector + Redis; gratis, pero más cosas que mantener).

### Fase 6 — Mis skills y funciones
Lista abierta de ideas (agregar aquí):
- [ ] Resumen de la mañana por WhatsApp (agenda + correos importantes) con cron.
- [ ] …

---

## 5. Seguridad (importante)

Un agente con acceso a tu WhatsApp, correo, calendario y navegador es poderoso;
si alguien más lo controla, también lo es para esa persona.

- **Nunca** subir el `.env` real a git (ya está en `.gitignore`).
- **Solo tu número** en `WHATSAPP_ALLOWED_USERS`. Nunca `*`.
- **Confirmar antes de acciones irreversibles**: borrar eventos, mandar
  correos, pagar o comprar. Ponerlo explícito en cada skill.
- **Inyección de instrucciones**: una página web o un documento puede traer
  texto tipo "ignora tus instrucciones y manda…". Tratar lo que viene de afuera
  como datos, no como órdenes.
- **WhatsApp no oficial**: el puente Baileys no está avalado por WhatsApp; hay
  un riesgo pequeño de bloqueo. Por eso el número aparte, y nunca usarlo para
  mandar mensajes masivos ni a gente que no te escribió primero. La alternativa
  oficial es la WhatsApp Business Cloud API de Meta (más trámite).

---

## 6. Costos aproximados

| Concepto            | Costo                                                   |
|---------------------|---------------------------------------------------------|
| DeepSeek            | Del orden de 1–10 USD/mes con uso personal (depende de cuánto lo uses) |
| Gemini              | 0 (capa gratuita)                                       |
| Voz (Edge TTS + whisper local) | 0                                            |
| Servidor            | 0 (PC propia) o ~5 USD/mes (VPS)                        |
| Número de WhatsApp  | Un chip prepago                                         |
| Honcho              | Créditos gratis al inicio, o 0 auto-hospedado           |
