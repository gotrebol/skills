# Plantilla de Dictamen de Referencia — Anotada (grounding)

Esta es la plantilla de dictamen **ya configurada** de Trébol (`Formato_Dictamen_Predeterminado.docx`, incluida en esta carpeta). Es la fuente de verdad de **cómo se ve una plantilla bien tokenizada** y la única que documenta variables que no están en la web. Cuando dudes de cómo tokenizar una sección de la plantilla de un cliente, busca aquí la sección equivalente.

> El archivo `.docx` original está en esta misma carpeta. Para inspeccionarlo: `extract-text grounding/Formato_Dictamen_Predeterminado.docx`. Está estructurado como una tabla de Word por filas/celdas.

## Cómo leer esta referencia

Cada sección muestra la **etiqueta del dictamen** y el **token** que va a su lado. Reutilízalos como patrón al configurar plantillas de clientes con secciones análogas.

---

## DATOS GENERALES
- DENOMINACIÓN O RAZÓN SOCIAL: `{legal_businessName}, {legal_businessType}`
- RFC: `{tax_businessTaxId}` · DURACIÓN: `{legal_businessDuration}`
- DOMICILIO SOCIAL: `{legal_businessLegalAddress}` · ACTIVIDAD ECONÓMICA: `{tax_businessMainActivity_economicActivity}`

### Domicilio fiscal
- FUENTE: `{#tax_hasConstancia}Constancia de Situación Fiscal{/}{^tax_hasConstancia}Faltó subir una Constancia de situación Fiscal{/}`
- DOMICILIO COMPLETO: `{fiscalAddress}` · VIGENTE HASTA: `{fiscalAddress_source_date}`

### Domicilio comercial
- FUENTE: `{#comercialAddress}Comprobante de domicilio{/}{^comercialAddress}Faltó subir un comprobante de domicilio{/}`
- DOMICILIO COMPLETO: `{comercialAddress}` · FECHA DEL DOCUMENTO: `{comercialAddress_source_date}` · SERVICIO: `{address_service_type}`

## INFORMACIÓN DE ACTAS (tabla, bucle)
Fila repetible: `{#actaSources}{documentNumber} | {documentDate} | No. {notaryNumber} Nom: {notaryName} | {type} | No. {folioNumber} Fecha: {folioDate} | {notaryState}{/}`
Dentro, explicación de eventos por acta: `{#orderDay}- Eventos de empresa: {EVENTOS_EMPRESA_EXPLICACION}, - Eventos de accionistas: {EVENTOS_ACCIONISTAS_EXPLICACION}, - Eventos de administración: {EVENTOS_ADMINISTRACION_EXPLICACION}, - Eventos de vigilancia: {EVENTOS_VIGILANCIA_EXPLICACION}, - Eventos de firmantes: {EVENTOS_FIRMANTES_EXPLICACION}{/}`

## DATOS DE CONSTITUCIÓN
`{registration_actaNumber}`, `{registration_registrationDate}`, `{registration_city}`, `{registration_notaryNumber}`, `{registration_notaryName}`, RPPC `{registration_folioNumber}`, FECHA RPPC `{registration_folioDate}`
FUENTE: `{#registration_hasConstitutive}Escritura pública número {registration_actaNumber} de fecha {registration_registrationDate}, otorgada ante la fe del Licenciado(a) {registration_notaryName}, titular de la notaría pública número {registration_notaryNumber} de {registration_city}, debidamente inscrita en el registro público de comercio de {registration_city} bajo el folio mercantil electrónico número {registration_folioNumber} de fecha {registration_folioDate}{/}{^registration_hasConstitutive}Faltó subir acta constitutiva{/}`

## APODERADO N DE LA PERSONA MORAL  (1-based; el bloque se repite cambiando `key_person_1` → `key_person_2`, …)
- NOMBRE COMPLETO: `{key_person_1}`
- NACIONALIDAD: `{key_person_1_nationality}` · FECHA DE NACIMIENTO: `{key_person_1_birth_date}`
- SEXO: `{key_person_1_gender}` · CURP: `{key_person_1_external_identities_curp}`
- DIRECCIÓN DEL DOCUMENTO DE IDENTIDAD: `{key_person_1_mx_address_tipo_vialidad} {key_person_1_mx_address_nombre_vialidad} {key_person_1_mx_address_numero_exterior} {key_person_1_mx_address_tipo_interior} {key_person_1_mx_address_numero_interior}, {key_person_1_mx_address_nombre_asentamiento}, {key_person_1_mx_address_municipio_o_ente_territorial} {key_person_1_mx_address_codigo_postal} {key_person_1_mx_address_entidad_federativa} {key_person_1_mx_address_localidad} {key_person_1_mx_address_pais}`
- ESCRITURA PODER: `{key_person_1_source_number}` · FECHA: `{key_person_1_source_date}` · NÚMERO DE NOTARIO: `{key_person_1_source_notaryNumber}`
- NOMBRE DE NOTARIO: `{key_person_1_source_notaryName}` · PLAZA: `{key_person_1_source_notaryCity}, {key_person_1_source_notaryState}`
- RPPC: `{key_person_1_source_folioNumber}` · FECHA RPPC: `{key_person_1_source_folioDate}`
- ROL DEL FIRMANTE: `{key_person_1_role_name}`
- Tabla de poderes (cada uno con CUENTA CON EL PODER / TIPO DE FIRMA / VIGENCIA / LIMITACIONES):
  - ADMINISTRACIÓN: `{key_person_1_power_administration_x_mark}` · `{key_person_1_power_administration_signature_type}` · `{key_person_1_power_administration_duration}` · `{key_person_1_power_administration_limits_description}`
  - CUENTAS BANCARIAS: `{key_person_1_power_bank_accounts_x_mark}` · `_signature_type` · `_duration` · `_limits_description`
  - OTORGAR/DELEGAR/SUSTITUIR PODERES: `{key_person_1_power_delegate_x_mark}` · `_signature_type` · `_duration` · `_limits_description`  ⚠ *ojo: la referencia usa `delegate`, que no está en la doc web (que lista 5 poderes: administration, assets_management, loans, bank_accounts, lawsuits_and_collections). Verificar contra una verificación real.*
  - TÍTULOS Y OPERACIONES DE CRÉDITO: `{key_person_1_power_loans_x_mark}` · `_signature_type` · `_duration` · `_limits_description`
  - DOMINIO: `{key_person_1_power_assets_management_x_mark}` · `_signature_type` · `_duration` · `_limits_description`
  - PLEITOS Y COBRANZAS: `{key_person_1_power_lawsuits_and_collections_x_mark}` · `_signature_type` · `_duration` · `_limits_description`
- FUENTE: `{#key_person_1_source_number}Escritura pública número {key_person_1_source_number} de fecha {key_person_1_source_date}…{/}{^key_person_1_source_number}No se encontraron poderes para el firmante #1{/}`

## ACCIONISTAS PRIMER NIVEL (tabla, bucle)
Fila repetible: `{#shareholders}{name} | {fixed_value} | {fixed_shares} | {variable_value} | {variable_shares} | {share_percentage} %{/}`
Fila TOTAL: `{capital_fixedValue} | {capital_fixedShares} | {capital_variableValue} | {capital_variableShares} | 100 %`
FUENTE: `{#shareholders_source_acta_number}Escritura pública número {shareholders_source_acta_number} de fecha {shareholders_source_source_date}…{/}{^shareholders_source_acta_number}No se encontró el cuadro accionario más reciente{/}`

## PROPIETARIOS REALES (ACCIONISTAS SEGUNDO NIVEL) — UBOs MX
Por empresa (`ubos_business_0`, `ubos_business_1`, …):
- Tabla accionaria: `{#ubos_business_0_shareholders}{name} {first_surname} {second_surname} | {share_percentage} %{/}`
- Datos generales por persona (`ubos_person_0`, `ubos_person_1`, …): `{#ubos_business_0_ubos_person_0_name}{ubos_business_0_ubos_person_0_name} {…_firstSurname} {…_secondSurname} | {…_gender} | {…_nationality} | {…_birthDate} | {…_curp} | {…_email} | {…_mx_address_*} | {…_occupation} | {…_phone}{/}`

## CONSEJO DE ADMINISTRACIÓN (tabla, bucle)
`{#board_members}{names} {firstSurname} {secondSurname} | {roleName}{/}`
`{#executives}{names} {firstSurname} {secondSurname} | {roleName}{/}`
FUENTE: `{#board_member_0_names}Escritura pública número {board_member_0_source_documentNumber} de fecha {board_member_0_source_documentDate}, otorgada ante la fe del licenciado {board_member_0_source_extraFields_notaryName}, titular de la notaría pública número {board_member_0_source_extraFields_notaryNumber} de {board_member_0_source_extraFields_notaryCity}, {board_member_0_source_extraFields_notaryState}{/}{^board_members_0_names}{/}{#executive_0_names}…{/}{^executive_0_names}{/}`
⚠ *La referencia escribe el inverso como `{^board_members_0_names}` (plural) mientras el directo es `{board_member_0_names}` (singular). Posible errata: el individual es `board_member_0`. Verificar.*

## CONSULTAS ADICIONALES
- **RENAPO**: `{#external_sources_renapo}NOMBRE DE PERSONA: {names} {first_surname} {second_surname} FUENTE: {consult_source} FECHA: {consult_date} ESTADO: {consult_status} RESULTADO: {curp} {gender} {birth_date} {nationality} {birth_entity} {evidentiary_document}{/}`
- **PLD (Quién es Quién)**: `{#external_sources_aml_validation}` con sub-bloques `{#aml_company}`, `{#aml_administrators}`, `{#aml_key_people}`, `{#aml_shareholders}`, `{#aml_ubos}`. Cada uno: `{subjectName} {validatedAt} {status}` y resultado anidado `{#validationResult}{#providerResponse}{#success}{#data}{NOMBRECOMP}{FULL_NAME}{NAME}{CONTRIBUYENTE}{RFC}{LISTA}{ESTATUS}{ID_PERSONA}{COINCIDENCIA}{CATEGORIA_RIESGO}{SANCION}{EXPEDIENTE}{CAUSA_IRREGULARIDAD}{DISPOSICION}{/}{/}` + inversos `{^success}No hay información de PLD…{/}` por nivel.
- **E.FIRMA Y SELLO DIGITAL**: FECHA `{fiel_scrapedAt}` ESTADO `{fiel_status}` · `{#external_sources_fiel_sello}{#scrapper_success}{company_name} {business_type} {rfc} {type} {serial_number} {status} {start_date} {final_date}{/}{/}` · error `{#external_sources}{^scrapper_success}RFC: {rfc} Estado: {scrapper_error}{/}{/}` · `{^external_sources_fiel_sello}No hay certificados de FIEL y SELLO activos{/}`
- **SIGER**: `{#external_sources_siger}FUENTE: {consult_source} FECHA: {consult_date} ESTADO: {consult_status} Razón social: {company_name} FME: {fme} Estatus FME: {fme_status} Entidad: {federal_entity} Municipio: {municipality} Oficina: {registry_office} {#documents}{preCodedForm} {admissionDate} {registrationDate} {documentNumber}{/documents}{/}`
- **INE**: `{#external_sources_ine_validation}FUENTE: {consult_source} FECHA: {consult_date} ESTADO: {consult_status} {is_valid} {cic} {voter_code} {emission_number} {federal_district} {local_district} {ocr_number} {registry_year} {emission_year}{/}`
- **QR DE LA CSF**: FECHA `{tax_additionalData_source_qrValidatedAt}` ESTADO `{#tax_additionalData_source_qrValidation}EXITOSO{/}{^tax_additionalData_source_qrValidation}NO EXITOSO{/}` RESULTADO `{#tax_additionalData}{#data}Denominación: {denominacion_o_razon_social} Régimen de capital: {regimen_de_capital} Fecha de constitución: {fecha_de_constitucion} … {fecha_de_alta}{/}{/}`

---

## Erratas/inconsistencias detectadas en la propia referencia (no replicarlas)

1. **Espacio dentro de llaves**: `{ key_person_1_source_notaryState}` — el espacio inicial rompe el reemplazo. Escribir sin espacio.
2. **Mayúsculas**: `{KEY_PERSON_2_ROLE_NAME}` en una celda — usar minúsculas `{key_person_2_role_name}`.
3. **Inverso plural** `{^board_members_0_names}` vs individual `board_member_0_names`.
4. **Poder `delegate`** en apoderados, no listado en la doc web de 5 poderes.
5. **Typo en el Word original** `tax_additonalData_source_qrValidatedAt` (falta la "i") — corregido arriba a `tax_additionalData_source_qrValidatedAt` (grafía confirmada en `corrections.md`).

Estas son justamente las cosas que el **ingeniero de variables** (persona) debe vigilar y que se registran en `corrections.md` cuando se confirme cuál es la forma correcta contra una verificación real.
