%md
## 🌎 Creación de External Tables en Databricks apuntando a S3

Una vez comprendidos los conceptos de Managed Tables y External Tables, vamos a documentar el proceso completo, paso a paso, para crear una **External Table** en Databricks (Free Edition) que apunte a un archivo `.csv` almacenado en Amazon S3 — usando como caso de estudio el dataset de **tips**.

A diferencia de Databricks clásico (con instance profiles), en **Unity Catalog** el acceso a S3 se gestiona mediante dos objetos:

* 🔐 **Storage Credential** → cómo autenticarse contra S3
* 📍 **External Location** → dónde están físicamente los datos

Y ambos dependen de un **IAM Role** configurado correctamente en AWS.

---

### 🧩 Arquitectura general del flujo

```
AWS (IAM Role) ──► Databricks (Storage Credential) ──► Databricks (External Location) ──► External Table
```

---

#### 1️⃣ Crear el IAM Role en AWS

⚠️ **Importante**: el rol **no** se crea seleccionando "AWS service" ni "Another AWS account" con tu propio Account ID. Databricks usa una **cuenta fija** (la del Unity Catalog Master Role), por lo que el rol debe crearse con un **Custom Trust Policy**.

#### Trust Policy inicial (con placeholder)

En IAM → Create role → **Custom Trust Policy**:

```json id="trust01"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::414351767826:role/unity-catalog-prod-UCMasterRole-14S5ZJVKOTYTL"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "0000"
        }
      }
    }
  ]
}
```

* 📌 `414351767826` es el **Account ID fijo** de Databricks (Unity Catalog Master Role) — es el mismo para todos los workspaces, no depende de tu cuenta ni de tu Workspace ID.
* 📌 `"0000"` es un **placeholder**. Se actualizará en el paso 4 con el External ID real que genera Databricks.

#### Permission Policy (acceso al bucket)

Adjuntar al rol una policy con permisos sobre el bucket específico:

```json id="permpol01"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::bucket-datasets-brayan",
        "arn:aws:s3:::bucket-datasets-brayan/*"
      ]
    }
  ]
}
```

Guardar el rol y copiar su **ARN** (lo necesitaremos en el siguiente paso):

```
arn:aws:iam::<TU-ACCOUNT-ID>:role/<NOMBRE-DEL-ROL>
```

---

#### 2️⃣ Crear el Storage Credential en Databricks (Mediante la UI de Databricks)

En **Catalog → ⚙️(Managed) → Credentials → Create credential**:

* `Storage credential`
* **Credential Type**: `AWS IAM Role`
* **Credential name**: `credential-databricks-aws-external-table`
* **IAM Role (ARN)**: pegar el ARN del rol creado en el paso 1
* Click **Create**

Databricks genera automáticamente un **External ID** real, que reemplaza al `"0000"` en el `Trust Policy inicial`.

> 💡 El campo "Account ID" que normalmente se busca en formularios genéricos de AWS **no aparece aquí** porque Databricks no usa tu cuenta como principal de confianza, sino la suya (la del UC Master Role).

---

#### 3️⃣ Validar la configuración (Self Assume Role)

Al crear la External Location, Databricks ejecuta varios checks automáticos:

| Check | Significado |
|---|---|
| ✅ Read | Permisos de lectura sobre el path S3 |
| ✅ Assume Role | Databricks puede asumir el rol |
| ❌ **Self Assume Role** | El rol debe poder asumirse **a sí mismo** |
| ✅ External ID Condition | El External ID coincide |
| ❌ File Events Resource Provision/Teardown | Relacionado a notificaciones (SQS/SNS), no crítico para tablas estáticas |

##### 🔑 Por qué falla "Self Assume Role"

Desde el 30 de junio de 2023, AWS exige que los roles IAM confíen explícitamente en sí mismos para llamadas `sts:AssumeRole`. Desde el 20 de enero de 2025, Databricks **bloquea** storage credentials con roles que no son self-assuming.

##### ✅ Solución: agregar el propio rol como segundo Principal

Modificamos el `Trust Policy inicial` por:
```json id="trustfinal01"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::414351767826:role/unity-catalog-prod-UCMasterRole-14S5ZJVKOTYTL",
          "arn:aws:iam::<TU-ACCOUNT-ID>:role/<NOMBRE-DEL-ROL>"
        ]
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "<EXTERNAL-ID-REAL-GENERADO-POR-DATABRICKS>"
        }
      }
    }
  ]
}
```

* 📌 El "self-assume" consiste en que el **mismo ARN del rol** aparezca dentro de su propia trust policy, junto al principal de Databricks.
* 📌 Reemplazar también `"0000"` por el **External ID real** entregado por Databricks al crear el credential.

Tras guardar este cambio en AWS, volver a Databricks y ejecutar **"Validate Configuration"** — el check de Self Assume Role debe pasar a ✅.

##### Sobre los checks de "File Events" (opcional)

Estos checks corresponden a notificaciones de cambios en el bucket (usadas por Auto Loader vía SQS/SNS). No son necesarios para una external table estática de un CSV. Si se quiere resolver, agregar permisos adicionales:

```json id="fileevents01"
{
  "Effect": "Allow",
  "Action": [
    "sqs:CreateQueue",
    "sqs:DeleteQueue",
    "sqs:GetQueueAttributes",
    "sqs:SetQueueAttributes",
    "sns:CreateTopic",
    "sns:DeleteTopic",
    "sns:GetTopicAttributes",
    "sns:SetTopicAttributes",
    "sns:Subscribe",
    "s3:GetBucketNotification",
    "s3:PutBucketNotification"
  ],
  "Resource": "*"
}
```

---

#### 4️⃣ Crear la External Location

Una vez validado el Storage Credential, se crea la External Location apuntando a la ruta S3:

En **Catalog → ⚙️(Managed) → External Locations → Create external location**:

* **How would you like to create an external location?**: `Manual`
* **External location name**: `external_location_tips`
* **Storage type**: `S3`
* **URL**: `s3://bucket-datasets-brayan/tips.csv`
* **Storage credential**: `credential-databricks-aws-external-table`
* Click **Create**

> 📌 Si, la External Location apunta directamente a un archivo (no a una carpeta), Databricks la marca como **"Single Read-only File"** — válido para lectura, pero con limitaciones de escritura/DML.

---

#### 5️⃣ Crear el catálogo y schema (si no existen)

```sql id="catschema01"
CREATE CATALOG IF NOT EXISTS catalog_databricks_2026_de;
CREATE SCHEMA IF NOT EXISTS catalog_databricks_2026_de.schema_databricks_2026_de;
```
---

## 🎯 Resumen del flujo completo

| Paso | Acción | Dónde |
|---|---|---|
| 1 | Crear IAM Role con Custom Trust Policy (principal fijo de Databricks `414351767826` + External ID placeholder) | AWS |
| 2 | Adjuntar Permission Policy con acceso al bucket S3 | AWS |
| 3 | Crear Storage Credential con el ARN del rol | Databricks |
| 4 | Copiar el External ID real generado | Databricks |
| 5 | Actualizar Trust Policy: reemplazar placeholder por External ID real + agregar el propio ARN del rol (self-assume) | AWS |
| 6 | Validar configuración (todos los checks en ✅, salvo File Events si no se necesita) | Databricks |
| 7 | Crear External Location apuntando al path S3 | Databricks |
| 8 | Crear catálogo y schema | Databricks |
---

#### 🎯 Conclusión

💡 El punto crítico de este proceso no está en la sintaxis SQL de `CREATE TABLE`, sino en la correcta configuración de la **trust policy** del IAM Role: debe confiar en el **Unity Catalog Master Role** de Databricks **y en sí mismo** (self-assuming), condicionado por el **External ID** único generado por el Storage Credential. Sin este paso, cualquier intento de crear la External Location fallará en el check de **Self Assume Role**.