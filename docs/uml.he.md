# UML ברמה גבוהה

מקור עברי לסקירת הארכיטקטורה. אנגלית: [`uml.md`](uml.md). דף אינטראקטיבי: [`../ui/uml.html`](../ui/uml.html) (EN / עב).

המטלה ביקשה סקירת ארכיטקטורה — רכיבים, איך הם מדברים, איפה הדאטה חי בכל שלב — וגם כלים והחלטות. זה הקובץ הזה.

## מה המערכת חייבת לעשות

- **העלאות טרה אמינות** — multipart חתום ל-Amazon S3. ה-API לא מעביר בייטים.
- **מטא-דאטה בהעלאה** — שדות המשתמש נשמרים ב-`draft`, לפני שהחלק הראשון נוחת.
- **העשרה איפה שאפשר** — עובדים אסינכרוניים כתוספים. סוג לא מוכר נשאר אזרח מלא.
- **גלישה לוגית** — אוספים ופילטרים, לא שחזור של מערכת קבצים.

---

## 1. דיאגרמת רכיבים

מישור הבקרה (לקוח + API) מדבר מטא-דאטה. מישור הבייטים (S3) מדבר קבצים. העובדים קוראים מ-S3 וכותבים משמעות חזרה ל-Postgres / OpenSearch.

```mermaid
flowchart LR
  subgraph Control["«component» מישור בקרה"]
    C["«component» לקוח<br/>React + TypeScript"]
    API["«component» Catalog API<br/>Spring Boot 3"]
  end

  subgraph Data["«component» מישור בייטים"]
    S3["«component» Amazon S3<br/>raw / derived"]
  end

  subgraph Work["«component» העשרה"]
    W["«component» עובדים<br/>Python"]
  end

  C -->|"JSON: סשנים, מטא-דאטה, חיפוש"| API
  C -->|"PUT / GET חתום"| S3
  API -->|"CreateMPU · חתימה · Complete"| S3
  API --> PG[(PostgreSQL<br/>אמת)]
  API --> Redis[(Redis<br/>התקדמות חיה)]
  API --> SQS[SQS]
  API --> OS[OpenSearch<br/>הטלה]
  SQS --> W
  W -->|"קריאת טווח · כתיבת נגזרות"| S3
  W --> PG
  W --> OS
```

| רכיב | מדבר עם | מה חי שם |
|---|---|---|
| **לקוח** | API (JSON), S3 (בייטים) | יומן resume ב-IndexedDB, מטא-דאטה בעריכה, חלקים בדרך |
| **Catalog API** | Postgres, Redis, SQS, OpenSearch, S3 (בקרה בלבד) | כלום דורבילי חוץ ממה שהוא כותב לחנויות |
| **Amazon S3** | לקוח, API, עובדים | אובייקטים גולמיים, חלקי multipart, תצוגות נגזרות |
| **PostgreSQL** | API, עובדים | assets, סשנים, ACL, מטא-דאטה של משתמש וטכני |
| **Redis** | API | התקדמות חיה. לא יומן ה-resume |
| **SQS** | API, עובדים | הודעות `asset.uploaded` |
| **עובדים** | S3, Postgres, OpenSearch, SQS | באפרי פענוח זמניים |
| **OpenSearch** | API, עובדים | מסמכי חיפוש שטוחים. אפשר לבנות מחדש |

---

## 2. דיאגרמת רצף — לקוח, שרת, עיבוד, אחסון

ארבעה שלבים, כמו שביקשו במטלה.

```mermaid
sequenceDiagram
  actor Client as לקוח
  participant Server as שרת<br/>Catalog API
  participant Storage as אחסון<br/>Amazon S3
  participant Processing as עיבוד<br/>עובדים

  Note over Client,Processing: 1. כוונה — קודם מטא-דאטה
  Client->>Server: POST /uploads + מטא-דאטה של המשתמש
  Server->>Storage: CreateMultipartUpload
  Storage-->>Server: uploadId
  Server-->>Client: סשן + URLs חתומים לחלקים

  Note over Client,Processing: 2. העברה — הבייטים הולכים ל-S3
  Client->>Storage: PUT חלקים במקביל
  Client->>Server: חלק N נחת (etag, crc)

  Note over Client,Processing: 3. חותמת — S3 מרכיב
  Client->>Server: POST complete + checksums
  Server->>Storage: CompleteMultipartUpload
  Storage-->>Server: האובייקט מורכב
  Server-->>Client: uploaded — אפשר להוריד

  Note over Client,Processing: 4. העשרה — אחרי שהקובץ קיים
  Server->>Processing: asset.uploaded
  Processing->>Storage: קריאת טווח של raw
  Processing->>Storage: PUT derived/
  Processing->>Server: מטא-דאטה טכני

  Note over Client,Processing: גלישה
  Client->>Server: GET /search · GET /assets/{id}
  Server-->>Client: GET חתום לתצוגה
  Client->>Storage: GET נגזרת
```

S3 שומר חלקים ומדביק אותם ב-Complete. הלקוח וה-API מנהלים את הסשן. S3 לא מריץ טרנזקציות SQL ולא מנהל וידאו.

---

## 3. דיאגרמת מחלקות — איפה הדאטה חי

```mermaid
classDiagram
  class Collection {
    id
    name
    acl
  }
  class Asset {
    id
    title
    state
    family
    size_bytes
  }
  class AssetObject {
    role primary|sidecar
    storage_key
    checksum
  }
  class UploadSession {
    multipart_upload_id
    part_size
    parts[]
  }
  class UserMetadata {
    tags
    fields
    source_path
  }
  class TechnicalMetadata {
    extractor
    schema_ver
    data JSONB
  }
  class DerivedArtifact {
    kind thumb|poster|spectrogram
    storage_key
  }
  class EnrichmentJob {
    state queued|ok|failed
  }

  Collection "1" --> "*" Asset
  Asset "1" --> "*" AssetObject
  Asset "1" --> "1" UserMetadata
  Asset "1" --> "*" TechnicalMetadata
  Asset "1" --> "*" DerivedArtifact
  Asset "1" --> "*" EnrichmentJob
  Asset "1" --> "1" UploadSession
```

- **Asset** הוא לוגי — מה שאנשים רואים.
- **AssetObject** הוא פיזי — מה ש-S3 מחזיק.
- **UserMetadata** הוא כוונה — נכתב ב-`POST /uploads`.
- **TechnicalMetadata** הוא תצפית — נכתב אחרי החותמת.

---

## 4. מכונת מצבים

```mermaid
stateDiagram-v2
  [*] --> draft
  draft --> uploading: החלק הראשון נוחת
  uploading --> uploaded: Complete + checksums תואמים
  uploading --> failed: שלמות / abort
  draft --> failed: ביטול סשן
  uploaded --> enriching: worker מתחיל
  enriching --> ready: ה-extractors הרלוונטיים סיימו
  enriching --> partial: extractor נכשל או סוג לא מוכר
  partial --> ready: מריצים extractor שוב
  failed --> draft: retry, המטא-דאטה מועתק
```

הורדה מותרת מ-`uploaded` והלאה. העשרה לא מחביאה קובץ. `failed` הוא רק שלמות / סשן.

---

## 5. כלים

| שכבה | בחירה | למה |
|---|---|---|
| לקוח | React + TypeScript | תור העלאה ארוך ו-inspector |
| API | Java 21, Spring Boot 3 | קטלוג טרנזקציוני. מטא-דאטה, לא בייטים |
| עובדים | Python 3.12 | exif, ffprobe, Tika, SIGMF, GDAL |
| אחסון | Amazon S3 | multipart מובנה, checksums, lifecycle, 50 TB |
| אמת | PostgreSQL 16 | יחסים ומכונת מצבים אמיתית |
| חיפוש | OpenSearch | פילטרים. אפשר לבנות מחדש. לא האמת |
| תור | Amazon SQS | החותמת חייבת לשרוד קריסת worker |
| התקדמות | Redis | לוח לבן בלבד. היומן ב-Postgres |

---

## 6. החלטות שלא אהפוך

1. **מישור בקרה ≠ מישור בייטים.** בייטים לא עוברים ב-API.
2. **Asset ≠ object.** צילום SIGMF הוא שורת קטלוג אחת ושני blobs.
3. **העשרה היא best-effort.** במטלה כתוב “where possible”. unknown הוא טיפוס.
4. **גלישה היא קטלוג.** אוספים + פילטרים. הנתיב הישן הוא פילטר, לא המוצר.
