### Documentação do Processo de Configuração do Ambiente Icecast com Nginx

#### Introdução

Este guia documenta o processo de configuração de um ambiente Docker para transmitir áudio usando o servidor Icecast, com o Nginx configurado como proxy reverso. A configuração inclui um arquivo `docker-compose.yaml` que orquestra os containers Docker necessários para rodar o Icecast e o Nginx.

#### 1. Estrutura do Projeto

A estrutura do projeto é organizada da seguinte forma:

```
/icecast-nginx-butt
├── icecast
│   ├── Dockerfile
│   ├── icecast.xml
│   └── start.sh
├── nginx
│   ├── Dockerfile
│   └── nginx.conf
└── docker-compose.yaml
```

#### 2. Configuração do Icecast

**Arquivo: `icecast/Dockerfile`**

Este Dockerfile configura o servidor Icecast. A imagem base utilizada é `debian:bullseye`, e o Icecast é instalado juntamente com algumas dependências necessárias:

```Dockerfile
FROM debian:bullseye

ENV DEBIAN_FRONTEND noninteractive

RUN apt-get -qq -y update && \
    apt-get -qq -y install icecast2 python-setuptools sudo && \
    apt-get clean

COPY "icecast.xml" "/usr/share/icecast2/icecast.xml"

RUN chown -R icecast2 /usr/share/icecast2

VOLUME ["/var/log/icecast2", "/usr/share/icecast2"]

ADD ./start.sh /start.sh

EXPOSE 8000

CMD ["/start.sh"]
```

**Decisões Tomadas:**
- **Imagem Base:** Utilizou-se `debian:bullseye` como imagem base por sua estabilidade.
- **Configuração do Icecast:** O arquivo `icecast.xml` é copiado para o diretório padrão do Icecast dentro do container.
- **Permissões:** As permissões do diretório `/usr/share/icecast2` são atribuídas ao usuário `icecast2` para garantir que o Icecast tenha acesso adequado.
- **Volume:** Dois volumes são definidos para persistência de logs e arquivos de configuração.
- **Script de Inicialização:** O script `start.sh` é utilizado para configurar dinamicamente as senhas e iniciar o serviço.

**Arquivo: `icecast/icecast.xml`**

Este arquivo contém a configuração do Icecast, que define parâmetros como limites de clientes, autenticação, portas de escuta e pontos de montagem.

**Arquivo: `icecast/start.sh`**

Este script é executado na inicialização do container, substituindo as senhas do Icecast pelas definidas nas variáveis de ambiente:

```sh
#!/bin/sh

env

set -x

if [ -n "$ICECAST_SOURCE_PASSWORD" ]; then
    sed -i "s/<source-password>[^<]*<\/source-password>/<source-password>$ICECAST_SOURCE_PASSWORD<\/source-password>/g" /usr/share/icecast2/icecast.xml
fi

# Outros comandos de substituição de senha...

sudo -Eu icecast2 icecast2 -n -c /usr/share/icecast2/icecast.xml
```

**Decisões Tomadas:**
- **Senhas Dinâmicas:** As senhas são configuradas dinamicamente com base nas variáveis de ambiente, tornando o ambiente mais flexível.

#### 3. Configuração do Nginx

**Arquivo: `nginx/Dockerfile`**

Este Dockerfile configura o servidor Nginx:

```Dockerfile
FROM nginx:latest

COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

**Decisões Tomadas:**
- **Imagem Base:** Utiliza-se `nginx:latest` para garantir que o Nginx esteja sempre atualizado.
- **Configuração Simples:** A configuração do Nginx é minimalista, utilizando o arquivo `nginx.conf`.

**Arquivo: `nginx/nginx.conf`**

Este arquivo configura o Nginx como proxy reverso para o Icecast:

```nginx
events {}

http {
    server {
        listen 80;

        location / {
            proxy_pass http://icecast:8000/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

**Decisões Tomadas:**
- **Proxy Reverso:** O Nginx redireciona todas as requisições HTTP para o Icecast, centralizando o ponto de entrada do serviço.

#### 4. Orquestração com Docker Compose

**Arquivo: `docker-compose.yaml`**

Este arquivo orquestra a construção e execução dos containers Icecast e Nginx:

```yaml
version: '3.8'

services:
  icecast:
    build:
      context: icecast/
    environment:
      - ICECAST_SOURCE_PASSWORD=suhett
      - ICECAST_ADMIN_PASSWORD=suhett
      - ICECAST_PASSWORD=suhett
      - ICECAST_RELAY_PASSWORD=suhett
    ports:
    - 8000:8000
    networks:
      - stream-network

  nginx:
    build: ./nginx
    ports:
      - "80:80"
    depends_on:
      - icecast
    networks:
      - stream-network

networks:
  stream-network:
    driver: bridge
```

**Decisões Tomadas:**
- **Rede Compartilhada:** Utiliza-se uma rede `bridge` para permitir a comunicação entre os containers Icecast e Nginx.
- **Dependência:** O Nginx é configurado para depender do Icecast, garantindo que o proxy reverso só seja iniciado após o servidor de streaming estar operacional.

#### 5. Instruções de Execução

1. **Clone o repositório:**

   ```sh
   git clone https://github.com/filipesuhett/icecast-nginx-butt.git
   cd icecast-nginx-butt
   ```

2. **Construa e inicie os containers:**

   ```sh
   docker-compose up --build
   ```

3. **Acesse o stream:**
   - Abra um navegador e acesse `http://localhost/stream` para visualizar o stream através do Nginx.

#### 6. Conclusão

Este documento detalha a configuração de um ambiente de streaming de áudio utilizando Icecast e Nginx, encapsulado em containers Docker. As decisões tomadas visaram simplicidade, segurança e flexibilidade, permitindo fácil replicação do ambiente em qualquer máquina que suporte Docker.