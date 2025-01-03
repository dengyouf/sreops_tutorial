# Mall 项目实战

## 1. 部署Nacos

```shell
echo "---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
data:
  database.name: bmFjb3NkYg==
  # DB name: nacosdb
  root.password: ""
  # root password: null
  user.name: bmFjb3M=
  # username: nacos
  user.password: bWFnZWR1LmNvbQo=
  # password: magedu.com" |tee 01-secret-mysql.yaml|kubectl apply -f -
```