Projeto MQTT Seguro: Arquitetura de Referência
Este projeto demonstra a implementação de um ecossistema MQTT seguro, robusto e containerizado, utilizando Docker e Docker Compose. Ele serve como uma arquitetura de referência para aplicações de IoT (Internet das Coisas) que exigem comunicação segura, autenticada e autorizada.

O sistema simula uma cadeia de eventos completa, composta por um broker central, um sensor que publica dados (publisher) e um cliente que monitora esses dados (subscriber).

Arquitetura do Sistema
O projeto é orquestrado pelo Docker Compose e consiste em três serviços principais que operam em uma rede isolada:

mqtt-broker: O coração do sistema. Utiliza a imagem oficial do Eclipse Mosquitto e é configurado para operar de forma segura, gerenciando todas as conexões e o fluxo de mensagens.

temperature-sensor-1: Um cliente MQTT escrito em Python que simula um sensor de temperatura. Ele se conecta ao broker de forma segura, se autentica e publica leituras de temperatura em um tópico específico a cada 5 segundos.

mqtt-subscriber: Um cliente leve, baseado na própria imagem do Mosquitto, que se inscreve em um tópico para receber e exibir as mensagens publicadas pelo sensor, permitindo a verificação em tempo real do fluxo de dados.

Principais Características
🔒 Segurança por Padrão: A comunicação é protegida em múltiplas camadas.

Criptografia TLS: Todo o tráfego entre os clientes e o broker é criptografado, prevenindo espionagem e ataques Man-in-the-Middle.

Autenticação Obrigatória: Acesso anônimo é desabilitado. Todos os clientes devem fornecer credenciais válidas para se conectar.

Autorização Granular (ACL): As permissões são rigidamente controladas. Cada cliente só pode acessar os tópicos necessários para sua função, seguindo o Princípio do Menor Privilégio.

🐳 Containerizado e Isolado: Todos os componentes rodam em contêineres Docker, garantindo um ambiente de execução consistente, portátil e isolado.

⚙️ Configuração Externalizada: As configurações sensíveis (senhas, usuários) são gerenciadas por meio de variáveis de ambiente (.env), separando o código dos segredos.

💾 Persistência de Dados: O broker é configurado para persistir dados e logs em volumes Docker, garantindo que mensagens retidas e registros não se percam.

Pré-requisitos
Para executar este projeto, você precisará ter instalado:

Docker (>= 20.10)

Docker Compose (>= 1.29)

Configuração Passo a Passo
Antes de iniciar os serviços, algumas etapas de configuração de segurança são necessárias.

1. Geração dos Certificados TLS
Para a criptografia funcionar, precisamos de certificados digitais. Para um ambiente de desenvolvimento, podemos gerar certificados autoassinados.

Crie o diretório para os certificados:

Bash

mkdir -p mosquitto/certs
Execute o comando openssl abaixo para gerar a Autoridade Certificadora (CA), a chave e o certificado do servidor:

Bash

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout mosquitto/certs/server.key \
  -out mosquitto/certs/server.cert \
  -subj "/CN=mqtt-broker"

# O comando acima também gera o server.cert, que funciona como nosso ca.crt neste caso.
# Para maior clareza, criamos uma cópia dele.
cp mosquitto/certs/server.cert mosquitto/certs/ca.crt
Isso criará três arquivos dentro de mosquitto/certs/ que o Mosquitto usará para habilitar o TLS.

2. Configuração de Credenciais (.env)
As credenciais dos clientes são gerenciadas externamente.
Copie o arquivo de exemplo para criar seu arquivo de configuração local:

Bash

cp .env.example .env
Agora, edite o arquivo .env e, se desejar, altere as senhas padrão:

Snippet de código

# mosquitto/config/.env
SENSOR_USER=sensor-user
SENSOR_PASSWORD=coloque_uma_senha_forte_aqui

SUBSCRIBER_USER=subscriber-user
SUBSCRIBER_PASSWORD=coloque_outra_senha_forte_aqui
3. Criação do Arquivo de Senhas do Mosquitto
O Mosquitto armazena as senhas em um arquivo próprio, de forma hasheada. Usaremos o Docker para executar o utilitário mosquitto_passwd e criar este arquivo.

Execute os comandos abaixo, substituindo as senhas pelas que você definiu no arquivo .env:

Bash

# Crie o diretório de configuração se ele não existir
mkdir -p mosquitto/config

# Crie o arquivo de senha com o primeiro usuário
docker run -it --rm -v "$(pwd)/mosquitto/config:/mosquitto/config" \
  eclipse-mosquitto:2.0.20 mosquitto_passwd -c -b /mosquitto/config/passwd sensor-user sua_senha_do_sensor

# Adicione o segundo usuário ao arquivo existente
docker run -it --rm -v "$(pwd)/mosquitto/config:/mosquitto/config" \
  eclipse-mosquitto:2.0.20 mosquitto_passwd -b /mosquitto/config/passwd subscriber-user sua_senha_do_subscriber
Isso criará o arquivo mosquitto/config/passwd com as credenciais criptografadas.

Execução do Projeto
Com toda a configuração de segurança no lugar, você pode iniciar o ecossistema.

Construir e Iniciar os Contêineres:

Bash

docker compose up -d --build
O -d executa em modo detached (em segundo plano).

Monitorar as Mensagens:
Para ver os dados de temperatura sendo recebidos em tempo real, acompanhe os logs do serviço de subscriber:

Bash

docker compose logs -f mqtt-subscriber
Você deverá ver mensagens como sensor/temperature 23.45 a cada 5 segundos.

Parar os Serviços:
Para parar e remover os contêineres, execute:

Bash

docker compose down