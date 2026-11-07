# Solución: AirbnbFacturas → e-Kuatia'i

## Sin clawcall (Paraguay)

Clawcall/MCP no se puede autenticar desde Paraguay en este entorno.
Canal alternativo: instrucciones en Drive/OneDrive + pegar al bot por el
chat que ya uses (WhatsApp/Telegram/WebChat de OpenClaw, o Cursor chat
con AirbnbFacturas).

- Google Doc: https://docs.google.com/document/d/1QAW_usSZIENVMSsEZ4i3-kkElvyP0lsvhGQhQXAxRAs/edit
- OneDrive: `AirbnbFacturas-REPARAR-ekuatiai.md` (raíz del drive)

## Timeline corregida (humano)

- **Desde el 26/10** el bot **sí** emitía sin problema.
- El corte es **más reciente** (no usar 26/10 como fecha de rotura).
- Portal e-Kuatia'i responde OK (verificado).
- Emisión manual en PC/iPad del humano: OK.
- Fallo actual: tramo **Emitir → Entrust/teraguapy** en la **PC del bot**
  (perfil Chrome `dnit-facturas` + WARP).

e-Kuatia'i **no tiene API**. Sin certificado `.p12` PCSC la única vía es
UI + CCFE. La solución operativa es reparar ese salto en la máquina del bot
(y, mientras tanto, emitir asistido).

---

## SOLUCIÓN AHORA (orden fijo — 3 pasos)

### Paso 1 — Hoy: facturar sin frenar el negocio (asistido)

1. Bot arma ficha, valida `base + IVA = total`, deja **Preview** listo.
2. Bot **NO** pulsa Emitir ni abre Entrust.
3. Humano en PC/iPad: Emitir → PIN CCFE → Aprobado.
4. Bot lee DTEs Emitidos y guarda nº + CDC.
5. Dedup por (fecha, RUC, monto). Sin inventar CDC. Sin reemitir en bucle.

Pegá al bot:

```
MODO ASISTIDO TEMPORAL (hasta reparar Entrust en la PC del bot):
- Completá ficha + Preview. STOP en Emitir.
- No abras teraguapy/esignsp. No borres cookies.
- Avisá “LISTO PARA FIRMA HUMANA” con fecha/RUC/monto.
- Cuando diga “ya firmé”, buscá el DTE en Emitidos y guardá nº+CDC.
Confirmá: “Modo asistido ON”.
```

### Paso 2 — Reparar la PC del bot (restaura emisión automática)

En la **misma** PC del bot, perfil **`dnit-facturas`** (no Fork-*), **sin** RPA:

1. Cerrar todas las pestañas `teraguapy` / `esignsp` con error.
2. Abrir e-Kuatia'i → login normal.
3. Emitir **una** factura chica a mano hasta PIN → Aprobado.

| Resultado | Qué hacer |
|-----------|-----------|
| Manual OK | El portal/CCFE andan. El RPA rompe el redirect → Paso 2b |
| Manual 404 / “No encontrado” | CCFE/sesión Entrust rota en esa máquina → Paso 2c |

#### 2b — Si manual OK (arreglar RPA / reglas del bot)

Pegá al bot:

```
REPARAR EMISIÓN AUTOMÁTICA — reglas vigentes:

OBLIGATORIO:
- perfil Chrome dnit-facturas + WARP (NO Fork-*)
- ficha → validar montos → Preview → click real Emitir (misma pestaña)
- Contributor/Entrust → PIN CCFE → esperar Aprobado
- dedup (fecha, RUC, monto); no inventar CDC

PROHIBIDO:
- borrar cookies de teraguapy.set.gov.py / esignsp (CDP o a mano)
- Network.deleteCookies / clearDataForOrigin en esos dominios
- cerrar pestañas teraguapy “por las dudas” antes de Emitir
- copiar/pegar URL de esignsp o reabrir signerProcessId viejos
- reemitir en bucle si sale 404

Si esignsp dice “No encontrado”/404:
- cerrá esa pestaña
- NO reutilices ese signerProcessId
- marcá NEEDS_REVIEW una sola vez y pará

Salí de modo asistido. Probá UNA emisión real y reportá OK+nº/CDC o error exacto.
```

#### 2c — Si manual también falla (reparar CCFE en esa PC)

1. En Chrome `dnit-facturas`: borrar **solo** datos de sitio de
   `teraguapy.set.gov.py` **una vez** (no antes de cada Emitir).
2. Cerrar Chrome por completo; reabrir el perfil; entrar a e-Kuatia'i.
3. Reintentar 1 factura manual.
4. Si sigue mal: reinstalar/verificar el **CCFE** de esa máquina
   (token/lector/software Contributor) en una oficina DNIT o con el
   soporte del certificado — el CCFE es **solo** para e-Kuatia'i.
5. Mientras tanto, quedate en Paso 1 (asistido).

### Paso 3 — Definitivo sin UI (cuando quieras automatizar de verdad)

Salir de e-Kuatia'i → SIFEN/API con `.p12` de un **PCSC** (no el CCFE del portal).
Proveedores REST o libs (`pysifen`, etc.). Sin eso, no hay emisión 100% headless.

---

## Qué NO hacer

- Tratar el 26/10 como fecha de rotura (el bot emitía **después**).
- Borrar cookies teraguapy en cada intento.
- Reemitir en bucle tras 404 esignsp.
- Inventar número/CDC.
- Usar el CCFE de e-Kuatia'i contra SIFEN SOAP/API.

---

## Diagnóstico técnico (resumen)

| Hecho | Conclusión |
|-------|------------|
| Ficha / Preview OK | El bot de datos está bien |
| Humano emite OK; portal UP | SET no está caído |
| Falla en PC del bot al firmar | Sesión/CCFE/RPA en esa máquina |
| `esignsp ... signerProcessId` → 404 | Entrust no reconoce el proceso de firma |
| e-Kuatia'i sin API pública | Automatizar estable exige vía SIFEN + `.p12` |

Fuentes DNIT:
- https://www.dnit.gov.py/web/e-kuatia/ekuatia-i
- https://www.dnit.gov.py/web/e-kuatia/quiero-emitir-electronicamente
