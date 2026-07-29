# Claude Sessions — Project Notes

Ez a fájl a **közös emlékezet** a projekten dolgozó Claude sessionök között:
- **Claude Code** (lokális, kód-implementációhoz) — olvassa és frissíti
- **Claude claude.ai-n** (design, guide-készítés, code review) — kap egy másolatot chatben

Utolsó frissítés: **2026-07-28** (`wsdl.SchemaPropertyEnum` legacy tábla bejegyzés)

## 🔄 Szinkronizációs állapot (ki írt utoljára / ki dolgozta fel)

| Mező | Érték |
|---|---|
| **Utoljára írt** | Claude Code |
| **Írás dátuma** | 2026-07-28 |
| **Mit írt (röviden)** | `wsdl.SchemaPropertyEnum` legacy tábla bejegyzés az "Ismert rések" közé (felhasználói visszakérdezésre) |
| **Claude Code feldolgozta?** | ✅ 2026-07-28 (ő maga írta) |
| **Claude Opus (claude.ai) feldolgozta?** | ✅ 2026-07-28 (első sikeres GitHub-fetch alapján) |

**Napló** (legutóbbi írások, legújabb felül — kb. 10 sort érdemes megtartani, a régebbieket törölni):

| Dátum | Ki írt | Mit (röviden) | Claude Code feldolgozta | Claude Opus feldolgozta |
|---|---|---|---|---|
| 2026-07-28 | Claude Code | `SchemaPropertyEnum` legacy tábla bejegyzés | ✅ (írta) | ✅ 2026-07-28 |
| 2026-07-24 | Claude Code | WSDL W5 lezárva — projekt funkcionálisan kész | ✅ (írta) | ✅ 2026-07-28 (első sikeres GitHub-fetch alapján) |
| 2026-07-24 | Claude Code | WSDL W2 lezárva, `ModuleEntitiesID` idempotencia-bug fix | ✅ (írta) | ✅ 2026-07-28 (első sikeres GitHub-fetch alapján) |

---

## 🎯 Projekt kontextus

Három .NET 8 / C# alkalmazás pipeline UniCredit banking integrációhoz:

```
┌─────────────────┐    ┌──────────────────┐    ┌────────┐
│ InterfaceImporter│──▶│ InterfaceToModel │──▶│ BE2WS  │
└──────────────────┘    └──────────────────┘    └────────┘
   swagger.*,               model.*                model.*
   wsdl.*                   (normalized)          → API mapping
   (raw structured)
```

- **InterfaceImporter** — parseoli a Swagger/OpenAPI és WSDL fájlokat a `swagger.*` és `wsdl.*` sémákba
- **InterfaceToModel** — a `swagger.*`/`wsdl.*`-ból generál `model.*` entitásokat (Class, Field, Operation, stb.)
- **BE2WS** — API-oldali mapping generálás a BackendInterface operationekből

Tech stack: .NET 8 + C#, Dapper, SQL Server, Microsoft.OpenApi.Models (swagger parsing), System.Xml.Linq (WSDL parsing).

---

## 🔑 Fontos névkonvenciók (eltérések a spec-ektől)

Néhány guide a projektet nem tükrözi 100%-ban. Az eltérések itt vannak dokumentálva:

### Séma névhasználat

| Guide/spec neve | Valós név a projektben | Megjegyzés |
|---|---|---|
| `wsdl.SchemaElement` (tábla) | `wsdl.Schema` | Meglévő tábla, bővítve nem duplikálva |
| `wsdl.SchemaElementField` (tábla) | `wsdl.SchemaProperty` | Meglévő tábla, bővítve nem duplikálva |
| `WsdlSchemaElement` (C# rekord) | `WsdlSchema` | InterfaceToModel-ben |
| `WsdlSchemaElementField` (C# rekord) | `WsdlSchemaProperty` | InterfaceToModel-ben |

### FK oszlopnevek az új XSD táblákban — ÁTNEVEZVE (2026-07-23)

Az öt új XSD tábla (`SchemaAttribute`, `SchemaAttributeGroupRef`, `SchemaGroupRef`, `SchemaElementRestriction`, `SchemaElementEnum`) eredetileg a spec szerinti `SchemaElementID`/`SchemaElementFieldID` neveket kapták, miközben a FK a valós `wsdl.Schema`/`wsdl.SchemaProperty` táblákra mutatott (nem létező `SchemaElement`/`SchemaElementField` táblára utaló, megtévesztő név). **Ezt átneveztük**:

- `SchemaElementID` → **`SchemaID`** (FK → `wsdl.Schema.ID`)
- `SchemaElementFieldID` → **`SchemaPropertyID`** (FK → `wsdl.SchemaProperty.ID`)
- `SchemaAttributeID` — változatlan (ez a név helyes volt, `wsdl.SchemaAttribute.ID`-re mutat)
- `TypeSchemaElementID` (mező-/attribútum-típus feloldás `SchemaProperty`/`SchemaAttribute`-on) — **változatlan maradt**, ez külön szemantika (a mező TÍPUSÁT reprezentáló Schema-ra mutat, nem a tulajdonost jelöli)

Migrációk (mindkettő `SIT_SOLAR_STAGING`-en fut le, mert `wsdl.*` séma):
- `InterfaceImporter/sql/migrations/wsdl_extension_phase2c_rename_columns.sql` — fő átnevező script (javított sorrenddel: CHECK constraint drop → sp_rename → CHECK constraint recreate)
- `InterfaceImporter/sql/migrations/wsdl_extension_phase2d_rename_columns_fix.sql` — kiegészítő javítás, mert a 2c első futása elakadt a `SchemaElementRestriction`/`SchemaElementEnum` táblákon (az "ExactlyOneOwner" CHECK constraint blokkolta az `sp_rename`-t — **tanulság: CHECK constraint mindig előbb drop-olva legyen, mint a rá hivatkozó oszlop átnevezése**)

Kód érintve mindkét projektben: `WsdlEntities.cs`, `WsdlMapper.cs`, `WsdlRepository.cs` (InterfaceImporter); `WsdlEntities.cs`, `WsdlReader.cs`, `WsdlFieldMapper.cs` (InterfaceToModel). Build + end-to-end teszt (CreditManagement WSDL) zöld az átnevezés után.

### wsdl.Type tábla

A séma referencia doksi (`wsdl_schema_reference.md`) egy idealizált verzióját írja le. A valós DDL eltér:

| Doksi szerint | Valóság |
|---|---|
| `TargetNamespace NVARCHAR(1024) NOT NULL` | `Namespace NVARCHAR(1024) NULL` |
| `ElementFormDefault VARCHAR(20) NULL` | nincs (még) |
| `AttributeFormDefault VARCHAR(20) NULL` | nincs (még) |
| `SchemaContent XML NOT NULL` | `SchemaContent XML NULL` |
| `Documentation NVARCHAR(MAX) NULL` 🔵 | **hozzá lett adva F1-ben** |

---

## ✅ Implementálva

### 2026-07 (nagy iteráció)

- ✅ **BE2WS**: `--apiname` CLI paraméter + `ApplicationEntities` bővítés
- ⚠️ ~~InterfaceToModel W1: WSDL bug fix + Endpoint gyártás~~ — **ez a bejegyzés téves volt, lásd Eltérésnapló 2026-07-23.** A valós, teljes implementáció lentebb "Swagger 1. iteráció + WSDL W1" néven szerepel.
- ✅ **InterfaceImporter swagger extension**: `SchemaProperty` 4 új mező
  - `EnumValues` (JSON array), `IsDeprecated`, `IsReadOnly`, `IsWriteOnly`
- ✅ **InterfaceToModel swagger 2b**: validációs mezők + `FieldEnumValue` tábla + Class metaadatok
  - `BackendInterfaceField`: 11 új mező
  - Új `FieldEnumValue` tábla
  - `Class`: 4 új mező
- ✅ **InterfaceImporter WSDL Fázis 1**: SOAP-oldali strukturált tárolás
  - Documentation minden meglévő táblán
  - Új `Import` tábla
  - Új `BindingOperationMessage/Header/Fault` táblák
  - Operation +5 új mező
- ✅ **InterfaceImporter WSDL Fázis 2**: XSD strukturált parsing
  - `wsdl.Schema` (a.k.a. SchemaElement) +10 mező
  - `wsdl.SchemaProperty` (a.k.a. SchemaElementField) új mezők
  - 5 új XSD tábla (Restriction, Enum, Attribute, AttrGroupRef, GroupRef)
  - Kétfázisú XSD parser (top-level regisztráció + reference feloldás)
  - Utólagos kiegészítés: `SchemaProperty` hiányzó `DefaultValue`/`FixedValue`/`Form` mezői pótolva (`wsdl_extension_phase2b_schemaproperty_fields.sql`)
  - Utólagos kiegészítés: inline (névtelen) `complexType` rekurzív feldolgozása mezőkön belül (szintetizált név: `{szülő}{mező}ComplexType`)
- ✅ **InterfaceToModel W3**: XSD strukturált feldolgozás → model (2026-07-23)
  - Új `model.BackendInterfaceSchema` tábla (`wsdl.XsdSchemaBlock` → egy sor/névtér)
  - `BackendInterfaceClass`: +`Compositor`, +`Form`
  - `BackendInterfaceField`: +8 mező (`FixedValue`, `Form`, `MinExclusive`, `MaxExclusive`, `WhiteSpace`, `ExplicitTimezone`, `IsXmlAttribute`, `IsProhibited`)
  - `WsdlFieldMapper` teljes újraírás: restriction/enum feloldás (inline → névvel ellátott simpleType fallback), `xsd:attribute` külön mezőként (`IsXmlAttribute`), `xsd:group`/`attributeGroup` rekurzív flatten, lineáris `OrdinalNumber` osztályonként
  - Migráció: `InterfaceToModel/sql/migrations/W3_wsdl_xsd_structured.sql` — **SANDBOX-on ÉS MAIN-en is le kell futtatni** (lásd lentebb "MAIN DB migrációk" szakasz)
  - Menet közben talált és javított bug: `WsdlClassMapper` névtér-ütközés (azonos nevű típus több névtérben → ugyanabba a target Class-ba olvadt, duplikált `OrdinalNumber`-t okozva). Javítás: sorrend-független, névtér-tudatos osztálynév-disambiguáció (lásd Eltérésnapló).
  - Végponttól végpontig tesztelve a CreditManagement WSDL-lel: 388 class, 1040 field, 29 operation, 36 BackendInterfaceSchema, 32 FieldEnumValue, 0 duplikált ordinal
- ✅ **Swagger 1. iteráció + WSDL W1 összevonva** (2026-07-23) — a két guide körkörösen egymást feltételezte előfeltételként, ezért egyben lett implementálva (lásd Eltérésnapló)
  - `--environmentid` CLI paraméter + `CatalogEnvironments` lookup (`MainDatabaseReader`, a `CatalogEntitiesID` mintáját követve)
  - Új `model.OperationParameter` tábla — `rest-path`/`rest-query`/`rest-header`/`rest-cookie`/`soap-header-in`/`soap-header-out` Location konvencióval (ez egyben lezárja a WSDL W4-et is, mert a konvenció eleve helyesen jön létre)
  - Új `ModelBackendInterfaceEndpoint` rekord + `WsdlEndpointMapper`/`SwaggerEndpointMapper` — a `model.BackendInterfaceEndpoint` és `model.CatalogEnvironments` alap táblák már léteztek a DB-ben (csak a kód hiányzott teljesen)
  - `BackendInterface` +10 metaadat mező (`DocumentID`, `DescriptorVersion`, `Title`, `Version`, `Description`, `ContactName`, `ContactEmail`, `ContactUrl`, `LicenseName`, `LicenseUrl`) — swagger oldalon `swagger.Document`-ből, WSDL oldalon `wsdl.Definition`-ből
  - `BackendInterface.Namespace` mérete 250→1024 karakter (a WSDL targetNamespace hosszabb lehet)
  - `WsdlOperationMapper` 5 hibajavítás: `ResourcePath` többé nem a SOAP address-t tartalmazza (az `BackendInterfaceEndpoint.EndpointURL`-be kerül), `ContentType`/`AcceptType` a SOAP verzió alapján (`text/xml` vs `application/soap+xml`), `RequestXmlRootElement` az input MessagePart-ból (nem az operation névből), `SoapStyle` propagálása `BindingOperation`/`Binding`-ből
  - `SwaggerOperationMapper` bővítve `MapOperationParameters`-szel
  - Migráció: `InterfaceToModel/sql/migrations/W1_swagger1_endpoint_operationparameter.sql` — **SANDBOX-on ÉS MAIN-en is le kell futtatni**
  - Menet közben talált és pótolt hiányosságok: `WsdlService`/`WsdlBinding` rekordok és beolvasásuk teljesen hiányoztak InterfaceToModel-ben (pedig a `wsdl.Service`/`wsdl.Binding` táblák már léteztek forrás oldalon Fázis 1 óta); `swagger.Parameter` `Deprecated`/`AllowEmpty`/`Example` mezői sosem lettek beolvasva; `swagger.Document`/`Server`/`ServerVariable` egyáltalán nem volt beolvasva InterfaceToModel oldalon
- ✅ **WSDL W2**: SOAP binding részletek → model (2026-07-24) — a guide szerint ez az utolsó nagy WSDL iteráció, lezárja a SOAP binding oldalt
  - `BackendInterfaceOperation`: +8 mező (`RequestBodyUse`, `ResponseBodyUse`, `RequestBodyNamespace`, `ResponseBodyNamespace`, `RequestBodyParts`, `ResponseBodyParts`, `RequestBodyEncodingStyle`, `ResponseBodyEncodingStyle`) — a `wsdl.BindingOperationMessage`-ből (`MessageRole = input/output`)
  - `OperationParameter`: +3 mező (`SoapUse`, `SoapNamespace`, `SoapEncodingStyle`) — SOAP header paraméterekhez
  - `BackendInterfaceField`: +3 mező (`SoapUse`, `SoapNamespace`, `SoapEncodingStyle`) — fault wrapper mezőkhöz
  - Új `WsdlHeaderParameterMapper`: `wsdl.BindingOperationHeader` → `model.OperationParameter` (`soap-header-in`/`soap-header-out` Location, a W1-ben már bevezetett konvenciót töltve fel ténylegesen)
  - `WsdlFieldMapper.MapOperationFaults` (új metódus): operation fault-ok → wrapper class mezők (`{OperationName}{FaultName}Fault` néven), a response wrapper class-ba
  - Pipeline bekötve: `WsdlImportService.MapHeaderParameters`/`MapOperationFaults`, hívva `ImportService.RunAsync`-ből a `MapEndpoints` után
  - Migráció: `InterfaceToModel/sql/migrations/W2_wsdl_soap_binding_details.sql` — **SANDBOX-on ÉS MAIN-en is le kell futtatni**
  - Menet közben talált és pótolt hiányosságok: `WsdlOperationFault` rekord + beolvasás teljesen hiányzott InterfaceToModel-ben (pedig `wsdl.OperationFault` forrás tábla létezett Fázis 1 óta); a guide feltételezett `WsdlBindingOperationHeader` oszlopnevek (`MessageRole`, `MessageID` mint Guid FK, `PartName`, `HeaderUse`, `HeaderNamespace`) eltértek a valós `wsdl.BindingOperationHeader` táblától (`Direction`, `MessageName` mint nyers string, `Part`, `Use`, `Namespace`) — lásd Eltérésnapló
  - Menet közben talált és **javított** külön bug (nem W2-höz tartozik, a W1 `BackendInterfaceEndpoint`-hoz): `ModuleEntitiesID` idempotencia-hiba — lásd Eltérésnapló "2026-07-24 — ModuleEntities idempotencia-bug"
  - **Végponttól végpontig letesztelve** a CreditManagement WSDL-lel (2026-07-24, `WsdlW3Test3` modulon, második futtatásként ugyanarra a modulra): 388 class, 1098 field (58 új fault mező), 29 operation (mind `SoapStyle=document`, `RequestBodyUse`/`ResponseBodyUse=literal`), 29 `soap-header-in` OperationParameter, fault mezők helyesen `{OperationName}{FaultName}Fault` néven (pl. `executeManualForwardToStructureServiceExceptionFault`), 0 duplikált `OrdinalNumber`
- ✅ **WSDL W5**: Namespace tárolás (2026-07-24) — **ez volt a projekt utolsó iterációja**, a guide szerint ezzel a projekt funkcionálisan kész
  - Új `model.BackendInterfaceNamespace` tábla — xmlns:* prefix→URI deklarációk a WSDL Definition-ről
  - Új `WsdlNamespace` C# rekord + `WsdlReader` beolvasás (`wsdl.Namespace`, oszlopnevek 1:1 egyeztek a guide feltételezésével — lásd Eltérésnapló, nem minden guide-hiba ismétlődött meg)
  - Új `WsdlNamespaceMapper` — 1:1 leképezés dedup-pal (első előfordulás nyer, prefix szerint rendezve)
  - Pipeline bekötve: `WsdlImportService.MapNamespaces`, hívva `ImportService.RunAsync`-ből a `MapOperationFaults` után
  - Migráció: `InterfaceToModel/sql/migrations/W5_wsdl_namespace.sql` — **SANDBOX-on ÉS MAIN-en lefuttatva** (2026-07-24)
  - **Guide-hiba korrigálva**: a guide a `BackendInterfaceNamespace` táblát `[BackendInterfaceID]` FK oszloppal tervezte, ami `model.BackendInterface.[ID]`-re mutatott volna — de a `model.BackendInterface` táblának **nincs `ID` oszlopa** (valós PK: `ProjectID`+`ModuleID`). Ehelyett `[ModuleID]` FK oszlopot használtunk, ugyanúgy mint a W3-as `model.BackendInterfaceSchema` táblánál — lásd Eltérésnapló
  - **Végponttól végpontig letesztelve** a CreditManagement WSDL-lel (`WsdlW3Test3` modulon): 40 namespace mapped (forrás=cél, nem volt dedup), sanity check OK (`xsd→http://www.w3.org/2001/XMLSchema`, `soap→http://schemas.xmlsoap.org/wsdl/soap/`, `tns`, `wsdl` mind jelen), regresszió stabil (388 class, 1098 field, 29 operation — változatlan a W2-höz képest)

---

## ⚠️ MAIN DB migrációk

**Fontos, könnyen elfelejthető szabály**: a `ModelWriter` upsert logikája (`UpsertAsync`/`UpsertClassAsync`) a target DB-ben nem talált osztályokat/mezőket a **MAIN DB-ből próbálja másolni** (pl. "Object" alaposztály, elemi Java típusok mint `java.lang.String`). Ehhez a `mainFindSql` **ugyanazokat a bővített oszlopokat** kérdezi le, mint a target — vagyis:

**Minden `model.*` sémát bővítő migrációt (swagger 2b, W3, jövőbeli iterációk) a SANDBOX/STAGING mellett a `SIT_SOLAR_MAIN`-en is le kell futtatni**, különben `Invalid column name` hibával elszáll az import, amint egy elemi típust vagy alaposztályt a target nem talál és MAIN-ből próbálja másolni.

Eddig lefuttatva MAIN-en: `swagger_2b_field_validation.sql`, `W3_wsdl_xsd_structured.sql`, `W1_swagger1_endpoint_operationparameter.sql`, `W2_wsdl_soap_binding_details.sql`, `W5_wsdl_namespace.sql`.

---

## ⏳ Következő lépések (prioritási sorrend)

**A projekt funkcionálisan kész** (2026-07-24, WSDL W5 lezárva — ez volt a guide szerint a projekt utolsó iterációja). Maradó feladatok a guide 5. szakasza szerint:

1. **Doksik frissítése** a `_CLAUDE_NOTES.md` Eltérésnaplója alapján:
   - `wsdl_schema_reference.md` — valós séma tükrözése (Schema/SchemaProperty nevek, SchemaID/SchemaPropertyID FK-k, wsdl.Type valós DDL, wsdl.BindingOperationHeader valós oszlopnevek)
   - `wsdl_to_model_full_spec.md` — valós implementáció (pl. `BackendInterfaceNamespace`/`BackendInterfaceSchema` `ModuleID` FK, nem `BackendInterfaceID`, mert `model.BackendInterface`-nek nincs `ID` oszlopa)
   - `iteration_w*.md` — vagy elavultak, vagy javítandók
2. Ez a lista **nem tartalmaz** funkcionális kódmunkát — csak dokumentáció-karbantartás.

### Ismert, még nem lefedett rések (nem blokkolók, csak feljegyezve)

- **Namespace-szűrés túl szigorú lehet** a `WsdlClassMapper.MapInterfaceClasses`-ben — csak azok a séma-típusok kapnak Class-t, amiknek a névtere közvetlenül szerepel egy message part-on. Mélyen beágyazott, nevesített enum/simpleType típusok (amikre csak egy nested field mutat, nem közvetlenül egy message part) emiatt kimaradhatnak — a `TypeSchemaElementID` ilyenkor nem oldódik fel `context.WsdlSchemaToClassID`-n keresztül. A CreditManagement teszt WSDL-ben ez nem okozott adatvesztést (minden feldolgozott mező helyesen resolveolt), de más WSDL-nél problémát okozhat.

- **`wsdl.SchemaPropertyEnum` — legacy, mára fölösleges tábla InterfaceImporter oldalon** (2026-07-28 derült ki, egy visszakérdezésre válaszolva). Mező-szintű `xsd:enumeration` értékeket tárol, kizárólag `SchemaPropertyID`-vel kulcsolva (nincs polimorfizmus, nincs `Documentation`) — a `WsdlMapper.CollectEnumValues` tölti (`InterfaceImporter/src/Parsing/WsdlMapper.cs:662`). Amikor a Fázis 2 migráció bevezette a polimorf `wsdl.SchemaElementEnum` táblát (owner: `SchemaID`/`SchemaPropertyID`/`SchemaAttributeID` — ezt használja a W3 `WsdlFieldMapper`), a migráció kommentje szerint (`wsdl_extension_phase2.sql:236-239`) **tudatos döntés volt nem törölni/kiváltani** a régi táblát — `CollectEnumValues` (→ `SchemaPropertyEnum`) és `AddEnumValues` (→ `SchemaElementEnum`) azóta is **ugyanarra az inline restriction node-ra** párhuzamosan fut minden mezőnél (`WsdlMapper.cs` 554. és 585. sor). **InterfaceToModel sosem olvassa a `SchemaPropertyEnum`-ot** (ellenőrizve: nincs rá hivatkozás) — a `model.*` oldalon ez holt adat, amit az InterfaceImporter feleslegesen ír tele minden importnál. Nem blokkoló, csak takarítási jelölt, ha valaha az InterfaceImporter oldalt karbantartják.

---

## 📋 Eltérésnapló (chronological)

Ide kerülnek a fejlesztés közben derülő eltérések a specekhez képest. Új bejegyzés minden inkonzisztenciához:

### 2026-07-08 — WSDL Fázis 2

**Guide feltételezés**: `wsdl.SchemaElement` és `wsdl.SchemaElementField` új táblák.

**Valóság**: már léteztek `wsdl.Schema` és `wsdl.SchemaProperty` néven.

**Döntés**: bővítettük a meglévőt új mezőkkel. Az 5 új XSD tábla FK oszlopneveit eredetileg megtartottuk (`SchemaElementID` stb.), de a FK targeteket a valós táblákra irányítottuk (Opció B). **2026-07-23-án ezt átneveztük Opció A-ra** — lásd lentebb.

### 2026-07-22 — `wsdl.SchemaProperty` hiányzó mezők (InterfaceImporter WSDL Fázis 2)

**Hiba**: a Fázis 2 migráció a `wsdl.SchemaProperty`-hez csak a `TypeSchemaElementID`-t adta hozzá, a mező-szintű `default`/`fixed`/`form` XSD attribútumokat (`DefaultValue`, `FixedValue`, `Form`) kifelejtettük — pedig az InterfaceToModel W3 guide-ja ezekre számított.

**Észlelés módja**: InterfaceToModel W3 futtatásakor Dapper materializációs hiba (`Invalid column name 'DefaultValue'. 'FixedValue'.`).

**Javítás**: `wsdl_extension_phase2b_schemaproperty_fields.sql` — utólagos `ALTER TABLE`. Kód: `SchemaPropertyEntity`, `WsdlMapper.CollectFields`, `WsdlRepository` bővítve.

### 2026-07-23 — FK oszlopnév átnevezés (`SchemaElementID`→`SchemaID`, `SchemaElementFieldID`→`SchemaPropertyID`)

**Felismerés**: chat-es code review rákérdezett, hogy a Fázis 2 XSD táblák FK oszlopnevei miért `SchemaElement*`, ha nincs `SchemaElement` tábla. Megerősítettük: Opció B (működik, de megtévesztő név).

**Döntés**: átnevezés Opció A-ra (`SchemaID`/`SchemaPropertyID`), mert a névválasztás hosszú távon többet árt (félrevezeti a jövőbeli olvasókat), mint amennyi kockázatot az átnevezés hoz.

**Buktató**: az `sp_rename` megtagadja egy oszlop átnevezését, ha egy CHECK constraint hivatkozik rá ("participates in enforced dependencies") — ez blokkolta a `SchemaElementRestriction`/`SchemaElementEnum` táblák (`ExactlyOneOwner` CHECK) átnevezését az első futáskor. **Szabály jövőre**: CHECK constraint mindig előbb drop-olva legyen, utána `sp_rename`, a végén CHECK recreate.

**Migrációk**: `wsdl_extension_phase2c_rename_columns.sql` (javítva) + `wsdl_extension_phase2d_rename_columns_fix.sql` (a részlegesen elakadt 2 tábla befejezése).

### 2026-07-23 — `WsdlClassMapper` névtér-ütközés (InterfaceToModel W3 tesztelés közben)

**Hiba**: a `WsdlClassMapper` az osztálynevet kizárólag a séma `Name`-jéből képezte, névtér-figyelmen kívül hagyással. A CreditManagement WSDL viszont rengeteg azonos nevű típust definiál különböző névterekben (párhuzamos WSFD/V1 variánsok — pl. `createCreditProposalRequest` 5 különböző névtérben). Ezek egyetlen target `Class`-ba olvadtak össze az upsert során, de a mezőik külön-külön 1-től induló `OrdinalNumber`-t kaptak → `CK_BackendInterfaceField_OrdinalNumber` (egyediségi) constraint-ütközés.

**Miért csak most jött elő**: korábban `OrdinalNumber` mindig `NULL` volt (nem volt kitöltve), a W3 vezette be a tényleges kitöltést — ez hozta felszínre a már meglévő, rejtett hibát.

**Javítás**: sorrend-független (determinisztikus, újraimportáláskor stabil) névtér-tudatos disambiguáció — ha egy séma `Name`-je több mint egyszer fordul elő a feldolgozandó sémák között, a névtér URI utolsó 2 path-szegmensét (PascalCase, "xsd"/"wsdl" kiszűrve) hozzáfűzzük az osztálynévhez. Végső biztonsági háló: ha ez is ütközik, számláló-suffix. Lásd `WsdlClassMapper.ExtractNamespaceSuffix`.

### 2026-07-23 — Téves "InterfaceToModel W1" bejegyzés + körkörös guide-függőség

**Hiba**: ez a fájl korábban ✅-ként jelölte az "InterfaceToModel W1"-et (endpoint gyártás, `--environmentid`, `CatalogEnvironments`). Amikor a swagger 1. iteráció guide-ja ezt előfeltételként várta, alapos kereséssel (subagent-tel ellenőriztetve) kiderült: **a kódban semmi nyoma nem volt** — se `BackendInterfaceEndpoint` tábla/rekord, se `--environmentid` paraméter, se `CatalogEnvironments` hivatkozás, se endpoint mapper. A bejegyzés vagy tervezett-de-soha-meg-nem-valósult volt, vagy elveszett egy korábbi session-ben, anélkül hogy ez a fájl frissült volna.

**Tanulság — bízz, de ellenőrizz**: mielőtt egy guide "előfeltétele már kész"-nek jelölt munkára építesz, **ellenőrizd a tényleges kódban**, ne csak ebben a fájlban. Ez a fájl emlékeztető, nem garancia — előfordulhat hogy elavul vagy téved.

**Másodlagos felfedezés — körkörös függőség a guide-okban**: a `swagger_1_guide.md` a WSDL W1-et várta előfeltételként (`iteration_w1_endpoint_operation.md`), de MAGA a WSDL W1 guide is a swagger 1-et várta előfeltételként ("A swagger 1. iteráció végrehajtva — az `--environmentid` CLI paraméter... már létezik"). Egyik guide sem volt ténylegesen végrehajtható a másik nélkül a leírás szerint.

**Feloldás**: ellenőriztük az adatbázist közvetlenül (nem a kódot) — kiderült hogy a `model.BackendInterfaceEndpoint` és `model.CatalogEnvironments` **alap táblák már léteztek** (SANDBOX-on és MAIN-en is, valószínűleg az eredeti model.* séma része, még az iterációk előttről). Ez feloldotta a kört: az infrastruktúra (táblák) megvolt, csak a C# kód hiányzott mindkét oldalról — így a két guide-ot egyben, egyetlen implementációként végrehajtottuk ("Swagger 1. iteráció + WSDL W1 összevonva", lásd fentebb).

### 2026-07-24 — `wsdl.BindingOperationHeader` oszlopnevek eltértek a WSDL W2 guide feltételezésétől

**Guide feltételezés**: `MessageRole`, `MessageID` (Guid, feloldott FK), `PartName`, `HeaderUse`, `HeaderNamespace`, `HeaderEncodingStyle`, `SortOrder`, `Documentation` oszlopok.

**Valóság**: a tábla (a korábbi WSDL Fázis 1 session-ben jött létre) `Direction`, `MessageName` (nyers string, NEM feloldott Guid), `Part`, `Use`, `Namespace`, `HeaderEncodingStyle`, `SortOrder`, `Documentation` oszlopokkal rendelkezik.

**Javítás**: a C# rekord (`WsdlBindingOperationHeader`) a szemantikus neveket kapta (`Direction`, `MessageName`, `PartName`, `HeaderUse`, `HeaderNamespace`), a `WsdlReader.cs` SQL SELECT-je pedig alias-olja a valós oszlopneveket ezekre (`[Part] AS [PartName]`, `[Use] AS [HeaderUse]`, `[Namespace] AS [HeaderNamespace]`). Mivel `MessageName` nyers string és nem `MessageID`, a `WsdlHeaderParameterMapper` egy `messageIdByName` reverse-lookup dictionary-t épít a `wsdl.MessageNames`-ből (ami `Dictionary<Guid,string>` ID→Name) a feloldáshoz.

**Tanulság**: ugyanaz mint a 2026-07-08-as bejegyzésnél — guide feltételezéseket mindig a tényleges DB séma ellen kell ellenőrizni, különösen ha a táblát egy korábbi (más guide alapján futtatott) session hozta létre.

### 2026-07-24 — `WsdlOperationFault` teljesen hiányzott InterfaceToModel-ből (WSDL W2 előfeltétel-ellenőrzés közben)

**Felismerés**: a WSDL W2 guide előfeltétel-ellenőrzése során (a felhasználó kifejezett kérésére: "ellenőrizd a kódban hogy WSDL Fázis 1 valóban létezik") kiderült, hogy bár a `wsdl.OperationFault` forrás tábla és `InterfaceImporter`-beli `OperationFaultEntity` (`ProjectID, ID, OperationID, MessageID, Name, Documentation`) Fázis 1 óta létezik, **InterfaceToModel oldalon sem a beolvasás, sem a C# rekord nem létezett** — hasonló minta, mint a `WsdlService`/`WsdlBinding` hiány a Swagger 1/W1 iterációnál.

**Javítás**: pótoltuk a `WsdlOperationFault` rekordot (`WsdlEntities.cs`), a beolvasó SQL-t (`WsdlReader.cs`) és a `WsdlData` lista bővítést — az `InterfaceImporter`-beli entitás oszlopait 1:1 tükrözve.

**Tanulság**: az "előfeltétel már kész" guide-állítást mindig a tényleges kódban kell ellenőrizni, nem csak a DB táblák létezését — a forrás oldali (InterfaceImporter) és a fogyasztó oldali (InterfaceToModel) beolvasás gyakran külön-külön hiányos, egymástól függetlenül.

### 2026-07-24 — `ModuleEntitiesID` idempotencia-bug (W1 `BackendInterfaceEndpoint`-ban, W2 tesztelés közben derült ki)

**Hiba**: `ImportService.RunAsync` Step 5 mindig `context.MainModuleEntitiesID ?? Guid.NewGuid()`-ot használt `context.ModuleEntitiesID`-nek — de `MainModuleEntitiesID` **csak a MAIN DB-t** kérdezte le, sosem a target (SANDBOX) DB-t. Amikor egy modult **másodszor** importáltunk (a WSDL W2 tesztelés pont ezt tette, mert a `WsdlW3Test3` modul már létezett a SANDBOX-ban egy korábbi futtatásból), a `ModelWriter.cs`-ben a `ModuleEntities` upsert (kb. 297. sor) a target DB-ben **megtalálta** a meglévő sort és UPDATE-elte (megtartva a DB-beli valós ID-t), de ez a feloldott ID **sosem került vissza** a `context.ModuleEntitiesID`-be (az `UpsertAsync` `void`-ot ad vissza, nem ID-t). Így a mapping fázisban korábban (még az írás előtt) lefutó `WsdlEndpointMapper` egy random, sosem perzisztált GUID-ot írt a `BackendInterfaceEndpoint.ModuleEntitiesID`-be → `CK_BackendInterfaceEndpoint` CHECK constraint hiba íráskor.

**Miért csak most jött elő**: az első (friss modulra történő) import mindig új `ModuleEntitiesID`-t generált, és mivel nem volt korábbi sor, nem volt ütközés — a hiba csak **ugyanarra a modulra való második importnál** látszik, amit a W1 tesztelésekor nem csináltunk meg (csak friss modult teszteltünk), de a W2 tesztelés (ugyanazt a `WsdlW3Test3` modult használva) igen.

**Javítás**: új `context.ExistingModuleEntitiesID` mező (`MappingContext.cs`), feltöltve egy új target DB lekérdezéssel `MainDatabaseReader.cs`-ben (ugyanaz a minta, mint `context.ExistingModuleID`-nél, csak `ModuleEntities`-re). `ImportService.cs` Step 5 prioritása: `ExistingModuleEntitiesID ?? MainModuleEntitiesID ?? Guid.NewGuid()`.

**Tanulság**: az `UpsertAsync` (void visszatérésű) minta veszélyes, ha a feloldott ID-t egy **korábbi** (mapping) fázisban már felhasználtuk — ez pontosan az a hibaosztály, amit a `UpsertClassAsync` (ID-t visszaadó variáns) a Class-oknál már kezel, de a `ModuleEntities`-nél még nem volt bevezetve. Ha egy jövőbeli új entitás (pl. egy leendő W-iteráció) hasonlóan "elő van állítva mapping közben, de csak írás közben dől el a végleges DB ID-ja" mintát követ, **ugyanez a bug osztály visszatérhet** — érdemes proaktívan végignézni, van-e még ilyen `context.XyzID` amit korán generálunk, de csak a `Existing*ID` (target DB) ellenőrzés nélkül.

### 2026-07-24 — `model.BackendInterface`-nek nincs `ID` oszlopa (WSDL W5 előfeltétel-ellenőrzés közben)

**Guide feltételezés**: a W5 guide `model.BackendInterfaceNamespace` táblát tervezett `[BackendInterfaceID]` FK oszloppal, ami `[model].[BackendInterface].[ID]`-re mutatott volna. A guide mapper-kódja is `context.BackendInterface?.ID`-t olvasott volna ki (nem létező property).

**Valóság**: `model.BackendInterface` PK-ja **`(ProjectID, ModuleID)`** — nincs önálló `ID` surrogate kulcsa. Ez rögtön kiderült volna migráció futtatáskor ("Invalid column name 'ID'" vagy hasonló FK-létrehozási hiba), ha nem ellenőrizzük előre.

**Felismerés módja**: az előfeltétel-ellenőrzés SQL-jét kibővítve közvetlenül lekérdeztük a `model.BackendInterface` valós oszlopait és PK-ját (`sys.columns`/`sys.indexes`), majd megkerestük, hogy más, már működő child tábla (pl. a W3-as `model.BackendInterfaceSchema`) hogyan hivatkozik `BackendInterface`-re — kiderült, hogy **`ModuleID`-vel**, nem szintetikus ID-vel.

**Javítás**: a migráció (`W5_wsdl_namespace.sql`), a C# rekord (`ModelBackendInterfaceNamespace`), a mapper (`WsdlNamespaceMapper`, `context.ModuleID`-t használva `context.BackendInterface?.ID` helyett) és a writer (`WriteBackendInterfaceNamespacesAsync`) mind `ModuleID`-t használ FK-ként, konzisztensen a `BackendInterfaceSchema` mintával.

**Tanulság**: amikor egy guide egy **még nem létező** tábla FK-ját egy meglévő táblára tervezi, ne csak a hivatkozott tábla *létezését* ellenőrizd, hanem a **tényleges FK célt** (oszlopnevet/PK-t) is — főleg, ha a tábla (mint `model.BackendInterface`) történelmileg soha nem kapott surrogate ID-t. A legjobb módszer: keress egy már működő, hasonló szerepű child táblát (itt: `BackendInterfaceSchema`) és kövesd annak mintáját ahelyett, hogy a guide feltételezését szó szerint implementálnád.

---

## 🗂️ Kapcsolódó doksik

Ezeket **frissíteni kell a projekt befejezésekor**, hogy tükrözzék a valós állapotot:

- `swagger_schema_reference.md` (v2.0)
- `wsdl_schema_reference.md` (v2.0) — **Néhány eltérés, lásd fentebb "wsdl.Type" és séma névhasználat**
- `swagger_to_model_full_spec.md`
- `wsdl_to_model_full_spec.md`
- `wsdl_to_model_confluence_doc.md`
- `wsdl_to_model_open_questions.md`
- Iteráció guide-ok (W1-W5, swagger 1, 2b, 2c)

---

## 💡 Konvenciók

### Kód
- Backward-compatible bővítések: új mezők alapértelmezett `null`/`false` értékkel
- Migrációk idempotensek: minden `ALTER` előtt `IF NOT EXISTS` guard
- Dapper anonymous object minta (nem `SqlParameter` collection)
- SQL identifier-ek szögletes zárójelben (`[schema].[Table]`)
- SQL parancsok `const string`-be a hívás előtt

### Guide-ok (chat-ből)
- Backward-kompatibilitás minden bővítésnél
- Vezethető végre inkrementálisan (fázisokra osztva ha nagy)
- Ellenőrzés: **kérdezz vissza ha struktúrális ütközést látsz** a projektben

---

## 🔄 Frissítés protokoll

**Amikor egy Claude session módosít egy releváns dolgot (kód, séma, döntés)**:
1. Frissítsd ezt a fájlt is
2. Ha eltérés van a spec-hez képest → új bejegyzés a "Eltérésnapló"-ba
3. Ha új iteráció befejeződik → jelöld be az "Implementálva" szakaszban
4. **Frissítsd a "Szinkronizációs állapot" táblát** (a fájl elején):
   - "Utoljára írt" = a saját azonosítód (`Claude Code` vagy `Claude Opus`)
   - "Írás dátuma" = ma
   - "Mit írt" = rövid összefoglaló
   - a **saját** "feldolgozta?" meződ → ✅ (dátummal, hiszen te írtad)
   - a **másik fél** "feldolgozta?" mezője → ⬜ (ő még nem tudja, hogy valami változott)
   - adj hozzá egy sort a "Napló" táblához is (a legrégebbi sorokat törölheted, ha ~10 fölé nőne)

**Amikor egy Claude session elolvassa ezt a fájlt (session/beszélgetés elején)**:
- Nézd meg a "Szinkronizációs állapot" táblát
- Ha a **másik** fél írt utoljára és a **te** "feldolgozta?" meződ ⬜ → most olvastad el, tehát állítsd ✅-ra (mai dátummal)
- Ha te magad írtál utoljára, nincs teendő

**Új Claude claude.ai chat elején**: töltsd fel ezt a fájlt → azonnal képben leszek. (Utána jelöld magad feldolgozottnak a fenti szabály szerint.)
