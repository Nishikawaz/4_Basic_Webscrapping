# Pipeline de scraping y enriquecimiento vía API

Pipeline ETL que scrapea el catálogo completo de [books.toscrape.com](https://books.toscrape.com/), enriquece cada libro con datos de autor desde Open Library y Wikipedia, y persiste todo en SQLite con un modelo relacional normalizado.

**Stack:** Python 3 · BeautifulSoup4 · Requests · SQLite

**Resultado:** 1.000 libros · 805 autores · 50 categorías · 973 relaciones libro–autor

---

## Cómo correrlo

```bash
git clone https://github.com/Nishikawaz/4_Basic_Webscrapping.git
cd 4_Basic_Webscrapping
pip install beautifulsoup4 requests
jupyter notebook pipeline.ipynb
```

Las celdas se ejecutan en orden. `pengu_books.db` ya viene poblada en el repo, así que se pueden correr solo las celdas de consulta (la última) sin volver a scrapear.

> ⏱️ Volver a ejecutar el pipeline completo lleva un rato: son ~50 categorías paginadas más una consulta a Open Library y otra a Wikipedia por autor, con pausas deliberadas entre pedidos.

---

## Qué hace

El pipeline tiene cuatro fases:

**1. Extracción de categorías.** Parsea el panel lateral de la home y guarda las 50 categorías con `INSERT OR IGNORE`, después arma un diccionario en memoria `nombre → id` para no consultar la base en cada inserción posterior.

**2. Scraping de libros.** Recorre cada categoría siguiendo la paginación: mientras exista botón "next", sigue. Por cada libro extrae título, precio y rating. El precio viene como `£51.77` y se limpia con regex a un float; el rating viene como clase CSS (`star-rating Three`) y se mapea a entero con un diccionario.

**3. Enriquecimiento vía API.** Por cada libro consulta Open Library para obtener el autor, y después Wikipedia para inferir el país a partir de la biografía. El título se limpia antes de consultar: `"Hai to Gensou no Grimgar, Vol. 01 (Hai to Gensou no Grimgar #1)"` no matchea nada en Open Library, pero sin la parte entre paréntesis sí.

**4. Consultas analíticas.** Reportes sobre la base ya poblada: libros de más de 3 estrellas por menos de £10, autor con peor promedio de rating (con mínimo de 5 libros para que la muestra signifique algo), categoría con mayor precio promedio, top 5 de autores por cantidad de obras.

---

## Modelo de datos

```
categories ──< books >── book_author ──< authors
```

| Tabla | Filas | Notas |
|---|---:|---|
| `categories` | 50 | `name` con `UNIQUE` |
| `books` | 1.000 | `CHECK` sobre precio ≥ 0 y rating entre 1 y 5 |
| `authors` | 805 | `external_api_id` con `UNIQUE` — el ID de Open Library |
| `book_author` | 973 | Tabla puente, PK compuesta, `ON DELETE CASCADE` |

El diagrama UML del esquema está en [`CH4_UML.jpg`](CH4_UML.jpg) y embebido en la primera celda del notebook.

**Índices:** además de los automáticos por PK y `UNIQUE`, hay índices sobre las FK (`books.category_id`, `book_author.author_id`), sobre las columnas analíticas (`books(rating, price)` compuesto, `authors.name`, `authors.country`, `books.title`) y un `UNIQUE` sobre `books.url`.

---

## Estructura

```
pipeline.ipynb        Notebook completo: esquema, scraping, enriquecimiento, consultas
pengu_books.db        Base SQLite ya poblada
CH4_UML.jpg           Diagrama UML del modelo relacional
```

---

## Decisiones de diseño

**Relación muchos-a-muchos con tabla puente.** Un libro puede tener varios autores y un autor varios libros, así que `book_author` es la única forma correcta de modelarlo. Guardar el autor como columna en `books` habría duplicado los datos del autor en cada obra suya. El `ON DELETE CASCADE` hace que borrar un libro limpie sus relaciones sin dejar filas huérfanas.

**`PRAGMA foreign_keys = ON` en cada conexión.** SQLite trae las claves foráneas **desactivadas por defecto** por retrocompatibilidad, y el pragma no persiste: hay que activarlo en cada conexión nueva. Por eso aparece repetido en todas las celdas que reconectan. Sin él, las FK se declaran pero no se aplican.

**Caché de autores en memoria.** Un diccionario evita consultar dos veces la API por el mismo autor. Con 1.000 libros y 805 autores, sin caché habría ~200 llamadas de red redundantes.

**Respeto por el servidor ajeno.** Tres medidas concretas: `User-Agent` propio (muchas APIs bloquean el default de Python), una pausa de 0,8 s entre consultas a Wikipedia, y reintentos con espera incremental ante un HTTP 429 (*Too Many Requests*). Además `requests.Session()` reutiliza la conexión TCP en vez de abrir una nueva por pedido. La fase de Open Library no lleva pausa fija: se apoya solo en el reintento ante 429.

**`INSERT OR IGNORE` + `UNIQUE` para idempotencia.** El pipeline se puede volver a correr sin duplicar nada: el índice único sobre `books.url` y sobre `authors.external_api_id` hace que un segundo pase simplemente ignore lo que ya está.

**Degradación elegante ante fallos de API.** La función de consulta devuelve `None` ante timeout, respuesta vacía o JSON malformado, en lugar de propagar la excepción. Un autor que Open Library no conoce deja el libro sin relación de autoría, pero no corta el pipeline a la mitad de 1.000 libros.

**Inferencia de país por gentilicio.** Wikipedia no expone la nacionalidad como campo estructurado, así que se busca el gentilicio en el texto de la biografía y se mapea contra un diccionario (`"american" → USA`, `"british" → UK`). Es heurístico y falible, pero es lo que hay sin una fuente estructurada.

---

## Contexto

Challenge de web scraping y modelado de datos. La consigna pedía extraer datos de una web, enriquecerlos con al menos una API externa, diseñar un esquema relacional normalizado con su UML, y responder preguntas analíticas con SQL.

books.toscrape.com es un sitio hecho explícitamente para practicar scraping, así que no hay problema ético ni de términos de servicio en recorrerlo.

---
