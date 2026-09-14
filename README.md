# Fase 2 — Datos reales (pendiente, no implementado)

Este documento recoge el plan para sustituir el JSON de prueba (`public/assets/data/countries.json`)
por datos reales, y la utilidad manual de publicación. **Nada de esto está construido todavía** —
es la referencia para cuando se retome.

## Resumen del flujo (manual, no automático)

1. Lanzas el prompt de abajo contra un modelo con búsqueda web (Claude con web search, o similar).
2. Revisas tú la respuesta (a mano) antes de publicar nada — con datos fiscales/salariales, un
   fallo de la IA sin revisar es peor que tardar un día más en actualizar.
3. Pegas el JSON resultante en la utilidad de publicación (aún sin construir — ver más abajo) o,
   mientras no exista, sustituyes directamente `public/assets/data/countries.json` a mano.
4. Commit + push al repo de datos (`C:\Projects\datapaycompass`).

**Cadencia esperada: ~1 vez al año** (los salarios mínimos casi siempre cambian en enero). No hace
falta ningún cron ni GitHub Action — ponte tú un recordatorio de calendario para enero.

## El prompt

```
Eres un motor de datos, no un asistente conversacional. Tu única salida es JSON válido, nada de
texto antes o después, nada de markdown, nada de ```json.

TAREA: Para cada país de la lista proporcionada, busca en la web el dato MÁS RECIENTE y OFICIAL de:
1. Salario mínimo interprofesional mensual bruto (en moneda local y su equivalente en EUR)
2. Tramos de IRPF vigentes (porcentaje y rango de ingresos por tramo)
3. Índice de coste de vida (referencia Numbeo, base Nueva York = 100)

REGLAS OBLIGATORIAS:
- Usa la herramienta de búsqueda web para CADA país, CADA dato. No respondas de memoria.
- Prioriza fuentes oficiales: ministerio de trabajo, agencia tributaria, Eurostat, OCDE, Numbeo.
- Si encuentras el dato, inclúyelo con su fuente (URL) y fecha de publicación.
- Si NO encuentras un dato fiable tras buscar, NO inventes ni aproximes. Pon el campo como null y
  "status": "no_encontrado" en ese bloque. Esto es preferible a un dato incorrecto.
- Si el dato encontrado es ambiguo o hay fuentes que se contradicen, pon "status": "conflicto".
- No redondees ni "estimes" cifras. Si el valor exacto no aparece en la fuente, es no_encontrado.
- No te niegues a responder por ningún país de la lista, ni des explicaciones fuera del JSON.
- Incluye también la fecha del tipo de cambio usado para valor_eur (fecha_tipo_cambio), no solo
  la fecha del dato de salario -si son de fechas distintas, la conversión puede ser inconsistente
  y nadie lo detecta hasta que alguien pregunta por qué no cuadra-.

FORMATO DE SALIDA (JSON estricto, un array con un objeto por país -ver schema.json de este
mismo proyecto, en docs/, para la forma exacta de cada campo):

[
  {
    "pais": "string",
    "codigo_iso": "string (ISO 3166-1 alpha-2)",
    "bandera_emoji": "string (emoji de bandera)",
    "salario_minimo": {
      "valor_moneda_local": number|null,
      "moneda": "string (ISO 4217)",
      "valor_eur": number|null,
      "periodo": "string|null (ej. mensual_12_pagas, mensual_14_pagas)",
      "fuente_url": "string|null",
      "fecha_dato": "YYYY-MM-DD|null",
      "fecha_tipo_cambio": "YYYY-MM-DD|null",
      "status": "ok|no_encontrado|conflicto"
    },
    "coste_vida_indice": {
      "valor": number|null,
      "base_referencia": "string",
      "fuente_url": "string|null",
      "fecha_dato": "YYYY-MM-DD|null",
      "status": "ok|no_encontrado|conflicto"
    },
    "irpf_tramos": [
      { "desde": number, "hasta": number|null, "porcentaje": number }
    ],
    "irpf_status": "ok|no_encontrado|conflicto",
    "irpf_fuente_url": "string|null",
    "fecha_actualizacion": "YYYY-MM-DD"
  }
]

LISTA DE PAÍSES: [pega aquí la lista completa de países de la UE que quieras cubrir]

Devuelve un array JSON con un objeto por país. Nada más.
```

### Por qué está diseñado así

- **`status: no_encontrado` como válvula de escape honesta** — sin eso, cualquier LLM tiende a
  rellenar huecos con estimaciones "para ayudar". Con esta opción explícita, prefiere marcar el
  hueco a inventar un número.
- **Exigir `fuente_url`** obliga en la práctica a que el modelo use búsqueda real en vez de tirar
  de memoria — si no busca, no tiene URL real que poner, y eso es detectable al revisar a mano.
- **JSON estricto sin campos opcionales ambiguos** — hace que cualquier validación automática
  futura (ver más abajo) sea sencilla: si falta un campo o el `status` no es uno de los 3 valores
  válidos, se puede rechazar el fichero entero en vez de publicarlo a medias.
- **`fecha_tipo_cambio` separada de `fecha_dato`** — evita inconsistencias silenciosas cuando el
  salario es de una fecha y el cambio de divisa se calculó en otra.

## La utilidad de publicación (sin construir)

Idea: una pantalla sencilla (dentro de la propia app, o una página web aparte tipo herramienta
interna) donde:

1. Pegas el JSON completo que te devolvió el prompt.
2. La utilidad valida que:
   - Es un array.
   - Cada país tiene todos los campos obligatorios.
   - Cada `status` es uno de `ok`/`no_encontrado`/`conflicto` -nunca otro valor ni vacío-.
   - Los países con `status: ok` en `salario_minimo` tienen `valor_eur` no nulo (si dice "ok" pero
     el valor es null, es una inconsistencia y debería bloquear la publicación).
3. Si la validación pasa, hace `git commit` + `git push` al repo `datapaycompass`, sustituyendo
   `countries.json`.
4. Si falla, muestra qué país/campo exactamente falló, sin publicar nada.

No hace falta que sea elaborada -un script de Node que leas y ejecutes a mano vale igual que una
pantalla bonita-, lo importante es la validación antes de publicar, no la interfaz.

## Cuándo ampliar de "salario mínimo" a más datos

Empezar solo con salario mínimo + coste de vida (dato simple, escrapeable, cambia poco) para todos
los países que se quiera cubrir. Los tramos de IRPF completos (reglas fiscales complejas, cambian
cada año, 27 sistemas distintos en la UE) añadirlos solo para los 3-4 países con más tráfico
cruzado real una vez haya datos de uso de la app -no antes-, para no comprometerse a mantener
27 sistemas fiscales a mano sin saber si hay demanda real.
