
# Shopify Support Multi‑Agent Workflow

Este repositorio contiene el código de un *workflow* asíncrono que orquesta varios agentes especializados para dar soporte a un ecommerce de Shopify, integrando además un conector de Gmail vía MCP (Model Context Protocol).

El objetivo es:
- Tomar un mensaje libre del usuario.
- Clasificarlo en una de varias **categorías de soporte**.
- Enrutar la conversación al agente especializado adecuado.
- Usar herramientas MCP de Shopify y Gmail *solo* para lectura/consulta de información relevante (carrito, catálogo, políticas, correos de soporte, etc.).

---

## 1. Estructura general

Archivo principal (ejemplo): `workflow.py`

Componentes clave:

- **Herramientas MCP (`HostedMCPTool`)**
  - `mcp`, `mcp1`, `mcp3`, `mcp4`: conectan con el servidor MCP de Shopify (`https://mystore.store/api/mcp`) con distintos subconjuntos de herramientas habilitadas.
  - `mcp2`: conecta con el servidor MCP de Gmail usando un `connector_id` (por ejemplo `connector_gmail`) y un *access token* de OAuth.

- **Agente de clasificación (`classify`)**
  - Modelo: `gpt-4.1`
  - Esquema de salida: `ClassifySchema` (`category: str`)
  - Categorías posibles:
    - `OrderPlacementStatus`
    - `ShippingDelivery`
    - `ReturnsCancellationsExchanges`
    - `PaymentBilling`
    - `ProductInfoAvailability`
    - `TechnicalIssues`
    - `PromotionsDiscountsPricing`
    - `CustomerComplaintsSatisfaction`
    - `AccountProfileOther`
  - El *prompt* obliga a devolver exactamente una categoría en formato JSON.

- **Agentes especializados de dominio**
  - `OrderPlacementStatus`
  - `ShippingDelivery`
  - `ReturnsCancellationsExchanges`
  - `PaymentBilling`
  - `ProductInfoAvailability`
  - `TechnicalIssues`
  - `PromotionsDiscountsPricing`
  - `CustomerComplaintsSatisfaction`
  - `AccountProfileOther`
  - `Fallback Agent` (para casos fuera de dominio)

Cada agente tiene:
- `name`
- `instructions` (rol, alcance y restricciones)
- `model` (p.ej. `gpt-4.1`)
- `tools` (lista de `HostedMCPTool` permitidos, si aplica)
- `model_settings` (temperatura, `top_p`, `max_tokens`, etc.)

---

## 2. Flujo del workflow

La entrada principal es el modelo Pydantic:

```python
class WorkflowInput(BaseModel):
    input_as_text: str
```

La función asíncrona de entrada:

```python
async def run_workflow(workflow_input: WorkflowInput):
    ...
```

Resumen del flujo:

1. **Inicialización de estado y conversación**
   - Se crea `state = {}` (por si se requiere estado adicional).
   - Se arma `conversation_history` con un único turno del usuario usando `input_as_text`.

2. **Clasificación**
   - Se ejecuta `Runner.run(classify, ...)` pasando el texto de entrada.
   - El resultado se guarda en `classify_result_temp`.
   - Se construye `classify_result` con:
     - `output_text`: JSON crudo.
     - `output_parsed`: instancia de `ClassifySchema` serializada a dict.
   - Se extrae `classify_category = classify_result["output_parsed"]["category"]`.

3. **Enrutamiento por categoría**
   - Según `classify_category` se entra en un gran bloque de `if / elif`:
     - `OrderPlacementStatus`
     - `ShippingDelivery`
     - `ReturnsCancellationsExchanges`
     - `PaymentBilling`
     - `ProductInfoAvailability`
     - `TechnicalIssues`
     - `PromotionsDiscountsPricing`
     - `CustomerComplaintsSatisfaction`
     - (cualquier otro caso → `fallback_agent`)
   - Dentro de cada bloque se vuelve a leer `classify_result["category"]` (equivalente) y se llama al agente correspondiente con:
     ```python
     result_temp = await Runner.run(
         some_agent,
         input=[*conversation_history],
         run_config=RunConfig(trace_metadata={...})
     )
     ```
   - Se actualiza `conversation_history` añadiendo los nuevos items:
     ```python
     conversation_history.extend([item.to_input_item() for item in result_temp.new_items])
     ```
   - Se expone un diccionario con `output_text` usando:
     ```python
     some_result = {
         "output_text": result_temp.final_output_as(str)
     }
     ```

4. **Trazas**
   - El workflow completo se ejecuta dentro de:
     ```python
     with trace("Shopify Support – Multi-agent"):
         ...
     ```
   - `RunConfig` añade metadatos como `workflow_id` y `__trace_source__` para facilitar el debugging.

> **Nota:** La lógica del árbol de `if/elif` es redundante (se repite el chequeo de categoría dentro de cada caso). Esto es funcional pero puede simplificarse en el futuro con un mapa de categorías → agente.

---

## 3. Herramientas MCP configuradas

### 3.1. Shopify MCP

Se definen varias instancias de `HostedMCPTool` apuntando a:

```python
"server_label": "My Store",
"server_url": "https://mystore.store/api/mcp"
```

Cada instancia habilita diferentes herramientas:

- `mcp`
  - `get_cart`
  - `search_shop_policies_and_faqs`

- `mcp1`
  - `search_shop_catalog`
  - `search_shop_policies_and_faqs`
  - `get_product_details`

- `mcp3`
  - `search_shop_catalog`
  - `get_cart`
  - `search_shop_policies_and_faqs`

- `mcp4`
  - `search_shop_catalog`
  - `search_shop_policies_and_faqs`

Todas requieren aprobación explícita:

```python
"require_approval": "always"
```

### 3.2. Gmail MCP

```python
mcp2 = HostedMCPTool(tool_config={
    "type": "mcp",
    "server_label": "gmail",
    "connector_id": "connector_gmail",
    "authorization": "<ACCESS_TOKEN_DE_GMAIL>",
    "allowed_tools": [
        "batch_read_email",
        "get_profile",
        "get_recent_emails",
        "read_email",
        "search_email_ids",
        "search_emails"
    ],
    "require_approval": "always"
})
```

> ⚠️ **Seguridad:** en producción no se recomienda *hardcodear* el `authorization` (access token) en el código. Lo ideal es:
>
> - Guardar el token en una variable de entorno (por ejemplo `GMAIL_MCP_ACCESS_TOKEN`).
> - Implementar un flujo de OAuth que refresque el token cuando expire.
> - Evitar subir tokens reales a repositorios públicos.

---

## 4. Agentes de dominio y responsabilidades

A modo de referencia rápida:

| Agente                          | Categoría objetivo                     | Herramientas MCP                       | Uso principal                                                                                                  |
|---------------------------------|----------------------------------------|----------------------------------------|----------------------------------------------------------------------------------------------------------------|
| `OrderPlacementStatus`         | `OrderPlacementStatus`                 | `mcp` (Shopify)                        | Revisar carrito, políticas de creación/confirmación de pedidos. No modifica pedidos.                           |
| `ShippingDelivery`             | `ShippingDelivery`                     | `mcp3` (Shopify)                       | Consultar políticas de envíos, tiempos de entrega, países, tracking, etc.                                     |
| `ReturnsCancellationsExchanges`| `ReturnsCancellationsExchanges`        | `mcp1` (Shopify)                       | Explicar reglas de devoluciones/cambios/cancelaciones y pasos a seguir.                                       |
| `PaymentBilling`               | `PaymentBilling`                       | `mcp4` (Shopify)                       | Preguntas de métodos de pago, reembolsos, cargos duplicados, facturas, etc.                                   |
| `ProductInfoAvailability`      | `ProductInfoAvailability`              | `mcp1` (Shopify)                       | Buscar productos, variantes, tallas, disponibilidad, materiales, sostenibilidad, etc.                         |
| `TechnicalIssues`              | `TechnicalIssues`                      | `mcp2` (Gmail)                         | Soporte técnico (login, checkout, errores). Puede leer correos de soporte para contexto adicional.            |
| `PromotionsDiscountsPricing`   | `PromotionsDiscountsPricing`           | `mcp3` (Shopify)                       | Precios, descuentos, promociones y su aplicación en el carrito.                                               |
| `CustomerComplaintsSatisfaction`| `CustomerComplaintsSatisfaction`      | `mcp1` (Shopify) + `mcp2` (Gmail)      | Quejas sobre producto/servicio. Lee políticas y correos previos para proponer soluciones alineadas.          |
| `AccountProfileOther`          | `AccountProfileOther`                  | —                                      | Temas de cuenta, datos personales, sostenibilidad, otras consultas generales.                                 |
| `Fallback Agent`               | Cualquier otro caso                    | —                                      | Manejo genérico cuando la categoría no encaja claramente en las anteriores.                                   |

---

## 5. Requisitos

### 5.1. Versión de Python

- Python 3.10+ recomendado (por uso de `list[TResponseInputItem]` y `async/await`).

### 5.2. Dependencias

Al menos:

- `pydantic` (v2+ si se usa `BaseModel` actual).
- Paquete o módulo que exponga:
  - `HostedMCPTool`
  - `Agent`
  - `ModelSettings`
  - `TResponseInputItem`
  - `Runner`
  - `RunConfig`
  - `trace`

Ejemplo de instalación genérica:

```bash
pip install pydantic
# + las dependencias específicas de tu entorno de Agents/MCP
```

En un proyecto real se recomienda mantener un `requirements.txt` o `pyproject.toml`.

---

## 6. Configuración

### 6.1. Variables de entorno recomendadas

Aunque el ejemplo incrusta URLs y tokens, una configuración más segura podría ser:

```bash
# .env (ejemplo)
STORE_MCP_URL=https://mystore.store/api/mcp
STORE_MCP_LABEL=mystore

GMAIL_MCP_CONNECTOR_ID=connector_gmail
GMAIL_MCP_ACCESS_TOKEN=ya29....   # token de acceso, renovado por tu flujo de OAuth
```

Y en el código:

```python
import os

STORE_MCP_URL = os.getenv("STORE_MCP_URL", "https://mystore.store/api/mcp")
GMAIL_MCP_ACCESS_TOKEN = os.getenv("GMAIL_MCP_ACCESS_TOKEN", "")
```

---

## 7. Cómo ejecutar el workflow

### 7.1. Dentro de un entorno de Agent Builder / Orchestrator

Si este archivo se usa dentro de un orquestador que ya conoce `run_workflow` como entrypoint:

1. Registrar el archivo como módulo de workflow.
2. Configurar el modelo de entrada como `WorkflowInput`.
3. Enviar peticiones con:
   ```json
   {
     "input_as_text": "tu mensaje de usuario aquí"
   }
   ```
4. El resultado final será el texto generado por el agente especializado llamado tras la clasificación.

### 7.2. Ejecución manual (ejemplo con asyncio)

Si quieres probarlo manualmente desde Python (y tienes las dependencias configuradas):

```python
import asyncio
from workflow import run_workflow, WorkflowInput

async def main():
    wf_input = WorkflowInput(input_as_text="My order hasn't arrived yet, can you check the tracking?")
    await run_workflow(wf_input)

if __name__ == "__main__":
    asyncio.run(main())
```

> Nota: la forma exacta de obtener el resultado dependerá de cómo tu entorno maneje el valor de retorno y las trazas del `Runner`. El ejemplo del código se centra en la orquestación interna, no en el formato de respuesta HTTP/CLI.

---

## 8. Notas y posibles mejoras

- **Reducción de duplicación:** El bloque gigante de `if/elif` puede reemplazarse por un mapa:
  ```python
  CATEGORY_TO_AGENT = {
      "OrderPlacementStatus": orderplacementstatus,
      "ShippingDelivery": shippingdelivery,
      ...
  }
  ```
  Lo que simplificaría el enrutamiento.

- **Gestión de tokens:** Externalizar los tokens (especialmente el de Gmail) a variables de entorno y añadir un flujo de refresco.

- **Logging estructurado:** Aprovechar `trace` y `RunConfig` para registrar de forma más explícita errores y tiempos de respuesta.

- **Tests:** Agregar pruebas unitarias y de integración:
  - Casos básicos por categoría.
  - Simulación de respuestas de Shopify MCP y Gmail MCP.

---

## 9. Licencia y uso

Adapta esta sección a las necesidades de tu proyecto:

```text
Este código está destinado a uso interno para el soporte de mystore.store.
Antes de reutilizarlo en otros proyectos o compartirlo públicamente,
asegúrate de eliminar credenciales sensibles y revisar tus obligaciones legales.
```

---
