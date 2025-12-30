
pipeline {
  agent any

  environment {
    // LocalStack edge port (direct). If you want NGINX later, change this to your proxy URL.
    LOCALSTACK_URL = 'http://localhost:4566'

    // Dummy credentials for LocalStack (signed requests). For --no-sign-request, remove these.
    AWS_ACCESS_KEY_ID     = 'test'
    AWS_SECRET_ACCESS_KEY = 'test'
    AWS_DEFAULT_REGION    = 'us-east-1'

    // Site specifics
    BUCKET_NAME = 'my-site-bucket'   // <— change if needed
    SITE_DIR    = 'dist'             // <— change to your build output
    HEALTH_URL  = 'http://localhost:4566/_localstack/health'
    HEALTH_TIMEOUT_SECONDS = '120'   // how long to wait for LocalStack
    HEALTH_POLL_INTERVAL   = '3'     // seconds between checks
  }

  stages {
    stage('Wait for LocalStack health') {
      steps {
        sh '''
          echo "Waiting for LocalStack to be healthy at: ${HEALTH_URL}"
          end=$(( $(date +%s) + ${HEALTH_TIMEOUT_SECONDS} ))
          while [ $(date +%s) -lt $end ]; do
            status=$(curl -s "${HEALTH_URL}" | jq -r '.services | to_entries[] | select(.key=="s3") | .value')
            # If jq not available, you can just check for non-empty body:
            if [ -n "$status" ]; then
              echo "LocalStack health (s3): $status"
            else
              echo "LocalStack health endpoint reachable."
            fi

            # Consider healthy when the endpoint returns 200 and non-empty JSON
            if curl -fsS "${HEALTH_URL}" >/dev/null; then
              echo "LocalStack health OK."
              exit 0
            fi
            echo "Not healthy yet. Retrying in ${HEALTH_POLL_INTERVAL}s…"
            sleep ${HEALTH_POLL_INTERVAL}
          done

          echo "LocalStack did not become healthy within ${HEALTH_TIMEOUT_SECONDS}s."
          exit 1
        '''
      }
    }

    stage('Build site') {
      steps {
        // Replace with your build (npm/yarn/maven/gradle). This is a placeholder.
        sh '''
          [ -d ${SITE_DIR} ] || mkdir -p ${SITE_DIR}
          echo "<html><body><h1>Hello from Jenkins → LocalStack S3</h1></body></html>" > ${SITE_DIR}/index.html
        '''
      }
    }

    stage('Create S3 bucket (idempotent)') {
      steps {
        sh '''
          # Create bucket if missing
          if ! aws s3api head-bucket --bucket ${BUCKET_NAME} --endpoint-url ${LOCALSTACK_URL} >/dev/null 2>&1; then
            echo "Creating bucket ${BUCKET_NAME} in LocalStack…"
            aws s3api create-bucket \
              --bucket ${BUCKET_NAME} \
              --region ${AWS_DEFAULT_REGION} \
              --endpoint-url ${LOCALSTACK_URL}

            echo "Enabling static website hosting…"
            aws s3api put-bucket-website \
              --bucket ${BUCKET_NAME} \
              --website-configuration '{
                "IndexDocument": {"Suffix":"index.html"},
                "ErrorDocument": {"Key":"index.html"}
              }' \
              --endpoint-url ${LOCALSTACK_URL}

            echo "Setting public-read policy (LocalStack only)…"
            aws s3api put-bucket-policy \
              --bucket ${BUCKET_NAME} \
              --policy "{
                \\"Version\\": \\"2012-10-17\\",
                \\"Statement\\": [{
                  \\"Effect\\": \\"Allow\\",
                  \\"Principal\\": \\"*\\",
                  \\"Action\\": [\\"s3:GetObject\\"],
                  \\"Resource\\": \\"arn:aws:s3:::${BUCKET_NAME}/*\\"
                }]
              }" \
              --endpoint-url ${LOCALSTACK_URL}
          else
            echo "Bucket ${BUCKET_NAME} already exists."
          fi
        '''
      }
    }

    stage('Upload website files') {
      steps {
        sh '''
          echo "Syncing ${SITE_DIR}/ to s3://${BUCKET_NAME}/ via ${LOCALSTACK_URL}…"
          aws s3 sync ${SITE_DIR}/ s3://${BUCKET_NAME}/ \
            --delete \
            --endpoint-url ${LOCALSTACK_URL}
        '''
      }
    }

    stage('Show test URL(s)') {
      steps {
        sh '''
          echo "LocalStack S3 path-style URL (served by LocalStack):"
          echo "  ${LOCALSTACK_URL}/${BUCKET_NAME}/index.html"
          echo ""
          echo "If you later use NGINX to front LocalStack, adapt LOCALSTACK_URL to your proxy and use:"
          echo "  http://<your-nginx-host>:<port>/${BUCKET_NAME}/index.html"
        '''
      }
    }
  }

  post {
    failure {
      echo 'Pipeline failed. Check LocalStack health and endpoint URL.'
    }
  }
}
