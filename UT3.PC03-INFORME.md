# INFORME DE APP KUBERNETES
## ESQUEMA DE FUNCIONAMIENTO DE LA APP
Para el proyecto se optó por el esquema proporcionado, en la cual tiene una parte de Base de datos y otra parte de la App, en la cual el usuario final tiene entrar en la web y el desarrollador puede entrar en la base de datos y los usuarios normales no puede, y todo esto creado con Kubernetes.

![ESQUEMA](/img/esquema-app-kube.png)
### ARCHIVOS QUE SE CREO FINALMENTE
Para generar todos los ficheros se escogió la siguiente estructurá para poder desarrollar la APP.

![FICHEROS](/img/ficheros-finales.png)
## BASE DE DATOS
Ahora ante de todo vamos a configurar la parte de **BASE DE DATOS**, como se menciono ante esta parte solo puede entrar el desarrollador y el administrador de la base de datos y de la aplicación.
#### VOLÚMENES PERSISTENTE
Vamos a generar los volúmenes, que son los recursos que se utilizara en el Kubernetes, para ello vamos a generar un **pv** y **PVC** en la cual se añadirá cuando se cree el **deployments** de **postgres** para poder crear la persistencia de la misma.
###### **persistence-volume.yaml**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgress-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: "/mnt/data/postgres"
```
###### **persistence-volume-claim.yaml**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgress-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
#### CONFIGMAPS
Ahora vamos a generar el configmap en la cual tiene un **script** de **bash** que vamos a poder crear un usuario en nuestra base datos que es necesario para que la **APP** funcione correctamente.
###### **configmap-postgress-initbd.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgress-initdb-config
data:
  postgres_init.sh: |-
    #!/bin/bash
    set -e
    psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
      # Crear usuario billingapp
      CREATE USER billingapp WITH PASSWORD 'qwerty';
      # Crear base de datos billingapp_db
      CREATE DATABASE billingapp_db;
      # Otorgar privilegios a billingapp y postgres
      GRANT ALL PRIVILEGES ON DATABASE billingapp_db TO billingapp;
      GRANT ALL PRIVILEGES ON DATABASE billingapp_db TO postgres;
    EOSQL
```
#### SECRETOS
Ahora vamos a producir los ficheros secretos en las cual vamos a poner la información **sensible** de una forma oculta o encriptada, la cual vamos a proteger la información sensible en el servidor de Kubernete, las que vamos a proteger sería para **postgres** y **pgadmin**.
###### **secret-postgress.yaml**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgress-secret
type: Opaque
data:
  user: cG9zdGdyZXM= # postgres
  password: cXdlcnR5 # qwerty
```
###### **secret-pgadmin.yaml**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pgadmin-secret
type: Opaque
data:
  email: YWRtaW5AcGdhZG1pbi5jb20= #admin@pgadmin.com
  password: cXdlcnR5 #qwerty
```
#### DEPLOYMENTS
Ahora vamos a generar los **deployments** de **postgres** y **pgadmin**, vamos a crear por primero el deployment de postgres y luego vamos a crear el de pgadmin con lo que vamos a usar variables de entorno que se utilizara para el usuario y la contraseña que se creó en los **secretos**.

En el deployment **postgres** se va a utilizar los volúmenes generados anteriormente que son de **configmaps** y de **persistencia** en la base de datos.
###### **deployment-postgress.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgress
  template:
    metadata:
      labels:
        app: postgress
    spec:
      containers:
      - name: postgress
        image: postgres:13
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgress-secret
              key: user
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgress-secret
              key: password
        - name: POSTGRES_DB
          value: billingapp_db
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgress-data
          mountPath: /var/lib/postgresql/data
        - name: postgress-initdb
          mountPath: /docker-entrypoint-initdb.d
      volumes:
      - name: postgress-data
        persistentVolumeClaim:
          claimName: postgress-pv-claim
      - name: postgress-initdb
        configMap:
          name: postgress-initbd-config
          items:
          - key: postgres_init.sh
            path: initdb.sh
```

En este caso solo vamos a utilizar el **secreto** generado anteriormente del usuario y contraseña de pgadmin.
###### **deployment-pgadmin.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgadmin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pgadmin
  template:
    metadata:
      labels:
        app: pgadmin
    spec:
      containers:
      - name: pgadmin
        image: dpage/pgadmin4:latest
        env:
        - name: PGADMIN_DEFAULT_EMAIL
          valueFrom:
            secretKeyRef:
              name: pgadmin-secret
              key: email
        - name: PGADMIN_DEFAULT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: pgadmin-secret
              key: password
        ports:
        - containerPort: 80
```
#### SERVICIOS
Ahora vamos a generar los servicios para poder tener acceso a la base de datos.

Para ello, tanto pgadmin y postgress en **nodeport** tiene el puerto **30432** y **30802** para poder conectarse y verlo desde la web usando el puerto que hemos puesto.
###### **service-postgress.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgress
spec:
  selector:
    app: postgress
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
      nodePort: 30432
  type: LoadBalancer
```
###### **service-pgadmin.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: pgadmin
spec:
  selector:
    app: pgadmin
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
      nodePort: 30802
  type: LoadBalancer
```
##### DENTRO DE PGADMIN
Ahora vamos a entrar en la web de pgadmin en al cual vamos a enlazar el **pgadmin** y **postgres**, si toda la configuración fue un éxito nos dejara entrar sin ningún problema con las credenciales que se pusieron los **secretos**.

![PGADMIN](/img/pgadmin-kuberbete.png)

Ahora vamos a meter la dirección **IP** y el **Puerto** y pondremos los usuarios y la contraseña que se creó en el **configmaps**.

![POSTGRES-PGADMIN](/img/pgadmin-agregar-postgres.png)
## APLICACIÓN
### BACKEND
Ahora vamos a utilizar el **dockerfile** en el cual está en la carpeta de **java**, en cuál vamos a usar el siguiente comando en **Docker**.
```bash
docker build --build-arg JAR_FILE=app-0.0.4-SNAPSHOT.jar -t backend-java:v1 .
```
 Una vez generado vamos a subirlo a **docker hub** para tenerlo a mano si en algún momento borramos la imagen del Docker por error, y se pueda descargar sin ningún problema.
  - Vamos a iniciar con la cuenta **Docker hub** para poder a subir nuestro Docker.
```bash
  docker login -u sunamylol
```
  - Ahora vamos a ponerle el **tag** para ponerle un nombre para cuando subamos al imagen.
```bash
docker tag backend-java:v1 sunamylol/backend:v1
```
  - Ya podemos subir la imagen a **docker hub**
```bash
docker push sunamylol/backend:v1
```
Ahora vamos a generar el **deployment** de **backend** con la imagen del docker subida en **docker hub** y el puerto **7080** para que funcione correctamente y se replicara como un maximo de **3**.
###### **deployment-app-back.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: back-pps-ale
spec:
  replicas: 3
  selector:
    matchLabels:
      app: back-pps-ale
  template:
    metadata:
      labels:
        app: back-pps-ale
    spec:
      containers:
      - name: app-back
        image: sunamylol/backend:v1
        ports:
        - containerPort: 7080
          name: billingappbport
```
Ahora vamos a generar el servicio en le cuál va a poner en funcionamiento nuestro **backend**, en el cual vamos a poner como puerto **7080** y **30780** y usando esta vez **NodePort**.
###### **service-app-back.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-back
spec:
  selector:
    app: app-back
  ports:
    - name: app-back
      port: 7080
      targetPort: 7080
      nodePort: 30780
  type: NodePort
```
### FRONTEND
Ahora vamos a generar el Docker de la parte de **frontend**, para ello se pondrá los mismos comandos anteriormente mencionado.
```bash
docker build -t frontend-angular:v1 .
```
Ahora vamos a subir la imagen generada a **Docker hub**, con los mismos comando anteriores.
```bash
docker tag frontend-angular:v1 sunamylol/frontend:v1
docker push sunamylol/frontend:v1
```
Ahora vamos a generar el deployment con la imagen de **frontend**, con lo que se pueda replicar como máximo de **2**.
###### **deployment-app-frontend.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app-front-ale
  template:
    metadata:
      labels:
        app: app-front-ale
    spec:
      containers:
      - name: app-front-ale
        image: sunamylol/frontend:v1
        ports:
        - containerPort: 80
          name: billingappfport
```
Ahora vamos a generar el servicio en el cual el usuario final va a poder entrar a la página web y añada los datos en la base de datos con el formulario.
###### **service-app-front.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-front-service
  labels:
    app: app-front-ale
spec:
  selector:
    app: app-front-ale
  ports:
  - name: app-front-service
    port: 80
  type: NodePort
```
##### DENTRO DE LA WEB
Si todo está correcto, el usuario final podrá iniciar en la web sin ningún problema, y podrá añadir datos a la base de datos.

![FRONTEND](/img/web-frontend.png)
Ahora el usuario podrá añadir datos a la base de datos.
## COMANDOS MAS USADO EN LA PRACTICA
```bash
k apply -f [FICHERO.YAML]
k get deployment
k get pods
k get services
mk service [SERVICIOS] --url
k logs -l app=[SERVICIOS]
k exec -it [PODS] -- /bin/bash
k describe pod [PODS]
```