# kafka_k8s_using_helm

Prerequisite
=============

Running EKS/AKS cluster.

Note: If you done have EBS-CSI add on please add it:

      eksctl create iamserviceaccount     --name ebs-csi-controller-sa     --namespace kube-system     --cluster eksdemo     --role-name AmazonEKS_EBS_CSI_DriverRole     --role-only     --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy     --approve
      eksctl create addon --name aws-ebs-csi-driver --cluster eksdemo --service-account-role-arn arn:aws:iam::176886134554:role/AmazonEKS_EBS_CSI_DriverRole --force --region us-east-1


Steps:
1. Install kafka cluster using helm chart:
 
            helm repo add bitnami https://charts.bitnami.com/bitnami
            helm install my-kafka bitnami/kafka --version 26.8.5 -f values.yaml

2. Capture the Login details in client.properties configuration file to connect to kafka:

            security.protocol=SASL_PLAINTEXT
            sasl.mechanism=SCRAM-SHA-256
            sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
            username="user1" \
            password="$(kubectl get secret my-kafka-user-passwords --namespace default -o jsonpath='{.data.client-passwords}' | base64 -d | cut -d , -f 1)";

3. Create a pod that you can use as a Kafka client run the following commands:
        
            kubectl run my-kafka-client --restart='Never' --image docker.io/bitnami/kafka:3.6.1-debian-11-r6 --namespace default --command -- sleep infinity
            kubectl cp --namespace default client.properties  my-kafka-client:/tmp/client.properties
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

Till this the setup is working if you are running client in same cluster. But if you want to access kafka from outside cluster.

            helm upgrade my-kafka bitnami/kafka --version 26.8.5 -f values.yaml \
            --set service.type=LoadBalancer \
            --set externalAccess.enabled=true \
            --set "externalAccess.autoDiscovery.enabled=true" \
            --set rbac.create=true \
            --set controller.automountServiceAccountToken=true \
            --set broker.automountServiceAccountToken=true

Now if you look at the services it should look like:

            # kubectl get svc
            NAME                                    TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)                         AGE
            kubernetes                              ClusterIP      10.100.0.1       <none>                                                                    443/TCP                         121m
            my-kafka                                LoadBalancer   10.100.28.153    aeb04757618a643ab9643df507065ee6-1468505733.us-east-1.elb.amazonaws.com   9092:31747/TCP,9095:32373/TCP   106m
            my-kafka-controller-0-external          LoadBalancer   10.100.24.67     aea296f7241c8479ebecfe82015eca09-906998625.us-east-1.elb.amazonaws.com    9094:31912/TCP                  20m
            my-kafka-controller-1-external          LoadBalancer   10.100.187.106   a19e00701b8e543259547e2a8d899594-1223245581.us-east-1.elb.amazonaws.com   9094:31854/TCP                  20m
            my-kafka-controller-2-external          LoadBalancer   10.100.134.73    a8f8e8ab06d1a4b669b5e10f66f2314a-880748335.us-east-1.elb.amazonaws.com    9094:31291/TCP                  20m
            my-kafka-controller-headless            ClusterIP      None             <none>                                                                    9094/TCP,9092/TCP,9093/TCP      106m

You can access the kakfa for outside using the my-kafka-controller-*-external External_IP addereses like below.

            kafka-console-producer.sh \
                        --producer.config /tmp/client.properties \
                        --broker-list aea296f7241c8479ebecfe82015eca09-906998625.us-east-1.elb.amazonaws.com:9094,a19e00701b8e543259547e2a8d899594-1223245581.us-east-1.elb.amazonaws.com:9094,a8f8e8ab06d1a4b669b5e10f66f2314a-880748335.us-east-1.elb.amazonaws.com:9094 \
                        --topic test

![image](https://github.com/tushardashpute/kafka_k8s_using_helm/assets/74225291/38202f64-6cbe-4c52-8705-2e8965610afa)

            kafka-console-consumer.sh \
            --consumer.config /tmp/client.properties \
            --bootstrap-server aeb04757618a643ab9643df507065ee6-1468505733.us-east-1.elb.amazonaws.com:9092 \
            --topic test \
            --from-beginning

![image](https://github.com/tushardashpute/kafka_k8s_using_helm/assets/74225291/501ea8c8-0399-48e3-8c16-30eef917dd40)
