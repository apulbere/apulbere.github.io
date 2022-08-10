---
layout: post
title: Running Java shebang with Kubernetes
tags: [jep-330, java 11, java 17, kubernetes]
---

Java 11, among other features, introduced the possibility of running a single Java source file. We are going to see a practical example of this feature running with kubernetes.

```hello-java.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: java-shebang-config-map
data:
  script.sh: |
    #!/usr/bin/java --source 17

    public class App {

        public static void main(String args[]) {
            System.out.println("Hello Java Script");
        }
    }
---
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  volumes:
  - name: java-shebang-config
    configMap:
      name: java-shebang-config-map
      defaultMode: 0777
  containers:
  - image: openjdk:17.0.2-oracle
    command: ["./scripts/script.sh"]
    name: app
    volumeMounts:
      - mountPath: /scripts
        name: java-shebang-config
  restartPolicy: Never
```

```
kubectl create -f hello-java.yml
kubectl logs app
```