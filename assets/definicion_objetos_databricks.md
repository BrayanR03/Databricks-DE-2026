### 🏗️ ¿Dónde podemos definir objetos en Databricks?

Dentro de Databricks podemos crear distintos objetos de la plataforma, tales como:

* 📂 Catálogos (Catalogs)
* 📂 Esquemas (Schemas)
* 📦 Volúmenes (Volumes)
* 🗄️ Delta Tables

Para ello disponemos de dos entornos principales de trabajo:

#### 1️⃣ SQL Editor

Permite ejecutar instrucciones SQL directamente sobre el Lakehouse.

Es una opción muy utilizada para:

* Crear objetos de Unity Catalog
* Administrar permisos
* Consultar información
* Definir tablas mediante SQL

#### 2️⃣ Notebooks de Databricks

Los notebooks ofrecen un entorno multi-lenguaje donde podemos combinar diferentes tecnologías dentro de un mismo flujo de trabajo.

Entre los lenguajes más utilizados encontramos:

* 🐍 Python
* ⚡ PySpark
* 📝 SQL
* 🔷 Scala
* 📊 R
* 🐧 Comandos del sistema operativo mediante `%sh`
* 📖 Markdown para documentación

Gracias a esta flexibilidad, una Delta Table puede crearse tanto mediante instrucciones SQL como utilizando la API de PySpark.

---

### 🛡️ Consideración importante: Unity Catalog

Independientemente de si utilizamos SQL Editor, Spark SQL o PySpark, la creación de objetos dentro de Databricks debe respetar la estructura de gobernanza definida en Unity Catalog.

Por ello, antes de crear nuestra primera Delta Table, revisaremos la jerarquía fundamental utilizada por Databricks:

📂 Catalog → 📂 Schema → 🗄️ Table

Esta misma estructura será utilizada en todos los ejemplos posteriores.
