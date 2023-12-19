# cloudformation-templates
This repository contains public cloudformation templates for logzio customers to automate aws integrations deployments
## Templates
- `ecs-fargate`
- `ecs-fargate_collector`
## Development: Add a new template
- Create a new directory in `./aws` path with the name of your new template -
- Create your template file with the name `sam-template.yaml`
- Create a `README.md` for your template
- Create a new workflow for your template to upload to s3, following this example:
```yaml
name: Upload new release <<template_name>>
on:
  release:
    types: [published]

jobs:
  upload_to_buckets:
    if: contains(github.event.release.tag_name, '<<template_name>>')
    name: Upload to S3 buckets
    runs-on: ubuntu-latest
    strategy:
      matrix:
        aws_region:
          - 'us-east-1'
          - 'us-east-2'
          - 'us-west-1'
          - 'us-west-2'
          - 'eu-central-1'
          - 'eu-north-1'
          - 'eu-west-1'
          - 'eu-west-2'
          - 'eu-west-3'
          - 'sa-east-1'
          - 'ap-northeast-1'
          - 'ap-northeast-2'
          - 'ap-northeast-3'
          - 'ap-south-1'
          - 'ap-southeast-1'
          - 'ap-southeast-2'
          - 'ca-central-1'
    steps:
      - name: Check out the repo
        uses: actions/checkout@v3
      - name: create new version
        run: |
          cp ./aws/<<template_name>>/sam-template.yaml ./sam-template-${{ matrix.aws_region }}.yaml
          sed -i "s/<<REGION>>/${{ matrix.aws_region }}/" "./sam-template-${{ matrix.aws_region }}.yaml"
      - name: Upload to aws
        run: |
          sudo apt-get update
          sudo apt-get install awscli
          aws configure set aws_access_key_id ${{ secrets.AWS_ACCESS_KEY }}
          aws configure set aws_secret_access_key ${{ secrets.AWS_SECRET_KEY }}
          aws configure set region ${{ matrix.aws_region }}
          aws s3 cp ./sam-template-${{ matrix.aws_region }}.yaml s3://logzio-aws-integrations-${{ matrix.aws_region }}/<<template_name>>/${{ github.event.release.tag_name }}/sam-template.yaml --acl public-read
      - name: Clean
        run: |
          rm ./sam-template-${{ matrix.aws_region }}.yaml
```
Replace <<template_name>> with the name of your new template
- Create a new release with name, tag name according to your template and its version (example: `ecs-fargate-1.0.0`) 
