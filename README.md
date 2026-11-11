# Aplicação PHP, MySQL e phpMyAdmin em Kubernetes

Conjunto de manifests de aprendizagem para uma aplicação PHP, MySQL e uma interface administrativa phpMyAdmin. O código-fonte da aplicação e a sua imagem final **não estão incluídos**: `bookstore.yaml` contém o placeholder `conta/repositorio`, que precisa de ser substituído.

## Componentes

| Manifest | Função |
| --- | --- |
| `bookstore.yaml` | Deployment da aplicação, porta 80 |
| `bookstore-service.yaml` | NodePort 30088, Service port 8088 |
| `bookstore-ingress.yaml` | Regra de host para um Ingress Controller existente |
| `mysql.yaml`, `mysql-service.yaml` | MySQL 8 e Service interno na porta 3306 |
| `phpmyadmin.yaml`, `phpmyadmin-service.yaml` | Interface administrativa, apenas Service interno |
| `mysql-secret.yaml` | Exemplo histórico com credenciais públicas de demonstração; não usar num ambiente real |
| `phpmyadmin-configmap.yaml` | Exemplo de ConfigMap; o Deployment atual usa PMA_HOST diretamente |

MySQL e phpMyAdmin usam `ClusterIP`: não precisam de um LoadBalancer público para este laboratório.

## Requisitos e preparação

Um cluster Kubernetes de teste, `kubectl`, uma imagem PHP acessível aos nodes e conhecimento da configuração/schema exigidos pela aplicação. Editar a imagem de `bookstore.yaml` e fornecer a configuração da base de dados que essa imagem exigir.

Se for usado Ingress, instalar primeiro um controller, configurar a classe suportada pelo cluster e trocar `mybookstore.com` pelo host de teste. Criar um recurso Ingress não instala o controller nem configura DNS.

## Implementar num namespace dedicado

Exemplos em Bash:

```bash
kubectl create namespace bookstore-lab
read -r -s -p "Palavra-passe MySQL do laboratório: " MYSQL_ROOT_PASSWORD
printf '\n'
kubectl -n bookstore-lab create secret generic mysql-secret --from-literal=mysql-root-username=root --from-literal=mysql-root-password="$MYSQL_ROOT_PASSWORD"
unset MYSQL_ROOT_PASSWORD
kubectl -n bookstore-lab apply -f mysql.yaml -f mysql-service.yaml
kubectl -n bookstore-lab rollout status deployment/mysql-deployment
kubectl -n bookstore-lab apply -f bookstore.yaml -f bookstore-service.yaml
kubectl -n bookstore-lab rollout status deployment/bookstore-app
kubectl -n bookstore-lab apply -f phpmyadmin.yaml -f phpmyadmin-service.yaml
```

Não aplicar a pasta inteira: isso incluiria o Secret histórico e um Ingress ainda não adaptado. A variável MYSQL_ROOT_USERNAME no manifest MySQL não cria um utilizador da aplicação; configurar esse utilizador/schema de acordo com a imagem PHP.

## Testar

```bash
kubectl -n bookstore-lab get pods,services
kubectl -n bookstore-lab port-forward service/bookstore-service 8088:8088
```

Abrir `http://127.0.0.1:8088/`. Noutra consola, usar `kubectl -n bookstore-lab port-forward service/phpmyadmin-service 8081:80` para administração local. Fazer login explicitamente com as credenciais do laboratório; o Deployment já não configura login automático como root.

## Limitações importantes

MySQL não tem um volume persistente neste exemplo: a substituição do pod pode perder dados. Não há migrações/schema da aplicação, configuração de TLS, política de backups ou probes de readiness incluídas. Não considerar o Deployment RUNNING como prova de que a aplicação comunica com MySQL.

## Limpeza

```bash
kubectl delete namespace bookstore-lab
```

Usar apenas se o namespace tiver sido criado exclusivamente para este exercício. Em clusters cloud, verificar também serviços/volumes/controllers faturáveis que tenham sido criados fora do namespace.
