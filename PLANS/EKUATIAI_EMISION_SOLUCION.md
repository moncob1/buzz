# Solución: emitir facturas en e-Kuatia'i sin el 404 de Entrust

## Diagnóstico (evidencia acumulada)

| Hecho | Conclusión |
|-------|------------|
| Ficha, montos, Preview OK | El bot de datos está bien |
| `GET .../esignsp/v2/ui?signerProcessId=...` → 404 | Entrust no reconoce el proceso de firma |
| No queda DTE del día en Emitidos | No hay nada que retomar; no inventar CDC |
| En la PC del humano emite OK | SET/Entrust **no** están caídos |
| En el Chrome del bot falla (antes a veces OK) | El tramo RPA/firma en esa máquina está roto |
| Limpiar cookies no arregló | No era solo “sesión sucia” |

**Causa estructural:** e-Kuatia'i **no tiene API pública**. La emisión gratuita obliga a UI + firma CCFE vía teraguapy/Entrust. Automatizar ese salto con RPA es frágil. El CCFE de e-Kuatia'i, según DNIT, **no es usable fuera** de e-Kuatia'i (no sirve para SIFEN SOAP/API con el mismo cert).

Fuentes DNIT:
- https://www.dnit.gov.py/web/e-kuatia/ekuatia-i
- https://www.dnit.gov.py/web/e-kuatia/quiero-emitir-electronicamente

---

## Solución A — HOY (garantizada): emisión asistida

Dejá de pelear el 404 en el bot. Separá roles:

1. **Bot:** arma ficha, valida `base + IVA = total`, deja Preview listo, registra borrador.
2. **Humano (PC que sí firma):** Emitir + PIN Entrust + confirmar Aprobado.
3. **Bot:** lee DTEs Emitidos / consulta por fecha-RUC-monto y guarda nº + CDC.

Reglas:
- Bot: **STOP Emitir** y **STOP firma Entrust**.
- Sin reemisiones en bucle.
- Dedup por (fecha, RUC receptor, monto) + CDC cuando exista.

Esto desbloquea la facturación **hoy**.

---

## Solución B — Mismo día: recuperar RPA (solo si el bot-machine firma a mano)

Test binario **en la máquina del bot**, perfil `dnit-facturas`, **sin automatización**:

1. Cerrar pestañas esignsp 404.
2. Emitir **una** factura chica a mano hasta PIN/Aprobado.

| Resultado | Acción |
|-----------|--------|
| Manual en bot también 404 | Certificado/Entrust roto en esa máquina → reinstalar/verificar CCFE; seguir con Solución A |
| Manual en bot OK | El portal anda; el RPA rompe el redirect → arreglar RPA (abajo) |

Arreglos RPA (solo si B manual OK):
- No borrar cookies teraguapy antes de Emitir.
- Misma pestaña, click real, sin copiar URL.
- No CDP agresivo en el salto Emitir→Entrust.
- Máximo 1 Emitir; si 404 → NEEDS_REVIEW.
- Comparar HAR humano vs bot del redirect.

---

## Solución C — DEFINITIVA (sin Entrust UI): salir de e-Kuatia'i hacia API/SIFEN

DNIT define **dos vías**:
1. **e-Kuatia'i** (gratis, portal, CCFE DNIT, 1 establecimiento / 1 punto).
2. **e-Kuatia / SIFEN con sistema propio o proveedor** (software + cert PCSC `.p12` + timbrado + CSC).

Para automatizar sin RPA hace falta la vía 2.

### Pasos
1. Obtener certificado digital **`.p12`/`.pfx`** de un **PCSC** habilitado (AC Raíz). **No** reutilizar el CCFE de e-Kuatia'i.
2. Habilitación como facturador electrónico en Marangatu para emisión vía SIFEN/e-Kuatia (no solo portal 'i).
3. Timbrado electrónico + CSC vigentes.
4. Elegir integración:
   - **Proveedor API REST** (JSON → CDC): Sifende, FactElect, Neosystem/Goekua, BillPy, etc.
   - **Directo SIFEN SOAP**: libs open source (`pysifen`, `facturacionelectronicapy-setapi`) + Manual Técnico V150.
5. Bot deja de usar Chrome: `POST` con datos ya validados → guardar CDC → KuDE.

### Flujo objetivo del bot

```
datos venta → validar montos → idempotency key
    → API emitir → CDC
    → poll/webhook hasta APROBADO|RECHAZADO
    → guardar CDC + KuDE
    → si falta dato / rechazo → NEEDS_REVIEW + log
```

Sin teraguapy. Sin PIN en UI. Sin 404 esignsp.

---

## Qué NO hacer más

- Reemitir en bucle cuando hay 404 esignsp.
- Borrar cookies “por las dudas” antes de cada Emitir.
- Inventar número/CDC.
- Intentar SIFEN SOAP con el CCFE de e-Kuatia'i (DNIT: no adaptable fuera del portal).

---

## Decisión recomendada

| Horizonte | Qué hacer |
|-----------|-----------|
| Ahora | **Solución A** (humano firma en PC OK) |
| Hoy/mañana | Test **Solución B** en máquina del bot |
| Esta semana | Arrancar **Solución C** (cert PCSC + API) si querés automatización real |

La única forma de “cargar sin problemas” de forma estable y automática es **dejar el RPA de e-Kuatia'i** y emitir por **SIFEN/API con certificado propio**.
