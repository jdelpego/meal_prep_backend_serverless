# Mealy (serverless)

Mealy is a meal optimization service that calculates practical portion sizes for a set of foods to hit calorie, macro and micronutrient targets. This repository contains a serverless (AWS SAM) version of the original Mealy application — a single AWS Lambda function with an API Gateway endpoint.

## What it does

- Accepts a meal request (list of foods + calorie/macro targets).
- Runs a constrained, weighted least-squares optimizer (scipy) to compute grams per food that match calorie, carbohydrate, protein and fat targets while also nudging micronutrient targets and ensuring realistic serving sizes.
- Returns portion sizes and a nutrition summary.

## Project layout

- `template.yaml` - AWS SAM template that defines the Lambda function and API event mapping.
- `optimize_meal/` - function source code and `requirements.txt` for dependencies (numpy, scipy, scikit-learn, pydantic).
- `events/event.json` - sample event you can use for local testing.

## Key notes (serverless)

- The function depends on compiled Python packages (`scipy`, `scikit-learn`, `numpy`). Build inside an Amazon Linux container (SAM's `--use-container`) or provide Lambda Layers with prebuilt wheels to avoid binary incompatibilities.
- The optimizer can be memory/CPU hungry — increasing `MemorySize` (e.g. 1024 MB) in `template.yaml` may reduce cold-start time and avoid timeouts.
- Large dependency sets may produce large deployable artifacts. If zipped or unzipped package size limits are exceeded, consider using a container image for Lambda or placing heavy libs in a Layer.

## API (current)

- Path: `/optimize_meal`
- Method: currently defined in `template.yaml` as `GET` (note: handler expects a JSON body). For API Gateway usage that sends a JSON body, change the method to `POST` in `template.yaml` or ensure your client sends body with a GET mapping.

### Request body (JSON)

```json
{
  "foods": ["chicken", "white_rice", "broccoli"],
  "kcalories": 700,
  "carbs_percent": 40,
  "protein_percent": 30,
  "fat_percent": 30
}
```

### Response

- The Lambda returns a JSON string in the `body` with portion sizes and nutrition totals (e.g., grams per food, macro/micro totals, scores).

## Local development & testing (recommended)

Prerequisites:
- AWS SAM CLI
- Docker (for `--use-container` builds)
- Python 3.11+ (for local scripts if you run them outside Lambda)

Build and deploy (first deploy):

```bash
cd /Users/jdelpego/Documents/meal_prep_backend_serverless
# Build using Amazon Linux in Docker so binary wheels are compatible with Lambda
sam build --use-container
sam deploy --guided
```

Run locally (invoke function with the example event):

```bash
# invoke the function directly
sam local invoke OptimizeMealFunction --event events/event.json

# or run an API locally and hit the endpoint (recommended for HTTP testing)
sam local start-api
# then, in another terminal:
curl -X POST http://127.0.0.1:3000/optimize_meal \
  -H "Content-Type: application/json" \
  -d '{"foods":["chicken","white_rice","broccoli"],"kcalories":700,"carbs_percent":40,"protein_percent":30,"fat_percent":30}'
```

Notes:
- `sam local start-api` exposes the API at `http://127.0.0.1:3000` by default. The function logical id is `OptimizeMealFunction` (see `template.yaml`).
- If `sam build` fails compiling SciPy or scikit-learn, ensure Docker is running and try again. If it still fails because of low disk or memory on your machine, build a Lambda Layer or use an ECR container image.

## Deployment tips

- If dependency size is an issue:
  - Create an AWS Lambda Layer containing prebuilt wheels for Amazon Linux and reference it from your function.
  - Or package the application as a container image and push to ECR; Lambda container images allow larger sizes and full control of build environment.
- Increase `MemorySize` inside `template.yaml` for better performance with heavy numeric libraries.

## Troubleshooting

- Invalid module / import errors after deployment: likely caused by building wheels on macOS. Use `sam build --use-container` to ensure Amazon Linux-compatible wheels.
- Function times out: increase `Timeout` or `MemorySize` in `template.yaml`.
- CloudFormation errors about missing logical resources: ensure logical IDs in `Resources` match `Outputs` (this repo's template was corrected to use `OptimizeMealFunction`).

## Credits and history

This repository started as a local (non-serverless) Python project and was adapted to a serverless SAM-based deployment. The original README content and details were preserved and updated for serverless usage.

## Contact

If you want me to add a `Makefile`, CI/CD pipeline, a Lambda Layer build script, or a container image workflow, tell me which you prefer and I can add it.
