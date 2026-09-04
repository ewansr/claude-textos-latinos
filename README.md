# claude-textos-latinos
# textos-espanol

**Capitalización, acentos y codificación para sistemas en español.**

Una skill para Claude — y dos módulos sin dependencias que puedes usar aunque no uses Claude — que corrigen los defectos de texto que hacen que un sistema en español parezca traducido del inglés.

---

## El problema

Tres cosas que se ven:

```
Nuevo Pedido De Compra          →  Nuevo pedido de compra
FACTURACION                     →  Facturación
TRANSPORTES DEL GOLFO SA DE CV  →  Transportes del Golfo SA de CV
```

Y tres que no se ven pero rompen cosas:

- **Mojibake.** Un CSV exportado desde Excel llega como `MÃ©xico` y así se guarda.
- **Unicode NFD.** macOS manda la `ñ` descompuesta en dos code points. Se ve idéntica en pantalla, pero `'muñoz' !== 'muñoz'` y la deduplicación falla en silencio.
- **Búsqueda con acentos.** El usuario teclea `munoz` y el sistema no encuentra `Muñoz`.

El español no usa el Title Case del inglés: en títulos, botones y encabezados solo va mayúscula la primera letra y los nombres propios. Y los acentos en mayúscula son obligatorios — la RAE nunca dictó lo contrario, era una limitación de las máquinas de escribir.

---

## Instalación

### Como skill de Claude

**Claude web, escritorio o móvil.** Descarga `textos-espanol.skill` de la sección de releases, súbelo a una conversación y presiona **Save skill**. Claude la usa sola cuando el tema aplica.

**Claude Code.** Clona el repositorio donde Claude busca las skills:

```bash
# Disponible en todos tus proyectos
git clone https://github.com/USUARIO/textos-espanol.git ~/.claude/skills/textos-espanol

# O dentro de un proyecto, para que todo el equipo la tenga al clonar
git clone https://github.com/USUARIO/textos-espanol.git .claude/skills/textos-espanol
```

### Como librería, sin Claude

Los módulos de `scripts/` no tienen dependencias. Cópialos y ya:

```bash
cp scripts/texto.js  src/lib/texto.js      # Node 16+ o navegador, ESM
cp scripts/texto.py  app/utils/texto.py    # Python 3.8+
```

---

## Uso

```js
import { limpiar, nombrePropio, sentencia, claveBusqueda, rfc, repararMojibake } from './texto.js';

limpiar('  Juan\u00A0 Pérez ')            // 'Juan Pérez'
nombrePropio('JUAN DE LA CRUZ PEREZ')     // 'Juan de la Cruz Perez'
nombrePropio("o'brien mcdonald")          // "O'Brien McDonald"
sentencia('Movimientos No Facturados')    // 'Movimientos no facturados'
sentencia('NUEVO PEDIDO DE COMPRA')       // 'Nuevo pedido de compra'
claveBusqueda('Muñoz Peña')               // 'munoz pena'
rfc(' lva240612-g34 ')                    // 'LVA240612G34'
repararMojibake('MÃ©xico')                // 'México'
```

Python: misma API en `snake_case`, mismos resultados.

```python
from texto import nombre_propio, sentencia, clave_busqueda

nombre_propio('MARIA DEL CARMEN MUÑOZ')   # 'Maria del Carmen Muñoz'
sentencia('Estado De Cuenta')             # 'Estado de cuenta'
```

### Funciones

| Función | Para qué |
|---|---|
| `limpiar` | NFC, recorte, espacios colapsados, invisibles fuera. Aplícala a **toda** cadena que entre al sistema |
| `limpiarMultilinea` | Igual, pero conserva saltos de línea |
| `mayusculas` / `minusculas` | Con acentos y ñ intactos, locale fijo |
| `nombrePropio` | Personas. Partículas en minúscula: `Juan de la Cruz` |
| `nombreCatalogo` | Lugares, calles, razones sociales, catálogos |
| `sentencia` | Títulos, botones, encabezados y mensajes |
| `claveBusqueda` | Sin acentos ni puntuación, para columna indexada |
| `slug` | Igual, con guiones, para URLs |
| `rfc` `curp` `placa` `vin` `contenedor` | Mayúsculas, solo alfanuméricos |
| `correo` `telefono` | Forma canónica |
| `esMojibake` / `repararMojibake` | Detección y reparación de doble codificación |
| `tipografia` | Comillas angulares, raya, puntos suspensivos. Solo para documentos |
| `faltanSignosApertura` | Encuentra las frases sin `¿` o `¡` |

---

## Lo que NO hace, a propósito

**No inventa acentos.** `nombrePropio('PEREZ')` devuelve `Perez`, nunca `Pérez`. Adivinar el acento de un apellido es corromper un dato personal: `Perez` y `Pérez` son apellidos distintos y ambos existen. Si tu sistema necesita el acento correcto, eso se resuelve pidiéndole al usuario que valide su nombre una vez, no con un diccionario.

Lo mismo con las etiquetas: `sentencia('FACTURACION')` devuelve `Facturacion`. Los acentos de la interfaz se escriben bien en el código; ninguna función los adivina. Para encontrarlos, `references/auditoria-sistema.md` trae una lista de las palabras que casi siempre se escriben mal en software administrativo.

**No normaliza el dato original.** Las funciones de capitalización son para mostrar. La razón social de un CFDI debe quedar exactamente como está registrada ante el SAT, aunque venga en mayúsculas. Lo único que se guarda transformado es la clave de búsqueda, y vive en su propia columna.

---

## Contenido

```
textos-espanol/
├── SKILL.md                     Reglas que Claude carga al activarse
├── scripts/
│   ├── texto.js                 Módulo JS, cero dependencias
│   └── texto.py                 Módulo Python, cero dependencias
└── references/
    ├── ortotipografia.md        Comillas, fechas, moneda MXN, siglas,
    │                            anglicismos frecuentes en interfaces
    ├── arquitectura-capas.md    Collations MySQL/PostgreSQL, columnas de
    │                            búsqueda, validación con Zod, CSV y Excel
    └── auditoria-sistema.md     Cómo arreglar un sistema ya en producción,
                                 por fases y sin romperlo
```

Las referencias no se cargan siempre: Claude lee la que corresponde según lo que estés haciendo. Como documentación suelta también se sostienen — `arquitectura-capas.md` sirve de guía de migración a `utf8mb4` aunque no uses la skill.

---

## Personalizar por proyecto

Cada sistema tiene sus siglas. Todas las funciones aceptan una lista adicional:

```js
nombreCatalogo('MSC MEDITERRANEAN', { siglas: ['MSC', 'CMA', 'ICAVE', 'ZAL'] });
sentencia('MOVIMIENTOS DEL PUERTO DE VERACRUZ', { propios: ['Puerto de Veracruz'] });
```

Si la vas a usar en varios proyectos de la misma empresa, haz un fork y agrega tus siglas directo en la constante `SIGLAS` de ambos módulos.

Las siglas que trae de fábrica cubren lo fiscal mexicano (SAT, CFDI, RFC, CURP, IMSS, STPS), logística (EIR, NIV, VIN, BL) y lo técnico habitual.

---

## Contribuir

Los casos borde de los nombres propios son infinitos y cambian por región. Si encuentras uno que falla —un apellido compuesto, una partícula de otra lengua, una abreviatura fiscal que faltaba— abre un issue con el valor de entrada y el resultado que esperabas, o manda un PR.

Si agregas una función, agrégala a los dos módulos: `texto.js` y `texto.py` deben dar resultados idénticos.

Lo que no entra: diccionarios de acentuación automática, por la razón explicada arriba.

---

## Licencia

MIT.
