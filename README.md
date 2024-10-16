# Spring AI Recipe Finder

Demo on how easy it is to build AI-driven applications with Spring Boot and Spring AI. 
It also shows how to implement advanced techniques for the adaption of foundation models with Function Calling and RAG.

![](docs/images/ui-sample.png)

[Slides: Building AI-Driven Spring Applications with Spring AI](docs/slides.pdf)

# Setup
## LLM
### Local LLM (Ollama)
As Ollama doesn't yet provide a text-to-image model, recipe image generation is not available with this setup.
#### Option 1
1. Download and install Ollama on your local machine [link](https://ollama.com/)
2. Start llama3.1 model (3.1 is the first Llama version that supports Function Calling, even if it's not working well with the 8B version)
    ```
    ollama run llama3.1
    ```
#### Option 2
By enabling the "ollama-compose" Spring profile, the llama3 model will be automatically started and configured with docker compose.
Depending on your system (e.g. ARM macs) this is not a recommended setup due to performance reasons.
```
export SPRING_PROFILES_ACTIVE=ollama-compose
```
### OpenAI
Set the API key via an environment variable or in [application-openai.yaml](src/main/resources/application-openai.yaml).
```
export SPRING_AI_OPENAI_API_KEY=<INSERT KEY HERE>
```
Run your application with the "openai" Spring Profile.
```
export SPRING_PROFILES_ACTIVE=openai
```

### Azure OpenAI
Set the API key and endpoint via environment variables or in [application-azure.yaml](src/main/resources/application-azure.yaml).
```
export SPRING_AI_AZURE_OPENAI_API_KEY=<INSERT KEY HERE>
export SPRING_AI_AZURE_OPENAI_ENDPOINT=https://{your-resource-name}.openai.azure.com
```

Make sure the deployment names of the models match exactly what's in your [application-azure.yaml](src/main/resources/application-azure.yaml) configuration file.

Currently, [**only some regions support image generation** with Dall-E](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/models#dall-e-models).
If you use a region that doesn't support it, you have to disable the image generation by setting `ai.azure.openai.image.enabled: false` in the [application-azure.yaml](src/main/resources/application-azure.yaml) configuration file to not run into errors.

Run your application with the "azure" Spring Profile.
```
export SPRING_PROFILES_ACTIVE=azure
```

### Vector DB
On your local machine, a Redis database is automatically started and configured with docker compose.

# Running the application locally
```
./gradlew bootRun
```
Open [http://localhost:8080](http://localhost:8080) in your browser. 
Enter the ingredients (e.g. "Potatoes") you want to find a recipe for in the form and press the "find" button.

## Function Calling 
By checking the "Prefer available ingredients" checkbox, [Function Calling](https://docs.spring.io/spring-ai/reference/1.0/concepts.html#_function_calling) will be enabled.
As the functionalities to add always available ingredients and for the API call to check the available ingredients in the fridge are not yet implemented, they can be configured via the
`app.always-available-ingredients` and `app.available-ingredients-in-fridge` properties in [application.yaml](src/main/resources/application.yaml).

Bacon and onions are currently configured for available ingredients in fridge.
With the input "Potatoes", you should get a recipe with potatoes and bacon.
![](docs/images/ui-sample-function-calling.png)

## Retrieval-Augmented Generation(RAG)
By checking the "Prefer own recipes" checkbox, [Retrieval-Augmented Generation](https://docs.spring.io/spring-ai/reference/1.0/concepts.html#concept-rag) will be enabled.

To upload your own PDF documents for recipes to the vector database, there is a REST API endpoint implemented. 
```
curl -XPOST -F "file=@$PWD/german_recipes.pdf" -F "pageBottomMargin=50" http://localhost:8080/api/v1/recipes/upload
```
The sample recipe part of this repository is a potato soup. With the input "Potatoes", you should get a recipe that goes in the direction of a potato soup.
![](docs/images/ui-sample-rag.png)

# Kubernetes Deployment

## Ollama
```
kubectl apply -f deployment/kubernetes/ollama.yaml
```

## OpenAI
```
export SPRING_AI_AZURE_OPENAI_API_KEY=<INSERT KEY HERE>
export SPRING_AI_AZURE_OPENAI_ENDPOINT=<INSERT ENDPOINT URL HERE>
envsubst < deployment/kubernetes/openai.yaml | kubectl apply -f -
```
## Azure OpenAI
```
export SPRING_AI_OPENAI_API_KEY=<INSERT KEY HERE>
envsubst < deployment/kubernetes/azure-openai.yaml | kubectl apply -f -
```

# Cloud Foundry Deployment

* Install the GenAI tile
* Install the PostgreSQL tile
* Create backing service instances

```console
cf create-service genai llama3.2 model-1
cf create-service postgres on-demand-postgres-db psql-1
```

* Build bootJar

```console
./gradlew bootJar
```

## Push app

```console
cf push -f deployment/tanzu-platform-for-cf/manifest.yml
```

## Post recipe pdf for RAG

```console
curl -XPOST \
  -F "file=@$PWD/german_recipes.pdf" \
  -F "pageBottomMargin=50" \
  http://<APP ROUTE>/api/v1/recipes/upload # update <APP ROUTE> accordingly
```

## Changelog

* Bumped to spring ai version 1.0.0-M3
* Added java-cfenv to autowire ai properties from bindings
* Added a `tas` profile
* Use PostgreSQL for the vectorstore rather than redis
* Added a cloud foundry manifest.yml file
* Added some extra logging

## Known Issues

* Responses are quite tempromental ... sometimes it works, sometimes it doesn't 🤷‍♂️
  * I think the problem is that sometimes the model doesn't return a response that adheres to the required JSON schema, but I'm not 100% sure about that
* I started trying to integrate Stable Diffusion for text-to-image generation
  * It's also not quite working and I'm not sure why
