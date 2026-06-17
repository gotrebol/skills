# Catálogo de Variables de Trébol (grounding)

Diccionario destilado de todas las variables (IDs) que Trébol reemplaza automáticamente al exportar una verificación contra una plantilla. Fuente: documentación viva en `https://www.gotrebol.com/docs/plantillas/` + la plantilla de referencia (`example-template-reference.md`). Ante conflicto con `corrections.md`, prevalece `corrections.md`.

> **Ground truth de grafía y forma:** para confirmar **cómo se escribe exactamente** una variable y **qué forma tiene el dato** (escalar / lista-bucle / objeto), consulta el payload real `dictionary-example.json` (ver `dictionary-example.md`). Ese JSON gana sobre este catálogo y sobre la referencia en cuanto a grafía, salvo que `corrections` diga otra cosa. Este catálogo sigue siendo útil para las familias que **no** aparecen en ese payload (ej. apoderados/consejo, si la verificación de ejemplo no los tenía).

> Regla de oro: **no se inventan variables.** Si un campo del cliente no aparece aquí (ni en la referencia ni en la web), se marca `SIN_VARIABLE`, no se "deduce" un ID plausible. Trébol solo reemplaza IDs que existen; un ID inventado se queda literal en el documento.

Convenciones de numeración (¡crítico, hay dos esquemas!):
- **Apoderados:** 1-based → `key_person_1`, `key_person_2`, …
- **Accionistas, consejo, administradores, comisarios:** 0-based → `shareholder_0`, `board_member_0`, `executive_0`, `auditor_0`, …
- **Colombia y UBOs:** indexados con `{n}` / `{i}` / `{j}` (ver sección Colombia; los ejemplos web mezclan separador `_` y `-` — ver nota).
- **UBOs México (segundo nivel, en la referencia):** `ubos_business_0`, `ubos_business_0_ubos_person_0`, …

---

## 1. Empresa / Estatutos / Fiscal / Domicilio  (doc: /plantillas/empresa)

### Detalles generales (acta más reciente)
| Campo | Variable |
|---|---|
| Nombre asignado a la verificación | `{verification_businessName}` |
| Nombre de la persona moral | `{legal_businessName}` |
| Tipo de sociedad | `{legal_businessType}` |
| Tipo de sociedad abreviado | `{legal_businessType_abbreviated}` |
| Número de registro RPPYC | `{legal_businessFolioNumber}` |
| Duración | `{legal_businessDuration}` |
| Domicilio legal | `{legal_businessLegalAddress}` |
| Puede suscribir títulos de crédito | `{legal_businessCreditorAbility}` |
| Puede dar activos en garantía | `{legal_businessAssetPledging}` |
| Puede ser garante | `{legal_businessGuarantorAbility}` |
| Fecha del acta más reciente | `{legal_latestActaDate}` |
| Fecha de expiración | `{legal_businessExpiryDate}` |

### Constitución
| Campo | Variable |
|---|---|
| Tiene acta constitutiva (condicional) | `{registration_hasConstitutive}` |
| Nombre al constituirse | `{registration_businessName}` |
| Tipo de sociedad al constituirse | `{registration_businessType}` / `{registration_businessType_abbreviated}` |
| Número de escritura | `{registration_actaNumber}` |
| Número de notaría | `{registration_notaryNumber}` |
| Nombre del notario | `{registration_notaryName}` |
| Ciudad / Estado de la notaría | `{registration_city}` / `{registration_state}` |
| Fecha de constitución | `{registration_registrationDate}` |
| Folio RPPYC al constituirse | `{registration_folioNumber}` |
| Fecha de inscripción RPPYC | `{registration_folioDate}` |

### Fiscal (CSF)
| Campo | Variable |
|---|---|
| Tiene constancia (condicional) | `{tax_hasConstancia}` |
| Nombre / Tipo de sociedad | `{tax_businessName}` / `{tax_businessType}` / `{tax_businessType_abbreviated}` |
| RFC | `{tax_businessTaxId}` |
| Actividades económicas (lista) | `{#tax_businessMainActivity}{economicActivity}, {percentage}%{/}` |
| Actividad principal | `{tax_businessMainActivity_economicActivity}` |
| Fecha / % actividad principal | `{tax_businessMainActivity_date}` / `{tax_businessMainActivity_percentage}` |
| Estatus en el padrón | `{tax_businessFiscalStatus}` |
| Régimen | `{tax_businessRegime}` |

### Domicilios

Cada domicilio de la empresa viene en **dos formas que conviven**: una **línea única** y una **versión desagregada** en sub-campos. Según la plantilla del cliente uses una u otra (ver "Cómo mapear domicilios" abajo).

| Campo | Variable |
|---|---|
| Dirección comercial (línea única) / fecha | `{comercialAddress}` / `{comercialAddress_source_date}` (+ `_day`, `_month`, `_year`) |
| Dirección fiscal (línea única) / fecha | `{fiscalAddress}` / `{fiscalAddress_source_date}` (+ `_day`, `_month`, `_year`) |
| Razón social asociada a comercial | `{comercialAddress_razon_social}` |

**⚠️ Las dos direcciones desagregadas NO comparten el mismo esquema de sub-llaves.** La comercial usa sub-llaves en **español**; la fiscal usa sub-llaves en **inglés**. No asumas simetría: no existe `fiscalAddressDisaggregated_codigo_postal` ni `comercialAddressDisaggregated_post_code`.

**Comercial desagregada (sub-llaves en ESPAÑOL)** — `comercialAddressDisaggregated_*`:
`_tipo_vialidad`, `_nombre_vialidad`, `_numero_exterior`, `_numero_interior`, `_tipo_interior`, `_tipo_asentamiento`, `_nombre_asentamiento`, `_codigo_postal`, `_municipio_o_ente_territorial`, `_entidad_federativa`, `_pais`

**Comprobante de domicilio** (sección "documento con el que se acredita"): el tipo va en `{address_service_type}` (texto) y, cuando la plantilla tiene casillas por tipo, en flags x_mark: `address_service_type_electricity` (energía eléctrica), `_gas` (gas natural), `_water` (agua), `_phone` (teléfono), `_property_tax` (impuesto predial), `_bank_statement` (estado de cuenta bancario), `_lease` (contrato de arrendamiento).

**Fiscal desagregada (sub-llaves en INGLÉS)** — `fiscalAddressDisaggregated_*`:
`_street_type`, `_street_name`, `_external_number`, `_internal_number`, `_neighborhood`, `_district`, `_post_code`, `_state`

> Nota: cada verificación puebla solo los sub-campos que su dato real contiene (una dirección "DOMICILIO CONOCIDO" puede no traer `_tipo_vialidad`). La ausencia de un sub-campo en un ejemplo no significa que la variable no exista; el esquema completo es el de arriba.

**Domicilios de personas** (apoderado, accionista, consejero, comisario, UBO) siguen otro patrón: línea única `*_address` (ej. `{shareholder_0_address}`) + desagregado MX en **español** `*_mx_address_*` (`_tipo_vialidad`, `_nombre_vialidad`, `_numero_exterior`, `_tipo_interior`, `_numero_interior`, `_tipo_asentamiento`, `_nombre_asentamiento`, `_codigo_postal`, `_municipio_o_ente_territorial`, `_entidad_federativa`, `_localidad`, `_pais`). Hay también una variante anidada `*_identity_person_id_mx_address_direccion_*` con los mismos sufijos.

#### Cómo mapear domicilios según la plantilla del cliente
- **Plantilla con una sola línea de dirección** → usa la variable de línea única (`{comercialAddress}` o `{fiscalAddress}`, o `{shareholder_0_address}` para personas).
- **Plantilla con campos separados** (calle, número, colonia, CP, municipio, estado…) → usa los sub-campos desagregados, **eligiendo el idioma correcto por familia**: español para comercial y para personas (`*_mx_address_*`), inglés para fiscal de la empresa.
- Si la plantilla separa la dirección pero el dato real no trae ese sub-campo, deja el token igual (la plataforma lo dejará vacío) o usa el inverso `{^...}` en Word para texto alternativo.


### Carátula bancaria de la empresa
`{banking_entityName}`, `{banking_address}`, `{banking_clabe_number}`, `{banking_bank_account_number}`, `{banking_bank_name}`, `{banking_bank_plaza}`, `{banking_rfc}`, `{banking_currency}`

### FIEL de la empresa
`{fiel_serialNumber}`

---

## 2. Apoderados / Representantes legales  (doc: /plantillas/apoderados) — **1-based** `key_person_N`

### Identidad
`{key_person_1}` (nombre), `{key_person_1_tax_id}`, `{key_person_1_role_name}`, `{key_person_1_id}`
RENAPO / external_identities: `{key_person_1_external_identities_names}`, `_firstSurname`, `_secondSurname`, `_gender`, `_birthDate`, `_nationality`, `_birthEntity`, `_curp`, `_registerEntity`, `_registerMunicipality`, `_evidentiaryDocument`, `_actNumber`, `_registryDate`

### Acta/poder de origen (provenance)
`{key_person_1_source_type}`, `_number`, `_date`, `_notaryName`, `_notaryNumber`, `_notaryCity`, `_notaryState`, `_folioNumber`, `_folioDate`

### Carátula bancaria / domicilio del apoderado
Banco: `{key_person_1_source_bankName}`, `_accountNumber`, `_clabeNumber`, `_currency`, `_rfc`; `{key_person_1_bankAddress}`
Comercial: `{key_person_1_comercialAddress}` (+ `_source_date`, `_day`, `_month`, `_year`)
Fiscal: `{key_person_1_fiscalAddress}` (+ `_source_date`, `_day`, `_month`, `_year`)
Domicilio MX desagregado (usado en la referencia): `{key_person_1_mx_address_tipo_vialidad}`, `_nombre_vialidad`, `_numero_exterior`, `_tipo_interior`, `_numero_interior`, `_nombre_asentamiento`, `_municipio_o_ente_territorial`, `_codigo_postal`, `_entidad_federativa`, `_localidad`, `_pais`

### Facultades (poderes) — 6 tipos. Patrón por tipo `<P>`:
Tipos: `administration` (administración), `assets_management` (dominio), `loans` (títulos y operaciones de crédito), `bank_accounts` (cuentas bancarias), `lawsuits_and_collections` (pleitos y cobranzas), `delegate` (otorgar/delegar/sustituir poderes — **confirmado real** contra template real de cliente; ver `example-client-mapping.md`).

Mapeo etiqueta-de-cliente → tipo (no siempre literal):
| Etiqueta típica en la plantilla | `<P>` |
|---|---|
| Administración | `administration` |
| Especial para cuentas bancarias | `bank_accounts` |
| Títulos y operaciones de crédito | `loans` |
| **Dominio** | `assets_management` |
| **Otorgar/Delegar/Sustituir poderes** | `delegate` |
| Pleitos y cobranzas | `lawsuits_and_collections` |

Por cada tipo:
- `{key_person_1_power_<P>_power_name}` — nombre del poder
- `{key_person_1_power_<P>_has_power}` — booleano
- `{key_person_1_power_<P>_duration}` — vigencia
- `{key_person_1_power_<P>_signature_type}` — `"joint"` (mancomunada) / `"individual"`
- `{key_person_1_power_<P>_has_limits}` / `_limits_description}`
- Condicionales de firma: `{#key_person_1_power_<P>_is_joint}M{/}` y `{#key_person_1_power_<P>_is_individual}I{/}`
- **`x_mark`** (usado en la referencia, no documentado en la web pero existe): `{key_person_1_power_<P>_x_mark}` — marca "X" si cuenta con el poder. Ojo: `assets_management` aparece como `key_person_1_power_assets_management_x_mark`.

Lista de poderes (solo Word): `{#key_person_1_powers}{power_name}: {signature_type}, {/key_person_1_powers}`

> Gotcha: la web documenta `assets_management_name` (sin `power_`) pero los demás con `power_name`. Verificar contra la referencia.

---

## 3. Accionistas / Capital  (doc: /plantillas/accionistas) — **0-based** `shareholder_0`

### Lista (bucle, solo Word)
`{#shareholders}{name} {total_shares} {total_value} {share_percentage}%{/}`
En la referencia el bucle usa además: `{fixed_value}`, `{fixed_shares}`, `{variable_value}`, `{variable_shares}` dentro de `{#shareholders}...{/}`.

### Individual `shareholder_0_*`
`_name`, `_type` (física/moral), `_nationality`, `_id_type`, `_id_number`, `_currency`, `_fixed_shares`, `_fixed_value`, `_variable_shares`, `_variable_value`, `_total_shares`, `_total_value`, `_share_percentage`

### Totales de capital
`{capital_fixedValue}`, `{capital_fixedShares}`, `{capital_variableValue}`, `{capital_variableShares}`, `{capital_totalValue}`, `{capital_totalShares}`

### Fuente del cuadro accionario (en la referencia)
`{shareholders_source_acta_number}`, `_source_date`, `_notary_name`, `_notary_number`, `_acta_city`, `_folio_number`, `_folio_date`
Patrón FUENTE: `{#shareholders_source_acta_number}…{/}{^shareholders_source_acta_number}No se encontró el cuadro accionario más reciente{/}`

---

## 4. Órgano de Administración  (doc: /plantillas/administracion) — **0-based**

### Tipo de administración (mutuamente excluyentes)
`{group_type}` (`"Consejo"` / `"Unipersonal"`), condicionales `{#is_counsel}…{/is_counsel}`, `{#is_unipersonal}…{/is_unipersonal}`
Acceso rápido: `{people_president}`, `{people_secretary}`, `{people_admin}`

### Consejo — bucle (solo Word)
`{#board_members}{names} {firstSurname} {secondSurname} {idNumber} {idType} {roleName}{/}` + condicionales `{is_president}`, `{is_secretary}`, marcas `{is_man_x_mark}`, `{is_woman_x_mark}`

### Consejo — individual `board_member_0_*`
`_names`, `_idNumber`, `_idType`, `_roleName`, `_event`, `_duration`, `_is_president`, `_is_secretary`, `_firstSurname`, `_secondSurname`, `_is_man_x_mark`, `_is_woman_x_mark`
Fuente: `{board_member_0_source_itemType}`, `_documentNumber`, `_documentDate`, `_extraFields_notaryName`, `_extraFields_notaryNumber`, `_extraFields_notaryCity`, `_extraFields_notaryState`
Domicilio: `{board_member_0_mx_address_tipo_vialidad}` … (mismo set MX que apoderados)

### Administrador único — bucle / individual `executive_0_*`
`{#executives}{names} {roleName} {idNumber} {idType}{/}`; individual `executive_0_names`, `_idNumber`, `_idType`, `_roleName`, `_event`, `_duration`, `_firstSurname`, `_secondSurname`, `_is_man_x_mark`, `_is_woman_x_mark`; fuente `executive_0_source_*` y domicilio `executive_0_mx_address_*` (mismos sufijos que board_member).

### Stakeholder principal
`{people_stakeholder_names}`, `_firstSurname`, `_secondSurname`, `_idNumber`, `_idType`, `_roleName`, `_is_man_x_mark`, `_is_woman_x_mark` + domicilio `people_stakeholder_mx_address_*`

---

## 5. Órgano de Vigilancia (comisarios)  (doc: /plantillas/vigilancia) — **0-based**

Bucle (solo Word): `{#auditors}{names} {roleName} {idNumber} {idType}{/}` + `{firstSurname}`, `{secondSurname}`, `{is_man_x_mark}`, `{is_woman_x_mark}`
Individual `auditor_0_*`: `_names`, `_idNumber`, `_idType`, `_roleName`, `_event`, `_duration`, `_firstSurname`, `_secondSurname`, `_is_man_x_mark`, `_is_woman_x_mark`
Fuente `auditor_0_source_*` y domicilio `auditor_0_mx_address_*` (mismos sufijos que board_member).

---

## 6. Colombia  (doc: /plantillas/colombia) — indexado `{n}`, `{i}`, `{j}`

> **Nota de separador (gotcha real):** las tablas de la web usan `_` (`{rut_co_1_nit}`) pero los ejemplos al final usan `-` (`{rut_co-1-businessName}`). Confirmar con el cliente / probar contra una verificación cuál acepta la cuenta antes de entregar. No mezclar dentro de un mismo documento.

- **RUT** `rut_co_{n}_…`: `nit`, `verificationDigit`, `businessName`, `country`, `department`, `city`, `mainAddress`, `emailAddress`, `phone1`, `phone2`, `constitutionDate`, `registrationDate`, `merchantNumber`; listas `activities_{i}_{code|startDate|description}`, `responsibilities_{i}_{code|name}`, `representatives_{i}_{type|startDate|idType|idNumber|names}`
- **Cámara de Comercio** `cc_co_ops_{n}_…`: `taxIdNumber`, `expeditionDate`, `legalName`, `legalEmail`, `phone`, `corporatePurpose`, `legalAddress_{city|country|addressLine1}`, `businessAddress_{city|country|addressLine1}`, `equity_{paid|authorized|subscribed}-{value|nominalValue|numberOfShares}`, `boardMembers_data_{i}_{id|name|role|id_type}`, `taxAuditors_data_{i}_{id|name|role|id_type}`, `extractedData_data_representantes_legales_{i}_…`, `documentMetadata_url`
- **RUES** `rues_{n}_…`: `taxIdNumber`, `taxIdVerificationCode`, `legalName`, `countryRegistration`, `stateRegistration`, `registrationCategory`, `registrationStatus`, `registrationCityChamber`, `registrationId`, `renewalDate`, `registrationDate`, `registryEndDate`, `lastUpdateDate`, `cancellationDate`, `cancellationMotive`, `societyType`, `organizationType`, `signatoryPowers`, `businessActivities_{i}_{class|name}`, `signatories_data_{i}_{id|name|role|id_type}`
- **Consulta NIT DIAN** `public_rut_co_{n}_…`: `nit`, `businessName`, `verificationDigit`, `date`, `status`, `message`, `rutValidationResult`
- **Dirección pública CC** `public_address_cc_co_{n}_…`: `legalName`, `organizationType`, `legalEmail`, `phone`, `taxIdNumber`, `businessAddress_{address|city|state|country}`
- **UBOs Colombia** `ubos_{n}_…`: `id`, `idNumber`, `idType`, `name`, `type` (person/business), `sharePercentage`, `nationality`, `residence`, `isPep`, `isPubliclyTraded`; accionistas `ubos_{n}_shareholders_{i}_…` y segundo nivel `ubos_{n}_shareholders_{i}_shareholders_{j}_…`

---

## 7. Consultas adicionales (NO documentadas en la web — solo en la plantilla de referencia)

Estas secciones aparecen en `example-template-reference.md` y son la única fuente de verdad para estas variables. Todas usan bucles/condicionales (solo Word).

- **RENAPO** `{#external_sources_renapo}{names} {first_surname} {second_surname} {consult_source} {consult_date} {consult_status} {curp} {gender} {birth_date} {nationality} {birth_entity} {evidentiary_document}{/}`
- **PLD (Quién es Quién)** `{#external_sources_aml_validation}` con sub-bloques `{#aml_company}`, `{#aml_administrators}`, `{#aml_key_people}`, `{#aml_shareholders}`, `{#aml_ubos}`, cada uno con `{subjectName}`, `{validatedAt}`, `{status}`, y resultados anidados `{#validationResult}{#providerResponse}{#success}{#data}{NOMBRECOMP}{FULL_NAME}{NAME}{CONTRIBUYENTE}{RFC}{LISTA}{ESTATUS}{ID_PERSONA}{COINCIDENCIA}{CATEGORIA_RIESGO}{SANCION}{EXPEDIENTE}{CAUSA_IRREGULARIDAD}{DISPOSICION}{/}…{/}` + inversos `{^success}No hay información de PLD…{/}`
- **FIEL y Sello digital** `{fiel_scrapedAt}`, `{fiel_status}`, `{#external_sources_fiel_sello}{#scrapper_success}{company_name}{business_type}{rfc}{type}{serial_number}{status}{start_date}{final_date}{/}{/}` + manejo de error `{scrapper_error}`
- **SIGER** `{#external_sources_siger}{consult_source}{consult_date}{consult_status}{company_name}{fme}{fme_status}{federal_entity}{municipality}{registry_office}{#documents}{preCodedForm}{admissionDate}{registrationDate}{documentNumber}{/documents}{/}`
- **INE** `{#external_sources_ine_validation}{consult_source}{consult_date}{consult_status}{is_valid}{cic}{voter_code}{emission_number}{federal_district}{local_district}{ocr_number}{registry_year}{emission_year}{/}`
- **QR de la CSF** `{tax_additonalData_source_qrValidatedAt}` (⚠ typo "additonal" en la referencia), condicional `{#tax_additionalData_source_qrValidation}EXITOSO{/}{^…}NO EXITOSO{/}`, datos `{#tax_additionalData}{#data}{denominacion_o_razon_social}{regimen_de_capital}{fecha_de_constitucion}{fecha_de_inicio_de_operaciones}{situacion_del_contribuyente}{fecha_del_ultimo_cambio_de_situacion}{entidad_federativa}{municipio_o_delegacion}{colonia}{tipo_de_vialidad}{nombre_de_la_vialidad}{numero_exterior}{numero_interior}{cp}{correo_electronico}{al}{regimen}{fecha_de_alta}{/}{/}`

---

## 8. UBOs México / Propietarios reales segundo nivel (referencia)

Tabla accionaria por empresa: `{#ubos_business_0_shareholders}{name} {first_surname} {second_surname} {share_percentage}%{/}`
Personas físicas reales por empresa: `{#ubos_business_0_ubos_person_0_name}{ubos_business_0_ubos_person_0_name} {…_firstSurname} {…_secondSurname} {…_gender} {…_nationality} {…_birthDate} {…_curp} {…_email} {…_mx_address_*} {…_occupation} {…_phone}{/}`
Se indexan por empresa (`ubos_business_0`, `ubos_business_1`, …) y por persona (`ubos_person_0`, `ubos_person_1`, …).

---

## 9. Extracciones personalizadas (customUserPrompts)

Si la cuenta tiene extracciones personalizadas activas, los resultados se exponen como `{sources_<índice>_customUserPrompts_<prompt_id>_<propiedad>}` (ej. `{sources_0_customUserPrompts_mi_revision_monto}`). Solo aplican si la cuenta del cliente las configuró; no asumir que existen.
