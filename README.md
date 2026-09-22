# Datos fiscales y salariales de la UE (en producción)

Este repositorio es la fuente de datos compartida para **PayCompass**, **EuroWorkCalc** y (desde
esta ampliación) la nueva app de independencia financiera. Las apps hacen fetch en vivo de
`contents/countries.json` (con caché local y una copia empaquetada de respaldo para el primer
arranque sin internet), así que actualizar este fichero actualiza las tres apps sin publicar nada
nuevo en las tiendas.

**Cadencia esperada: ~1 vez al año** (los salarios mínimos y tramos casi siempre cambian en enero,
tras los presupuestos generales de cada país). No hay cron ni GitHub Action — recordatorio de
calendario para enero.

## Flujo de actualización (manual, a propósito)

1. Lanza el prompt de abajo contra un modelo con búsqueda web real (Claude con web search o
   similar).
2. Revisa tú la respuesta a mano antes de publicar nada — con datos fiscales, un fallo de la IA
   sin revisar es peor que tardar un día más.
3. Sustituye `contents/countries.json` por el resultado (o mergea campo a campo si solo cambian
   algunos países).
4. Commit + push. Las tres apps recogen el cambio en su siguiente fetch, sin necesidad de subir
   nada a Google Play.

## Esquema actual (un objeto por país)

```
{
  "pais": "string",
  "codigo_iso": "string (ISO 3166-1 alpha-2)",
  "bandera_emoji": "string",

  "salario_minimo": {
    "valor_moneda_local": number|null, "moneda": "ISO 4217", "valor_eur": number|null,
    "periodo": "string|null (ej. mensual_12_pagas, mensual_14_pagas)",
    "fuente_url": "string|null", "fecha_dato": "YYYY-MM-DD|null",
    "fecha_tipo_cambio": "YYYY-MM-DD|null", "status": "ok|no_encontrado|conflicto"
  },

  "coste_vida_indice": {
    "valor": number|null, "base_referencia": "string (Numbeo, Nueva York = 100)",
    "fuente_url": "string|null", "fecha_dato": "YYYY-MM-DD|null", "status": "ok|no_encontrado|conflicto"
  },

  "salario_medio": {
    "valor_eur": number|null, "fuente_url": "string|null", "fecha_dato": "string (año o fecha)",
    "status": "ok|no_encontrado|conflicto"
  },

  "irpf_tramos": [ { "desde": number, "hasta": number|null, "porcentaje": number } ],
  "irpf_status": "ok|no_encontrado|conflicto",
  "irpf_fuente_url": "string|null",

  "seguridad_social": {
    "porcentaje_empleado": number, "moneda": "ISO 4217", "fuente_url": "string",
    "fecha_dato": "YYYY-MM-DD", "nota": "string opcional", "status": "ok"
  },
  "seguridad_social_status": "ok|pendiente",

  "minimo_personal": {
    "tipo": "base|cuota", "valor": number, "moneda": "EUR (SIEMPRE, ver nota de conversión)",
    "fuente_url": "string", "fecha_dato": "YYYY-MM-DD", "nota": "string opcional", "status": "ok"
  },
  "minimo_personal_status": "ok|no_aplica|pendiente",

  "tramos_ahorro": [ { "desde": number, "hasta": number|null, "porcentaje": number } ],
  "tramos_ahorro_status": "ok|no_encontrado|conflicto",
  "tramos_ahorro_fuente_url": "string|null",

  "fecha_actualizacion": "YYYY-MM-DD"
}
```

### Notas sobre campos ya existentes (por qué son como son)

- **`irpf_tramos` vs `tramos_ahorro`**: son dos impuestos DISTINTOS sobre la renta. `irpf_tramos`
  es la tarifa general (rendimientos del trabajo/salario) y la usan PayCompass y EuroWorkCalc.
  `tramos_ahorro` es la tarifa del ahorro (dividendos, intereses, plusvalías de inversión) y la usa
  la app de independencia financiera. En España, por ejemplo, son tablas con tramos y porcentajes
  distintos — nunca reutilizar una tabla para la otra.
- **Todo importe monetario de tramos/mínimos se guarda en EUR, nunca en moneda local**, incluso
  para países fuera del euro. Si conviertes un valor de moneda local a EUR, usa el MISMO tipo de
  cambio implícito que ya esté usado en `salario_minimo` para ese país (es decir,
  `salario_minimo.valor_moneda_local / salario_minimo.valor_eur`), para que todos los importes del
  país sean coherentes entre sí. Mezclar tipos de cambio de fechas distintas dentro del mismo país
  ha sido ya una fuente real de errores (ver historial de commits).
- **`minimo_personal.tipo`**: `"base"` significa que el valor se resta del salario ANTES de
  aplicar los tramos; `"cuota"` significa que se resta de la cuota ya calculada sobre el tramo
  completo (método español, art. 63 LIRPF). Aplica el mismo criterio si en el futuro se añade un
  mínimo exento equivalente para `tramos_ahorro`.
- **`status: no_encontrado` es una válvula de escape honesta**, no un error. Las apps que consumen
  este JSON tratan cualquier bloque que no sea `status: "ok"` como "no mostrar/marcar pendiente",
  nunca como 0 ni como una aproximación.

## El prompt (cúbrelo TODO de una vez, no solo salario mínimo)

```
Eres un motor de datos, no un asistente conversacional. Tu única salida es JSON válido, nada de
texto antes o después, nada de markdown, nada de ```json.

TAREA: Para cada país de la lista proporcionada, busca en la web el dato MÁS RECIENTE y OFICIAL de:
1. Salario mínimo interprofesional mensual bruto (moneda local y equivalente en EUR)
2. Salario medio anual bruto (equivalente en EUR)
3. Índice de coste de vida (referencia Numbeo, base Nueva York = 100)
4. Tramos de IRPF vigentes sobre RENDIMIENTOS DEL TRABAJO (tarifa general) - porcentaje y rango de
   ingresos por tramo, en EUR
5. Tramos de tributación del AHORRO (dividendos, intereses, plusvalías de inversión) - porcentaje
   y rango por tramo, en EUR. ESTO ES UN IMPUESTO DISTINTO al del punto 4, no reutilices la misma
   tabla aunque en algún país coincidan los porcentajes por casualidad.
6. Porcentaje de cotización a la Seguridad Social A CARGO DEL EMPLEADO (no del empleador; muchas
   fuentes solo dan el total o el del empleador, verifica cuál es antes de usarlo)
7. Mínimo personal/familiar exento de IRPF sobre el trabajo: indica si se resta de la base
   ("tipo": "base") o de la cuota ya calculada ("tipo": "cuota"), y el valor en EUR

REGLAS OBLIGATORIAS:
- Usa la herramienta de búsqueda web para CADA país, CADA dato. No respondas de memoria.
- Prioriza fuentes oficiales: ministerio de trabajo, agencia tributaria, seguridad social,
  Eurostat, OCDE, Numbeo.
- Si encuentras el dato, inclúyelo con su fuente (URL) y fecha de publicación.
- Si NO encuentras un dato fiable tras buscar, NO inventes ni aproximes. Pon el campo como null y
  "status": "no_encontrado" en ese bloque. Esto es preferible a un dato incorrecto.
- Si el dato encontrado es ambiguo o hay fuentes que se contradicen, pon "status": "conflicto".
- No redondees ni "estimes" cifras. Si el valor exacto no aparece en la fuente, es no_encontrado.
- No te niegues a responder por ningún país de la lista, ni des explicaciones fuera del JSON.
- TODOS los importes de tramos y mínimos van en EUR, incluso para países fuera del euro. Si
  conviertes de moneda local, usa el MISMO tipo de cambio implícito que uses para el salario
  mínimo de ese mismo país (no mezcles fechas de cambio distintas dentro de un mismo país).
- Incluye la fecha del tipo de cambio usado (fecha_tipo_cambio), no solo la fecha del dato.
- Para la cotización a la Seguridad Social, verifica explícitamente que el porcentaje que
  reportas es el del EMPLEADO, no el del empleador ni el total combinado.
- Para el mínimo personal, si el mecanismo real del país es una reducción progresiva o una
  fórmula compleja (no un valor fijo), usa el valor más representativo para un contribuyente
  soltero sin descendientes y dilo explícitamente en el campo "nota", en vez de fingir precisión
  que no existe.
- Si un país no tiene un impuesto separado sobre el ahorro (todo tributa igual que el trabajo),
  pon "tramos_ahorro_status": "no_aplica" y deja "tramos_ahorro" como array vacío.

FORMATO DE SALIDA (JSON estricto, un array con un objeto por país, exactamente con esta forma):

[
  {
    "pais": "string",
    "codigo_iso": "string (ISO 3166-1 alpha-2)",
    "bandera_emoji": "string (emoji de bandera)",
    "salario_minimo": {
      "valor_moneda_local": number|null, "moneda": "string (ISO 4217)", "valor_eur": number|null,
      "periodo": "string|null", "fuente_url": "string|null", "fecha_dato": "YYYY-MM-DD|null",
      "fecha_tipo_cambio": "YYYY-MM-DD|null", "status": "ok|no_encontrado|conflicto"
    },
    "salario_medio": {
      "valor_eur": number|null, "fuente_url": "string|null", "fecha_dato": "string|null",
      "status": "ok|no_encontrado|conflicto"
    },
    "coste_vida_indice": {
      "valor": number|null, "base_referencia": "string", "fuente_url": "string|null",
      "fecha_dato": "YYYY-MM-DD|null", "status": "ok|no_encontrado|conflicto"
    },
    "irpf_tramos": [ { "desde": number, "hasta": number|null, "porcentaje": number } ],
    "irpf_status": "ok|no_encontrado|conflicto",
    "irpf_fuente_url": "string|null",
    "tramos_ahorro": [ { "desde": number, "hasta": number|null, "porcentaje": number } ],
    "tramos_ahorro_status": "ok|no_encontrado|conflicto|no_aplica",
    "tramos_ahorro_fuente_url": "string|null",
    "seguridad_social": {
      "porcentaje_empleado": number|null, "moneda": "EUR", "fuente_url": "string|null",
      "fecha_dato": "YYYY-MM-DD|null", "nota": "string opcional", "status": "ok|no_encontrado"
    },
    "seguridad_social_status": "ok|pendiente",
    "minimo_personal": {
      "tipo": "base|cuota", "valor": number|null, "moneda": "EUR", "fuente_url": "string|null",
      "fecha_dato": "YYYY-MM-DD|null", "nota": "string opcional", "status": "ok|no_encontrado"
    },
    "minimo_personal_status": "ok|no_aplica|pendiente",
    "fecha_actualizacion": "YYYY-MM-DD"
  }
]

LISTA DE PAÍSES: [pega aquí la lista completa de países que quieras cubrir]

Devuelve un array JSON con un objeto por país. Nada más.
```

### Por qué está diseñado así

- **Un único prompt para todo** en vez de uno por dato: así una sola pasada anual mantiene las
  tres apps a la vez, sin arrastrar campos desincronizados entre ellas.
- **`status: no_encontrado`/`no_aplica` como válvulas de escape honestas** — sin esto, cualquier
  LLM tiende a rellenar huecos con estimaciones "para ayudar". Con la opción explícita, prefiere
  marcar el hueco a inventar un número.
- **Exigir `fuente_url`** obliga en la práctica a que el modelo use búsqueda real en vez de tirar
  de memoria — si no busca, no tiene URL real que poner, y eso es detectable al revisar a mano.
- **`tramos_ahorro` separado de `irpf_tramos`** — son impuestos distintos con tablas distintas; ya
  hubo que revertir una confusión de este tipo entre tramos de IRPF y conversión de divisa (ver
  historial), así que el prompt lo remarca explícitamente para no repetir el error con el ahorro.

## Consumidores de este repositorio

| App | Usa de este JSON |
|---|---|
| PayCompass | `salario_minimo`, `salario_medio`, `coste_vida_indice`, `irpf_tramos` |
| EuroWorkCalc | `irpf_tramos`, `seguridad_social`, `minimo_personal` |
| App de independencia financiera (FIRE) | `tramos_ahorro`, `coste_vida_indice` |

Si se añade una app nueva que consuma un dato distinto de este mismo JSON, añade también su fila
aquí y su(s) campo(s) al esquema y al prompt de arriba — la idea es que este documento sea SIEMPRE
el reflejo exacto de qué pide el prompt anual, no una versión antigua de él.
