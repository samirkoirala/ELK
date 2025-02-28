#  DevOps Assessment

## 1. Kubernetes Cluster Setup
Cluster Status

        root@masternode:~# kubectl get nodes
		NAME         STATUS   ROLES                  AGE    VERSION
		masternode   Ready    control-plane,master   2d8h   v1.31.5+k3s1
		workernode   Ready    <none>                 2d8h   v1.31.5+k3s1
Setup Description
			 - Used K3s for lightweight Kubernetes deployment
			 -  Master node runs the control plane and etcd
			 -  Worker node joined using the node token from master
			    `sudo  cat /var/lib/rancher/k3s/server/node-token`
			 -  Used default networking components provided by k3s

## 2. Java Application Deployment
**Dockerfile**

    FROM  tomcat:10.1
	COPY  sample.war  /usr/local/tomcat/webapps/
	EXPOSE  8080
	CMD  ["catalina.sh",  "run"]

**Kubernetes Manifests**

	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: java-app
	spec:
	  replicas: 2
	  selector:
	    matchLabels:
	      app: java-app
	  template:
	    metadata:
	      labels:
	        app: java-app
	    spec:
	      containers:
	      - name: java-app
	        image: java-app:1.0
	        ports:
	        - containerPort: 8080

### Deployment Logs

	kubectl get pods
	NAME                       READY   STATUS    RESTARTS      AGE
	java-app-fc68fdf67-g7vmm   1/1     Running   1 (29h ago)   2d5h
	java-app-fc68fdf67-tttl7   1/1     Running   0             26h


## 3. Service Exposure (NodePort + Ingress)
NodePort Service (service-nodeport.yaml):

	apiVersion: v1
	kind: Service
	metadata:
	  name: java-app-nodeport
	spec:
	  type: NodePort
	  ports:
	  - port: 8080
	    targetPort: 8080
	    nodePort: 30080
	  selector:
	    app: java-app

Ingress Resource (ingress.yaml):

    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: java-app-ingress
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /
    spec:
      rules:
      - host: java-app.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: java-app-nodeport
                port:
                  number: 8080

### Traffic Routing Explanation

 -  External request to java-app.example.com is received by the Ingress Controller
-   NGINX Ingress Controller routes traffic based on hostname and path
-   Traffic is forwarded to the java-app Service on port 8080
-   Service routes to one of the java-app pods based on selector
-   The application responds through the same path in reverse

# 4. ELK Stack and Filebeat Setup
***Deploy Filebeat as a DaemonSet in Kubernetes***

**Filebeat configuration (filebeat-config.yaml):**

	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: filebeat-config
	  namespace: default
	data:
	  filebeat.yml: |-
	    filebeat.inputs:
	    - type: container
	      paths:
	        - /var/log/containers/*java-app*.log
	      json.keys_under_root: true
	      json.add_error_key: true
	      json.message_key: log

	    processors:
	      - add_kubernetes_metadata:
	          host: ${NODE_NAME}
	          matchers:
	          - logs_path:
	              logs_path: "/var/log/containers/"

	    output.logstash:
	      hosts: ["192.168.1.70:5044"]

**DaemonSet for Filebeat (filebeat-daemonset.yaml):**

	apiVersion: apps/v1
	kind: DaemonSet
	metadata:
	  name: filebeat
	  namespace: default
	  labels:
	    app: filebeat
	spec:
	  selector:
	    matchLabels:
	      app: filebeat
	  template:
	    metadata:
	      labels:
	        app: filebeat
	    spec:
	      serviceAccountName: filebeat
	      terminationGracePeriodSeconds: 30
	      hostNetwork: true
	      dnsPolicy: ClusterFirstWithHostNet
	      containers:
	      - name: filebeat
	        image: docker.elastic.co/beats/filebeat:7.16.2
	        args: [
	          "-c", "/etc/filebeat.yml",
	          "-e",
	        ]
	        env:
	        - name: NODE_NAME
	          valueFrom:
	            fieldRef:
	              fieldPath: spec.nodeName
	        securityContext:
	          runAsUser: 0
	        resources:
	          limits:
	            memory: 200Mi
	          requests:
	            cpu: 100m
	            memory: 100Mi
	        volumeMounts:
	        - name: config
	          mountPath: /etc/filebeat.yml
	          readOnly: true
	          subPath: filebeat.yml
	        - name: data
	          mountPath: /usr/share/filebeat/data
	        - name: varlibdockercontainers
	          mountPath: /var/lib/docker/containers
	          readOnly: true
	        - name: varlog
	          mountPath: /var/log
	          readOnly: true
	      volumes:
	      - name: config
	        configMap:
	          defaultMode: 0600
	          name: filebeat-config
	      - name: varlibdockercontainers
	        hostPath:
	          path: /var/lib/docker/containers
	      - name: varlog
	        hostPath:
	          path: /var/log
	      - name: data
	        hostPath:
	          path: /var/lib/filebeat-data
	          type: DirectoryOrCreate

**ServiceAccount for Filebeat (filebeat-rbac.yaml):**

	apiVersion: v1
	kind: ServiceAccount
	metadata:
	  name: filebeat
	  namespace: default
	---
	apiVersion: rbac.authorization.k8s.io/v1
	kind: ClusterRole
	metadata:
	  name: filebeat
	rules:
	- apiGroups: [""] 
	  resources:
	  - namespaces
	  - pods
	  verbs:
	  - get
	  - watch
	  - list
	---
	apiVersion: rbac.authorization.k8s.io/v1
	kind: ClusterRoleBinding
	metadata:
	  name: filebeat
	subjects:
	- kind: ServiceAccount
	  name: filebeat
	  namespace: default
	roleRef:
	  kind: ClusterRole
	  name: filebeat
	  apiGroup: rbac.authorization.k8s.io

**Apply Filebeat Configurations**

    kubectl apply -f filebeat-rbac.yaml
	kubectl apply -f filebeat-config.yaml
	kubectl apply -f filebeat-daemonset.yaml

***YAML files for deploying Elasticsearch, Logstash, and Kibana***
 Deployed ELK on docker using docker compose from my host windows pc as my VMs resource could not run ELK properly (lack of resources)

logstash.conf

	 input {

	beats {

	port => 5044

	}

	}

	  

	filter {

	if [kubernetes][container][name] == "java-app" {

	grok {

	match => { "message" => "%{COMBINEDAPACHELOG}" }

	}

	}

	}

	  

	output {

	elasticsearch {

	hosts => ["elasticsearch:9200"]

	index => "k8s-logs-%{+YYYY.MM.dd}"

	}

	}

logstash.yaml

    http.host: "0.0.0.0"

docker-compose.yaml

	services:

	elasticsearch:

	image: docker.elastic.co/elasticsearch/elasticsearch:7.16.2

	container_name: elasticsearch

	environment:

	- discovery.type=single-node

	- bootstrap.memory_lock=true

	- "ES_JAVA_OPTS=-Xms512m -Xmx512m"

	ulimits:

	memlock:

	soft: -1

	hard: -1

	ports:

	- 9200:9200

	networks:

	- elk

	  

	logstash:

	image: docker.elastic.co/logstash/logstash:7.16.2

	container_name: logstash

	ports:

	- 5044:5044

	- 9600:9600

	volumes:

	- ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml

	- ./logstash/pipeline:/usr/share/logstash/pipeline

	networks:

	- elk

	depends_on:

	- elasticsearch

	  

	kibana:

	image: docker.elastic.co/kibana/kibana:7.16.2

	container_name: kibana

	ports:

	- 5601:5601

	environment:

	ELASTICSEARCH_HOSTS: http://elasticsearch:9200

	networks:

	- elk

	depends_on:

	- elasticsearch

	  

	networks:

	elk:

	driver: bridge

 **Sample output or screenshot from Kibana showing logs from the Java application**

https://photos.app.goo.gl/YhmmbS1gLk7M8iCa9