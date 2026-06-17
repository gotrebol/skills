# Ejemplo de Plantilla Configurada — Institución Financiera Ejemplo (PDF de formulario)

Este es un **ejemplo trabajado** de una plantilla de cliente configurada por Trébol: la "SOLICITUD DE DICTAMEN PERSONA MORAL" de Grupo Financiero Ejemplo, en **PDF con campos de formulario** (AcroForm, ~1.092 campos, 19 páginas). Sirve como referencia de **cómo se ve un mapeo real** y de varias decisiones de configuración que no son obvias.

## Hallazgo #1 (crítico) — En PDF, el token va en el VALOR del campo, no en el nombre

En este PDF los **nombres internos de los campos son arbitrarios** (`Texto100`, `RFC_2`, `Contrato_Apo1_EscrituraPoder`, etc.) y el **token de Trébol `{variable}` está puesto como el VALOR / texto por defecto del campo**. Ejemplo: un campo llamado internamente `Texto138` tiene como valor `{key_person_1_names}`.

Esto **corrige el supuesto** de que "en PDF el nombre del campo = la variable". La convención observada en una plantilla real es:
- **Convención A (la confirmada en este ejemplo):** el token `{variable}` se escribe como **valor/apariencia por defecto** del campo de formulario; el nombre del campo se deja como venga.
- **Convención B (la que asumía el helper):** el nombre del campo = la variable.

Cuando configures un PDF, **replica la convención que ya trae la plantilla del cliente**. Si el cliente entrega un PDF con campos pero sin tokens, la forma segura (y la que la plataforma espera, según este ejemplo) es **poner el token como valor del campo**. Cada token sigue siendo único por instancia; no se duplican.

## Hallazgo #2 — Mapeo de poderes (etiqueta del cliente → variable Trébol)

El template tiene una rejilla de poderes por apoderado. El mapeo de la etiqueta del cliente a la familia de poder de Trébol **no es literal** y conviene tenerlo fijo:

| Etiqueta en la plantilla del cliente | Familia de poder Trébol (`key_person_N_power_<P>_*`) |
|---|---|
| PODER ADMINISTRACIÓN | `administration` |
| PODER ESPECIAL PARA CUENTAS BANCARIAS | `bank_accounts` |
| PODER TÍTULOS Y OPERACIONES DE CRÉDITO | `loans` |
| PODER DOMINIO | `assets_management` |
| PODER OTORGAR / DELEGAR / SUSTITUIR PODERES | `delegate` |

Notas:
- **`PODER DOMINIO` → `assets_management`** (no existe un poder llamado "dominio"; el dominio es `assets_management`).
- **`delegate` es un poder REAL y confirmado** por este template (otorgar/delegar/sustituir poderes). Resuelve la duda que estaba en `corrections`.
- El sexto poder `lawsuits_and_collections` (pleitos y cobranzas) existe en el catálogo pero este cliente no lo desglosa en casilla propia.
- Cada poder usa los sub-campos: `_x_mark` (la casilla "CUENTA CON EL PODER"), `_signature_type` (TIPO DE FIRMA), `_duration` (VIGENCIA), `_limits_description` (LIMITACIONES).

## Hallazgo #3 — DOMICILIO: una sola sección, dos posibles fuentes

El cliente nos pidió **dos variantes del mismo PDF**: una llamada "Dirección comercial" y otra "Dirección fiscal". Ambas resultaron tener **los mismos tokens** en la sección DOMICILIO: en las dos se mapeó a la **dirección comercial desagregada** (`comercialAddressDisaggregated_*`). Es decir, para esta plantilla el equipo decidió llenar el domicilio con la comercial.

Lección de configuración: la sección "DOMICILIO" de una plantilla puede alimentarse de la dirección **comercial** o **fiscal** según lo que el cliente quiera ver; el mapeo lo decides tú. Recuerda la asimetría de sub-llaves (comercial en español, fiscal en inglés — ver `variable-catalog.md` → "Domicilios"). En este PDF, el domicilio de la empresa usa las sub-llaves comerciales en español, incluyendo las de interior: `comercialAddressDisaggregated_tipo_vialidad`, `_nombre_vialidad`, `_numero_exterior`, `_numero_interior`, `_tipo_interior`, `_nombre_asentamiento`, `_codigo_postal`.

El "DOCUMENTO CON EL QUE ACREDITA" (comprobante de domicilio) se mapea con flags por tipo: `address_service_type_electricity`, `_gas`, `_water`, `_phone`, `_property_tax`, `_bank_statement`, `_lease` (casillas), más `address_service_type` (el texto del tipo) y la fecha `comercialAddress_source_date_day/_month/_year`.

## Hallazgo #4 — Instancias de slots fijos (numeración por familia)

El template tiene **slots fijos** que se mapean a instancias indexadas, respetando la numeración por familia:
- **Apoderados** APODERADO 1–4 → `key_person_1` … `key_person_4` (**1-based**), cada uno con nombre (`_names`, `_firstSurname`, `_secondSurname`), escritura del poder (`_source_number`, `_source_notaryName`, `_source_notaryNumber`, `_source_notaryCity`, `_source_notaryState`, `_source_folioNumber`, `_source_date_day/_month/_year`, `_source_folioInscriptionDate_day/_month/_year`), rol (`_role_name`) y la rejilla de poderes.
- **Consejo de administración** → `board_member_0` … `board_member_9` (**0-based**): `_names`, `_firstSurname`, `_secondSurname`, `_roleName`.
- **Accionistas personas físicas** → `shareholder_person_0` … `_4` (**0-based**) con domicilio MX desagregado.
- **Propietarios reales (UBOs)** → `ubos_business_0` y `ubos_business_1` (hasta 2 empresas/capas), cada una con `ubos_person_0` … `_4`.
- **Personas autorizadas / organigrama** → familias `forms_*` (ver Hallazgo #5).

## Hallazgo #5 — Familias `forms_*` (personas autorizadas y organigrama)

- **Personas autorizadas**: `forms_authorizedPersons_{first|second|third}AuthorizedPerson_{names|firstLastName|secondLastName}` (y `_proofOfAddress` para el comprobante).
- **Organigrama / jerarquía siguiente a Dirección General**: `forms_organization_{first|second|third}NextHierarchy_{names|firstLastName|secondLastName|position}`.

## Hallazgo #6 — UBOs: dos niveles y identidades anidadas

- Nivel "entrada UBO": `ubos_business_0_ubos_0_{names|firstSurname|secondSurname|share_percentage}` (incluye **`share_percentage`**, el % de participación).
- Nivel "detalle persona": `ubos_business_0_ubos_person_0_{names|firstSurname|secondSurname|gender|nationality|birthDate_*|birthEntity|email|phone|occupation|id_number_rfc|mx_address_*}`.
- **Identidad externa por CURP (anidada)**: `ubos_business_0_ubos_person_0_externalIdentities_curp_applicantData_curp`. ⚠️ Hay inconsistencia de grafía en el origen: algunas claves usan camelCase (`externalIdentities_curp_applicantData_curp`) y otras snake (`external_identities_curp_applicantData_secondSurname`). Verifica la grafía exacta contra el dato real / `dictionary-example.json` antes de cerrar; anótalo en `corrections` si lo confirmas.

## Resumen de variables nuevas confirmadas por este template (no estaban en el JSON de ejemplo)

- Flags de comprobante de domicilio: `address_service_type_{electricity,gas,water,phone,property_tax,bank_statement,lease}`.
- Domicilio comercial interior: `comercialAddressDisaggregated_{numero_interior,tipo_interior}`.
- Domicilio MX de personas, interior: `*_mx_address_{numero_interior,tipo_interior}`.
- Poder `delegate` completo: `key_person_N_power_delegate_{x_mark,signature_type,duration,limits_description}`.
- Apoderado, sub-fuente: `key_person_N_source_folioInscriptionDate_{day,month,year}`.
- `forms_organization_{first,second,third}NextHierarchy_{names,firstLastName,secondLastName,position}`.
- `forms_authorizedPersons_{second,third}AuthorizedPerson_{names,firstLastName,secondLastName}`.
- UBO: `ubos_business_N_ubos_N_share_percentage`; `ubos_business_N_ubos_person_N_externalIdentities_curp_applicantData_*`; `people_stakeholder_{id_number_rfc,ubo_forms_email,ubo_forms_phone}`.

> Recordatorio de precedencia: estas variables quedan documentadas en el catálogo. Para grafía exacta, `corrections` manda, luego `dictionary-example.json`. La ausencia en el JSON no las descarta — este template real las confirma.
