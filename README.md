# Recipe API

A REST API for managing personal recipes. Users can create an account, get an authentication token, and save recipes with ingredients, tags, and an optional image. Recipes and their tags/ingredients are scoped to the authenticated user.

Built with Django, Django REST Framework, PostgreSQL, and Docker Compose. The API schema and interactive Swagger UI are generated with drf-spectacular.

> **Deployment status:** There is no hosted API or AWS deployment. Hosting on AWS would incur costs, so this project is intended to be run and tested locally. A separate deployment-style Compose configuration is included, but it is not a live deployment.

## Run locally

You need Docker with the Docker Compose plugin installed and running. From the repository root:

```sh
docker compose up --build
```

Compose starts PostgreSQL and the Django development server, waits for the database, and applies migrations. Once startup finishes, visit:

- API documentation (Swagger UI): http://localhost:8000/api/docs/
- OpenAPI schema: http://localhost:8000/api/schema/

The development Compose file supplies its own database settings; you do **not** need to create a `.env` file to run it. Stop the services with `Ctrl+C`, or run `docker compose down` from another terminal. The database is stored in a Docker volume, so it persists between runs.

## How the API works

Create a user with an email and password, request a token, then send that token in the `Authorization: Token <token>` header on protected endpoints. Each user sees only their own recipes, tags, and ingredients. When creating or updating a recipe, pass tags and ingredients as arrays of objects containing a `name`; the API reuses matching items belonging to that user or creates them as needed.

| Endpoint | Methods | Purpose |
| --- | --- | --- |
| `/api/user/create/` | POST | Register an account (public) |
| `/api/user/token/` | POST | Get an authentication token (public) |
| `/api/user/me/` | GET, PUT, PATCH | View or update your profile |
| `/api/recipe/recipes/` | GET, POST | List or create your recipes |
| `/api/recipe/recipes/<id>/` | GET, PUT, PATCH, DELETE | View, update, or delete a recipe |
| `/api/recipe/recipes/<id>/upload-image/` | POST | Upload an image using multipart form data |
| `/api/recipe/tags/` | GET | List your tags |
| `/api/recipe/tags/<id>/` | PUT, PATCH, DELETE | Update or delete a tag |
| `/api/recipe/ingredients/` | GET | List your ingredients |
| `/api/recipe/ingredients/<id>/` | PUT, PATCH, DELETE | Update or delete an ingredient |

Recipes can be filtered with comma-separated IDs, e.g. `GET /api/recipe/recipes/?tags=1,2` or `?ingredients=1,2`. To list only tags or ingredients used by recipes, add `?assigned_only=1` to their list endpoint. Tags and ingredients are created through recipe requests rather than standalone POST endpoints.

## Try it with curl

Run these from another terminal after starting Compose. Use the token returned by the second request in place of `YOUR_TOKEN` below.

```sh
# 1. Register
curl -X POST http://localhost:8000/api/user/create/ \
  -H 'Content-Type: application/json' \
  -d '{"email":"cook@example.com","password":"testpass123","name":"Home Cook"}'

# 2. Get a token (response contains a "token" field)
curl -X POST http://localhost:8000/api/user/token/ \
  -H 'Content-Type: application/json' \
  -d '{"email":"cook@example.com","password":"testpass123"}'

# 3. Create a recipe
curl -X POST http://localhost:8000/api/recipe/recipes/ \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Token YOUR_TOKEN' \
  -d '{"title":"Tomato pasta","time_minutes":20,"price":"4.50","description":"Quick weeknight dinner","tags":[{"name":"Dinner"}],"ingredients":[{"name":"Tomato"},{"name":"Pasta"}]}'

# 4. List your recipes
curl http://localhost:8000/api/recipe/recipes/ \
  -H 'Authorization: Token YOUR_TOKEN'
```

The create response includes a recipe `id`. To upload an image, use that ID and an existing local image file:

```sh
curl -X POST http://localhost:8000/api/recipe/recipes/1/upload-image/ \
  -H 'Authorization: Token YOUR_TOKEN' \
  -F 'image=@/path/to/photo.jpg'
```

You can also explore the endpoints and request formats in the Swagger UI. In local development, uploaded media is served by Django under `/static/media/`.

## Run the automated tests

The test suite covers users, authentication, recipes, tags, ingredients, image uploads, and database behavior. Run it with a temporary app container and the Compose PostgreSQL service:

```sh
docker compose run --rm app sh -c "python manage.py wait_for_db && python manage.py test"
```

To run the linter:

```sh
docker compose run --rm app sh -c "flake8"
```

Docker Compose builds the development image with the test/lint dependencies. Django creates a separate test database, so the test suite does not use your application data.

## Deployment configuration

`docker-compose-deploy.yml` describes a separate Django/uWSGI, PostgreSQL, and Nginx setup. It reads `DB_NAME`, `DB_USER`, `DB_PASS`, `DJANGO_SECRET_KEY`, and `DJANGO_ALLOWED_HOSTS` from an environment file (see `.env.sample` for the expected names). These are example values only; replace them with strong credentials and appropriate allowed hosts before using that configuration. The development instructions above do not require this file or any AWS services.
