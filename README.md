# Cenário de acesso RDP Acme

Na era do trabalho remoto generalizado e dos serviços baseados em nuvem, é vital ter acesso remoto seguro e eficiente aos recursos. Este artigo explora uma solução que combina NGINX, um servidor HTTP de alto desempenho e proxy reverso, Apache Guacamole, um gateway de área de trabalho remota sem cliente e um servidor RDP para criar uma solução de acesso remoto segura, robusta e fácil de usar. Orientaremos você na configuração dessa arquitetura, com informações detalhadas sobre encaminhamento de porta, opções de autenticação e implantação do Guacamole usando o Docker.

Para exemplo coloco um modelo para como podemos implementar uma solução para acesso RDP como sugestão e um primeiro guideline.

> **Importante**: Não é uma proposta técnica e nem comercial e a ideia aqui é contribuir como um apoio técnico para quem interessar.


## **NGINX**

Abaixo está um exemplo simples de um arquivo de configuração NGINX (`nginx.conf`) que encaminha solicitações HTTP para um servidor chamado `acme.com` no caminho `/rdp` para um serviço Remote Desktop Protocol (RDP) em um servidor Windows com endereço IP `10.0.0.1` (por exemplo). A porta padrão RDP é 3389.

O Nginx não oferece suporte ao encaminhamento de tráfego diretamente para o serviço RDP, pois ele opera em um protocolo completamente diferente (TCP) do HTTP/HTTPS, que é o que o Nginx foi projetado para trabalhar. No entanto, é possível com o uso de uma ferramenta como **Guacamole** que fornece RDP como um serviço da Web com o qual o Nginx pode interagir.

**nginx.conf**

```nginx
worker_processes 1;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;

    server {
        listen 80;

        server_name ccme.com;

        location /rdp {
            proxy_pass http://10.0.0.1:8080/guacamole/; # Partindo da premissa que o guacamole está na porta 8080
            proxy_buffering off;
            proxy_http_version 1.1;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
}
```

Esta configuração assume que o Guacamole está sendo executado em seu Windows Server (10.0.0.1) e está acessível na porta 8080. O Guacamole fornece uma interface baseada na web para conexões RDP, e é o `proxy_pass` nesta configuração que aponta para esse serviço. No tópico referente ao Guacamole coloco um exemplo de subir ele num container, que é uma implementação simples também e escala bem.

Outro ponto, é que esta configuração não inclui nenhuma configuração HTTPS, que geralmente é recomendada para qualquer coisa voltada para a Internet. Este é um exemplo muito básico e você gostaria de incluir configuração HTTPS, log, tratamento de erros e possivelmente outra configuração, dependendo do seu caso de uso.

**Lembrando que**: O NGINX não oferece suporte nativo ao RDP porque são diferentes tipos de protocolos (o RDP usa TCP e UDP enquanto o NGINX usa HTTP/HTTPS). A configuração acima destina-se a ilustrar como você pode configurar o NGINX para reverter o proxy para um servidor Guacamole, que pode lidar com conexões RDP.

###
O NGINX oferece suporte à autenticação HTTP básica, na qual os usuários podem ser autenticados usando um arquivo de senha local.

Veja como você pode configurar isso:

1. **Instale o pacote `httpd-tools`**. Este pacote contém o utilitário `htpasswd`, que você usará para criar o arquivo de senha. Dependendo do seu sistema, o comando de instalação pode ser `sudo apt-get install apache2-utils` (Ubuntu) ou `sudo yum install httpd-tools` (CentOS).

2. **Crie um arquivo de senha**. Use o utilitário `htpasswd` para criar um arquivo de senha. Aqui está um exemplo de comando:

    ```
    sudo htpasswd -c /etc/nginx/.htpasswd user1
    ```

    A opção `-c` cria um novo arquivo. `user1` é o nome de usuário. Você será solicitado a inserir e confirmar uma senha para o usuário. Se você deseja adicionar usuários adicionais, omita a opção `-c`:

    ```
    sudo htpasswd /etc/nginx/.htpasswd user2
    ```

3. **Atualize a configuração do NGINX**. Em seu arquivo `nginx.conf`, você precisará especificar o caminho para o arquivo de senha que acabou de criar usando as diretivas `auth_basic` e `auth_basic_user_file`. Aqui está um exemplo de configuração:

```nginx
    worker_processes 1;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;

    server {
        listen 80;

        server_name acme.com;

        location /rdp {
            auth_basic "Restricted Content";    # Enable Basic Auth
            auth_basic_user_file /etc/nginx/.htpasswd;  # Path to the password file

            proxy_pass http://10.0.0.1:8080/guacamole/;
            proxy_buffering off;
            proxy_http_version 1.1;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
}

```

Isso solicitará que os usuários insiram um nome de usuário e senha quando navegarem para "acme.com/rdp". O nome de usuário e a senha devem corresponder a uma das entradas no arquivo de senha.

4. **Recarregue o NGINX**. Depois de salvar suas alterações, você precisará recarregar a configuração do NGINX:

    ```
    sudo systemctl reload nginx
    ```

Lembre-se de que a Autenticação básica HTTP transmite credenciais em texto simples (codificado em base64), portanto, ela sempre deve ser usada em conjunto com HTTPS para proteger as credenciais.
### NGINX com autenticação do office 365

É possível usar o Microsoft Office 365 (especificamente o Azure AD) para autenticar o acesso aos seus aplicativos com proxy por meio do NGINX. Esse processo envolve a configuração de um cliente OAuth 2.0 no Azure AD, o que permitirá que o NGINX delegue a autenticação ao Azure AD. Dessa forma, os usuários devem primeiro fazer login com suas credenciais do Office 365 antes de poderem acessar o aplicativo.

No entanto, esse processo não é direto e não pode ser alcançado apenas usando a funcionalidade integrada do NGINX. Você precisará de um componente adicional conhecido como proxy de autenticação, que atua como um middleware que pode lidar com o processo de autenticação OAuth/OIDC. O `oauth2_proxy` (recentemente renomeado para `oauth2-proxy`) é uma ferramenta comumente usada para esse propósito.

Aqui está um exemplo simples de como seu arquivo `nginx.conf` pode ficar depois de configurá-lo:

```nginx
worker_processes 1;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;

    server {
        listen 80;

        location /oauth2/ {
            proxy_pass       http://oauth2-proxy;
            proxy_set_header Host                    $host;
            proxy_set_header X-Real-IP               $remote_addr;
            proxy_set_header X-Scheme                $scheme;
            proxy_set_header X-Auth-Request-Redirect $request_uri;
        }

        location / {
            auth_request /oauth2/auth;
            error_page 401 = /oauth2/sign_in;

            proxy_pass http://guacamole:8080/guacamole/;
            add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

            # Include proxy params common to all applications
            include proxy_params;
        }
    }
}
```

Este exemplo de arquivo `nginx.conf` configura a autenticação via `oauth2_proxy` para o aplicativo Guacamole.

No entanto, configurar isso envolve mais do que apenas a configuração do NGINX. Você também precisará:

1. Registre um cliente OAuth 2.0 no Azure AD.
2. Configure e execute uma instância `oauth2_proxy`, passando seu ID e segredo do cliente, entre outras coisas.
3. Configure o NGINX para usar `oauth2_proxy` como um provedor de autenticação, conforme mostrado no exemplo `nginx.conf`.

Para obter detalhes sobre como configurar o Azure AD OAuth 2.0 e `oauth2_proxy`, consulte a documentação oficial do projeto Azure AD e `oauth2_proxy`.

## **GUACAMOLE**

Vale entrar no site deles para entender o que eles fazem. A Solução é opensource.

[Guacamole Web Site](https://guacamole.apache.org/)

A instalação do Apache Guacamole envolve várias etapas, incluindo a configuração de um servidor Tomcat, a criação de um banco de dados MySQL ou PostgreSQL para uso do Guacamole e a compilação do Guacamole a partir do código-fonte. Você pode usar o Docker para simplificar um pouco o processo.

Aqui está um guia passo a passo para implantar o Guacamole no Docker e conectar-se a um servidor RDP:

1. **Instale o Docker e o Docker Compose**

    Primeiro, certifique-se de que o Docker e o Docker Compose estejam instalados em seu servidor. Caso contrário, siga os guias de instalação oficiais do Docker para instalar o Docker Engine e o Docker Compose.

2. **Puxe as imagens do Guacamole Docker**

    O Guacamole é dividido em duas partes: "guacamole-server", que fornece o componente proxy nativo do lado do servidor, e "guacamole-client", que fornece o aplicativo da Web HTML5. Ambos são necessários. Felizmente, existe uma imagem oficial do Docker que fornece ambos juntos. Você pode obter essas imagens usando o comando `docker pull`:

    ```bash
    docker pull guacamole/guacamole
    docker pull guacamole/guacd
    docker pull mysql:5.7
    ```

3. **Crie um arquivo de composição do Docker**

    Em seguida, crie um arquivo `docker-compose.yml` para gerenciar esses serviços. Nesse arquivo, configuraremos os contêineres guacamole, guacd (o daemon do proxy) e MySQL.

   ```yaml
   version: '3'
   services:
       guacd:
           image: guacamole/guacd
           restart: always
       guacamole:
           image: guacamole/guacamole
           depends_on:
               - guacd
               - mysql
           environment:
               GUACD_HOSTNAME: guacd
               MYSQL_HOSTNAME: mysql
               MYSQL_DATABASE: guacamole_db
               MYSQL_USER: guacamole_user
               MYSQL_PASSWORD: guacamole_password
           ports:
               - 8080:8080
           restart: always
       mysql:
           image: mysql:5.7
           environment:
               MYSQL_ROOT_PASSWORD: root_password
               MYSQL_DATABASE: guacamole_db
               MYSQL_USER: guacamole_user
               MYSQL_PASSWORD: guacamole_password
           volumes:
               - ./initdb:/docker-entrypoint-initdb.d
           restart: always
   ```

Neste arquivo, substitua `guacamole_db`, `guacamole_user`, `guacamole_password` e `root_password` por seus próprios valores.

4. **Inicializar o banco de dados**

    Guacamole precisa de um banco de dados inicializado para funcionar corretamente. Você pode encontrar scripts SQL para isso na imagem oficial do Guacamole Docker. Primeiro, crie um diretório para esses scripts:

    ```bash
    mkdir initdb
    ```

    Em seguida, extraia os scripts:

    ```bash
    docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --mysql > initdb/initdb.sql
    ```

5. **Inicie os Serviços**

    Com todos esses componentes instalados, você pode iniciar seu servidor Guacamole com o seguinte comando:

    ```bash
    docker-compose up -d
    ```

    Depois de executar esse comando, o Docker extrairá as imagens necessárias (se ainda não tiverem sido baixadas), criará os contêineres necessários e iniciará os serviços.

6. **Configurar Guacamole**

    Depois que tudo estiver em execução, você poderá acessar a tela de login do Guacamole navegando para `http://<your-server-ip>:8080/guacamole/` em seu navegador da web. O nome de usuário padrão é `guacadmin` com a senha `guacadmin`.

    Após o login, você pode adicionar seu servidor RDP em "Conexões" no menu de configurações. Insira o endereço IP (10.0.0.1), porta (3389) e informações de autenticação para o servidor RDP.

Lembre-se de que esta é uma configuração básica destinada a fins de teste e não é segura para uso em produção. Para um ambiente de produção, você deve proteger sua instalação do Guacamole com HTTPS e alterar as credenciais de login padrão.

## Arquitetura Final

O diagrama de arquitetura de alto nível para a solução que integra NGINX, Guacamole e o RDP Server ficaria assim:

```text
+------------------+      +--------------+     +----------------+     +---------------+
|                  |      |              |     |                |     |               |
|  User's Browser  <----->  NGINX Server |<--->|   Guacamole    |<--->| RDP Server    |
|                  |      |              |     |  Docker Image  |     |               |
+------------------+      +--------------+     +----------------+     +---------------+
    (Internet)            (DMZ Zone)              (DMZ Zone)         (Internal Network)
```

1. **Navegador do usuário**: Os usuários interagem com o sistema a partir de seus navegadores. Eles fazem solicitações ao servidor NGINX.

2. **Servidor NGINX**: O servidor NGINX é responsável por receber as solicitações do cliente e fazer proxy delas para a imagem Guacamole Docker. Ele atua como um proxy reverso e fornece uma camada de controle e segurança. O servidor NGINX geralmente está localizado em uma zona desmilitarizada (DMZ) e é o único sistema exposto à Internet.

3. **Guacamole Docker Image**: É aqui que o servidor Guacamole (composto pelos serviços guacamole e guacd) está sendo executado. Ele fornece uma interface baseada na Web em HTML5 para conexões RDP. O Guacamole recebe as solicitações de proxy do NGINX, interpreta-as e envia os comandos apropriados ao servidor RDP.

4. **Servidor RDP**: Este é o Windows Server ao qual os usuários desejam se conectar. O servidor RDP executa os comandos que recebe do Guacamole e retorna o resultado, que o Guacamole envia de volta ao cliente via NGINX.

>*Nota*: Esta é uma versão simplificada de uma arquitetura potencial. Em um cenário do mundo real, haveria mais considerações sobre segurança, alta disponibilidade, desempenho, monitoramento e assim por diante. Por exemplo, você pode querer adicionar um firewall ou usar um Sistema de Prevenção de Intrusão (IPS). Você também pode querer considerar o balanceamento de carga se espera lidar com muitas conexões simultâneas. A criptografia (HTTPS/TLS) também deve ser implementada para todo o tráfego baseado na web.

# Conclusão
Em resumo, integrando os recursos do NGINX, Apache Guacamole e um servidor RDP, podemos implementar um serviço de desktop remoto seguro e eficiente. Examinamos como essas tecnologias interagem para formar uma solução segura e fácil de usar, com estratégias de autenticação aprimorando ainda mais a segurança. No entanto, lembre-se de que necessidades e requisitos individuais podem afetar sua arquitetura e configurações e devem sempre ser levados em consideração no projeto e na implementação de sua solução de acesso remoto. Compreender e alavancar essas tecnologias é fundamental para estabelecer uma solução de acesso remoto seguro e eficaz.

##### Criado por Taígo Soares