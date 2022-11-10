# Deploying a FastAPI app on Google Cloud Run
This is a very simple hello-world-walkthrough with FastAPI and Cloud Run.


## Initial Setup
To play through this tutorial, I recommend creating a new project. You can do this in the console or [with the Cloud SDK](https://cloud.google.com/sdk/gcloud/reference/projects/create) (recommended). You can find your billing account id [here](https://console.cloud.google.com/billing)

First, create a new repository through the console in artifact registry for region "europe-west3" with name `docker-fastapi`.

Create  your new project.
```bash
export PROJECT_ID=<YOUR_UNIQUE_LOWER_CASE_PROJECT_ID>
export DOCKER_REPOSITORY="docker-fastapi"
export BILLING_ACCOUNT_ID=<YOUR_BILLING_ACCOUNT_ID>
export APP=myapp 
export PORT=1234
export REGION="europe-west3"
export TAG="$REGION.pkg.dev/$PROJECT_ID/$DOCKER_REPOSITORY/$APP:tag1"

gcloud projects create $PROJECT_ID --name="My FastAPI App"

# Set Default Project (all later commands will use it) 
gcloud config set project $PROJECT_ID

# Add Billing Account (no free lunch^^)
gcloud beta billing projects link $PROJECT_ID --billing-account=$BILLING_ACCOUNT_ID
```

Enable the services you need.
```bash
gcloud services enable cloudbuild.googleapis.com \
    containerregistry.googleapis.com \
    run.googleapis.com
```

## Simple Hello World App

Let's keep it bloody simple and focus on the general process. This is a our hello world app `src/main.py`.
```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
async def root():
    return {"message": "Hello World"}
```

If you want to test it locally, run:
```bash
pip install -r src/requirements.txt
cd src 
uvicorn main:app --reload
```

You can check the output in your browser by opening http://127.0.0.1:1234 or "curling" the app in another terminal tab via:
```bash
curl http://127.0.0.1:$PORT
```
As a response you'll see `{"message":"Hello World"}`.

## App into Docker

You need to put your app into an image before deploying it. Here you see an example docker file. Note that you should use a PORT as environment variable. If you don't you'll get an error like `Cloud Run error: The user-provided container failed to start and listen on the port defined provided by the PORT=8080 environment variable.`

```docker
FROM python:3.9-slim@sha256:980b778550c0d938574f1b556362b27601ea5c620130a572feb63ac1df03eda5 

ENV PYTHONUNBUFFERED True

ENV APP_HOME /app
WORKDIR $APP_HOME
COPY . ./

ENV PORT 1234

RUN pip install --no-cache-dir -r requirements.txt

# As an example here we're running the web service with one worker on uvicorn.
CMD exec uvicorn main:app --host 0.0.0.0 --port ${PORT} --workers 1
```

To try your app locally with docker simply run inside `src`:
```bash
docker build -t $TAG .
docker run -dp $PORT:$PORT -e PORT=$PORT $TAG
```

Again you can check it in your browser our curl it:
```bash
 curl http://127.0.0.1:$PORT
```

## Deployment
If everything worked out so far, we're ready to deploy our app. First we push the docker image we created to the artifact registry repository.

Before, we need to authenticate to the repository

```bash
gcloud auth configure-docker $REGION-docker.pkg.dev
```

Then, we can push the image that we created with `docker build` previously.

```bash
docker push "$TAG"
```

After this is done, well it's finally time to deploy your cloud run app :).
```bash
gcloud run deploy $APP --image $TAG --platform managed --region $REGION --allow-unauthenticated
```

## Test it
Note, this may take some minutes. The URL of your app will show up in the terminal. But you can also check your app via:
```bash
# See all info about the app
gcloud run services describe $APP --region $REGION
```

Note, even though we've chosen a random port like `1234`, the deployed service will use `8080` by default. This is why we need `--port ${PORT}` in the last line of our Dockerfile.

Let's send a request to our app.
```bash
# get url
URL=$(gcloud run services describe $APP --region $REGION --format 'value(status.url)')

curl $URL
```
Perfect! Again, we receive `{"message":"Hello World"}`.

## Clean up
If you don't need the app anymore remove it. (You may also delete the whole project instead.)
```bash
gcloud run services delete $APP --region $REGION

# Check if it disAPPeared (optional)
gcloud run services list
```