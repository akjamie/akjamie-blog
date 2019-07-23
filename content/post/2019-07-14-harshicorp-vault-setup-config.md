---
title: "Harshicorp Vault Setup Config and integration with spring boot"
subtitle: "Vault secures, stores, and tightly controls access to tokens, passwords, certificates, API keys, and other secrets in modern computing"
excerpt: ""
description: "介绍用docker-compose安装Vault并配置nginx代理，并与spring cloud application进行集成"
date: 2019-07-14 12:10:27
author: "Jamie Zhang"
image: "/img/background-01.jpg"
published: true
tags:
    - Harshicorp Vault
    - Docker
    - Spring Cloud
URL: "/2019/07/14/harshicorp-vault-setup-config/"
categories: [ Tech ]
---
# Install vault
Prepare *docker-compose.yml*, content as below (just a sample here, please don't use it for production, there are more aspects should be considered)
```
version: '3'
services:
  consul:
    container_name: consul.server
    command: agent -server -bind 0.0.0.0 -client 0.0.0.0 -bootstrap-expect=1
    image: consul:latest
    restart: always
    volumes:
      - ./consul.server/config:/consul/config
      - ./consul.server/data:/consul/data
    ports:
      - "9300:9300"
      - "9500:9500"
      - "9600:9600/udp"
  vault:
    container_name: vault.server
    image: vault
    ports:
    - "8200:8200"
    restart: always
    links:
        - consul:consul.server
    volumes:
    - ./vault.server/config:/mnt/vault/config
    - ./vault.server/data:/mnt/vault/data
    - ./vault.server/logs:/mnt/vault/logs
    - ./vault.server/config:/vault/config
    - /etc/letsencrypt:/etc/letsencrypt
    cap_add:
    - IPC_LOCK
    command: server
```
## consul config - config.json
```
   {
     "datacenter": "data-center-1",
     "node_name": "master-node",
     "log_level": "INFO",
     "server": true,
     "data_dir": "/consul/data",
     "ui" : true,
     "ports": {
       "http": 8500,
       "server": 8300
     }
   } 
```
## vault config - vault.hcl
```
ui = true
backend "consul" {
  address = "consul:8500"
  advertise_addr = "http://consul:8300"
  scheme = "http"
}
listener "tcp" {
  address = "0.0.0.0:8200
  tls_disable = 1
}
disable_mlock = true
```
## Use docker-compose up -d to run these services, and check its status
![](/img/2019-07-14-harshicorp-vault-setup-config/15532408-351f3fa54f1ebc1a.png)
![](/img/2019-07-14-harshicorp-vault-setup-config/15532408-698c9ee3afd1d4f9.png)
# Configure the vault
in #1, we created the vault container, but vault is still in uninitialized status (unsealed)
## Vault init

```
curl -X PUT \
  https://demo.xxx.com/vault/v1/sys/init \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: cb5d640a-152e-844f-cda3-5e8be4e3eec3' \
  -d '{"secret_shares":5, "secret_threshold":2}'
  
  {
    "keys": [
        "977595193b3a8cb4ae70338859f0a7264b2757775a51cca461731b1bff4b61798f",
        "9f6c61623da515b36671ca2677f5be084f670751e2da6a05d88970ebe832d6430b",
        "d1f13304840d0445e23ebe7ea39d23dc6821ddcb9fc97ce5ea2c8a9e82ca132ca0",
        "d49bda98c1723e659f5bb87af7774988e709ff9ad4337dc4c4851cf8a89c7dfd7f",
        "dfa480970d31b18b7c7943211cdd76636f51b17a55ee2d7a6718052a50d7198847"
    ],
    "keys_base64": [
        "l3WVGTs6jLSucDOIWfCnJksnV3daUcykYXMbG/9LYXmP",
        "n2xhYj2lFbNmccomd/W+CE9nB1Hi2moF2Ilw6+gy1kML",
        "0fEzBIQNBEXiPr5+o50j3Ggh3cufyXzl6iyKnoLKEyyg",
        "1JvamMFyPmWfW7h693dJiOcJ/5rUM33ExIUc+Kicff1/",
        "36SAlw0xsYt8eUMhHN12Y29RsXpV7i16ZxgFKlDXGYhH"
    ],
    "root_token": "s.nBXC2nmVs7IVLHk6zsU7ZoaT"
  }
```
## Vault unseal
```
curl -X PUT \
  https://demo.xxx.com/vault/v1/sys/unseal \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: a43db86c-26d4-de01-9a13-eefc23857eb7' \
  -d '{"key": "n2xhYj2lFbNmccomd/W+CE9nB1Hi2moF2Ilw6+gy1kML"}'
  
  {
    "type": "shamir",
    "initialized": true,
    "sealed": false,
    "t": 2,
    "n": 5,
    "progress": 0,
    "nonce": "",
    "version": "1.1.0",
    "migration": false,
    "cluster_name": "vault-cluster-8492c510",
    "cluster_id": "c73cee19-c9ab-81f1-9563-8fd060084370",
    "recovery_seal": false
  }
```
## create read/write token
a. create admin policy
```
curl -X POST \
  https://demo.xxx.com/vault/v1/sys/policy/admin-policy \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: c03aae97-d6c1-fa79-f8d5-f4476fe8347a' \
  -H 'X-Vault-Token: s.nBXC2nmVs7IVLHk6zsU7ZoaT' \
  -d '{"policy":"{\"path\":{\"secret/*\":{\"capabilities\":[\"create\", \"read\", \"update\", \"delete\", \"list\"]}}}"}'
```
b.create user policy
similar request as step <label style="color:red">create admin-policy</label>
c.check the policy we created
```
curl -X GET \
  https://demo.xxx.com/vault/v1/sys/policy/admin-policy \
  -H 'Accept: application/json' \
  -H 'Cache-Control: no-cache' \
  -H 'Postman-Token: 6874a6ca-070c-8d20-f4bb-183e1ff663d1' \
  -H 'X-Vault-Token: s.nBXC2nmVs7IVLHk6zsU7ZoaT'
  
{
  "name": "admin-policy",
  "rules": "{\"path\":{\"secrets/*\":{\"capabilities\":[\"create\", \"read\", \"update\", \"delete\", \"list\"]}}}",
  "request_id": "51ba3350-f14d-0e3f-27b4-610b6442d296",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
      "name": "admin-policy",
      "rules": "{\"path\":{\"secret/*\":{\"capabilities\":[\"create\", \"read\", \"update\", \"delete\", \"list\"]}}}"
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null
}  
```
## Generate Token against policies
```
curl -X POST \
  https://demo.xxx.com/vault/v1/auth/token/create \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: 21ef1963-b06d-b4ea-547c-03539a681ca5' \
  -H 'X-Vault-Token: s.nBXC2nmVs7IVLHk6zsU7ZoaT' \
  -d '{"policies": ["admin-policy"]}'

{
    "request_id": "a5251b70-e14c-5c56-a81e-6ad59445a44f",
    "lease_id": "",
    "renewable": false,
    "lease_duration": 0,
    "data": null,
    "wrap_info": null,
    "warnings": null,
    "auth": {
        "client_token": "s.AXTHvuudLS6C4PYjSvLLabr9",
        "accessor": "wVg70OelyhDMwa4MojtCKeZ1",
        "policies": [
            "admin-policy",
            "default"
        ],
        "token_policies": [
            "admin-policy",
            "default"
        ],
        "metadata": null,
        "lease_duration": 2764800,
        "renewable": true,
        "entity_id": "",
        "token_type": "service",
        "orphan": false
    }
}

{
    "request_id": "35212529-0b4c-3c50-0489-94df50a5f781",
    "lease_id": "",
    "renewable": false,
    "lease_duration": 0,
    "data": null,
    "wrap_info": null,
    "warnings": null,
    "auth": {
        "client_token": "s.7A1UuLB7aJdiGhn5Qaj2FlxN",
        "accessor": "8H2pMpG4SnKHMgH8nr56FsUJ",
        "policies": [
            "default",
            "user-policy"
        ],
        "token_policies": [
            "default",
            "user-policy"
        ],
        "metadata": null,
        "lease_duration": 2764800,
        "renewable": true,
        "entity_id": "",
        "token_type": "service",
        "orphan": false
    }
}
```
## Use token to read/write data
$\color{red}{Admin-token}$ to put key/value, otherwise got <label style="color:red">error occurred:permission denied</label>
```
curl -X POST \
  https://demo.xxx.com/vault/v1/secret/test \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: 0b3cd187-14fd-c94f-9198-794e71b906af' \
  -H 'X-Vault-Token: s.AXTHvuudLS6C4PYjSvLLabr9' \ 
 
  -d '{
	"token":"test-token"
}'
<label style="color:red">user-token</label> to get key/value

curl -X GET \
  https://demo.xxx.com/vault/v1/secret/test \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: 2f1e1e02-db11-5fdc-426e-ad0adc735b47' \
  -H 'X-Vault-Token: s.7A1UuLB7aJdiGhn5Qaj2FlxN' 

{
    "request_id": "8a1dfacd-7514-ae7d-8133-d3295fafb394",
    "lease_id": "",
    "renewable": false,
    "lease_duration": 2764800,
    "data": {
        "token": "test-token"
    },
    "wrap_info": null,
    "warnings": null,
    "auth": null
}
```
## Error: no handler for route 'secret/test'
check secrets engine if corresponding is enabled or not, commands: 
a. docker exec -it <container-name> /bin/sh
b.set VAULT_ADDR=http://127.0.0.1:8200, and VAULT_TOKEN={root token}
c. vault secrets list, to list out all secrets, by default,  kv secrets engine at: secret/ should be enabled, if not found, please move forward to #d
d. vault secrets enable -path=secret kv
e. test the secrets engine, vault kv put secret/hello foo=world excited=yes 
f. vault kv get secret/hello, to check the data u've just pushed

# Springboot application intergrates with Vault
## Create project and import related maven dependencies
```
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <spring.cloud.version>Greenwich.RELEASE</spring.cloud.version>
</properties>

<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.1.3.RELEASE</version>
</parent>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring.cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-vault-config</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-vault-config-databases</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
        <exclusions>
            <exclusion>
                <groupId>junit</groupId>
                <artifactId>junit</artifactId>
            </exclusion>
        </exclusions>
    </dependency>

    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-api</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-engine</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <optional>true</optional>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-configuration-processor</artifactId>
        <optional>true</optional>
    </dependency>

</dependencies>
```
## Prepare test data
```
curl -X POST \
  https://demo.xxx.com/v1/secret/vault-test \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: 6cad13c9-14d3-8260-dd74-6cef2b2ee822' \
  -H 'X-Vault-Token: s.AXTHvuudLS6C4PYjSvLLabr9' \
  -d '{
	"username":"Jamie",
	"password":"Hlhjjj&jkjlkjklhHHJKKYUJKjTFttrdfhHg#h$$HHHH"
}'
```
## bootstrap.yml
```
server:
  port: 8084

logging.level.org.springframework.vault: DEBUG

spring:
  application:
    name: vault-test
spring.cloud:
    vault:
      host: demo.xxx.com
      port: 443
      scheme: https
      authentication: TOKEN
      config:
          order: -10
      kv:
        application-name: vault-test
        default-context: vault-test
        backend: secret
        enabled: true
        profile-separator: /
      token: s.7A1UuLB7aJdiGhn5Qaj2FlxN  
```
## Implement a controller to inject the data loaded from vault
```
@RestController
public class TestController {

	@Value("${username:test}")
	private String userName;

	@Value("${password:test}")
	private String password;

	@GetMapping("/")
	@ResponseBody
	public Map<String, String> test() {
		Map<String, String> rsp = new HashMap<String, String>();
		rsp.put("userName", userName);
		rsp.put("password", password);    
		return rsp;
	}    
}
```
>
Findings found during testing:
*if change the spring.cloud.vault.config.order to a lower value, for example -100, the configs loaded from vault can take effects on spring auto-configured beans, like HikaiCP datasource, can put credentials onto vault in k/v format rather than keep these sensitive data in explicit config file, spring.datasource.username=xxx,spring.datasource.password=xxx*

## Test it through postman
```
curl -X GET \
  http://localhost:8084/ \
  -H 'Cache-Control: no-cache' \
  -H 'Postman-Token: 9466ffbc-e8d5-355f-cb2c-c29907a5af9d'

{
    "password": "Hlhjjj&jkjlkjklhHHJKKYUJKjTFttrdfhHg#h$$HHHH",
    "userName": "Jamie"
}
```
you can find sources code on [github](https://github.com/cloud-poc/springboot-vault-test)

参考文章: 
[https://www.vaultproject.io/docs/concepts/policies.html](https://www.vaultproject.io/docs/concepts/policies.html)
[https://www.hashicorp.com/resources/getting-vault-enterprise-installed-running](https://www.hashicorp.com/resources/getting-vault-enterprise-installed-running)
[https://learn.hashicorp.com/vault/secrets-management/sm-pki-engine](https://learn.hashicorp.com/vault/secrets-management/sm-pki-engine)
[https://www.jianshu.com/p/267f2d9ae87e](https://www.jianshu.com/p/267f2d9ae87e)
[https://blog.51cto.com/7308310/2338146?source=dra](https://blog.51cto.com/7308310/2338146?source=dra)
