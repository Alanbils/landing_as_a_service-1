# Landing as a Service Infrastructure

This repository contains a Terragrunt-based infrastructure setup for a serverless application that generates landing pages using AWS Bedrock. The infrastructure is organized under the `infrastructure/` directory with reusable Terraform modules and environment-specific configurations for `dev` and `prod` in `us-west-2`.

## Structure

```
infrastructure/
  terraform_modules/        # Reusable Terraform modules
  terraform/environment/
    dev/          # Development environment
    prod/         # Production environment
```

Each environment deploys:
- S3 buckets for input HTML and generated output
- IAM roles for the Lambda function
- A Lambda function that calls Bedrock
- API Gateway for invoking the Lambda
- Cognito user pool for authentication
- CloudFront distribution to serve generated pages

Before running Terragrunt you must set the bucket and region for storing
Terraform state. For example:

```bash
export TG_STATE_BUCKET=laas-dev-tfstate
export TG_REGION=us-west-2
```

Before applying Terragrunt, build the Lambda deployment package:

```bash
cd infrastructure/terraform_modules/lambda
./build.sh
```

Run `terragrunt run-all apply` from the desired environment directory to deploy.

## Prerequisites

- AWS credentials configured (via the AWS CLI or environment variables).
- Terraform **1.3+** and Terragrunt **0.47+** installed.
- Access to the Amazon Bedrock service in your AWS account.
## Bedrock model
The Lambda uses the `BEDROCK_MODEL_ID` variable to pick a model. For image generation set this to `amazon.titan-image-generator-v1` in the environment terragrunt files.


## Deploying environments

1. Change into the environment directory (`infrastructure/terraform/environment/dev` or `infrastructure/terraform/environment/prod`).
2. Run `terragrunt run-all init` followed by `terragrunt run-all apply`.

Terragrunt will provision all buckets, roles, the Lambda function, API Gateway, Cognito user pool and CloudFront distribution for that environment.

## Running the front-end and triggering the Lambda

Upload an HTML file (for example `index.html`) to the `s3_input` bucket created
during deployment. The Lambda reads this template from S3. To trigger the page
generation you POST a JSON payload with the following fields to the API Gateway
endpoint:

```
{
  "imagen": "URL de la imagen",
  "titulo": "Título principal",
  "subtitulo": "Subtítulo",
  "beneficios": ["Beneficio 1", "Beneficio 2", "Beneficio 3"],
  "cta": "Texto del botón CTA"
}
```

The `beneficios` field must be an array of strings. When using the sample front‑end, enter one benefit per line in the textarea and the script will split them into the required array format.

An invocation using `curl` might look like:

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"imagen":"https://example.com/hero.jpg","titulo":"Mi producto","subtitulo":"Subtítulo","beneficios":["Beneficio 1","Beneficio 2"],"cta":"Comprar"}' \
$(terragrunt output -raw api_endpoint)
```

You can also test the function directly from the AWS Lambda console. Create a
new test event using the following JSON payload:

```json
{
  "body": "{\"imagen\":\"https://example.com/hero.jpg\",\"titulo\":\"Mi producto\",\"subtitulo\":\"Subtítulo\",\"beneficios\":[\"Beneficio 1\",\"Beneficio 2\"],\"cta\":\"Comprar\"}",
  "isBase64Encoded": false
}
```

A minimal front-end could simply POST the HTML content to this API URL.

Create a `web/config.js` file by copying `web/config.example.js` and fill in
your values. The file should export a global `config` object with the following
fields:
`apiEndpoint`, `cloudfrontUrl`, `userPoolId` and `userPoolClientId`.

## Where to find the generated output

The Lambda writes the generated page to the `s3_output` bucket. It is also served via the CloudFront distribution. Obtain the distribution domain name with:

```bash
terragrunt output -raw distribution_domain_name
```

Navigate to that domain or open the object from the output S3 bucket to view the resulting landing page.

## Sample response

The Lambda returns a JSON body similar to the following on success:

```json
{
  "htmlUrl": "https://bucket.s3.amazonaws.com/12345.html?AWSAccessKeyId=...",
  "imagePresignedUrl": "https://bucket.s3.amazonaws.com/images/67890.png?AWSAccessKeyId=...",
  "imageS3Url": "s3://bucket/images/67890.png",
  "promptUsado": "texto del prompt utilizado"
}
```

The script in `web/main.js` expects these fields to be present in the response.


## Serving the sample web page

Follow these steps to try the simple front‑end included under `web/`:

1. Copy the example configuration and fill in the required values:

   ```bash
   cp web/config.example.js web/config.js
   ```

   Edit `web/config.js` and provide your `apiEndpoint`, `cloudfrontUrl`,
   `userPoolId` and `userPoolClientId` (these can be obtained with
   `terragrunt output`).

2. Serve `web/index.html` locally or upload the files to an S3 bucket
   and front it with CloudFront. For quick local testing you can run:

   ```bash
   npx http-server web
   # or
   python3 -m http.server --directory web 8080
   ```

   Then open the printed URL in your browser.

Submitting the form sends a JSON body containing `imagen`, `titulo`, `subtitulo`,
`beneficios` and `cta` to the configured API Gateway endpoint. API Gateway invokes
the Lambda function, which returns a JSON object with `htmlUrl`, `imagePresignedUrl`,
`imageS3Url` and `promptUsado`. The script in `web/main.js` uses `htmlUrl` to
redirect the browser to the generated page.



## License

This project is licensed under the [MIT License](LICENSE).

