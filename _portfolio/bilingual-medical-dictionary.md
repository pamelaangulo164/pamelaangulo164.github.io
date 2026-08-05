---
title: "Bilingual Medical Dictionary API"
excerpt: "An English–Spanish medical terminology application developed with Python, Flask, MySQL, Docker, and RESTful APIs.<br/><img src='/images/medical-dictionary-uml.png'>"
collection: portfolio
permalink: /portfolio/bilingual-medical-dictionary/
author_profile: true
---

## Overview

I developed the Bilingual Medical Dictionary API as my final project for LING 508: Computational Techniques for Linguists. The goal was to build a complete software application that combined object-oriented programming, relational database design, RESTful API development, and software engineering practices within a human language technology project. Instead of creating a basic list of translation pairs, I designed a system that stores English and Spanish medical terms together with definitions, grammatical information, and contextual examples.

The idea for the project came from my previous work as a medical translator and interpreter. In a hospital setting, I often needed to consult more than one resource to confirm the meaning of a term, find an appropriate Spanish equivalent, and verify how the term was used in context. Many general-purpose dictionaries provide a direct translation but do not capture the relationships among synonyms, meanings, grammatical properties, and example sentences. I wanted to explore how those relationships could be represented as structured linguistic data.

The application uses a layered architecture. A Flask API receives requests from the client, a service layer handles the application logic, a repository layer communicates with a MySQL database, and Python data models represent the dictionary entities. Separating these responsibilities made the code easier to read, test, and modify.

The data model includes English terms, Spanish terms, meanings, and examples. This structure allowed the project to represent more than a one-to-one translation relationship. An English term can be associated with a meaning, and that meaning can connect to Spanish terminology and example sentences. The model also stores part-of-speech information for English terms and grammatical gender for Spanish terms.

The API provides endpoints for checking whether the service is running, retrieving an English medical term, and adding a new bilingual entry. Requests and responses use JSON, making the backend independent of a particular user interface. It could therefore be connected to a browser-based application, mobile interface, or another client in the future.

The project also required work beyond implementing the main features. I configured a MySQL database in Docker, created automated tests with pytest, documented the API, and maintained the source code with Git and GitHub. These tasks gave me experience organizing a multi-file application and working with tools that are commonly used in professional software development.

By the end of the project, the application could store and retrieve bilingual medical terminology through a REST API. The project helped me understand how linguistic data, database design, and backend development can be combined into a maintainable language technology application.

## System Architecture

I organized the application into separate layers so that each part of the system had a clear responsibility.

The API layer contains the Flask routes. It receives HTTP requests, reads input parameters or JSON data, and returns the appropriate response and HTTP status code. The routes pass dictionary operations to the service layer rather than communicating with the database directly.

The service layer contains the main application logic. It validates information, creates the objects that represent dictionary entries, coordinates repository operations, and converts the returned objects into JSON-compatible dictionaries.

The repository layer manages database access. SQL and MySQL connection logic are kept outside the API and service code, which reduces duplication and prevents database-specific details from spreading throughout the application.

The model layer defines the linguistic objects used by the program, including English terms, Spanish terms, meanings, examples, parts of speech, and grammatical gender.

Automated tests are stored separately and verify the behavior of the models, service layer, and repository implementation.

The project is organized as follows:

```text
Bilingual Medical Dictionary/
├── api/
│   └── app.py                 Flask application and REST endpoints
├── services/
│   └── service.py             Validation and application logic
├── db/
│   ├── repository.py          Repository interface
│   └── mysql_repository.py    MySQL persistence
├── models/
│   └── models.py              Dictionary entities and enums
├── tests/                     Automated tests
├── data/                      Sample project data
├── documents/                 UML and API documentation
├── templates/                 HTML templates
├── docker-compose.yml         MySQL container configuration
├── requirements.txt           Python dependencies
├── pytest.ini                 Test configuration
└── README.md                  Setup and usage documentation
```

This organization made it easier to locate problems during development. For example, an HTTP error could be investigated in the API layer, while an incorrect database result could be traced through the repository without changing the Flask routes.

## Technologies Used

- Languages: Python, HTML
- Framework: Flask
- Database: MySQL
- Testing: pytest
- Containerization: Docker, Docker Compose
- Version control: Git, GitHub
- Data format: JSON

## Database and Object Model

Designing the data model was one of the most important parts of the project. A simple table containing an English word and a Spanish translation would have been easier to implement, but it would not represent the relationships I wanted the dictionary to support.

The application was designed around four primary entities:

- `EnglishTerm`
- `SpanishTerm`
- `Meaning`
- `Example`

### EnglishTerm

The `EnglishTerm` entity represents an English medical term and stores its part of speech. A term can be connected to one or more meanings, allowing the application to distinguish vocabulary from the concept it expresses.

Example fields include:

- `term`
- `pos`
- `term_id`

### SpanishTerm

The `SpanishTerm` entity represents the corresponding Spanish terminology. It also stores grammatical gender when that information applies.

Example fields include:

- `term`
- `gender`
- `term_id`

### Meaning

The `Meaning` entity stores the definition of the medical concept. Separating the definition from the individual terms provides a more flexible way to represent bilingual terminology than placing all information in a single record.

Example fields include:

- `description`
- `meaning_id`
- associated Spanish terms
- associated examples

### Example

The `Example` entity stores contextual sentences. Each example includes a language label and the example text. This makes it possible to supplement dictionary entries with information about how a term is used rather than presenting it only as an isolated word.

Example fields include:

- `language`
- `text`
- `example_id`

### Entity Relationships

The model supports relationships in which:

- An English term can be associated with one or more meanings.
- A meaning can be associated with Spanish terminology.
- A meaning can include multiple contextual examples.
- English and Spanish entries can retain their own grammatical information.

The UML diagram below shows the primary classes and relationships in the application.

<img
  class="project-image"
  src="{{ '/images/medical-dictionary-uml.png' | relative_url }}"
  alt="UML diagram for the Bilingual Medical Dictionary API">

<p class="image-caption">
Figure 1. Object model for the English terms, Spanish terms, meanings, and contextual examples used in the application.
</p>

The Python model classes reflect the structure shown in the diagram. Working with objects instead of raw database rows made the service code easier to understand and provided a consistent representation of dictionary entries throughout the application.

## REST API Implementation

I used Flask to create the REST API. The API receives client requests and delegates dictionary operations to `DictionaryService`. This keeps request handling separate from application and database logic.

The lookup endpoint accepts an English medical term through the `english` query parameter:

```python
@app.get("/api/v1/lookup")
def lookup():
    lemma = request.args.get("english", "").strip()

    if not lemma:
        return jsonify(
            {"error": "missing query param 'english'"}
        ), 400

    try:
        entry = service.lookup_english(lemma)
    except ValueError as exc:
        return jsonify({"error": str(exc)}), 400

    if entry is None:
        return jsonify(
            {
                "error": "not found",
                "term": lemma,
            }
        ), 404

    return jsonify(service.serialize_entry(entry)), 200
```

This endpoint performs three main checks. It returns `400 Bad Request` when the required query parameter is missing or invalid, `404 Not Found` when the term is not present in the database, and `200 OK` with a JSON response when the lookup succeeds.

The successful response is created through the service layer rather than by passing the original Python object directly to Flask. This is important because the model objects contain relationships to other objects. The `serialize_entry` method converts the relevant fields into a controlled dictionary that can be safely returned as JSON.

```python
def serialize_entry(self, et: EnglishTerm) -> Dict[str, Any]:
    return {
        "term": et.term,
        "pos": et.pos.value,
        "term_id": str(et.term_id),
        "meanings": [
            {
                "meaning_id": str(meaning.meaning_id),
                "description": meaning.description,
                "spanish_terms": [
                    {
                        "term_id": str(spanish.term_id),
                        "term": spanish.term,
                        "gender": spanish.gender.value,
                    }
                    for spanish in meaning.spanish_terms
                ],
                "examples": [
                    {
                        "example_id": str(example.example_id),
                        "language": example.language,
                        "text": example.text,
                    }
                    for example in meaning.examples
                ],
            }
            for meaning in et.meanings
        ],
    }
```

This method determines exactly which information appears in the API response. It also converts enum values and identifiers into JSON-compatible values.

The following screenshot shows the response returned for a lookup of the English medical term `fever`.

<img
  class="project-image"
  src="{{ '/images/lookup-fever-response.png' | relative_url }}"
  alt="JSON response returned for a lookup of the English term fever">

<p class="image-caption">
Figure 2. JSON response returned by the lookup endpoint for the English medical term “fever.”
</p>

The application also includes a health-check endpoint:

```text
GET /healthz
```

and an endpoint for adding a new dictionary entry:

```text
POST /api/v1/add
```

The POST endpoint accepts a JSON payload containing an English term, part of speech, definition, Spanish term, grammatical gender, and optional example sentences. It validates the request before passing the information to the service and repository layers.

### Dockerized MySQL Database

The application stores its dictionary entries in a MySQL database running inside a Docker container. Docker Compose provided a consistent database environment and reduced the amount of local configuration needed to run the project. The Flask application connects to the container through the MySQL port exposed on the local machine.

<img
  class="project-image"
  src="{{ '/images/mysql-docker.png' | relative_url }}"
  alt="MySQL database running inside a Docker container">

<p class="image-caption">
Figure 3. MySQL Docker container running the database used by the Bilingual Medical Dictionary API.
</p>

## Testing

I used pytest to test the project components. The tests covered the model classes, service behavior, and repository operations. Keeping tests in a separate directory made it possible to verify individual parts of the application without relying only on manual testing through the browser.

Automated testing was particularly useful after changes to the object model and repository layer. It helped me confirm that existing functionality still worked and made debugging more systematic.

The tests can be run from the project root with:

```powershell
$env:PYTHONPATH="."
pytest
```

## Challenges and Solutions

One challenge was deciding how to represent bilingual terminology without reducing every entry to a direct word-for-word translation. My earlier design linked English and Spanish terms too directly, which made it harder to represent definitions and contextual examples. I revised the design so that meanings served as a separate part of the model. This gave the application a clearer way to organize terms and their shared concepts.

A second challenge involved separating the application into independent layers. Placing routes, database access, and object creation in the same file would have been faster initially, but it would have made the program difficult to test and maintain. I separated the code into API, service, repository, and model modules. This required more planning, but it made the responsibilities of each file clearer.

I also had to learn how to configure MySQL through Docker Compose. The Flask application depended on a running database container, so I needed to understand Docker services, ports, environment settings, and database connections. Troubleshooting the connection between Flask and MySQL gave me experience diagnosing issues across more than one component of an application.

Another issue appeared when Flask attempted to convert linked dataclass objects directly into JSON. The model contained relationships that could refer back to other objects, which caused recursive serialization. I resolved this by using the service layer to build a plain dictionary containing only the fields needed in the response. This prevented circular references and gave me greater control over the API output.

## Learning Outcomes

This project strengthened my Python programming skills because it required me to build a multi-component application rather than complete a set of isolated exercises. I wrote and debugged code across the API, service, repository, model, and testing layers. I also documented the project through the README, API descriptions, code comments, and UML diagram.

I gained a better understanding of how software engineering principles can be applied to linguistic data. The object and database models were designed around relationships among terms, meanings, grammatical information, and contextual examples. This required me to think about how linguistic concepts should be represented computationally before implementing them in code.

The project also gave me practical experience integrating tools commonly used in backend development and HLT workflows. I used Flask to expose the application through HTTP endpoints, MySQL to persist structured data, Docker to run the database in a reproducible environment, pytest to verify functionality, and Git and GitHub to manage the codebase.

Finally, the project improved my technical communication and project organization skills. I needed to explain the application architecture, document how to run the project, create a visual model of the system, and organize the repository so that another developer could understand its components. These tasks reinforced the importance of writing software that is not only functional but also readable and documented.

## Results and Future Work

At the completion of the project, the application could connect to a MySQL database, add bilingual medical terminology, retrieve English entries, and return structured JSON responses through Flask endpoints. The project met its main objective of combining a linguistic data model with a working backend application.

The current terminology dataset is small, so one future improvement would be to add a larger collection of validated medical terms. The dictionary could also be expanded to support Spanish-to-English searches, partial matching, synonyms, usage notes, pronunciation information, and additional medical domains.

A browser-based interface could make the system more accessible to interpreters, translators, healthcare workers, and students who do not interact directly with REST APIs. Future versions could also include authentication and administrative tools for adding or editing entries, along with stronger validation for duplicate or inconsistent terminology.

## Source Code

The complete project repository includes the Flask API, Python models, service and repository layers, MySQL configuration, automated tests, UML diagram, and setup documentation.

[View the Bilingual Medical Dictionary API on GitHub](https://github.com/pamelaangulo164/LING508-project)