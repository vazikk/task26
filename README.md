# task26
1) Запустил 4 виртуалки (1 мастер и 3 воркера) <br>

   <img width="269" height="199" alt="image" src="https://github.com/user-attachments/assets/b450240f-cce7-4df7-9c7a-cb5dc4c8c149" /> <br>

2) Установил python, pip, git, ansible и начал установку kubespray: <br>
   ```
   git clone https://github.com/kubernetes-sigs/kubespray.git
   cd kubespray
   sudo pip3 install -r requirements.txt
   ```
   <br>
   Потом создаем inventory/mycluster/inventory.ini <br>
  
   ```
    [all]
    node1 ansible_host=172.31.94.61 ip=172.31.94.61
    node2 ansible_host=172.31.86.163 ip=172.31.86.163
    node3 ansible_host=172.31.91.200 ip=172.31.91.200
    node4 ansible_host=172.31.89.246 ip=172.31.89.246
    
    [kube_control_plane]
    node1
    
    [etcd]
    node1
    
    [kube_node]
    node2
    node3
    node4
    
    [calico_rr]
    
    [k8s_cluster:children]
    kube_control_plane
    kube_node
   ```

   После запускаем playbook предварительно добавив ssh-ключ для подключения <br>
   ```
   ansible-playbook -i inventory/mycluster/hosts.yaml cluster.yml -b
   ```
   cluster.yml - файл с тем что ansible должен выполнить на хостах <br>
   После всей скачки скопировал файл конфига и проверил что всё работает <br>

   <img width="505" height="133" alt="image" src="https://github.com/user-attachments/assets/65d9803e-04d2-4750-a021-cbeae8eff554" /> <br>

3) Создал namespaces <br>
   <img width="265" height="42" alt="image" src="https://github.com/user-attachments/assets/bdeec608-7ef3-402d-901f-e97b72f208bf" /> <br>
   <img width="275" height="25" alt="image" src="https://github.com/user-attachments/assets/c8477191-3d57-41dd-8417-1d9ccc4f9c1b" /> <br>

4) Создал директорию под Customize ``nginx-kustomize``, в ней создал ещё 2 директории ``base и overlays`` и внутри ``overlays`` создал ещё 3 директории ``test, prod, dev`` <br>

5) Начал создавать файлы сначала в BASE: <br>
   deployment.yaml <br>
   ```
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx:alpine
            ports:
            - containerPort: 80
            volumeMounts:
            - name: nginx-config
              mountPath: /usr/share/nginx/html
              readOnly: true
          volumes:
          - name: nginx-config
            configMap:
              name: nginx-config
   ```
   service.yaml <br>
   ```
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-service
    spec:
      selector:
        app: nginx
      ports:
      - port: 80
        targetPort: 80
      type: ClusterIP
   ```
  
   configmap.yaml <br> 
   ```
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: nginx-config
    data:
      index.html: |
        <!DOCTYPE html>
        <html>
        <head>
              <title>Base Environment</title>
          </head>
          <body>
              <h1>Current Environment: Base</h1>
          </body>
        </html>
   ```
   kustomization.yaml <br>
   ```
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    resources:
    - deployment.yaml
    - service.yaml
    - configmap.yaml
   ```

6) Теперь идёт настройка оверлеев: <br>
   1. dev <br>
      configmap-patch.yaml <br> 
      ```
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: nginx-config
      data:
        index.html: |
          <html>
          <head>
              <title>Dev Environment</title>
          </head>
          <body>
              <h1>Current Environment: Dev</h1>
          </body>
          </html>
      ```
      service-patch.yaml <br>
      ```
      apiVersion: v1
      kind: Service
      metadata:
        name: nginx-service
      spec:
        type: NodePort
        ports:
        - port: 80
          targetPort: 80
          nodePort: 30081
      ```
      kustomization.yaml <br>
      ```
      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      
      namespace: dev
      namePrefix: dev-
      
      resources:
      - ../../base
      
      patches:
      - path: configmap-patch.yaml
        target:
          kind: ConfigMap
          name: nginx-config
      - path: service-patch.yaml
        target:
          kind: Service
          name: nginx-service
      ```
   2. test <br>
      configmap-patch.yaml <br> 
      ```
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: nginx-config
      data:
        index.html: |
          <html>
          <head>
              <title>Test Environment</title>
          </head>
          <body>
              <h1>Current Environment: Test</h1>
          </body>
          </html>
      ```
      service-patch.yaml <br>
      ```
      apiVersion: v1
      kind: Service
      metadata:
        name: nginx-service
      spec:
        type: NodePort
        ports:
        - port: 80
          targetPort: 80
          nodePort: 30080

      ```
      kustomization.yaml <br>
      ```
      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      namespace: test
      namePrefix: test-
      resources:
      - ../../base
      patches:
      - path: configmap-patch.yaml
        target:
          kind: ConfigMap
          name: nginx-config
      - path: service-patch.yaml
        target:
          kind: Service
          name: nginx-service
      ```
   3. Prod <br>
      configmap-patch.yaml <br> 
      ```
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: nginx-config
      data:
        index.html: |
          <html>
          <head>
              <title>Prod Environment</title>
          </head>
          <body>
              <h1>Current Environment: Prod</h1>
          </body>
          </html>
      ```
      service-patch.yaml <br>
      ```
      apiVersion: v1
      kind: Service
      metadata:
        name: nginx-service
      spec:
        type: NodePort
        ports:
        - port: 80
          targetPort: 80
          nodePort: 30082


      ```
      kustomization.yaml <br>
      ```
      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      
      namespace: prod
      namePrefix: prod-
      
      resources:
      - ../../base
      
      patches:
      - path: configmap-patch.yaml
        target:
          kind: ConfigMap
          name: nginx-config
      - path: service-patch.yaml
        target:
          kind: Service
          name: nginx-service
      ```
7) Делоим все окружения командой `` kubectl apply -k`` (-k указвает на использование kustomize) <br>
<img width="826" height="47" alt="image" src="https://github.com/user-attachments/assets/a452f21a-fe23-4c11-802d-3ae1e05b165a" />
<img width="757" height="48" alt="image" src="https://github.com/user-attachments/assets/c0c44dde-7da1-4cee-a4d4-b88b478bd77e" />

8) Результат: <br>
<img width="622" height="188" alt="image" src="https://github.com/user-attachments/assets/afd4bf76-835f-41a1-ad44-b29e11d698d9" />
<img width="560" height="161" alt="image" src="https://github.com/user-attachments/assets/a703ace5-ee8b-42e2-933b-b2cea1337b5c" />
<img width="603" height="175" alt="image" src="https://github.com/user-attachments/assets/18b475d9-d56f-454b-9c08-158fd4cb6989" />



      
