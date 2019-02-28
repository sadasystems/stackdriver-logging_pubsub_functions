stackdriver-logging_pubsub_functions
=====================================

Summary
---------
this document show steps to filter stackdriver logs, stream them to cloud pub/sub and trigger a cloud function to further process them.

For the sake of the demo we will be creating a kubernetes cluster which we will be using to filter its logs and ship them to Cloud PubSub.

Steps
------

1. create a k8s cluster called log-demo

        zone=us-west1
        gcloud container clusters create log-demo-cluster --zone $zone

2. create a pubsub topic where we will send our cluster logs

        gcloud pubsub topics create log-demo-cluster-errors

3. create a function to process the pubsub messages(logs)

Here, we are simply printing back the filtered log but it's up to you to process the log as you wish and perhaps store it later in a storage service like cloud BigTable, BigQuery, GCS. you can write your function in any of the following: python, nodejs, GO.

        cat<< EOF > main.py
        def hello_pubsub(data, context):
            """Background Cloud Function to be triggered by Pub/Sub.
            Args:
                data (dict): The dictionary with data specific to this type of event.
                context (google.cloud.functions.Context): The Cloud Functions event
                metadata.
            """
            import base64

            if 'data' in data:
                payload = base64.b64decode(data['data']).decode('utf-8')
            else:
                payload = 'EMPTY'
            print('I am a log entry, do somehting with me: {}!'.format(payload))
        EOF

4. deploy the function while specifying our previously created topic as the trigger

       gcloud functions deploy hello_pubsub --source `pwd` --runtime python37 --trigger-topic log-demo-cluster-errors

5. create a pubsub log export sink while filtering logs specific to our gke cluster which has severity `warning` and above:
`NOTE`: change PROJECT_ID to your project id
        gcloud beta logging sinks create pub-sub-sink \
            pubsub.googleapis.com/projects/[PROJECT_ID]/topics/log-demo-cluster-errors \
            --log-filter='resource.type="container" AND resource.labels.cluster_name="log-demo-cluster" AND severity>=WARNING'

6. get the pubsub sink writer identity service account

        sink_sa=$(gcloud beta logging sinks describe pub-sub-sink --format="(writerIdentity)" | awk '{print $2}')

7. Grant role of cloud pub sub publisher to the service account
gcloud projects add-iam-policy-binding rad-tests --member=$sink_sa --role=roles/pubsub.publisher

8. Create some errors withing the cluster (deploy many replicas)

        kubectl run nginx --image=nginx --replicas=20

9. Get the logs from our cloud function execution:

        gcloud functions logs read hello_pubsub
