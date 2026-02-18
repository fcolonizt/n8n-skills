# Workflow n8n v1: Conciliación automática de facturación (Farmacia)

## Nombre sugerido
**FARMACIA - Conciliación automática facturación v1**

## Diagrama lógico de nodos (orden recomendado)
1. **Google Drive Trigger - Drogueria** (folder watch: `/FARMACIA/01_DROGUERIAS_IN/`)
2. **Descargar Excel Drogueria** (Google Drive Download)
3. **Leer Excel Drogueria** (Spreadsheet File Read)
4. **Code - Normalizar Drogueria**
5. **Listar Excels Sucursales** (Google Drive List/Search)
6. **Split In Batches - Sucursales** (batchSize: 1)
7. **Code - Guardar archivo sucursal actual**
8. **Descargar Excel Sucursal** (Google Drive Download)
9. **Leer Excel Sucursal** (Spreadsheet File Read)
10. **Code - Normalizar Sucursal**
11. **(loop) volver a Split In Batches - Sucursales**
12. **Code - Comparador/Conciliador** (sale por output *done* de Split In Batches)
13. **Spreadsheet File - Escribir Reporte** (Write XLSX/CSV)
14. **Code - Nombre de Archivo Reporte**
15. **Subir Reporte a Drive** (`/FARMACIA/03_REPORTES_OUT/`)

---

## Esquema estándar de salida (normalizado)
Cada fila normalizada (droguería y sucursal) debe tener:

- `origen`: `"drogueria"` o `"sucursal"`
- `sucursal_id` (solo sucursal)
- `proveedor` (solo droguería si existe)
- `cuit`
- `nro_factura`
- `fecha` (`YYYY-MM-DD`)
- `importe_total` (number)
- `match_key` = `cuit + '|' + nro_factura + '|' + fecha`

---

## Configuración clave de cada nodo

## 1) Google Drive Trigger - Drogueria
- **Node**: Google Drive Trigger
- **Evento**: `fileCreated`
- **Carpeta monitoreada**: `/FARMACIA/01_DROGUERIAS_IN/`
- **Filtro recomendado**: solo `.xlsx` / `.csv`

## 2) Descargar Excel Drogueria
- **Node**: Google Drive
- **Operation**: `Download`
- **File ID**: `={{$json.id}}`
- **Binary Property**: `data`

## 3) Leer Excel Drogueria
- **Node**: Spreadsheet File
- **Operation**: `Read from file`
- **Binary Property**: `data`
- **Header row**: `true`

## 4) Code - Normalizar Drogueria
- **Mode**: `Run Once for All Items`
- **Salida**: filas normalizadas + persistencia en `staticData.drogueria`

## 5) Listar Excels Sucursales
- **Node**: Google Drive
- **Operation**: `Search/List`
- **Folder**: `/FARMACIA/02_SUCURSALES_IN/`
- **Query** (ejemplo):
  - `mimeType='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet' or mimeType='text/csv'`

## 6) Split In Batches - Sucursales
- **Batch size**: `1`
- **Output 1**: iteración
- **Output 2 (done)**: dispara conciliación

## 7) Code - Guardar archivo sucursal actual
- Guarda nombre actual en `staticData.currentSucursalFileName`

## 8) Descargar Excel Sucursal
- **Node**: Google Drive
- **Operation**: `Download`
- **File ID**: `={{$json.id}}`
- **Binary Property**: `data`

## 9) Leer Excel Sucursal
- **Node**: Spreadsheet File
- **Operation**: `Read from file`
- **Binary Property**: `data`
- **Header row**: `true`

## 10) Code - Normalizar Sucursal
- **Mode**: `Run Once for All Items`
- Extrae `sucursal_id` desde nombre de archivo (ej: `Sucursal_07_2026-02.xlsx` → `07`)
- Acumula en `staticData.sucursales`

## 12) Code - Comparador/Conciliador
- Toma `staticData.drogueria` + `staticData.sucursales`
- Genera estados:
  - `OK`
  - `DIFERENCIA_IMPORTE`
  - `FALTA_EN_SUCURSAL`
  - `FALTA_EN_DROGUERIA`
  - `DUPLICADO_EN_SUCURSALES`
- Define `staticData.reporteFileName` con formato:
  - `Reporte_Conciliacion_YYYY-MM-DD_HHMM.xlsx`

## 13) Spreadsheet File - Escribir Reporte
- **Operation**: `Write to file`
- **Format**: `xlsx` (o `csv` si preferís)
- **Binary Property**: `data`

## 14) Code - Nombre de Archivo Reporte
- Copia `binary.data`
- Inyecta `json.fileName = staticData.reporteFileName`

## 15) Subir Reporte a Drive
- **Node**: Google Drive
- **Operation**: `Upload`
- **Binary Data**: `true`
- **Binary Property**: `data`
- **File Name**: `={{$json.fileName}}`
- **Destino**: `/FARMACIA/03_REPORTES_OUT/`

---

## Código completo de los Code nodes

> Nota: los `COL` son **editables** para mapear nombres de columnas reales.

### A) Code - Normalizar Drogueria
```javascript
const staticData = this.getWorkflowStaticData('global');

const COL = {
  proveedor: ['proveedor', 'laboratorio', 'drogueria', 'distribuidor'],
  cuit: ['cuit', 'cuit_emisor', 'cuit_proveedor'],
  nro_factura: ['nro_factura', 'factura', 'numero_factura', 'comprobante'],
  fecha: ['fecha', 'fecha_factura', 'emision'],
  importe_total: ['importe_total', 'total', 'monto_total', 'importe']
};

function getValue(row, aliases) {
  for (const key of aliases) {
    const found = Object.keys(row).find(k => k.toLowerCase().trim() === key.toLowerCase().trim());
    if (found && row[found] !== undefined && row[found] !== null && row[found] !== '') return row[found];
  }
  return null;
}

function toDateYMD(value) {
  if (!value) return null;
  const str = String(value).trim();

  if (/^\d{4}-\d{2}-\d{2}$/.test(str)) return str;

  const ddmmyyyy = str.match(/^(\d{1,2})[\/-](\d{1,2})[\/-](\d{2,4})$/);
  if (ddmmyyyy) {
    const dd = ddmmyyyy[1].padStart(2, '0');
    const mm = ddmmyyyy[2].padStart(2, '0');
    const yyyy = ddmmyyyy[3].length === 2 ? `20${ddmmyyyy[3]}` : ddmmyyyy[3];
    return `${yyyy}-${mm}-${dd}`;
  }

  const jsDate = new Date(str);
  if (!Number.isNaN(jsDate.getTime())) return jsDate.toISOString().slice(0, 10);

  return null;
}

function toNumber(value) {
  if (value === null || value === undefined || value === '') return 0;
  if (typeof value === 'number') return value;

  let str = String(value).trim().replace(/\s/g, '');
  const hasComma = str.includes(',');
  const hasDot = str.includes('.');

  if (hasComma && hasDot) {
    str = str.replace(/\./g, '').replace(',', '.');
  } else if (hasComma && !hasDot) {
    str = str.replace(',', '.');
  }

  const num = Number(str);
  return Number.isFinite(num) ? num : 0;
}

const normalized = $input.all().map((item) => {
  const row = item.json;
  const cuit = String(getValue(row, COL.cuit) ?? '').replace(/\D/g, '');
  const nroFactura = String(getValue(row, COL.nro_factura) ?? '').trim();
  const fecha = toDateYMD(getValue(row, COL.fecha));
  const importeTotal = toNumber(getValue(row, COL.importe_total));

  return {
    json: {
      origen: 'drogueria',
      sucursal_id: null,
      proveedor: getValue(row, COL.proveedor) ?? null,
      cuit,
      nro_factura: nroFactura,
      fecha,
      importe_total: importeTotal,
      match_key: `${cuit}|${nroFactura}|${fecha}`
    }
  };
});

staticData.drogueria = normalized.map(i => i.json);
return normalized;
```

### B) Code - Normalizar Sucursal
```javascript
const staticData = this.getWorkflowStaticData('global');
const fileName = staticData.currentSucursalFileName || '';

const COL = {
  cuit: ['cuit', 'cuit_cliente', 'cuit_emisor', 'cliente_cuit'],
  nro_factura: ['nro_factura', 'factura', 'numero_factura', 'comprobante'],
  fecha: ['fecha', 'fecha_factura', 'emision'],
  importe_total: ['importe_total', 'total', 'monto_total', 'importe']
};

function getValue(row, aliases) {
  for (const key of aliases) {
    const found = Object.keys(row).find(k => k.toLowerCase().trim() === key.toLowerCase().trim());
    if (found && row[found] !== undefined && row[found] !== null && row[found] !== '') return row[found];
  }
  return null;
}

function toDateYMD(value) {
  if (!value) return null;
  const str = String(value).trim();

  if (/^\d{4}-\d{2}-\d{2}$/.test(str)) return str;

  const ddmmyyyy = str.match(/^(\d{1,2})[\/-](\d{1,2})[\/-](\d{2,4})$/);
  if (ddmmyyyy) {
    const dd = ddmmyyyy[1].padStart(2, '0');
    const mm = ddmmyyyy[2].padStart(2, '0');
    const yyyy = ddmmyyyy[3].length === 2 ? `20${ddmmyyyy[3]}` : ddmmyyyy[3];
    return `${yyyy}-${mm}-${dd}`;
  }

  const jsDate = new Date(str);
  if (!Number.isNaN(jsDate.getTime())) return jsDate.toISOString().slice(0, 10);

  return null;
}

function toNumber(value) {
  if (value === null || value === undefined || value === '') return 0;
  if (typeof value === 'number') return value;

  let str = String(value).trim().replace(/\s/g, '');
  const hasComma = str.includes(',');
  const hasDot = str.includes('.');

  if (hasComma && hasDot) {
    str = str.replace(/\./g, '').replace(',', '.');
  } else if (hasComma && !hasDot) {
    str = str.replace(',', '.');
  }

  const num = Number(str);
  return Number.isFinite(num) ? num : 0;
}

function extractSucursalId(name) {
  const m = String(name || '').match(/sucursal[_\- ]?(\d{1,3})/i);
  if (!m) return 'SIN_ID';
  return m[1].padStart(2, '0');
}

const sucursalId = extractSucursalId(fileName);
const normalized = $input.all().map((item) => {
  const row = item.json;
  const cuit = String(getValue(row, COL.cuit) ?? '').replace(/\D/g, '');
  const nroFactura = String(getValue(row, COL.nro_factura) ?? '').trim();
  const fecha = toDateYMD(getValue(row, COL.fecha));
  const importeTotal = toNumber(getValue(row, COL.importe_total));

  return {
    json: {
      origen: 'sucursal',
      sucursal_id: sucursalId,
      proveedor: null,
      cuit,
      nro_factura: nroFactura,
      fecha,
      importe_total: importeTotal,
      match_key: `${cuit}|${nroFactura}|${fecha}`
    }
  };
});

if (!Array.isArray(staticData.sucursales)) staticData.sucursales = [];
for (const row of normalized) staticData.sucursales.push(row.json);

return normalized;
```

### C) Code - Comparador/Conciliador
```javascript
const staticData = this.getWorkflowStaticData('global');
const drogueria = Array.isArray(staticData.drogueria) ? staticData.drogueria : [];
const sucursales = Array.isArray(staticData.sucursales) ? staticData.sucursales : [];

function round2(v) {
  return Math.round((Number(v) + Number.EPSILON) * 100) / 100;
}

const sucMap = new Map();
for (const s of sucursales) {
  const key = s.match_key;
  if (!sucMap.has(key)) sucMap.set(key, []);
  sucMap.get(key).push(s);
}

const droMap = new Map();
for (const d of drogueria) {
  const key = d.match_key;
  if (!droMap.has(key)) droMap.set(key, []);
  droMap.get(key).push(d);
}

const allKeys = new Set([...droMap.keys(), ...sucMap.keys()]);
const report = [];

for (const key of allKeys) {
  const dRows = droMap.get(key) || [];
  const sRows = sucMap.get(key) || [];

  if (dRows.length === 0 && sRows.length > 0) {
    for (const s of sRows) {
      report.push({
        estado: 'FALTA_EN_DROGUERIA',
        match_key: key,
        cuit: s.cuit,
        nro_factura: s.nro_factura,
        fecha: s.fecha,
        importe_drogueria: null,
        importe_sucursal: s.importe_total,
        diferencia_importe: null,
        proveedor: null,
        sucursal_id: s.sucursal_id
      });
    }
    continue;
  }

  if (dRows.length > 0 && sRows.length === 0) {
    for (const d of dRows) {
      report.push({
        estado: 'FALTA_EN_SUCURSAL',
        match_key: key,
        cuit: d.cuit,
        nro_factura: d.nro_factura,
        fecha: d.fecha,
        importe_drogueria: d.importe_total,
        importe_sucursal: null,
        diferencia_importe: null,
        proveedor: d.proveedor || null,
        sucursal_id: null
      });
    }
    continue;
  }

  const sucIds = [...new Set(sRows.map(r => r.sucursal_id).filter(Boolean))];
  if (sucIds.length > 1) {
    for (const s of sRows) {
      const d = dRows[0];
      report.push({
        estado: 'DUPLICADO_EN_SUCURSALES',
        match_key: key,
        cuit: s.cuit,
        nro_factura: s.nro_factura,
        fecha: s.fecha,
        importe_drogueria: d?.importe_total ?? null,
        importe_sucursal: s.importe_total,
        diferencia_importe: d ? round2(s.importe_total - d.importe_total) : null,
        proveedor: d?.proveedor || null,
        sucursal_id: s.sucursal_id
      });
    }
    continue;
  }

  const d = dRows[0];
  const s = sRows[0];
  const diff = round2((s.importe_total || 0) - (d.importe_total || 0));
  const estado = Math.abs(diff) <= 0.01 ? 'OK' : 'DIFERENCIA_IMPORTE';

  report.push({
    estado,
    match_key: key,
    cuit: d.cuit || s.cuit,
    nro_factura: d.nro_factura || s.nro_factura,
    fecha: d.fecha || s.fecha,
    importe_drogueria: d.importe_total,
    importe_sucursal: s.importe_total,
    diferencia_importe: diff,
    proveedor: d.proveedor || null,
    sucursal_id: s.sucursal_id || null
  });
}

const now = new Date();
const yyyy = now.getFullYear();
const mm = String(now.getMonth() + 1).padStart(2, '0');
const dd = String(now.getDate()).padStart(2, '0');
const hh = String(now.getHours()).padStart(2, '0');
const mi = String(now.getMinutes()).padStart(2, '0');
const fileName = `Reporte_Conciliacion_${yyyy}-${mm}-${dd}_${hh}${mi}.xlsx`;

staticData.reporteFileName = fileName;
staticData.drogueria = [];
staticData.sucursales = [];

return report.map(r => ({ json: r }));
```

---

## JSON de exportación de n8n
Podés importar directamente este archivo como base v1:

- `docs/workflows/farmacia-conciliacion-v1.json`

> Si tu instancia de n8n usa una versión distinta del nodo de Google Drive, ajustá únicamente los parámetros de carpeta (`folderId/parents`) y credenciales; la lógica de conciliación no cambia.

