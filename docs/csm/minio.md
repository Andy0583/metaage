**上傳Image**
```
podman login harbor.ocp.andy.com -u admin -p Harbor12345
podman load -i /root/minio-image.tar
podman tag quay.io/minio/minio:latest harbor.ocp.andy.com/ocp4/minio:latest
podman push harbor.ocp.andy.com/ocp4/minio:latest
```

**建立 imagePullSecret**
```
oc create secret docker-registry harbor-secret \
  --docker-server=harbor.ocp.andy.com \
  --docker-username=admin \
  --docker-password=Harbor12345 \
  -n test
```

**建立 MinIO Deployment**
```
cat > minio-deploy.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
        - name: minio
          image: harbor.ocp.andy.com/ocp4/minio:latest
          args:
            - server
            - /data
            - --console-address
            - ":9001"
          env:
            - name: MINIO_ROOT_USER
              value: "minioadmin"
            - name: MINIO_ROOT_PASSWORD
              value: "minioadmin123"
          ports:
            - containerPort: 9000
              name: api
            - containerPort: 9001
              name: console
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: test-nas-pvc
EOF

oc apply -f minio-deploy.yaml
```

**建立 Service**
```
cat > minio-svc.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: test
spec:
  selector:
    app: minio
  ports:
    - name: api
      port: 9000
      targetPort: 9000
    - name: console
      port: 9001
      targetPort: 9001
EOF

oc apply -f minio-svc.yaml
```

**建立Route**
```
oc create route edge minio-console --service=minio --port=9001 -n test
oc create route edge minio-api --service=minio --port=9000 -n test

oc get pods -n test
```

**查詢連線網址**
```
oc get route minio-console -n test -o jsonpath='{.spec.host}'
```
帳號：minioadmin <br>
密碼：minioadmin123

