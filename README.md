# Part 2: Add a Database to Our Flask Application

In Part 2, we’ll work through containerizing an application that requires a database. Docker offers base images for many kinds of databases. This makes it much easier to manage our databases in various environments. Rather than downloading PostgreSQL, installing, configuring, and then running the database on your system directly, we’ll use the Docker-provided image and run it in a container.

In Part 2, we'll be using a sample application that uses FastAPI. Learn more about [FastAPI here](https://fastapi.tiangolo.com/)!

## Step 1: Clone the Sample Repository.
The sample application is not included in your assignment repository. Rather, we'll clone the application from another remote repository.
1. Open a directory on your local machine where you want the Part 2 application to live. In a terminal, cd into the directory where you want to clone the repo, and run the following command:

```bash
git clone https://github.com/z-collective-io/python-docker-dev-example.git
```
2. Once the application is cloned, `cd` into `python-docker-dev-example`. Delete the `.git` directory using `rm -rf .git`.

## Step 2: Create Dockerfile using docker init
1. From the `python-docker-dev-example` directory, run `docker init` to create the necessary Docker assets. 

![docker_init_wizard.png](../img/docker_init_wizard.png)

When prompted, use the following examples:

```text

? What application platform does your project use? Python

? What version of Python do you want to use? 3.11.4

? What port do you want your app to listen on? 8001

? What is the command to run your app? python3 -m uvicorn app:app --host=0.0.0.0 --port=8001

```

## Step 3: Add a local database and persist data
Our sample application requires a database as part of the build. In order to run a database, we will need to create two things: a database container and a **volume** that Docker can manage to store persistent data and configuration. 

We’ll create these resources by using **Docker compose**. 
1. In your project directory, open the `compose.yaml` file that was created after we ran `docker init` . Although `docker init` handled most of what we needed to create, we still need to update it for our application.

2. Uncomment all of the database instructions. See below for what the `compose.yaml` file should look like: 

```docker
*# Here the instructions define your application as a service called "server".*
*# This service is built from the Dockerfile in the current directory.*
*# You can add other services your application may depend on here, such as a*
*# database or a cache. For examples, see the Awesome Compose repository:*
*# https://github.com/docker/awesome-compose*
services:
  server:
    build:
      context: .
    ports:
      - 8001:8001
    environment:
      - POSTGRES_SERVER=db
      - POSTGRES_USER=postgres
      - POSTGRES_DB=example
      - POSTGRES_PASSWORD_FILE=/run/secrets/db-password
    depends_on:
      db:
        condition: service_healthy
    secrets:
      - db-password
  db:
    image: postgres
    restart: always
    user: postgres
    secrets:
      - db-password
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=example
      - POSTGRES_PASSWORD_FILE=/run/secrets/db-password
    expose:
      - 5432
    healthcheck:
      test: ["CMD", "pg_isready"]
      interval: 10s
      timeout: 5s
      retries: 5
volumes:
  db-data:
secrets:
  db-password:
    file: db/password.txt
```
**Note the following new fields:**

- `depends_on` tells Docker Compose to start the database before your application. This is a super power for Docker Compose- you're able to define dependecies and start orders using Compose, which is going to make each subsequent run of this application so much more straight forward.
- `db-data` volume persists the database data between container restarts. 
- `secrets` is a special compose field that lets us define where any requisite secrets can be found so that our application can run. 
- Notice also that the last line of the `compose.yaml` file includes a path to a file `db/password.txt`. We have to create that because it’s not present in the source repository. We have to create this before running `docker compose up`.
- Notice that the `compose.yaml` file doesn’t specify a network for these 2 services. Compose will automatically create a network and connect the services to it. Yet another Docker Compose superpower.

3. In our cloned repository directory, create a new directory named `db`, and inside that directory create a file named `password.txt` that contains the password for the database. Use your IDE (VSCode) or text editor and add the following contents to the password.txt file:
  ```text
  mysecretpassword
  ```

4. Your python-docker-dev directory should now have this structure:

```markdown
|- python-docker-dev-example/
|   |- db/
|   |  |- password.txt
|   |- app.py
|   |- config.py
|   |- requirements.txt
|   |- .dockerignore
|   |- compose.yaml
|   |- Dockerfile
|   |- README.md
|   |- .pre-commit-config.yaml
```

5. From the root of your project (i.e. `python-docker-dev-example/`) run the following command to start your application:

```
docker compose up --build
```

Passing `--build` flag will compile your image and then start the containers all in one step.

First, the database container will start, and then the Uvicorn container will start. Wait until you receive the message, “Uvicorn running on http://0.0.0.0:8801” and then continue.

6. Let’s test our API endpoint. Open a new terminal and make a request to the server using the curl commands:

We’ll create an object with a POST method:

```markdown
 curl -X 'POST' \
  'http://0.0.0.0:8001/heroes/' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "id": 1,
  "name": "my hero",
  "secret_name": "austing",
  "age": 12
}'
```

You will receive the following response:

```
{
  "age": 12,
  "id": 1,
  "name": "my hero",
  "secret_name": "austing"
}
```

Now we’ll make a GET request using curl:

```markdown
curl -X 'GET' \
  'http://0.0.0.0:8001/heroes/' \
  -H 'accept: application/json'
```

We should receive the same response we did when we made our POST request, because we’ve only added one object in our database.

Press `CTRL+C` to stop the application. 

## Step 4: Automatically Update Services

We’re going to use a tool called Compose Watch to automatically update our running Compose services while we edit and save our code. 

1. Open your `compose.yaml` file in your IDE, and add the Compose Watch instructions below: 

 

```markdown
services:
  server:
    build:
      context: .
    ports:
      - 8001:8001
    environment:
      - POSTGRES_SERVER=db
      - POSTGRES_USER=postgres
      - POSTGRES_DB=example
      - POSTGRES_PASSWORD_FILE=/run/secrets/db-password
    depends_on:
      db:
        condition: service_healthy
    secrets:
      - db-password
    develop:
      watch:
        - action: rebuild
          path: .
  db:
    image: postgres
    restart: always
    user: postgres
    secrets:
      - db-password
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=example
      - POSTGRES_PASSWORD_FILE=/run/secrets/db-password
    expose:
      - 5432
    healthcheck:
      test: ["CMD", "pg_isready"]
      interval: 10s
      timeout: 5s
      retries: 5
volumes:
  db-data:
secrets:
  db-password:
    file: db/password.txt
```

2. Run the following command to run our application with Compose Watch:

```markdown
docker compose watch
```

3. Now let’s curl the application:

```markdown
curl http://localhost:8001
```

You should receive a response saying “Hello, Docker!” 

4. If we change our application’s source files locally, they will now be immediately reflected in the running container. 

Open `python-docker-dev-example/app.py` in your IDE and update the “Hello, Docker!” string to say “Hello, <Your Name Here>!” 

5. Save the changes to `app.py` and then wait for a moment for the application to rebuild. Now curl the application again and verify that the update text appears:

```markdown
curl http://localhost:8001
```

Run: 

```
docker compose down
```

to shut down your running containers.

## Step 5: Commit Your Code to GitHub

1. In your project directory, add a .gitignore file and add the following content: 
    
    ```
    # Byte-compiled / optimized / DLL files
    __pycache__/
    *.py[cod]
    *$py.class
    
    # C extensions
    *.so
    
    # Distribution / packaging
    .Python
    build/
    develop-eggs/
    dist/
    downloads/
    eggs/
    .eggs/
    lib/
    lib64/
    parts/
    sdist/
    var/
    wheels/
    share/python-wheels/
    *.egg-info/
    .installed.cfg
    *.egg
    MANIFEST
    
    # Unit test / coverage reports
    htmlcov/
    .tox/
    .nox/
    .coverage
    .coverage.*
    .cache
    nosetests.xml
    coverage.xml
    *.cover
    *.py,cover
    .hypothesis/
    .pytest_cache/
    cover/
    
    # PEP 582; used by e.g. github.com/David-OConnor/pyflow and github.com/pdm-project/pdm
    __pypackages__/
    
    # Environments
    .env
    .venv
    env/
    venv/
    ENV/
    env.bak/
    venv.bak/
    ```
2. Next, go to your profile in GitHub. Click on the 'Repositories' tab, and create a new repository called `assignment-8-part-2`.  *This repository name needs to be lowercase so that we can use it in Part 3*. 
![create_a_repository](../img/create_a_repository.png)

Please make the repository public. Don't initialize the repository with a README file. 
![set_up_repo](../img/set_up_repo.png)

3. Once the repository is created, initalize git locally, add the remote URL to your local git repository and push the local files. You can get the remote URL from the repository set up screen:
![new_repo_url](../img/new_repo_url.png)

    ```bash
    git init
    git remote add origin <URL for your repository>
    git branch -M main
    ```
4. Now add your files and write a commit message:

    ```bash
    git add .
    git commit -m "<commit message here>"
    git push origin main
    ```

[See documentation here if you’re having trouble.](https://github.com/git-guides/git-init)