# kafka_k8s_using_helm

Prerequisite
=============

Running EKS/AKS cluster.

Steps:
1. Install kafka cluster using helm chart

        helm repo add bitnami https://charts.bitnami.com/bitnami
        helm install my-kafka bitnami/kafka --version 26.8.5

2. Capture the Login details in client.properties configuration file to connect to kafka:

        security.protocol=SASL_PLAINTEXT
        sasl.mechanism=SCRAM-SHA-256
        sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
        username="user1" \
        password="$(kubectl get secret my-kafka-user-passwords --namespace default -o jsonpath='{.data.client-passwords}' | base64 -d | cut -d , -f 1)";

3. Create a pod that you can use as a Kafka client run the following commands:

        kubectl run my-kafka-client --restart='Never' --image docker.io/bitnami/kafka:3.6.1-debian-11-r6 --namespace default --command -- sleep infinity
        kubectl cp --namespace default /path/to/client.properties my-kafka-client:/tmp/client.properties
        kubectl exec --tty -i my-kafka-client --namespace default -- bash

4. PRODUCER:

        kafka-console-producer.sh --producer.config /tmp/client.properties \
        --broker-list my-kafka-controller-0.my-kafka-controller-headless.default.svc.cluster.local:9092,my-kafka-controller-1.my-kafka-controller-headless.default.svc.cluster.local:9092,my-kafka-controller-2.my-kafka-controller-headless.default.svc.cluster.local:9092 \
        --topic quickstart-events

5. CONSUMER:

                 kafka-console-consumer.sh \
                --consumer.config /tmp/client.properties \
                --bootstrap-server my-kafka.default.svc.cluster.local:9092 \
                --topic quickstart-events \
                --from-beginning
